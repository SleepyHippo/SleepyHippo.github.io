---
layout: post
title:  "ILRuntime爬坑笔记1——背景知识篇"
date:   2021-03-28 13:01:45 +0800
categories: Unity ILRuntime
---

## **前言**
这是2019年在给项目做热更框架支持时做的调研笔记，现在重新整理分享出来

ILRuntime是一种C#代码热更新的方案，在学习使用的过程中，不可避免地会碰到很多疑问，比如：直接加载执行Dll，和用ILRuntime来解释Dll，有什么区别？为什么IOS不能直接加载执行Dll？C#编译出来的字节码是怎么被执行的？在寻找答案的过程中，笔者发现需要了解一些背景知识，才能搞懂这些问题，所以先用这篇文章介绍一下相关的背景知识。

## **代码是如何被执行的？**
所有代码都是这样被执行的：从源代码开始，经过一系列转换（后面展开），最终生成机器码（有时候也叫原生码，或本地码），由CPU执行。

### 编译
我们把代码的这种转换叫做编译，编译就是把更高层的A语言代码转换为更底层的B语言代码的一种行为。编译产出的其实是另一种语言，并不涉及代码的执行。编译器（Compiler）就是可以执行编译行为的程序。

可以把程序的生命周期分为CompileTime和RunTime。一个程序开始执行，就进入RunTime周期。根据编译的时机是在CompileTime还是RunTime，我们可以分为两种：
AOT（Ahead-Of-Time）编译器和JIT（Just-In-Time）编译器，其中的Time，就是指RunTime。

### 解释
还有一个动词经常和编译一起出现，就是“解释”。解释器（Interpreter）执行代码的过程就叫解释，解释器的作用是，在运行时读取代码并执行。解释器最终会执行代码，但并不规定如何执行代码，可能其中就夹杂着编译的过程（例如JIT）。

### 对比
虽然从广义来看，解释的概念其实是包括了编译，不过为了方便，后面说的解释都是狭义上的，也就是指执行代码的过程中不包含编译的步骤。

编译的好处，是加快执行代码的速度，简化执行代码的难度。举个栗子，如果不用编译，用纯解释的方式执行代码，就好比只使用最基本的数学/物理公理来解高考题，用到的每条定理都需要从公理推导出来，如果有10道题用了同一个定理，就需要重复推导10次这个定理。执行速度是会很慢，并且实现难度也高。当然解释执行也有好处，就是对开发人员更友好，有更好的跨平台可移植性。

编译器：
- 本质是输入代码，输出另一种代码
    - 一般会分为多层，经过多次转换，最终生成机器码，给CPU执行
    - JIT、AOT、Full AOT等，是不同的编译策略
    
解释器：
- 本质是输入代码，输出结果
    - 在程序运行时执行
    - 是一个黑箱，内部实现方式不确定，经常也可以用编译的方式来实现
    - 不是不需要编译，只是不需要用户显式地使用编译器来得到可执行代码
    
### 从机器码生成时机的角度来看

最终的机器码可以在CompileTime就生成，或者到RunTime才生成。
  
如果在CompileTime就生成机器码，这种编译方式叫Full AOT，C、C++的常规运行方式就是这种类型。
  
如果在RunTime才生成机器码，可以完全使用解释的方式来执行；也可以使用JIT的方式执行，也就是运行时编译后再执行机器码。Javascript，Python，Lua等脚本语言，比较经常会用这两种执行方式。
  
如果在RunTime生成机器码，RunTime的输入可以是代码，也可以是代码编译过的中间代码，C#和Java就是典型的生成中间代码的例子。这是为了在性能和可移植性中取得一个平衡。在RunTime生成的东西越接近机器码，性能越好，可移植性越差，反之，在RunTime之前完全不生成机器码就等于是纯解释的方式来执行代码，性能最差，但可移植性就越好。（其实可移植性是一个工作量的问题）

### 参考
- [JVM-字节码解释执行引擎](http://reimuwang.org/2017/12/11/JVM-%E5%AD%97%E8%8A%82%E7%A0%81%E8%A7%A3%E9%87%8A%E6%89%A7%E8%A1%8C%E5%BC%95%E6%93%8E/)
- [你知道「编译」与「解释」的区别吗？](http://huang-jerryc.com/2016/11/20/do-you-konw-the-different-between-compiler-and-interpreter/)
- [Understanding the differences: traditional interpreter, JIT compiler, JIT interpreter and AOT compiler](https://softwareengineering.stackexchange.com/questions/246094/understanding-the-differences-traditional-interpreter-jit-compiler-jit-interp)
- https://rednaxelafx.iteye.com/blog/492667


## **Unity代码如何执行的？**
Unity默认的C#代码，是编译执行的，以前，只有Mono一种编译流程，从Unity4.6.1p5开始，引入了一种新的编译流程IL2CPP。

### **Mono流程：**
![mono编译流程]({{ site.url }}/assets/images/mono编译流程.png)


一句话来说就是：C# 编译成 CIL（dll）字节码，由Mono VM运行
具体一点，前半部分：
```
     MonoC#编译器         ILASM编译器                    MonoVM
C# ---------------> CIL ---------------> CIL字节码  --------------> 结果
```
后半部分，Mono VM执行字节码有3种方式：

| | 运行前 | 运行时 |
| ------------------------- | ---------------------------------------- | -------------------------------------------  |
| JIT（Android、Win、WiiU） | 无处理 | 逐条读入，逐条解析翻译成机器码交给CPU再执行 |
| AOT（Unity没有用这种方式） | 把大部分的CIL，编译成机器码，剩下的走JIT | 有编译的交给CPU直接执行，没有的走JIT |
| Full AOT（IOS平台） | 把所有的CIL编译成机器码 | 直接执行 |


### **IL2CPP流程：**

![IL2CPP编译流程]({{ site.url }}/assets/images/IL2CPP编译流程.png)

一句话来说就是：C# 编译成 CIL，裁剪后，再由il2cpp.exe编译成C++代码，再交给对应平台的编译器，编译成对应平台的机器码，链接上libil2cpp库（IL2CPP VM），然后运行

具体一点：
```
                                                                                                         
    MonoC#编译器           ILASM编译器                裁剪                     il2cpp.exe(AOT)           不同平台的C++编译器           链接libil2cpp库（IL2CPP VM）
C# ---------------> CIL ---------------> CIL字节码 --------> 裁剪后的CIL字节码 ---------------> C++代码 ---------------------> 机器码 ----------------------------> 结果
```

### 参考
- [Unity ScriptingRestrictions](https://docs.unity3d.com/Manual/ScriptingRestrictions.html)
- [Unity将来时：IL2CPP是什么？](https://zhuanlan.zhihu.com/p/19972689)
- [Mono的aot](https://www.mono-project.com/docs/advanced/aot/)
- [C#是如何执行的](https://cloud.tencent.com/developer/article/1134387)
- [c# .net mono 版本](https://www.cnblogs.com/zhaoqingqing/p/5762867.html)
- [Mono为何能跨平台？聊聊CIL(MSIL)](https://www.cnblogs.com/murongxiaopifu/p/4211964.html)
- [谁偷了我的热更新？Mono，JIT，iOS](https://www.cnblogs.com/murongxiaopifu/p/4278947.html)

## **热更新本质上是什么？**

热更本质上是，在运行时，执行新的逻辑。要达到这个目的，实际上有两种方式：

- 执行新的机器码
  - C#的托管Dll：下载dll（字节码），运行时加载，使用Mono的JIT方式编译执行
  - 本地（Native）Dll：下载dll（机器码），加载到内存里，执行
- 按照不同的顺序，执行原有的代码
  - 在Excel、Prefab、ScriptableObject等不同载体上进行配置，更新配置就可以调用不同的代码逻辑分支
  - 在语言里，内嵌一个解释器，下载下来的脚本语言，实际上只是按照不同的顺序，调用已经发布出去的代码，
    - LUA
    - ILRuntime
    - ...
