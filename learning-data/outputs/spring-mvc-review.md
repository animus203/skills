# Spring MVC 源码复习

## 一、整体架构

```
HTTP 请求
  → DispatcherServlet（前端控制器）
    → HandlerMapping（找 Controller）
    → HandlerAdapter（执行 Controller）
    → HandlerInterceptor（拦截器）
    → ViewResolver（解析视图）
    → View（渲染页面）
```

---

## 二、DispatcherServlet —— 前端控制器

### 继承体系

```
HttpServlet → HttpServletBean → FrameworkServlet → DispatcherServlet
```

| 类 | 职责 |
|----|------|
| `HttpServletBean` | 将 Servlet 配置参数转换为 Bean 属性 |
| `FrameworkServlet` | 初始化 WebApplicationContext，处理 HTTP 方法分发 |
| `DispatcherServlet` | 请求分发的核心，协调 9 大组件 |

### 9 大核心组件

| 组件 | 作用 | 默认实现 |
|------|------|---------|
| `HandlerMapping` | URL → Handler 映射 | `RequestMappingHandlerMapping` |
| `HandlerAdapter` | 执行 Handler | `RequestMappingHandlerAdapter` |
| `HandlerExceptionResolver` | 异常处理 | `ExceptionHandlerExceptionResolver` |
| `ViewResolver` | 视图解析 | `InternalResourceViewResolver` |
| `MultipartResolver` | 文件上传 | `StandardServletMultipartResolver` |
| `LocaleResolver` | 本地化 | `AcceptHeaderLocaleResolver` |
| `ThemeResolver` | 主题 | `FixedThemeResolver` |
| `RequestToViewNameTranslator` | 请求→视图名 | `DefaultRequestToViewNameTranslator` |
| `FlashMapManager` | 重定向参数 | `SessionFlashMapManager` |

### doDispatch() 核心流程

```
doDispatch()
  1. checkMultipart() —— 文件上传处理
  2. getHandler() —— HandlerMapping 找匹配的 Controller
  3. getHandlerAdapter() —— 找到能执行的 Adapter
  4. applyPreHandle() —— 执行拦截器 preHandle
  5. handle() —— 执行 Controller 方法
  6. applyPostHandle() —— 执行拦截器 postHandle
  7. processDispatchResult() —— 处理结果（异常/视图渲染）
  8. triggerAfterCompletion() —— 执行拦截器 afterCompletion
```

---

## 三、HandlerMapping —— URL 到 Handler 的映射

### 继承体系

```
HandlerMapping
  ↑ AbstractHandlerMapping
    ↑ AbstractHandlerMethodMapping
      ↑ RequestMappingHandlerMapping（最常用）
```

### 启动时扫描

```java
afterPropertiesSet()
  → initHandlerMethods()
    → 遍历所有 Bean
      → isHandler() —— 检查是否有 @Controller 或 @RequestMapping
        → detectHandlerMethods() —— 注册方法映射
          → getMappingForMethod() —— 创建 RequestMappingInfo
            → createRequestMappingInfo(method) —— 方法级别
            → createRequestMappingInfo(handlerType) —— 类级别（前缀）
            → 合并：typeInfo.combine(info)
          → registerHandlerMethod() —— 注册到 MappingRegistry
```

### RequestMappingInfo 结构

```java
RequestMappingInfo
  ├── PatternsRequestCondition —— URL 路径匹配
  ├── RequestMethodsRequestCondition —— HTTP 方法匹配
  ├── ParamsRequestCondition —— 请求参数匹配
  ├── HeadersRequestCondition —— 请求头匹配
  ├── ConsumesRequestCondition —— Content-Type 匹配
  └── ProducesRequestCondition —— Accept 匹配
```

### 运行时匹配

```java
getHandler(request)
  → getHandlerInternal(request)
    → lookupHandlerMethod(lookupPath, request)
      → 查 urlLookup（直接 URL 匹配）
      → 查 mappingLookup（通配符匹配）
      → 多个匹配？排序选择最佳
      → 返回 HandlerMethod
```

### URL 匹配规则（AntPathMatcher）

| 通配符 | 说明 |
|--------|------|
| `?` | 匹配单个字符 |
| `*` | 匹配单层路径 |
| `**` | 匹配多层路径 |
| `{var}` | 路径变量 |
| `{var:regex}` | 正则路径变量 |

---

## 四、HandlerAdapter —— 执行 Controller 方法

### 核心实现：RequestMappingHandlerAdapter

```java
handleInternal()
  → invokeHandlerMethod()
    ├── 创建 WebDataBinderFactory（@InitBinder）
    ├── 创建 ModelFactory（@ModelAttribute）
    ├── 创建 ServletInvocableHandlerMethod
    │   ├── 设置参数解析器（argumentResolvers）
    │   └── 设置返回值处理器（returnValueHandlers）
    ├── 创建 ModelAndViewContainer
    │
    ├── invokeAndHandle()
    │   ├── invokeForRequest()
    │   │   ├── getMethodArgumentValues() —— 解析参数
    │   │   │   ├── @RequestParam → RequestParamMethodArgumentResolver
    │   │   │   ├── @PathVariable → PathVariableMethodArgumentResolver
    │   │   │   ├── @RequestBody → RequestResponseBodyMethodProcessor
    │   │   │   │   └── readWithMessageConverters() → Jackson → Java 对象
    │   │   │   └── HttpServletRequest → ServletRequestMethodArgumentResolver
    │   │   └── doInvoke() —— 反射调用方法
    │   │
    │   └── handleReturnValue() —— 处理返回值
    │       ├── @ResponseBody → RequestResponseBodyMethodProcessor
    │       │   └── writeWithMessageConverters() → Java 对象 → Jackson JSON
    │       ├── String → ViewNameMethodReturnValueHandler
    │       ├── ModelAndView → ModelAndViewMethodReturnValueHandler
    │       └── ResponseEntity → HttpEntityMethodProcessor
    │
    └── getModelAndView() —— 构建 ModelAndView
```

### 参数解析器

| 解析器 | 支持 |
|--------|------|
| `RequestParamMethodArgumentResolver` | `@RequestParam` |
| `PathVariableMethodArgumentResolver` | `@PathVariable` |
| `RequestResponseBodyMethodProcessor` | `@RequestBody` |
| `ServletRequestMethodArgumentResolver` | `HttpServletRequest` |
| `ServletResponseMethodArgumentResolver` | `HttpServletResponse` |
| `RequestHeaderMethodArgumentResolver` | `@RequestHeader` |

### 返回值处理器

| 处理器 | 支持 |
|--------|------|
| `RequestResponseBodyMethodProcessor` | `@ResponseBody` |
| `ModelAndViewMethodReturnValueHandler` | `ModelAndView` |
| `ViewNameMethodReturnValueHandler` | `String`（视图名）|
| `HttpEntityMethodProcessor` | `ResponseEntity` |

### HttpMessageConverter

| 转换器 | 格式 |
|--------|------|
| `ByteArrayHttpMessageConverter` | `application/octet-stream` |
| `StringHttpMessageConverter` | `text/*` |
| `MappingJackson2HttpMessageConverter` | `application/json` |

---

## 五、HandlerInterceptor —— 请求拦截器

### 接口定义

```java
public interface HandlerInterceptor {
    default boolean preHandle(HttpServletRequest request, HttpServletResponse response,
        Object handler) throws Exception { return true; }
    
    default void postHandle(HttpServletRequest request, HttpServletResponse response,
        Object handler, @Nullable ModelAndView modelAndView) throws Exception {}
    
    default void afterCompletion(HttpServletRequest request, HttpServletResponse response,
        Object handler, @Nullable Exception ex) throws Exception {}
}
```

### 执行顺序

```
请求进入
  → preHandle 1 → preHandle 2 → preHandle 3
    → 执行 Handler（Controller 方法）
  ← postHandle 3 ← postHandle 2 ← postHandle 1
    → 渲染视图（如果有）
  ← afterCompletion 3 ← afterCompletion 2 ← afterCompletion 1
```

**preHandle 正序，postHandle/afterCompletion 逆序。** 这是责任链模式，保证先进入的后退出。

### 与 Filter 的区别

| 特性 | HandlerInterceptor | Filter |
|------|-------------------|--------|
| 所属规范 | Spring MVC | Servlet |
| 执行时机 | DispatcherServlet 内部 | Servlet 容器层面 |
| 能否获取 Handler | ✅ 可以 | ❌ 不能 |
| 能否修改 ModelAndView | ✅ 可以 | ❌ 不能 |
| 配置方式 | `WebMvcConfigurer` | `web.xml` / `@WebFilter` |

---

## 六、ViewResolver —— 视图解析

### 继承体系

```
ViewResolver
  ↑ AbstractCachingViewResolver
    ↑ UrlBasedViewResolver
      ↑ InternalResourceViewResolver —— JSP
```

### InternalResourceViewResolver

```java
// prefix = "/WEB-INF/views/"
// suffix = ".jsp"
// viewName = "home"
// → /WEB-INF/views/home.jsp
```

### 视图解析流程

```java
render(ModelAndView mv, request, response)
  → resolveViewName(viewName, locale)
    → 遍历所有 ViewResolver
      → ViewResolver.resolveViewName()
        → 查缓存（AbstractCachingViewResolver）
        → 缓存未命中 → 创建 View
    → 返回 View
  → view.render(model, request, response)
    → InternalResourceView.render()
      → exposeModelAsRequestAttributes() —— Model → Request 属性
      → RequestDispatcher.forward() —— 转发到 JSP
```

### 前后端分离时代

在 REST API 项目中：
- `@RestController` = `@Controller` + `@ResponseBody`
- 返回值直接由 `HttpMessageConverter` 转为 JSON
- ViewResolver 不参与

---

## 七、完整请求处理时序图

```
客户端 HTTP 请求
    ↓
Servlet 容器（Tomcat）
    ↓
Filter 1 → Filter 2 → Filter 3
    ↓
DispatcherServlet.service()
    ↓
FrameworkServlet.processRequest()
    ↓
doDispatch()
    ├── checkMultipart() —— 文件上传
    ├── getHandler()
    │   └── RequestMappingHandlerMapping
    │       └── lookupHandlerMethod()
    │           └── 返回 HandlerMethod
    ├── getHandlerAdapter()
    │   └── RequestMappingHandlerAdapter
    ├── applyPreHandle()
    │   └── HandlerInterceptor.preHandle() 正序
    ├── handle()
    │   └── RequestMappingHandlerAdapter.handleInternal()
    │       └── invokeHandlerMethod()
    │           └── ServletInvocableHandlerMethod.invokeAndHandle()
    │               ├── invokeForRequest()
    │               │   ├── getMethodArgumentValues() —— 参数解析
    │               │   └── doInvoke() —— 反射调用
    │               └── handleReturnValue() —— 返回值处理
    ├── applyPostHandle()
    │   └── HandlerInterceptor.postHandle() 逆序
    ├── processDispatchResult()
    │   ├── processHandlerException() —— 异常处理
    │   └── render() —— 视图渲染
    │       ├── resolveViewName() —— ViewResolver
    │       └── view.render() —— 渲染页面
    └── triggerAfterCompletion()
        └── HandlerInterceptor.afterCompletion() 逆序
    ↓
响应返回客户端
```

---

## 八、关键设计认知

1. **前端控制器模式：** DispatcherServlet 作为唯一入口，统一处理所有请求，将具体逻辑委托给各个组件。

2. **组件化设计：** 9 大组件各司其职，通过接口解耦，便于扩展和替换。

3. **启动时预扫描：** HandlerMapping 在启动时扫描所有 `@RequestMapping` 建立映射表，运行时 O(1) 查表，避免每次请求都反射扫描。

4. **适配器模式：** HandlerAdapter 适配不同类型的 Handler（`HandlerMethod`、`Controller` 接口、`HttpRequestHandler` 等），DispatcherServlet 无需关心 Handler 的具体类型。

5. **责任链模式：** HandlerInterceptor 的正序 preHandle + 逆序 postHandle/afterCompletion，保证资源正确释放。

6. **缓存优化：** `AbstractCachingViewResolver` 缓存解析后的 View 对象，避免重复解析。

---

## 九、源码坐标速查

| 类/接口 | 路径 | 核心方法/作用 |
|---------|------|-------------|
| `DispatcherServlet` | `org.springframework.web.servlet.DispatcherServlet` | `doDispatch()` |
| `HandlerMapping` | `org.springframework.web.servlet.HandlerMapping` | `getHandler()` |
| `RequestMappingHandlerMapping` | `...mvc.method.annotation.RequestMappingHandlerMapping` | 解析 `@RequestMapping` |
| `HandlerAdapter` | `org.springframework.web.servlet.HandlerAdapter` | `handle()` |
| `RequestMappingHandlerAdapter` | `...mvc.method.annotation.RequestMappingHandlerAdapter` | `handleInternal()` |
| `HandlerInterceptor` | `org.springframework.web.servlet.HandlerInterceptor` | `preHandle()`、`postHandle()`、`afterCompletion()` |
| `HandlerExecutionChain` | `org.springframework.web.servlet.HandlerExecutionChain` | 执行链管理 |
| `ViewResolver` | `org.springframework.web.servlet.ViewResolver` | `resolveViewName()` |
| `AbstractCachingViewResolver` | `...view.AbstractCachingViewResolver` | 视图缓存 |
| `InternalResourceViewResolver` | `...view.InternalResourceViewResolver` | JSP 视图解析 |
| `View` | `org.springframework.web.servlet.View` | `render()` |
