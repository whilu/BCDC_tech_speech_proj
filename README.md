# Beat Blade 项目中关于 IoC 与依赖注入(DI)的实践

## IoC

### 什么是 IoC

> 控制反转（Inversion of Control，缩写为 IoC），是面向对象编程中的一种设计原则，用来减低代码之间的耦合度。 - 维基百科

控制反转到底是什么被反转了？这里指的是依赖对象的获取。在传统面向对象编程中，依赖对象往往是主动 `new` 出来；但是在 IoC 中，依赖对象转变为由 IoC 容器提供。

### IoC 能做到什么

IoC 是一种设计原则，帮助我们降低代码之间的耦合度。刚刚说到，传统面向对象编程中在类内部主动产生(`new`)所依赖的对象，导致类与类之间耦合严重。IoC 的应用，依赖对象的查找与产生统一交由 IoC 容器控制，对于开发者来说代码结构更加清晰。

### Beat Blade 项目中为什么使用 IoC

主要有以下几个原因:

- 松散耦合；

- Beat Blade 项目需求源源不断产生，以及拥有不同版本(海外版、国内版、版号过审版和 Switch 版等）不同功能需求，IoC 保证了程序有一套灵活的体系结构更容易扩展；

- 项目参与的开发同学断断续续有增加，便于新同学能够在一套指导思想下更好的复用 & 扩展现有功能；

- 开发时间一直在延长，更好的保证代码质量；

- 方便开发同学自己测试。

### 实现控制反转(IoC)的方式

- 依赖查找，主动获取依赖对象的方式。在需要依赖对象时向 IoC 容器提供相关信息，从容器中取得对象。

- 依赖注入，被动注入依赖对象方式。在某个类依赖某些对象时，在这个类构造时 IoC 容器根据一些相关信息注入相应的依赖对象到类相应的属性（成员变量）中。

下面就介绍一下 Beat Blade 项目中关于「依赖注入」的一些技术演进。

## 依赖注入(DI)

### 依赖注入概念

依赖注入（Dependency injection）的意思为，给予调用方它所需要的事物。在这里我们讨论的是，为类中属性或成员变量给予它所需要的实例对象。

### Beat Blade 项目中实现依赖注入(DI)的几种方式

### C# 特性(Attribute)

在讲 C# 中的依赖注入实现方式之前，必须要提及的是「特性」。一个 C# 特性的定义如下:

```c#
[AttributeUsage(AttributeTargets.Property, AllowMultiple = false, Inherited = true)]
public class InjectAttribute : Attribute
{
    public InjectAttribute() { }
}
```

使用特性，可以有效地将元数据或声明性信息与代码（程序集、类型、方法、属性等）相关联。将特性与程序实体相关联后，可以在运行时使用反射这项技术查询特性。在依赖注入中，通常会使用自定义特性标记需要注入的属性或成员变量等，以此告知 IoC 容器这里需要提供依赖对象。

下面，我们就借由特性来看看几种依赖注入。

### 运行时反射实现依赖注入

假设现在有一个 `Cat` 类，在我们的 `MonoBehaviour` 中依赖使用这个类，下面就使用反射和特性实现简单的依赖注入。

```c#
public class IoCMonoBehaviour : MonoBehaviour
{
    [InjectAttribute]
    public Cat Cat { get; set; }
    
    private void Awake()
    {
        // Do inject
        Injector.Inject(this);
    }

    private void Start()
    {
        // Print cat's name
        Debug.Log(Cat.name);
    }
}
```

核心注入方法:

```c#
public class Injector
{
    public static void Inject(object target)
    {
        Type type = target.GetType();
        
        MemberInfo[] members = type.FindMembers(
            MemberTypes.Property, 
            BindingFlags.SetProperty | BindingFlags.Public | BindingFlags.Instance, 
            null, null);
        
        foreach (MemberInfo member in members)
        {
            object[] injections = member.GetCustomAttributes(typeof(InjectAttribute), true);
            if (injections.Length > 0 && member is PropertyInfo)
            {
                PropertyInfo propertyInfo = member as PropertyInfo;
                propertyInfo.SetValue(target, Activator.CreateInstance(propertyInfo.PropertyType));
            }
        }
    }
}
```

可以看出，上面的代码中我们没有使用 `new` 关键字就能够使用 `Cat` 对象。所以通过「反射 + 特性」实现运行时依赖注入并不难，但是当一个类存在大量依赖需要注入时，反射查找以及动态创建对象带来的性能损耗就成了一个值得很关注的问题，特别是在对性能极其敏感移动设备上。

反射必定会产生性能开销，那么如何尽可能减小了，在 Beat Blade 项目中做了如下几点优化:

1. 依赖注入过程均摊到场景加载的 Loading 中；

2. 缓存依赖对象；

3. 单例的使用。

##### iOS 的 Full AOT 反射实现注入依赖存在问题吗？

我们知道，iOS 系统要求所有代码必须全部在运行前全部编译（即 Full AOT）。那么上面利用反射实现的依赖注入会有问题吗？答案是只要不使用 `System.Reflection.Emit` 下的 API，Full AOT 下都不会有任何问题，`System.Reflection.Emit` 下的 API 动态产生代码是不被允许的。

### CIL 编织

#### CIL(IL) 概念

> 通用中间语言（Common Intermediate Language）是一种属于通用语言架构和 .NET 框架的低阶（lowest-level）的人类可读的编程语言。目标为 .NET 框架的语言被编译成 CIL，然后汇编成字节码。CIL类似一个面向对象的组合语言，并且它是完全基于堆栈的。它运行在虚拟机上，其主要的语言有 C#、Visual Basic .NET(VB.NET)、C++/CLI 以及 J#。 - 维基百科

#### Unity IL2CPP 构建流

在说 CIL 代码编织前先来看看 IL2CPP 构建流，如图:

<center>
<img src="https://raw.githubusercontent.com/whilu/lujun.co-storge/master/image/il2cpp-toolchain-smaller.png" width="70%" height="70%" />
</center>

在 Unity IL2CPP 构建流中，C# 代码首先被编译为 CIL code，随后 CIL code 经由 IL2CPP 被转换成 CPP，最终再经由目标平台的 C++ 编译器将 CPP 代码编译成目标机器码以执行。

前面说过反射实现依赖注入会带来一些性能开销，对于无法忍受反射的开发者来说，CIL 编织无疑既可以保证代码的质量，同样也可以实现依赖注入；只要我们在 IL2CPP 转换这一步进行之前对构建流进行拦截，同时对 CIL code 进行一番修改，即可实现编译前的依赖注入。

#### 使用一些工具编写 CIL 代码

1. 使用 `System.Reflection.Emit` 命名空间下的相关 API 实现 CIL 指令编写。虽然在 Full AOT 模式下不被允许使用，但是在 Full AOT 之前借助 Emit 生成 CIL 代码是没问题的；

2. 使用开源项目 Mono.Cecil 编写 CIL 代码。

下面就以 Mono.Cecil 使用为例，说说使用 CIL 编织技术实现依赖注入的方案。

#### CIL 实现依赖注入

还是以前面的 `Cat` 注入为例，首先看看未注入之前我们的 `IoCMonoBehaviour` 类中 `Awake` 方法的 CIL 代码:

```cpp
.method private hidebysig 
	instance void Awake () cil managed 
{
    // Method begins at RVA 0x2d76
    // Code size 2 (0x2)
    .maxstack 8

    IL_0000: nop
    IL_0001: ret
} // end of method IoCMonoBehaviour::Awake
```

然后我们使用 Mono.Cecil 编织 CIL 代码，以辅助注入 `Cat` 对象，代码如下:

```c#
private void ILInject()
{
    // 获取相应的 definition
    AssemblyDefinition assemblyDefinition = AssemblyDefinition.ReadAssembly("assemblyPath", null);
    ModuleDefinition moduleDefinition = assemblyDefinition.MainModule;
    MethodDefinition methodDefinition = moduleDefinition.Types[0].Methods[0];
    ILProcessor ilProcessor = methodDefinition.Body.GetILProcessor();
        
    // 生成 CIL 指令语句
    Instruction i1 = ilProcessor.Create(OpCodes.Ldarg, 0);
    Instruction i2 = ilProcessor.Create(OpCodes.Newobj,
        moduleDefinition.ImportReference(typeof(Cat).GetConstructor(new Type[] { })));
    Instruction i3 = ilProcessor.Create(OpCodes.Call,
        moduleDefinition.ImportReference(typeof(IoCMonoBehaviour).GetMethod("set_Cat", new Type[0])));
    
    // 插入生成的 CIL 指令
    ilProcessor.Append(i1);
    ilProcessor.Append(i2);
    ilProcessor.Append(i3);
}
```

上面的编织过程，主要进行了对 `IoCMonoBehaviour` 类中 `Cat` 属性的赋值操作。具体如下:

- 首先根据 DLL 路径加载 `AssemblyDefinition`；

- 然后根据 `AssemblyDefinition` 依次获取到 `ModuleDefinition`（这里指命名空间 `com.battlecryhq.beatrunner`）、`TypeDefinition`（具体的 `IoCMonoBehaviour` 类）以及 `MethodDefinition`（类 `IoCMonoBehaviour` 中的 `Awake` 方法）；

- 获取到 `Awake` 方法的 ILProcessor 之后，就能够对该方法中的 CIL 代码进行操作；

- `ilProcessor.Create(OpCodes.Ldarg, 0)` 会生成 `ldarg.0` 这条语句，表示取当前方法的第一个参数（即当前类的引用 this）；当 CLR 指令执行到此处时会将 `this` 压入方法数据栈;

- `ilProcessor.Create(OpCodes.Newobj, moduleDefinition.ImportReference(typeof(Cat).GetConstructor(new Type[] { })))` 这行代码会产生 `newobj instance void com.battlecryhq.beatrunner.Cat::.ctor()` 语句；当 CLR 执行这条指令后会创建一个新的 `Cat` 对象到堆中，并将其引用压进方法的数据栈；

- `ilProcessor.Create(OpCodes.Call, moduleDefinition.ImportReference(typeof(IoCMonoBehaviour).GetMethod("set_Cat", new Type[0])))` 这行代码会产生 `call instance void com.battlecryhq.beatrunner.IoCMonoBehaviour::set_Cat(class com.battlecryhq.beatrunner.Cat)` 这条语句；CLR 执行这条指令会先从方法的数据栈中取出前面压入的两个数据 `this` 和 `Cat` 对象的引用，用于 `set_Cat` 方法的 invoke。

至此，`Cat` 属性就被成功的设置了相应的对象实例。再看看注入之后的 `Awake` 方法完整的 CIL 代码:

```cpp
.method private hidebysig 
    instance void Awake () cil managed 
{
    // Method begins at RVA 0x4bf1a
    // Code size 14 (0xe)
    .maxstack 8

    IL_0000: nop
    IL_0001: ldarg.0
    IL_0002: newobj instance void com.battlecryhq.beatrunner.Cat::.ctor()
    IL_0007: call instance void com.battlecryhq.beatrunner.IoCMonoBehaviour::set_Cat(class com.battlecryhq.beatrunner.Cat)
    IL_000c: nop
    IL_000d: ret
} // end of method IoCMonoBehaviour::Awake
```

CIL 编织既能做到非侵入式的依赖注入，也能解决反射依赖注入带来的性能问题。由于在 IL2CPP 转换之前进行拦截，所以与 IL2CPP 构建相兼容。那么这种方式会存在什么问题吗？

#### CIL 编织存在的问题

虽然解决了使用反射实现依赖注入带来的性能问题，但往往从没有十全十美的解决方案。CIL 编织实现依赖注入也不例外，目前 Beat Blade 项目中使用遇到的问题主要有以下几点:

- CIL 指令繁多，学习成本较高；

- C# 中最常使用的泛型，在 CIL 编织过程中变得较为复杂（在 CIL 编织阶段要确定所有泛型的具体类型，这一块的处理需要特别谨慎）；

- 仅仅生成了 CIL 代码而没有源 C# 代码与之对应，所以调试比较困难；

- 基于 Unity 构建并上传至 Unity 崩溃分析服务的符号表部分可能无法匹配，若这部分代码出现异常可能会在后台看不到相应的出错堆栈信息。

基于这些问题，目前 Beat Blade 项目中只有关于 Unity UI 的动态查找与脚本的绑定部分使用了 CIL 编织实现注入。

难道就真的没有更好的解决方案了吗？

### 自动生成辅助（容器）类实现依赖注入

要想解决上述 CIL 实现依赖注入产生的问题，并保留其编译前处理依赖注入的能力，容器辅助类代码自动生成技术应运而生。简单来说，就是应用一些设计模式，生成固定的辅助（容器）类，由自动生成的辅助类产生依赖的对象，从而达到依赖注入的目的。

这种方式拦截 Unity 构建流中 CIL 编译这一步。在 C# 代码被编译成 CIL 之前，通过寻找被一组预定「特性」标记的属性以及相关类(接口)，由「特性」携带的参数得到依赖类与被依赖类之间的关系，从而生成相关辅助类，再由辅助（容器）类提供依赖并实现注入。

### 自动生成辅助（容器）类实现

说起来很绕，还是来看看大致的实现方案。

#### 自定义「特性」

- `InjectAttribute` 用来描述需要被注入的域；

```c#
[AttributeUsage(AttributeTargets.Field, AllowMultiple = false, Inherited = false)]
public class InjectAttribute : Attribute { }
```

- `ProvidesAttribute`，这个特性用来描述产生依赖对象的方法；

```c#
[AttributeUsage(AttributeTargets.Method, AllowMultiple = false, Inherited = false)]
public class ProvidesAttribute : Attribute { }
```

- `ModuleAttribute` 用来描述拥有产生一个或多个依赖对象方法的类；

```c#
[AttributeUsage(AttributeTargets.Class, AllowMultiple = false, Inherited = false)]
public class ModuleAttribute : Attribute { }
```

- `ComponentAttribute` 用来关联依赖产生方与依赖注入组件需求方（定义依赖于被依赖方的关系），从而依赖注入组件知道从何处获得依赖对象。

```c#
[AttributeUsage(AttributeTargets.Interface, AllowMultiple = false, Inherited = false)]
public class ComponentAttribute : Attribute
{
    public Type Type { get; set; }

    public ComponentAttribute(Type type)
    {
        Type = type;
    }
}
```

#### Module 以及 Component

- Module 类定义拥有产生一个或多个依赖对象的方法的类；

```c#
[Module]
public class GameMonoBehaviourModule
{
    [Provides]
    public Cow ProvideCow()
    {
        return new Cow("Cow");
    }
    
    [Provides]
    public Duck ProvideDuck()
    {
        return new Duck("Duck");
    }
}
```

- Component 接口连接依赖产生方与依赖注入组件需求方。

```c#
[Component(typeof(GameMonoBehaviourModule))]
public interface IGameMonoBehaviourComponent
{
    void Inject(GameMonoBehaviour gameMonoBehaviour);
}
```

#### 使用 CodeDOM 生成源代码

CodeDOM 是什么？它提供表示多种常见源代码元素的类型。可以设计一个程序，它使用 CodeDOM 元素生成源代码模型来组合对象图。简单来说，使用 CodeDOM 提供的 API 在编译前期生成我们的辅助依赖注入的代码。

对于 CodeDOM 编织 C# 代码不做过多介绍，在这里主要理解使用这种技术的原理。下面看看使用 CodeDOM 后生成的依赖注入辅助类:

<center>
<img src="https://raw.githubusercontent.com/whilu/lujun.co-storge/master/image/bcdc_tech_speech_code_generate_flow.png" width="70%" height="70%" />
</center>

- `GameMonoBehaviourModule_ProvideFactory` 类主要负责连接依赖提供的 Module 类用于产生依赖对象；

- `GameMonoBehaviour_MemberInjector` 类主要用来对依赖需求方实现依赖注入；

- `GameMonoBehaviourIGameMonoBehaviourComponent` 主要用来连接上述两个类以及为开发者提供注入初始化代码入口。

最后在我们的依赖需求使用类中，使用生成的 `GameMonoBehaviourIGameMonoBehaviourComponent` 类实现依赖注入:

```c#
public class GameMonoBehaviour : MonoBehaviour
{
    [InjectAttribute] 
    public Cow _cow;
    
    [InjectAttribute] 
    public Duck _duck;
    
    private void Awake()
    {
        GameMonoBehaviourIGameMonoBehaviourComponent.Builder().Build().Inject(this);
        
        _cow.Speak();
        _duck.Speak();
    }
}
```

#### 总结一下辅助（容器）类实现依赖注入

由于操作的是 C# 源码，所以前面提到的 CIL 会产生的问题都得到了解决；同样编译期就确定了依赖注入如何提供，不需要运行时动态查找和生成，可解决反射注入带来的性能问题。

那么这种方式会有存在问题吗？

- 首先，对于使用者需要适应一下这种模式，由于不再是只有一个「特性」标记，所以需要时刻清楚依赖与被依赖的关系；

- 需要定义多几个接口（类）来确定依赖与被依赖的关系；

上面的问题都是值得靠开发者解决的，这样可以让代码更加灵活可扩展。

目前这种依赖注入方式在 Beat Blade 项目中小范围使用中，未来会进一步完善这一框架（例如单例注入、Lazy injections、Provider injections 等），然后慢慢替换现在项目中存在的可能产生问题的注入。

## 总结

在软件开发过程中，各种设计模式起着不可或缺的作用。作为一种设计原则，控制反转（IoC）为我们提供了一种良好的代码松散耦合的规范；项目代码结构更加清晰且便于扩展，能够更加充分保证了代码质量。

Beat Blade 项目从 IoC 最初由反射实现，演进为编译前（中）期 CIL 自动编织 + 辅助类自动生成 + 反射三种技术实现 IoC，既享受了 IoC 带来的便利性，也保证了程序上的稳定性。














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
