# Spring IoC 容器源码复习

## 一、核心知识体系

### 1. 元数据层：BeanDefinition

BeanDefinition 是 Spring 对 Bean 配置的抽象，实现了**配置与实例的解耦**。

**继承体系：**
- `BeanDefinition`（接口）
  - `AbstractBeanDefinition`（抽象类，定义通用属性）
  - `RootBeanDefinition` —— 最终合并后的定义，用于实际创建 Bean
  - `GenericBeanDefinition` —— 通用定义，可动态设置 parent
  - `ScannedGenericBeanDefinition` —— `@Component` 扫描用
  - `AnnotatedGenericBeanDefinition` —— `@Configuration` 配置类用

**关键设计：** 先收集全部 BeanDefinition，再统一实例化。这使得 Spring 能在创建前分析依赖关系，处理循环依赖。

---

### 2. 容器层：BeanFactory

BeanFactory 是 IoC 容器的核心接口，定义了 Bean 的获取、判断、类型查询等基础行为。

**继承体系：**
- `BeanFactory`
  - `HierarchicalBeanFactory` —— 支持父子容器（如 Spring MVC 的 Root + Servlet 上下文）
  - `AutowireCapableBeanFactory` —— 支持自动装配和外部对象创建
  - `ConfigurableBeanFactory` —— 支持容器配置（添加 BPP、注册 Scope）
  - `ConfigurableListableBeanFactory` —— 完整功能（预实例化、获取 BeanDefinition）

**核心实现：** `DefaultListableBeanFactory`

| 数据结构 | 作用 |
|---------|------|
| `beanDefinitionMap` | ConcurrentHashMap，存储所有 BeanDefinition |
| `singletonObjects` | 一级缓存，存储完整单例 |
| `earlySingletonObjects` | 二级缓存，存储提前暴露的半成品 |
| `singletonFactories` | 三级缓存，存储 ObjectFactory |

---

### 3. Bean 获取流程：doGetBean()

```
getBean(name)
  → transformedBeanName(name) —— 解析别名和 FactoryBean 前缀
  → getSingleton(beanName) —— 查三级缓存
    → 命中？返回实例
    → 未命中 → 检查父容器 → 标记正在创建 → 合并 BeanDefinition
      → 处理 dependsOn 依赖 → createBean()
```

**核心设计：递归 + 缓存。** 遇到依赖递归调用 getBean(dep)，单例通过缓存保证只创建一次。

---

### 4. Bean 创建流程：doCreateBean()

```
doCreateBean()
  1. createBeanInstance() —— 实例化（new 对象，属性为空）
  2. addSingletonFactory() —— 将 ObjectFactory 放入三级缓存
  3. populateBean() —— 属性填充（@Autowired 注入发生在这里）
  4. initializeBean() —— 初始化
       → invokeAwareMethods()
       → BPP.postProcessBeforeInitialization() —— @PostConstruct
       → invokeInitMethods() —— afterPropertiesSet / init-method
       → BPP.postProcessAfterInitialization() —— AOP 代理创建
  5. registerDisposableBeanIfNecessary() —— 注册销毁回调
```

---

### 5. 循环依赖与三级缓存

**为什么需要三级而不是两级？** 因为 AOP 代理。

- **一级（singletonObjects）：** 存储完整的单例 Bean
- **二级（earlySingletonObjects）：** 存储提前暴露的引用，避免重复调用工厂
- **三级（singletonFactories）：** 存储 `ObjectFactory`，延迟创建代理

**关键时序：**

```
A 实例化 → 放入三级缓存 → A 注入 B → B 实例化 → B 注入 A
  → getSingleton("A") → 三级缓存命中 → ObjectFactory.getObject()
    → getEarlyBeanReference() → 提前创建代理（如果需要）
    → B 拿到代理 A（或原始 A）→ B 完成初始化
  → A 继续完成属性填充和初始化
```

**构造器注入无法解决循环依赖：** 因为构造器参数在 `createBeanInstance()` 就需要，而三级缓存在实例化之后才放入。

---

### 6. 依赖注入：populateBean()

**AutowiredAnnotationBeanPostProcessor 的处理流程：**

1. **预处理阶段**（`postProcessMergedBeanDefinition`）：
   - 扫描类及其父类的所有 `@Autowired`/`@Value` 字段和方法
   - 构建 `InjectionMetadata` 缓存到 `injectionMetadataCache`

2. **注入阶段**（`postProcessProperties`）：
   - 从缓存获取 `InjectionMetadata`
   - 遍历 `AutowiredFieldElement` / `AutowiredMethodElement`
   - 创建 `DependencyDescriptor` → `beanFactory.resolveDependency()`
   - 按类型查找候选 → `@Primary` → `@Priority` → 字段名匹配
   - 反射设置字段值 / 调用方法

**@Autowired vs @Resource：**
- `@Autowired`：先按类型，再按名称，由 `AutowiredAnnotationBeanPostProcessor` 处理
- `@Resource`：先按名称，再按类型，由 `CommonAnnotationBeanPostProcessor` 处理

---

### 7. ApplicationContext.refresh()

ApplicationContext 是 BeanFactory 的超集，集成了事件、资源、国际化、AOP 等基础设施。

**refresh() 的 12 步：**

| 步骤 | 方法 | 作用 |
|------|------|------|
| 1 | `prepareRefresh()` | 初始化 Environment，记录启动时间 |
| 2 | `obtainFreshBeanFactory()` | 获取/刷新内部的 BeanFactory |
| 3 | `prepareBeanFactory()` | 注册类加载器、添加默认 BPP |
| 4 | `postProcessBeanFactory()` | 【扩展点】子类自定义 |
| 5 | `invokeBeanFactoryPostProcessors()` | 【关键】解析 @Configuration、@ComponentScan、@Bean |
| 6 | `registerBeanPostProcessors()` | 【关键】注册所有 BPP |
| 7 | `initMessageSource()` | 国际化 |
| 8 | `initApplicationEventMulticaster()` | 事件广播器 |
| 9 | `onRefresh()` | 【扩展点】Spring Boot 中创建 WebServer |
| 10 | `registerListeners()` | 注册事件监听器 |
| 11 | `finishBeanFactoryInitialization()` | 【关键】`preInstantiateSingletons()`，实例化所有非懒加载单例 |
| 12 | `finishRefresh()` | 发布 ContextRefreshedEvent |

**Spring Boot 启动流程：**

```
SpringApplication.run()
  → createApplicationContext() —— 推断上下文类型
  → prepareContext() —— 加载主类、注册特殊 Bean
  → refreshContext() —— 调用 refresh()
```

---

### 8. BeanPostProcessor 扩展点

BPP 是 Spring 的插件化架构核心。Spring 自身功能（AOP、注解注入、事务等）都基于 BPP 实现。

**继承体系：**

| 接口 | 方法 | 作用 |
|------|------|------|
| `BeanPostProcessor` | `postProcessBefore/AfterInitialization` | 初始化前后介入 |
| `InstantiationAwareBeanPostProcessor` | `postProcessBefore/AfterInstantiation`, `postProcessProperties` | 实例化阶段介入，处理属性注入 |
| `SmartInstantiationAwareBeanPostProcessor` | `predictBeanType`, `determineCandidateConstructors`, `getEarlyBeanReference` | 智能实例化：预测类型、推断构造器、早期引用 |
| `MergedBeanDefinitionPostProcessor` | `postProcessMergedBeanDefinition` | 合并 BeanDefinition 时缓存元数据 |
| `DestructionAwareBeanPostProcessor` | `postProcessBeforeDestruction` | 销毁前处理 |

**Spring 内置核心 BPP：**

| BPP | 作用 | 阶段 |
|-----|------|------|
| `AutowiredAnnotationBeanPostProcessor` | `@Autowired`、`@Value` | 属性填充 |
| `CommonAnnotationBeanPostProcessor` | `@Resource`、`@PostConstruct`、`@PreDestroy` | 属性填充 + 初始化前 + 销毁前 |
| `ApplicationContextAwareProcessor` | Aware 接口注入 | 初始化前 |
| `AbstractAutoProxyCreator` | AOP 代理创建 | 初始化后（循环依赖时提前） |

**BPP 执行顺序：** `PriorityOrdered` → `Ordered` → 普通。`AutowiredAnnotationBeanPostProcessor` 是 `PriorityOrdered`，确保在其他 BPP 之前完成注入。

---

## 二、源码坐标速查

| 类/接口 | 完整路径 | 核心方法 |
|---------|---------|---------|
| `BeanFactory` | `org.springframework.beans.factory.BeanFactory` | `getBean()` |
| `DefaultListableBeanFactory` | `...beans.factory.support.DefaultListableBeanFactory` | `preInstantiateSingletons()`, `resolveDependency()` |
| `AbstractBeanFactory` | `...beans.factory.support.AbstractBeanFactory` | `doGetBean()` |
| `AbstractAutowireCapableBeanFactory` | `...beans.factory.support.AbstractAutowireCapableBeanFactory` | `createBean()`, `doCreateBean()`, `populateBean()`, `initializeBean()` |
| `DefaultSingletonBeanRegistry` | `...beans.factory.support.DefaultSingletonBeanRegistry` | `getSingleton()`, `addSingletonFactory()`, `addSingleton()` |
| `BeanDefinition` | `...beans.factory.config.BeanDefinition` | 元数据定义接口 |
| `RootBeanDefinition` | `...beans.factory.support.RootBeanDefinition` | 最终合并定义 |
| `ApplicationContext` | `...context.ApplicationContext` | 应用上下文接口 |
| `AbstractApplicationContext` | `...context.support.AbstractApplicationContext` | `refresh()` |
| `AutowiredAnnotationBeanPostProcessor` | `...beans.factory.annotation.AutowiredAnnotationBeanPostProcessor` | `postProcessMergedBeanDefinition()`, `postProcessProperties()` |
| `AbstractAutoProxyCreator` | `...aop.framework.autoproxy.AbstractAutoProxyCreator` | `postProcessAfterInitialization()`, `getEarlyBeanReference()` |

---

## 三、关键设计认知

1. **配置与实例解耦：** BeanDefinition 的存在让 Spring 能在实例化前全局分析依赖关系。

2. **三级缓存的本质：** 不只是解决循环依赖，更要保证循环依赖中注入的 AOP 代理对象一致。三级缓存（ObjectFactory）支持延迟创建代理。

3. **模板方法模式：** `refresh()` 是典型的模板方法，定义固定流程，子类通过 `postProcessBeanFactory()`、`onRefresh()` 等钩子扩展。

4. **责任链模式：** 多个 BPP 按顺序形成链，每个 BPP 都可以替换 Bean 实例。这是 Spring 插件化的基础。

5. **组合优于继承：** ApplicationContext 持有 BeanFactory 而非继承，保持职责分离。

6. **延迟实例化的代价与收益：** ApplicationContext 默认启动时实例化所有单例，牺牲启动速度换取运行时性能。`@Lazy` 可以按需加载。
