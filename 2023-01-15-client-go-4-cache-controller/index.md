# client-go解析(4) - cache.Controller


## 概述

本文中提到的 reflector, 特指 informer 中的 reflector, 即传入的 cache.Store 为 DeltaFIFO



## Controller

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




