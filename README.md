- [https://blogs.unity3d.com/2015/05/06/an-introduction-to-ilcpp-internals/](Unity tech for IL2CPP)
- [https://blog.csdn.net/xtlisk/article/details/39099199](AOT和JIT)
- [https://www.cnblogs.com/humble/archive/2012/11/25/2787738.html](什么是Emit,什么是反射,二者区别到底是什么?)
- [https://blog.csdn.net/it_man/article/details/8660536](什么是元数据 (MetaData))
- [https://www.cnblogs.com/timeObjserver/p/9615711.html](U3D开发中关于脚本方面的限制-有关IOS反射和JIT的支持问题
)
- [https://blog.csdn.net/hany3000/article/details/44033249](Unity在iOS平台下的Mono在Full AOT模式下的限制)
- [https://www.zhihu.com/question/52572852/answer/131161685](如果没有PGO，JIT 编译相比AOT 编译有哪些优势？)
- [https://zhuanlan.zhihu.com/p/48236683](使用 Profile Guided Optimization 提升 Application 的性能)
- [https://www.zhihu.com/question/23414153/answer/48197885?utm_campaign=shareopn&utm_content=group3_Answer&utm_medium=social&utm_oi=33611554226176&utm_source=wechat_session&s_r=0](Unity 精通要掌握)
- [https://blog.csdn.net/lenfien/article/details/51175753](C++虚函数的调用过程)
- [https://blog.csdn.net/qq_43390943/article/details/99063262](虚函数的底层调用过程)
- [https://www.cnblogs.com/malecrab/p/5572730.html](C/C++杂记：虚函数的实现的基本原理)
- []()

选题
Beat Blade 项目中关于 IoC 与依赖注入(DI)的实践

为什么使用 IoC？

Beat Blade 中现存的 DI 方式
运行时注入 - 反射(Strange IoC 提供)
静态注入 - IL Emit(Unity GUI 组件绑定至 IoC 框架)
静态注入 - 生成辅助类(正在尝试的一种方式，编译期生成辅助类注入)

性能对比:

Full AOT 模式下，能否与 IL2CPP 兼容?

