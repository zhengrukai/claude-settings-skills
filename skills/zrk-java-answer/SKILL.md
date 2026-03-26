---
name: zrk-java-reviewer
description: 问答场景
---

## 一、Java 基础知识点

### 1. 面向对象编程（OOP）
- **封装** - 隐藏内部实现，提供公共接口
- **继承** - 代码复用，建立类层次关系
- **多态** - 同一接口多种实现，提高灵活性
- **抽象** - 提取共同特征，简化复杂性

### 2. 设计原则（SOLID）
- **S** - 单一职责原则：一个类只负责一个功能
- **O** - 开闭原则：对扩展开放，对修改关闭
- **L** - 里氏替换原则：子类可替换父类
- **I** - 接口隔离原则：客户端不依赖不需要的接口
- **D** - 依赖倒置原则：依赖抽象而非具体实现

### 3. 常见设计模式
- **创建型** - Singleton、Factory、Builder、Prototype
- **结构型** - Adapter、Decorator、Facade、Proxy
- **行为型** - Strategy、Observer、Template Method、State

---

## 二、Spring Boot 核心概念

### 1. 依赖注入（DI）
- 通过构造器、Setter、字段注入依赖
- 降低类之间的耦合度
- 便于单元测试和模块替换

### 2. 面向切面编程（AOP）
- 横切关注点：日志、事务、权限、性能监控
- 通知类型：Before、After、Around、AfterReturning、AfterThrowing
- 切点表达式：精确定位需要增强的方法

### 3. 事务管理
- **ACID 特性** - 原子性、一致性、隔离性、持久性
- **隔离级别** - READ_UNCOMMITTED、READ_COMMITTED、REPEATABLE_READ、SERIALIZABLE
- **传播行为** - REQUIRED、REQUIRES_NEW、NESTED、SUPPORTS 等

### 4. 常用注解
- `@SpringBootApplication` - 启动类标记
- `@Component/@Service/@Repository/@Controller` - 组件注册
- `@Autowired/@Resource/@Inject` - 依赖注入
- `@Transactional` - 事务管理
- `@Cacheable/@CacheEvict` - 缓存控制

---

## 三、数据库与 ORM

### 1. SQL 优化
- 使用索引加速查询
- 避免全表扫描
- 合理使用 JOIN、GROUP BY、HAVING
- N+1 查询问题及解决方案

### 2. MyBatis/JPA 最佳实践
- 参数化查询防 SQL 注入
- 使用 PreparedStatement
- 批量操作优化性能
- 懒加载与急加载的权衡

### 3. 数据库连接池
- HikariCP、Druid 等连接池配置
- 连接泄漏检测与防止
- 连接超时与重试机制

---

## 四、并发与多线程

### 1. 线程安全
- **synchronized** - 内置锁，粗粒度控制
- **volatile** - 可见性保证，不保证原子性
- **Lock/ReentrantLock** - 显式锁，细粒度控制
- **ConcurrentHashMap** - 线程安全的 Map 实现

### 2. 线程池
- **ThreadPoolExecutor** - 核心线程数、最大线程数、队列、拒绝策略
- **ScheduledExecutorService** - 定时任务执行
- **ForkJoinPool** - 分治并行处理

### 3. 并发工具类
- **CountDownLatch** - 等待多个线程完成
- **CyclicBarrier** - 线程同步屏障
- **Semaphore** - 信号量控制并发数
- **AtomicInteger/AtomicReference** - 原子操作

---

## 五、异常处理与日志

### 1. 异常分类
- **检查异常** - 必须捕获或声明，如 IOException
- **运行时异常** - 可选捕获，如 NullPointerException
- **自定义异常** - 业务特定异常

### 2. 异常处理最佳实践
- 捕获具体异常而非 Exception
- 不吞掉异常，至少记录日志
- 使用 try-with-resources 自动关闭资源
- 异常链保留原始异常信息

### 3. 日志框架
- **SLF4J** - 日志门面，屏蔽具体实现
- **Logback/Log4j2** - 具体实现
- **日志级别** - TRACE、DEBUG、INFO、WARN、ERROR
- **异步日志** - 提高性能，避免阻塞业务线程

---

## 六、性能优化

### 1. 算法复杂度
- **时间复杂度** - O(1)、O(n)、O(n²)、O(nlogn) 等
- **空间复杂度** - 内存占用分析
- **权衡** - 时间换空间或空间换时间

### 2. 缓存策略
- **本地缓存** - HashMap、Guava Cache、Caffeine
- **分布式缓存** - Redis、Memcached
- **缓存失效** - LRU、LFU、TTL 等策略
- **缓存穿透/击穿/雪崩** - 预防与应对

### 3. 资源管理
- **内存泄漏** - 长生命周期对象持有短生命周期对象
- **连接泄漏** - 数据库连接、HTTP 连接未正确关闭
- **文件句柄泄漏** - 文件流未关闭

---

## 七、安全防护

### 1. 常见漏洞（OWASP Top 10）
- **SQL 注入** - 使用参数化查询防止
- **XSS 攻击** - 输入验证、输出编码
- **CSRF 攻击** - Token 验证、SameSite Cookie
- **认证/授权** - 强密码、权限最小化、定期审计

### 2. 数据保护
- **敏感信息加密** - 密码、身份证、银行卡等
- **传输加密** - HTTPS、TLS
- **存储加密** - 数据库加密、文件加密
- **日志脱敏** - 不记录敏感信息

### 3. 输入验证
- 白名单验证优于黑名单
- 长度、格式、范围检查
- 使用 Bean Validation（@NotNull、@Pattern 等）

---

## 八、微服务架构

### 1. 服务通信
- **同步** - REST、gRPC、Dubbo
- **异步** - 消息队列（RabbitMQ、Kafka）
- **服务发现** - Eureka、Consul、Nacos
- **负载均衡** - 轮询、加权、最少连接

### 2. 分布式事务
- **两阶段提交** - 强一致性，性能差
- **最终一致性** - 异步补偿、消息队列
- **Saga 模式** - 长事务分解为本地事务

### 3. 限流与熔断
- **限流** - 令牌桶、漏桶算法
- **熔断** - 快速失败，防止级联故障
- **降级** - 返回默认值或缓存数据
- **重试** - 指数退避、幂等性保证

---

## 九、测试与质量保证

### 1. 单元测试
- **JUnit** - 测试框架
- **Mockito** - Mock 对象库
- **AssertJ** - 流式断言
- **测试覆盖率** - 目标 70%+ 覆盖率

### 2. 集成测试
- **TestContainers** - Docker 容器化测试环境
- **@SpringBootTest** - Spring Boot 集成测试
- **数据库隔离** - 每个测试独立数据库

### 3. 代码质量
- **SonarQube** - 代码质量分析
- **代码审查** - Pull Request 审查
- **静态分析** - 检查潜在 bug、代码坏味道

---

## 十、常见问题解答

### Q1: 什么时候使用接口，什么时候使用抽象类？
**A:** 接口定义契约（能做什么），抽象类定义共同实现（怎么做）。多实现用接口，共享代码用抽象类。

### Q2: Spring 中 @Autowired 和 @Resource 的区别？
**A:** @Autowired 按类型注入（Spring），@Resource 按名称注入（Java 标准）。优先使用构造器注入。

### Q3: 如何避免 N+1 查询问题？
**A:** 使用 JOIN 一次查询、批量查询、或缓存。在 ORM 中配置急加载（FetchType.EAGER）。

### Q4: 什么是幂等性？为什么重要？
**A:** 多次调用结果相同。在分布式系统中防止重复操作导致的数据不一致。

### Q5: 如何设计高可用系统？
**A:** 冗余、监控、告警、自动故障转移、限流降级、异步处理、缓存。

## 输出要求
每次回答结束后，将完整输出内容保存为 Markdown 文件：
- 路径：C:\Users\Administrator\.claude\answer\
- 文件名格式：{YYYY-MM-DD}_{主题关键词}.md
- 使用 Write 工具完成保存，并告知用户文件路径