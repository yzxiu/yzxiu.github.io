# Containerd解析(2) - layer


## 概述

本文只讨论containerd默认配置下的行为，不涉及docker。

参考：**[content-flow.md](https://github.com/containerd/containerd/blob/main/docs/content-flow.md)**

执行 `client.Pull(ctx, "docker.io/library/redis:5.0.9", containerd.WithPullUnpack)`之后，

containerd获取的内容如下(已排序)：

```
/var/lib/containerd/io.containerd.content.v1.content/blobs
└── sha256
    ├── 2a9865e55c37293b71df051922022898d8e4ec0f579c9b53a0caee1b170bc81c - the index
    ├── 9bb13890319dc01e5f8a4d3d0c4c72685654d682d568350fd38a02b1d70aee6b - the manifest for linux/amd64
    ├── 987b553c835f01f46eb1859bc32f564119d5833801a27b25a0ca5c6b8b6e111a - the config
    ├── bb79b6b2107fea8e8a47133a660b78e3a546998fcf0427be39ac9a0af4a97e90 - layer 0
    ├── 1ed3521a5dcbd05214eb7f35b952ecf018d5a6610c32ba4e315028c556f45e94 - layer 1
    ├── 5999b99cee8f2875d391d64df20b6296b63f23951a7d41749f028375e887cd05 - layer 2
    ├── bfee6cb5fdad6b60ec46297f44542ee9d8ac8f01c072313a51cd7822df3b576f - layer 3
    ├── fd36a1ebc6728807cbb1aa7ef24a1861343c6dc174657721c496613c7b53bd07 - layer 4
    └── 97481c7992ebf6f22636f87e4d7b79e962f928cdbe6f2337670fa6c9a9636f04 - layer 5
```

**➜**  **~** ctr content ls（已整理排序）

```
the index:
sha256:2a9865e55c37293b71df051922022898d8e4ec0f579c9b53a0caee1b170bc81c
containerd.io/gc.ref.content.m.4=sha256:ee0e1f8d8d338c9506b0e487ce6c2c41f931d1e130acd60dc7794c3a246eb59e,
containerd.io/gc.ref.content.m.3=sha256:613f4797d2b6653634291a990f3e32378c7cfe3cdd439567b26ca340b8946013,
containerd.io/gc.ref.content.m.0=sha256:9bb13890319dc01e5f8a4d3d0c4c72685654d682d568350fd38a02b1d70aee6b,
containerd.io/gc.ref.content.m.1=sha256:aeb53f8db8c94d2cd63ca860d635af4307967aa11a2fdead98ae0ab3a329f470,
containerd.io/gc.ref.content.m.6=sha256:4b7860fcaea5b9bbd6249c10a3dc02a5b9fb339e8aef17a542d6126a6af84d96,
containerd.io/gc.ref.content.m.7=sha256:d66dfc869b619cd6da5b5ae9d7b1cbab44c134b31d458de07f7d580a84b63f69,
containerd.io/gc.ref.content.m.5=sha256:1072145f8eea186dcedb6b377b9969d121a00e65ae6c20e9cd631483178ea7ed,
containerd.io/gc.ref.content.m.2=sha256:17dc42e40d4af0a9e84c738313109f3a95e598081beef6c18a05abb57337aa5d,
containerd.io/distribution.source.docker.io=library/redis,

the manifest for linux/amd64
sha256:9bb13890319dc01e5f8a4d3d0c4c72685654d682d568350fd38a02b1d70aee6b 
containerd.io/gc.ref.content.config=sha256:987b553c835f01f46eb1859bc32f564119d5833801a27b25a0ca5c6b8b6e111a,
containerd.io/gc.ref.content.l.0=sha256:bb79b6b2107fea8e8a47133a660b78e3a546998fcf0427be39ac9a0af4a97e90,
containerd.io/gc.ref.content.l.1=sha256:1ed3521a5dcbd05214eb7f35b952ecf018d5a6610c32ba4e315028c556f45e94,
containerd.io/gc.ref.content.l.2=sha256:5999b99cee8f2875d391d64df20b6296b63f23951a7d41749f028375e887cd05,
containerd.io/gc.ref.content.l.3=sha256:bfee6cb5fdad6b60ec46297f44542ee9d8ac8f01c072313a51cd7822df3b576f,
containerd.io/gc.ref.content.l.4=sha256:fd36a1ebc6728807cbb1aa7ef24a1861343c6dc174657721c496613c7b53bd07,
containerd.io/gc.ref.content.l.5=sha256:97481c7992ebf6f22636f87e4d7b79e962f928cdbe6f2337670fa6c9a9636f04,
containerd.io/distribution.source.docker.io=library/redis

the config
sha256:987b553c835f01f46eb1859bc32f564119d5833801a27b25a0ca5c6b8b6e111a
containerd.io/gc.ref.snapshot.overlayfs=sha256:33bd296ab7f37bdacff0cb4a5eb671bcb3a141887553ec4157b1e64d6641c1cd,
containerd.io/distribution.source.docker.io=library/redis

sha256:bb79b6b2107fea8e8a47133a660b78e3a546998fcf0427be39ac9a0af4a97e90 containerd.io/uncompressed=sha256:d0fe97fa8b8cefdffcef1d62b65aba51a6c87b6679628a2b50fc6a7a579f764c
sha256:1ed3521a5dcbd05214eb7f35b952ecf018d5a6610c32ba4e315028c556f45e94 containerd.io/uncompressed=sha256:832f21763c8e6b070314e619ebb9ba62f815580da6d0eaec8a1b080bd01575f7
sha256:5999b99cee8f2875d391d64df20b6296b63f23951a7d41749f028375e887cd05 containerd.io/uncompressed=sha256:223b15010c47044b6bab9611c7a322e8da7660a8268949e18edde9c6e3ea3700
sha256:bfee6cb5fdad6b60ec46297f44542ee9d8ac8f01c072313a51cd7822df3b576f containerd.io/uncompressed=sha256:b96fedf8ee00e59bf69cf5bc8ed19e92e66ee8cf83f0174e33127402b650331d
sha256:fd36a1ebc6728807cbb1aa7ef24a1861343c6dc174657721c496613c7b53bd07 containerd.io/uncompressed=sha256:aff00695be0cebb8a114f8c5187fd6dd3d806273004797a00ad934ec9cd98212
sha256:97481c7992ebf6f22636f87e4d7b79e962f928cdbe6f2337670fa6c9a9636f04 containerd.io/uncompressed=sha256:d442ae63d423b4b1922875c14c3fa4e801c66c689b69bfd853758fde996feffb
```
{{< admonition >}}
config 后面的标签 `containerd.io/gc.ref.snapshot.overlayfs`=sha256:33bd296ab7f37bdacff0cb4a5eb671bcb3a141887553ec4157b1e64d6641c1cd 
表示该镜像的最后一层(snapshot)为：33bd296ab7f37bdacff0cb4a5eb671bcb3a141887553ec4157b1e64d6641c1cd
{{< /admonition >}}

```
➜  ~ ctr snapshot ls
KEY                                                                      PARENT                                                                  KIND      
 2/sha256:d0fe97fa8b8cefdffcef1d62b65aba51a6c87b6679628a2b50fc6a7a579f764c
 4/sha256:2ae5fa95c0fce5ef33fbb87a7e2f49f2a56064566a37a83b97d3f668c10b43d6 sha256:d0fe97fa8b8cefdffcef1d62b65aba51a6c87b6679628a2b50fc6a7a579f764c Committed 
 6/sha256:a8f09c4919857128b1466cc26381de0f9d39a94171534f63859a662d50c396ca sha256:2ae5fa95c0fce5ef33fbb87a7e2f49f2a56064566a37a83b97d3f668c10b43d6 Committed 
 8/sha256:aa4b58e6ece416031ce00869c5bf4b11da800a397e250de47ae398aea2782294 sha256:a8f09c4919857128b1466cc26381de0f9d39a94171534f63859a662d50c396ca Committed 
10/sha256:bc8b010e53c5f20023bd549d082c74ef8bfc237dc9bbccea2e0552e52bc5fcb1 sha256:aa4b58e6ece416031ce00869c5bf4b11da800a397e250de47ae398aea2782294 Committed 
12/sha256:33bd296ab7f37bdacff0cb4a5eb671bcb3a141887553ec4157b1e64d6641c1cd sha256:bc8b010e53c5f20023bd549d082c74ef8bfc237dc9bbccea2e0552e52bc5fcb1 Committed 
```

{{< admonition >}}
id为12的 `snapshot` 与上面提到的最后一层相对应。
{{< /admonition >}}



## 通过 diff_ids 找到  Final Layer

 Final Layer 也可以通过 config 配置文件中的 diff_ids 数组求出来

```
"diff_ids": [
            "sha256:d0fe97fa8b8cefdffcef1d62b65aba51a6c87b6679628a2b50fc6a7a579f764c",
            "sha256:832f21763c8e6b070314e619ebb9ba62f815580da6d0eaec8a1b080bd01575f7",
            "sha256:223b15010c47044b6bab9611c7a322e8da7660a8268949e18edde9c6e3ea3700",
            "sha256:b96fedf8ee00e59bf69cf5bc8ed19e92e66ee8cf83f0174e33127402b650331d",
            "sha256:aff00695be0cebb8a114f8c5187fd6dd3d806273004797a00ad934ec9cd98212",
            "sha256:d442ae63d423b4b1922875c14c3fa4e801c66c689b69bfd853758fde996feffb"
]
```

```go
func TestFinalLayer(t *testing.T) {
	diffIDs := []digest.Digest{
		"sha256:d0fe97fa8b8cefdffcef1d62b65aba51a6c87b6679628a2b50fc6a7a579f764c",
		"sha256:832f21763c8e6b070314e619ebb9ba62f815580da6d0eaec8a1b080bd01575f7",
		"sha256:223b15010c47044b6bab9611c7a322e8da7660a8268949e18edde9c6e3ea3700",
		"sha256:b96fedf8ee00e59bf69cf5bc8ed19e92e66ee8cf83f0174e33127402b650331d",
		"sha256:aff00695be0cebb8a114f8c5187fd6dd3d806273004797a00ad934ec9cd98212",
		"sha256:d442ae63d423b4b1922875c14c3fa4e801c66c689b69bfd853758fde996feffb"}
	finalLayer := identity.ChainID(diffIDs).String()
	fmt.Println(finalLayer)
}

=== RUN   TestFinalLayer
sha256:33bd296ab7f37bdacff0cb4a5eb671bcb3a141887553ec4157b1e64d6641c1cd
--- PASS: TestFinalLayer (0.00s)
PASS
```

创建容器的时候，需要可读写层。这个可读写层的parent就是这里提到的Final Layer



## diff_ids

// TODO config 配置文件中的 diff_ids 数组是如何生成的？


