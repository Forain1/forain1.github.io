---
title: GameFramework项目学习之ResourceManager(2)
date: 2026-05-08
categories: [项目学习]
tags: [游戏框架, 游戏开发]
description: 分析资源管理器的资源管理大致流程,分别从单机和可更新模式进行入手
---


## 资源模块的配置步骤

### 1. 设置只读区和读写区

```cs
SetReadOnlyPath(string readOnlyPath)//如Unity中的streamingAssets
SetReadWritePath(string readWritePath)//如Unity中的persistentDataPath
```


### 2. 设置资源模式

需要调用`SetResourceMode(ResourceMode resourceMode)`来设置
需要注意的是,一旦资源模式被设置了,就不能够进行修改

对于这个单机模式,只需要创建以下两个组件

```cs
PackageVersionListSerializer
ResourceIniter
```

读取包体版本列表 -> 建立资源索引 -> 可以加载



对于联机模式而言
需要创建以下几个组件
```
UpdatableVersionListSerializer
ReadOnlyVersionListSerializer
ReadWriteVersionListSerializer
ResourcePackVersionListSerializer
VersionListProcessor
ResourceChecker
ResourceUpdater
```

检查版本列表 -> 校验资源 -> 检查差异 -> 下载更新 -> 可以加载

### 3. 设置当前变体

需要调用`SetCurrentVariant(string currentVariant)`
如果没有资源变体,则可以不进行设置

### 4. 设置对象池管理器

`SetObjectPoolManager(IObjectPoolManager objectPoolManager)`,通过这个接口来设置对象池管理器
它实际转交给`ResourceLoader`

在这个加载器中,会利用这个管理器来创建两个池子
```cs
m_AssetPool = objectPoolManager.CreateMultiSpawnObjectPool<AssetObject>("Asset Pool");
m_ResourcePool = objectPoolManager.CreateMultiSpawnObjectPool<ResourceObject>("Resource Pool");
```

所以之后加载同一个资源时，可以先查对象池，不一定重新读磁盘

### 5. 设置`ResourceHelper`

实现具体接口`IResourceHelper`,用来读版本文件、卸载 Scene、释放 AssetBundle/对象
可以理解为和引擎层进行桥接的接口

### 6. 添加加载资源代理 Helper

实现具体接口`ILoadResourceAgentHelper`,用来异步读 AssetBundle、解析 AssetBundle、从 AssetBundle.LoadAssetAsync 加载具体资源
也是桥接接口,需要自己实现


在了解了基本的配置步骤后,下面从单机模式和可更新模式两种资源加载形式进行流程分析

## 单机模式

启动流程大致如下
```cs
SetReadOnlyPath
SetReadWritePath
SetResourceMode(Package)
SetCurrentVariant
SetObjectPoolManager
SetResourceHelper
AddLoadResourceAgentHelper

InitResources
LoadAsset / LoadScene / LoadBinary
```

主要看`InitResources`这个方法
它进行了以下几步比较关键的步骤:
1. 读取版本列表
2. 反序列化`PackageVersionList`,需要注意的是,关于版本列表的序列化和反序列化回调都需要上层注入,即该框架没有相应的具体实现,只提供了一个文件头部信息标记的序列化和反序列化
  PackageVersionList 里主要有四类数据：
     - Asset[]	所有可加载的 Asset 名称和依赖
     - Resource[]	所有资源包信息
     - FileSystem[]	哪些 Resource 存在虚拟文件系统中
     - ResourceGroup[]	资源组信息
3. 根据上一步得到的列表信息,存成更直接的数据形式(为了节省空间,我们在版本列表中存放的是索引信息,而这里需要更直接,即名字),于此同时,这一步也会进行变体的过滤
   - Asset -> AssetInfo 
   - Resource -> ResourceInfo
   - 还有文件系统和资源组的生成,不过如果版本列表实现没有的话,这两个就为空
4. 完成后调用回调`ResourceInitComplete()`,回到管理器,之后就可以进行`LoadAsset`了


## 可更新模式

启动流程大致如下
```cs
SetReadOnlyPath
SetReadWritePath
SetResourceMode(Updatable 或 UpdatableWhilePlaying)
SetCurrentVariant
SetObjectPoolManager
SetFileSystemManager
SetDownloadManager
SetResourceHelper
AddLoadResourceAgentHelper

CheckVersionList
UpdateVersionList
VerifyResources
CheckResources
UpdateResources/ApplyResources

LoadAsset / LoadScene / LoadBinary
```

首先上层需要获取远程版本的元信息,在此框架中没有内置,需要自己手动实现
```cs
latestInternalResourceVersion
versionListLength
versionListHashCode
versionListCompressedLength
versionListCompressedHashCode
UpdatePrefixUri
```
框架通过这些变量信息才能判断和下载最新版本列表


### 1. `CheckVersionList`
主要依靠于`VersionListProcessor`来进行对应版本号的检查

首先检查本地读写区是否存在资源版本文件,如果没有则返回`NeedUpdate`,如果有则读取资源版本文件,然后通过解析出来版本号和上层的版本号进行判断
若版本号不同也需要返回`NeedUpdate`

### 2. `UpdateVersionList`

如果前面一步返回了`NeedUpdate`,那么则需要调用该函数
```cs
UpdateVersionList(
    versionListLength,
    versionListHashCode,
    versionListCompressedLength,
    versionListCompressedHashCode,
    callbacks
)
```

通过该函数去下载远程的资源版本文件,下载后会进行长度,CRC的校验

### 3. `VerifyResources`

通过该函数来进行资源的验证,每一帧可以规定进行验证的资源大小为多少
即调用了该函数后,在文件加载完毕后通过回调不是立马进行全部资源文件的校验,而是生成一个`List<VerifyInfo>`变量,之后在每帧的`Update`中不断取出每个`VerifyInfo`对象进行校验
进行一系列验证操作后,会判断这个资源是否合法,如果不合法则将其从这个list中剔除
只要中途有一个资源不合法,则需要等到最后再依据已有的`List<VerifyInfo>`重新生成一份读写区的versionlist
需要注意的是,对于可更新模式,和单机模式不同
单机模式下,只需要一份PackageVersionList即可解决所有资源元信息,且默认资源随包下载为可信的
再可更新模式下,一共有三份资源版本文件,起到不同的作用,在下个部分就会说明其作用,这里做简要解释:
- `ReadOnlyPath/GameFrameworkList.dat`:用来告诉框架只读区实际有哪些 Resource,是带包就下载好的,与可更新模块关联不大
- `ReadWritePath/GameFrameworkList.dat`:记录读写区实际已经下载/应用了哪些 Resource,用于记录热更新资源实际在本地合法的资源有哪些
- `ReadWritePath/GameFrameworkVersion.dat`:判断当前版本应该有哪些 Asset 和 Resource,用来记录实际上应该有的资源有哪些,从远程服务端下载下来的就是这一份

### 4. `CheckResources`

同时读取上述的三张表,然后生成`CheckInfo`,把资源对应的`CheckInfo`中的versioninfo给填入,如果没有这个versioninfo则制空
```cs
private readonly ResourceName m_ResourceName;
private CheckStatus m_Status;
private bool m_NeedRemove;
private bool m_NeedMoveToDisk;
private bool m_NeedMoveToFileSystem;
private RemoteVersionInfo m_VersionInfo;
private LocalVersionInfo m_ReadOnlyInfo;
private LocalVersionInfo m_ReadWriteInfo;
private string m_CachedFileSystemName;
```

由于远程服务端下载下来的资源版本文件我们认为是可信的,因此所有资源我们都能至少生成一份`CheckInfo`
经过Check后就能够得到资源的下面几种状态:
StorageInReadOnly   -> 只读区已有，且和最新版本一致
StorageInReadWrite  -> 读写区已有，且和最新版本一致
Update              -> 本地没有，或本地资源和最新版本不一致
Unavailable         -> 资源存在，但不是当前 Variant
Disuse              -> 最新版本表里已经没有这个资源

然后根据这几种状态就可以存入到`ResourceManager`中的`m_ResourceInfos`中,只不过`m_Ready`字段根据实际填入

需要Update的资源通过回调由`ResourceManager`转发到`ResourceUpdater`中进行保存

### 5. `UpdateResources`

在上一步执行完后,我们的更新器中已经保存了需要更新的资源信息,但是此时只是进行登记,并没有进行资源的实际更新
等上层进行`resourceManager.UpdateResources(callback)`的调用后,才会真正进行Update,同理也不是直接在这个函数中直接更新完
而是放入更新队列中,然后再由资源管理器每一帧去调用对应的更新函数,然后再在其中从更新队列中取出资源信息然后进行资源的下载

在`UpdatableWhilePlaying`模式下,`LoadAsset`遇到`Ready=false`时调用`UpdateResource`立即触发下载,而不是直接一口气全部Update

如果这一步使用的是`ApplyResources`,则是使用已经下载的资源包进行分析,取出自己所需要的资源包来进行使用
