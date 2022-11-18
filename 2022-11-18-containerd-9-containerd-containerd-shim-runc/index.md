# Containerd解析(9) - Containerd,Containerd-shim,runc的依存关系


## 概述

参考[文章](https://fankangbest.github.io/2017/11/24/containerd-containerd-shim%E5%92%8Crunc%E7%9A%84%E4%BE%9D%E5%AD%98%E5%85%B3%E7%B3%BB/)，该文时间比较久远，containerd的很多逻辑已经发生变化。

学习该文思路，重新整理`Containerd`,`Containerd-shim`,`runc` 之间的依存关系。



<br>

## 整体关系

使用nerdctl启动容器`n run -d --name runcdev q946666800/runcdev`

进程如下：

```
root        3716       1  0 10:36 ?        00:00:00 /usr/bin/containerd
root        4509       1  0 10:37 ?        00:00:00 /usr/bin/containerd-shim-runc-v2
root        4567    4509  0 10:37 ?        00:00:00  \_ /main-example
```

containerd 与 shim的父进程都是1, 容器进程父进程是shim。

接下来，分析下面四种情况：

1. containerd进程存在的情况下，杀死containerd-shim进程；
2. containerd进程存在的情况下，杀死容器进程；
3. containerd进程不存在的情况下，杀死containerd-shim进程，然后启动containerd进程；
4. containerd进程不存在的情况下，杀死容器进程，然后启动containerd进程；



<br>

## 第一种情况

containerd进程存在的情况下，杀死containerd-shim进程。

```
~ ps -ef | egrep 'containerd|example' | grep -v grep
root        3716       1  0 10:36 ?        00:00:00 /usr/bin/containerd
root        4509       1  0 10:37 ?        00:00:00 /usr/bin/containerd-shim-runc-v2
root        4567    4509  0 10:37 ?        00:00:00  \_ /main-example
```

使用`kill -9 4509`杀死shim进程，看到容器进程也跟着退出。

```
~ ps -ef | egrep 'containerd|example' | grep -v grep
root        3716       1  0 10:36 ?        00:00:00 /usr/bin/containerd
```

{{< admonition info >}}
结论：在containerd运行的情况下，杀死containerd-shim，**容器进程会退出**。
{{< /admonition >}}







<br>








