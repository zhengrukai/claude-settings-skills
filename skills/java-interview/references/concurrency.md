# Java 并发 - 面试知识点参考

## 目录
1. [线程基础](#线程基础)
2. [synchronized](#synchronized)
3. [volatile](#volatile)
4. [AQS 与锁](#aqs-与锁)
5. [线程池](#线程池)
6. [并发工具类](#并发工具类)
7. [CAS 与原子类](#cas-与原子类)

---

## 线程基础

### 线程状态流转

```
NEW → RUNNABLE ⇄ BLOCKED/WAITING/TIMED_WAITING → TERMINATED
```

- **BLOCKED**：等待 synchronized 锁
- **WAITING**：`Object.wait()` / `Thread.join()` / `LockSupport.park()`
- **TIMED_WAITING**：上述方法的超时版本

**sleep vs wait**：
| 对比 | sleep | wait |
|------|-------|------|
| 所属类 | Thread | Object |
| 是否释放锁 | 否 | 是 |
| 唤醒方式 | 超时自动 | notify/超时 |
| 使用场景 | 任意 | synchronized 块内 |

---

## synchronized

### 底层原理（高频⭐⭐⭐⭐⭐）

**对象头（Mark Word）**存储锁状态：
```
无锁 → 偏向锁 → 轻量级锁 → 重量级锁（单向升级，不可降级）
```

| 锁状态 | 适用场景 | 原理 |
|--------|---------|------|
| 偏向锁 | 只有一个线程访问 | Mark Word 记录线程 ID，免 CAS |
| 轻量级锁 | 多线程交替（无真正竞争）| CAS 将 Mark Word 指向栈中 Lock Record |
| 重量级锁 | 多线程真实竞争 | 操作系统互斥量（mutex），线程进入内核态阻塞 |

**字节码层面**：
- 同步方法：`ACC_SYNCHRONIZED` 标志
- 同步块：`monitorenter` / `monitorexit` 指令（monitorexit 出现两次，一次正常，一次异常）

**synchronized 与 ReentrantLock 对比**：
| 对比 | synchronized | ReentrantLock |
|------|-------------|---------------|
| 实现 | JVM 内置 | Java 层（AQS）|
| 可中断 | 不可中断 | `lockInterruptibly()` |
| 公平锁 | 不支持 | 支持 |
| 条件变量 | 一个（wait/notify）| 多个（Condition）|
| 锁超时 | 不支持 | `tryLock(timeout)` |
| 自动释放 | 是（代码块结束）| 需手动 `unlock()`（finally）|

---

## volatile

### 两大作用（高频⭐⭐⭐⭐⭐）

**1. 可见性**：写操作立即刷新到主内存，读操作从主内存读取（通过内存屏障实现）

**2. 禁止指令重排序**：通过插入内存屏障（Memory Barrier）

**无法保证原子性**：
```java
volatile int count = 0;
count++; // 非原子：read → increment → write，多线程下不安全
```

### DCL 单例（高频）

```java
public class Singleton {
    // volatile 禁止初始化时的指令重排
    private static volatile Singleton instance;
    
    public static Singleton getInstance() {
        if (instance == null) {                    // 第一次检查（无锁）
            synchronized (Singleton.class) {
                if (instance == null) {             // 第二次检查（有锁）
                    instance = new Singleton();     // 1.分配内存 2.初始化 3.赋值
                }
            }
        }
        return instance;
    }
}
```

**为什么需要 volatile**：`new Singleton()` 分三步，JIT 可能重排为 1→3→2，导致另一线程拿到未初始化的对象。

---

## AQS 与锁

### AQS（AbstractQueuedSynchronizer）（高频⭐⭐⭐⭐）

**核心设计**：
- `volatile int state`：同步状态（ReentrantLock 中表示重入次数）
- CLH 双向队列：存放等待线程节点
- `tryAcquire` / `tryRelease`：子类实现具体语义

**独占锁获取流程**：
1. `tryAcquire()` 尝试获取锁
2. 失败 → 构造 Node 加入 CLH 队列尾部
3. 自旋 + `LockSupport.park()` 等待
4. 前驱节点释放后 → `LockSupport.unpark()` 唤醒

**基于 AQS 实现的工具**：
- `ReentrantLock`（独占）
- `ReentrantReadWriteLock`（共享+独占）
- `Semaphore`（共享）
- `CountDownLatch`（共享）

### ReentrantLock 公平锁 vs 非公平锁

- **非公平**（默认）：新来的线程先 CAS 抢锁，失败才排队
- **公平**：严格按队列顺序，新线程直接排队

**为什么默认非公平**：吞吐量更高（减少线程切换），但可能导致饥饿

---

## 线程池

### 核心参数（高频⭐⭐⭐⭐⭐）

```java
new ThreadPoolExecutor(
    int corePoolSize,        // 核心线程数（长期存活）
    int maximumPoolSize,     // 最大线程数
    long keepAliveTime,      // 非核心线程空闲存活时间
    TimeUnit unit,           // 时间单位
    BlockingQueue workQueue, // 任务队列
    ThreadFactory threadFactory,     // 线程工厂
    RejectedExecutionHandler handler // 拒绝策略
);
```

### 任务执行流程

```
新任务提交
↓
核心线程数未满 → 创建核心线程执行
↓
核心线程满 → 任务入队列
↓
队列满 → 创建非核心线程执行（总数 ≤ maxPoolSize）
↓
maxPoolSize 也满 → 执行拒绝策略
```

### 四种拒绝策略

| 策略 | 行为 |
|------|------|
| AbortPolicy（默认）| 抛出 RejectedExecutionException |
| CallerRunsPolicy | 由提交任务的线程自己执行 |
| DiscardPolicy | 静默丢弃新任务 |
| DiscardOldestPolicy | 丢弃队列最老任务，重新提交 |

### 常用阻塞队列

| 队列 | 特点 | 适用 |
|------|------|------|
| LinkedBlockingQueue | 无界（Integer.MAX_VALUE）| FixedThreadPool（慎用！）|
| SynchronousQueue | 不存储，直接移交 | CachedThreadPool |
| ArrayBlockingQueue | 有界 | 推荐生产使用 |
| PriorityBlockingQueue | 优先级排序 | 有优先级需求 |

### 线程池参数设置参考

- **CPU 密集型**：核心数 = CPU 核数 + 1
- **IO 密集型**：核心数 = CPU 核数 × 2（或根据等待时间/执行时间比计算）
- **生产建议**：根据业务实测调整，避免使用 Executors 工厂方法（隐藏风险）

**Executors 问题**：
- `newFixedThreadPool` / `newSingleThreadExecutor`：队列无界，OOM
- `newCachedThreadPool`：线程数无上限，OOM

---

## 并发工具类

### CountDownLatch vs CyclicBarrier

| 对比 | CountDownLatch | CyclicBarrier |
|------|---------------|---------------|
| 计数方向 | 倒数到 0 | 累加到 N |
| 可重用 | 不可（一次性）| 可重用（reset）|
| 等待方 | 一个或多个主线程等待 | 所有参与线程互相等待 |
| 场景 | 主线程等待子任务完成 | 多线程到达同一起跑线 |

### Semaphore（信号量）

```java
Semaphore sem = new Semaphore(3); // 最多 3 个并发
sem.acquire();  // 获取许可（减 1）
// ... 执行
sem.release();  // 释放许可（加 1）
```

用于限流、连接池控制。

---

## CAS 与原子类

### CAS（Compare And Swap）

```
CAS(内存地址, 期望值, 新值)：
  if (内存值 == 期望值) → 更新为新值，返回 true
  else → 不更新，返回 false
```

**ABA 问题**：值从 A 变 B 再变 A，CAS 认为没变化
**解决**：`AtomicStampedReference`（带版本号）

**原子类家族**：
- `AtomicInteger` / `AtomicLong` / `AtomicBoolean`
- `AtomicReference` / `AtomicStampedReference`
- `LongAdder`（高并发计数，分段累加，比 AtomicLong 性能好）

### ThreadLocal

**原理**：每个 Thread 内部有一个 `ThreadLocalMap`，key 为 ThreadLocal（弱引用），value 为存储值

**内存泄漏风险**：ThreadLocal 被回收（key 变 null）但 value 不会回收，需手动 `remove()`

**典型使用**：数据库连接、用户登录信息传递（Spring 的 `RequestContextHolder`）

**InheritableThreadLocal**：子线程可继承父线程的 ThreadLocal 值（线程池场景下需注意失效问题，可用 TransmittableThreadLocal 解决）
