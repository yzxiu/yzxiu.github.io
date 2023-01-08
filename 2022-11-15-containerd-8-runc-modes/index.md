# Containerd解析(8) - Runc Modes


## 概述

在[runc](https://github.com/opencontainers/runc/blob/main/docs/terminals.md)中，

有两种 [Terminal Modes](https://github.com/opencontainers/runc/blob/main/docs/terminals.md#-terminal-modes)，分别是：[New Terminal](https://github.com/opencontainers/runc/blob/main/docs/terminals.md#-new-terminal)、[Pass-Through](https://github.com/opencontainers/runc/blob/main/docs/terminals.md#-pass-through)

有两种 [Runc Modes](https://github.com/opencontainers/runc/blob/main/docs/terminals.md#-runc-modes)，分别是：[Foreground](https://github.com/opencontainers/runc/blob/main/docs/terminals.md#foreground)、[Detached](https://github.com/opencontainers/runc/blob/main/docs/terminals.md#detached)



接下来，通过runc中的代码，了解这几中 modes。



## setupIO

```go
// setupIO modifies the given process config according to the options.
// Terminal Modes:
//   New Terminal
//   Pass-Through
// runc Modes:
//   Foreground
//   Detached
func setupIO(process *libcontainer.Process, rootuid, rootgid int, createTTY, detach bool, sockpath string) (*tty, error) {
	if createTTY {
		process.Stdin = nil
		process.Stdout = nil
		process.Stderr = nil
		t := &tty{}
		if !detach {
			// 1, New Terminal & Foreground
			if err := t.initHostConsole(); err != nil {
				return nil, err
			}
			parent, child, err := utils.NewSockPair("console")
			if err != nil {
				return nil, err
			}
			process.ConsoleSocket = child
			t.postStart = append(t.postStart, parent, child)
			t.consoleC = make(chan error, 1)
			go func() {
				t.consoleC <- t.recvtty(parent)
			}()
		} else {
			// 2, New Terminal & Detached
			// the caller of runc will handle receiving the console master
			conn, err := net.Dial("unix", sockpath)
			if err != nil {
				return nil, err
			}
			uc, ok := conn.(*net.UnixConn)
			if !ok {
				return nil, errors.New("casting to UnixConn failed")
			}
			t.postStart = append(t.postStart, uc)
			socket, err := uc.File()
			if err != nil {
				return nil, err
			}
			t.postStart = append(t.postStart, socket)
			process.ConsoleSocket = socket
		}
		return t, nil
	}
	// when runc will detach the caller provides the stdio to runc via runc's 0,1,2
	// and the container's process inherits runc's stdio.
	// 3, Pass-Through & Detached
	if detach {
		inheritStdio(process)
		return &tty{}, nil
	}

	// 4, Pass-Through & Foreground
	return setupProcessPipes(process, rootuid, rootgid)
}
```





### New Terminal & Foreground

```go
process.Stdin = nil
process.Stdout = nil
process.Stderr = nil
t := &tty{}
		
// 1, New Terminal & Foreground
if err := t.initHostConsole(); err != nil {
	return nil, err
}
parent, child, err := utils.NewSockPair("console")
if err != nil {
	return nil, err
}
process.ConsoleSocket = child
t.postStart = append(t.postStart, parent, child)
t.consoleC = make(chan error, 1)
go func() {
	t.consoleC <- t.recvtty(parent)
}()
```

```bash
runc --root \
/run/containerd/runc/default \
run \
-b \
/home/xiu/CloudDrive/runcdev/mycontainer-terminal \
demo
```

`Foreground`模式下，runc进程会一直保留在前台，不会退出。

这里调用 t.recvtty(parent)

```go
func (t *tty) recvtty(socket *os.File) (Err error) {
   f, err := utils.RecvFd(socket)
   if err != nil {
      return err
   }
   cons, err := console.ConsoleFromFile(f)
   if err != nil {
      return err
   }
   err = console.ClearONLCR(cons.Fd())
   if err != nil {
      return err
   }
   epoller, err := console.NewEpoller()
   if err != nil {
      return err
   }
   epollConsole, err := epoller.Add(cons)
   if err != nil {
      return err
   }
   defer func() {
      if Err != nil {
         _ = epollConsole.Close()
      }
   }()
   go func() { _ = epoller.Wait() }()
   go func() { _, _ = io.Copy(epollConsole, os.Stdin) }()
   t.wg.Add(1)
   go t.copyIO(os.Stdout, epollConsole)

   // Set raw mode for the controlling terminal.
   if err := t.hostConsole.SetRaw(); err != nil {
      return fmt.Errorf("failed to set the terminal from the stdin: %w", err)
   }
   go handleInterrupt(t.hostConsole)

   t.epoller = epoller
   t.console = epollConsole
   t.closers = []io.Closer{epollConsole}
   return nil
}
```

上面 init 之前，

下面是 init 过程，

```go
// 组装command 模版，生成init 命令，设置大量以 _xxx 这样格式的环境变量，即大量的extraFiles 文件
cmd := c.commandTemplate(p, childInitPipe, childLogPipe)
```

```go
func (c *Container) commandTemplate(p *Process, childInitPipe *os.File, childLogPipe *os.File) *exec.Cmd {
	cmd := exec.Command("/proc/self/exe", "init")
	cmd.Args[0] = os.Args[0]
    // 当  New Terminal 模式时
    // p.Stdin、p.Stdout、p.Stderr都被设置为nil
	cmd.Stdin = p.Stdin
	cmd.Stdout = p.Stdout
	cmd.Stderr = p.Stderr
	cmd.Dir = c.config.Rootfs
	if cmd.SysProcAttr == nil {
		cmd.SysProcAttr = &unix.SysProcAttr{}
	}
	cmd.Env = append(cmd.Env, "GOMAXPROCS="+os.Getenv("GOMAXPROCS"))
	cmd.ExtraFiles = append(cmd.ExtraFiles, p.ExtraFiles...)
	if p.ConsoleSocket != nil {
		cmd.ExtraFiles = append(cmd.ExtraFiles, p.ConsoleSocket)
		cmd.Env = append(cmd.Env,
			"_LIBCONTAINER_CONSOLE="+strconv.Itoa(stdioFdCount+len(cmd.ExtraFiles)-1),
		)
	}
	cmd.ExtraFiles = append(cmd.ExtraFiles, childInitPipe)
	cmd.Env = append(cmd.Env,
		"_LIBCONTAINER_INITPIPE="+strconv.Itoa(stdioFdCount+len(cmd.ExtraFiles)-1),
		"_LIBCONTAINER_STATEDIR="+c.root,
	)

	cmd.ExtraFiles = append(cmd.ExtraFiles, childLogPipe)
	cmd.Env = append(cmd.Env,
		"_LIBCONTAINER_LOGPIPE="+strconv.Itoa(stdioFdCount+len(cmd.ExtraFiles)-1),
		"_LIBCONTAINER_LOGLEVEL="+p.LogLevel,
	)

	// NOTE: when running a container with no PID namespace and the parent process spawning the container is
	// PID1 the pdeathsig is being delivered to the container's init process by the kernel for some reason
	// even with the parent still running.
	if c.config.ParentDeathSignal > 0 {
		cmd.SysProcAttr.Pdeathsig = unix.Signal(c.config.ParentDeathSignal)
	}
	return cmd
}
```



```go
// runc/libcontainer/process_linux.go
err := p.cmd.Start()
```

![image-20221116105003022](https://raw.githubusercontent.com/yzxiu/images/blog/2022-11/20221116-105003.png "init process")





容器初始化过程中，查看父进程发送给子进程的初始化数据：

```go
// runc/libcontainer/process_linux.go
func (p *initProcess) sendConfig() error {
	// send the config to the container's init process, we don't use JSON Encode
	// here because there might be a problem in JSON decoder in some cases, see:
	// https://github.com/docker/docker/issues/14203#issuecomment-174177790
	return utils.WriteJSON(p.messageSockPair.parent, p.config)
}
```

![image-20221116105320570](https://raw.githubusercontent.com/yzxiu/images/blog/2022-11/20221116-105321.png "p.config")



runc init 中：

```go
// 设置console
if l.config.CreateConsole {
   if err := setupConsole(l.consoleSocket, l.config, true); err != nil {
      return err
   }
   if err := system.Setctty(); err != nil {
      return err
   }
}
```

```go
// setupConsole sets up the console from inside the container, and sends the
// master pty fd to the config.Pipe (using cmsg). This is done to ensure that
// consoles are scoped to a container properly (see runc#814 and the many
// issues related to that). This has to be run *after* we've pivoted to the new
// rootfs (and the users' configuration is entirely set up).
func setupConsole(socket *os.File, config *initConfig, mount bool) error {
	defer socket.Close()
	// At this point, /dev/ptmx points to something that we would expect. We
	// used to change the owner of the slave path, but since the /dev/pts mount
	// can have gid=X set (at the users' option). So touching the owner of the
	// slave PTY is not necessary, as the kernel will handle that for us. Note
	// however, that setupUser (specifically fixStdioPermissions) *will* change
	// the UID owner of the console to be the user the process will run as (so
	// they can actually control their console).

	pty, slavePath, err := console.NewPty()
	if err != nil {
		return err
	}

	// After we return from here, we don't need the console anymore.
	defer pty.Close()

	if config.ConsoleHeight != 0 && config.ConsoleWidth != 0 {
		err = pty.Resize(console.WinSize{
			Height: config.ConsoleHeight,
			Width:  config.ConsoleWidth,
		})

		if err != nil {
			return err
		}
	}

	// Mount the console inside our rootfs.
	if mount {
		if err := mountConsole(slavePath); err != nil {
			return err
		}
	}
	// While we can access console.master, using the API is a good idea.
	if err := utils.SendFd(socket, pty.Name(), pty.Fd()); err != nil {
		return err
	}
	// Now, dup over all the things.
	return dupStdio(slavePath)
}
```

```go
// dupStdio opens the slavePath for the console and dups the fds to the current
// processes stdio, fd 0,1,2.
func dupStdio(slavePath string) error {
   fd, err := unix.Open(slavePath, unix.O_RDWR, 0)
   if err != nil {
      return &os.PathError{
         Op:   "open",
         Path: slavePath,
         Err:  err,
      }
   }
   // 打开控制台的 slavePath 并将 fds 复制到当前进程 stdio，fd 0,1,2。
   // 这里的 0 1 2 代表容器进程的标准输入，标准输出，标准错误
   // 如果这里把 1 2 去掉，那在前台运行的时候，将看不到容器输出内容
   for _, i := range []int{0, 1, 2} {
      if err := unix.Dup3(fd, i, 0); err != nil {
         return err
      }
   }
   return nil
}
```









### New Terminal & Detached

```go
// the caller of runc will handle receiving the console master
conn, err := net.Dial("unix", sockpath)
if err != nil {
	return nil, err
}
uc, ok := conn.(*net.UnixConn)
if !ok {
	return nil, errors.New("casting to UnixConn failed")
}
t.postStart = append(t.postStart, uc)
socket, err := uc.File()
if err != nil {
	return nil, err
}
t.postStart = append(t.postStart, socket)
process.ConsoleSocket = socket
```









### Pass-Through & Detached

```go
if detach {
 inheritStdio(process)
 return &tty{}, nil
}
```

```go
func inheritStdio(process *libcontainer.Process) {
	process.Stdin = os.Stdin
	process.Stdout = os.Stdout
	process.Stderr = os.Stderr
}
```







### Pass-Through & Foreground

略






