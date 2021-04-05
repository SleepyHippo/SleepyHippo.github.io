---
layout: post
title:  "ILRuntime爬坑笔记2——使用篇"
date:   2021-04-04 18:04:45 +0800
categories: Unity ILRuntime
---

这篇文章主要介绍ILRuntime大致的实现原理，和使用过程中碰到的坑，2019年那会用的ILRuntime版本是1.4和1.6中间的某个master分支上的节点，新版本应该有不少变化。

## **实现原理**

### 基本工作流程是怎么样的？
  - 启动游戏后，下载dll（CIL字节码），调用方法加载dll。
  - 主工程可以用ILR提供的两种方式来调用热更代码：
    1. appdomain.Invoke("Hotfix.Game", "Initialize", null, null); 调用热更工程的静态方法
    2. 也可以用[反射](https://ourpalm.github.io/ILRuntime/public/v1/guide/reflection.html)创建热更中定义的类型的实例对象，并调用实例方法。
  - 热更工程则可以用正常的C#方式直接调用主工程的代码。

### 主工程是怎么调用热更工程的代码的？
  - 主工程是不能直接调用热更代码的（比如直接new对象，直接调用静态方法），能调用的，其实都是ILR的框架代码，ILR的框架代码把主工程调用时传递的参数，转换为解释执行某段热更代码。
  - 热更代码是通过解释器来执行的，其中定义的类型，创建的对象，变量等，在主工程的视角来看，全部都是不存在的，主工程视角看起来，热更中的类型是ILType对象、方法是ILMethod对象、对象是StackObject对象。

### 热更工程代码为什么能调用主工程代码？
  - 主工程定义了一个方法mainFun(int x)，有两种方式能被调用，一种是直接在代码里写mainFunc(1)来调用，另一种是反射。这也对应了热更工程调用主工程的两种实现方式。
  - 一种就是，解释器解释ILR代码后，最终通过反射调用指定的主工程代码，在热更的入口方法里，随意调用一个主工程的函数，在主工程函数里Debug.Log一下，就可以看到完整的调用堆栈
  - 另一种则是，所谓的[CLR绑定](https://ourpalm.github.io/ILRuntime/public/v1/guide/bind.html)的方式，就是对一些主工程经常被调用的方法生成绑定代码，绑定代码分为两部分，一部分在运行时提前用反射拿到要绑定方法的MethodBase，作为字典的key存下来，另一部分则是一个包装方法，内部调用了绑定方法，用ILR的方式处理参数和返回值，这个包装方法会作为字典的value存下来。当ILR检测到热更代码要调用这些主工程代码的时候，就查找字典，改为调用包装方法，从而避开反射的开销。官方有提供Binding的自动生成。并建议只有经常调用的方法，才写Binding。这种提升性能的方式和[这篇文章](http://blogs.msmvps.com/jonskeet/2008/08/09/making-reflection-fly-and-exploring-delegates/)的思路是一致的
  - 还有一些比较特殊的方法，例如Activator.CreateInstance、Type.GetType、MethodBase.Invoke，需要用特殊的方法处理才能在热更工程中调用，这些在ILR框架内使用[CLR重定向](https://ourpalm.github.io/ILRuntime/public/v1/guide/bind.html)的方法做了处理，上面说的CLR绑定也是基于CLR重定向的机制实现的。

### 热更工程的类为什么可以继承主工程的类？
  - 因为热更工程中定义的class subHot，在运行时是不存在的，所以需要定义一个[Adapter](https://ourpalm.github.io/ILRuntime/public/v1/guide/cross-domain.html)来继承主工程的class baseMain，在热更工程里写new subHot()，ILRuntime内部会最后转换成new Adapter()，在调用具体方法的时候，再转去调用subHot的逻辑


## **各种东西怎么实现的？**
- MonoBehavior：
  - 热更工程不方便定义新的MonoBehavior，引用UI控件可以有两种方式，一种是Find之后GetComponent，一种是使用一个通用的MonoBehavior定义`List<UnityEngine.Object>`来保存，我们选择第二种方式，更加灵活一点。
- 协议序列化、数据组件、显示管理器等：
  - 可以复用主工程的基类，如果主工程是泛型类，就把基类拷贝到热更工程中。
- 配置表：
  - 主工程每张表是用一个SO来存储，热更工程没办法新建SO，所以改成和服务器端一样，运行时用CSV方式读取。

## **使用中遇到的问题**

### Adapter
  - Adapter可以自动生成！但是生成的工具build不出来。。github上说工具已经没人维护了。。
  - 手写的话，记得给参数赋值！
  - 如果不带参数，直接传null，不然有gc
  - 属性adapter是当作方法来调用的，方法名是属性名前面加"get_"，参数不要填！（官方Demo是错的）
  - 一定需要一个无参构造函数，目前没想到方法能在构造函数里传参，如果有基类构造函数需要参数，子类构造函数先传空给基类，再开别的方法来赋值
  - 没办法继承泛型基类（除非基类中的T是具体类型）
  - 在热更工程中调用基类的方法，有时候会有问题导致Unity卡死，解决方法是加上base或this的前缀来调用，还需要用更精确的例子来确认原因

### 泛型
  - 热更工程如果使用了没有在主工程中直接使用过的泛型类或泛型方法，在IL2CPP平台会在运行时出现问题
    - 比如：`TemplateClass<T>`和MainClass都是主工程的类，主工程没有用过`TemplateClass<MainClass>`，在热更工程里写`TemplateClass<TMainClass>`运行时就会报错。真想这么用的话可以在主工程中写一个方法，在里面定义几个热更要用的泛型实例的局部变量。
    - 泛型方法也一样
    - 使用主工程的泛型类，T使用热更中的类型也是不行的
  - 热更工程定义泛型基类继承主工程是不行的：`HotfixSub<T> : MainBase<T>`
  - 热更工程定义普通类继承主工程泛型基类是可以的：`HotfixSub ： MainBase<GGUIMonoCommonTab>`
  - 参考[官网泛型](https://ourpalm.github.io/ILRuntime/public/v1/guide/il2cpp.html)的说明

### 委托
  - 如果主工程要要用到热更中定义的委托，比如List.Find、List.Sort，参数如果是委托，需要增加[委托适配器](https://ourpalm.github.io/ILRuntime/public/v1/guide/delegate.html)，这个会导致代码无法热更，如果业务一定需要热更，就要换一些写法

### 死循环
  - 打印协议log，因为ILR里的协议结构体本身会循环引用，没处理好就会导致死循环
  - 在单例的构造函数里，调用了ILR里的函数，ILR里又调了这个单例，因为单例还没有赋值，就会导致死循环（不过这个问题和ILRuntime无关= =）
  - 查找在哪里死循环的方式：开调试，触发死循环，按中断，看堆栈

### 禁止的用法
  - 主工程不能使用热更dll中的类型，主工程中的泛型类不能使用热更中的类型当做类型参数（很重要）
      - 解决方法是代码拷贝一份，namespace改成不一样的
  - 不能继承MonoBehaviour
  - enum.Parse不能用
      - EXXXFactory，手写switch case判断（之后可以增加自动生成）
  - volatile关键字不支持（单例不能用这个）
      - `System.Collections.Generic.KeyNotFoundException: Cannot find Type:MG.Hotfix.Model modreq(System.Runtime.CompilerServices.IsVolatile)`
  - 热更类不要同时继承多个主工程接口
      - 先创建新接口，再继承
  - 热更类不能同时继承主工程的类和接口
  - 热更类继承了主工程的类或接口时，不要再继承热更工程的接口
  - Dictionary的TryGetValue，如果out是struct或者值类型变量，可能会有错误
  - List需要Sort的话，Sort的函数参数都用object，不然有可能需要写RegisterFunctionDelegate，但是如果要热更，就不能加这些

### 其他
  - 调用了某个方法，但是结果错误，基本都是Adapter没写对
  - 工程不会自动导入从外部新增加的文件（比如协议文件）
    - 手动修改csproj文件，改为通配符模式，并忽略几个文件夹，如：

    ```
    <ItemGroup>
        <Compile Include="**\*.cs" />
        <Compile Remove="bin\**" />
    </ItemGroup>
    ``` 

    - 如果使用vs开发
      - 某些操作会让vs强行重新生成.csproj，目前发现的有
          - 删除文件
      - 外部新增文件时，VS不会自动刷新，可以用alt+f+j+1来重新打开解决方案
      - 所以如果被vs修改了.csproj，记得在git上重置掉，不要提交！
      - [gitlab有锁定文件的功能！](https://docs.gitlab.com/ee/user/project/file_lock.html)但是。。19刀/人月。。
  - 热更工程的枚举，在解释器里，是使用int来替代的，equals这种，就调用不到enum的equals实现，可以干脆直接用类里的const替代
  - IOS（或Android使用IL2CPP）需要使用link.xml来阻值代码裁剪
