# client-go解析(1)


## 概述

k8s许多重要的基础类型，都在client-go中定义。

如 `cache.Store`、`cache.Reflector`等。

接下来分析这些重要的接口和类型，以及他们的组合(比如`informer`)在k8s中的使用。





## 使用

client-go的一些用法，可以参考  https://github.com/iximiuz/client-go-examples

