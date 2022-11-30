# Containerd解析(9) - Containerd,Containerd-shim,runc的依存关系


## 概述

参考[文章](https://fankangbest.github.io/2017/11/24/containerd-containerd-shim%E5%92%8Crunc%E7%9A%84%E4%BE%9D%E5%AD%98%E5%85%B3%E7%B3%BB/)，该文时间比较久远，containerd的很多逻辑已经发生变化。

学习该文思路，重新整理`Containerd`,`Containerd-shim`,`runc` 之间的依存关系。



<br>

## 整体关系

使用nerdctl启动容器`n run -d --name runcdev q946666800/runcdev`

进程如下：

```
root        3716       1  0 10:36 ?        00:00:00 /usr/bin/containerd
root        4509       1  0 10:37 ?        00:00:00 /usr/bin/containerd-shim-runc-v2
root        4567    4509  0 10:37 ?        00:00:00  \_ /main-example
```

containerd 与 shim的父进程都是1, 容器进程父进程是shim。

接下来，分析下面四种情况：

1. containerd进程存在的情况下，杀死containerd-shim进程；
2. containerd进程存在的情况下，杀死容器进程；
3. containerd进程不存在的情况下，杀死containerd-shim进程，然后启动containerd进程；
4. containerd进程不存在的情况下，杀死容器进程，然后启动containerd进程；



<br>

## 第一种情况

containerd进程存在的情况下，杀死containerd-shim进程。

```
~ ps -ef | egrep 'containerd|example' | grep -v grep
root        3716       1  0 10:36 ?        00:00:00 /usr/bin/containerd
root        4509       1  0 10:37 ?        00:00:00 /usr/bin/containerd-shim-runc-v2
root        4567    4509  0 10:37 ?        00:00:00  \_ /main-example
```

使用`kill -9 4509`杀死shim进程，看到容器进程也跟着退出。

```
~ ps -ef | egrep 'containerd|example' | grep -v grep
root        3716       1  0 10:36 ?        00:00:00 /usr/bin/containerd
```

{{< admonition info >}}
结论：在containerd运行的情况下，杀死containerd-shim，**容器进程会退出**。
{{< /admonition >}}

看下为什么容器进程会退出。

查看`containerd`启动`containerd-shim`的相关代码，也就是 `m.startShim()`

```go
func (m *ShimManager) startShim(ctx context.Context, bundle *Bundle, id string, opts runtime.CreateOpts) (*shim, error) {
   ...
   ...
   shim, err := b.Start(ctx, protobuf.FromAny(topts), func() {
      log.G(ctx).WithField("id", id).Info("shim disconnected")

      cleanupAfterDeadShim(context.Background(), id, ns, m.shims, m.events, b)
      // Remove self from the runtime task list. Even though the cleanupAfterDeadShim()
      // would publish taskExit event, but the shim.Delete() would always failed with ttrpc
      // disconnect and there is no chance to remove this dead task from runtime task lists.
      // Thus it's better to delete it here.
      m.shims.Delete(ctx, id)
   })
   if err != nil {
      return nil, fmt.Errorf("start failed: %w", err)
   }

   return shim, nil
}
```

注意到，在`b.Start` 后面传入了一个回调函数

```go
func() {
      log.G(ctx).WithField("id", id).Info("shim disconnected")

      cleanupAfterDeadShim(context.Background(), id, ns, m.shims, m.events, b)
      // Remove self from the runtime task list. Even though the cleanupAfterDeadShim()
      // would publish taskExit event, but the shim.Delete() would always failed with ttrpc
      // disconnect and there is no chance to remove this dead task from runtime task lists.
      // Thus it's better to delete it here.
      m.shims.Delete(ctx, id)
}
```

从该函数名 `cleanupAfterDeadShim` 可以推测主要是做一些清理工作。

先看一下 b.Start()

```go
func (b *binary) Start(ctx context.Context, opts *types.Any, onClose func()) (_ *shim, err error) {
	args := []string{"-id", b.bundle.ID}
	switch logrus.GetLevel() {
	case logrus.DebugLevel, logrus.TraceLevel:
		args = append(args, "-debug")
	}
	args = append(args, "start")
	
    // 1. 构建cmd
	cmd, err := client.Command(
		ctx,
		&client.CommandConfig{
			Runtime:      b.runtime,
			Address:      b.containerdAddress,
			TTRPCAddress: b.containerdTTRPCAddress,
			Path:         b.bundle.Path,
			Opts:         opts,
			Args:         args,
			SchedCore:    b.schedCore,
		})
	if err != nil {
		return nil, err
	}
    ...
    ...
    // 2. 启动shim server，并获取 sock address
	out, err := cmd.CombinedOutput()
	if err != nil {
		return nil, fmt.Errorf("%s: %w", out, err)
	}
	address := strings.TrimSpace(string(out))
	conn, err := client.Connect(address, client.AnonDialer)
	if err != nil {
		return nil, err
	}
    // 连接丢失的回调函数，包含刚刚传进来的 onClose
	onCloseWithShimLog := func() {
		onClose()
		cancelShimLog()
		f.Close()
	}
	// Save runtime binary path for restore.
	if err := os.WriteFile(filepath.Join(b.bundle.Path, "shim-binary-path"), []byte(b.runtime), 0600); err != nil {
		return nil, err
	}
    // 3.与sock address 建立连接，并在连接关闭的时候调用传进来的回调函数onClose
	client := ttrpc.NewClient(conn, ttrpc.WithOnClose(onCloseWithShimLog))
	return &shim{
		bundle: b.bundle,
		client: client,
	}, nil
}

```

1. 构建cmd
2. 启动`shim server`，并获取 `sock address`
3. 通过`sock address`与刚刚启动的`shim server`建立连接，并在`连接关闭`时调用回调函数`onClose`，从而执行 `cleanupAfterDeadShim`

```go
func cleanupAfterDeadShim(ctx context.Context, id, ns string, rt *runtime.NSMap[ShimInstance], events *exchange.Exchange, binaryCall *binary) {
   ...
   response, err := binaryCall.Delete(ctx)
   ...
}
```

```go
func (b *binary) Delete(ctx context.Context) (*runtime.Exit, error) {
   log.G(ctx).Info("cleaning up dead shim")
   ...
   cmd, err := client.Command(ctx,
      &client.CommandConfig{
         Runtime:      b.runtime,
         Address:      b.containerdAddress,
         TTRPCAddress: b.containerdTTRPCAddress,
         Path:         bundlePath,
         Opts:         nil,
         Args: []string{
            "-id", b.bundle.ID,
            "-bundle", b.bundle.Path,
            "delete",
         },
   })
   ...
   if err := cmd.Run(); err != nil {
      log.G(ctx).WithField("cmd", cmd).WithError(err).Error("failed to delete")
      return nil, fmt.Errorf("%s: %w", errb.String(), err)
   }
   ...
}
```

查看cmd：

![image-20221118162848004](https://raw.githubusercontent.com/yzxiu/images/blog/2022-11/20221118-162848.png "clean up cmd")

可以看到，cleanup是通过`containerd-shim`的 `delete` 命令进行runc容器的清理，具体细节这里不进行展开。





<br>

## 第二种情况

containerd进程存在的情况下，杀死容器进程;

~~一方面，在容器进程退出时，containerd-shim也会捕获到信号退出，这将在第四种情况下详细分析。~~

~~另一方面，容器进程退出，containerd中的monitor会会捕获到该事件，从而触发容器进程退出流程，这是本小节详细分析的内容。~~

杀死容器进程，`shim` `service`中的订阅者`reaper.Default.Subscribe()`会收到退出状态，

```go
s := &service{
   context:    ctx,
   events:     make(chan interface{}, 128),
   ec:         reaper.Default.Subscribe(),
   ep:         ep,
   shutdown:   sd,
   containers: make(map[string]*runc.Container),
}
```

然后将该退出状态推送至 containerd。









## 第三种情况

containerd进程不存在的情况下，杀死containerd-shim进程，然后启动containerd进程



首先杀死 containerd 进程，shim进程和容器进程依然存在。

```
[xiu-desktop] ~ sp shim      
[shim] 相关进程信息如下：
root      118803       1  0 11月27 ?      00:00:01 /usr/bin/containerd-shim-runc-v2 -namespace default -id dc236284189782b7c37131ad23473acaf8b8282e3532baf18928ec4e3949713e -address /run/containerd/containerd.sock -debug
[xiu-desktop] ~ sp main-example
[main-example] 相关进程信息如下：
root      118880  118803  0 11月27 ?      00:00:04 /main-example

```

接着杀死shim进程，容器进程依然存在，并且被进程1接收（父进程变为1）

```
[xiu-desktop] ~ kp shim  
[shim] 相关进程信息如下：
root      118803       1  0 11月27 ?      00:00:01 /usr/bin/containerd-shim-runc-v2 -namespace default -id dc236284189782b7c37131ad23473acaf8b8282e3532baf18928ec4e3949713e -address /run/containerd/containerd.sock -debug
是否关闭 [shim] 相关进程? (y/n)
y
已关闭 1 个 [shim] 相关进程
[xiu-desktop] ~ sp main-example
[main-example] 相关进程信息如下：
root      118880       1  0 11月27 ?      00:00:04 /main-example
```

然后再启动containerd，容器进程消失。

```
[xiu-desktop] ~ sp main-example
没有 [main-example] 相关进程
```

这时候容器变为 created 状态

```
[xiu-desktop] ~ n ps -a
CONTAINER ID    IMAGE                                  COMMAND            CREATED        STATUS     PORTS    NAMES
dc2362841897    docker.io/q946666800/runcdev:latest    "/main-example"    4 hours ago    Created             runcdev
```

代码解析：

残留容器进程的清理工作，主要在 `RuntimePluginV2` 插件初始化过程中触发的，相关调用堆栈如下：

```
v2.(*binary).Delete (binary.go:154) github.com/containerd/containerd/runtime/v2
v2.cleanupAfterDeadShim (shim.go:143) github.com/containerd/containerd/runtime/v2
v2.(*ShimManager).loadShims (shim_load.go:141) github.com/containerd/containerd/runtime/v2
v2.(*ShimManager).loadExistingTasks (shim_load.go:46) github.com/containerd/containerd/runtime/v2
v2.NewShimManager (manager.go:152) github.com/containerd/containerd/runtime/v2
v2.init.0.func1 (manager.go:84) github.com/containerd/containerd/runtime/v2
plugin.(*Registration).Init (plugin.go:115) github.com/containerd/containerd/plugin
server.New (server.go:230) github.com/containerd/containerd/services/server
command.App.func1.1 (main.go:194) github.com/containerd/containerd/cmd/containerd/command
runtime.goexit (asm_amd64.s:1571) runtime
 - 异步堆栈跟踪
command.App.func1 (main.go:191) github.com/containerd/containerd/cmd/containerd/command
```

查看几个重要的方法：

#### loadExistingTasks

```go
func (m *ShimManager) loadExistingTasks(ctx context.Context) error {
   nsDirs, err := os.ReadDir(m.state)
   if err != nil {
      return err
   }
   // 遍历 ns
   for _, nsd := range nsDirs {
      if !nsd.IsDir() {
         continue
      }
      ns := nsd.Name()
      // skip hidden directories
      if len(ns) > 0 && ns[0] == '.' {
         continue
      }
      log.G(ctx).WithField("namespace", ns).Debug("loading tasks in namespace")
      if err := m.loadShims(namespaces.WithNamespace(ctx, ns)); err != nil {
         log.G(ctx).WithField("namespace", ns).WithError(err).Error("loading tasks in namespace")
         continue
      }
      if err := m.cleanupWorkDirs(namespaces.WithNamespace(ctx, ns)); err != nil {
         log.G(ctx).WithField("namespace", ns).WithError(err).Error("cleanup working directory in namespace")
         continue
      }
   }
   return nil
}
```

![image-20221201000557547](https://raw.githubusercontent.com/yzxiu/images/blog/2022-12/20221201-000558.png "ns dir")



#### loadShims

```go
func (m *ShimManager) loadShims(ctx context.Context) error {
   ns, err := namespaces.NamespaceRequired(ctx)
   if err != nil {
      return err
   }
   shimDirs, err := os.ReadDir(filepath.Join(m.state, ns))
   if err != nil {
      return err
   }
   for _, sd := range shimDirs {
      if !sd.IsDir() {
         continue
      }
      id := sd.Name()
      // skip hidden directories
      if len(id) > 0 && id[0] == '.' {
         continue
      }
      bundle, err := LoadBundle(ctx, m.state, id)
      if err != nil {
         // fine to return error here, it is a programmer error if the context
         // does not have a namespace
         return err
      }
      // fast path
      bf, err := os.ReadDir(bundle.Path)
      if err != nil {
         bundle.Delete()
         log.G(ctx).WithError(err).Errorf("fast path read bundle path for %s", bundle.Path)
         continue
      }
      if len(bf) == 0 {
         bundle.Delete()
         continue
      }

      var (
         runtime string
      )

      // If we're on 1.6+ and specified custom path to the runtime binary, path will be saved in 'shim-binary-path' file.
      if data, err := os.ReadFile(filepath.Join(bundle.Path, "shim-binary-path")); err == nil {
         runtime = string(data)
      } else if err != nil && !os.IsNotExist(err) {
         log.G(ctx).WithError(err).Error("failed to read `runtime` path from bundle")
      }

      // Query runtime name from metadata store
      if runtime == "" {
         container, err := m.containers.Get(ctx, id)
         if err != nil {
            log.G(ctx).WithError(err).Errorf("loading container %s", id)
            if err := mount.UnmountAll(filepath.Join(bundle.Path, "rootfs"), 0); err != nil {
               log.G(ctx).WithError(err).Errorf("failed to unmount of rootfs %s", id)
            }
            bundle.Delete()
            continue
         }
         runtime = container.Runtime.Name
      }

      runtime, err = m.resolveRuntimePath(runtime)
      if err != nil {
         bundle.Delete()
         log.G(ctx).WithError(err).Error("failed to resolve runtime path")
         continue
      }

      binaryCall := shimBinary(bundle,
         shimBinaryConfig{
            runtime:      runtime,
            address:      m.containerdAddress,
            ttrpcAddress: m.containerdTTRPCAddress,
            schedCore:    m.schedCore,
         })
      instance, err := loadShim(ctx, bundle, func() {
         log.G(ctx).WithField("id", id).Info("shim disconnected")

         cleanupAfterDeadShim(context.Background(), id, ns, m.shims, m.events, binaryCall)
         // Remove self from the runtime task list.
         m.shims.Delete(ctx, id)
      })
       
      // 在这里进入 cleanupAfterDeadShim，进行清理工作
      if err != nil {
         cleanupAfterDeadShim(ctx, id, ns, m.shims, m.events, binaryCall)
         continue
      }
      shim := newShimTask(instance)

      // There are 3 possibilities for the loaded shim here:
      // 1. It could be a shim that is running a task.
      // 2. It could be a sandbox shim.
      // 3. Or it could be a shim that was created for running a task but
      // something happened (probably a containerd crash) and the task was never
      // created. This shim process should be cleaned up here. Look at
      // containerd/containerd#6860 for further details.

      _, sgetErr := m.sandboxStore.Get(ctx, id)
      pInfo, pidErr := shim.Pids(ctx)
      if sgetErr != nil && errors.Is(sgetErr, errdefs.ErrNotFound) && (len(pInfo) == 0 || errors.Is(pidErr, errdefs.ErrNotFound)) {
         log.G(ctx).WithField("id", id).Info("cleaning leaked shim process")
         // We are unable to get Pids from the shim and it's not a sandbox
         // shim. We should clean it up her.
         // No need to do anything for removeTask since we never added this shim.
         shim.delete(ctx, false, func(ctx context.Context, id string) {})
      } else {
         m.shims.Add(ctx, shim.ShimInstance)
      }
   }
   return nil
}
```

![image-20221201000809738](https://raw.githubusercontent.com/yzxiu/images/blog/2022-12/20221201-000810.png "shimDirs")

文件夹内容：

```
drwx------ 3 root root   240 11月 30 16:23 ./
drwx--x--x 3 root root    60 11月 30 16:23 ../
-rw-r--r-- 1 root root    89 11月 30 16:23 address
-rw-r--r-- 1 root root 15058 11月 30 16:23 config.json
-rw-r--r-- 1 root root     5 11月 30 16:23 init.pid
prwx------ 1 root root     0 11月 30 16:23 log|
-rw-r--r-- 1 root root     0 11月 30 16:23 log.json
-rw------- 1 root root     2 11月 30 16:23 options.json
drwxr-xr-x 1 root root  4096 11月 28 02:22 rootfs/
-rw------- 1 root root     0 11月 30 16:23 runtime
-rw------- 1 root root    32 11月 30 16:23 shim-binary-path
lrwxrwxrwx 1 root root   122 11月 30 16:23 work -> /var/lib/containerd/io.containerd.runtime.v2.task/default/7206459bb2db6c1636a9d2931e1b0edcd5ae82fd3426796d8959c76c3c81c1c8/

```

接下来调用：

cleanupAfterDeadShim()  ->  Delete()

与情况1类似。







## 第四种情况

containerd进程不存在的情况下，杀死容器进程，然后启动containerd进程

先杀掉 containerd进程，shim 和 容器进程都存在：

```
[xiu-desktop] ~ sp shim        
[shim] 相关进程信息如下：
root      140133       1  0 02:22 ?        00:00:00 /usr/bin/containerd-shim-runc-v2 -namespace default -id 7206459bb2db6c1636a9d2931e1b0edcd5ae82fd3426796d8959c76c3c81c1c8 -address /run/containerd/containerd.sock -debug
[xiu-desktop] ~ sp main-example
[main-example] 相关进程信息如下：
root      140163  140133  0 02:22 ?        00:00:00 /main-example
```



然后杀掉容器进程，shim进程依然存在：

```
[xiu-desktop] ~ sudo kill -9 140163
[xiu-desktop] ~ sp shim            
[shim] 相关进程信息如下：
root      140133       1  0 02:22 ?        00:00:00 /usr/bin/containerd-shim-runc-v2 -namespace default -id 7206459bb2db6c1636a9d2931e1b0edcd5ae82fd3426796d8959c76c3c81c1c8 -address /run/containerd/containerd.sock -debug

```



然后重启containerd，shim进程消失

```
[xiu-desktop] ~ sp shim        
没有 [shim] 相关进程
[xiu-desktop] ~ sp main-example
没有 [main-example] 相关进程
```



这时候容器变为 created 状态

```
[xiu-desktop] ~ n ps -a
CONTAINER ID    IMAGE                                  COMMAND            CREATED          STATUS     PORTS    NAMES
7206459bb2db    docker.io/q946666800/runcdev:latest    "/main-example"    4 minutes ago    Created             runcdev
```

