---
title: GameFramework项目学习之ResourceManager(3)
date: 2026-05-08
categories: [项目学习]
tags: [游戏框架, 游戏开发]
description: 分析资源管理器的资源加载流程
---

上一篇笔记中记录了资源管理器中是如何一步步从资源版本文件到实际资源下载的
在这篇笔记中将重点放在已有实际资源的情况下是如何加载的

# ResourceLoader

首先通过资源管理器提供的接口调用对应`Asset`的加载,转发到`ResourceLoader`中进行真正的加载

## 1. 先进行资源的检查

即检查资源管理器中该资源是否存在且已经准备好了,如果准备好了则可以进行下一步的加载(对于可以边玩边下载的模式,则即使没准备好也认为可以进行下一步)

## 2. 进行资源的加载

对于通过上一步检查的资源而言,认为可以进行加载了
则创建主任务

```cs
LoadAssetTask mainTask = LoadAssetTask.Create(
    assetName,
    assetType,
    priority,
    resourceInfo,
    dependencyAssetNames,
    loadAssetCallbacks,
    userData
);
```
如果主资源有依赖则递归去创建`LoadDependencyAssetTask`去加载依赖资源

把这些创建的任务全部放到任务池子中等待后续调度,关于任务池不在这篇笔记中进行过多分析
大致可以这样理解任务池的工作原理:
- LoadAsset 创建加载任务，并加入 TaskPool 的等待队列。
- 每帧 TaskPool.Update 会寻找空闲 Agent
- 空闲 Agent 调用 Start(task)，尝试执行任务
- Agent 内部根据 ResourceInfo 判断是否可加载、是否要等待依赖、是否命中对象池
- 如果需要读取资源，Agent 调用注入的 ILoadResourceAgentHelper 接口
- Helper 负责真正的异步读取、解析、加载，并通过事件通知 Agent
- Agent 收到完成事件后注册 ResourceObject / AssetObject，调用成功或失败回调，并标记 task.Done
- 下一次 TaskPool.Update 发现任务完成，把 Agent 重置并放回空闲队列

总体而言这里的`Agent`可以理解为一个任务执行槽位,需要注意的是这里的`Agent`框架已经给我们实现好了为`LoadResourceAgent`,但是这个Agent缺少一个底层加载资源的接口,即我们上篇文章中提到的`ILoadResourceAgentHelper`

当Agent开始工作后的流程应该是这样的
```
Agent.Start(task)
        ↓
查 AssetPool
        ↓
发现已有 AssetObject
        ↓
OnAssetObjectReady(assetObject)
        ↓
直接调用成功回调
        ↓
m_Task.Done = true
        ↓
返回 StartTaskStatus.Done
        ↓
TaskPool 释放任务，Agent 回空闲池
```

游戏逻辑就可以通过一开始调用`LoadAsset`时候的回调来对得到的资源进行操作

暂时就写这么多,之后如果需要更详细的分析再回过头来填坑吧...
