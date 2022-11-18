# Containerd解析(3) - shim


## 概述

记录shim相关的疑问和分析

<br>

## shim如何监控多个容器？

在k8s中，创建一个pod，如：`k run nginx --image=nginx`

实际上会创建两个容器，一个是 `pause` 容器（也称为`sandbox`容器），一个是`nginx`容器。

这两个容器的父进程都是同一个shim，如下：

![image-20221117172644477](https://raw.githubusercontent.com/yzxiu/images/blog/2022-11/20221117-172645.png "ps -ef --forest")

接下来通过代码调试，了解这个过程：

运行命令  `k run nginx --image=nginx`

#### 创建`pause`容器

```go
shim, err := m.startShim(ctx, bundle, id, opts)
...
...
out, err := cmd.CombinedOutput()
```

![image-20221117195622367](https://raw.githubusercontent.com/yzxiu/images/blog/2022-11/20221117-195623.png)

out的值

```
unix:///run/containerd/s/a27575c24e27006499fdd59b050358b4db870446a64a857fa02abae7d096a9b6
```



####   创建`nginx`容器

```go
shim, err := m.startShim(ctx, bundle, id, opts)
...
...
out, err := cmd.CombinedOutput()
```

![image-20221117195728875](https://raw.githubusercontent.com/yzxiu/images/blog/2022-11/20221117-195729.png)

out的值

```
unix:///run/containerd/s/a27575c24e27006499fdd59b050358b4db870446a64a857fa02abae7d096a9b6
```

创建的容器如下：

![image-20221117200321363](https://raw.githubusercontent.com/yzxiu/images/blog/2022-11/20221117-200322.png)

可以看到，虽然调用了两次 `shim, err := m.startShim(ctx, bundle, id, opts)`，但是返回了相同的shim服务地址，查看新增的shim进程，也只有一个，所以：

{{< admonition info >}}

containerd为同一个pod创建容器时，只有`pause`容器启动了`shim`进程(shim服务端)，并返回一个shim的通信地址。

后续启动同一个pod的其他容器时，虽然也调用了`m.startShim()`，但不会启动新的shim进程，而且返回**相同**的通信地址。

所以，同一个pod下的容器，都是由同一个shim进程创建管理。

{{< /admonition >}}



#### 代码实现

查看`shim` `start`命令相关代码：

```go
func (manager) Start(ctx context.Context, id string, opts shim.StartOpts) (_ string, retErr error) {
	// 构建另一个shim cmd
    // 真正的 shim server是由这个cmd启动的
    // 当前cmd返回了 sock address
    cmd, err := newCommand(ctx, id, opts.ContainerdBinary, opts.Address, opts.TTRPCAddress, opts.Debug)
	if err != nil {
		return "", err
	}
    // grouping 默认为 id
	grouping := id
	spec, err := readSpec()
	if err != nil {
		return "", err
	}
    // 查看 config.json 中是否存在 groupID
	for _, group := range groupLabels {
		if groupID, ok := spec.Annotations[group]; ok {
			grouping = groupID
			break
		}
	}
    // 计算shim服务端的address地址
    // unix:///run/containerd/s/a27575c24e27006499fdd59b050358b4db870446a64a857fa02abae7d096a9b6
	address, err := shim.SocketAddress(ctx, opts.Address, grouping)
	if err != nil {
		return "", err
	}
	
    // 尝试创建sock
	socket, err := shim.NewSocket(address)
	if err != nil {
		// the only time where this would happen is if there is a bug and the socket
		// was not cleaned up in the cleanup method of the shim or we are using the
		// grouping functionality where the new process should be run with the same
		// shim as an existing container
		if !shim.SocketEaddrinuse(err) {
			return "", fmt.Errorf("create new shim socket: %w", err)
		}
        // 如果创建失败，可能已经创建过
        // 尝试连接，并直接返回 address
        // pod中除了pause容器，其他容器属于这种情况
		if shim.CanConnect(address) {
			if err := shim.WriteAddress("address", address); err != nil {
				return "", fmt.Errorf("write existing socket for shim: %w", err)
			}
			return address, nil
		}
		if err := shim.RemoveSocket(address); err != nil {
			return "", fmt.Errorf("remove pre-existing socket: %w", err)
		}
		if socket, err = shim.NewSocket(address); err != nil {
			return "", fmt.Errorf("try create new shim socket 2x: %w", err)
		}
	}
	defer func() {
		if retErr != nil {
			socket.Close()
			_ = shim.RemoveSocket(address)
		}
	}()

	// make sure that reexec shim-v2 binary use the value if need
	if err := shim.WriteAddress("address", address); err != nil {
		return "", err
	}

	f, err := socket.File()
	if err != nil {
		return "", err
	}
	// 将文件描述符传递给子进程
	cmd.ExtraFiles = append(cmd.ExtraFiles, f)

	goruntime.LockOSThread()
	if os.Getenv("SCHED_CORE") != "" {
		if err := schedcore.Create(schedcore.ProcessGroup); err != nil {
			return "", fmt.Errorf("enable sched core support: %w", err)
		}
	}

	if err := cmd.Start(); err != nil {
		f.Close()
		return "", err
	}

	goruntime.UnlockOSThread()

	defer func() {
		if retErr != nil {
			cmd.Process.Kill()
		}
	}()
	// make sure to wait after start
	go cmd.Wait()
	if data, err := io.ReadAll(os.Stdin); err == nil {
		if len(data) > 0 {
			var any ptypes.Any
			if err := proto.Unmarshal(data, &any); err != nil {
				return "", err
			}
			v, err := typeurl.UnmarshalAny(&any)
			if err != nil {
				return "", err
			}
			if opts, ok := v.(*options.Options); ok {
				if opts.ShimCgroup != "" {
					if cgroups.Mode() == cgroups.Unified {
						cg, err := cgroupsv2.LoadManager("/sys/fs/cgroup", opts.ShimCgroup)
						if err != nil {
							return "", fmt.Errorf("failed to load cgroup %s: %w", opts.ShimCgroup, err)
						}
						if err := cg.AddProc(uint64(cmd.Process.Pid)); err != nil {
							return "", fmt.Errorf("failed to join cgroup %s: %w", opts.ShimCgroup, err)
						}
					} else {
						cg, err := cgroups.Load(cgroups.V1, cgroups.StaticPath(opts.ShimCgroup))
						if err != nil {
							return "", fmt.Errorf("failed to load cgroup %s: %w", opts.ShimCgroup, err)
						}
						if err := cg.AddProc(uint64(cmd.Process.Pid)); err != nil {
							return "", fmt.Errorf("failed to join cgroup %s: %w", opts.ShimCgroup, err)
						}
					}
				}
			}
		}
	}
	if err := shim.AdjustOOMScore(cmd.Process.Pid); err != nil {
		return "", fmt.Errorf("failed to adjust OOM score for shim: %w", err)
	}
	return address, nil
}
```

#### 小结

`shim`执行`start`命令的时候，首先根据config.json的annotations字段，查看有没有`groupLabels`，如下：

```go
var groupLabels = []string{
   "io.containerd.runc.v2.group",
   "io.kubernetes.cri.sandbox-id",
}
```

```json
// pause容器的 annotations
"annotations": {
        "io.kubernetes.cri.container-type": "sandbox",
        "io.kubernetes.cri.sandbox-id": "773f854a158825b6489ea0b90c4cbfe516ff1e4d302cda0ccd930d8237d16c0d",
        "io.kubernetes.cri.sandbox-log-directory": "/var/log/pods/default_nginx_d484032b-b35c-4511-8db9-42050023ac61",
        "io.kubernetes.cri.sandbox-name": "nginx",
        "io.kubernetes.cri.sandbox-namespace": "default"
}
```

```json
// nginx容器的 annotations
"annotations": {
        "io.kubernetes.cri.container-name": "nginx",
        "io.kubernetes.cri.container-type": "container",
        "io.kubernetes.cri.image-name": "docker.io/library/nginx:latest",
        "io.kubernetes.cri.sandbox-id": "773f854a158825b6489ea0b90c4cbfe516ff1e4d302cda0ccd930d8237d16c0d",
        "io.kubernetes.cri.sandbox-name": "nginx",
        "io.kubernetes.cri.sandbox-namespace": "default"
}
```

如果存在 groupLabels，则使用 groupID 替换 id。

所以，同一个pod里面的pause 容器和 nginx容器，计算出来的 shim sock address 是一样的。

首先创建 pause 容器的时候，socket file不存在，使用cmd正常启动shim server，然后返回socket address。

接下来创建nginx容器的时候，socket file已存在，验证socket连接，成功则**直接**返回 socket address地址。



<br>

## shim意外退出，会发生什么？

先说结论：

如果shim意外退出，由shim创建的子进程(也就是容器)会跟着退出。

containerd会运行`m.startShim()`重新启动`shim`和重新启动相关容器














