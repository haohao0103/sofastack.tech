
---
title: "类委托加载"
aliases: "/sofa-boot/docs/sofa-ark-class-loader-delegation"
siblings_only: "true"
---
## 类委托加载

注：这里委托加载不止包含类委托，还包含资源委托加载。

SOFAArk 通过多 ClassLoader 的方式提供类隔离的能力，并在此基础上具备多版本隔离和热部署的能力。当然类如果纯粹隔离还不够，还需要考虑ClassLoader之间的类共享。SOFAArk 通过类委托机制，提供了一套丰富的委托加载能力。定义了PluginClassLoader 和 BizClassLoader，不同类型具备不同的类查找逻辑，同时提供ClassLoaderHook能力，允许自定义模块间的类委托加载。SOFAArk 框架本身的类委托加载是非常灵活的，但是在实际落地时，需要有一套类委托加载的最佳实践。

SOFAArk 支持的场景主要分为两类：
1. 多Plugin的类隔离：考虑如何相同类不同版本不会相互冲突
2. 合并部署/热部署：原应用如何拆分，或多个应用如何合并

这里我们主要考虑第2种，多 ClassLoader 如何布局，在内部探索出的最佳实践多 ClassLoader 布局为
 ![image](https://user-images.githubusercontent.com/3754074/169092647-ba4047b5-cce5-4151-a696-e05ae62c2e81.png)

模块里的类和资源查找优先从模块里查找列，查找不到从再基座里查找。

最佳实践
类委托加载的准则是中间件相关的依赖需要放在同一个的 ClassLoader 里进行加载执行，达到这种方式的最佳实践有两种：

### 强制委托加载

由于中间件相关的依赖一般需要在同一个ClassLoader里加载运行，所以可以制定一个中间件依赖的白名单，强制这些依赖委托给基座加载。SOFAArk 已经提供白名单能力，详细查看与 `sofa.ark.plugin.export.class.enable` 有关源码。

#### 强制加载优缺点

- 优点

模块开发者不需要感知哪些依赖属于需要强制加载由同一个 ClassLoader 加载的依赖

- 缺点

白名单里要强制加载的依赖列表需要维护，列表的缺失需要更新基座，较为重要的升级需要推所有的基座升级。

### 自定义委托加载

模块里pom通过设置依赖的 scope 为 provided主动指定哪些要委托给基座加载。通过模块瘦身工具自动把与基座重复的依赖委托给基座加载，基座通过预置中间件的依赖（可选，虽然模块暂时不会用到，但可以提前引入，以备后续模块需要引入的时候不需再发布基座即可引入）

#### 1. 基座预置依赖

基座可以提前预置一些依赖

#### 2. 通过两种方式委托给基座加载

- 模块pom里依赖 scope 设置为 provided
- biz 打包插件sofa-ark-maven-plugin里设置 excludeGroupIds 或 excludeArtifactIds

#### 3. 只有模块声明过的依赖才可以委托给基座加载

模块启动的时候，框架会有一些扫描逻辑，这些扫描如果不做限制会查找到模块和基座的所有类、资源，导致一些模块明明不需要的功能尝试去初始化，从而报错。SOFAArk 2.0.3之后新增了模块的 declaredMode, 来限制只有模块里声明过的依赖才可以委托给基座加载。只需在模块的 sofa-ark-maven-plugin 打包插件的 Configurations 里增加  <declaredMode>true</declaredMode>即可。

#### 自定义委托加载优缺点

- 优点

不需要维护强制加载列表，当部分需要由同一 ClassLoader 加载的依赖没有设置为统一加载时，可以修改模块就可以修复，不需要发布基座（除非基座确实依赖）。

- 缺点

对模块自动瘦身的依赖较强

### 对比与总结

|委托加载类型|依赖缺失排查成本|修复成本|模块改造成本|维护成本|
|----|----|----|----|----|
|强制加载|类转换失败或类查找失败，成本中|更新 plugin，发布基座，高|低|高|
|自定义委托加载 |类转换失败或类查找失败，成本中|更新模块依赖，如果基座依赖不足，需要更新基座并发布，中|高|低|
|自定义委托加载 + 基座预置依赖 + 模块自动瘦身|类转换失败或类查找失败，成本中|更新模块依赖，设置为provided，低|低|低|

### 推荐自定义委托加载方式

1. 模块自定义委托加载 + 模块自动瘦身
2. 模块开启 declaredMode 
3. 基座预置依赖

#### 开启后的副作用

1. 由于基座提前启动，模块再次启动的时候，如果依赖了基座的依赖里有发布服务，那么模块在启动的时候还会再次发布，如果该问题有一定影响可以模块pom里exclude掉对应的依赖。
