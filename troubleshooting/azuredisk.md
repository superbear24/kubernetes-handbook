# AzureDisk 排错

[AzureDisk](https://docs.microsoft.com/zh-cn/azure/virtual-machines/windows/about-disks-and-vhds) 为 Azure 上面运行的虚拟机提供了弹性块存储服务，它以 VHD 的形式挂载到虚拟机中，并可以在 Kubernetes 容器中使用。

根据配置的不同，Kubernetes 支持的 AzureDisk 可以分为以下几类

- Managed Disks: 由 Azure 自动管理磁盘和存储账户
- Blob Disks:
  - Dedicated (默认)：为每个 AzureDisk 创建单独的存储账户，当删除 PVC 的时候删除该存储账户
  - Shared：AzureDisk 共享 ResourceGroup 内的同一个存储账户，这时删除 PVC 不会删除该存储账户

> 注意：AzureDisk 的类型必须跟 VM OS Disk 类型一致，即要么都是 Manged Disks，要么都是 Blob Disks。当两者不一致时，AzureDisk PV 会报无法挂载的错误。

使用 [acs-engine](https://github.com/Azure/acs-engine) 部署的 Kubernetes 集群，会自动创建两个 StorageClass，默认为managed-standard（即HDD）：

```sh
kubectl get storageclass
NAME                PROVISIONER                AGE
default (default)   kubernetes.io/azure-disk   45d
managed-premium     kubernetes.io/azure-disk   53d
managed-standard    kubernetes.io/azure-disk   53d
```

## AzureDisk 挂载失败

在 AzureDisk 从一个 Pod 迁移到另一 Node 上面的 Pod 时或者同一台 Node 上面使用了多块 AzureDisk 时有可能会碰到这个问题。这是由于 kube-controller-manager 未对 AttachDisk 和 DetachDisk 操作加锁从而引发了竞争问题（[kubernetes#60101](https://github.com/kubernetes/kubernetes/issues/60101) [acs-engine#2002](https://github.com/Azure/acs-engine/issues/2002) [ACS#12](https://github.com/Azure/ACS/issues/12)）。

通过 kube-controller-manager 的日志，可以查看具体的错误原因。常见的错误日志为

```sh
Cannot attach data disk 'cdb-dynamic-pvc-92972088-11b9-11e8-888f-000d3a018174' to VM 'kn-edge-0' because the disk is currently being detached or the last detach operation failed. Please wait until the disk is completely detached and then try again or delete/detach the disk explicitly again.
```

临时性解决方法为

（1）更新所有受影响的虚拟机状态

```powershell
$vm = Get-AzureRMVM -ResourceGroupName $rg -Name $vmname  
Update-AzureRmVM -ResourceGroupName $rg -VM $vm -verbose -debug
```

（2）重启虚拟机 

- `kubectl cordon NODE`
- 如果 Node 上运行有 StatefulSet，需要手动删除相应的 Pod
- `kubectl drain NODE`
- `Get-AzureRMVM -ResourceGroupName $rg -Name $vmname | Restart-AzureVM`
- `kubectl uncordon NODE`

该问题的修复 [#60183](https://github.com/kubernetes/kubernetes/pull/60183) 将包含在 v1.10 中。

## 挂载新的 AzureDisk 后，该 Node 中其他 Pod 已挂载的 AzureDisk 不可用

在 Kubernetes v1.7 中，AzureDisk 默认的缓存策略修改为 `ReadWrite`，这会导致在同一个 Node 中挂载超过 5 块 AzureDisk 时，已有 AzureDisk 的盘符会随机改变（[kubernetes#60344](https://github.com/kubernetes/kubernetes/issues/60344) [kubernetes#57444](https://github.com/kubernetes/kubernetes/issues/57444) [AKS#201](https://github.com/Azure/AKS/issues/201) [acs-engine#1918](https://github.com/Azure/acs-engine/issues/1918)）。比如，当挂载第六块 AzureDisk 后，原来 lun0 磁盘的挂载盘符有可能从 `sdc` 变成 `sdk`：

```sh
$ tree /dev/disk/azure
...
â””â”€â”€ scsi1
    â”œâ”€â”€ lun0 -> ../../../sdk
    â”œâ”€â”€ lun1 -> ../../../sdj
    â”œâ”€â”€ lun2 -> ../../../sde
    â”œâ”€â”€ lun3 -> ../../../sdf
    â”œâ”€â”€ lun4 -> ../../../sdg
    â”œâ”€â”€ lun5 -> ../../../sdh
    â””â”€â”€ lun6 -> ../../../sdi
```

这样，原来使用 lun0 磁盘的 Pod 就无法访问 AzureDisk 了

```sh
[root@admin-0 /]# ls /datadisk
ls: reading directory .: Input/output error
```

临时性解决方法是设置 AzureDisk StorageClass 的 `cachingmode: None`，如

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: managed-standard
provisioner: kubernetes.io/azure-disk
parameters:
  skuname: Standard_LRS
  kind: Managed
  cachingmode: None
```

该问题的修复 [#60346](https://github.com/kubernetes/kubernetes/pull/60346) 将包含在 v1.10 中。

## AzureDisk 挂载慢

AzureDisk PVC 的挂载过程一般需要 1 分钟的时间，这些时间主要消耗在 Azure ARM API 的调用上（查询 VM 以及挂载 Disk）。[#57432](https://github.com/kubernetes/kubernetes/pull/57432) 为 Azure VM 增加了一个缓存，消除了 VM 的查询时间，将整个挂载过程缩短到大约 30 秒。该修复包含在v1.9.2+ 和 v1.10 中。

## Azure German Cloud 无法使用 AzureDisk

Azure German Cloud 仅在 v1.7.9+、v1.8.3+ 以及更新版本中支持（[#50673](https://github.com/kubernetes/kubernetes/pull/50673)），升级 Kubernetes 版本即可解决。

## 参考文档

- [Known kubernetes issues on Azure](https://github.com/andyzhangx/demo/tree/master/issues)
- [Introduction of AzureDisk](https://docs.microsoft.com/zh-cn/azure/virtual-machines/windows/about-disks-and-vhds) 
- [AzureDisk volume examples](https://github.com/kubernetes/examples/tree/master/staging/volumes/azure_disk)