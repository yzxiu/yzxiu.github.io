# Containerd解析(5) - go-runc


在 containerd-shim-runc-v2 的代码中，调用 runc 的操作都是由 [go-runc](https://github.com/containerd/go-runc) 这个库去完成的。



## reaper.Default

reaper.Default 虽然没有在 go-runc 里面。但实际执行过程在使用了它的实现。

```go
// Default is the default monitor initialized for the package
var Default = &Monitor{
	subscribers: make(map[chan runc.Exit]*subscriber),
}

// Monitor monitors the underlying system for process status changes
type Monitor struct {
	sync.Mutex

	subscribers map[chan runc.Exit]*subscriber
}
```



http://www.asznl.com/post/32

https://plpan.github.io/pod-terminating-%E6%8E%92%E6%9F%A5%E4%B9%8B%E6%97%85/

