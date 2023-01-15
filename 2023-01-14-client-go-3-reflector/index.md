# client-go解析(3) - cache.Reflector


## 概述

`cache.Reflector` 可以说是k8s最重要的组件，它串联起k8s的整个流程。

在服务端(apiserver) ，使用 reflector 向 etcd 获取资源数据。

在连接端(informer、kubelet)，使用 reflector 向 apiserver 获取资源数据。

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


