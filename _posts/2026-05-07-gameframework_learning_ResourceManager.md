---
title: GameFramework项目学习之ResourceManager(1)
date: 2026-05-07
categories: [项目学习]
tags: [游戏框架, 游戏开发]
description: 应该是整个框架中最为庞大的模块了
---

# ResourcesManager

作为第一个正式学习的模块,其实也是所有模块中最庞大的,在学习过程中发现实现的内容有太多太多,因此如果深入实现细节将会迷失方向,因此这个模块仅分析类中的各个部分在做什么,而不深入讨论细节(以后如果有需要再回过头来看吧),其实笔者也在当天就看了具体实现,但是由于代码量太大,看过了很容易忘记,因此不打算在这篇文章中整理

在分析代码之前,需要有一些相关背景:

## 资源

代码中按照AssetBundle的形式将资源分为了`Resource`和`Asset`,可以理解为`Resource`为各种`Asset`的压缩包
即一个`Resource`中存放着多个`Asset`,而`Asset`之间又有依赖关系,这种依赖关系是可以跨`Resource`的,因此代码为了保持这种依赖关系的完整,实现了许多代码来维护,在之后将用`资源`来表示`Resource`,而用`资产`来表示`Asset`

框架中,资源文件名以这种方式命名:`本名`+`变名`(一般用来区分不同语言下的资源)+`扩展名`(.xxx)

## 虚拟文件系统

框架中实现了一个在用户层面的文件系统,主要的目的是为了减少小文件多次IO,即我们可以把用户层面的这个虚拟文件系统理解为一个大的容器,这个容器中包含了多个资源文件

## 资源模式

框架中把资源大致划分为了4种模式:
- `Unspecified `:一般用不到
- `Package`:单机模式,即资源全部下载到本地,一次资源下载后续无需更新
- `Updatable`:预下载的可更新模式,即每次打开游戏需要检查资源是否完备,如果不完备则需要从服务器下载最新的
- `UpdatableWhilePlaying`:使用时下载的可更新模式,大致同上,只不过在游玩过程中就可以下载资源进行更新

## 检查资源状态

框架中是如何判断资源版本是否能够对应得上的呢?
答案就是通过`VersionListFile`,通过对比本地和服务端的版本列表文件来判断
首先通过一个简单的版本号来进行版本列表是否最新的判断
当版本列表是最新的时候,就可以根据已有资源和未有资源的比对来决定下载哪些资源了,后续在代码分析模块会更加详细分析


## 代码分析

### 成员变量

```cs
//服务器远程版本列表文件名
private const string RemoteVersionListFileName = "GameFrameworkVersion.dat";
//本地版本列表文件名
private const string LocalVersionListFileName = "GameFrameworkList.dat";
//默认资源扩展名即资源文件以.dat结尾
private const string DefaultExtension = "dat";
//默认临时文件扩展名
private const string TempExtension = "tmp";
//虚拟文件系统相关参数
private const int FileSystemMaxFileCount = 1024 * 16;
private const int FileSystemMaxBlockCount = 1024 * 256;
```


```cs
//一个资产名到资产信息的映射
private Dictionary<string, AssetInfo> m_AssetInfos;
//一个资源名到资源信息的映射
private Dictionary<ResourceName, ResourceInfo> m_ResourceInfos;
//资源名到读写区资源信息的映射,所谓读写区资源即可更新的资源,是上面的子集,只是为了方便管理和查找
private SortedDictionary<ResourceName, ReadWriteResourceInfo> m_ReadWriteResourceInfos;
//只读区和读写区资源的虚拟文件系统,这里不需要过多了解
private readonly Dictionary<string, IFileSystem> m_ReadOnlyFileSystems;
private readonly Dictionary<string, IFileSystem> m_ReadWriteFileSystems;
//资源组,所谓的资源组就是用来分批下载资源的,通过把资源分批,可以让游戏在不同阶段进行不同资源的下载
private readonly Dictionary<string, ResourceGroup> m_ResourceGroups;
```


- 资产信息

```cs
private readonly string m_AssetName; //资产名
private readonly ResourceName m_ResourceName; //资源名
private readonly string[] m_DependencyAssetNames;//所依赖的资产名
```

- 资源信息

```cs
private readonly ResourceName m_ResourceName;//资源名
private readonly string m_FileSystemName;//所属的文件系统名(无则置空)
private readonly LoadType m_LoadType;//加载方式:主要分为三大类,从文件,从内存,通过二进制
private readonly int m_Length;//资源长度
private readonly int m_HashCode;//资源的哈希码(用于校验)
private readonly int m_CompressedLength;//资源压缩后的长度
private readonly bool m_StorageInReadOnly;//是否存于只读区
private bool m_Ready;//资源是否准备好了
```

- 资源组

```cs
private readonly string m_Name; //组名
private readonly Dictionary<ResourceName, ResourceInfo> m_ResourceInfos;//资源组对应的资源集合
private readonly HashSet<ResourceName> m_ResourceNames;//资源组包含的资源有哪些,主要是用于快速判断是否存在于此资源组中
private long m_TotalLength;//资源组总长度
private long m_TotalCompressedLength;//资源组压缩后的总长度
```


```cs
//一系列序列化器,用来序列化版本信息相关
//单机模式下使用
private PackageVersionListSerializer m_PackageVersionListSerializer;//单机模式下的序列化器

//更新模式(两种)下使用
private UpdatableVersionListSerializer m_UpdatableVersionListSerializer;//可更新资源的序列化器
private ReadOnlyVersionListSerializer m_ReadOnlyVersionListSerializer;//位于只读区资源的序列化器
private ReadWriteVersionListSerializer m_ReadWriteVersionListSerializer;//位于读写区资源的序列化器
private ResourcePackVersionListSerializer m_ResourcePackVersionListSerializer;//资源包序列化器
```

```cs
private IFileSystemManager m_FileSystemManager;//文件系统管理器,由外部插入

private ResourceIniter m_ResourceIniter;//资源初始化器 进行资源系统初始化 
private VersionListProcessor m_VersionListProcessor;//处理远程版本列表的下载和解析
private ResourceVerifier m_ResourceVerifier;//验证下载资源的完整性
private ResourceChecker m_ResourceChecker;//在初始化期间验证资源
private ResourceUpdater m_ResourceUpdater;//管理资源更新下载
private ResourceLoader m_ResourceLoader;//处理资源的加载 依赖关系 对象池化
private IResourceHelper m_ResourceHelper;//负责处理底层的资源加载和释放操作 由顶层提供
```

其他成员变量相对来说不那么重要,有分析到再进行解释