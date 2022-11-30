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







## shim启动过程

主要分析 `containerd-shim-runc-v2`

```go
package main

import (
	"context"

	"github.com/containerd/containerd/runtime/v2/runc/manager"
	_ "github.com/containerd/containerd/runtime/v2/runc/pause"
	_ "github.com/containerd/containerd/runtime/v2/runc/task/plugin"
	"github.com/containerd/containerd/runtime/v2/shim"
)

func main() {
	shim.RunManager(context.Background(), manager.NewShimManager("io.containerd.runc.v2"))
}
```

```go
// NewShimManager returns an implementation of the shim manager
// using runc
func NewShimManager(name string) shim.Manager {
	return &manager{
		name: name,
	}
}
```

这里使用的实现类为

```go
// containerd/runtime/v2/runc/manager/manager_linux.go
type manager struct {
	name string
}
```

```go
// RunManager initialzes and runs a shim server
// TODO(2.0): Rename to Run
func RunManager(ctx context.Context, manager Manager, opts ...BinaryOpts) {
	var config Config
	for _, o := range opts {
		o(&config)
	}

	ctx = log.WithLogger(ctx, log.G(ctx).WithField("runtime", manager.Name()))

	if err := run(ctx, manager, nil, "", config); err != nil {
		fmt.Fprintf(os.Stderr, "%s: %s", manager.Name(), err)
		os.Exit(1)
	}
}

```

```go
func run(ctx context.Context, manager Manager, initFunc Init, name string, config Config) error {
   parseFlags()
   if versionFlag {
      fmt.Printf("%s:\n", filepath.Base(os.Args[0]))
      fmt.Println("  Version: ", version.Version)
      fmt.Println("  Revision:", version.Revision)
      fmt.Println("  Go version:", version.GoVersion)
      fmt.Println("")
      return nil
   }

   if namespaceFlag == "" {
      return fmt.Errorf("shim namespace cannot be empty")
   }

   setRuntime()

   // 1, 接收系统信号
   // smp := []os.Signal{unix.SIGTERM, unix.SIGINT, unix.SIGPIPE}
   // smp = append(smp, unix.SIGCHLD)
   signals, err := setupSignals(config)
   if err != nil {
      return err
   }

   // prctl(PR_SET_CHILD_SUBREAPER,1)
   // 让当前进程像init进程一样来收养孤儿进程，称为subreaper进程
   // 孤儿进程成会被祖先中距离最近的 supreaper 进程收养
   if !config.NoSubreaper {
      if err := subreaper(); err != nil {
         return err
      }
   }

   ttrpcAddress := os.Getenv(ttrpcAddressEnv)
   publisher, err := NewPublisher(ttrpcAddress)
   if err != nil {
      return err
   }
   defer publisher.Close()

   ctx = namespaces.WithNamespace(ctx, namespaceFlag)
   ctx = context.WithValue(ctx, OptsKey{}, Opts{BundlePath: bundlePath, Debug: debugFlag})
   ctx, sd := shutdown.WithShutdown(ctx)
   defer sd.Shutdown()

   // 初始化manager
   // 这里的manager并不为nil
   if manager == nil {
      service, err := initFunc(ctx, id, publisher, sd.Shutdown)
      if err != nil {
         return err
      }
      plugin.Register(&plugin.Registration{
         Type: plugin.TTRPCPlugin,
         ID:   "task",
         Requires: []plugin.Type{
            plugin.EventPlugin,
         },
         InitFn: func(ic *plugin.InitContext) (interface{}, error) {
            return taskService{service}, nil
         },
      })
      manager = shimToManager{
         shim: service,
         name: name,
      }
   }

   // Handle explicit actions
   switch action {
   case "delete":
      logger := log.G(ctx).WithFields(logrus.Fields{
         "pid":       os.Getpid(),
         "namespace": namespaceFlag,
      })
      go reap(ctx, logger, signals)
      ss, err := manager.Stop(ctx, id)
      if err != nil {
         return err
      }
      data, err := proto.Marshal(&shimapi.DeleteResponse{
         Pid:        uint32(ss.Pid),
         ExitStatus: uint32(ss.ExitStatus),
         ExitedAt:   protobuf.ToTimestamp(ss.ExitedAt),
      })
      if err != nil {
         return err
      }
      if _, err := os.Stdout.Write(data); err != nil {
         return err
      }
      return nil
   case "start":
      opts := StartOpts{
         ContainerdBinary: containerdBinaryFlag,
         Address:          addressFlag,
         TTRPCAddress:     ttrpcAddress,
         Debug:            debugFlag,
      }

      // 启动 shim server，并返回 sock address
      address, err := manager.Start(ctx, id, opts)
      if err != nil {
         return err
      }
      if _, err := os.Stdout.WriteString(address); err != nil {
         return err
      }
      return nil
   }

   if !config.NoSetupLogger {
      ctx, err = setLogger(ctx, id)
      if err != nil {
         return err
      }
   }

   plugin.Register(&plugin.Registration{
      Type: plugin.InternalPlugin,
      ID:   "shutdown",
      InitFn: func(ic *plugin.InitContext) (interface{}, error) {
         return sd, nil
      },
   })

   // Register event plugin
   plugin.Register(&plugin.Registration{
      Type: plugin.EventPlugin,
      ID:   "publisher",
      InitFn: func(ic *plugin.InitContext) (interface{}, error) {
         return publisher, nil
      },
   })

   var (
      initialized   = plugin.NewPluginSet()
      ttrpcServices = []ttrpcService{}

      ttrpcUnaryInterceptors = []ttrpc.UnaryServerInterceptor{}
   )
   plugins := plugin.Graph(func(*plugin.Registration) bool { return false })
   for _, p := range plugins {
      id := p.URI()
      log.G(ctx).WithField("type", p.Type).Infof("loading plugin %q...", id)

      initContext := plugin.NewContext(
         ctx,
         p,
         initialized,
         // NOTE: Root is empty since the shim does not support persistent storage,
         // shim plugins should make use state directory for writing files to disk.
         // The state directory will be destroyed when the shim if cleaned up or
         // on reboot
         "",
         bundlePath,
      )
      initContext.Address = addressFlag
      initContext.TTRPCAddress = ttrpcAddress

      // load the plugin specific configuration if it is provided
      //TODO: Read configuration passed into shim, or from state directory?
      //if p.Config != nil {
      // pc, err := config.Decode(p)
      // if err != nil {
      //    return nil, err
      // }
      // initContext.Config = pc
      //}

      result := p.Init(initContext)
      if err := initialized.Add(result); err != nil {
         return fmt.Errorf("could not add plugin result to plugin set: %w", err)
      }

      instance, err := result.Instance()
      if err != nil {
         if plugin.IsSkipPlugin(err) {
            log.G(ctx).WithError(err).WithField("type", p.Type).Infof("skip loading plugin %q...", id)
            continue
         }
         return fmt.Errorf("failed to load plugin %s: %w", id, err)
      }

      if src, ok := instance.(ttrpcService); ok {
         logrus.WithField("id", id).Debug("registering ttrpc service")
         ttrpcServices = append(ttrpcServices, src)

      }

      if src, ok := instance.(ttrpcServerOptioner); ok {
         ttrpcUnaryInterceptors = append(ttrpcUnaryInterceptors, src.UnaryInterceptor())
      }
   }

   if len(ttrpcServices) == 0 {
      return fmt.Errorf("required that ttrpc service")
   }

   unaryInterceptor := chainUnaryServerInterceptors(ttrpcUnaryInterceptors...)
   server, err := newServer(ttrpc.WithUnaryServerInterceptor(unaryInterceptor))
   if err != nil {
      return fmt.Errorf("failed creating server: %w", err)
   }

   for _, srv := range ttrpcServices {
      if err := srv.RegisterTTRPC(server); err != nil {
         return fmt.Errorf("failed to register service: %w", err)
      }
   }

   if err := serve(ctx, server, signals, sd.Shutdown); err != nil {
      if err != shutdown.ErrShutdown {
         return err
      }
   }

   // NOTE: If the shim server is down(like oom killer), the address
   // socket might be leaking.
   if address, err := ReadAddress("address"); err == nil {
      _ = RemoveSocket(address)
   }

   select {
   case <-publisher.Done():
      return nil
   case <-time.After(5 * time.Second):
      return errors.New("publisher not closed")
   }
}
```

`shim server`启动分为两个步骤

1. 调用 `containerd-shim-runc-v2 start`命令，在命令中构建cmd启动真正的shim server，然后返回与之连接的sock address。
2. 启动真正的`shim server`进程。

所以，真正启动`shim server`的代码，并不在

```go
switch action {
    // delete命令用于在shim进程意外退出(或者有其他情况？)，清理相关的容器进程
   case "delete":
   // start命令在代码中构建cmd，通过cmd从另外的进程启动 shim server，然后返回 sock address
   case "start":
}
```

所以继续看`switch action`之后的代码...

```go
// 注册插件
plugin.Register(&plugin.Registration{
   Type: plugin.InternalPlugin,
   ID:   "shutdown",
   InitFn: func(ic *plugin.InitContext) (interface{}, error) {
      return sd, nil
   },
})

// Register event plugin
plugin.Register(&plugin.Registration{
   Type: plugin.EventPlugin,
   ID:   "publisher",
   InitFn: func(ic *plugin.InitContext) (interface{}, error) {
      return publisher, nil
   },
})
```

启动服务

```go
// serve serves the ttrpc API over a unix socket in the current working directory
// and blocks until the context is canceled
func serve(ctx context.Context, server *ttrpc.Server, signals chan os.Signal, shutdown func()) error {
   dump := make(chan os.Signal, 32)
   setupDumpStacks(dump)

   path, err := os.Getwd()
   if err != nil {
      return err
   }

   l, err := serveListener(socketFlag)
   if err != nil {
      return err
   }
   go func() {
      defer l.Close()
      if err := server.Serve(ctx, l); err != nil && !errors.Is(err, net.ErrClosed) {
         log.G(ctx).WithError(err).Fatal("containerd-shim: ttrpc server failure")
      }
   }()
   logger := log.G(ctx).WithFields(logrus.Fields{
      "pid":       os.Getpid(),
      "path":      path,
      "namespace": namespaceFlag,
   })
   go func() {
      for range dump {
         dumpStacks(logger)
      }
   }()

   // 监听shim进程的退出信号
   // signal.Notify(ch, syscall.SIGINT, syscall.SIGTERM)
   go handleExitSignals(ctx, logger, shutdown)
    
    
   // smp := []os.Signal{unix.SIGTERM, unix.SIGINT, unix.SIGPIPE}
   // smp = append(smp, unix.SIGCHLD)
   // 实际上只处理了 SIGCHLD
   return reap(ctx, logger, signals)
}
```



## shim Monitor

shim对容器进程的监控，是shim进程比较核心的一个功能。接下来对该功能的实现进行解析。

首先查看启动过程中， `TaskService` 的启动代码：

```go
// NewTaskService creates a new instance of a task service
func NewTaskService(ctx context.Context, publisher shim.Publisher, sd shutdown.Service) (taskAPI.TaskService, error) {
   var (
      ep  oom.Watcher
      err error
   )
   if cgroups.Mode() == cgroups.Unified {
      ep, err = oomv2.New(publisher)
   } else {
      ep, err = oomv1.New(publisher)
   }
   if err != nil {
      return nil, err
   }
   go ep.Run(ctx)
   s := &service{
      context:    ctx,
      events:     make(chan interface{}, 128),
      // s.ec进行了订阅
      // 这个订阅比较重要，发送通知的事件基本都是这个订阅产生的
      ec:         reaper.Default.Subscribe(),
      ep:         ep,
      shutdown:   sd,
      containers: make(map[string]*runc.Container),
   }
   // 从 s.ec 中获取退出事件，处理后发送到 s.events 中
   go s.processExits()
    
   // 修改 runcC.Monitor 的实现
   runcC.Monitor = reaper.Default
    
   if err := s.initPlatform(); err != nil {
      return nil, fmt.Errorf("failed to initialized platform behavior: %w", err)
   }
   
   // 从 s.events 中获取时间，过滤处理之后，使用 publisher 发送事件
   go s.forward(ctx, publisher)
   sd.RegisterCallback(func(context.Context) error {
      close(s.events)
      return nil
   })

   if address, err := shim.ReadAddress("address"); err == nil {
      sd.RegisterCallback(func(context.Context) error {
         return shim.RemoveSocket(address)
      })
   }
   return s, nil
}
```





#### reap(ctx context.Context, logger *logrus.Entry, signals chan os.Signal)

主循环，持续监听 `SIGCHLD` 事件

```go
func reap(ctx context.Context, logger *logrus.Entry, signals chan os.Signal) error {
   logger.Info("starting signal loop")

   for {
      select {
      case <-ctx.Done():
         return ctx.Err()
      case s := <-signals:
         // Exit signals are handled separately from this loop
         // They get registered with this channel so that we can ignore such signals for short-running actions (e.g. `delete`)
         switch s {
         case unix.SIGCHLD:
            /
            if err := reaper.Reap(); err != nil {
               logger.WithError(err).Error("reap exit status")
            }
         case unix.SIGPIPE:
         }
      }
   }
}
```

##### Reap() error

SIGCHLD 事件处理

```go
// Reap should be called when the process receives an SIGCHLD.  Reap will reap
// all exited processes and close their wait channels
func Reap() error {
   now := time.Now()
   // 1
   exits, err := reap(false)
   for _, e := range exits {
      // 2
      done := Default.notify(runc.Exit{
         Timestamp: now,
         Pid:       e.Pid,
         Status:    e.Status,
      })

      select {
      case <-done:
      case <-time.After(1 * time.Second):
      }
   }
   return err
}
```

###### reap(wait bool) (exits []exit, err error)

获取子进程退出状态

```go
// reap reaps all child processes for the calling process and returns their
// exit information
func reap(wait bool) (exits []exit, err error) {
   var (
      ws  unix.WaitStatus
      rus unix.Rusage
   )
   flag := unix.WNOHANG
   if wait {
      flag = 0
   }
   for {
      pid, err := unix.Wait4(-1, &ws, flag, &rus)
      if err != nil {
         if err == unix.ECHILD {
            return exits, nil
         }
         return exits, err
      }
      if pid <= 0 {
         return exits, nil
      }
      exits = append(exits, exit{
         Pid:    pid,
         Status: exitStatus(ws),
      })
   }
}
```

###### notify(e runc.Exit)

将退出事件通知给各个订阅者。

这个函数看起来有些复杂，实际上可以理解为：

```go
func (m *Monitor) notify(e runc.Exit) chan struct{} {
    ...
    for _, s := range subscribers {
        s.c <- e
	}
    ...
}
```

```go
func (m *Monitor) notify(e runc.Exit) chan struct{} {
   const timeout = 1 * time.Millisecond
   var (
      done    = make(chan struct{}, 1)
      timer   = time.NewTimer(timeout)
      success = make(map[chan runc.Exit]struct{})
   )
   stop(timer, true)

   go func() {
      defer close(done)

      for {
         var (
            failed      int
            subscribers = m.getSubscribers()
         )
         for _, s := range subscribers {
            s.do(func() {
               if s.closed {
                  return
               }
               if _, ok := success[s.c]; ok {
                  return
               }
               timer.Reset(timeout)
               recv := true
               select {
               case s.c <- e:
                  success[s.c] = struct{}{}
               case <-timer.C:
                  recv = false
                  failed++
               }
               stop(timer, recv)
            })
         }
         // all subscribers received the message
         if failed == 0 {
            return
         }
      }
   }()
   return done
}
```



#### Monitor.Start(cmd) & Monitor.Wait(cmd, ec)

```go
// Start starts the command a registers the process with the reaper
func (m *Monitor) Start(c *exec.Cmd) (chan runc.Exit, error) {
   ec := m.Subscribe()
   if err := c.Start(); err != nil {
      m.Unsubscribe(ec)
      return nil, err
   }
   return ec, nil
}

// Wait blocks until a process is signal as dead.
// User should rely on the value of the exit status to determine if the
// command was successful or not.
func (m *Monitor) Wait(c *exec.Cmd, ec chan runc.Exit) (int, error) {
   for e := range ec {
      if e.Pid == c.Process.Pid {
         // make sure we flush all IO
         c.Wait()
         m.Unsubscribe(ec)
         return e.Status, nil
      }
   }
   // return no such process if the ec channel is closed and no more exit
   // events will be sent
   return -1, ErrNoSuchProcess
}
```

```go
// Subscribe to process exit changes
func (m *Monitor) Subscribe() chan runc.Exit {
   c := make(chan runc.Exit, bufferSize)
   m.Lock()
   m.subscribers[c] = &subscriber{
      c: c,
   }
   m.Unlock()
   return c
}

// Unsubscribe to process exit changes
func (m *Monitor) Unsubscribe(c chan runc.Exit) {
   m.Lock()
   s, ok := m.subscribers[c]
   if !ok {
      m.Unlock()
      return
   }
   s.close()
   delete(m.subscribers, c)
   m.Unlock()
}
```








