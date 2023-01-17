# client-go解析(5) - informer


## 概述

NewSharedIndexInformer() 是出镜率非常高的函数，最终创建了一个 sharedIndexInformer。

我们这里暂且将 informer 等同于 sharedIndexInformer。



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
   wg.StartWithChannel(processorStopCh, s.processor.run)

   defer func() {
      s.startedLock.Lock()
      defer s.startedLock.Unlock()
      s.stopped = true // Don't want any new listeners
   }()
   s.controller.Run(stopCh)
}
```




