# openebs-mayastor安装与使用


## 概述

#### k8s环境：

| name  | ip           | role   | sys                    | disk1 | disk2 |
| ----- | ------------ | ------ | ---------------------- | ----- | ----- |
| node1 | 192.168.4.55 | master | ubuntu22.04 5.15.0-105 | 64g   | 20g   |
| node2 | 192.168.4.56 | node   | ubuntu22.04 5.15.0-105 | 64g   | 20g   |
| node3 | 192.168.4.57 | node   | ubuntu22.04 5.15.0-105 | 64g   | 20g   |
| node4 | 192.168.4.58 | node   | ubuntu22.04 5.15.0-105 | 64g   | null  |

node1 ~ node3 有两块硬盘，vda 为64g，用作根目录，vdb为20g，用作 mayastor 存储。

node4 只有一块硬盘，用作根目录。



## 安装

参考文档：https://mayastor.gitbook.io/introduction



#### 准备工作：

**启用 nvme-tcp 内核模块**

临时启用：

modprobe nvme-tcp

永久启用：

添加 /etc/modules-load.d/nvme_tcp.conf 文件，内容为 nvme-tcp



**HugePage support** 

```bash
# 查询
[qemu-51] ~ grep HugePages /proc/meminfo
AnonHugePages:         0 kB
ShmemHugePages:        0 kB
FileHugePages:         0 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0

# 临时开启
echo 1024 | sudo tee /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages

# 永久开启
echo vm.nr_hugepages = 1024 | sudo tee -a /etc/sysctl.conf

# 重启
reboot

# 重新查询
[qemu-51] ~ grep HugePages /proc/meminfo
AnonHugePages:         0 kB
ShmemHugePages:        0 kB
FileHugePages:         0 kB
HugePages_Total:    1024
HugePages_Free:     1024
HugePages_Rsvd:        0
HugePages_Surp:        0


# 在已经部署 io-engine 的机器查询结果如下：
root@node1:~# grep HugePages /proc/meminfo
AnonHugePages:         0 kB
ShmemHugePages:        0 kB
FileHugePages:         0 kB
HugePages_Total:    1024
HugePages_Free:      566
HugePages_Rsvd:        0
HugePages_Surp:        0

```

**为 Mayastor 节点候选者添加标签**

```bash
kubectl label node <node_name> openebs.io/engine=mayastor
```

mayastor 会为打了标签的节点安装 io-engine

使用 mayastor 存储卷的pod，要加上

```yaml
  nodeSelector:
    openebs.io/engine: mayastor
```



#### 安装

使用 helm 命令安装：

```bash
helm repo add mayastor https://openebs.github.io/mayastor-extensions/ 

helm repo update

# 直接在线安装
helm install mayastor mayastor/mayastor -n mayastor --create-namespace --version 2.5.0

```



也可以下载 [Artifact ](https://artifacthub.io/) 进行安装：

```bash
helm pull mayastor/mayastor --version 2.5.0

tar -xvf mayastor-2.5.0.tgz

cd mayastor

# 安装
helm install mayastor -n mayastor --create-namespace .

# 更新
helm upgrade -n mayastor mayastor .
```



mayastor默认会安装很多组建，如loki，测试环境可以先去掉。

```yaml
loki-stack:
  # -- Enable loki log collection for our components
  enabled: false
  
eventing:
  enabled: false
  
obs:
  callhome:
    # -- Enable callhome
    enabled: false

```



io-engine 组建，默认会开启两个线程进行满负载运行，在测试环境可以调低(调为1)

```yaml
  # -- The number of cores that each io-engine instance will bind to.
  cpuCount: "1"
```



安装完成后，需要先创建存储池。

使用 kubectl mayastor get block-devices <node_name> 获取磁盘id路径

```bash
[xiu-tb] ~ kubectl mayastor get block-devices dev2
 DEVNAME    DEVTYPE    SIZE    AVAILABLE  MODEL             DEVPATH                                                                         MAJOR  MINOR  DEVLINKS 
 /dev/sda2  partition  9.1TiB  yes        ST20000NM007D-3D  /devices/pci0000:00/0000:00:17.0/ata6/host5/target5:0:0/5:0:0:0/block/sda/sda2  8      2      "/dev/disk/by-id/scsi-0ATA_ST20000NM007D-3D_ZVTDD0BH-part2", "/dev/disk/by-id/ata-ST20000NM007D-3DJ103_ZVTDD0BH-part2", "/dev/disk/by-partlabel/primary", "/dev/disk/by-id/scsi-35000c500e82ef984-part2", "/dev/disk/by-partuuid/9ac76c9f-bf23-40a0-816c-d4a2b4851d26", "/dev/disk/by-id/scsi-SATA_ST20000NM007D-3D_ZVTDD0BH-part2", "/dev/disk/by-path/pci-0000:00:17.0-ata-6-part2", "/dev/disk/by-id/wwn-0x5000c500e82ef984-part2", "/dev/disk/by-id/scsi-1ATA_ST20000NM007D-3DJ103_ZVTDD0BH-part2"
```

```yaml
apiVersion: "openebs.io/v1beta1"
kind: DiskPool
metadata:
  name: pool-on-node-1
  namespace: mayastor
spec:
  node: workernode-1-hostname
  disks: ["/dev/disk/by-id/<id>"]
```



## 操作命令

文档： https://github.com/openebs/mayastor-extensions/blob/develop/k8s/plugin/README.md

```bash
# 节点
[xiu-tb] ~ k mayastor get nodes
 ID         GRPC ENDPOINT          STATUS 
dev1        192.168.120.181:10124  Online 
dev3        192.168.120.183:10124  Online 
dev4        192.168.120.184:10124  Online 
dev2        192.168.120.182:10124  Online

# 获取卷
~ kubectl mayastor get volumes
 ID                                    REPLICAS  TARGET-NODE  ACCESSIBILITY  STATUS  SIZE  THIN-PROVISIONED  ALLOCATED  SNAPSHOTS  SOURCE 
 e385cd45-fade-4085-944e-de068b13d9ec  2         node4        nvmf           Online  1GiB  false             1GiB       0          <none>

# Pools
k get dsp -n mayastor
--
NAME                 NODE    STATE     POOL_STATUS   CAPACITY      USED         AVAILABLE
diskpool-node1       node1   Created   Online        21449670656   3221225472   18228445184
diskpool-node2       node2   Created   Online        21449670656   2147483648   19302187008
diskpool-node2-vdc   node2   Created   Online        21449670656   0            21449670656
diskpool-node3       node3   Created   Online        21449670656   3221225472   18228445184

# 或者
k mayastor get pools       
--
 ID                  DISKS                                                     MANAGED  NODE   STATUS  CAPACITY  ALLOCATED  AVAILABLE  COMMITTED 
 diskpool-node1      aio:///dev/vdb?uuid=22dcbd42-82a7-4932-8cc4-a7b80881c521  true     node1  Online  20GiB     3GiB       17GiB      3GiB 
 diskpool-node3      aio:///dev/vdb?uuid=03b1060e-ef1d-4db7-b748-41a92e66bffd  true     node3  Online  20GiB     3GiB       17GiB      3GiB 
 diskpool-node2      aio:///dev/vdb?uuid=87f91bf1-2d23-427b-9746-4c5f75927c01  true     node2  Online  20GiB     2GiB       18GiB      2GiB 
 diskpool-node2-vdc  aio:///dev/vdc?uuid=046b02ed-5c7d-4113-b17f-22005441569f  true     node2  Online  20GiB     0 B        20GiB      0 B

# 检索特定卷的复制拓扑结构
[xiu-tb] ~  kubectl mayastor get volume-replica-topology 525fa92f-d15c-4d2f-beb0-b7b5bbb3449a
 ID                                    NODE       POOL            STATUS  CAPACITY  ALLOCATED  SNAPSHOTS  CHILD-STATUS  REASON  REBUILD 
 2bd0c445-aee0-415d-b4ae-59815715be27  dev3       pool-on-node-3  Online  5GiB      5GiB       0 B        Online        <none>  <none> 


```









## 问题记录

1. 

由于某些原因，存储卷已经挂载到目标节点上：

```bash
Disk /dev/nvme2n2: 7.102 GiB, 8584674304 bytes, 16766942 sectors
Disk model: Mayastor NVMe controller                
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 131072 bytes
```

但是没有继续mount到指定目录，如：

```
/var/lib/kubelet/plugins/kubernetes.io/csi/io.openebs.csi-mayastor/583bce23822c5277d79c6ff8baf082cc6cca290e252ba079a2c59e928e44e07c/globalmount
```

pod报错如下：

```
Failed to publish volume 634cad9c-472a-4109-b182-6806798f0617: volume is staged as "ro" but publish requires "rw"

Failed to stage volume 634cad9c-472a-4109-b182-6806798f0617: device attach timeout

```

手动删除pod，pod调度到原来的节点，问题依然存在。



思路是将 pod 调度到别的node，让 mayastor 组建自动处理。

可以将原来的 node 做 cordon，`k cordon dev2`

将pod删除，pod调度到其他节点，mayastor 将存储卷挂载到其他节点上。

在将 node uncordon，`k uncordon dev2`。









## 小结


