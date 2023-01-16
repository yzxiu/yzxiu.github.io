# client-go解析(3) - cache.Reflector


## 概述

`cache.Reflector` 可以说是k8s最重要的组件，它串联起k8s的整个流程。

在服务端(`apiserver`) ，使用 reflector 向 etcd 获取资源数据。

在连接端(`informer`、`kubelet` ...)，使用 reflector 向 apiserver 获取资源数据。

k8s的整个逻辑流程中，所有这些获取资源数据相关的操作，都封装在 reflector 里面，可以看出 reflector 对于理解 k8s 的重要性。



## 定义

```go
// Reflector watches a specified resource and causes all changes to be reflected in the given store.
type Reflector struct {
   // name identifies this reflector. By default it will be a file:line if possible.
   name string

   // The name of the type we expect to place in the store. The name
   // will be the stringification of expectedGVK if provided, and the
   // stringification of expectedType otherwise. It is for display
   // only, and should not be used for parsing or comparison.
   typeDescription string
   // An example object of the type we expect to place in the store.
   // Only the type needs to be right, except that when that is
   // `unstructured.Unstructured` the object's `"apiVersion"` and
   // `"kind"` must also be right.
   expectedType reflect.Type
   // The GVK of the object we expect to place in the store if unstructured.
   expectedGVK *schema.GroupVersionKind
   // The destination to sync up with the watch source
   store Store
   // listerWatcher is used to perform lists and watches.
   listerWatcher ListerWatcher

   // backoff manages backoff of ListWatch
   backoffManager wait.BackoffManager
   // initConnBackoffManager manages backoff the initial connection with the Watch call of ListAndWatch.
   initConnBackoffManager wait.BackoffManager
   // MaxInternalErrorRetryDuration defines how long we should retry internal errors returned by watch.
   MaxInternalErrorRetryDuration time.Duration

   resyncPeriod time.Duration
   // ShouldResync is invoked periodically and whenever it returns `true` the Store's Resync operation is invoked
   ShouldResync func() bool
   // clock allows tests to manipulate time
   clock clock.Clock
   // paginatedResult defines whether pagination should be forced for list calls.
   // It is set based on the result of the initial list call.
   paginatedResult bool
   // lastSyncResourceVersion is the resource version token last
   // observed when doing a sync with the underlying store
   // it is thread safe, but not synchronized with the underlying store
   lastSyncResourceVersion string
   // isLastSyncResourceVersionUnavailable is true if the previous list or watch request with
   // lastSyncResourceVersion failed with an "expired" or "too large resource version" error.
   isLastSyncResourceVersionUnavailable bool
   // lastSyncResourceVersionMutex guards read/write access to lastSyncResourceVersion
   lastSyncResourceVersionMutex sync.RWMutex
   // WatchListPageSize is the requested chunk size of initial and resync watch lists.
   // If unset, for consistent reads (RV="") or reads that opt-into arbitrarily old data
   // (RV="0") it will default to pager.PageSize, for the rest (RV != "" && RV != "0")
   // it will turn off pagination to allow serving them from watch cache.
   // NOTE: It should be used carefully as paginated lists are always served directly from
   // etcd, which is significantly less efficient and may lead to serious performance and
   // scalability problems.
   WatchListPageSize int64
   // Called whenever the ListAndWatch drops the connection with an error.
   watchErrorHandler WatchErrorHandler
}
```





## 使用

看一下 reflector 在k8s中的几处应用

### apiserver

apiserver 中，Reflector 使用 watchCache 作为 store，向 etcd 中获取数据

```go
staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go

func NewCacherFromConfig(config Config) (*Cacher, error) {
	...
	...
	cacher := &Cacher{
		...
		...
	}
	...
	...
	watchCache := newWatchCache(
		config.KeyFunc, cacher.processEvent, config.GetAttrsFunc, config.Versioner, config.Indexers, config.Clock, config.GroupResource)
	listerWatcher := NewCacherListerWatcher(config.Storage, config.ResourcePrefix, config.NewListFunc)
	reflectorName := "storage/cacher.go:" + config.ResourcePrefix

	reflector := cache.NewNamedReflector(reflectorName, listerWatcher, obj, watchCache, 0)
	...
	...
	cacher.watchCache = watchCache
	cacher.reflector = reflector
	...
	...
	return cacher, nil
}
```

### kubelet

kubelet 中，Reflector 使用 UndeltaStore 作为 store，向 apiserver 中获取数据

```gO
/home/xiu/Github/kubernetes/pkg/kubelet/config/apiserver.go

// newSourceApiserverFromLW holds creates a config source that watches and pulls from the apiserver.
func newSourceApiserverFromLW(lw cache.ListerWatcher, updates chan<- interface{}) {
	...
    ///
	r := cache.NewReflector(lw, &v1.Pod{}, cache.NewUndeltaStore(send, cache.MetaNamespaceKeyFunc), 0)
	go r.Run(wait.NeverStop)
}

```

### informer(cache.controller)

informer(cache.controller) 中, Reflector 使用 DeltaFIFO 作为 store，向 apiserver 中获取数据

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

   wg.StartWithChannel(stopCh, r.Run)

   wait.Until(c.processLoop, time.Second, stopCh)
   wg.Wait()
}
```



## Run()

```go
// Run repeatedly uses the reflector's ListAndWatch to fetch all the
// objects and subsequent deltas.
// Run will exit when stopCh is closed.
func (r *Reflector) Run(stopCh <-chan struct{}) {
   klog.V(3).Infof("Starting reflector %s (%s) from %s", r.typeDescription, r.resyncPeriod, r.name)
   wait.BackoffUntil(func() {
      if err := r.ListAndWatch(stopCh); err != nil {
         r.watchErrorHandler(r, err)
      }
   }, r.backoffManager, true, stopCh)
   klog.V(3).Infof("Stopping reflector %s (%s) from %s", r.typeDescription, r.resyncPeriod, r.name)
}
```

在 `wait.BackoffUntil(func(){}, , r.backoffManager, true, stopCh)` 调用 `ListAndWatch`.

先了解一下 `BackoffUntil` 的运行规则:

```go
func TestBackoffUntil(t *testing.T) {
	stopCh := make(chan struct{})
	realClock := &clock.RealClock{}
	backoffManager := NewExponentialBackoffManager(800*time.Millisecond, 30*time.Second, 2*time.Minute, 2.0, 1.0, realClock)
	i := 1
	t1 := time.Now()
	BackoffUntil(func() {
		t.Logf("第 %d 次运行, 时间差: %v", i, time.Now().Sub(t1))
		time.Sleep(time.Second)
		t1 = time.Now()
		i++
		return
	}, backoffManager, true, stopCh)
}

=== RUN   TestBackoffUntil
    wait_test.go:41: 第 1 次运行, 时间差: 185ns
    wait_test.go:41: 第 2 次运行, 时间差: 1.284438667s
    wait_test.go:41: 第 3 次运行, 时间差: 3.105450176s
    wait_test.go:41: 第 4 次运行, 时间差: 5.330910046s
    wait_test.go:41: 第 5 次运行, 时间差: 9.201774039s
    wait_test.go:41: 第 6 次运行, 时间差: 18.253303837s
    wait_test.go:41: 第 7 次运行, 时间差: 43.22295292s
    wait_test.go:41: 第 8 次运行, 时间差: 31.998912477s
    wait_test.go:41: 第 9 次运行, 时间差: 34.72983897s
    wait_test.go:41: 第 10 次运行, 时间差: 32.941101108s
    wait_test.go:41: 第 11 次运行, 时间差: 39.06361719s
    wait_test.go:41: 第 12 次运行, 时间差: 45.498957001s
    wait_test.go:41: 第 13 次运行, 时间差: 54.462938656s
    wait_test.go:41: 第 14 次运行, 时间差: 36.463043438s
    wait_test.go:41: 第 15 次运行, 时间差: 41.460279724s
```

所以,当 `ListAndWatch` 错误退出的时候, reflector会根据配置, 在一段之间后重新执行.



## ListAndWatch()

```go
// ListAndWatch first lists all items and get the resource version at the moment of call,
// and then use the resource version to watch.
// It returns error if ListAndWatch didn't even try to initialize watch.
func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error {
	...
	...
    // 获取资源数据,然后调用 cache.Store.Replace()
    // 设置 ResourceVersion
	err := r.list(stopCh)
	...
	...
	for {
		...
		...
		w, err := r.listerWatcher.Watch(options)
		...
		....
        // 获取增量数据
		err = watchHandler(start, w, r.store, r.expectedType, r.expectedGVK, r.name, r.typeDescription, r.setLastSyncResourceVersion, r.clock, resyncerrc, stopCh)
		...
		...
	}
}
```

### r.list(stopCh)

```go
// list simply lists all items and records a resource version obtained from the server at the moment of the call.
// the resource version can be used for further progress notification (aka. watch).
func (r *Reflector) list(stopCh <-chan struct{}) error {
   ...
   ...
   var list runtime.Object
   var paginatedResult bool
   var err error
   listCh := make(chan struct{}, 1)
   panicCh := make(chan interface{}, 1)
   go func() {
      ...
      ...
      list, paginatedResult, err = pager.List(context.Background(), options)
      ...
      ...
   }()
   ...
   ...
   listMetaInterface, err := meta.ListAccessor(list)
   ...
   resourceVersion = listMetaInterface.GetResourceVersion()
   ...
   items, err := meta.ExtractList(list)
   ...
   ...
   if err := r.syncWith(items, resourceVersion); err != nil {
      return fmt.Errorf("unable to sync list result: %v", err)
   }
   ...
   r.setLastSyncResourceVersion(resourceVersion)
   ...
   return nil
}
```

```go
// syncWith replaces the store's items with the given list.
func (r *Reflector) syncWith(items []runtime.Object, resourceVersion string) error {
   found := make([]interface{}, 0, len(items))
   for _, item := range items {
      found = append(found, item)
   }
   return r.store.Replace(found, resourceVersion)
}
```

### watchHandler()

```go
// watchHandler watches w and sets setLastSyncResourceVersion
func watchHandler(start time.Time,
   w watch.Interface,
   store Store,
   expectedType reflect.Type,
   expectedGVK *schema.GroupVersionKind,
   name string,
   expectedTypeName string,
   setLastSyncResourceVersion func(string),
   clock clock.Clock,
   errc chan error,
   stopCh <-chan struct{},
) error {
   eventCount := 0

   // Stopping the watcher should be idempotent and if we return from this function there's no way
   // we're coming back in with the same watch interface.
   defer w.Stop()

loop:
   for {
      select {
      case <-stopCh:
         return errorStopRequested
      case err := <-errc:
         return err
      case event, ok := <-w.ResultChan():
         ...
         ...
         switch event.Type {
         case watch.Added:
            err := store.Add(event.Object)
            if err != nil {
               utilruntime.HandleError(fmt.Errorf("%s: unable to add watch event object (%#v) to store: %v", name, event.Object, err))
            }
         case watch.Modified:
            err := store.Update(event.Object)
            if err != nil {
               utilruntime.HandleError(fmt.Errorf("%s: unable to update watch event object (%#v) to store: %v", name, event.Object, err))
            }
         case watch.Deleted:
            // TODO: Will any consumers need access to the "last known
            // state", which is passed in event.Object? If so, may need
            // to change this.
            err := store.Delete(event.Object)
            if err != nil {
               utilruntime.HandleError(fmt.Errorf("%s: unable to delete watch event object (%#v) from store: %v", name, event.Object, err))
            }
         case watch.Bookmark:
            // A `Bookmark` means watch has synced here, just update the resourceVersion
         default:
            utilruntime.HandleError(fmt.Errorf("%s: unable to understand watch event %#v", name, event))
         }
         ...
         ...
      }
   }
   ...
   ...
   return nil
}
```



## 小结

Relector 在 Run 起来之后, 通过传入的 lw, 先获取指定资源, 然后存入对应的 store 中.

{{< admonition info >}}
具体到 `informer` 中的reflector, list 会产生一系列 `Sync/Replace` 的 `Delta` 数据, 然后通过watch, 根据 event.Type, 产生  `Added / Updated / Deleted` 的 `Delta` 数据. 

reflector `Run()` 起来之后,就作为`生产者`产生 `Delta` 数据, 至于怎么`消费(Pop)`这些 `Delta` 数据, 可以关注cache.controller 怎么调用Pop()
{{< /admonition >}}












