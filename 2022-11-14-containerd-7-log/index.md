# Containerd解析(7) - log


## 概述

了解 nerdctl 是怎样处理容器日志的，后续再分析kubelet的处理方式。









## nerdctl 中的日志处理

容器日志处理，其实就是处理runc程序的标准输出，标准错误。

分析两种常用的情况。

### nerdctl run -t --name runcdev1 q946666800/runcdev

runc启动命令：

![image-20221116152106606](https://raw.githubusercontent.com/yzxiu/images/blog/2022-11/20221116-152107.png "runc启动命令")

查看config.json配置

![image-20221116152354617](https://raw.githubusercontent.com/yzxiu/images/blog/2022-11/20221116-152355.png "terminal: true")

可知该模式为 `New Terminal & Detached` ，需要外部传入 `--console-socket`

从runc启动参数可以看到

```
12 = {string} "--console-socket"
13 = {string} "/run/user/1000/pty4072274856/pty.sock"
```



查看如何创建并监听 `/run/user/1000/pty4072274856/pty.sock` ？？？

查看代码，可以看到在shim的 runc.NewContainer()  ->  p.Create() 代码中，创建了 pty.sock

```go
if r.Terminal {
	if socket, err = runc.NewTempConsoleSocket(); err != nil {
		return fmt.Errorf("failed to create OCI runtime console socket: %w", err)
	}
	defer socket.Close()
} else {
	if pio, err = createIO(ctx, p.id, p.IoUID, p.IoGID, p.stdio); err != nil {
		return fmt.Errorf("failed to create init process I/O: %w", err)
	}
	p.io = pio
}
```

NewTempConsoleSocket()创建了 pty.sock，并将sock的路径参数传递给runc。



接下来查看 shim 创建的`pty.sock`怎样与`nerdctl`关联起来？？？

查看 `nerdctl run -it` 相关代码，

```go
// 
var con console.Console
if flagT {
   con = console.Current()
   defer con.Reset()
   if err := con.SetRaw(); err != nil {
      return err
   }
}
```

```go
// nerdctl/pkg/taskutil/taskutil.go
// 注意， cio.WithStreams(in, con, nil)这里传入了 in，con
// con是nertctl进程本身的终端。
ioCreator = cio.NewCreator(cio.WithStreams(in, con, nil), cio.WithTerminal)


// WithTerminal sets the terminal option
func WithTerminal(opt *Streams) {
	opt.Terminal = true
}
// WithStreams sets the stream options to the specified Reader and Writers
func WithStreams(stdin io.Reader, stdout, stderr io.Writer) Opt {
	return func(opt *Streams) {
		opt.Stdin = stdin
		opt.Stdout = stdout
		opt.Stderr = stderr
	}
}

// NewCreator returns an IO creator from the options
func NewCreator(opts ...Opt) Creator {
	streams := &Streams{}
	for _, opt := range opts {
		opt(streams)
	}
	if streams.FIFODir == "" {
		streams.FIFODir = defaults.DefaultFIFODir
	}
    // 所以，在这里的时候
	return func(id string) (IO, error) {
        // 将 stdio 转换为字符串路径
		fifos, err := NewFIFOSetInDir(streams.FIFODir, id, streams.Terminal)
		if err != nil {
			return nil, err
		}
		if streams.Stdin == nil {
			fifos.Stdin = ""
		}
		if streams.Stdout == nil {
			fifos.Stdout = ""
		}
		if streams.Stderr == nil {
			fifos.Stderr = ""
		}
		return copyIO(fifos, streams)
	}
}
```

最终在 copyIO 中，会创建 /run/containerd/fifo/392418153 路径下创建相关sock，并与nerdctl终端相关联，如下：

```
root@xiu-desktop:/run/containerd/fifo/7416553# tree
.
├── 2e8e9f34cedc5bcca69af8e0e98afae4452463a3b5094b03b71a970f9d88eefa-stdin
└── 2e8e9f34cedc5bcca69af8e0e98afae4452463a3b5094b03b71a970f9d88eefa-stdout
```

随后，nerdctl将这两个路径传递给containerd，containerd传递给shim。

在shim中，接收到的请求如下：

![image-20221116210237546](https://raw.githubusercontent.com/yzxiu/images/blog/2022-11/20221116-210238.png)



```go
if socket != nil {
   console, err := socket.ReceiveMaster()
   if err != nil {
      return fmt.Errorf("failed to retrieve console master: %w", err)
   }
   console, err = p.Platform.CopyConsole(ctx, console, p.id, r.Stdin, r.Stdout, r.Stderr, &p.wg)
   if err != nil {
      return fmt.Errorf("failed to start console copy: %w", err)
   }
   p.console = console
} else {
   if err := pio.Copy(ctx, &p.wg); err != nil {
      return fmt.Errorf("failed to start io pipe copy: %w", err)
   }
}
```

```go
func (p *linuxPlatform) CopyConsole(ctx context.Context, console console.Console, id, stdin, stdout, stderr string, wg *sync.WaitGroup) (cons c驱动onsole.Console, retErr error) {
   if p.epoller == nil {
      return nil, errors.New("uninitialized epoller")
   }

   epollConsole, err := p.epoller.Add(console)
   if err != nil {
      return nil, err
   }

   var cwg sync.WaitGroup
   if stdin != "" {
      in, err := fifo.OpenFifo(context.Background(), stdin, syscall.O_RDONLY|syscall.O_NONBLOCK, 0)
      if err != nil {
         return nil, err
      }
      cwg.Add(1)
      go func() {
         cwg.Done()
         bp := bufPool.Get().(*[]byte)
         defer bufPool.Put(bp)
         io.CopyBuffer(epollConsole, in, *bp)
         // we need to shutdown epollConsole when pipe broken
         epollConsole.Shutdown(p.epoller.CloseConsole)
         epollConsole.Close()
      }()
   }

   uri, err := url.Parse(stdout)
   if err != nil {
      return nil, fmt.Errorf("unable to parse stdout uri: %w", err)
   }

   // uri.Scheme = ""
   switch uri.Scheme {
   case "binary":
      ...
      ...

   default:
      // 
      outw, err := fifo.OpenFifo(ctx, stdout, syscall.O_WRONLY, 0)
      if err != nil {
         return nil, err
      }
      outr, err := fifo.OpenFifo(ctx, stdout, syscall.O_RDONLY, 0)
      if err != nil {
         return nil, err
      }
      wg.Add(1)
      cwg.Add(1)
      go func() {
         cwg.Done()
         buf := bufPool.Get().(*[]byte)
         defer bufPool.Put(buf)
         io.CopyBuffer(outw, epollConsole, *buf)

         outw.Close()
         outr.Close()
         wg.Done()
      }()
      cwg.Wait()
   }

   return epollConsole, nil
}
```

在shim中，通过 传递进来的 

![image-20221116210506511](https://raw.githubusercontent.com/yzxiu/images/blog/2022-11/20221116-210507.png)

与创建socket相关联，将容器的 console 传输至 nerdctl 进程 console。

所以，使用 -it 运行容器时，可以与容器进程直接进行交互。

<br>

### nerdctl run -d --name runcdev1 q946666800/runcdev

runc启动方式为：`Pass-Through & Detached`

首先，查看shim，create的请求参数：

![image-20221116211250529](https://raw.githubusercontent.com/yzxiu/images/blog/2022-11/20221116-211251.png)

{{< admonition info >}}

注意，这里的 Stdio 与上面的区别。

在 `-it` 模式中，有Stdin, Stdout,没有Stderr，是因为我们使用终端模式运行，需要输入，并且Stdout和Stderr会统一输出到终端，所以合并在Stdout中

在`-d`模式中，因为进程是后台运行，所以不需要Stdin，Stdout和Stderr可以分开传输。

{{< /admonition >}}

```go
if r.Terminal {
   if socket, err = runc.NewTempConsoleSocket(); err != nil {
      return fmt.Errorf("failed to create OCI runtime console socket: %w", err)
   }
   defer socket.Close()
} else {
   if pio, err = createIO(ctx, p.id, p.IoUID, p.IoGID, p.stdio); err != nil {
      return fmt.Errorf("failed to create init process I/O: %w", err)
   }
   p.io = pio
}
```

这回执行的是 `createIO`

```go
func createIO(ctx context.Context, id string, ioUID, ioGID int, stdio stdio.Stdio) (*processIO, error) {
	pio := &processIO{
		stdio: stdio,
	}
	if stdio.IsNull() {
		i, err := runc.NewNullIO()
		if err != nil {
			return nil, err
		}
		pio.io = i
		return pio, nil
	}
	u, err := url.Parse(stdio.Stdout)
	if err != nil {
		return nil, fmt.Errorf("unable to parse stdout uri: %w", err)
	}
	if u.Scheme == "" {
		u.Scheme = "fifo"
	}
	pio.uri = u
    // u.scheme = "binary"
	switch u.Scheme {
	case "fifo":
		pio.copy = true
		pio.io, err = runc.NewPipeIO(ioUID, ioGID, withConditionalIO(stdio))
	case "binary":
		pio.io, err = NewBinaryIO(ctx, id, u)
	case "file":
		filePath := u.Path
		if err := os.MkdirAll(filepath.Dir(filePath), 0755); err != nil {
			return nil, err
		}
		var f *os.File
		f, err = os.OpenFile(filePath, os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
		if err != nil {
			return nil, err
		}
		f.Close()
		pio.stdio.Stdout = filePath
		pio.stdio.Stderr = filePath
		pio.copy = true
		pio.io, err = runc.NewPipeIO(ioUID, ioGID, withConditionalIO(stdio))
	default:
		return nil, fmt.Errorf("unknown STDIO scheme %s", u.Scheme)
	}
	if err != nil {
		return nil, err
	}
	return pio, nil
}
```

```go
// NewBinaryIO runs a custom binary process for pluggable shim logging
func NewBinaryIO(ctx context.Context, id string, uri *url.URL) (_ runc.IO, err error) {
   ns, err := namespaces.NamespaceRequired(ctx)
   if err != nil {
      return nil, err
   }

   var closers []func() error
   defer func() {
      if err == nil {
         return
      }
      result := multierror.Append(err)
      for _, fn := range closers {
         result = multierror.Append(result, fn())
      }
      err = multierror.Flatten(result)
   }()

   out, err := newPipe()
   if err != nil {
      return nil, fmt.Errorf("failed to create stdout pipes: %w", err)
   }
   closers = append(closers, out.Close)

   serr, err := newPipe()
   if err != nil {
      return nil, fmt.Errorf("failed to create stderr pipes: %w", err)
   }
   closers = append(closers, serr.Close)

   r, w, err := os.Pipe()
   if err != nil {
      return nil, err
   }
   closers = append(closers, r.Close, w.Close)

   // 1, 配置日志驱动子进程
   cmd := NewBinaryCmd(uri, id, ns)
   // 2, 配置子进程额外文件描述符
   cmd.ExtraFiles = append(cmd.ExtraFiles, out.r, serr.r, w)
   // don't need to register this with the reaper or wait when
   // running inside a shim
   if err := cmd.Start(); err != nil {
      return nil, fmt.Errorf("failed to start binary process: %w", err)
   }
   closers = append(closers, func() error { return cmd.Process.Kill() })

   // close our side of the pipe after start
   if err := w.Close(); err != nil {
      return nil, fmt.Errorf("failed to close write pipe after start: %w", err)
   }

   // wait for the logging binary to be ready
   b := make([]byte, 1)
   if _, err := r.Read(b); err != nil && err != io.EOF {
      return nil, fmt.Errorf("failed to read from logging binary: %w", err)
   }

   return &binaryIO{
      cmd: cmd,
      out: out,
      err: serr,
   }, nil
}
```

日志驱动cmd信息如下：

![image-20221116214343939](https://raw.githubusercontent.com/yzxiu/images/blog/2022-11/20221116-214344.png)

```GO
if opts != nil && opts.IO != nil {
	opts.Set(cmd)
}

// 将上面的文件描述符，传递给容器进程
func (b *binaryIO) Set(cmd *exec.Cmd) {
	if b.out != nil {
		cmd.Stdout = b.out.w
	}
	if b.err != nil {
		cmd.Stderr = b.err.w
	}
}
```

查看日志驱动，

```go
// nerdctl/pkg/logging/logging.go
func Main(argv2 string) error {
	fn, err := getLoggerFunc(argv2)
	if err != nil {
		return err
	}
    
	logging.Run(fn)
	return nil
}
```

所谓日志驱动，就是编写处理日志的`fn`函数，然后传递给 `logging.Run`

```go
// github.com/containerd/containerd@v1.7.0-beta.0/runtime/v2/logging/logging_unix.go
config := &Config{
	ID:        os.Getenv("CONTAINER_ID"),
	Namespace: os.Getenv("CONTAINER_NAMESPACE"),
	Stdout:    os.NewFile(3, "CONTAINER_STDOUT"),
	Stderr:    os.NewFile(4, "CONTAINER_STDERR"),
}
var (
	sigCh = make(chan os.Signal, 32)
	errCh = make(chan error, 1)
	wait  = os.NewFile(5, "CONTAINER_WAIT")
)
signal.Notify(sigCh, unix.SIGTERM)
go func() {
	errCh <- fn(ctx, config, wait.Close)
}()
// 通过文件描述符获取 shim 传递过来的文件句柄。
// 然后将配置传递至 自定义的fn函数。查看fn
```

```go

// nerdctl/pkg/logging/logging.go
func getLoggerFunc(dataStore string) (logging.LoggerFunc, error) {
	if dataStore == "" {
		return nil, errors.New("got empty data store")
	}
	return func(_ context.Context, config *logging.Config, ready func() error) error {
		if config.Namespace == "" || config.ID == "" {
			return errors.New("got invalid config")
		}
		logConfigFilePath := LogConfigFilePath(dataStore, config.Namespace, config.ID)
		if _, err := os.Stat(logConfigFilePath); err == nil {
			logConfig, err := LoadLogConfig(dataStore, config.Namespace, config.ID)
			if err != nil {
				return err
			}
			driver, err := GetDriver(logConfig.Driver, logConfig.Opts)
			if err != nil {
				return err
			}
			if err := ready(); err != nil {
				return err
			}
			return driver.Process(dataStore, config)
		} else if !errors.Is(err, os.ErrNotExist) {
			// the file does not exist if the container was created with nerdctl < 0.20
			return err
		}
		return nil
	}, nil
}
```

```go
func (jsonLogger *JSONLogger) Process(dataStore string, config *logging.Config) error {
	var jsonFilePath string
	if logPath, ok := jsonLogger.Opts[LogPath]; ok {
		jsonFilePath = logPath
	} else {
		jsonFilePath = jsonfile.Path(dataStore, config.Namespace, config.ID)
	}
	l := &logrotate.Logger{
		Filename: jsonFilePath,
	}
	//maxSize Defaults to unlimited.
	var capVal int64
	capVal = -1
	if capacity, ok := jsonLogger.Opts[MaxSize]; ok {
		var err error
		capVal, err = units.FromHumanSize(capacity)
		if err != nil {
			return err
		}
		if capVal <= 0 {
			return fmt.Errorf("max-size must be a positive number")
		}
	}
	l.MaxBytes = capVal
	maxFile := 1
	if maxFileString, ok := jsonLogger.Opts[MaxFile]; ok {
		var err error
		maxFile, err = strconv.Atoi(maxFileString)
		if err != nil {
			return err
		}
		if maxFile < 1 {
			return fmt.Errorf("max-file cannot be less than 1")
		}
	}
	// MaxBackups does not include file to write logs to
	l.MaxBackups = maxFile - 1
	return jsonfile.Encode(l, config.Stdout, config.Stderr)
}
```

最终的日志处理，是使用 `jsonfile` 框架，将日志保存到 `jsonFilePath` 路径中。



### nerdctl-log-example

下面通过简单的代码，模拟 `nerdctl run -d --name runcdev1 q946666800/runcdev`日志的搜集过程。

```go
// app.go
package main

import (
	"fmt"
	"time"
)

func main() {
	i := 5
	for i > 0 {
		fmt.Println(time.Now())
		time.Sleep(time.Second)
		i--
	}
}
```

```go
// drive.go
package main

import (
   "bufio"
   "context"
   "encoding/json"
   "fmt"
   "io"
   "os"
   "os/signal"
   "sync"
   "time"

   "github.com/fahedouch/go-logrotate"
   "github.com/sirupsen/logrus"
   "golang.org/x/sys/unix"
)

func main() {

   fmt.Println("log drive start!!!")

   ctx, cancel := context.WithCancel(context.Background())
   defer cancel()

   var (
      sigCh = make(chan os.Signal, 32)
      errCh = make(chan error, 1)
   )
   signal.Notify(sigCh, unix.SIGTERM)

   // cmd.ExtraFiles = append(cmd.ExtraFiles, out.r, serr.r, w)
   out := os.NewFile(3, "CONTAINER_STDOUT")
   serr := os.NewFile(4, "CONTAINER_STDERR")
   wait := os.NewFile(5, "CONTAINER_WAIT")

   go func() {
      errCh <- logger(ctx, out, serr, wait.Close)
   }()

   for {
      select {
      case <-sigCh:
         cancel()
      case err := <-errCh:
         if err != nil {
            fmt.Fprintln(os.Stderr, err)
            os.Exit(1)
         }
         fmt.Println("log drive exit 0")
         os.Exit(0)
      }
   }

}

func logger(_ context.Context, out *os.File, serr *os.File, ready func() error) error {

   // Notify the shim that it is ready
   // call wait.Close
   // r will receive io.EOF error
   if err := ready(); err != nil {
      return err
   }

   // log path
   jsonFilePath := "app.log"
   l := &logrotate.Logger{
      Filename: jsonFilePath,
   }
   return Encode(l, out, serr)
}

// Entry is compatible with Docker "json-file" logs
type Entry struct {
   Log    string    `json:"log,omitempty"`    // line, including "\r\n"
   Stream string    `json:"stream,omitempty"` // "stdout" or "stderr"
   Time   time.Time `json:"time"`             // e.g. "2020-12-11T20:29:41.939902251Z"
}

func Encode(w io.WriteCloser, stdout, stderr io.Reader) error {
   enc := json.NewEncoder(w)
   var encMu sync.Mutex
   var wg sync.WaitGroup
   wg.Add(2)
   f := func(r io.Reader, name string) {
      defer wg.Done()
      br := bufio.NewReader(r)
      e := &Entry{
         Stream: name,
      }
      for {
         line, err := br.ReadString(byte('\n'))
         if err != nil {
            logrus.WithError(err).Errorf("failed to read line from %q", name)
            return
         }
         e.Log = line
         e.Time = time.Now().UTC()
         encMu.Lock()
         encErr := enc.Encode(e)
         encMu.Unlock()
         if encErr != nil {
            logrus.WithError(err).Errorf("failed to encode JSON")
            return
         }
      }
   }
   go f(stdout, "stdout")
   go f(stderr, "stderr")
   wg.Wait()
   return nil
}
```

```go
// shim.go
package main

import (
	"fmt"
	"io"
	"log"
	"os"
	"os/exec"
)

func main() {

	// start log driver
	pio, err := driveIO()
	if err != nil {
		log.Fatal(err)
	}

	// start app
	cmd := exec.Command("./app-example")
	cmd.Stdout = pio.out.w
	cmd.Stderr = pio.err.w

	err = cmd.Start()
	if err != nil {
		log.Fatal(err)
	}

	err = cmd.Wait()
	if err != nil {
		log.Fatal(err)
	}
}

func newPipe() (*pipe, error) {
	r, w, err := os.Pipe()
	if err != nil {
		return nil, err
	}
	return &pipe{
		r: r,
		w: w,
	}, nil
}

type pipe struct {
	r *os.File
	w *os.File
}

type binaryIO struct {
	cmd      *exec.Cmd
	out, err *pipe
}

func (p *pipe) Close() error {
	if err := p.w.Close(); err != nil {
	}
	if err := p.r.Close(); err != nil {
	}
	return fmt.Errorf("pipe close error")
}

func driveIO() (_ *binaryIO, err error) {

	var closers []func() error

	// app out pipe
	out, err := newPipe()
	if err != nil {
		return nil, err
	}
	closers = append(closers, out.Close)

	// app err pipe
	serr, err := newPipe()
	if err != nil {
		return nil, err
	}
	closers = append(closers, serr.Close)

	// drive ready pipe
	r, w, err := os.Pipe()
	if err != nil {
		return nil, err
	}
	closers = append(closers, r.Close, w.Close)

	cmd := exec.Command("./drive-example")
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	cmd.ExtraFiles = append(cmd.ExtraFiles, out.r, serr.r, w)

	if err := cmd.Start(); err != nil {
		return nil, err
	}
	closers = append(closers, func() error { return cmd.Process.Kill() })

	// close our side of the pipe after start
	if err := w.Close(); err != nil {
		return nil, fmt.Errorf("failed to close write pipe after start: %w", err)
	}

	// wait for the logging binary to be ready
	b := make([]byte, 1)
	if _, err := r.Read(b); err != nil && err != io.EOF {
		return nil, fmt.Errorf("failed to read from logging binary: %w", err)
	}

	return &binaryIO{
		cmd: cmd,
		out: out,
		err: serr,
	}, nil
}

```



模拟 `nerdctl run -d --name runcdev1 q946666800/runcdev` 的日志收集过程。

运行 `./shim-example`,
首先启动驱动程序，返回相关pio，
```go
pio, err := driveIO()
if err != nil {
   log.Fatal(err)
}
```

配置应用程序，将stdio传递给上面的pio
```go
// start app
cmd := exec.Command("./app-example")
cmd.Stdout = pio.out.w
cmd.Stderr = pio.err.w
```

启动应用程序，日志驱动将应用程序日志，以 `json` 的形式，写入到 `app.log` 文件中。

```
[xiu-desktop] nerdctl-log-example git:(master) ./shim-example
log drive start!!!
ERRO[0005] failed to read line from "stderr"             error=EOF
ERRO[0005] failed to read line from "stdout"             error=EOF
log drive exit 0
[xiu-desktop] nerdctl-log-example git:(master) cat app.log         
{"log":"2022-11-16 22:32:15.465213291 +0800 CST m=+0.000014187\n","stream":"stdout","time":"2022-11-16T14:32:15.46528788Z"}
{"log":"2022-11-16 22:32:16.466022681 +0800 CST m=+1.000823617\n","stream":"stdout","time":"2022-11-16T14:32:16.466197357Z"}
{"log":"2022-11-16 22:32:17.467090879 +0800 CST m=+2.001891785\n","stream":"stdout","time":"2022-11-16T14:32:17.467138819Z"}
{"log":"2022-11-16 22:32:18.467998254 +0800 CST m=+3.002799190\n","stream":"stdout","time":"2022-11-16T14:32:18.468071711Z"}
{"log":"2022-11-16 22:32:19.468046521 +0800 CST m=+4.002847457\n","stream":"stdout","time":"2022-11-16T14:32:19.468104589Z"}

```

nerdctl中日志处理方式大致如此。

[demo](https://github.com/yzxiu/nerdctl-log-example)

<br>

## kubelet中的日志处理

待补充

