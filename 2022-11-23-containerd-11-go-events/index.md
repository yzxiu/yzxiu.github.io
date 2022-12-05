# Containerd解析(11) - event & go-events


## 概述

在 containerd 的 event 中，主要用到了 [go-events](https://github.com/docker/go-events) 这个包。

为 Go 实现一个可组合的事件分发包。



## 初始化

在containerd启动过程中，

```go
// New creates and initializes a new containerd server
func New(ctx context.Context, config *srvconfig.Config) (*Server, error) {
	...
	...
	var events      = exchange.NewExchange()
    ...
    ...
    for _, p := range plugins {
		...
        initContext := plugin.NewContext(
			ctx,
			p,
			initialized,
			config.Root,
			config.State,
		)
		initContext.Events = events
		initContext.Address = config.GRPC.Address
		initContext.TTRPCAddress = config.TTRPC.Address
        ...
        ...
	}
}

```

使用 `exchange.NewExchange()` 初始化，并将它传递个各个`plugin`，用于处理事件。



`containerd`中的events处理逻辑与`shim` 中的Monitor逻辑类似，在需要订阅的地方调用`Subscribe`，然后`channel`，`channel`会接受到对应的`Envelope`(也就是相关事件)。

`Subscribe`在`cri`插件中的应用比较多。



containerd中的事件分发还与grpc进行了结合，可以通过grpc接口进行事件的订阅。

grpc接口的定义：

```protobuf
service Events {

	rpc Publish(PublishRequest) returns (google.protobuf.Empty);
	
	rpc Forward(ForwardRequest) returns (google.protobuf.Empty);

	rpc Subscribe(SubscribeRequest) returns (stream Envelope);
}
```

服务端实现：

```go
func (s *service) Subscribe(req *api.SubscribeRequest, srv api.Events_SubscribeServer) error {
	ctx, cancel := context.WithCancel(srv.Context())
	defer cancel()

	eventq, errq := s.events.Subscribe(ctx, req.Filters...)
	for {
		select {
		case ev := <-eventq:
			if err := srv.Send(toProto(ev)); err != nil {
				return fmt.Errorf("failed sending event to subscriber: %w", err)
			}
		case err := <-errq:
			if err != nil {
				return fmt.Errorf("subscription error: %w", err)
			}

			return nil
		}
	}
}
```


























