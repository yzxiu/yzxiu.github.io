# virtualbox 使用命令行创建虚拟机


## 概述：
> 以下内容为ChatGPT生成

virtualbox 使用命令行创建虚拟机，要求使用现有的vdi文件作为硬盘，网络使用enp3s0作为侨接



以下是使用命令行创建虚拟机的步骤：

1. 打开终端，进入VirtualBox安装目录下的VBoxManage所在目录。

2. 输入以下命令，创建一个名为test的虚拟机，使用现有的vdi文件作为硬盘，网络使用enp3s0作为侨接：

```
VBoxManage createvm --name "test" --register
VBoxManage storagectl "test" --name "SATA Controller" --add sata --controller IntelAHCI
VBoxManage storageattach "test" --storagectl "SATA Controller" --port 0 --device 0 --type hdd --medium /path/to/vdi/file.vdi
VBoxManage modifyvm "test" --nic1 bridged --bridgeadapter1 enp3s0
```

其中，/path/to/vdi/file.vdi需要替换成实际的vdi文件路径，enp3s0需要替换成实际的网络接口名称。

3. 输入以下命令，设置虚拟机的CPU和内存大小：

```
VBoxManage modifyvm "test" --cpus 2 --memory 2048
```

其中，--cpus 2表示虚拟机使用2个CPU核心，--memory 2048表示虚拟机分配2048MB内存。

4. 输入以下命令，启动虚拟机：

```
VBoxManage startvm "test"
```

至此，使用命令行创建虚拟机并使用现有的vdi文件作为硬盘，网络使用enp3s0作为侨接的操作完成。
