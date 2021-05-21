StorageClass

<!-- TOC -->

- [介绍](#介绍)
- [Provisioner](#provisioner)
- [Reclaim Policy](#reclaim-policy)
- [Allow Volume Expansion[允许扩展]](#allow-volume-expansion允许扩展)
- [Mount Options](#mount-options)
- [Volume Binding Mode](#volume-binding-mode)
- [Allowed Topologies](#allowed-topologies)
- [Parameters](#parameters)

<!-- /TOC -->

# 介绍  
StorageClass 为管理员提供了描述存储 "类" 的方法。 不同的类型可能会映射到不同的服务质量等级或备份策略，或是由集群管理员制定的任意策略。 Kubernetes 本身并不清楚各种类代表的什么。这个类的概念在其他存储系统中有时被称为 "配置文件"。

StorageClass资源  
每个StorageClass都包含 provisioner、parameters、reclaimPolicy字段

StorageClass 对象的命名很重要，用户使用这个命名来请求生成一个特定的类。且一旦创建了对象就不能再对其进行更新。

管理员可以为没有申请绑定到StorageClass的pvc指定一个默认的存储类。

# Provisioner
每个StorageClass都有一个Provisioner，用来决定使用哪个卷插件提供pv。

# Reclaim Policy
reclaimPolicy字段指定回收策略，其值可为：
　　Delete （default）
　　Retain

# Allow Volume Expansion[允许扩展]
allowVolumeExpansion可以配置为可扩展，允许用户通过编辑响应的pvc对象来调整卷大小。
以下类型的卷支持卷扩展：
```
　　gcePersistentDisk
　　awsElasticBlockStore
　　Cinder
    glusterfs
    rbd
    Azure File
    Portworx
    FlexVolume
    Csi
```
> **注意**：此功能仅可用于扩展卷，不能用于缩小卷
　　
# Mount Options
由 StorageClass 动态创建的 PersistentVolume 将使用类中 mountOptions 字段指定的挂载选项。  

如果卷插件不支持挂载选项，却指定了该选项，则制备操作会失败。 挂载选项在 StorageClass 和 PV 上都不会做验证，所以如果挂载选项无效，那么这个 PV 就会失败。

# Volume Binding Mode
volumeBindingMode字段控制了卷绑定和动态制备应该发生在什么时候。其值为：  
　　Immediate （default）表示一旦创建了 PersistentVolumeClaim 也就完成了卷绑定和动态制备。   
　　WaitForFirstConsumer该模式将延迟 PersistentVolume 的绑定和制备，直到使用该 PersistentVolumeClaim 的 Pod 被创建。   

以下插件支持动态供应的 WaitForFirstConsumer 模式：
> AWSElasticBlockStore  
> GCEPersistentDisk  
> AzureDisk  

以下插件支持预创建绑定PersistentVolume的 waitForFirstConsumer 模式：
> AWSElasticBlockStore  
> GCEPersistentDisk  
> AzureDisk  
> Local  

>**注意**：动态配置和预先创建的PV也支持CSI卷，但需要查看特定CSI驱动程序的文档以查看其支持的拓扑键名和例子

# Allowed Topologies
如果使用waitForFirstConsumer的模式，一般情况下没有必要将制备限制为特定的拓扑结构。如果有需要可以使用allowedTopologies

这个例子描述了如何将供应卷的拓扑限制在特定的区域，在使用时应该根据插件 支持情况替换 zone 和 zones 参数。

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
volumeBindingMode: WaitForFirstConsumer
allowedTopologies:
- matchLabelExpressions:
  - key: failure-domain.beta.kubernetes.io/zone
    values:
    - us-central1-a
    - us-central1-b
```

# Parameters
Storage Classes 的参数描述了存储类的卷。不同的制备器，可以接受不同的参数。 例如，参数 type 的值 io1 和参数 iopsPerGB 特定于 EBS PV。 当参数被省略时，会使用默认值。  
一个 StorageClass 最多可以定义 512 个参数。这些参数对象的总长度不能 超过 256 KiB, 包括参数的键和值。

具体例子参考官网文档：
https://kubernetes.io/zh/docs/concepts/storage/storage-classes/









yaml filed：
```
FIELDS:
   allowVolumeExpansion	<boolean>
   allowedTopologies	<[]Object>
      matchLabelExpressions	<[]Object>
         key	<string>
         values	<[]string>
   apiVersion	<string>
   kind	<string>
   metadata	<Object>
      annotations	<map[string]string>
      clusterName	<string>
      creationTimestamp	<string>
      deletionGracePeriodSeconds	<integer>
      deletionTimestamp	<string>
      finalizers	<[]string>
      generateName	<string>
      generation	<integer>
      labels	<map[string]string>
      managedFields	<[]Object>
         apiVersion	<string>
         fieldsType	<string>
         fieldsV1	<map[string]>
         manager	<string>
         operation	<string>
         time	<string>
      name	<string>
      namespace	<string>
      ownerReferences	<[]Object>
         apiVersion	<string>
         blockOwnerDeletion	<boolean>
         controller	<boolean>
         kind	<string>
         name	<string>
         uid	<string>
      resourceVersion	<string>
      selfLink	<string>
      uid	<string>
   mountOptions	<[]string>
   parameters	<map[string]string>
   provisioner	<string>
   reclaimPolicy	<string>
   volumeBindingMode	<string>

```