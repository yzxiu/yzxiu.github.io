# client-go解析(4) - cache.Controller


## 概述

本文中提到的 reflector, 特指 informer 中的 reflector, 即传入的 cache.Store 为 DeltaFIFO。



## 接口

### cache.Controller

```go
// Controller is a low-level controller that is parameterized by a
// Config and used in sharedIndexInformer.
type Controller interface {
   // Run does two things.  One is to construct and run a Reflector
   // to pump objects/notifications from the Config's ListerWatcher
   // to the Config's Queue and possibly invoke the occasional Resync
   // on that Queue.  The other is to repeatedly Pop from the Queue
   // and process with the Config's ProcessFunc.  Both of these
   // continue until `stopCh` is closed.
   // 1, 构建并启动Reflector
   // 2, Pop from the Queue and process with the Config's ProcessFunc
   Run(stopCh <-chan struct{})

   // HasSynced delegates to the Config's Queue
   HasSynced() bool

   // LastSyncResourceVersion delegates to the Reflector when there
   // is one, otherwise returns the empty string
   LastSyncResourceVersion() string
}
```

```go
// `*controller` implements Controller
type controller struct {
   config         Config
   reflector      *Reflector
   reflectorMutex sync.RWMutex
   clock          clock.Clock
}
```



## Run()

```go
// Run begins processing items, and will continue until a value is sent down stopCh or it is closed.
// It's an error to call Run more than once.
// Run blocks; call via go.
func (c *controller) Run(stopCh <-chan struct{}) {
   defer utilruntime.HandleCrash()
   go func() {
      <-stopCh
      c.config.Queue.Close()
   }()
   // 构建Reflector
   r := NewReflectorWithOptions(
      c.config.ListerWatcher,
      c.config.ObjectType,
      c.config.Queue,
      ReflectorOptions{
         ResyncPeriod:    c.config.FullResyncPeriod,
         TypeDescription: c.config.ObjectDescription,
      },
   )
   r.ShouldResync = c.config.ShouldResync
   r.WatchListPageSize = c.config.WatchListPageSize
   r.clock = c.clock
   if c.config.WatchErrorHandler != nil {
      r.watchErrorHandler = c.config.WatchErrorHandler
   }

   c.reflectorMutex.Lock()
   c.reflector = r
   c.reflectorMutex.Unlock()

   var wg wait.Group
   // run reflector
   wg.StartWithChannel(stopCh, r.Run)
   // 启动消费者
   wait.Until(c.processLoop, time.Second, stopCh)
   wg.Wait()
}
```

```go
func (c *controller) processLoop() {
   for {
      obj, err := c.config.Queue.Pop(PopProcessFunc(c.config.Process))
      if err != nil {
         if err == ErrFIFOClosed {
            return
         }
         if c.config.RetryOnError {
            // This is the safe way to re-enqueue.
            c.config.Queue.AddIfNotPresent(obj)
         }
      }
   }
}
```

可以看到, reflector 在 processLoop() 中进行消费，最终交给 c.config.Process 处理。

```go
func newInformer(
	lw ListerWatcher,
	objType runtime.Object,
	resyncPeriod time.Duration,
	h ResourceEventHandler,
	clientState Store,
	transformer TransformFunc,
) Controller {
	// This will hold incoming changes. Note how we pass clientState in as a
	// KeyLister, that way resync operations will result in the correct set
	// of update/delete deltas.
	fifo := NewDeltaFIFOWithOptions(DeltaFIFOOptions{
		KnownObjects:          clientState,
		EmitDeltaTypeReplaced: true,
	})

	cfg := &Config{
		Queue:            fifo,
		ListerWatcher:    lw,
		ObjectType:       objType,
		FullResyncPeriod: resyncPeriod,
		RetryOnError:     false,

		Process: func(obj interface{}, isInInitialList bool) error {
			if deltas, ok := obj.(Deltas); ok {
				return processDeltas(h, clientState, transformer, deltas, isInInitialList)
			}
			return errors.New("object given as Process argument is not Deltas")
		},
	}
	return New(cfg)
}

```

```go
// Multiplexes updates in the form of a list of Deltas into a Store, and informs
// a given handler of events OnUpdate, OnAdd, OnDelete
func processDeltas(
   // Object which receives event notifications from the given deltas
   handler ResourceEventHandler,
   clientState Store,
   transformer TransformFunc,
   deltas Deltas,
   isInInitialList bool,
) error {
   // from oldest to newest
   for _, d := range deltas {
      obj := d.Object
      if transformer != nil {
         var err error
         obj, err = transformer(obj)
         if err != nil {
            return err
         }
      }

      switch d.Type {
      case Sync, Replaced, Added, Updated:
         if old, exists, err := clientState.Get(obj); err == nil && exists {
            if err := clientState.Update(obj); err != nil {
               return err
            }
            handler.OnUpdate(old, obj)
         } else {
            if err := clientState.Add(obj); err != nil {
               return err
            }
            handler.OnAdd(obj, isInInitialList)
         }
      case Deleted:
         if err := clientState.Delete(obj); err != nil {
            return err
         }
         handler.OnDelete(obj)
      }
   }
   return nil
}
```

无论是什么类型的 Delta ，都是先更新 clientState(也就是DeltaFIFO中的`knownObjects`). 然后再交给 `handler` 去处理。

```go
type ResourceEventHandler interface {
   OnAdd(obj interface{})
   OnUpdate(oldObj, newObj interface{})
   OnDelete(obj interface{})
}
```

## 小结

cache.controller 逻辑比较简单。

在 Run 方法中初始化并启动 Reflector，并启动消费者进行消费。

具体的消费方法，是 `cache.Config.Process`。

