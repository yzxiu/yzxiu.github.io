# client-go解析(2) - cache.Store


## 概述

如果把`k8s`当成`资源管理系统`, 那`cache.Store`无疑是`最核心`的接口, 用于`缓存`,`存储`资源 。

reflector  依赖于 cache.Store 的实现做存储，根据不同的实现有不同的功能。所以，正确的理解cache.Store的不同实现，是理解 reflector 的关键。

接口定义如下：

```go
//  client-go/tools/cache/store.go
type Store interface {
	Add(obj interface{}) error
	Update(obj interface{}) error
	Delete(obj interface{}) error
	List() []interface{}
	ListKeys() []string
	Get(obj interface{}) (item interface{}, exists bool, err error)
	GetByKey(key string) (item interface{}, exists bool, err error)
	Replace([]interface{}, string) error
	Resync() error
}
```

先看一下cache.Store的接口与实现类:

client-go中:

![image-20230113094558703](https://raw.githubusercontent.com/yzxiu/images/blog/2023-01/20230113-094559.png "client-go")



k8s中:

![image-20230113095542135](https://raw.githubusercontent.com/yzxiu/images/blog/2023-01/20230113-095543.png "k8s")

{{< admonition >}}
在k8s中添加了两个实现类:

cacheStore 和 watchCache,分别用于kubelet和apiserver,后续再作分析

{{< /admonition >}}



接口 Indexer 和 Queue,分别在 cache.Store 的基础上,拓展了一些功能

indexer的实现类只有一个

![image-20230113101036931](https://raw.githubusercontent.com/yzxiu/images/blog/2023-01/20230113-101037.png "indexer")



Queue的实现类有2个

![image-20230113101114329](https://raw.githubusercontent.com/yzxiu/images/blog/2023-01/20230113-101114.png "Queue")

{{< admonition >}}
在client-go中,还存在一些 workqueue,在client-go/util/workqueue 包中,大致如下:

![image-20230113102822621](https://raw.githubusercontent.com/yzxiu/images/blog/2023-01/20230113-102823.png "workqueue")

这里注意区分,不要混淆

{{< /admonition >}}

除此之外, 直接实现cache.Store的还有 ExpirationCache, UndeltaStore, cacheStore, watchCache, 

UndeltaStore 是用于kubelet 获取pod数据用到的，watchCache 是apiserver缓存etcd数据据用到的。



## cache.Store的相关实现

由上面的分析, 这里将 cache.Store的实现分为3组.



### 基础 (cache)

在store.go文件中，就包含了 cache.Store的默认实现 cache，同时也是接口Indexer的实现类。

Indexer 接口主要是在 Store 接口的基础上拓展了对象的检索功能：

```go
type Indexer interface {
   Store
   Index(indexName string, obj interface{}) ([]interface{}, error) // 根据索引名和给定的对象返回符合条件的所有对象
   IndexKeys(indexName, indexedValue string) ([]string, error)     // 根据索引名和索引值返回符合条件的所有对象的 key
   ListIndexFuncValues(indexName string) []string                  // 列出索引函数计算出来的所有索引值
   ByIndex(indexName, indexedValue string) ([]interface{}, error)  // 根据索引名和索引值返回符合条件的所有对象
   GetIndexers() Indexers                     // 获取所有的 Indexers，对应 map[string]IndexFunc 类型
   AddIndexers(newIndexers Indexers) error    // 这个方法要在数据加入存储前调用，添加更多的索引方法，默认只通过 namespace 检索
}
```

Indexer 的默认实现是 cache：

```go
type cache struct {
   cacheStorage ThreadSafeStore
   keyFunc KeyFunc
}
```

相关构造器:

```go
// NewStore returns a Store implemented simply with a map and a lock.
func NewStore(keyFunc KeyFunc) Store {
   return &cache{
      cacheStorage: NewThreadSafeStore(Indexers{}, Indices{}),
      keyFunc:      keyFunc,
   }
}

// NewIndexer returns an Indexer implemented simply with a map and a lock.
func NewIndexer(keyFunc KeyFunc, indexers Indexers) Indexer {
   return &cache{
      cacheStorage: NewThreadSafeStore(indexers, Indices{}),
      keyFunc:      keyFunc,
   }
}
```

由注释可知，基础的Store，由一个map和lock实现。

根据是否传入 `indexers` 索引函数，判断是否开启索引功能。



可以从测试用例中,了解他们的用法

```go
// 数据类型
type testStoreObject struct {
   id  string
   val string
}
```

```go
// 获取key的函数,使用 id 作为Key
func testStoreKeyFunc(obj interface{}) (string, error) {
   return obj.(testStoreObject).id, nil
}
```

#### Store

```go
func TestCache(t *testing.T) {
   doTestStore(t, NewStore(testStoreKeyFunc))
}
```

Store 添加一个元素之后,

![image-20230113113522685](https://raw.githubusercontent.com/yzxiu/images/blog/2023-01/20230113-113523.png "Store")

Store实现了基本的crud功能, index 为空.

#### Indexer

```go
func TestIndex(t *testing.T) {
   doTestIndex(t, NewIndexer(testStoreKeyFunc, testStoreIndexers()))
}
```

与构造Store相比, 构造Indexer多传进了一个 Indexers map,如下:

```go
func testStoreIndexers() Indexers {
	indexers := Indexers{}
	indexers["by_val"] = testStoreIndexFunc
	return indexers
}
func testStoreIndexFunc(obj interface{}) ([]string, error) {
	return []string{obj.(testStoreObject).val}, nil
}
```

该函数定义了一个索引 "by_val", 具体是使用对象的 val 作为索引值.

添加元素后, indexer内容如下:

```go
indexer.Add(mkObj("a", "b"))
indexer.Add(mkObj("c", "b"))
indexer.Add(mkObj("e", "f"))
indexer.Add(mkObj("g", "h"))
```

![image-20230113114154787](https://raw.githubusercontent.com/yzxiu/images/blog/2023-01/20230113-114155.png "Indexer")

与 Store 相比, index 不为空.

`indexers` 存储了索引函数

`indices` 存储了具体的索引数据



### 队列

队列接口定义如下

```go
type Queue interface {
   Store
    
   Pop(PopProcessFunc) (interface{}, error)
   AddIfNotPresent(interface{}) error
   HasSynced() bool
   Close()
}
```

Queue是在**Store基础上扩展了Pop接口可以让对象有序的弹出**



#### FIFO

```go
// FIFO is a Queue in which (a) each accumulator is simply the most
// recently provided object and (b) the collection of keys to process
// is a FIFO.  The accumulators all start out empty, and deleting an
// object from its accumulator empties the accumulator.  The Resync
// operation is a no-op.
//
// Thus: if multiple adds/updates of a single object happen while that
// object's key is in the queue before it has been processed then it
// will only be processed once, and when it is processed the most
// recent version will be processed. This can't be done with a channel
//
// FIFO solves this use case:
//   - You want to process every object (exactly) once.
//   - You want to process the most recent version of the object when you process it.
//   - You do not want to process deleted objects, they should be removed from the queue.
//   - You do not want to periodically reprocess objects.
//
// Compare with DeltaFIFO for other use cases.
type FIFO struct {
   lock sync.RWMutex
   cond sync.Cond
   // We depend on the property that every key in `items` is also in `queue`
   items map[string]interface{}
   queue []string

   // populated is true if the first batch of items inserted by Replace() has been populated
   // or Delete/Add/Update was called first.
   populated bool
   // initialPopulationCount is the number of items inserted by the first call of Replace()
   initialPopulationCount int

   // keyFunc is used to make the key used for queued item insertion and retrieval, and
   // should be deterministic.
   keyFunc KeyFunc

   // Indication the queue is closed.
   // Used to indicate a queue is closed so a control loop can exit when a queue is empty.
   // Currently, not used to gate any of CRUD operations.
   closed bool
}
```

构造函数:

```go
// NewFIFO returns a Store which can be used to queue up items to
// process.
func NewFIFO(keyFunc KeyFunc) *FIFO {
	f := &FIFO{
		items:   map[string]interface{}{},
		queue:   []string{},
		keyFunc: keyFunc,
	}
	f.cond.L = &f.lock
	return f
}

```

关于FIFO, 需要注意的是,如果pop处理失败,会重新放入队列

FIFO在client-go/k8s中并不多见,重点是下面的 DeltaFIFO

#### DeltaFIFO

定义:

```go
// DeltaFIFO is a producer-consumer queue, where a Reflector is
// intended to be the producer, and the consumer is whatever calls
// the Pop() method.
// 将 DeltaFIFO 当成一个生产-消费队列
type DeltaFIFO struct {
   lock sync.RWMutex
   cond sync.Cond
   items map[string]Deltas
   queue []string
   populated bool
   initialPopulationCount int
   keyFunc KeyFunc
   knownObjects KeyListerGetter
   closed bool
   emitDeltaTypeReplaced bool
}
```

##### Add() / Update() / Delete()

add() / update() / delete() 可以视为生产者，生产类型为 Added / Updated / Deleted 的 `Delta` 数据

```go
func (f *DeltaFIFO) Add(obj interface{}) error {
	f.lock.Lock()
	defer f.lock.Unlock()
	f.populated = true
	return f.queueActionLocked(Added, obj)
}
func (f *DeltaFIFO) Update(obj interface{}) error {
	f.lock.Lock()
	defer f.lock.Unlock()
	f.populated = true
	return f.queueActionLocked(Updated, obj)
}
func (f *DeltaFIFO) Delete(obj interface{}) error {
	id, err := f.KeyOf(obj)
	if err != nil {
		return KeyError{obj, err}
	}
	f.lock.Lock()
	defer f.lock.Unlock()
	f.populated = true
	if f.knownObjects == nil {
		if _, exists := f.items[id]; !exists {
			return nil
		}
	} else {
		_, exists, err := f.knownObjects.GetByKey(id)
		_, itemsExist := f.items[id]
		if err == nil && !exists && !itemsExist {
			return nil
		}
	}
	return f.queueActionLocked(Deleted, obj)
}
```

```go
func (f *DeltaFIFO) queueActionLocked(actionType DeltaType, obj interface{}) error {
   id, err := f.KeyOf(obj)
   if err != nil {
      return KeyError{obj, err}
   }
   oldDeltas := f.items[id]
   newDeltas := append(oldDeltas, Delta{actionType, obj})
   newDeltas = dedupDeltas(newDeltas)

   if len(newDeltas) > 0 {
      if _, exists := f.items[id]; !exists {
         f.queue = append(f.queue, id)
      }
      f.items[id] = newDeltas
      f.cond.Broadcast()
   } else {
      if oldDeltas == nil {
         klog.Errorf("Impossible dedupDeltas for id=%q: oldDeltas=%#+v, obj=%#+v; ignoring", id, oldDeltas, obj)
         return nil
      }
      klog.Errorf("Impossible dedupDeltas for id=%q: oldDeltas=%#+v, obj=%#+v; breaking invariant by storing empty Deltas", id, oldDeltas, obj)
      f.items[id] = newDeltas
      return fmt.Errorf("Impossible dedupDeltas for id=%q: oldDeltas=%#+v, obj=%#+v; broke DeltaFIFO invariant by storing empty Deltas", id, oldDeltas, obj)
   }
   return nil
}
```

```go
f := NewDeltaFIFOWithOptions(DeltaFIFOOptions{KeyFunction: testFifoObjectKeyFunc})
f.Add(mkFifoObj("foo", 10))
f.Update(mkFifoObj("foo", 12))
f.Delete(mkFifoObj("foo", 15))
```

![image-20230113165311586](https://raw.githubusercontent.com/yzxiu/images/blog/2023-01/20230113-165312.png  "Add() / Update() / Delete() ")

##### Replace()

 Replace() 可以视为生产者，生产类型为 Sync / Replaced 的 Delta 数据

```go
// Replace atomically does two things: (1) it adds the given objects
// using the Sync or Replace DeltaType and then (2) it does some deletions.
// 1, 为给定的 objects 添加一个 Sync / Replace 的Delta数据
// 2, 添加一些 Deleted 数据
// 
// In particular: for every pre-existing key K that is not the key of
// an object in `list` there is the effect of
// `Delete(DeletedFinalStateUnknown{K, O})` where O is current object
// of K. 
// 对于所有存在于 list 中，但不存在于 pre-existing 中的Key K，
// 添加一个 Deleted 的 DeletedFinalStateUnknown 数据。
// 对于 pre-existing 和评定，和 K 对应 Object，在下面有详细说明
// 
// If `f.knownObjects == nil` then the pre-existing keys are
// those in `f.items` and the current object of K is the `.Newest()`
// of the Deltas associated with K.  Otherwise the pre-existing keys
// are those listed by `f.knownObjects` and the current object of K is
// what `f.knownObjects.GetByKey(K)` returns.
// 1, 当 f.knownObjects == nil
// pre-existing 为 f.items, 对应的 object 为 Newest()(即Deltas数组中的最后一个)
// 2, 当 f.knownObjects != nil,
// pre-existing 为 f.knownObjects.ListKeys(), 对应的 object 为 f.knownObjects.GetByKey(k)
func (f *DeltaFIFO) Replace(list []interface{}, _ string) error {
	f.lock.Lock()
	defer f.lock.Unlock()
	keys := make(sets.String, len(list))

	// keep backwards compat for old clients
	action := Sync
	if f.emitDeltaTypeReplaced {
		action = Replaced
	}

	// Add Sync/Replaced action for each new item.
	for _, item := range list {
		key, err := f.KeyOf(item)
		if err != nil {
			return KeyError{item, err}
		}
		keys.Insert(key)
		if err := f.queueActionLocked(action, item); err != nil {
			return fmt.Errorf("couldn't enqueue object: %v", err)
		}
	}

	if f.knownObjects == nil {
		// Do deletion detection against our own list.
		queuedDeletions := 0
		for k, oldItem := range f.items {
			if keys.Has(k) {
				continue
			}
			// Delete pre-existing items not in the new list.
			// This could happen if watch deletion event was missed while
			// disconnected from apiserver.
			var deletedObj interface{}
			if n := oldItem.Newest(); n != nil {
				deletedObj = n.Object
			}
			queuedDeletions++
			if err := f.queueActionLocked(Deleted, DeletedFinalStateUnknown{k, deletedObj}); err != nil {
				return err
			}
		}

		if !f.populated {
			f.populated = true
			// While there shouldn't be any queued deletions in the initial
			// population of the queue, it's better to be on the safe side.
			f.initialPopulationCount = keys.Len() + queuedDeletions
		}

		return nil
	}

	// Detect deletions not already in the queue.
	knownKeys := f.knownObjects.ListKeys()
	queuedDeletions := 0
	for _, k := range knownKeys {
		if keys.Has(k) {
			continue
		}

		deletedObj, exists, err := f.knownObjects.GetByKey(k)
		if err != nil {
			deletedObj = nil
			klog.Errorf("Unexpected error %v during lookup of key %v, placing DeleteFinalStateUnknown marker without object", err, k)
		} else if !exists {
			deletedObj = nil
			klog.Infof("Key %v does not exist in known objects store, placing DeleteFinalStateUnknown marker without object", k)
		}
		queuedDeletions++
		if err := f.queueActionLocked(Deleted, DeletedFinalStateUnknown{k, deletedObj}); err != nil {
			return err
		}
	}

	if !f.populated {
		f.populated = true
		f.initialPopulationCount = keys.Len() + queuedDeletions
	}

	return nil
}
```

Replace() 的逻辑在注释中已经说的非常详细。



##### Resync()

 Resync() 可以视为生产者，生产类型为 Sync 的 Delta 数据

```go
// Resync adds, with a Sync type of Delta, every object listed by
// `f.knownObjects` whose key is not already queued for processing.
// If `f.knownObjects` is `nil` then Resync does nothing.
func (f *DeltaFIFO) Resync() error {
   f.lock.Lock()
   defer f.lock.Unlock()

   if f.knownObjects == nil {
      return nil
   }

   keys := f.knownObjects.ListKeys()
   for _, k := range keys {
      if err := f.syncKeyLocked(k); err != nil {
         return err
      }
   }
   return nil
}
```

```go
func (f *DeltaFIFO) syncKeyLocked(key string) error {
   obj, exists, err := f.knownObjects.GetByKey(key)
   if err != nil {
      klog.Errorf("Unexpected error %v during lookup of key %v, unable to queue object for sync", err, key)
      return nil
   } else if !exists {
      klog.Infof("Key %v does not exist in known objects store, unable to queue object for sync", key)
      return nil
   }

   // If we are doing Resync() and there is already an event queued for that object,
   // we ignore the Resync for it. This is to avoid the race, in which the resync
   // comes with the previous value of object (since queueing an event for the object
   // doesn't trigger changing the underlying store <knownObjects>.
   // 对于 f.items 中已经存在的 Object，不添加 Sync Delta数据。
   id, err := f.KeyOf(obj)
   if err != nil {
      return KeyError{obj, err}
   }
   if len(f.items[id]) > 0 {
      return nil
   }

   if err := f.queueActionLocked(Sync, obj); err != nil {
      return fmt.Errorf("couldn't queue object: %v", err)
   }
   return nil
}
```



##### Pop()

Pop() 可以视为消费者

```go
// Pop blocks until the queue has some items, and then returns one.  If
// multiple items are ready, they are returned in the order in which they were
// added/updated. The item is removed from the queue (and the store) before it
// is returned, so if you don't successfully process it, you need to add it back
// with AddIfNotPresent().
// process function is called under lock, so it is safe to update data structures
// in it that need to be in sync with the queue (e.g. knownKeys). The PopProcessFunc
// may return an instance of ErrRequeue with a nested error to indicate the current
// item should be requeued (equivalent to calling AddIfNotPresent under the lock).
// process should avoid expensive I/O operation so that other queue operations, i.e.
// Add() and Get(), won't be blocked for too long.
// process 函数是在锁定的情况下调用的，因此可以安全地更新其中需要与队列同步的数据结构（例如 knownKeys）。 PopProcessFunc 可能会返回一个带有嵌套错误的 ErrRequeue 实例，以指示当前项目应该重新排队（相当于在锁下调用 AddIfNotPresent）。进程应避免昂贵的 IO 操作，以便其他队列操作，即 Add() 和 Get() 不会被阻塞太久。
//
// Pop returns a 'Deltas', which has a complete list of all the things
// that happened to the object (deltas) while it was sitting in the queue.
// Pop 返回一个“Deltas”，其中包含对象（deltas）在队列中时发生的所有事情的完整列表。
func (f *DeltaFIFO) Pop(process PopProcessFunc) (interface{}, error) {
   f.lock.Lock()
   defer f.lock.Unlock()
   for {
      for len(f.queue) == 0 {
         // When the queue is empty, invocation of Pop() is blocked until new item is enqueued.
         // When Close() is called, the f.closed is set and the condition is broadcasted.
         // Which causes this loop to continue and return from the Pop().
         if f.closed {
            return nil, ErrFIFOClosed
         }

         f.cond.Wait()
      }
      isInInitialList := !f.hasSynced_locked()
      id := f.queue[0]
      f.queue = f.queue[1:]
      depth := len(f.queue)
      if f.initialPopulationCount > 0 {
         f.initialPopulationCount--
      }
      item, ok := f.items[id]
      if !ok {
         // This should never happen
         klog.Errorf("Inconceivable! %q was in f.queue but not f.items; ignoring.", id)
         continue
      }
      delete(f.items, id)
      // Only log traces if the queue depth is greater than 10 and it takes more than
      // 100 milliseconds to process one item from the queue.
      // Queue depth never goes high because processing an item is locking the queue,
      // and new items can't be added until processing finish.
      // https://github.com/kubernetes/kubernetes/issues/103789
      if depth > 10 {
         trace := utiltrace.New("DeltaFIFO Pop Process",
            utiltrace.Field{Key: "ID", Value: id},
            utiltrace.Field{Key: "Depth", Value: depth},
            utiltrace.Field{Key: "Reason", Value: "slow event handlers blocking the queue"})
         defer trace.LogIfLong(100 * time.Millisecond)
      }
      err := process(item, isInInitialList)
      if e, ok := err.(ErrRequeue); ok {
         f.addIfNotPresent(id, item)
         err = e.Err
      }
      // Don't need to copyDeltas here, because we're transferring
      // ownership to the caller.
      return item, err
   }
}
```

### 组合

#### UndeltaStore

kubelet 中，Reflector 使用 UndeltaStore 作为 store，向 apiserver 中获取数据



#### watchCache

apiserver 中，Reflector 使用 watchCache 作为 store，向 etcd 中获取数据

