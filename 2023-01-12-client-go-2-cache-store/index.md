# client-go解析(2) - cache.Store


## 概述

如果把`k8s`当成`资源管理系统`, 那`cache.Store`无疑是`最核心`的接口, 用于`缓存`,`存储`资源 。

reflector 依赖于 cache.Store 的实现做存储，根据不同的实现有不同的功能。所以，正确的理解cache.Store的不同实现，是理解 reflector 的关键。

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

这里只分析基础的cache实现

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

关于FIFO, 需要注意的事,如果pop处理失败,会重新放入队列

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

通过单元测试来了解特性

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

![image-20230113165311586](https://raw.githubusercontent.com/yzxiu/images/blog/2023-01/20230113-165312.png "DeltaFIFO")

##### Replace()

 Replace() 可以视为生产者，生产类型为 Sync / Replaced 的 Delta 数据



##### Resync()

 Resync() 可以视为生产者，生产类型为 Sync 的 Delta 数据





##### Pop()

Pop() 可以视为消费者



### 组合



#### UndeltaStore



#### watchCache


