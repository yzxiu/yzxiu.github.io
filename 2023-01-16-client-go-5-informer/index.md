# client-go解析(5) - informer & sharedIndexInformer


## 概述

NewSharedIndexInformer() 是出镜率非常高的函数，最终创建了一个 sharedIndexInformer。

我们这里暂且将常说的 “informer” 等同于 sharedIndexInformer。



## 接口

### cache.SharedInformer

```go
type SharedInformer interface {
   AddEventHandler(handler ResourceEventHandler) (ResourceEventHandlerRegistration, error)
   AddEventHandlerWithResyncPeriod(handler ResourceEventHandler, resyncPeriod time.Duration) (ResourceEventHandlerRegistration, error)
   RemoveEventHandler(handle ResourceEventHandlerRegistration) error
   GetStore() Store
   GetController() Controller
   Run(stopCh <-chan struct{})
   HasSynced() bool
   LastSyncResourceVersion() string
   SetWatchErrorHandler(handler WatchErrorHandler) error
   SetTransform(handler TransformFunc) error
   IsStopped() bool
}
```

接口 `cache.SharedInformer` 在 `cache.Controller` 的基础上，拓展了一些与 `share` 相关的方法，如下：

```go
   AddEventHandler(handler ResourceEventHandler) (ResourceEventHandlerRegistration, error)
   AddEventHandlerWithResyncPeriod(handler ResourceEventHandler, resyncPeriod time.Duration) (ResourceEventHandlerRegistration, error)
   RemoveEventHandler(handle ResourceEventHandlerRegistration) error
   GetStore() Store
   GetController() Controller
   SetWatchErrorHandler(handler WatchErrorHandler) error
   SetTransform(handler TransformFunc) error
   IsStopped() bool
```

这些方法中，与handler相关的：

AddEventHandler()

AddEventHandlerWithResyncPeriod()

RemoveEventHandler()

### cache.SharedIndexInformer

```go
// SharedIndexInformer provides add and get Indexers ability based on SharedInformer.
type SharedIndexInformer interface {
   SharedInformer
   // AddIndexers add indexers to the informer before it starts.
   AddIndexers(indexers Indexers) error
   GetIndexer() Indexer
}
```

cache.SharedIndexInformer 在 cache.SharedInformer 的基础上，拓展了 Index 相关方法。



## cache.sharedIndexInformer

```go
type sharedIndexInformer struct {
   indexer    Indexer
   controller Controller
   processor             *sharedProcessor
   cacheMutationDetector MutationDetector
   listerWatcher ListerWatcher
   objectType runtime.Object
   objectDescription string
   resyncCheckPeriod time.Duration
   defaultEventHandlerResyncPeriod time.Duration
   clock clock.Clock
   started, stopped bool
   startedLock      sync.Mutex
   blockDeltas sync.Mutex
   watchErrorHandler WatchErrorHandler
   transform TransformFunc
}
```

与 cache.controller 类似，sharedIndexInformer 的主要逻辑都在 Run 方法中

### Run()

```go
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
   defer utilruntime.HandleCrash()

   if s.HasStarted() {
      klog.Warningf("The sharedIndexInformer has started, run more than once is not allowed")
      return
   }
   fifo := NewDeltaFIFOWithOptions(DeltaFIFOOptions{
      KnownObjects:          s.indexer,
      EmitDeltaTypeReplaced: true,
   })

   cfg := &Config{
      Queue:             fifo,
      ListerWatcher:     s.listerWatcher,
      ObjectType:        s.objectType,
      ObjectDescription: s.objectDescription,
      FullResyncPeriod:  s.resyncCheckPeriod,
      RetryOnError:      false,
      ShouldResync:      s.processor.shouldResync,
	  // 重点1
      Process:           s.HandleDeltas,
      WatchErrorHandler: s.watchErrorHandler,
   }

   func() {
      s.startedLock.Lock()
      defer s.startedLock.Unlock()

      s.controller = New(cfg)
      s.controller.(*controller).clock = s.clock
      s.started = true
   }()

   // Separate stop channel because Processor should be stopped strictly after controller
   processorStopCh := make(chan struct{})
   var wg wait.Group
   defer wg.Wait()              // Wait for Processor to stop
   defer close(processorStopCh) // Tell Processor to stop
   wg.StartWithChannel(processorStopCh, s.cacheMutationDetector.Run)
   // 重点2
   wg.StartWithChannel(processorStopCh, s.processor.run)

   defer func() {
      s.startedLock.Lock()
      defer s.startedLock.Unlock()
      s.stopped = true // Don't want any new listeners
   }()
   s.controller.Run(stopCh)
}
```

sharedIndexInformer 的 Run 方法看起来并不复杂：

1. 使用自身的 s.indexer 作为 KnownObjects 构建 DeltaFIFO：`fifo`
2. 使用自身的 **s.HandleDeltas** 作为 Process 构建 cache.Config：`cfg`
3. 使用上面构建的 Config 构建 cache.Controller：`New(cfg)`
4. 启动 **s.processor**： `s.processor.run`
5. 启动 cache.Controller：`s.controller.Run(stopCh)`

从前面分析可知，DeltaFIFO队列的消费首先会通过 config.Process，查看 s.HandleDeltas

```go
func (s *sharedIndexInformer) HandleDeltas(obj interface{}, isInInitialList bool) error {
   s.blockDeltas.Lock()
   defer s.blockDeltas.Unlock()

   if deltas, ok := obj.(Deltas); ok {
      return processDeltas(s, s.indexer, s.transform, deltas, isInInitialList)
   }
   return errors.New("object given as Process argument is not Deltas")
}

func processDeltas(
	// Object which receives event notifications from the given deltas
	handler ResourceEventHandler,
	clientState Store,
	transformer TransformFunc,
	deltas Deltas,
	isInInitialList bool,
) error {}
```

sharedIndexInformer 自身作为 handler，查看其作为handler实现的方法：

```go
// Conforms to ResourceEventHandler
func (s *sharedIndexInformer) OnAdd(obj interface{}, isInInitialList bool) {
   // Invocation of this function is locked under s.blockDeltas, so it is
   // save to distribute the notification
   s.cacheMutationDetector.AddObject(obj)
   s.processor.distribute(addNotification{newObj: obj, isInInitialList: isInInitialList}, false)
}

// Conforms to ResourceEventHandler
func (s *sharedIndexInformer) OnUpdate(old, new interface{}) {
   isSync := false

   // If is a Sync event, isSync should be true
   // If is a Replaced event, isSync is true if resource version is unchanged.
   // If RV is unchanged: this is a Sync/Replaced event, so isSync is true

   if accessor, err := meta.Accessor(new); err == nil {
      if oldAccessor, err := meta.Accessor(old); err == nil {
         // Events that didn't change resourceVersion are treated as resync events
         // and only propagated to listeners that requested resync
         isSync = accessor.GetResourceVersion() == oldAccessor.GetResourceVersion()
      }
   }

   // Invocation of this function is locked under s.blockDeltas, so it is
   // save to distribute the notification
   s.cacheMutationDetector.AddObject(new)
   s.processor.distribute(updateNotification{oldObj: old, newObj: new}, isSync)
}

// Conforms to ResourceEventHandler
func (s *sharedIndexInformer) OnDelete(old interface{}) {
   // Invocation of this function is locked under s.blockDeltas, so it is
   // save to distribute the notification
   s.processor.distribute(deleteNotification{oldObj: old}, false)
}
```

交给 s.processor.distribute() 处理，继续查看：

```go
func (p *sharedProcessor) distribute(obj interface{}, sync bool) {
   p.listenersLock.RLock()
   defer p.listenersLock.RUnlock()

   for listener, isSyncing := range p.listeners {
      switch {
      case !sync:
         // non-sync messages are delivered to every listener
         listener.add(obj)
      case isSyncing:
         // sync messages are delivered to every syncing listener
         listener.add(obj)
      default:
         // skipping a sync obj for a non-syncing listener
      }
   }
}

func (p *processorListener) add(notification interface{}) {
	if a, ok := notification.(addNotification); ok && a.isInInitialList {
		p.syncTracker.Start()
	}
	p.addCh <- notification
}
```

最终，交给 sharedProcessor.processor.listeners 处理，这里的 listerer 有多个，这就是 `share`  体现。

程序可以在多个地方监听资源，但只会启动一个 cache.Reflector(或者说，cache.controller)，cache.Reflector 进行 `Pop` 资源消费的时候，被转发到 sharedProcessor.processor.listeners 中保存的多个监听者。



## sharedProcessor

```go
// sharedProcessor has a collection of processorListener and can
// distribute a notification object to its listeners.  There are two
// kinds of distribute operations.  The sync distributions go to a
// subset of the listeners that (a) is recomputed in the occasional
// calls to shouldResync and (b) every listener is initially put in.
// The non-sync distributions go to every listener.
type sharedProcessor struct {
   listenersStarted bool
   listenersLock    sync.RWMutex
   // Map from listeners to whether or not they are currently syncing
   listeners map[*processorListener]bool
   clock     clock.Clock
   wg        wait.Group
}
```







## 小结


















