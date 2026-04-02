# Spring 框架 - 面试知识点参考

## 目录
1. [IoC 容器](#ioc-容器)
2. [AOP](#aop)
3. [Bean 生命周期](#bean-生命周期)
4. [事务管理](#事务管理)
5. [循环依赖](#循环依赖)
6. [SpringBoot 自动配置](#springboot-自动配置)

---

## IoC 容器

### IoC 与 DI

- **IoC（控制反转）**：对象的创建控制权从代码转移到容器
- **DI（依赖注入）**：容器主动将依赖注入到对象中（IoC 的具体实现方式）

**三种注入方式**：
```java
// 1. 构造器注入（推荐，依赖不可变，便于测试）
@Autowired
public UserService(UserDao userDao) { ... }

// 2. Setter 注入（依赖可选时）
@Autowired
public void setUserDao(UserDao userDao) { ... }

// 3. 字段注入（不推荐，无法单元测试，隐藏依赖）
@Autowired
private UserDao userDao;
```

### BeanFactory vs ApplicationContext

| 对比 | BeanFactory | ApplicationContext |
|------|-------------|-------------------|
| 加载时机 | 懒加载（用时才初始化）| 立即加载（启动时初始化）|
| 功能 | 基础 IoC | IoC + AOP + 事件 + 国际化 + ... |
| 适用 | 资源受限环境 | 企业应用（绝大多数场景）|

---

## AOP

### 核心概念

| 概念 | 解释 |
|------|------|
| Aspect（切面）| 横切关注点的模块化 |
| Joinpoint（连接点）| 可被增强的方法执行点 |
| Pointcut（切点）| 通过表达式选择哪些连接点 |
| Advice（通知）| 在切点执行的增强逻辑 |
| Weaving（织入）| 将切面应用到目标对象的过程 |

### 五种通知类型

```java
@Before("execution(* com.example.service.*.*(..))")
public void before() {}          // 前置：方法执行前

@AfterReturning("...")
public void afterReturn() {}     // 后置：正常返回后

@AfterThrowing("...")
public void afterThrow() {}      // 异常：抛出异常后

@After("...")
public void after() {}           // 最终：相当于 finally

@Around("...")
public Object around(ProceedingJoinPoint pjp) {
    // 最强大：可控制是否执行目标方法
    Object result = pjp.proceed();
    return result;
}
```

**执行顺序（Spring 5.2.7+）**：
- 正常：Around前 → Before → 方法 → Around后 → AfterReturning → After
- 异常：Around前 → Before → 方法 → AfterThrowing → After

### AOP 实现原理

- **JDK 动态代理**：目标类有接口时使用，运行期生成代理类
- **CGLIB**：目标类无接口时使用，字节码继承目标类
- **Spring Boot 2.x**：默认使用 CGLIB（`spring.aop.proxy-target-class=true`）

---

## Bean 生命周期（高频⭐⭐⭐⭐⭐）

```
1. 实例化（Instantiation）
   - 调用构造函数创建对象

2. 属性填充（Populate）
   - 注入 @Autowired 依赖

3. 初始化（Initialization）
   - BeanNameAware.setBeanName()
   - BeanFactoryAware.setBeanFactory()
   - ApplicationContextAware.setApplicationContext()
   - BeanPostProcessor.postProcessBeforeInitialization()
   - @PostConstruct 方法
   - InitializingBean.afterPropertiesSet()
   - init-method
   - BeanPostProcessor.postProcessAfterInitialization()  ← AOP 代理在这里生成

4. 使用中

5. 销毁（Destruction）
   - @PreDestroy 方法
   - DisposableBean.destroy()
   - destroy-method
```

**记忆口诀**：实例化 → 填充属性 → Aware → BPP前 → PostConstruct → afterPropertiesSet → init → BPP后 → 使用 → PreDestroy → destroy

---

## 事务管理（高频⭐⭐⭐⭐）

### 七种传播行为

| 传播行为 | 描述 |
|---------|------|
| **REQUIRED**（默认）| 有事务用已有的，没有则新建 |
| **REQUIRES_NEW** | 总是新建事务，挂起外部事务 |
| **SUPPORTS** | 有事务就用，没有就非事务执行 |
| **NOT_SUPPORTED** | 非事务执行，挂起外部事务 |
| **MANDATORY** | 必须有事务，没有则抛异常 |
| **NEVER** | 非事务执行，有事务则抛异常 |
| **NESTED** | 在已有事务内创建嵌套事务（保存点）|

### 四种隔离级别

与数据库一致（见 database.md），Spring 默认使用数据库默认隔离级别（MySQL 是 REPEATABLE_READ）。

### @Transactional 失效场景（高频）

1. **方法非 public**：AOP 代理无法拦截
2. **同类内方法调用**：`this.methodB()` 不经过代理
3. **异常被 catch 吞掉**：事务感知不到异常
4. **异常类型不匹配**：默认只回滚 RuntimeException，受检异常不回滚（需配置 `rollbackFor`）
5. **数据库不支持事务**：如 MyISAM
6. **Bean 未被 Spring 管理**：如手动 new 的对象

```java
// 正确写法：指定回滚异常类型
@Transactional(rollbackFor = Exception.class)
public void doSomething() throws Exception { ... }
```

---

## 循环依赖（高频⭐⭐⭐⭐）

### 三级缓存

```
一级缓存（singletonObjects）：成品 Bean
二级缓存（earlySingletonObjects）：半成品 Bean（已实例化，未完成属性填充）
三级缓存（singletonFactories）：ObjectFactory（可生成 Bean 的工厂）
```

**解决流程（A 依赖 B，B 依赖 A）**：
1. A 开始创建 → 实例化 → 放入三级缓存
2. A 填充属性时需要 B → 触发 B 创建
3. B 开始创建 → 实例化 → 放入三级缓存
4. B 填充属性时需要 A → 从三级缓存获取 A 的工厂 → 生成 A 的早期引用 → 放入二级缓存
5. B 完成初始化 → 放入一级缓存
6. A 获取到 B → 继续完成初始化 → 放入一级缓存

**为什么需要三级缓存（而不是两级）？**
三级缓存存的是 `ObjectFactory`，可以在获取时生成 AOP 代理对象（懒生成），保证代理对象的唯一性。

**无法解决的循环依赖**：
- 构造器注入（实例化阶段就需要依赖，此时还未入缓存）
- `@Scope("prototype")` 的 Bean

---

## SpringBoot 自动配置（高频⭐⭐⭐⭐）

### @SpringBootApplication 组合注解

```java
@SpringBootConfiguration  // 相当于 @Configuration
@EnableAutoConfiguration  // 开启自动配置
@ComponentScan            // 扫描当前包及子包
```

### 自动配置原理

1. `@EnableAutoConfiguration` 导入 `AutoConfigurationImportSelector`
2. 读取 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`（旧版：`spring.factories`）
3. 获取所有 `XXXAutoConfiguration` 类名
4. 根据 `@ConditionalOnXxx` 条件注解过滤，符合条件的才加载

**常用条件注解**：
| 注解 | 条件 |
|------|------|
| `@ConditionalOnClass` | classpath 中存在某个类 |
| `@ConditionalOnMissingBean` | 容器中不存在某个 Bean |
| `@ConditionalOnProperty` | 配置项满足条件 |
| `@ConditionalOnWebApplication` | 是 Web 应用 |

### 自定义 Starter 步骤

1. 创建 `xxx-spring-boot-starter` 模块（约定命名）
2. 编写 `XxxAutoConfiguration`（添加 `@Configuration`）
3. 在 `META-INF/spring/...AutoConfiguration.imports` 中注册
4. 使用 `@ConditionalOnXxx` 控制加载条件
