# Containerd解析(12) - network


## 概述

分析containerd网络，从不同的上层应用：

1. nerdctl run -d --name runcdev -p 8080:80 nginx
2. kubectl run nginx --image=nginx

<br>

根据containerd 的 **[Scope and principles](https://containerd.io/scope/)**

The following table specifies the various components of containerd and general features of container runtimes. The table specifies whether or not the feature/component is in or out of scope.

| Name           | Description                                                  | In/Out | Reason                                                       |
| :------------- | :----------------------------------------------------------- | :----- | :----------------------------------------------------------- |
| execution      | Provide an extensible execution layer for executing a container | in     | Create,start, stop pause, resume exec, signal, delete        |
| cow filesystem | Built in functionality for overlay, aufs, and other copy on write filesystems for containers | in     |                                                              |
| distribution   | Having the ability to push and pull images as well as operations on images as a first class API object | in     | containerd will fully support the management and retrieval of images |
| metrics        | container-level metrics, cgroup stats, and OOM events        | in     |                                                              |
| networking     | creation and management of network interfaces                | out    | Networking will be handled and provided to containerd via higher level systems. |
| build          | Building images as a first class API                         | out    | Build is a higher level tooling feature and can be implemented in many different ways on top of containerd |
| volumes        | Volume management for external data                          | out    | The API supports mounts, binds, etc where all volumes type systems can be built on top of containerd. |
| logging        | Persisting container logs                                    | out    | Logging can be build on top of containerd because the container’s STDIO will be provided to the clients and they can persist any way they see fit. There is no io copying of container STDIO in containerd. |

支持：    运行/文件系统/拉取，推送/指标

不支持：网络/构建/外部存储卷/日志

从上表可知，containerd并不负责容器网络的配置工作。

<br>

## nerdctl run -d --name runcdev -p 8080:80 nginx

nerdctl使用cri插件进行网络配置，默认插件为`bridge`

#### 从一个报错开始

删除 cni bridge二进制文件，`sudo rm -rf /opt/cni/bin/bridge`，重新创建容器，报错如下：

```
[xiu-notebook] ~ n run -d --name runcdev -p 8080:80 nginx
[sudo] password for xiu: 
FATA[0005] failed to create shim task: OCI runtime create failed: container_linux.go:380: starting container process caused: process_linux.go:545: container init caused: Running hook #0:: error running hook: exit status 1, stdout: , stderr: time="2022-12-02T13:58:11+08:00" level=fatal msg="failed to call cni.Setup: plugin type=\"bridge\" failed (add): failed to find plugin \"bridge\" in path [/opt/cni/bin]"
Failed to write to log, write /var/lib/nerdctl/1935db59/containers/default/7aef42386e604726fec25f9f62acbf128c1b5648cd41259eefa3503602f65353/oci-hook.createRuntime.log: file already closed: unknown
```

查看hook相关定义：[Hooks](https://github.com/opencontainers/runtime-spec/blob/main/config.md#posix-platform-hooks)

#### CreateRuntime Hooks

The `createRuntime` hooks MUST be called as part of the [`create`](https://github.com/opencontainers/runtime-spec/blob/main/runtime.md#create) operation after the runtime environment has been created (according to the configuration in config.json) but before the `pivot_root` or any equivalent operation has been executed.

The `createRuntime` hooks' path MUST resolve in the [runtime namespace](https://github.com/opencontainers/runtime-spec/blob/main/glossary.md#runtime-namespace). The `createRuntime` hooks MUST be executed in the [runtime namespace](https://github.com/opencontainers/runtime-spec/blob/main/glossary.md#runtime-namespace).

On Linux, for example, they are called after the container namespaces are created, so they provide an opportunity to customize the container (e.g. the network namespace could be specified in this hook).

The definition of `createRuntime` hooks is currently underspecified and hooks authors, should only expect from the runtime that the mount namespace have been created and the mount operations performed. Other operations such as cgroups and SELinux/AppArmor labels might not have been performed by the runtime.

Note: `runc` originally implemented `prestart` hooks contrary to the spec, namely as part of the `create` operation (instead of during the `start` operation). This incorrect implementation actually corresponds to `createRuntime` hooks. For runtimes that implement the deprecated `prestart` hooks as `createRuntime` hooks, `createRuntime` hooks MUST be called after the `prestart` hooks.

#### hook

查看容器config.json中hook的相关内容：

```json
{
    "ociVersion": "1.0.2-dev",
    "process": {},
    "root": {
        "path": "rootfs"
    },
    "hostname": "5bed8c875756",
    "mounts": [],
    "hooks": {
        "createRuntime": [
            {
                "path": "/home/xiu/CloudDrive/UBUNTU/application/bin/nerdctl",
                "args": [
                    "/home/xiu/CloudDrive/UBUNTU/application/bin/nerdctl",
                    "internal",
                    "oci-hook",
                    "createRuntime"
                ],
                "env": [
                ]
            }
        ],
        "poststop": [
            {
                "path": "/home/xiu/CloudDrive/UBUNTU/application/bin/nerdctl",
                "args": [
                    "/home/xiu/CloudDrive/UBUNTU/application/bin/nerdctl",
                    "internal",
                    "oci-hook",
                    "postStop"
                ],
                "env": [
                ]
            }
        ]
    },
    "annotations": {
        "io.containerd.image.config.stop-signal": "SIGQUIT",
        "nerdctl/extraHosts": "null",
        "nerdctl/hostname": "5bed8c875756",
        "nerdctl/log-uri": "binary:///home/xiu/CloudDrive/UBUNTU/application/bin/nerdctl?_NERDCTL_INTERNAL_LOGGING=%2Fvar%2Flib%2Fnerdctl%2F1935db59",
        "nerdctl/name": "runcdev",
        "nerdctl/namespace": "default",
        "nerdctl/networks": "[\"bridge\"]",
        "nerdctl/platform": "linux/amd64",
        "nerdctl/ports": "[{\"HostPort\":8080,\"ContainerPort\":80,\"Protocol\":\"tcp\",\"HostIP\":\"0.0.0.0\"}]",
        "nerdctl/state-dir": "/var/lib/nerdctl/1935db59/containers/default/5bed8c875756abdac43e664d2e7a1950a838675ca143ab71e3a4d8bd0ef9d59f"
    },
    "linux": {}
}
```

nerdctl在创建容器过程中，会添加`createRuntime`，`poststop`两个`hook`，不管存不存在`-p`参数，构造hook的方法如下：

```go
nerdctl/cmd/nerdctl/run.go
func withNerdctlOCIHook(cmd *cobra.Command, id string) (oci.SpecOpts, error) {
	selfExe, f := globalFlags(cmd)
	args := append([]string{selfExe}, append(f, "internal", "oci-hook")...)
	return func(_ context.Context, _ oci.Client, _ *containers.Container, s *specs.Spec) error {
		if s.Hooks == nil {
			s.Hooks = &specs.Hooks{}
		}
		crArgs := append(args, "createRuntime")
		s.Hooks.CreateRuntime = append(s.Hooks.CreateRuntime, specs.Hook{
			Path: selfExe,
			Args: crArgs,
			Env:  os.Environ(),
		})
		argsCopy := append([]string(nil), args...)
		psArgs := append(argsCopy, "postStop")
		s.Hooks.Poststop = append(s.Hooks.Poststop, specs.Hook{
			Path: selfExe,
			Args: psArgs,
			Env:  os.Environ(),
		})
		return nil
	}, nil
}
```

runc在启动容器过程中，父进程会执行相应的hook方法，如下：

```go
runc/libcontainer/process_linux.go

if err := hooks[configs.CreateRuntime].RunHooks(s); err != nil {
	return err
}
```

```go
runc/libcontainer/configs/config.go

func (c Command) Run(s *specs.State) error {
	b, err := json.Marshal(s)
	if err != nil {
		return err
	}
	var stdout, stderr bytes.Buffer
	cmd := exec.Cmd{
		Path:   c.Path,
		Args:   c.Args,
		Env:    c.Env,
		Stdin:  bytes.NewReader(b),
		Stdout: &stdout,
		Stderr: &stderr,
	}
	if err := cmd.Start(); err != nil {
		return err
	}
	errC := make(chan error, 1)
	go func() {
		err := cmd.Wait()
		if err != nil {
			err = fmt.Errorf("error running hook: %w, stdout: %s, stderr: %s", err, stdout.String(), stderr.String())
		}
		errC <- err
	}()
	var timerCh <-chan time.Time
	if c.Timeout != nil {
		timer := time.NewTimer(*c.Timeout)
		defer timer.Stop()
		timerCh = timer.C
	}
	select {
	case err := <-errC:
		return err
	case <-timerCh:
		_ = cmd.Process.Kill()
		<-errC
		return fmt.Errorf("hook ran past specified timeout of %.1fs", c.Timeout.Seconds())
	}
}
```



相当于使用命令行启动 `nerdctl internal oci-hook createRuntime`

{{< admonition info >}}
从这里也可以看出，`runc`的 `createRuntime hook`，使用的是直接调用外部二进制文件的方式
{{< /admonition >}}

查看nerdctl中的 `internal oci-hook createRuntime`相关代码：

```go
func newInternalCommand() *cobra.Command {
	var internalCommand = &cobra.Command{
		Use:           "internal",
		Short:         "DO NOT EXECUTE MANUALLY",
		Hidden:        true,
		SilenceUsage:  true,
		SilenceErrors: true,
	}

	internalCommand.AddCommand(
		newInternalOCIHookCommandCommand(),
	)

	return internalCommand
}
func newInternalOCIHookCommandCommand() *cobra.Command {
	var internalOCIHookCommand = &cobra.Command{
		Use:           "oci-hook",
		Short:         "OCI hook",
		RunE:          internalOCIHookAction,
		SilenceUsage:  true,
		SilenceErrors: true,
	}
	return internalOCIHookCommand
}
func internalOCIHookAction(cmd *cobra.Command, args []string) error {
	event := ""
	if len(args) > 0 {
		event = args[0]
	}
	if event == "" {
		return errors.New("event type needs to be passed")
	}
	dataStore, err := getDataStore(cmd)
	if err != nil {
		return err
	}
	cniPath, err := cmd.Flags().GetString("cni-path")
	if err != nil {
		return err
	}
	cniNetconfpath, err := cmd.Flags().GetString("cni-netconfpath")
	if err != nil {
		return err
	}
	return ocihook.Run(os.Stdin, os.Stderr, event,
		dataStore,
		cniPath,
		cniNetconfpath,
	)
}
func Run(stdin io.Reader, stderr io.Writer, event, dataStore, cniPath, cniNetconfPath string) error {
	if stdin == nil || event == "" || dataStore == "" || cniPath == "" || cniNetconfPath == "" {
		return errors.New("got insufficient args")
	}

	var state specs.State
	if err := json.NewDecoder(stdin).Decode(&state); err != nil {
		return err
	}

	containerStateDir := state.Annotations[labels.StateDir]
	if containerStateDir == "" {
		return errors.New("state dir must be set")
	}
	if err := os.MkdirAll(containerStateDir, 0700); err != nil {
		return fmt.Errorf("failed to create %q: %w", containerStateDir, err)
	}
	logFilePath := filepath.Join(containerStateDir, "oci-hook."+event+".log")
	logFile, err := os.Create(logFilePath)
	if err != nil {
		return err
	}
	defer logFile.Close()
	logrus.SetOutput(io.MultiWriter(stderr, logFile))

	opts, err := newHandlerOpts(&state, dataStore, cniPath, cniNetconfPath)
	if err != nil {
		return err
	}

	switch event {
	case "createRuntime":
		return onCreateRuntime(opts)
	case "postStop":
		return onPostStop(opts)
	default:
		return fmt.Errorf("unexpected event %q", event)
	}
}
```

```go
func onCreateRuntime(opts *handlerOpts) error {
	loadAppArmor()

	if opts.cni != nil {
		portMapOpts, err := getPortMapOpts(opts)
		if err != nil {
			return err
		}
		nsPath, err := getNetNSPath(opts.state)
		if err != nil {
			return err
		}
		ctx := context.Background()
		hs, err := hostsstore.NewStore(opts.dataStore)
		if err != nil {
			return err
		}
		ipAddressOpts, err := getIPAddressOpts(opts)
		if err != nil {
			return err
		}
		macAddressOpts, err := getMACAddressOpts(opts)
		if err != nil {
			return err
		}
		var namespaceOpts []gocni.NamespaceOpts
		namespaceOpts = append(namespaceOpts, portMapOpts...)
		namespaceOpts = append(namespaceOpts, ipAddressOpts...)
		namespaceOpts = append(namespaceOpts, macAddressOpts...)
		hsMeta := hostsstore.Meta{
			Namespace:  opts.state.Annotations[labels.Namespace],
			ID:         opts.state.ID,
			Networks:   make(map[string]*types100.Result, len(opts.cniNames)),
			Hostname:   opts.state.Annotations[labels.Hostname],
			ExtraHosts: opts.extraHosts,
			Name:       opts.state.Annotations[labels.Name],
		}
		cniRes, err := opts.cni.Setup(ctx, opts.fullID, nsPath, namespaceOpts...)
		if err != nil {
			return fmt.Errorf("failed to call cni.Setup: %w", err)
		}
		cniResRaw := cniRes.Raw()
		for i, cniName := range opts.cniNames {
			hsMeta.Networks[cniName] = cniResRaw[i]
		}

		b4nnEnabled, err := bypass4netnsutil.IsBypass4netnsEnabled(opts.state.Annotations)
		if err != nil {
			return err
		}

		if err := hs.Acquire(hsMeta); err != nil {
			return err
		}

		if rootlessutil.IsRootlessChild() {
			if b4nnEnabled {
				bm, err := bypass4netnsutil.NewBypass4netnsCNIBypassManager(opts.bypassClient, opts.rootlessKitClient)
				if err != nil {
					return err
				}
				err = bm.StartBypass(ctx, opts.ports, opts.state.ID, opts.state.Annotations[labels.StateDir])
				if err != nil {
					return fmt.Errorf("bypass4netnsd not running? (Hint: run `containerd-rootless-setuptool.sh install-bypass4netnsd`): %w", err)
				}
			} else if len(opts.ports) > 0 {
				pm, err := rootlessutil.NewRootlessCNIPortManager(opts.rootlessKitClient)
				if err != nil {
					return err
				}
				for _, p := range opts.ports {
					if err := pm.ExposePort(ctx, p); err != nil {
						return err
					}
				}
			}
		}
	}
	return nil
}
```

在 `opts.cni.Setup `中进行网络配置。





<br>

## kubectl run nginx --image=nginx


