# Spring AOP 源码复习

## 一、AOP 核心概念

### 1. 术语对照

| 术语 | 含义 | Spring 中的体现 |
|------|------|----------------|
| **JoinPoint** | 连接点，程序执行过程中的某个点 | 方法调用 |
| **Pointcut** | 切点，匹配哪些 JoinPoint | `AspectJExpressionPointcut`、`ClassFilter`、`MethodMatcher` |
| **Advice** | 通知，切点处执行的逻辑 | `MethodInterceptor`、`MethodBeforeAdvice`、`AfterReturningAdvice` |
| **Advisor** | 切点+通知的粘合剂 | `PointcutAdvisor`、`DefaultPointcutAdvisor` |
| **Aspect** | 切面，多个切点+通知的集合 | `@Aspect` 注解类 |
| **Weaving** | 织入，将切面应用到目标对象 | 运行时织入（代理模式） |
| **Proxy** | 代理对象 | `JdkDynamicAopProxy`、`CglibAopProxy` |

### 2. Spring AOP vs AspectJ

| 特性 | Spring AOP | AspectJ |
|------|-----------|---------|
| 织入时机 | 运行时 | 编译时/加载时 |
| 实现方式 | 代理模式（JDK/CGLIB） | 字节码修改 |
| 拦截范围 | 仅方法调用 | 方法、字段、构造器、异常 |
| 性能 | 有代理开销 | 无额外开销 |

**Spring AOP 的设计哲学：** 方法拦截覆盖 90% 企业级场景，代理模式实现简单、与 IoC 天然集成。

---

## 二、代理实现层

### 1. 代理选择策略

`DefaultAopProxyFactory` 决定使用哪种代理：

```java
if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
    // 使用 CGLIB
    if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
        return new JdkDynamicAopProxy(config);  // 目标是接口，仍用 JDK
    }
    return new ObjenesisCglibAopProxy(config);  // CGLIB
} else {
    return new JdkDynamicAopProxy(config);  // JDK
}
```

| 条件 | 代理方式 |
|------|---------|
| 目标实现了接口，且未强制 `proxyTargetClass` | JDK 动态代理 |
| 目标未实现接口 | CGLIB |
| `proxyTargetClass = true` | CGLIB |
| 目标是接口 | JDK（即使 proxyTargetClass=true） |

### 2. JDK 动态代理

**原理：** 基于 `java.lang.reflect.Proxy` 生成实现目标接口的代理类。

**核心类：**
- `java.lang.reflect.Proxy` —— 生成代理类
- `java.lang.reflect.InvocationHandler` —— 方法调用处理
- `JdkDynamicAopProxy` —— Spring 实现 `InvocationHandler`
- `ReflectiveMethodInvocation` —— 责任链执行

**生成的代理类结构：**
```java
public final class $Proxy0 extends Proxy implements UserService {
    public void createUser(String name) {
        h.invoke(this, m3, new Object[]{name});  // 转发到 InvocationHandler
    }
}
```

**方法调用流程：**
```
外部调用 proxy.createUser()
  → $Proxy0.createUser() → InvocationHandler.invoke()
  → JdkDynamicAopProxy.invoke()
    → getInterceptorsAndDynamicInterceptionAdvice() —— 获取拦截器链
    → 有拦截器？ReflectiveMethodInvocation.proceed()
    → 无拦截器？直接反射调用目标方法
```

**ReflectiveMethodInvocation 责任链：**
```java
public Object proceed() {
    if (currentInterceptorIndex == interceptors.size() - 1) {
        return invokeJoinpoint();  // 调用目标方法（反射）
    }
    Object interceptor = interceptors.get(++currentInterceptorIndex);
    return ((MethodInterceptor) interceptor).invoke(this);
}
```

### 3. CGLIB 代理

**原理：** 通过 ASM 字节码生成目标类的子类，重写非 final 方法。

**核心类：**
- `net.sf.cglib.proxy.Enhancer` —— 增强器
- `net.sf.cglib.proxy.MethodInterceptor` —— CGLIB 拦截器
- `CglibAopProxy.DynamicAdvisedInterceptor` —— Spring CGLIB 拦截器
- `MethodProxy` —— 方法代理（FastClass 优化）

**FastClass 机制：**
- CGLIB 为代理类和目标类各生成一个 `FastClass`
- 用 `int` 索引标识方法，避免反射开销
- `MethodProxy.invokeSuper()` 通过 FastClass 直接调用

**生成的代理类结构：**
```java
public class UserService$$Enhancer extends UserService {
    private MethodInterceptor CGLIB$CALLBACK_0;
    
    public void createUser(String name) {
        CGLIB$CALLBACK_0.intercept(this, method, args, proxy);
    }
    
    // 用于绕过代理直接调用父类
    void CGLIB$createUser$0(String name) {
        super.createUser(name);
    }
}
```

**JDK vs CGLIB 对比：**

| 特性 | JDK | CGLIB |
|------|-----|-------|
| 实现方式 | 实现接口 | 继承目标类 |
| 要求 | 目标必须实现接口 | 目标不能是 final 类/方法 |
| 方法调用 | 反射 | FastClass 索引（无反射） |
| 代理生成速度 | 快 | 慢 |
| 代理类数量 | 代理接口方法 | 代理所有非 final 方法 |

---

## 三、Advisor 与拦截器链

### 1. Advisor 继承体系

```
Advisor
  ├── PointcutAdvisor —— 方法拦截
  │     ├── DefaultPointcutAdvisor
  │     ├── NameMatchMethodPointcutAdvisor
  │     ├── RegexpMethodPointcutAdvisor
  │     ├── AspectJExpressionPointcutAdvisor
  │     └── InstantiationModelAwarePointcutAdvisorImpl —— @Aspect 生成
  └── IntroductionAdvisor —— 引入新接口
```

### 2. @Aspect 到 Advisor 的转换

```
@Aspect 类
  → ReflectiveAspectJAdvisorFactory.getAdvisors()
    → 遍历每个 @Before/@After/@Around 方法
      → 创建 AspectJExpressionPointcut（解析表达式）
      → 创建对应的 Advice（AspectJMethodBeforeAdvice 等）
      → 封装为 InstantiationModelAwarePointcutAdvisorImpl
```

**一个 @Aspect 类中的每个通知方法生成一个独立的 Advisor。**

### 3. 拦截器链构建

`DefaultAdvisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice()`：

1. 遍历所有 Advisor
2. `ClassFilter.matches(targetClass)` —— 类级别过滤（启动时）
3. `MethodMatcher.matches(method, targetClass)` —— 方法静态匹配（启动时）
4. 匹配成功 → `AdvisorAdapterRegistry.getInterceptors(advisor)` → 转换为 `MethodInterceptor`
5. `MethodMatcher.isRuntime()` → 动态匹配包装为 `InterceptorAndDynamicMethodMatcher`
6. 结果缓存到 `methodCache`

### 4. AdvisorAdapter 适配器

将非 `MethodInterceptor` 的 Advice 包装为统一接口：

| Advice 类型 | 适配器 | 生成的 Interceptor |
|------------|--------|------------------|
| `MethodBeforeAdvice` | `MethodBeforeAdviceAdapter` | `MethodBeforeAdviceInterceptor` |
| `AfterReturningAdvice` | `AfterReturningAdviceAdapter` | `AfterReturningAdviceInterceptor` |
| `ThrowsAdvice` | `ThrowsAdviceAdapter` | `ThrowsAdviceInterceptor` |
| `MethodInterceptor` | 不需要适配 | 直接使用 |

**@Around 直接是 `MethodInterceptor`，不需要适配器。** 原因：@Around 需要完全控制 `proceed()` 调用，天然符合拦截器模式。

---

## 四、Pointcut 详解

### 1. Pointcut 组成

```java
public interface Pointcut {
    ClassFilter getClassFilter();      // 类过滤器
    MethodMatcher getMethodMatcher();  // 方法匹配器
}
```

### 2. MethodMatcher 的两种匹配

| 类型 | 方法 | 时机 | 场景 |
|------|------|------|------|
| **静态匹配** | `matches(Method, Class)` | 启动时 | `execution()`、`within()`、`@annotation()` |
| **动态匹配** | `matches(Method, Class, Object...)` | 运行时 | `args()`（依赖参数值） |

### 3. AspectJExpressionPointcut

使用 AspectJ 的 `PointcutParser` 解析表达式，`ShadowMatch` 判断匹配结果：

```java
PointcutExpression expression = parser.parsePointcutExpression("execution(...)");
ShadowMatch shadowMatch = expression.matchesMethodExecution(method);

if (shadowMatch.alwaysMatches()) { ... }      // 确定匹配
else if (shadowMatch.neverMatches()) { ... }  // 确定不匹配
else { ... }                                   // 需要运行时判断
```

---

## 五、Advice 详解

### 1. Advice 继承体系

```
Advice
  ├── BeforeAdvice
  │     └── MethodBeforeAdvice —— @Before
  ├── AfterAdvice
  │     ├── AfterReturningAdvice —— @AfterReturning
  │     └── ThrowsAdvice —— @AfterThrowing
  └── Interceptor (AOP Alliance)
        └── MethodInterceptor —— @Around、@After
```

### 2. 各类 Advice 的执行位置

```
方法调用
  → ExposeInvocationInterceptor（暴露 MethodInvocation）
    → MethodBeforeAdviceInterceptor —— @Before 执行
      → AspectJAfterAdvice —— @After（finally）
        → AfterReturningAdviceInterceptor —— @AfterReturning
          → ThrowsAdviceInterceptor —— @AfterThrowing
            → invokeJoinpoint() —— 目标方法执行
```

**注意：** `@After` 在 `try-finally` 中，保证无论是否异常都执行。

---

## 六、Spring AOP 触发流程

```
1. @EnableAspectJAutoProxy
   └── 注册 AnnotationAwareAspectJAutoProxyCreator（BPP）

2. refresh() → registerBeanPostProcessors()
   └── 注册 BPP

3. Bean 创建 → initializeBean()
   └── AbstractAutoProxyCreator.postProcessAfterInitialization()
       └── wrapIfNecessary(bean, beanName, cacheKey)
           ├── findCandidateAdvisors() —— 获取所有 Advisor
           ├── findAdvisorsThatCanApply() —— 筛选匹配的 Advisor
           │   └── AopUtils.canApply() —— ClassFilter + MethodMatcher
           ├── 有匹配 Advisor？
           │   ├── createProxy() —— 创建代理
           │   │   ├── DefaultAopProxyFactory —— 选择 JDK/CGLIB
           │   │   ├── 生成代理类
           │   │   └── 返回代理对象
           │   └── 用代理替换原始 Bean
           └── 无匹配 → 返回原始 Bean

4. 方法调用
   └── 代理对象拦截
       ├── JDK: JdkDynamicAopProxy.invoke()
       └── CGLIB: DynamicAdvisedInterceptor.intercept()
           └── getInterceptorsAndDynamicInterceptionAdvice() —— 查缓存或构建链
               └── ReflectiveMethodInvocation.proceed() —— 责任链执行
```

---

## 七、关键设计认知

1. **代理模式的局限性：** Spring AOP 只能拦截外部方法调用，同类内部调用（`this.method()`）无法被拦截，因为 `this` 指向目标对象而非代理对象。

2. **@Transactional 只对 public 方法生效：** JDK 代理只能代理接口 public 方法；Spring 显式检查方法修饰符，保证两种代理行为一致。

3. **三级缓存与 AOP 代理：** 循环依赖解决中，三级缓存的 `ObjectFactory` 延迟创建代理，保证循环依赖中注入的代理对象一致。

4. **责任链模式：** Spring AOP 使用责任链而非递归嵌套，每个 `MethodInterceptor` 控制是否调用 `proceed()`，实现灵活的通知组合。

5. **静态匹配缓存：** 拦截器链按方法缓存（`methodCache`），静态 Pointcut 只需计算一次，后续 O(1) 查缓存。

---

## 八、源码坐标速查

| 类/接口 | 路径 | 核心方法/作用 |
|---------|------|-------------|
| `AopProxy` | `org.springframework.aop.framework.AopProxy` | 代理接口 |
| `JdkDynamicAopProxy` | `...aop.framework.JdkDynamicAopProxy` | `invoke()` |
| `CglibAopProxy` | `...aop.framework.CglibAopProxy` | `getProxy()`、`DynamicAdvisedInterceptor.intercept()` |
| `DefaultAopProxyFactory` | `...aop.framework.DefaultAopProxyFactory` | `createAopProxy()` |
| `ReflectiveMethodInvocation` | `...aop.framework.ReflectiveMethodInvocation` | `proceed()` |
| `Advisor` | `org.springframework.aop.Advisor` | Advisor 接口 |
| `PointcutAdvisor` | `org.springframework.aop.PointcutAdvisor` | 含 Pointcut 的 Advisor |
| `Pointcut` | `org.springframework.aop.Pointcut` | 切点接口 |
| `AspectJExpressionPointcut` | `...aop.aspectj.AspectJExpressionPointcut` | AspectJ 表达式切点 |
| `DefaultAdvisorChainFactory` | `...aop.framework.DefaultAdvisorChainFactory` | `getInterceptorsAndDynamicInterceptionAdvice()` |
| `AdvisorAdapterRegistry` | `...aop.framework.adapter.AdvisorAdapterRegistry` | Advice 转 MethodInterceptor |
| `ReflectiveAspectJAdvisorFactory` | `...aop.aspectj.annotation.ReflectiveAspectJAdvisorFactory` | `@Aspect` 转 Advisor |
| `AbstractAutoProxyCreator` | `...aop.framework.autoproxy.AbstractAutoProxyCreator` | `postProcessAfterInitialization()` |
