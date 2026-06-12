# Spring 事务管理源码复习

## 一、事务管理核心接口

### 1. PlatformTransactionManager

Spring 事务的最高层抽象，定义事务的基本操作：

```java
public interface PlatformTransactionManager {
    TransactionStatus getTransaction(@Nullable TransactionDefinition definition);
    void commit(TransactionStatus status);
    void rollback(TransactionStatus status);
}
```

| 实现类 | 适用场景 |
|--------|---------|
| `DataSourceTransactionManager` | 单数据源 JDBC/MyBatis |
| `JtaTransactionManager` | 分布式事务 |
| `HibernateTransactionManager` | Hibernate ORM |

### 2. TransactionDefinition

定义事务的属性：

| 属性 | 说明 |
|------|------|
| `propagationBehavior` | 传播行为（REQUIRED、REQUIRES_NEW 等）|
| `isolationLevel` | 隔离级别（READ_COMMITTED、REPEATABLE_READ 等）|
| `timeout` | 超时时间（秒）|
| `readOnly` | 是否只读 |

### 3. TransactionStatus

事务运行时的状态：

| 方法 | 说明 |
|------|------|
| `isNewTransaction()` | 是否是新事务 |
| `hasSavepoint()` | 是否有 Savepoint |
| `isRollbackOnly()` | 是否标记了只回滚 |
| `isCompleted()` | 是否已完成 |

---

## 二、事务代理机制

### 1. @EnableTransactionManagement 的作用

```java
@EnableTransactionManagement
  → 注册 InfrastructureAdvisorAutoProxyCreator（BPP）
  → 注册 BeanFactoryTransactionAttributeSourceAdvisor（Advisor）
  → 注册 TransactionInterceptor（Advice）
  → 注册 AnnotationTransactionAttributeSource（解析 @Transactional）
```

### 2. TransactionInterceptor 的执行流程

```java
TransactionInterceptor.invoke()
  → invokeWithinTransaction(method, targetClass, invocation)
    → 解析 @Transactional 注解
    → 获取 PlatformTransactionManager
    → createTransactionIfNecessary() —— 开启事务
    → invocation.proceedWithInvocation() —— 执行业务
    → 正常返回：commitTransactionAfterReturning() —— 提交
    → 抛出异常：completeTransactionAfterThrowing() —— 回滚
```

### 3. Connection 路由机制

**为什么 JdbcTemplate/MyBatis 能自动使用同一个事务连接？**

```
开启事务：
  DataSourceTransactionManager.doBegin()
    → Connection con = dataSource.getConnection()
    → con.setAutoCommit(false)
    → TransactionSynchronizationManager.bindResource(dataSource, conHolder)
      // 绑定 (DataSource, ConnectionHolder) 到 ThreadLocal

执行 SQL：
  JdbcTemplate.update(sql)
    → DataSourceUtils.getConnection(dataSource)
      → TransactionSynchronizationManager.getResource(dataSource)
        // 从 ThreadLocal 查找绑定的 ConnectionHolder
        // 找到了！返回事务中的 Connection
        // 没找到，从 DataSource 获取新 Connection

事务结束：
  解绑资源，恢复 autoCommit
```

---

## 三、事务传播行为

### 1. 7 种传播行为

| 传播行为 | 有事务时 | 无事务时 | 说明 |
|---------|---------|---------|------|
| `REQUIRED` | 加入 | 新建 | 默认 |
| `REQUIRES_NEW` | 挂起，新建 | 新建 | 独立事务 |
| `NESTED` | Savepoint | 新建 | 嵌套事务 |
| `SUPPORTS` | 加入 | 非事务 | 支持事务 |
| `MANDATORY` | 加入 | 抛异常 | 强制事务 |
| `NOT_SUPPORTED` | 挂起 | 非事务 | 不支持事务 |
| `NEVER` | 抛异常 | 非事务 | 从不要事务 |

### 2. 核心决策树

```
getTransaction()
  → isExistingTransaction()?
    ├── 否
    │   ├── MANDATORY → 抛异常
    │   ├── REQUIRED/REQUIRES_NEW/NESTED → startTransaction()
    │   └── 其他 → 非事务执行
    └── 是
        ├── NEVER → 抛异常
        ├── NOT_SUPPORTED → suspend() → 非事务
        ├── REQUIRES_NEW → suspend() → startTransaction()
        ├── NESTED → createSavepoint()
        └── REQUIRED/SUPPORTS/MANDATORY → 加入已有事务
```

### 3. 挂起与恢复

**suspend()：**
1. 暂停事务同步
2. `doSuspend()` —— 解绑 ConnectionHolder
3. 清空 ThreadLocal（事务名、隔离级别等）
4. 返回 `SuspendedResourcesHolder`

**resume()：**
1. `doResume()` —— 重新绑定 ConnectionHolder
2. 恢复事务上下文
3. 恢复事务同步

### 4. REQUIRES_NEW vs NESTED 的本质区别

| 维度 | REQUIRES_NEW | NESTED |
|------|-------------|--------|
| Connection | 新的独立 Connection | 同一个 Connection |
| 回滚影响 | 内层回滚不影响外层 | 内层回滚到 Savepoint，不影响外层 |
| 提交独立 | 内层独立提交 | 外层统一提交 |
| 数据库支持 | 所有数据库 | 需要支持 Savepoint（JDBC 3.0+）|

### 5. 传播行为的时序

**REQUIRED：**
```
外层 methodA() → startTransaction()
  → Connection A
  → 调用 methodB() → joinTransaction()
    → 使用 Connection A
  → methodB 完成，不提交
→ methodA 完成 → commit(Connection A)
```

**REQUIRES_NEW：**
```
外层 methodA() → startTransaction()
  → Connection A
  → 调用 methodB() → suspend(Connection A) → startTransaction()
    → Connection B（独立事务）
    → methodB 完成 → commit(Connection B) → resume(Connection A)
  → Connection A 恢复
→ methodA 完成 → commit(Connection A)
```

**NESTED：**
```
外层 methodA() → startTransaction()
  → Connection A
  → 调用 methodB() → createSavepoint(Connection A)
    → 使用 Connection A
    → methodB 异常 → rollbackToSavepoint(Connection A)
  → Connection A 继续
→ methodA 完成 → commit(Connection A)
```

---

## 四、事务隔离级别

### 1. 隔离级别定义

```java
ISOLATION_DEFAULT = -1;              // 数据库默认
ISOLATION_READ_UNCOMMITTED = 1;      // 读未提交
ISOLATION_READ_COMMITTED = 2;        // 读已提交
ISOLATION_REPEATABLE_READ = 4;       // 可重复读
ISOLATION_SERIALIZABLE = 8;          // 串行化
```

### 2. 并发问题

| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
|---------|------|-----------|------|
| READ_UNCOMMITTED | ❌ | ❌ | ❌ |
| READ_COMMITTED | ✅ | ❌ | ❌ |
| REPEATABLE_READ | ✅ | ✅ | ✅ (InnoDB) |
| SERIALIZABLE | ✅ | ✅ | ✅ |

### 3. Spring 如何设置隔离级别

```java
DataSourceTransactionManager.doBegin()
  → DataSourceUtils.prepareConnectionForTransaction(con, definition)
    → con.setTransactionIsolation(definition.getIsolationLevel())
```

**数据库实现：** 隔离级别由数据库引擎实现，Spring 只是通过 JDBC API 告诉数据库。

---

## 五、事务同步机制

### 1. TransactionSynchronization 接口

```java
public interface TransactionSynchronization {
    void suspend();
    void resume();
    void flush();
    void beforeCommit(boolean readOnly);   // 提交前
    void beforeCompletion();               // 完成前
    void afterCommit();                    // 提交后
    void afterCompletion(int status);      // 完成后
}
```

### 2. 注册与触发

```java
// 注册
TransactionSynchronizationManager.registerSynchronization(sync);

// 触发时机：
beforeCommit()   → 已决定提交，commit() 尚未执行
commit()         → JDBC Connection.commit()
afterCommit()    → commit() 已成功
beforeCompletion() → 事务结束前
afterCompletion()  → 事务完全结束
```

### 3. 实际应用场景

**事务提交后发 MQ：**
```java
@Transactional
public void createOrder(Order order) {
    orderDao.save(order);
    
    TransactionSynchronizationManager.registerSynchronization(
        new TransactionSynchronization() {
            @Override
            public void afterCommit() {
                mqTemplate.send("order-topic", order);
            }
        }
    );
}
```

**使用 @TransactionalEventListener：**
```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void handleOrderCreated(OrderCreatedEvent event) {
    mqTemplate.send("order-topic", event.getOrder());
}
```

---

## 六、关键设计认知

1. **线程绑定资源（Thread-bound Resources）：** Spring 事务通过 `TransactionSynchronizationManager` 将 `Connection` 绑定到当前线程的 `ThreadLocal`，保证同一线程中所有数据库操作复用同一个 `Connection`。

2. **挂起与恢复：** `REQUIRES_NEW` 和 `NOT_SUPPORTED` 需要挂起外层事务。挂起时解绑资源、清空 ThreadLocal；恢复时重新绑定。外层和内层事务**物理隔离**（不同 Connection）。

3. **Savepoint 机制：** `NESTED` 通过 JDBC `Connection.setSavepoint()` 实现逻辑嵌套。同一个 Connection，通过 Savepoint 隔离回滚。

4. **rollbackOn 规则：** 默认 `RuntimeException` 和 `Error` 回滚，Checked Exception 不回滚。可通过 `rollbackFor` / `noRollbackFor` 自定义。

5. **只读事务：** `readOnly = true` 设置 `Connection.setReadOnly(true)`，数据库可以优化锁策略。但只对非第一个查询生效（MySQL InnoDB 忽略此设置）。

6. **同类内部调用不生效：** `this.method()` 不经过代理，事务切面不生效。解决方式：注入自身代理、使用 `AopContext.currentProxy()`、重构代码。

---

## 七、源码坐标速查

| 类/接口 | 路径 | 核心方法/作用 |
|---------|------|-------------|
| `PlatformTransactionManager` | `org.springframework.transaction.PlatformTransactionManager` | 事务管理器接口 |
| `AbstractPlatformTransactionManager` | `...support.AbstractPlatformTransactionManager` | `getTransaction()`、`commit()`、`rollback()` |
| `DataSourceTransactionManager` | `...datasource.DataSourceTransactionManager` | `doBegin()`、`doCommit()`、`doRollback()` |
| `TransactionInterceptor` | `...interceptor.TransactionInterceptor` | `invoke()` |
| `TransactionAspectSupport` | `...interceptor.TransactionAspectSupport` | `invokeWithinTransaction()` |
| `TransactionSynchronizationManager` | `...support.TransactionSynchronizationManager` | ThreadLocal 资源管理 |
| `TransactionSynchronization` | `...support.TransactionSynchronization` | 事务同步回调接口 |
| `DataSourceUtils` | `...datasource.DataSourceUtils` | `getConnection()`、`prepareConnectionForTransaction()` |
| `@EnableTransactionManagement` | `...annotation.EnableTransactionManagement` | 开启事务管理 |
| `ProxyTransactionManagementConfiguration` | `...annotation.ProxyTransactionManagementConfiguration` | 注册事务 Bean |
