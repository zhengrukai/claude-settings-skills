# Java 核心 - 面试知识点参考

## 目录
1. [集合框架](#集合框架)
2. [String 相关](#string-相关)
3. [反射与动态代理](#反射与动态代理)
4. [异常体系](#异常体系)
5. [泛型](#泛型)
6. [IO/NIO](#ionio)
7. [Java 8+ 新特性](#java-8-新特性)

---

## 集合框架

### HashMap（高频⭐⭐⭐⭐⭐）

**底层结构**：数组 + 链表 + 红黑树（JDK 8+）

**关键参数**：
- 默认容量：`16`（必须是 2 的幂）
- 负载因子：`0.75`（时间与空间平衡）
- 树化阈值：`8`（链表 → 红黑树）
- 退树阈值：`6`（红黑树 → 链表）
- 最小树化容量：`64`（数组长度不足时优先扩容）

**hash 扰动函数**：
```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
作用：将高 16 位混入低 16 位，减少低位相同时的哈希碰撞。

**put 流程**：
1. hash(key) → (n-1) & hash 定位桶
2. 桶空 → 直接放入
3. key 相同 → 覆盖
4. 链表 → 尾插法追加；长度≥8 且数组≥64 → 转红黑树
5. size > threshold → resize()（容量翻倍，重新散列）

**JDK 7 vs JDK 8 区别**：
| 对比点 | JDK 7 | JDK 8 |
|--------|-------|-------|
| 插入方式 | 头插法 | 尾插法 |
| 数据结构 | 数组 + 链表 | 数组 + 链表 + 红黑树 |
| 并发问题 | 扩容死循环 | 数据丢失（无死循环）|

**高频追问**：
- 为什么 loadFactor 是 0.75？→ 泊松分布，碰撞概率与内存的最优平衡点
- 树化阈值为什么是 8？→ 泊松分布下链表超 8 节点概率 < 千万分之一
- 容量为什么必须是 2 的幂？→ `(n-1) & hash` 等价于取模，位运算更快，且能均匀分布

---

### ConcurrentHashMap（高频⭐⭐⭐⭐⭐）

**JDK 7**：分段锁（Segment 继承 ReentrantLock），默认 16 个 Segment，支持 16 个线程并发写

**JDK 8**：CAS + synchronized
- 桶为空时用 CAS 插入
- 桶非空时用 synchronized 锁头节点
- 锁粒度降到单个桶，并发度更高

**关键操作**：
```java
// 扩容：多线程协同扩容，每个线程负责一段
// size()：使用 baseCount + CounterCell[] 分段累加
```

**与 HashMap 的区别**：线程安全、不允许 null key/value（会抛 NPE）

---

### ArrayList vs LinkedList

| 对比 | ArrayList | LinkedList |
|------|-----------|------------|
| 底层 | 动态数组 | 双向链表 |
| 随机访问 | O(1) | O(n) |
| 头插/删 | O(n) | O(1) |
| 内存 | 紧凑（局部性好） | 每节点有前后指针 |
| 扩容 | 1.5 倍 | 无需扩容 |

**结论**：99% 场景用 ArrayList；需要大量头插/头删时考虑 LinkedList（实际很少）

---

### HashSet / LinkedHashSet / TreeSet

- **HashSet**：基于 HashMap，value 固定为 `PRESENT` 对象
- **LinkedHashSet**：基于 LinkedHashMap，维护插入顺序
- **TreeSet**：基于 TreeMap（红黑树），自然排序或自定义 Comparator

---

## String 相关

### String / StringBuilder / StringBuffer

| 对比 | String | StringBuilder | StringBuffer |
|------|--------|---------------|--------------|
| 可变性 | 不可变 | 可变 | 可变 |
| 线程安全 | 安全（不可变） | 不安全 | 安全（synchronized）|
| 性能 | 最低（拼接创建新对象）| 最高 | 中 |

**String 不可变的原因**：
1. `char[]` 用 `final` 修饰（JDK 9+ 改为 `byte[]`）
2. 没有提供修改数组内容的方法
3. 类本身是 `final`，无法被继承覆盖

**String 常量池**：
- 字面量赋值 → 字符串常量池
- `new String("abc")` → 堆中新对象，`intern()` 可入池
- JDK 7+ 常量池移到堆中（原在方法区 PermGen）

---

## 反射与动态代理

### 反射

```java
Class<?> clazz = Class.forName("com.example.Foo");
Method method = clazz.getDeclaredMethod("bar", String.class);
method.setAccessible(true);
method.invoke(instance, "arg");
```

**性能问题**：反射有额外开销（安全检查、方法查找），生产中频繁调用应缓存 Method 对象。

### 动态代理

**JDK 动态代理**：
- 要求目标类必须实现接口
- 通过 `Proxy.newProxyInstance()` + `InvocationHandler` 实现
- 底层生成 `$Proxy0` 类继承 `Proxy`、实现目标接口

**CGLIB 动态代理**：
- 不要求接口，通过继承目标类实现
- 底层使用 ASM 字节码操作
- 无法代理 `final` 类或 `final` 方法

**Spring AOP 选择规则**：有接口用 JDK，无接口用 CGLIB（Spring Boot 2.x 后默认 CGLIB）

---

## 异常体系

```
Throwable
├── Error（JVM 级别，不可恢复：OOM、StackOverflow）
└── Exception
    ├── RuntimeException（非受检异常：NPE、IndexOutOfBounds、ClassCast）
    └── 受检异常（必须 try-catch 或 throws：IOException、SQLException）
```

**高频追问**：
- `finally` 一定会执行吗？→ 除非 `System.exit()` 或 JVM 崩溃
- `try-with-resources` 原理？→ 编译器生成 `finally` 块调用 `close()`，实现 `AutoCloseable`

---

## 泛型

**类型擦除**：泛型仅在编译期有效，运行期擦除为原始类型（Object 或上界）

```java
List<String> list = new ArrayList<>();
// 运行时：list 实际是 List，无法通过反射获取到 String
```

**通配符**：
- `? extends T`：上界通配符，只读（协变）
- `? super T`：下界通配符，只写（逆变）
- `PECS 原则`：Producer Extends, Consumer Super

---

## IO/NIO

| 对比 | BIO | NIO | AIO |
|------|-----|-----|-----|
| 模型 | 同步阻塞 | 同步非阻塞 | 异步非阻塞 |
| 核心 | Stream | Channel + Buffer + Selector | AsynchronousChannel |
| 适用 | 连接少 | 连接多、短连接 | 连接多、长连接 |

**NIO 三大核心**：
- **Channel**：双向数据通道（FileChannel、SocketChannel）
- **Buffer**：数据容器，position/limit/capacity 三指针
- **Selector**：多路复用器，单线程监控多 Channel

**Netty** 基于 NIO，解决了 Java 原生 NIO 的 epoll bug 和编程复杂性问题。

---

## Java 8+ 新特性

### Lambda & 函数式接口

```java
// 四大核心函数式接口
Function<T, R>       // T → R，有入有出
Consumer<T>          // T → void，有入无出
Supplier<T>          // () → T，无入有出
Predicate<T>         // T → boolean，判断
```

### Stream API

```java
list.stream()
    .filter(x -> x > 0)       // 中间操作（惰性）
    .map(String::valueOf)      // 中间操作
    .collect(Collectors.toList()); // 终止操作（触发计算）
```

**注意**：Stream 只能消费一次；`parallel()` 并行流注意线程安全。

### Optional

```java
Optional.ofNullable(user)
    .map(User::getName)
    .orElse("默认值");
```

### CompletableFuture（高频）

```java
CompletableFuture.supplyAsync(() -> fetchData())
    .thenApply(data -> process(data))
    .thenAccept(result -> save(result))
    .exceptionally(e -> { log(e); return null; });
```

**高频追问**：thenApply vs thenCompose 的区别？→ thenApply 映射值，thenCompose 用于返回 CompletableFuture 的函数（flatMap）
