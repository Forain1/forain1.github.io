---
title: 游戏编程模式
date: 2026-04-29
categories: [阅读笔记]
tags: [设计模式, 游戏开发]
description: 此篇文章用来记录书籍<<游戏编程模式>>的阅读过程, 原文采用 cpp 实现设计模式, 尝试使用 C# 重写
---


# 前言

之所以选择阅读这本书是因为笔者在之前就了解过一点设计模式,但是都是处于应付八股面试,并没有真正的去理解和深入学习各种设计模式的优缺点,更不用说如何真正运用到项目中了

当笔者在真正实习写项目的时候,发现自己写的代码可维护性过差,写到后面自己看自己的代码都觉得呕吐,因此决定系统性的阅读一下设计模式相关的书籍,希望能以此改善一下自己的代码风格

在写下这篇阅读笔记前,笔者已经整体阅读过此书,因此仅作总结和提取,以强化自己的记忆

同时这本书是用cpp来举的例子,由于笔者目前主要使用的是Unity进行的游戏开发,因此一些概念可能会同时用cpp或者cs进行描述

# 经典设计模式

这本书大致可以分为两个部分,其中第一部分主要是对经典的设计模式进行系统性的解释,主要参考了Gof的说法

## 命令模式

> 将一个请求封装成一个对象,从而允许你使用不同的请求将客户端参数化,同时支持请求操作的undo和redo
{: .prompt-tip}

文中举出来的例子主要是按键绑定这一现象,由于游戏中往往需要给用户改键的功能,因此当处理用户的输入的时候,我们不应该根据案件输入直接去调用对应的函数

更加合理的做法应该是通过一个指针(or引用类型)存储一个命令对象,通过按键绑定不同的命令对象,从而实现改键后调用不同的命令


使用命令模式前代码可能如下
```cs
public class InputHandler
{
    void HandleInput(){
      if (isPressed(BUTTON_X)) jump();
      else if (isPressed(BUTTON_Y)) fireGun();
      else if (isPressed(BUTTON_A)) swapWeapon();
      else if (isPressed(BUTTON_B)) lurchIneffectively();
    }
}
```

使用命令模式后的代码如下
```cs
public class InputHandler
{
    void HandleInput(){
        if (isPressed(BUTTON_X)) buttonX_.execute();
        else if (isPressed(BUTTON_Y)) buttonY_.execute();
        else if (isPressed(BUTTON_A)) buttonA_.execute();
        else if (isPressed(BUTTON_B)) buttonB_.execute();
    }

  ICommand buttonX_;
  ICommand buttonY_;
  ICommand buttonA_;
  ICommand buttonB_;
}
```

原文中使用继承来限定接口的实现,但是本身C#就自带接口这一语法

```cs
public interface ICommand
{
  public void Execute();
  public void Undo();
}
```

之后其他的命令都必须继承该接口,这里取书中的例子进行改造来展示
```cs
public class MoveUnitCommand : ICommand
{
    private Unit unit;
    private int x;
    private int y;
    private int xBefore;
    private int yBefore;

    public MoveUnitCommand(Unit unit, int x, int y)
    {
        this.unit = unit;
        this.x = x;
        this.y = y;

        xBefore = 0;
        yBefore = 0;
    }

    public void Execute()
    {
        // 记录移动前的位置，方便撤销
        xBefore = unit.X;
        yBefore = unit.Y;

        unit.MoveTo(x, y);
    }

    public void Undo()
    {
        unit.MoveTo(xBefore, yBefore);
    }
}
```

如果要具体实现redo和undo的话,只需要用一个栈来维护即可

书中提到,实际上类实现命令模式是对有闭包语言的模仿(虽然cpp11也有闭包相关,但是笔者没怎么用过),实际上是挺有道理的,因为我们就是在execute函数执行前维护了一系列这个命令所需要的数据,而这正是闭包函数替我们做的事情(即延长变量的生命周期)


## 享元模式

> 使用共享以高效地支持大量的细粒度对象
{: .prompt-tip}

书中用GPU渲染时CPU传输资源为例子,实际上用一句话总结就是共享的数据在类中用指针(引用对象)来指向


对于每一颗树,我们定义下面的结构
```cs
public class Tree
{
    private readonly TreeType _type;

    public float X { get; }
    public float Y { get; }

    public Tree(TreeType type, float x, float y)
    {
        _type = type;
        X = x;
        Y = y;
    }

    public void Draw()
    {
        _type.Draw(X, Y);
    }
}
```

TreeType为树所共享的数据
```cs
public class TreeType
{
    public string Mesh { get; }
    public string Texture { get; }
    public string Material { get; }

    public TreeType(string mesh, string texture, string material)
    {
        Mesh = mesh;
        Texture = texture;
        Material = material;
    }

    public void Draw(float x, float y)
    {
        Console.WriteLine($"在 ({x}, {y}) 绘制 {Mesh}, {Texture}, {Material}");
    }
}
```

为了维持每种type只实例化出来一份,还可以考虑用工厂模式来生成
```cs
public class TreeTypeFactory
{
    private readonly Dictionary<string, TreeType> _treeTypes = new();

    public TreeType GetTreeType(string name, string mesh, string texture, string material)
    {
        if (!_treeTypes.TryGetValue(name, out TreeType type))
        {
            type = new TreeType(mesh, texture, material);
            _treeTypes[name] = type;
        }

        return type;
    }
}
```

这样一来重复的数据就消除了

## 观察者模式

>在对象间定义一种一对多的依赖关系，以便当某对象的状态改变时，与它存在依赖关系的所有对象都能收到通知并自动进行更新
{: .prompt-tip}

对于观察者模式,C#直接使用event关键字来集成这一模式了

实际上这一模式又和MVC模式紧密相关,因为MVC的底层实际上就是观察者模式

观察者(subject)对于观察者(observer)来说是透明的

书中用cpp实现观察者模式的方法是subject通过一个链表来维护一串observer,观察者需要继承一个通知触发接口,同时需要保存一个指向下一个观察者指针,甚至可以结合之前的享元模式的思维进行优化,即链表中维护节点,节点中实现链表的基本结构,再通过一个指针指向我们真正的observer,这样一来就可以保证observer可以一对多

实现这个模式需要注意的除了注册observer以外,要注意当observer被销毁的时候需要注销


一个subject的例子
```cs
public class PlayerData
{
    public int Gold { get; private set; }

    public event Action<int> OnGoldChanged;

    public void AddGold(int amount)
    {
        Gold += amount;

        OnGoldChanged?.Invoke(Gold);
    }
}
```

一个observer的例子
```cs
public class GoldUI : MonoBehaviour
{
    public PlayerData playerData;
    public TMP_Text goldText;

    private void OnEnable()
    {
        playerData.OnGoldChanged += Refresh;
    }

    private void OnDisable()
    {
        playerData.OnGoldChanged -= Refresh;
    }

    private void Refresh(int gold)
    {
        goldText.text = gold.ToString();
    }
}
```

其中
PlayerData 负责发布事件
GoldUI 负责订阅事件

## 原型模式

>使用特定原型实例来创建特定种类的对象，并且通过拷贝原型来创建新的对象
{: .prompt-tip }

实现的效果实际上类似于Unity当中的预制体实例化,这里的预制体实际上就是原型
原型模式就是为了解决同样的对象,由于配置过程可能过于繁琐,因此我们可以实现一个原型,让原型具备克隆方法,从而实现由一个已有的对象生成另一个对象


一个接口可能设计如下
```cs
public interface IPrototype<T>{
  public T Clone();
}
```

```cs
public class Monster : IPrototype<Monster>
{
    public string Name;
    public int MaxHp;
    public int Attack;
    public float MoveSpeed;

    public Monster Clone()
    {
        return new Monster
        {
            Name = this.Name,
            MaxHp = this.MaxHp,
            Attack = this.Attack,
            MoveSpeed = this.MoveSpeed
        };
    }
}
```

但是需要注意的是,对于拷贝的浅拷贝和深拷贝可能是需要深思的地方,在使用时需要注意

书中提到的一个我认为比较有意思的就是可以实现原型链,通过原型链找到最近可以拷贝数据的原型,从而简化对数据的配置




## 单例模式

>确保一个类只有一个实例，并为其提供一个全局访问入口
{: .prompt-tip }

这个笔者在自己的项目中经常使用,但是书中却说最好不要使用这一模式,想来确实有道理,单例模式本身就类似于全局变量的存在,而全局变量在多人合作的大项目中是禁忌,因此也许之后在写项目的时候需要少用这一模式

常用在各种`Manager`中

单例模式又有懒汉和饱汉风格,前者是用到才生成,后者是一开始就生成单例

## 状态模式

>允许一个对象在其内部状态改变时改变自身的行为。对象看起来好像是在修改自身类
{: .prompt-tip }

本质就是把各种if语句给拆到各种state类中,对象中只保存当前state的引用,然后通过当前stage去进行update状态等一系列操作

在计算理论这门课中(~~除去离散数学最讨厌的课~~)有提到有限状态机这一概念,而状态模式就是对这一概念的上层实现


```cs
public interface IPlayerState
{
    void Enter(Player player);
    void HandleInput(Player player);
    void Update(Player player);
    void Exit(Player player);
}
```


```cs
public class Player
{
    private IPlayerState currentState;

    public Player()
    {
        ChangeState(new IdleState());
    }

    public void ChangeState(IPlayerState newState)
    {
        currentState?.Exit(this);
        currentState = newState;
        currentState.Enter(this);
    }

    public void HandleInput()
    {
        currentState.HandleInput(this);
    }

    public void Update()
    {
        currentState.Update(this);
    }

    public void Jump()
    {
        Console.WriteLine("玩家跳跃");
    }

    public void Move()
    {
        Console.WriteLine("玩家移动");
    }

    public void PlayAnimation(string animName)
    {
        Console.WriteLine($"播放动画：{animName}");
    }
}
```

虽然`Player`提供了一系列的操作函数,但是都不是Player自己调用的,而是state去调用的


除去上面最简单的有限状态机以外,书中还提到了并发状态机(即同时存在多个状态机,互不干扰),层次状态机(提取状态机状态共有特性为基类),下推状态机(多了一个栈供状态机使用)


## 序列型模式

To be done...
