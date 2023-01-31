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

添加 listener

```go
func (s *sharedIndexInformer) AddEventHandlerWithResyncPeriod(handler ResourceEventHandler, resyncPeriod time.Duration) (ResourceEventHandlerRegistration, error) {
   s.startedLock.Lock()
   defer s.startedLock.Unlock()

   if s.stopped {
      return nil, fmt.Errorf("handler %v was not added to shared informer because it has stopped already", handler)
   }

   if resyncPeriod > 0 {
      if resyncPeriod < minimumResyncPeriod {
         klog.Warningf("resyncPeriod %v is too small. Changing it to the minimum allowed value of %v", resyncPeriod, minimumResyncPeriod)
         resyncPeriod = minimumResyncPeriod
      }

      if resyncPeriod < s.resyncCheckPeriod {
         if s.started {
            klog.Warningf("resyncPeriod %v is smaller than resyncCheckPeriod %v and the informer has already started. Changing it to %v", resyncPeriod, s.resyncCheckPeriod, s.resyncCheckPeriod)
            resyncPeriod = s.resyncCheckPeriod
         } else {
            // if the event handler's resyncPeriod is smaller than the current resyncCheckPeriod, update
            // resyncCheckPeriod to match resyncPeriod and adjust the resync periods of all the listeners
            // accordingly
            s.resyncCheckPeriod = resyncPeriod
            s.processor.resyncCheckPeriodChanged(resyncPeriod)
         }
      }
   }

   listener := newProcessListener(handler, resyncPeriod, determineResyncPeriod(resyncPeriod, s.resyncCheckPeriod), s.clock.Now(), initialBufferSize, s.HasSynced)

   if !s.started {
      return s.processor.addListener(listener), nil
   }

   // in order to safely join, we have to
   // 1. stop sending add/update/delete notifications
   // 2. do a list against the store
   // 3. send synthetic "Add" events to the new handler
   // 4. unblock
   s.blockDeltas.Lock()
   defer s.blockDeltas.Unlock()

   handle := s.processor.addListener(listener)
   for _, item := range s.indexer.List() {
      // Note that we enqueue these notifications with the lock held
      // and before returning the handle. That means there is never a
      // chance for anyone to call the handle's HasSynced method in a
      // state when it would falsely return true (i.e., when the
      // shared informer is synced but it has not observed an Add
      // with isInitialList being true, nor when the thread
      // processing notifications somehow goes faster than this
      // thread adding them and the counter is temporarily zero).
      listener.add(addNotification{newObj: item, isInInitialList: true})
   }
   return handle, nil
}
```

## processorListener

```go
// processorListener relays notifications from a sharedProcessor to
// one ResourceEventHandler --- using two goroutines, two unbuffered
// channels, and an unbounded ring buffer.  The `add(notification)`
// function sends the given notification to `addCh`.  One goroutine
// runs `pop()`, which pumps notifications from `addCh` to `nextCh`
// using storage in the ring buffer while `nextCh` is not keeping up.
// Another goroutine runs `run()`, which receives notifications from
// `nextCh` and synchronously invokes the appropriate handler method.
//
// processorListener also keeps track of the adjusted requested resync
// period of the listener.
type processorListener struct {
   nextCh chan interface{}
   addCh  chan interface{}

   handler ResourceEventHandler

   syncTracker *synctrack.SingleFileTracker

   // pendingNotifications is an unbounded ring buffer that holds all notifications not yet distributed.
   // There is one per listener, but a failing/stalled listener will have infinite pendingNotifications
   // added until we OOM.
   // TODO: This is no worse than before, since reflectors were backed by unbounded DeltaFIFOs, but
   // we should try to do something better.
   pendingNotifications buffer.RingGrowing

   // requestedResyncPeriod is how frequently the listener wants a
   // full resync from the shared informer, but modified by two
   // adjustments.  One is imposing a lower bound,
   // `minimumResyncPeriod`.  The other is another lower bound, the
   // sharedIndexInformer's `resyncCheckPeriod`, that is imposed (a) only
   // in AddEventHandlerWithResyncPeriod invocations made after the
   // sharedIndexInformer starts and (b) only if the informer does
   // resyncs at all.
   requestedResyncPeriod time.Duration
   // resyncPeriod is the threshold that will be used in the logic
   // for this listener.  This value differs from
   // requestedResyncPeriod only when the sharedIndexInformer does
   // not do resyncs, in which case the value here is zero.  The
   // actual time between resyncs depends on when the
   // sharedProcessor's `shouldResync` function is invoked and when
   // the sharedIndexInformer processes `Sync` type Delta objects.
   resyncPeriod time.Duration
   // nextResync is the earliest time the listener should get a full resync
   nextResync time.Time
   // resyncLock guards access to resyncPeriod and nextResync
   resyncLock sync.Mutex
}
```

这里需要注意的是，processorListener 定义了两个 channel，从初始化方法可知，

```go
func newProcessListener(handler ResourceEventHandler, requestedResyncPeriod, resyncPeriod time.Duration, now time.Time, bufferSize int, hasSynced func() bool) *processorListener {
   ret := &processorListener{
      nextCh:                make(chan interface{}),
      addCh:                 make(chan interface{}),
      handler:               handler,
      syncTracker:           &synctrack.SingleFileTracker{UpstreamHasSynced: hasSynced},
      pendingNotifications:  *buffer.NewRingGrowing(bufferSize),
      requestedResyncPeriod: requestedResyncPeriod,
      resyncPeriod:          resyncPeriod,
   }

   ret.determineNextResync(now)

   return ret
}
```

`nextCh` `addCh` 都是非缓冲 channel，processorListener 启动了两个 goroutine 来处理它们。

### processorListener.run()

```go
func (p *processorListener) run() {
	// this call blocks until the channel is closed.  When a panic happens during the notification
	// we will catch it, **the offending item will be skipped!**, and after a short delay (one second)
	// the next notification will be attempted.  This is usually better than the alternative of never
	// delivering again.
	stopCh := make(chan struct{})
	wait.Until(func() {
		for next := range p.nextCh {
			switch notification := next.(type) {
			case updateNotification:
				p.handler.OnUpdate(notification.oldObj, notification.newObj)
			case addNotification:
				p.handler.OnAdd(notification.newObj, notification.isInInitialList)
				if notification.isInInitialList {
					p.syncTracker.Finished()
				}
			case deleteNotification:
				p.handler.OnDelete(notification.oldObj)
			default:
				utilruntime.HandleError(fmt.Errorf("unrecognized notification: %T", next))
			}
		}
		// the only way to get here is if the p.nextCh is empty and closed
		close(stopCh)
	}, 1*time.Second, stopCh)
}
```

run() 方法比较简单，从 `p.nextCh` 拿到数据，根据数据类型，转发给` p.handler` 的对应方法。

### processorListener.pop()

```go
func (p *processorListener) pop() {
	defer utilruntime.HandleCrash()
	defer close(p.nextCh) // Tell .run() to stop

	var nextCh chan<- interface{}
	var notification interface{}
	for {
		select {
		case nextCh <- notification:
			// Notification dispatched
			var ok bool
			notification, ok = p.pendingNotifications.ReadOne()
			if !ok { // Nothing to pop
				nextCh = nil // Disable this select case
			}
		case notificationToAdd, ok := <-p.addCh:
			if !ok {
				return
			}
			if notification == nil { // No notification to pop (and pendingNotifications is empty)
				// Optimize the case - skip adding to pendingNotifications
				notification = notificationToAdd
				nextCh = p.nextCh
			} else { // There is already a notification waiting to be dispatched
				p.pendingNotifications.WriteOne(notificationToAdd)
			}
		}
	}
}
```

pop()的逻辑比较复杂，参考：[k8s client-go informer中的processorlistener数据消费，缓存的分析](https://zhuanlan.zhihu.com/p/371394075) 

{{< admonition abstract >}}

数据要么放入消费通道(nextCh)，要么放入缓存(pendingNotifications)，缓存中的数据最终也会放入消费通道。这是一种巧妙的设计方式。

{{< /admonition >}}



## 小结


















