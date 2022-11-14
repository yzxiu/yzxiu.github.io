# Containerd解析(6) - restart


## 概述

在runc的start命令代码如下：

```go
switch status {
// 对于一个已经处于Created状态的容器，执行Exec
case libcontainer.Created:
   return container.Exec()
case libcontainer.Stopped:
   return errors.New("cannot start a container that has stopped")
case libcontainer.Running:
   return errors.New("cannot start an already running container")
default:
   return fmt.Errorf("cannot start a container in the %s state\n", status)
}
```

说明已经 `Stopped` 的容器`runc`是不支持重新启动的。



但是在 docker，或这nerdctl 中，都存在 restart 命令。

解析来以 nerdctl 为例，看一下restart命令的实现。

```go
if err := stopContainer(ctx, found.Container, timeout); err != nil {
	return err
}
if err := startContainer(ctx, found.Container, false, client); err != nil {
	return err
}
```

```go
func startContainer(ctx context.Context, container containerd.Container, flagA bool, client *containerd.Client) error {
    ...
    ...
	if oldTask, err := container.Task(ctx, nil); err == nil {
		if _, err := oldTask.Delete(ctx); err != nil {
			logrus.WithError(err).Debug("failed to delete old task")
		}
	}
	task, err := container.NewTask(ctx, taskCIO)
	if err != nil {
		return err
	}
	...
    ...
	if err := task.Start(ctx); err != nil {
		return err
	}
    ...
    ...
}
```

先oldTask.Delete，然后NewTask。

## oldTask.Delete

nerdctl  -> containerd -> shim



#### 通过runc删除容器

```go
err := p.runtime.Delete(ctx, p.id, nil)
```

![image-20221113203756562](https://raw.githubusercontent.com/yzxiu/images/blog/2022-11/20221113-203757.png "runc delete")



#### 取消rootfs挂载

```go
if err2 := mount.UnmountAll(p.Rootfs, 0); err2 != nil {
	log.G(ctx).WithError(err2).Warn("failed to cleanup rootfs mount")
	if err == nil {
		err = fmt.Errorf("failed rootfs umount: %w", err2)
	}
}
```

接下来通过 NewTask 重新创建容器，Start 启动容器，完成了容器的restart过程。



## 总结

从`runc`的角度来看，restart 过程是删除了容器，再重新创建了一个同名容器。






