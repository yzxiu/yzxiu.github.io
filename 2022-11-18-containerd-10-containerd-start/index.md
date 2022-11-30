# Containerd解析(10) - Containerd Start


## 概述

了解containerd的启动过程。

重点关注api接口 `ContainersClient`、 `TasksClient`的实现。

ContainersClient的实现类是：

```go
// containerd/services/containers/local.go
type local struct {
   containers.Store
   db        *metadata.DB
   publisher events.Publisher
}
```

TasksClient的实现类是：

```go
type local struct {
	runtimes   map[string]runtime.PlatformRuntime
	containers containers.Store
	store      content.Store
	publisher  events.Publisher

	monitor   runtime.TaskMonitor
	v2Runtime runtime.PlatformRuntime
}
```



<br>




