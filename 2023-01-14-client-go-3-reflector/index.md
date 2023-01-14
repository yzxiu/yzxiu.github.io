# client-go解析(3) - cache.Reflector


## 概述

`cache.Reflector` 可以说是k8s最重要的组件，它串联起k8s的整个流程。

在服务端(apiserver) ，使用 reflector 向 etcd 获取资源数据。

在连接端(informer、kubelet)，使用 reflector 向 apiserver 获取资源数据。

k8s的整个逻辑流程中，所有这些获取资源数据相关的操作，都封装在 reflector 里面，可以看出 reflector 对于理解 k8s 的重要性。




