# Containerd解析(1) - example


## 概述

本文使用containerd默认配置，不涉及docker。

通过一个简单的例子，粗略了解containerd创建并运行一个容器的过程。

主要了解`containerd`怎样与`containerd-shim-runc-v2`进行交互，以及`containerd-shim-runc-v2`怎样调用`runc`与监控容器。

代码版本： [1c90a442489720eec95342e1789ee8a5e1b9536f](https://github.com/containerd/containerd/tree/1c90a442489720eec95342e1789ee8a5e1b9536f)

<br>

## Example

使用官方的[例子](https://github.com/containerd/containerd/blob/main/docs/getting-started.md#full-example)，对[containerd](https://github.com/containerd/containerd)，[containerd-shim-runc-v2](https://github.com/containerd/containerd/tree/main/cmd/containerd-shim-runc-v2) 的代码进行debug

```go
package main

import (
	"context"
	"fmt"
	"log"
	"syscall"
	"time"

	"github.com/containerd/containerd"
	"github.com/containerd/containerd/cio"
	"github.com/containerd/containerd/oci"
	"github.com/containerd/containerd/namespaces"
)

func main() {
	if err := redisExample(); err != nil {
		log.Fatal(err)
	}
}

func redisExample() error {
	// create a new client connected to the default socket path for containerd
	client, err := containerd.New("/run/containerd/containerd.sock")
	if err != nil {
		return err
	}
	defer client.Close()

	// create a new context with an "example" namespace
    // 这里将 example 改为
	ctx := namespaces.WithNamespace(context.Background(), "default")

	// pull the redis image from DockerHub
	image, err := client.Pull(ctx, "docker.io/library/redis:alpine", containerd.WithPullUnpack)
	if err != nil {
		return err
	}

	// create a container
	container, err := client.NewContainer(
		ctx,
		"redis-server",
		containerd.WithImage(image),
		containerd.WithNewSnapshot("redis-server-snapshot", image),
		containerd.WithNewSpec(oci.WithImageConfig(image)),
	)
	if err != nil {
		return err
	}
	defer container.Delete(ctx, containerd.WithSnapshotCleanup)

	// create a task from the container
	task, err := container.NewTask(ctx, cio.NewCreator(cio.WithStdio))
	if err != nil {
		return err
	}
	defer task.Delete(ctx)

	// make sure we wait before calling start
	exitStatusC, err := task.Wait(ctx)
	if err != nil {
		fmt.Println(err)
	}

	// call start on the task to execute the redis server
	if err := task.Start(ctx); err != nil {
		return err
	}

	// sleep for a lil bit to see the logs
	time.Sleep(3 * time.Second)

	// kill the process and get the exit status
	if err := task.Kill(ctx, syscall.SIGTERM); err != nil {
		return err
	}

	// wait for the process to fully exit and print out the exit status

	status := <-exitStatusC
	code, _, err := status.Result()
	if err != nil {
		return err
	}
	fmt.Printf("redis-server exited with status: %d\n", code)

	return nil
}
```

<br>

##  [client] client.Pull()

```go
// pull the redis image from DockerHub
image, err := client.Pull(ctx,"docker.io/library/redis:alpine",containerd.WithPullUnpack)
if err != nil {
	return err
}
```

containerd启动时，初始文件夹如下

```
/var/lib/containerd/
├── io.containerd.content.v1.content
│   └── ingest
├── io.containerd.metadata.v1.bolt
│   └── meta.db
├── io.containerd.runtime.v1.linux
├── io.containerd.runtime.v2.task
├── io.containerd.snapshotter.v1.aufs
│   └── snapshots
├── io.containerd.snapshotter.v1.btrfs
├── io.containerd.snapshotter.v1.native
│   └── snapshots
├── io.containerd.snapshotter.v1.overlayfs
│   └── snapshots
└── tmpmounts
```

```
/run/containerd/
├── containerd.sock
├── containerd.sock.ttrpc
├── io.containerd.runtime.v1.linux
└── io.containerd.runtime.v2.tas
```

```go
// Pull downloads the provided content into containerd's content store
// and returns a platform specific image object
func (c *Client) Pull(ctx context.Context, ref string, opts ...RemoteOpt) (_ Image, retErr error) {

	pullCtx := defaultRemoteContext()
	for _, o := range opts {
		if err := o(c, pullCtx); err != nil {
			return nil, err
		}
	}

	if pullCtx.PlatformMatcher == nil {
		if len(pullCtx.Platforms) > 1 {
			return nil, errors.New("cannot pull multiplatform image locally, try Fetch")
		} else if len(pullCtx.Platforms) == 0 {
			pullCtx.PlatformMatcher = c.platform // MatchComparer 能够匹配和比较平台以过滤和排序平台。
		} else {
			p, err := platforms.Parse(pullCtx.Platforms[0])
			if err != nil {
				return nil, fmt.Errorf("invalid platform %s: %w", pullCtx.Platforms[0], err)
			}

			pullCtx.PlatformMatcher = platforms.Only(p)
		}
	}

	ctx, done, err := c.WithLease(ctx)
	if err != nil {
		return nil, err
	}
	defer done(ctx)

	var unpacks int32
	var unpackEg *errgroup.Group
	var unpackWrapper func(f images.Handler) images.Handler

	if pullCtx.Unpack {
		// unpacker only supports schema 2 image, for schema 1 this is noop.
		u, err := c.newUnpacker(ctx, pullCtx)
		if err != nil {
			return nil, fmt.Errorf("create unpacker: %w", err)
		}
		unpackWrapper, unpackEg = u.handlerWrapper(ctx, pullCtx, &unpacks)
		defer func() {
			if err := unpackEg.Wait(); err != nil {
				if retErr == nil {
					retErr = fmt.Errorf("unpack: %w", err)
				}
			}
		}()
		wrapper := pullCtx.HandlerWrapper
		pullCtx.HandlerWrapper = func(h images.Handler) images.Handler {
			if wrapper == nil {
				return unpackWrapper(h)
			}
			return unpackWrapper(wrapper(h))
		}
	}

    // 获取镜像的主要逻辑都在 fetch 方法
	img, err := c.fetch(ctx, pullCtx, ref, 1)
	if err != nil {
		return nil, err
	}

	// NOTE(fuweid): unpacker defers blobs download. before create image
	// record in ImageService, should wait for unpacking(including blobs
	// download).
	if pullCtx.Unpack {
		if unpackEg != nil {
            // 等待镜像相关文件下载完成
			if err := unpackEg.Wait(); err != nil {
				return nil, err
			}
		}
	}

    // 调用containerd接口，往 /var/lib/containerd/io.containerd.metadata.v1.bolt/meta.db
    // 数据库写入一个 images
	img, err = c.createNewImage(ctx, img)
	if err != nil {
		return nil, err
	}

	i := NewImageWithPlatform(c, img, pullCtx.PlatformMatcher)

	if pullCtx.Unpack {
		if unpacks == 0 {
			// Try to unpack is none is done previously.
			// This is at least required for schema 1 image.
			if err := i.Unpack(ctx, pullCtx.Snapshotter, pullCtx.UnpackOpts...); err != nil {
				return nil, fmt.Errorf("failed to unpack image on snapshotter %s: %w", pullCtx.Snapshotter, err)
			}
		}
	}

	return i, nil
}
```
查看`fetch`方法：
```go
func (c *Client) fetch(ctx context.Context, rCtx *RemoteContext, ref string, limit int) (images.Image, error) {
	store := c.ContentStore()
    
    // 通过 https://registry-1.docker.io/v2/library/redis/manifests/5.0.9 获取 Digest
	// 内容大致如下：sha256:2a9865e55c37293b71df051922022898d8e4ec0f579c9b53a0caee1b170bc81c
	name, desc, err := rCtx.Resolver.Resolve(ctx, ref)
	if err != nil {
		return images.Image{}, fmt.Errorf("failed to resolve reference %q: %w", ref, err)
	}

	fetcher, err := rCtx.Resolver.Fetcher(ctx, name)
	if err != nil {
		return images.Image{}, fmt.Errorf("failed to get fetcher for %q: %w", name, err)
	}

	var (
		handler images.Handler

		isConvertible bool
		converterFunc func(context.Context, ocispec.Descriptor) (ocispec.Descriptor, error)
		limiter       *semaphore.Weighted
	)

	if desc.MediaType == images.MediaTypeDockerSchema1Manifest && rCtx.ConvertSchema1 {
		schema1Converter := schema1.NewConverter(store, fetcher)

		handler = images.Handlers(append(rCtx.BaseHandlers, schema1Converter)...)

		isConvertible = true

		converterFunc = func(ctx context.Context, _ ocispec.Descriptor) (ocispec.Descriptor, error) {
			return schema1Converter.Convert(ctx)
		}
	} else {
		// Get all the children for a descriptor
		childrenHandler := images.ChildrenHandler(store)
		// Set any children labels for that content
		childrenHandler = images.SetChildrenMappedLabels(store, childrenHandler, rCtx.ChildLabelMap)
		if rCtx.AllMetadata {
			// Filter manifests by platforms but allow to handle manifest
			// and configuration for not-target platforms
			childrenHandler = remotes.FilterManifestByPlatformHandler(childrenHandler, rCtx.PlatformMatcher)
		} else {
			// Filter children by platforms if specified.
			childrenHandler = images.FilterPlatforms(childrenHandler, rCtx.PlatformMatcher)
		}
		// Sort and limit manifests if a finite number is needed
		if limit > 0 {
			childrenHandler = images.LimitManifests(childrenHandler, rCtx.PlatformMatcher, limit)
		}

		// set isConvertible to true if there is application/octet-stream media type
		convertibleHandler := images.HandlerFunc(
			func(_ context.Context, desc ocispec.Descriptor) ([]ocispec.Descriptor, error) {
				if desc.MediaType == docker.LegacyConfigMediaType {
					isConvertible = true
				}

				return []ocispec.Descriptor{}, nil
			},
		)

		appendDistSrcLabelHandler, err := docker.AppendDistributionSourceLabel(store, ref)
		if err != nil {
			return images.Image{}, err
		}

		handlers := append(rCtx.BaseHandlers,
			remotes.FetchHandler(store, fetcher),
			convertibleHandler,
			childrenHandler,
			appendDistSrcLabelHandler,
		)

		handler = images.Handlers(handlers...)

		converterFunc = func(ctx context.Context, desc ocispec.Descriptor) (ocispec.Descriptor, error) {
			return docker.ConvertManifest(ctx, store, desc)
		}
	}

	if rCtx.HandlerWrapper != nil {
		handler = rCtx.HandlerWrapper(handler)
	}

	if rCtx.MaxConcurrentDownloads > 0 {
		limiter = semaphore.NewWeighted(int64(rCtx.MaxConcurrentDownloads))
	}

    // 遍历下载相关配置文件和分层压缩文件
	// 主要的下载任务在这里进行
	if err := images.Dispatch(ctx, handler, limiter, desc); err != nil {
		return images.Image{}, err
	}

	if isConvertible {
		if desc, err = converterFunc(ctx, desc); err != nil {
			return images.Image{}, err
		}
	}

	return images.Image{
		Name:   name,
		Target: desc,
		Labels: rCtx.Labels,
	}, nil
}

```

#### Digest

```go
name, desc, err := rCtx.Resolver.Resolve(ctx, ref)
```

我们直接使用 https://registry-1.docker.io/v2/library/redis/manifests/5.0.9 接口去请求 Digest 信息，会报401错误，如下：

```bash
➜  ~ curl -X HEAD -I https://registry-1.docker.io/v2/library/redis/manifests/5.0.9                                                
HTTP/1.1 401 Unauthorized
content-type: application/json
docker-distribution-api-version: registry/2.0
www-authenticate: Bearer realm="https://auth.docker.io/token",service="registry.docker.io",scope="repository:library/redis:pull"
date: Tue, 01 Nov 2022 05:59:02 GMT
content-length: 156
strict-transport-security: max-age=31536000
docker-ratelimit-source: 47.242.188.158
```

注意到返回的头部 `www-authenticate` 字段告诉我们如何去获取token。

```bash
➜  ~ curl https://auth.docker.io/token\?service\=registry.docker.io\&scope\=repository:library/redis:pull                                                                        
{"token":"××××","access_token":"××××","expires_in":300,"issued_at":"2022-11-01T06:11:20.10593176Z"}
```

带上token再次发送请求：

```bash
➜  ~ curl -X HEAD -I -H "Accept:application/vnd.docker.distribution.manifest.v2+json, application/vnd.docker.distribution.manifest.list.v2+json, application/vnd.oci.image.manifest.v1+json, application/vnd.oci.image.index.v1+json, */*" -H "Authorization:Bearer <token>" https://registry-1.docker.io/v2/library/redis/manifests/5.0.9

HTTP/1.1 200 OK
content-length: 1862
content-type: application/vnd.docker.distribution.manifest.list.v2+json
docker-content-digest: sha256:2a9865e55c37293b71df051922022898d8e4ec0f579c9b53a0caee1b170bc81c
docker-distribution-api-version: registry/2.0
etag: "sha256:2a9865e55c37293b71df051922022898d8e4ec0f579c9b53a0caee1b170bc81c"
date: Tue, 01 Nov 2022 06:47:50 GMT
strict-transport-security: max-age=31536000
ratelimit-limit: 100;w=21600
ratelimit-remaining: 100;w=21600
docker-ratelimit-source: 112.96.67.118
```

得到 docker-content-digest: sha256:2a9865e55c37293b71df051922022898d8e4ec0f579c9b53a0caee1b170bc81c

#### [index](https://github.com/opencontainers/image-spec/blob/main/image-index.md) / [manifest](https://github.com/opencontainers/image-spec/blob/main/manifest.md) / [config](https://github.com/opencontainers/image-spec/blob/main/config.md)

```go
if err := images.Dispatch(ctx, handler, limiter, desc); err != nil {
	return images.Image{}, err
}
```

Dispatch是一个递归方法，使用上面获取到的Digest，通过递归的方式的方式下载了容器的相关配置文件和镜像分层文件。

首先使用`Digest`获取`index`文件，内容如下：

```json
curl -X GET -H "Accept:application/vnd.docker.distribution.manifest.v2+json, application/vnd.docker.distribution.manifest.list.v2+json, application/vnd.oci.image.manifest.v1+json, application/vnd.oci.image.index.v1+json, */*" -H "Authorization:Bearer <token>" https://registry-1.docker.io/v2/library/redis/manifests/5.0.9

{
    "manifests":[
        {
            "digest":"sha256:9bb13890319dc01e5f8a4d3d0c4c72685654d682d568350fd38a02b1d70aee6b",
            "mediaType":"application\/vnd.docker.distribution.manifest.v2+json",
            "platform":{
                "architecture":"amd64",
                "os":"linux"
            },
            "size":1572
        },
        {
            "digest":"sha256:aeb53f8db8c94d2cd63ca860d635af4307967aa11a2fdead98ae0ab3a329f470",
            "mediaType":"application\/vnd.docker.distribution.manifest.v2+json",
            "platform":{
                "architecture":"arm",
                "os":"linux",
                "variant":"v5"
            },
            "size":1573
        },
        {
            "digest":"sha256:17dc42e40d4af0a9e84c738313109f3a95e598081beef6c18a05abb57337aa5d",
            "mediaType":"application\/vnd.docker.distribution.manifest.v2+json",
            "platform":{
                "architecture":"arm",
                "os":"linux",
                "variant":"v7"
            },
            "size":1573
        },
        {
            "digest":"sha256:613f4797d2b6653634291a990f3e32378c7cfe3cdd439567b26ca340b8946013",
            "mediaType":"application\/vnd.docker.distribution.manifest.v2+json",
            "platform":{
                "architecture":"arm64",
                "os":"linux",
                "variant":"v8"
            },
            "size":1573
        },
        {
            "digest":"sha256:ee0e1f8d8d338c9506b0e487ce6c2c41f931d1e130acd60dc7794c3a246eb59e",
            "mediaType":"application\/vnd.docker.distribution.manifest.v2+json",
            "platform":{
                "architecture":"386",
                "os":"linux"
            },
            "size":1572
        },
        {
            "digest":"sha256:1072145f8eea186dcedb6b377b9969d121a00e65ae6c20e9cd631483178ea7ed",
            "mediaType":"application\/vnd.docker.distribution.manifest.v2+json",
            "platform":{
                "architecture":"mips64le",
                "os":"linux"
            },
            "size":1572
        },
        {
            "digest":"sha256:4b7860fcaea5b9bbd6249c10a3dc02a5b9fb339e8aef17a542d6126a6af84d96",
            "mediaType":"application\/vnd.docker.distribution.manifest.v2+json",
            "platform":{
                "architecture":"ppc64le",
                "os":"linux"
            },
            "size":1573
        },
        {
            "digest":"sha256:d66dfc869b619cd6da5b5ae9d7b1cbab44c134b31d458de07f7d580a84b63f69",
            "mediaType":"application\/vnd.docker.distribution.manifest.v2+json",
            "platform":{
                "architecture":"s390x",
                "os":"linux"
            },
            "size":1573
        }
    ],
    "mediaType":"application\/vnd.docker.distribution.manifest.list.v2+json",
    "schemaVersion":2
}
```

根据 amd64,linux，继续获取 `manifest`（9bb13890319dc01e5f8a4d3d0c4c72685654d682d568350fd38a02b1d70aee6b），内容如下：

```json
{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
   "config": {
      "mediaType": "application/vnd.docker.container.image.v1+json",
      "size": 7648,
      "digest": "sha256:987b553c835f01f46eb1859bc32f564119d5833801a27b25a0ca5c6b8b6e111a"
   },
   "layers": [
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 27092228,
         "digest": "sha256:bb79b6b2107fea8e8a47133a660b78e3a546998fcf0427be39ac9a0af4a97e90"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 1732,
         "digest": "sha256:1ed3521a5dcbd05214eb7f35b952ecf018d5a6610c32ba4e315028c556f45e94"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 1417672,
         "digest": "sha256:5999b99cee8f2875d391d64df20b6296b63f23951a7d41749f028375e887cd05"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 7348264,
         "digest": "sha256:bfee6cb5fdad6b60ec46297f44542ee9d8ac8f01c072313a51cd7822df3b576f"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 98,
         "digest": "sha256:fd36a1ebc6728807cbb1aa7ef24a1861343c6dc174657721c496613c7b53bd07"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 409,
         "digest": "sha256:97481c7992ebf6f22636f87e4d7b79e962f928cdbe6f2337670fa6c9a9636f04"
      }
   ]
}
```

manifest中包含了config文件和镜像各个layer的压缩文件。

pull方法运行后，文件夹内容变化如下：

![image-20221101164751121](https://raw.githubusercontent.com/yzxiu/images/blog/2022-11/20221101-164752.png "after pull")

{{< admonition >}}
/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/ 文件夹中的内容，并不是直接由 /var/lib/containerd/io.containerd.content.v1.content/blobs/sha256/ 中的压缩包解压而来，而是通过 [diff](https://github.com/containerd/containerd/blob/main/docs/content-flow.md#snapshots) 工具，与上一层镜像的对比差异之后得到的。
{{< /admonition >}}

<br>

##  [client] client.NewContainer()

```go
// create a container
container, err := client.NewContainer(
	ctx,
	"redis-server",
	containerd.WithImage(image),
	containerd.WithNewSnapshot("redis-server-snapshot", image),
	containerd.WithNewSpec(oci.WithImageConfig(image)),
)
```

NewContainer()方法如下：

```go
// NewContainer will create a new container with the provided id.
// The id must be unique within the namespace.
func (c *Client) NewContainer(ctx context.Context, id string, opts ...NewContainerOpts) (Container, error) {
	ctx, done, err := c.WithLease(ctx)
	if err != nil {
		return nil, err
	}
	defer done(ctx)

	container := containers.Container{
		ID: id,
		Runtime: containers.RuntimeInfo{
			Name: c.runtime,
		},
	}
	for _, o := range opts {
		if err := o(ctx, c, &container); err != nil {
			return nil, err
		}
	}
	r, err := c.ContainerService().Create(ctx, container)
	if err != nil {
		return nil, err
	}
	return containerFromRecord(c, r), nil
}
```

查看前面传入的参数：

```go
"redis-server",
containerd.WithImage(image),
containerd.WithNewSnapshot("redis-server-snapshot", image),
containerd.WithNewSpec(oci.WithImageConfig(image)),
```

iredis-server为容器id，其他三个为初始化选项。

### WithImage：

```go
// WithImage sets the provided image as the base for the container
// 将提供的图像设置为容器的基础
func WithImage(i Image) NewContainerOpts {
	return func(ctx context.Context, client *Client, c *containers.Container) error {
		c.Image = i.Name()
		return nil
	}
}
```

### WithNewSpec：

```go
// WithNewSpec generates a new spec for a new container
// 为新容器生成新规范
func WithNewSpec(opts ...oci.SpecOpts) NewContainerOpts {
	return func(ctx context.Context, client *Client, c *containers.Container) error {
		s, err := oci.GenerateSpec(ctx, client, c, opts...)
		if err != nil {
			return err
		}
		c.Spec, err = typeurl.MarshalAny(s)
		return err
	}
}
```

### WithNewSnapshot:

```go
// WithNewSnapshot allocates a new snapshot to be used by the container as the
// root filesystem in read-write mode
// 分配一个新的快照供容器用作读写模式下的根文件系统
func WithNewSnapshot(id string, i Image, opts ...snapshots.Opt) NewContainerOpts {
	return func(ctx context.Context, client *Client, c *containers.Container) error {
        // 从 meta.db 中拿到config，从而拿到diffIDs
		diffIDs, err := i.RootFS(ctx)
		if err != nil {
			return err
		}
		// 根据 diffIDs 获取 ChainIDs
        // 进而获取 parent（即镜像的finalLayer，作为可读写层的parent）
		parent := identity.ChainID(diffIDs).String()
		c.Snapshotter, err = client.resolveSnapshotterName(ctx, c.Snapshotter)
		if err != nil {
			return err
		}
		s, err := client.getSnapshotter(ctx, c.Snapshotter)
		if err != nil {
			return err
		}
        // Prepare
		if _, err := s.Prepare(ctx, id, parent, opts...); err != nil {
			return err
		}
		c.SnapshotKey = id
		c.Image = i.Name()
		return nil
	}
}
```

```go
func (p *proxySnapshotter) Prepare(ctx context.Context, key, parent string, opts ...snapshots.Opt) ([]mount.Mount, error) {
	var local snapshots.Info
	for _, opt := range opts {
		if err := opt(&local); err != nil {
			return nil, err
		}
	}
    // 调用 containerd 的 snapshots.Prepare()接口
	resp, err := p.client.Prepare(ctx, &snapshotsapi.PrepareSnapshotRequest{
		Snapshotter: p.snapshotterName,
		Key:         key,
		Parent:      parent,
		Labels:      local.Labels,
	})
	if err != nil {
		return nil, errdefs.FromGRPC(err)
	}
	return toMounts(resp.Mounts), nil
}
```

#### [containerd] client.Prepare()

/containerd/services/snapshots/service.go L88

```go
func (s *service) Prepare(ctx context.Context, pr *snapshotsapi.PrepareSnapshotRequest) (*snapshotsapi.PrepareSnapshotResponse, error) {
	log.G(ctx).WithField("parent", pr.Parent).WithField("key", pr.Key).Debugf("prepare snapshot")
	sn, err := s.getSnapshotter(pr.Snapshotter)
	if err != nil {
		return nil, err
	}

	var opts []snapshots.Opt
	if pr.Labels != nil {
		opts = append(opts, snapshot.WithLabels(pr.Labels))
	}
    // 创建 snapshot
	mounts, err := sn.Prepare(ctx, pr.Key, pr.Parent, opts...)
	if err != nil {
		return nil, errdefs.ToGRPC(err)
	}

	return &snapshotsapi.PrepareSnapshotResponse{
		Mounts: fromMounts(mounts),
	}, nil
}
```

```go
func (s *snapshotter) Prepare(ctx context.Context, key, parent string, opts ...snapshots.Opt) ([]mount.Mount, error) {
    // 创建可读写层并入库
	mounts, err := s.Snapshotter.Prepare(ctx, key, parent, opts...)
	if err != nil {
		return nil, err
	}
    // 发送event？这部分内容待补充
	if err := s.publisher.Publish(ctx, "/snapshot/prepare", &eventstypes.SnapshotPrepare{
		Key:         key,
		Parent:      parent,
		Snapshotter: s.name,
	}); err != nil {
		return nil, err
	}
	return mounts, nil
}
```

Prepare方法执行完后，新增了一个Snapshot，如下：

```
➜  ~ ctr snapshot ls
[sudo] password for xiu: 
KEY                                                                     PARENT                                                                  KIND      
redis-server-snapshot                                                   sha256:33bd296ab7f37bdacff0cb4a5eb671bcb3a141887553ec4157b1e64d6641c1cd Active    
sha256:2ae5fa95c0fce5ef33fbb87a7e2f49f2a56064566a37a83b97d3f668c10b43d6 sha256:d0fe97fa8b8cefdffcef1d62b65aba51a6c87b6679628a2b50fc6a7a579f764c Committed 
sha256:33bd296ab7f37bdacff0cb4a5eb671bcb3a141887553ec4157b1e64d6641c1cd sha256:bc8b010e53c5f20023bd549d082c74ef8bfc237dc9bbccea2e0552e52bc5fcb1 Committed 
sha256:a8f09c4919857128b1466cc26381de0f9d39a94171534f63859a662d50c396ca sha256:2ae5fa95c0fce5ef33fbb87a7e2f49f2a56064566a37a83b97d3f668c10b43d6 Committed 
sha256:aa4b58e6ece416031ce00869c5bf4b11da800a397e250de47ae398aea2782294 sha256:a8f09c4919857128b1466cc26381de0f9d39a94171534f63859a662d50c396ca Committed 
sha256:bc8b010e53c5f20023bd549d082c74ef8bfc237dc9bbccea2e0552e52bc5fcb1 sha256:aa4b58e6ece416031ce00869c5bf4b11da800a397e250de47ae398aea2782294 Committed 
sha256:d0fe97fa8b8cefdffcef1d62b65aba51a6c87b6679628a2b50fc6a7a579f764c                                                                         Committed 

```

可以看到该snapshot的key是我们传进的参数`redis-server-snapshot`，parent是刚才提到的镜像的final layer：`33bd296ab7f37bdacff0cb4a5eb671bcb3a141887553ec4157b1e64d6641c1cd`。



###  [containerd] c.ContainerService().Create()

接下来进入create流程，直接调用containerd的create接口。

containerd/services/containers/local.go L116

```go
func (l *local) Create(ctx context.Context, req *api.CreateContainerRequest, _ ...grpc.CallOption) (*api.CreateContainerResponse, error) {
	var resp api.CreateContainerResponse

	if err := l.withStoreUpdate(ctx, func(ctx context.Context) error {
		container := containerFromProto(req.Container)

		created, err := l.Store.Create(ctx, container)
		if err != nil {
			return err
		}

		resp.Container = containerToProto(&created)

		return nil
	}); err != nil {
		return &resp, errdefs.ToGRPC(err)
	}
    // 发送event？这部分内容待补充
	if err := l.publisher.Publish(ctx, "/containers/create", &eventstypes.ContainerCreate{
		ID:    resp.Container.ID,
		Image: resp.Container.Image,
		Runtime: &eventstypes.ContainerCreate_Runtime{
			Name:    resp.Container.Runtime.Name,
			Options: resp.Container.Runtime.Options,
		},
	}); err != nil {
		return &resp, err
	}

	return &resp, nil
}
```

container参数内容如下：

{{< style "text-align:center;" >}}

<img src="https://raw.githubusercontent.com/yzxiu/images/blog/2022-11/20221102-200458.png" alt="image-20221102200457911" style="zoom: 65%;" />

{{< /style >}}

入库的内容如下：

{{< style "text-align:center;" >}}

<img src="https://raw.githubusercontent.com/yzxiu/images/blog/2022-11/20221102-201935.png" alt="image-20221102201650182" style="zoom: 73%;" />

{{< /style >}}

使用 nerdctl查看，如下：

```
➜  ~ sudo nerdctl ps -a
CONTAINER ID    IMAGE                            COMMAND                   CREATED          STATUS     PORTS    NAMES
redis-server    docker.io/library/redis:5.0.9    "docker-entrypoint.s…"    8 minutes ago    Created 
```

由上面分析可知，create操作只在数据库中插入了一条数据。

##  [client] container.NewTask()

```go
func (c *container) NewTask(ctx context.Context, ioCreate cio.Creator, opts ...NewTaskOpts) (_ Task, err error) {
	i, err := ioCreate(c.id)
	if err != nil {
		return nil, err
	}
	defer func() {
		if err != nil && i != nil {
			i.Cancel()
			i.Close()
		}
	}()
    // 配置三个std
    // Stdin=/run/containerd/fifo/908671294/redis-server-stdin
    // Stdout=/run/containerd/fifo/908671294/redis-server-stdout
    // Stderr=/run/containerd/fifo/908671294/redis-server-stderr
	cfg := i.Config()
	request := &tasks.CreateTaskRequest{
		ContainerID: c.id,
		Terminal:    cfg.Terminal,
		Stdin:       cfg.Stdin,
		Stdout:      cfg.Stdout,
		Stderr:      cfg.Stderr,
	}
	r, err := c.get(ctx)
	if err != nil {
		return nil, err
	}
	if r.SnapshotKey != "" {
		if r.Snapshotter == "" {
			return nil, fmt.Errorf("unable to resolve rootfs mounts without snapshotter on container: %w", errdefs.ErrInvalidArgument)
		}

		// get the rootfs from the snapshotter and add it to the request
		s, err := c.client.getSnapshotter(ctx, r.Snapshotter)
		if err != nil {
			return nil, err
		}
        // Options[0]: index=off
        // Options[1]: workdir=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/7/work
        // Options[2]: upperdir=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/7/fs
        // Options[3]: lowerdir=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/6/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/5/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/4/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/3/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/2/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/1/fs
		mounts, err := s.Mounts(ctx, r.SnapshotKey)
		if err != nil {
			return nil, err
		}
		spec, err := c.Spec(ctx)
		if err != nil {
			return nil, err
		}
		for _, m := range mounts {
			if spec.Linux != nil && spec.Linux.MountLabel != "" {
				context := label.FormatMountLabel("", spec.Linux.MountLabel)
				if context != "" {
					m.Options = append(m.Options, context)
				}
			}
			request.Rootfs = append(request.Rootfs, &types.Mount{
				Type:    m.Type,
				Source:  m.Source,
				Options: m.Options,
			})
		}
	}
	info := TaskInfo{
		runtime: r.Runtime.Name,
	}
	for _, o := range opts {
		if err := o(ctx, c.client, &info); err != nil {
			return nil, err
		}
	}
	if info.RootFS != nil {
		for _, m := range info.RootFS {
			request.Rootfs = append(request.Rootfs, &types.Mount{
				Type:    m.Type,
				Source:  m.Source,
				Options: m.Options,
			})
		}
	}
	if info.Options != nil {
		any, err := typeurl.MarshalAny(info.Options)
		if err != nil {
			return nil, err
		}
		request.Options = any
	}
	t := &task{
		client: c.client,
		io:     i,
		id:     c.id,
		c:      c,
	}
	if info.Checkpoint != nil {
		request.Checkpoint = info.Checkpoint
	}
    // 
	response, err := c.client.TaskService().Create(ctx, request)
	if err != nil {
		return nil, errdefs.FromGRPC(err)
	}
	t.pid = response.Pid
	return t, nil
}

```

最终组成的request请求参数如下：

![image-20221102204255754](https://raw.githubusercontent.com/yzxiu/images/blog/2022-11/20221102-204256.png "task request")

调用containerd TaskService().Create()接口。

### [containerd] (l *local) Create()

containerd/services/tasks/local.go L164

```go
func (l *local) Create(ctx context.Context, r *api.CreateTaskRequest, _ ...grpc.CallOption) (*api.CreateTaskResponse, error) {
	container, err := l.getContainer(ctx, r.ContainerID)
	if err != nil {
		return nil, errdefs.ToGRPC(err)
	}
	checkpointPath, err := getRestorePath(container.Runtime.Name, r.Options)
	if err != nil {
		return nil, err
	}
	// jump get checkpointPath from checkpoint image
	if checkpointPath == "" && r.Checkpoint != nil {
		checkpointPath, err = os.MkdirTemp(os.Getenv("XDG_RUNTIME_DIR"), "ctrd-checkpoint")
		if err != nil {
			return nil, err
		}
		if r.Checkpoint.MediaType != images.MediaTypeContainerd1Checkpoint {
			return nil, fmt.Errorf("unsupported checkpoint type %q", r.Checkpoint.MediaType)
		}
		reader, err := l.store.ReaderAt(ctx, ocispec.Descriptor{
			MediaType:   r.Checkpoint.MediaType,
			Digest:      digest.Digest(r.Checkpoint.Digest),
			Size:        r.Checkpoint.Size,
			Annotations: r.Checkpoint.Annotations,
		})
		if err != nil {
			return nil, err
		}
		_, err = archive.Apply(ctx, checkpointPath, content.NewReader(reader))
		reader.Close()
		if err != nil {
			return nil, err
		}
	}
	opts := runtime.CreateOpts{
		Spec: container.Spec,
		IO: runtime.IO{
			Stdin:    r.Stdin,
			Stdout:   r.Stdout,
			Stderr:   r.Stderr,
			Terminal: r.Terminal,
		},
		Checkpoint:     checkpointPath,
		Runtime:        container.Runtime.Name,
		RuntimeOptions: container.Runtime.Options,
		TaskOptions:    r.Options,
		SandboxID:      container.SandboxID,
	}
	if r.RuntimePath != "" {
		opts.Runtime = r.RuntimePath
	}
	for _, m := range r.Rootfs {
		opts.Rootfs = append(opts.Rootfs, mount.Mount{
			Type:    m.Type,
			Source:  m.Source,
			Options: m.Options,
		})
	}
	if strings.HasPrefix(container.Runtime.Name, "io.containerd.runtime.v1.") {
		log.G(ctx).Warn("runtime v1 is deprecated since containerd v1.4, consider using runtime v2")
	} else if container.Runtime.Name == plugin.RuntimeRuncV1 {
		log.G(ctx).Warnf("%q is deprecated since containerd v1.4, consider using %q", plugin.RuntimeRuncV1, plugin.RuntimeRuncV2)
	}
	rtime, err := l.getRuntime(container.Runtime.Name)
	if err != nil {
		return nil, err
	}
	_, err = rtime.Get(ctx, r.ContainerID)
	if err != nil && !errdefs.IsNotFound(err) {
		return nil, errdefs.ToGRPC(err)
	}
	if err == nil {
		return nil, errdefs.ToGRPC(fmt.Errorf("task %s: %w", r.ContainerID, errdefs.ErrAlreadyExists))
	}
    // Create 启动新的 shim 实例并创建新任务
	c, err := rtime.Create(ctx, r.ContainerID, opts)
	if err != nil {
		return nil, errdefs.ToGRPC(err)
	}
	labels := map[string]string{"runtime": container.Runtime.Name}
	if err := l.monitor.Monitor(c, labels); err != nil {
		return nil, fmt.Errorf("monitor task: %w", err)
	}
	pid, err := c.PID(ctx)
	if err != nil {
		return nil, fmt.Errorf("failed to get task pid: %w", err)
	}
	return &api.CreateTaskResponse{
		ContainerID: r.ContainerID,
		Pid:         pid,
	}, nil
}
```

####  rtime.Create()

rtime.Create(ctx, r.ContainerID, opts)是个比较重要的方法，终于看到和 shim 相关的内容了

```go
// Create launches new shim instance and creates new task
func (m *TaskManager) Create(ctx context.Context, taskID string, opts runtime.CreateOpts) (runtime.Task, error) {
   // 1
   shim, err := m.manager.Start(ctx, taskID, opts)
   if err != nil {
      return nil, fmt.Errorf("failed to start shim: %w", err)
   }

   // Cast to shim task and call task service to create a new container task instance.
   // This will not be required once shim service / client implemented.
   shimTask := newShimTask(shim)
   // 2
   t, err := shimTask.Create(ctx, opts)
   if err != nil {
      // NOTE: ctx contains required namespace information.
      m.manager.shims.Delete(ctx, taskID)

      dctx, cancel := timeout.WithContext(context.Background(), cleanupTimeout)
      defer cancel()

      sandboxed := opts.SandboxID != ""
      _, errShim := shimTask.delete(dctx, sandboxed, func(context.Context, string) {})
      if errShim != nil {
         if errdefs.IsDeadlineExceeded(errShim) {
            dctx, cancel = timeout.WithContext(context.Background(), cleanupTimeout)
            defer cancel()
         }

         shimTask.Shutdown(dctx)
         shimTask.Client().Close()
      }

      return nil, fmt.Errorf("failed to create shim task: %w", err)
   }

   return t, nil
}
```
##### m.manager.Start()

```go
// Start launches a new shim instance
func (m *ShimManager) Start(ctx context.Context, id string, opts runtime.CreateOpts) (_ ShimInstance, retErr error) {
   // 在 /run/containerd/io.containerd.runtime.v2.task/路径下，创建容器工作目录
   // /run/containerd/io.containerd.runtime.v2.task/default/redis-server/work -> /var/lib/containerd/io.containerd.runtime.v2.task/default/redis-server
   // 并将 config.json 写入 /run/containerd/io.containerd.runtime.v2.task/default/redis-server/
   // 很明显，这在为runc的运行准备bundle
   bundle, err := NewBundle(ctx, m.root, m.state, id, opts.Spec)
   if err != nil {
      return nil, err
   }
   defer func() {
      if retErr != nil {
         bundle.Delete()
      }
   }()

   // This container belongs to sandbox which supposed to be already started via sandbox API.
   if opts.SandboxID != "" {
      process, err := m.Get(ctx, opts.SandboxID)
      if err != nil {
         return nil, fmt.Errorf("can't find sandbox %s", opts.SandboxID)
      }

      // Write sandbox ID this task belongs to.
      if err := os.WriteFile(filepath.Join(bundle.Path, "sandbox"), []byte(opts.SandboxID), 0600); err != nil {
         return nil, err
      }

      address, err := shimbinary.ReadAddress(filepath.Join(m.state, process.Namespace(), opts.SandboxID, "address"))
      if err != nil {
         return nil, fmt.Errorf("failed to get socket address for sandbox %q: %w", opts.SandboxID, err)
      }

      // Use sandbox's socket address to handle task requests for this container.
      if err := shimbinary.WriteAddress(filepath.Join(bundle.Path, "address"), address); err != nil {
         return nil, err
      }

      shim, err := loadShim(ctx, bundle, func() {})
      if err != nil {
         return nil, fmt.Errorf("failed to load sandbox task %q: %w", opts.SandboxID, err)
      }

      if err := m.shims.Add(ctx, shim); err != nil {
         return nil, err
      }

      return shim, nil
   }

   shim, err := m.startShim(ctx, bundle, id, opts)
   if err != nil {
      return nil, err
   }
   defer func() {
      if retErr != nil {
         m.cleanupShim(shim)
      }
   }()

   if err := m.shims.Add(ctx, shim); err != nil {
      return nil, fmt.Errorf("failed to add task: %w", err)
   }

   return shim, nil
}
```

###### NewBundle()

```go
// NewBundle returns a new bundle on disk
func NewBundle(ctx context.Context, root, state, id string, spec typeurl.Any) (b *Bundle, err error) {
	if err := identifiers.Validate(id); err != nil {
		return nil, fmt.Errorf("invalid task id %s: %w", id, err)
	}

	ns, err := namespaces.NamespaceRequired(ctx)
	if err != nil {
		return nil, err
	}
	work := filepath.Join(root, ns, id)
	b = &Bundle{
		ID:        id,
		Path:      filepath.Join(state, ns, id),
		Namespace: ns,
	}
	var paths []string
	defer func() {
		if err != nil {
			for _, d := range paths {
				os.RemoveAll(d)
			}
		}
	}()
	// create state directory for the bundle
	if err := os.MkdirAll(filepath.Dir(b.Path), 0711); err != nil {
		return nil, err
	}
	if err := os.Mkdir(b.Path, 0700); err != nil {
		return nil, err
	}
	if typeurl.Is(spec, &specs.Spec{}) {
		if err := prepareBundleDirectoryPermissions(b.Path, spec.GetValue()); err != nil {
			return nil, err
		}
	}
	paths = append(paths, b.Path)
	// create working directory for the bundle
	if err := os.MkdirAll(filepath.Dir(work), 0711); err != nil {
		return nil, err
	}
	rootfs := filepath.Join(b.Path, "rootfs")
	if err := os.MkdirAll(rootfs, 0711); err != nil {
		return nil, err
	}
	paths = append(paths, rootfs)
	if err := os.Mkdir(work, 0711); err != nil {
		if !os.IsExist(err) {
			return nil, err
		}
		os.RemoveAll(work)
		if err := os.Mkdir(work, 0711); err != nil {
			return nil, err
		}
	}
	paths = append(paths, work)
	// symlink workdir 
    // /run/containerd/io.containerd.runtime.v2.task/default/redis-server/work -> /var/lib/containerd/io.containerd.runtime.v2.task/default/redis-server
	if err := os.Symlink(work, filepath.Join(b.Path, "work")); err != nil {
		return nil, err
	}
	if spec := spec.GetValue(); spec != nil {
		// write the spec to the bundle
		err = os.WriteFile(filepath.Join(b.Path, configFilename), spec, 0666)
		if err != nil {
			return nil, fmt.Errorf("failed to write %s", configFilename)
		}
	}
	return b, nil
}
```

新增的文件夹如下：

```
xiu-desktop# pwd
/run/containerd/io.containerd.runtime.v2.task/default
xiu-desktop# tree
.
└── redis-server
    ├── config.json
    ├── rootfs
    └── work -> /var/lib/containerd/io.containerd.runtime.v2.task/default/redis-server
```

###### shim, err := m.startShim()

```go
func (m *ShimManager) startShim(ctx context.Context, bundle *Bundle, id string, opts runtime.CreateOpts) (*shim, error) {
	ns, err := namespaces.NamespaceRequired(ctx)
	if err != nil {
		return nil, err
	}

	topts := opts.TaskOptions
	if topts == nil || topts.GetValue() == nil {
		topts = opts.RuntimeOptions
	}

	runtimePath, err := m.resolveRuntimePath(opts.Runtime)
	if err != nil {
		return nil, fmt.Errorf("failed to resolve runtime path: %w", err)
	}

	b := shimBinary(bundle, shimBinaryConfig{
		runtime:      runtimePath,
		address:      m.containerdAddress,
		ttrpcAddress: m.containerdTTRPCAddress,
		schedCore:    m.schedCore,
	})
    // 启动shim
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
启动shim详情：

```go 
func (b *binary) Start(ctx context.Context, opts *types.Any, onClose func()) (_ *shim, err error) {
	args := []string{"-id", b.bundle.ID}
	switch logrus.GetLevel() {
	case logrus.DebugLevel, logrus.TraceLevel:
		args = append(args, "-debug")
	}
	args = append(args, "start")

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
	// Windows needs a namespace when openShimLog
	ns, _ := namespaces.Namespace(ctx)
	shimCtx, cancelShimLog := context.WithCancel(namespaces.WithNamespace(context.Background(), ns))
	defer func() {
		if err != nil {
			cancelShimLog()
		}
	}()
	f, err := openShimLog(shimCtx, b.bundle, client.AnonDialer)
	if err != nil {
		return nil, fmt.Errorf("open shim log pipe: %w", err)
	}
	defer func() {
		if err != nil {
			f.Close()
		}
	}()
	// open the log pipe and block until the writer is ready
	// this helps with synchronization of the shim
	// copy the shim's logs to containerd's output
	go func() {
		defer f.Close()
		_, err := io.Copy(os.Stderr, f)
		// To prevent flood of error messages, the expected error
		// should be reset, like os.ErrClosed or os.ErrNotExist, which
		// depends on platform.
		err = checkCopyShimLogError(ctx, err)
		if err != nil {
			log.G(ctx).WithError(err).Error("copy shim log")
		}
	}()
    // containerd 启动 shim 进程，shim启动一个简单的rpc服务
    // 返回unix:///run/containerd/s/41107b8f6663c77e690f1e545ff41ce9039b6106896f6cf5a137e23c73c363c1
	out, err := cmd.CombinedOutput()
	if err != nil {
		return nil, fmt.Errorf("%s: %w", out, err)
	}
	address := strings.TrimSpace(string(out))
    // 连接 shim
	conn, err := client.Connect(address, client.AnonDialer)
	if err != nil {
		return nil, err
	}
	onCloseWithShimLog := func() {
		onClose()
		cancelShimLog()
		f.Close()
	}
	// Save runtime binary path for restore.
    // 保存运行时二进制路径
    // xiu-desktop# cat /run/containerd/io.containerd.runtime.v2.task/default/redis-server/shim-binary-path 
    // /usr/bin/containerd-shim-runc-v2
	if err := os.WriteFile(filepath.Join(b.bundle.Path, "shim-binary-path"), []byte(b.runtime), 0600); err != nil {
		return nil, err
	}
	client := ttrpc.NewClient(conn, ttrpc.WithOnClose(onCloseWithShimLog))
	return &shim{
		bundle: b.bundle,
		client: client,
	}, nil
}
```

containerd调用 `containerd-shim-runc-v2 `的代码`out, err := cmd.CombinedOutput()`

cmd的详细信息如下：

![image-20221102214500311](https://raw.githubusercontent.com/yzxiu/images/blog/2022-11/20221102-214500.png "containerd 启动 containerd-shim-runc-v2 的参数，启动命令为 start")

这里给 containerd-shim-runc-v2 发送了一个`start`命令。

关于shim的启动过程，后续可以在细看。

实际上 containerd-shim-runc-v2 也是一个简单的rpc服务器，使用 [ttrpc](https://github.com/containerd/ttrpc)

containerd-shim-runc-v2 启动完成后，返回了一个本地的连接地址，如下

unix:///run/containerd/s/41107b8f6663c77e690f1e545ff41ce9039b6106896f6cf5a137e23c73c363c1

###### 小结

m.manager.Start()方法准备了bundle文件夹，config.json配置文件，启动了shim。



##### shimTask.Create()

```go
func (s *shimTask) Create(ctx context.Context, opts runtime.CreateOpts) (runtime.Task, error) {
   topts := opts.TaskOptions
   if topts == nil || topts.GetValue() == nil {
      topts = opts.RuntimeOptions
   }
   request := &task.CreateTaskRequest{
      ID:         s.ID(),
      Bundle:     s.Bundle(),
      Stdin:      opts.IO.Stdin,
      Stdout:     opts.IO.Stdout,
      Stderr:     opts.IO.Stderr,
      Terminal:   opts.IO.Terminal,
      Checkpoint: opts.Checkpoint,
      Options:    protobuf.FromAny(topts),
   }
   for _, m := range opts.Rootfs {
      request.Rootfs = append(request.Rootfs, &types.Mount{
         Type:    m.Type,
         Source:  m.Source,
         Options: m.Options,
      })
   }

   // 调用 containerd-shim-runc-v2 的 create 接口
   _, err := s.task.Create(ctx, request)
   if err != nil {
      return nil, errdefs.FromGRPC(err)
   }

   return s, nil
}
```

###### s.task.Create()

containerd/api/runtime/task/v2/shim_ttrpc.pb.go L175

```go
func (c *taskClient) Create(ctx context.Context, req *CreateTaskRequest) (*CreateTaskResponse, error) {
	var resp CreateTaskResponse
	if err := c.client.Call(ctx, "containerd.task.v2.Task", "Create", req, &resp); err != nil {
		return nil, err
	}
	return &resp, nil
}
```

上面 `m.startShim() `使用`containerd-shim-runc-v2`开启了一个简单的rpc服务，并保存了它的相关client

然后这里调用它的create方法

这里我们可以调试 containerd-shim-runc-v2 的相关代码，看一下 containerd 调用 shim 的create方法之后，会发生哪些事情。

下面进入 `containerd-shim-runc-v2` 的相关代码

<br>

### [containerd-shim-runc-v2] Create()

containerd/runtime/v2/runc/task/service.go L118

```go
// Create a new initial process and container with the underlying OCI runtime
func (s *service) Create(ctx context.Context, r *taskAPI.CreateTaskRequest) (_ *taskAPI.CreateTaskResponse, err error) {
	s.mu.Lock()
	defer s.mu.Unlock()

	container, err := runc.NewContainer(ctx, s.platform, r)
	if err != nil {
		return nil, err
	}

	s.containers[r.ID] = container

	s.send(&eventstypes.TaskCreate{
		ContainerID: r.ID,
		Bundle:      r.Bundle,
		Rootfs:      r.Rootfs,
		IO: &eventstypes.TaskIO{
			Stdin:    r.Stdin,
			Stdout:   r.Stdout,
			Stderr:   r.Stderr,
			Terminal: r.Terminal,
		},
		Checkpoint: r.Checkpoint,
		Pid:        uint32(container.Pid()),
	})

	return &taskAPI.CreateTaskResponse{
		Pid: uint32(container.Pid()),
	}, nil
}
```

#### runc.NewContainer(ctx, s.platform, r)

```go
// NewContainer returns a new runc container
func NewContainer(ctx context.Context, platform stdio.Platform, r *task.CreateTaskRequest) (_ *Container, retErr error) {
	ns, err := namespaces.NamespaceRequired(ctx)
	if err != nil {
		return nil, fmt.Errorf("create namespace: %w", err)
	}

	opts := &options.Options{}
	if r.Options.GetValue() != nil {
		v, err := typeurl.UnmarshalAny(r.Options)
		if err != nil {
			return nil, err
		}
		if v != nil {
			opts = v.(*options.Options)
		}
	}

	var mounts []process.Mount
	for _, m := range r.Rootfs {
		mounts = append(mounts, process.Mount{
			Type:    m.Type,
			Source:  m.Source,
			Target:  m.Target,
			Options: m.Options,
		})
	}

	rootfs := ""
	if len(mounts) > 0 {
		rootfs = filepath.Join(r.Bundle, "rootfs")
		if err := os.Mkdir(rootfs, 0711); err != nil && !os.IsExist(err) {
			return nil, err
		}
	}

	config := &process.CreateConfig{
		ID:               r.ID,
		Bundle:           r.Bundle,
		Runtime:          opts.BinaryName,
		Rootfs:           mounts,
		Terminal:         r.Terminal,
		Stdin:            r.Stdin,
		Stdout:           r.Stdout,
		Stderr:           r.Stderr,
		Checkpoint:       r.Checkpoint,
		ParentCheckpoint: r.ParentCheckpoint,
		Options:          r.Options,
	}

	if err := WriteOptions(r.Bundle, opts); err != nil {
		return nil, err
	}
	// For historical reason, we write opts.BinaryName as well as the entire opts
	if err := WriteRuntime(r.Bundle, opts.BinaryName); err != nil {
		return nil, err
	}
	defer func() {
		if retErr != nil {
			if err := mount.UnmountAll(rootfs, 0); err != nil {
				logrus.WithError(err).Warn("failed to cleanup rootfs mount")
			}
		}
	}()
    // 在这里进行rootfs的挂载
	for _, rm := range mounts {
		m := &mount.Mount{
			Type:    rm.Type,
			Source:  rm.Source,
			Options: rm.Options,
		}
		if err := m.Mount(rootfs); err != nil {
			return nil, fmt.Errorf("failed to mount rootfs component %v: %w", m, err)
		}
	}

	p, err := newInit(
		ctx,
		r.Bundle,
		filepath.Join(r.Bundle, "work"),
		ns,
		platform,
		config,
		opts,
		rootfs,
	)
	if err != nil {
		return nil, errdefs.ToGRPC(err)
	}
    // 组装 p 和 config，准备调用runc
	if err := p.Create(ctx, config); err != nil {
		return nil, errdefs.ToGRPC(err)
	}
	container := &Container{
		ID:              r.ID,
		Bundle:          r.Bundle,
		process:         p,
		processes:       make(map[string]process.Process),
		reservedProcess: make(map[string]struct{}),
	}
	pid := p.Pid()
	if pid > 0 {
		var cg interface{}
		if cgroups.Mode() == cgroups.Unified {
			g, err := cgroupsv2.PidGroupPath(pid)
			if err != nil {
				logrus.WithError(err).Errorf("loading cgroup2 for %d", pid)
				return container, nil
			}
			cg, err = cgroupsv2.LoadManager("/sys/fs/cgroup", g)
			if err != nil {
				logrus.WithError(err).Errorf("loading cgroup2 for %d", pid)
			}
		} else {
			cg, err = cgroups.Load(cgroups.V1, cgroups.PidPath(pid))
			if err != nil {
				logrus.WithError(err).Errorf("loading cgroup for %d", pid)
			}
		}
		container.cgroup = cg
	}
	return container, nil
}
```

在 `p.Create(ctx, config)`之前，有一个比较重要的操作，就是 `m.Mount(rootfs)`

##### m.Mount(rootfs)

```go
for _, rm := range mounts {
	m := &mount.Mount{
		Type:    rm.Type,
		Source:  rm.Source,
		Options: rm.Options,
	}
	if err := m.Mount(rootfs); err != nil {
		return nil, fmt.Errorf("failed to mount rootfs component %v: %w", m, err)
	}
}
```

`m.Mount(rootfs)`执行了挂载。

相关参数如下：

![image-20221103135749661](https://raw.githubusercontent.com/yzxiu/images/blog/2022-11/20221103-135750.png "m")

其中，Options[3]的内容如下：

```
lowerdir=
/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/6/fs:
/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/5/fs:
/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/4/fs:
/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/3/fs:
/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/2/fs:
/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/1/fs
```

![image-20221103135822520](https://raw.githubusercontent.com/yzxiu/images/blog/2022-11/20221103-135823.png "rootfs")

执行完成后，/run/containerd/文件夹内容如下：

<img src="https://raw.githubusercontent.com/yzxiu/images/blog/2022-11/20221103-095252.png" alt="image-20221103095250783" title="挂在 rootfs" style="zoom: 80%;" />

##### p.Create(ctx, config)

```go
// Create the process with the provided config
func (p *Init) Create(ctx context.Context, r *CreateConfig) error {
	var (
		err     error
		socket  *runc.Socket
		pio     *processIO
		pidFile = newPidFile(p.Bundle)
	)

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
	if r.Checkpoint != "" {
		return p.createCheckpointedState(r, pidFile)
	}
	opts := &runc.CreateOpts{
		PidFile:      pidFile.Path(),
		NoPivot:      p.NoPivotRoot,
		NoNewKeyring: p.NoNewKeyring,
	}
    // 配置相关io
	if p.io != nil {
		opts.IO = p.io.IO()
	}
	if socket != nil {
		opts.ConsoleSocket = socket
	}
    // 
	if err := p.runtime.Create(ctx, r.ID, r.Bundle, opts); err != nil {
		return p.runtimeError(err, "OCI runtime create failed")
	}
	if r.Stdin != "" {
		if err := p.openStdin(r.Stdin); err != nil {
			return err
		}
	}
	ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
	defer cancel()
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
	pid, err := pidFile.Read()
	if err != nil {
		return fmt.Errorf("failed to retrieve OCI runtime container pid: %w", err)
	}
	p.pid = pid
	return nil
}
```

先看一下这里 p 和 config 的具体内容：

![image-20221102223922468](https://raw.githubusercontent.com/yzxiu/images/blog/2022-11/20221102-223923.png "p")

![image-20221102223958607](https://raw.githubusercontent.com/yzxiu/images/blog/2022-11/20221102-223959.png "config")

###### p.runtime.Create()

```go
// Create creates a new container and returns its pid if it was created successfully
func (r *Runc) Create(context context.Context, id, bundle string, opts *CreateOpts) error {
	args := []string{"create", "--bundle", bundle}
	if opts != nil {
		oargs, err := opts.args()
		if err != nil {
			return err
		}
		args = append(args, oargs...)
	}
	cmd := r.command(context, append(args, id)...)
	if opts != nil && opts.IO != nil {
		opts.Set(cmd)
	}
	cmd.ExtraFiles = opts.ExtraFiles

	if cmd.Stdout == nil && cmd.Stderr == nil {
		data, err := cmdOutput(cmd, true, nil)
		defer putBuf(data)
		if err != nil {
			return fmt.Errorf("%s: %s", err, data.String())
		}
		return nil
	}
	ec, err := Monitor.Start(cmd)
	if err != nil {
		return err
	}
	if opts != nil && opts.IO != nil {
		if c, ok := opts.IO.(StartCloser); ok {
			if err := c.CloseAfterStart(); err != nil {
				return err
			}
		}
	}
	status, err := Monitor.Wait(cmd, ec)
	if err == nil && status != 0 {
		err = fmt.Errorf("%s did not terminate successfully: %w", cmd.Args[0], &ExitError{status})
	}
	return err
}
```

 `ec, err := Monitor.Start(cmd)`

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
```

查看最终cmd的内容：

<img src="https://raw.githubusercontent.com/yzxiu/images/blog/2022-11/20221103-100353.png" alt="image-20221103100352489" style="zoom:80%;" />

{{< admonition info >}}
shim的 `Create()` 方法主要有两个操作：
1) 挂载rootfs目录
2) 调用了 `runc create` 命令。
{{< /admonition >}}

查看容器状态

```
➜  ~ n ps -a      
CONTAINER ID    IMAGE                            COMMAND                   CREATED           STATUS     PORTS    NAMES
redis-server    docker.io/library/redis:5.0.9    "docker-entrypoint.s…"    43 minutes ago    Created

➜  ~ runc list
ID             PID         STATUS      BUNDLE                                                               CREATED                          OWNER
redis-server   67705       created     /run/containerd/io.containerd.runtime.v2.task/default/redis-server   2022-11-03T02:05:53.611043305Z   root

➜  ~ runc state redis-server
{
  "ociVersion": "1.0.2-dev",
  "id": "redis-server",
  "pid": 67705,
  "status": "created",
  "bundle": "/run/containerd/io.containerd.runtime.v2.task/default/redis-server",
  "rootfs": "/run/containerd/io.containerd.runtime.v2.task/default/redis-server/rootfs",
  "created": "2022-11-03T02:05:53.611043305Z",
  "owner": ""
}%                                                                                        

➜  ~ runc ps redis-server   
UID          PID    PPID  C STIME TTY          TIME CMD
root       67705   63337  0 10:05 ?        00:00:00 runc init
```

容器现在处于 runc create 之后的created(runc init)状态。

<br>

### [containerd] l.monitor.Monitor(c, labels)

```go
func (m *cgroupsMonitor) Monitor(c runtime.Task, labels map[string]string) error {
	if err := m.collector.Add(c, labels); err != nil {
		return err
	}
	t, ok := c.(*linux.Task)
	if !ok {
		return nil
	}
	cg, err := t.Cgroup()
	if err != nil {
		if errdefs.IsNotFound(err) {
			return nil
		}
		return err
	}
	err = m.oom.Add(c.ID(), c.Namespace(), cg, m.trigger)
	if err == cgroups.ErrMemoryNotSupported {
		logrus.WithError(err).Warn("OOM monitoring failed")
		return nil
	}
	return err
}

```



<br>

### 小结

从上文的分析可以看出，NewTask()完成了容器启动前的所有准备工作。

`containerd`创建task相关文件夹(也就是runc所需要的bundle)，并创建spec(也就是config.json)。

`shim`根据配置挂载rootfs，然后调用`runc -b ... create`创建容器，等待`runc start`启动容器。



<br>

## [client] task.Start(ctx)

```go
// call start on the task to execute the redis server
if err := task.Start(ctx); err != nil {
   return err
}
```

### [containerd] (l *local) Start()

```go
func (l *local) Start(ctx context.Context, r *api.StartRequest, _ ...grpc.CallOption) (*api.StartResponse, error) {
   t, err := l.getTask(ctx, r.ContainerID)
   if err != nil {
      return nil, err
   }
   p := runtime.Process(t)
   if r.ExecID != "" {
      if p, err = t.Process(ctx, r.ExecID); err != nil {
         return nil, errdefs.ToGRPC(err)
      }
   }
   // containerd-shim-runc-v2
   if err := p.Start(ctx); err != nil {
      return nil, errdefs.ToGRPC(err)
   }
   state, err := p.State(ctx)
   if err != nil {
      return nil, errdefs.ToGRPC(err)
   }
   return &api.StartResponse{
      Pid: state.Pid,
   }, nil
}
```

#### [containerd-shim-runc-v2] p.Start(ctx)

```go
// Start a process
func (s *service) Start(ctx context.Context, r *taskAPI.StartRequest) (*taskAPI.StartResponse, error) {
   container, err := s.getContainer(r.ID)
   if err != nil {
      return nil, err
   }

   // hold the send lock so that the start events are sent before any exit events in the error case
   s.eventSendMu.Lock()
   // 
   p, err := container.Start(ctx, r)
   if err != nil {
      s.eventSendMu.Unlock()
      return nil, errdefs.ToGRPC(err)
   }

   switch r.ExecID {
   case "":
      switch cg := container.Cgroup().(type) {
      case cgroups.Cgroup:
         if err := s.ep.Add(container.ID, cg); err != nil {
            logrus.WithError(err).Error("add cg to OOM monitor")
         }
      case *cgroupsv2.Manager:
         allControllers, err := cg.RootControllers()
         if err != nil {
            logrus.WithError(err).Error("failed to get root controllers")
         } else {
            if err := cg.ToggleControllers(allControllers, cgroupsv2.Enable); err != nil {
               if userns.RunningInUserNS() {
                  logrus.WithError(err).Debugf("failed to enable controllers (%v)", allControllers)
               } else {
                  logrus.WithError(err).Errorf("failed to enable controllers (%v)", allControllers)
               }
            }
         }
         if err := s.ep.Add(container.ID, cg); err != nil {
            logrus.WithError(err).Error("add cg to OOM monitor")
         }
      }

      s.send(&eventstypes.TaskStart{
         ContainerID: container.ID,
         Pid:         uint32(p.Pid()),
      })
   default:
      s.send(&eventstypes.TaskExecStarted{
         ContainerID: container.ID,
         ExecID:      r.ExecID,
         Pid:         uint32(p.Pid()),
      })
   }
   s.eventSendMu.Unlock()
   return &taskAPI.StartResponse{
      Pid: uint32(p.Pid()),
   }, nil
}
```

```go
// Start a container process
func (c *Container) Start(ctx context.Context, r *task.StartRequest) (process.Process, error) {
   p, err := c.Process(r.ExecID)
   if err != nil {
      return nil, err
   }
   // 
   if err := p.Start(ctx); err != nil {
      return nil, err
   }
   if c.Cgroup() == nil && p.Pid() > 0 {
      var cg interface{}
      if cgroups.Mode() == cgroups.Unified {
         g, err := cgroupsv2.PidGroupPath(p.Pid())
         if err != nil {
            logrus.WithError(err).Errorf("loading cgroup2 for %d", p.Pid())
         }
         cg, err = cgroupsv2.LoadManager("/sys/fs/cgroup", g)
         if err != nil {
            logrus.WithError(err).Errorf("loading cgroup2 for %d", p.Pid())
         }
      } else {
         cg, err = cgroups.Load(cgroups.V1, cgroups.PidPath(p.Pid()))
         if err != nil {
            logrus.WithError(err).Errorf("loading cgroup for %d", p.Pid())
         }
      }
      c.cgroup = cg
   }
   return p, nil
}
```

```go
// Start the init process
func (p *Init) Start(ctx context.Context) error {
   p.mu.Lock()
   defer p.mu.Unlock()
   
   // 
   return p.initState.Start(ctx)
}
```

```go
func (s *createdState) Start(ctx context.Context) error {
   // 
   if err := s.p.start(ctx); err != nil {
      return err
   }
   return s.transition("running")
}
```

```go
func (p *Init) start(ctx context.Context) error {
   // 
   err := p.runtime.Start(ctx, p.id)
   return p.runtimeError(err, "OCI runtime start failed")
}
```

```go
// Start will start an already created container
func (r *Runc) Start(context context.Context, id string) error {
   // 
   return r.runOrError(r.command(context, "start", id))
}
```

```go
// runOrError will run the provided command.  If an error is
// encountered and neither Stdout or Stderr was set the error and the
// stderr of the command will be returned in the format of <error>:
// <stderr>
func (r *Runc) runOrError(cmd *exec.Cmd) error {
	if cmd.Stdout != nil || cmd.Stderr != nil {
		ec, err := Monitor.Start(cmd)
		if err != nil {
			return err
		}
		status, err := Monitor.Wait(cmd, ec)
		if err == nil && status != 0 {
			err = fmt.Errorf("%s did not terminate successfully: %w", cmd.Args[0], &ExitError{status})
		}
		return err
	}
    // 
	data, err := cmdOutput(cmd, true, nil)
	defer putBuf(data)
	if err != nil {
		return fmt.Errorf("%s: %s", err, data.String())
	}
	return nil
}
```

```go
// callers of cmdOutput are expected to call putBuf on the returned Buffer
// to ensure it is released back to the shared pool after use.
func cmdOutput(cmd *exec.Cmd, combined bool, started chan<- int) (*bytes.Buffer, error) {
   b := getBuf()

   cmd.Stdout = b
   if combined {
      cmd.Stderr = b
   }
   // 
   ec, err := Monitor.Start(cmd)
   if err != nil {
      return nil, err
   }
   if started != nil {
      started <- cmd.Process.Pid
   }

   status, err := Monitor.Wait(cmd, ec)
   if err == nil && status != 0 {
      err = fmt.Errorf("%s did not terminate successfully: %w", cmd.Args[0], &ExitError{status})
   }

   return b, err
}
```

```go
// Start starts the command a registers the process with the reaper
func (m *Monitor) Start(c *exec.Cmd) (chan runc.Exit, error) {
   ec := m.Subscribe()
   // 
   if err := c.Start(); err != nil {
      m.Unsubscribe(ec)
      return nil, err
   }
   return ec, nil
}
```

在 `containerd-shim-runc-v2`中套了很多层调用，最终还是回到了 `(m *Monitor) Start()`

查看此时的cmd参数：

<img src="https://raw.githubusercontent.com/yzxiu/images/blog/2022-11/20221103-111446.png" alt="image-20221103111445944" style="zoom:80%;" />

执行了 `runc start`

查看此时容器状态

```
➜  ~ n ps -a
CONTAINER ID    IMAGE                            COMMAND                   CREATED           STATUS    PORTS    NAMES
redis-server    docker.io/library/redis:5.0.9    "docker-entrypoint.s…"    18 minutes ago    Up                 
➜  ~ 
➜  ~ 
➜  ~ runc list
ID             PID         STATUS      BUNDLE                                                               CREATED                         OWNER
redis-server   75707       running     /run/containerd/io.containerd.runtime.v2.task/default/redis-server   2022-11-03T03:04:04.47687884Z   root
➜  ~ 
➜  ~ 
➜  ~ runc state redis-server
{
  "ociVersion": "1.0.2-dev",
  "id": "redis-server",
  "pid": 75707,
  "status": "running",
  "bundle": "/run/containerd/io.containerd.runtime.v2.task/default/redis-server",
  "rootfs": "/run/containerd/io.containerd.runtime.v2.task/default/redis-server/rootfs",
  "created": "2022-11-03T03:04:04.47687884Z",
  "owner": ""
}

➜  ~ 
➜  ~ runc ps redis-server   
UID          PID    PPID  C STIME TTY          TIME CMD
systemd+   75707   75685  0 11:04 ?        00:00:00 redis-server *:6379
```

<br>

## [client] task.Kill()

```go
// kill the process and get the exit status
if err := task.Kill(ctx, syscall.SIGTERM); err != nil {
   return err
}
```

### [containerd] (l *local) Kill()

```go
func (l *local) Kill(ctx context.Context, r *api.KillRequest, _ ...grpc.CallOption) (*ptypes.Empty, error) {
   t, err := l.getTask(ctx, r.ContainerID)
   if err != nil {
      return nil, err
   }
   p := runtime.Process(t)
   if r.ExecID != "" {
      if p, err = t.Process(ctx, r.ExecID); err != nil {
         return nil, errdefs.ToGRPC(err)
      }
   }
   // 
   if err := p.Kill(ctx, r.Signal, r.All); err != nil {
      return nil, errdefs.ToGRPC(err)
   }
   return empty, nil
}
```

```go
func (s *shimTask) Kill(ctx context.Context, signal uint32, all bool) error {
   // 
   if _, err := s.task.Kill(ctx, &task.KillRequest{
      ID:     s.ID(),
      Signal: signal,
      All:    all,
   }); err != nil {
      return errdefs.FromGRPC(err)
   }
   return nil
}
```

####  [containerd-shim-runc-v2] s.task.Kill()

```go
// Kill a process with the provided signal
func (s *service) Kill(ctx context.Context, r *taskAPI.KillRequest) (*ptypes.Empty, error) {
   container, err := s.getContainer(r.ID)
   if err != nil {
      return nil, err
   }
   // 
   if err := container.Kill(ctx, r); err != nil {
      return nil, errdefs.ToGRPC(err)
   }
   return empty, nil
}
```

```go
// Kill a process
func (c *Container) Kill(ctx context.Context, r *task.KillRequest) error {
   p, err := c.Process(r.ExecID)
   if err != nil {
      return err
   }
   // 
   return p.Kill(ctx, r.Signal, r.All)
}
```

```go
// Kill the init process
func (p *Init) Kill(ctx context.Context, signal uint32, all bool) error {
   p.mu.Lock()
   defer p.mu.Unlock()
   // 
   return p.initState.Kill(ctx, signal, all)
}
```

```go
func (s *runningState) Kill(ctx context.Context, sig uint32, all bool) error {
    //
    return s.p.kill(ctx, sig, all)
}
```

```go
func (p *Init) kill(ctx context.Context, signal uint32, all bool) error {
   	// 
    err := p.runtime.Kill(ctx, p.id, int(signal), &runc.KillOpts{
      All: all,
   })
   return checkKillError(err)
}
```

```go
// Kill sends the specified signal to the container
func (r *Runc) Kill(context context.Context, id string, sig int, opts *KillOpts) error {
   args := []string{
      "kill",
   }
   if opts != nil {
      args = append(args, opts.args()...)
   }
   // 
   return r.runOrError(r.command(context, append(args, id, strconv.Itoa(sig))...))
}
```

```go
// runOrError will run the provided command.  If an error is
// encountered and neither Stdout or Stderr was set the error and the
// stderr of the command will be returned in the format of <error>:
// <stderr>
func (r *Runc) runOrError(cmd *exec.Cmd) error {
   if cmd.Stdout != nil || cmd.Stderr != nil {
      ec, err := Monitor.Start(cmd)
      if err != nil {
         return err
      }
      status, err := Monitor.Wait(cmd, ec)
      if err == nil && status != 0 {
         err = fmt.Errorf("%s did not terminate successfully: %w", cmd.Args[0], &ExitError{status})
      }
      return err
   }
   // 
   data, err := cmdOutput(cmd, true, nil)
   defer putBuf(data)
   if err != nil {
      return fmt.Errorf("%s: %s", err, data.String())
   }
   return nil
}
```

```go
// callers of cmdOutput are expected to call putBuf on the returned Buffer
// to ensure it is released back to the shared pool after use.
func cmdOutput(cmd *exec.Cmd, combined bool, started chan<- int) (*bytes.Buffer, error) {
   b := getBuf()

   cmd.Stdout = b
   if combined {
      cmd.Stderr = b
   }
   // 
   ec, err := Monitor.Start(cmd)
   if err != nil {
      return nil, err
   }
   if started != nil {
      started <- cmd.Process.Pid
   }

   status, err := Monitor.Wait(cmd, ec)
   if err == nil && status != 0 {
      err = fmt.Errorf("%s did not terminate successfully: %w", cmd.Args[0], &ExitError{status})
   }

   return b, err
}
```

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
```

同样的，在 `containerd-shim-runc-v2`中套了很多层调用，最终还是回到了 `(m *Monitor) Start()`

查看此时的cmd参数：

<img src="https://raw.githubusercontent.com/yzxiu/images/blog/2022-11/20221103-115011.png" alt="image-20221103115008534" style="zoom:80%;" />

发送了 `runc kill redis-server 15` 命令。

执行完成后，容器状态如下：

```
➜  ~ n ps -a
CONTAINER ID    IMAGE                            COMMAND                   CREATED           STATUS                           PORTS    NAMES
redis-server    docker.io/library/redis:5.0.9    "docker-entrypoint.s…"    50 minutes ago    Exited (0) About a minute ago             
➜  ~ 
➜  ~ 
➜  ~ runc list           
ID             PID         STATUS      BUNDLE                                                               CREATED                         OWNER
redis-server   0           stopped     /run/containerd/io.containerd.runtime.v2.task/default/redis-server   2022-11-03T03:04:04.47687884Z   root
➜  ~ 
➜  ~ 
➜  ~ runc state redis-server
{
  "ociVersion": "1.0.2-dev",
  "id": "redis-server",
  "pid": 0,
  "status": "stopped",
  "bundle": "/run/containerd/io.containerd.runtime.v2.task/default/redis-server",
  "rootfs": "/run/containerd/io.containerd.runtime.v2.task/default/redis-server/rootfs",
  "created": "2022-11-03T03:04:04.47687884Z",
  "owner": ""
}
➜  ~ 
➜  ~ 
➜  ~ runc ps redis-server   
UID          PID    PPID  C STIME TTY          TIME CMD
```

## defer task.Delete()

## defer container.Delete()

## 总结

本文通过一个简单的例子，忽略了较多细节，了解一个容器在containerd中的主要启动过程。


