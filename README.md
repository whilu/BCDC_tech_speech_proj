# Beat Blade 项目中关于 IoC 与依赖注入(DI)的实践

### IoC

#### 什么是 IoC

控制反转（Inversion of Control，缩写为 IoC），是面向对象编程中的一种设计原则，用来减低代码之间的耦合度。 - 维基百科

控制反转到底是什么被反转了？这里指的是依赖对象的获取。在传统面向对象编程中，依赖对象往往是主动 new 出来；但是在 IoC 中，依赖对象转变为由 IoC 容器提供。

#### IoC 能做到什么

IoC 是一种设计原则，帮助我们降低代码之间的耦合度。刚刚说到，传统面向对象编程中在类内部主动产生(new)所依赖的对象，导致类与类之间耦合严重。IoC 的应用，依赖对象的查找与产生统一交由 IoC 容器控制，对于开发者来说代码结构更加清晰。

#### Beat Blade 项目中为什么使用 IoC

主要有以下几个原因:

- 松散耦合；

- Beat Blade 项目需求源源不断产生，以及拥有不同版本(海外版、国内版、版号过审版和 Switch 版等）不同功能需求，IoC 保证了程序有一套灵活的体系结构更容易扩展；

- 项目参与的开发同学断断续续有增加，便于新同学能够在一套指导思想下更好的复用 & 扩展现有功能；

- 开发时间一直在延长，更好的保证代码质量；

- 方便开发同学自己测试。

#### 实现控制反转(IoC)的方式

- 依赖查找，主动获取依赖对象的方式。在需要依赖对象时向 IoC 容器提供相关信息，从容器中取得对象。

- 依赖注入，被动注入依赖对象方式。在某个类依赖某些对象时，在这个类构造时 IoC 容器根据一些相关信息注入相应的依赖对象到类相应的属性（成员变量）中。

下面就介绍一下 Beat Blade 项目中关于「依赖注入」的一点技术演进。

### 依赖注入(DI)

#### 依赖注入概念

依赖注入（Dependency injection）的意思为，给予调用方它所需要的事物。在这里我们讨论的是，为我们的属性或成员变量给予它所需要的实例对象。

#### 依赖注入与 IoC 关系

### Beat Blade 项目中实现依赖注入(DI)的几种方式

#### 反射生成依赖

##### C# 中的反射

##### 实现

##### 在 iOS 的 Full AOT 以及现使用的 IL2CPP 模式反射生成依赖会存在问题吗？

##### 运行时通过反射动态生成依赖带来的性能问题

#### IL 编织

##### IL(CIL) 概念

##### 编写 IL 代码

##### Unity 中 IL 代码编织并插入目标程序的工作流

##### IL 编织后的 DLL 与 IL2CPP 兼容吗

##### IL 编织问题

IL 编译后期 IL2CPP 前期生成代码，性能比反射好。那么会存在什么问题吗？

#### 对注入类自动生成 C# 代码

##### 抛开 IL 编织，探寻可读性更高的代码生成

##### 实现



























- [https://blogs.unity3d.com/2015/05/06/an-introduction-to-ilcpp-internals/](Unity tech for IL2CPP)
- [https://blog.csdn.net/xtlisk/article/details/39099199](AOT和JIT)
- [https://www.cnblogs.com/humble/archive/2012/11/25/2787738.html](什么是Emit,什么是反射,二者区别到底是什么?)
- [https://blog.csdn.net/it_man/article/details/8660536](什么是元数据 (MetaData))
- [https://www.cnblogs.com/timeObjserver/p/9615711.html](U3D开发中关于脚本方面的限制-有关IOS反射和JIT的支持问题)
- [https://blog.csdn.net/hany3000/article/details/44033249](Unity在iOS平台下的Mono在Full AOT模式下的限制)
- [https://www.zhihu.com/question/52572852/answer/131161685](如果没有PGO，JIT 编译相比AOT 编译有哪些优势？)
- [https://zhuanlan.zhihu.com/p/48236683](使用 Profile Guided Optimization 提升 Application 的性能)
- [https://www.zhihu.com/question/23414153/answer/48197885?utm_campaign=shareopn&utm_content=group3_Answer&utm_medium=social&utm_oi=33611554226176&utm_source=wechat_session&s_r=0](Unity 精通要掌握)
- [https://blog.csdn.net/lenfien/article/details/51175753](C++虚函数的调用过程)
- [https://blog.csdn.net/qq_43390943/article/details/99063262](虚函数的底层调用过程)
- [https://www.cnblogs.com/malecrab/p/5572730.html](C/C++杂记：虚函数的实现的基本原理)
