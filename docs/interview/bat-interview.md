


# BAT 面试题汇总及详解：进大厂必看


## 目录

- [零、面试开场与项目介绍](#零面试开场与项目介绍)
- [一、Java 多线程与并发](#一java-多线程与并发)
- [二、JVM 相关](#二jvm-相关)
- [三、Java 基础与集合](#三java-基础与集合)
- [四、Spring 生态](#四spring-生态)
- [五、中间件（Dubbo / MQ / Spring Cloud）](#五中间件dubbo--mq--spring-cloud)
- [六、数据库与 SQL](#六数据库与-sql)
- [七、Redis](#七redis)
- [八、分布式与高并发设计](#八分布式与高并发设计)
- [九、算法与数据结构](#九算法与数据结构)
- [十、网络与操作系统](#十网络与操作系统)
- [附录：面试技巧](#附录面试技巧)


## 零、面试开场与项目介绍

大厂面试通常以「自我介绍 + 项目深挖」开场，技术题反而是第二幕。

### 0.1 自我介绍要点

- 工作经历与职责：说清**你负责什么**，而不是团队做了什么
- 突出 1~2 个有技术亮点的项目：架构、性能优化、疑难问题排查
- 控制时间在 2~3 分钟，为后续追问留空间

### 0.2 项目介绍怎么讲

面试官想判断：

1. 你对做过的事是否理解透彻（不是背简历）
2. 项目复杂度是否匹配岗位级别
3. 你在团队中的真实贡献

**推荐结构**：背景 → 你的职责 → 技术方案 → 难点与解决 → 量化结果（QPS、延迟、成本等）


## 一、Java 多线程与并发

### 1.1 线程池的原理，为什么要创建线程池？

#### 为什么要用线程池？

| 问题                    | 说明                         |
| ----------------------- | ---------------------------- |
| 频繁创建/销毁线程开销大 | 涉及内核态切换、栈内存分配   |
| 无限制创建线程          | 可能导致 OOM 或系统卡死      |
| 难以统一管理            | 队列、拒绝策略、监控都不方便 |

#### 工作原理

`ThreadPoolExecutor` 的核心流程：

```
提交任务
  → 核心线程数未满？创建核心线程执行
  → 队列未满？入队等待
  → 线程数 < maximumPoolSize？创建非核心线程执行
  → 触发拒绝策略
```

#### 创建方式

```java
// 推荐：显式配置 ThreadPoolExecutor
ExecutorService pool = new ThreadPoolExecutor(
    4, 8, 60L, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(100),
    new ThreadFactoryBuilder().setNameFormat("biz-%d").build(),
    new ThreadPoolExecutor.CallerRunsPolicy()
);

// JDK 提供的工厂方法（生产环境慎用 Executors）
Executors.newFixedThreadPool(10);   // 无界队列，可能 OOM
Executors.newCachedThreadPool();    // 线程数无上限
Executors.newSingleThreadExecutor();
```

> **面试加分**：说明为什么不推荐 `Executors`——无界队列/无界线程数在生产环境有 OOM 风险。

**Java 线程池类继承体系**（`ThreadPoolExecutor`、`ScheduledThreadPoolExecutor`、`ForkJoinPool` 等）：

![Java Executor 类继承体系](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/bat-interview/bat-interview-01-thread-pool-executor.jpeg)


### 1.2 线程的生命周期

Java 线程六种状态（`Thread.State`）：

| 状态          | 说明                              |
| ------------- | --------------------------------- |
| NEW           | 新建，未 start                    |
| RUNNABLE      | 可运行（含就绪和运行中）          |
| BLOCKED       | 等待获取 synchronized 锁          |
| WAITING       | 无限期等待（wait/join/park）      |
| TIMED_WAITING | 有限期等待（sleep/wait(timeout)） |
| TERMINATED    | 终止                              |

![Java 线程生命周期状态转换图](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/bat-interview/bat-interview-02-thread-lifecycle.jpeg)

**僵死进程（僵尸进程）**：子进程已退出，但父进程未调用 `wait()` 回收，子进程成为僵尸进程，占用 PID。Java 层面类比：线程池线程不销毁 + ThreadLocal 未清理 → 内存泄漏。


### 1.3 什么是线程安全？如何实现？

**线程安全**：多线程访问共享数据时，程序行为与单线程一致，不出现数据错乱。

**实现方式**：

1. **互斥同步**：`synchronized`、`ReentrantLock`、读写锁
2. **非阻塞同步**：CAS（`AtomicInteger`、`LongAdder`）
3. **无同步方案**：
   - 可重入代码（纯局部变量，无共享状态）
   - `ThreadLocal`（线程隔离副本）
   - 不可变对象（`final` + 安全发布）


### 1.4 线程池核心参数与合理配置

```java
public ThreadPoolExecutor(
    int corePoolSize,              // 核心线程数
    int maximumPoolSize,           // 最大线程数
    long keepAliveTime,            // 非核心线程空闲存活时间
    TimeUnit unit,
    BlockingQueue<Runnable> workQueue,  // 任务队列
    ThreadFactory threadFactory,        // 线程工厂
    RejectedExecutionHandler handler    // 拒绝策略
);
```

**拒绝策略**：

| 策略                | 行为                            |
| ------------------- | ------------------------------- |
| AbortPolicy（默认） | 抛 `RejectedExecutionException` |
| CallerRunsPolicy    | 调用者线程执行                  |
| DiscardPolicy       | 静默丢弃                        |
| DiscardOldestPolicy | 丢弃队列最老任务                |

**线程池大小估算**：

- **CPU 密集型**：`核心数 + 1`
- **IO 密集型**：`核心数 × 2`，或 `(等待时间/CPU时间 + 1) × 核心数`
- **混合型**：拆分线程池，分别处理


### 1.5 volatile 与 ThreadLocal

#### volatile

- **作用**：保证可见性 + 禁止指令重排序（不保证原子性）
- **原理**：写操作加内存屏障，强制刷回主内存；读操作从主内存读
- **场景**：状态标志、DCL 单例中的 `instance`、独立观察（定期刷新配置）

![volatile 关键字与 Java 内存模型（JMM）](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/bat-interview/bat-interview-03-volatile-jmm.jpeg)

#### ThreadLocal

- **原理**：每个 `Thread` 持有 `ThreadLocalMap`，以 `ThreadLocal` 为 key 存线程私有副本
- **场景**：数据库连接、用户 Session、`SimpleDateFormat` 线程隔离
- **内存泄漏**：`ThreadLocal` 的 key 是弱引用，value 是强引用；线程池线程不销毁时，value 无法回收。**解决**：`finally` 中调用 `remove()`

![ThreadLocal 底层数据结构](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/bat-interview/bat-interview-04-threadlocal-structure.jpeg)

#### ThreadLocal 什么时候会出现 OOM？

`ThreadLocal` 变量维护在 `Thread` 内部的 `ThreadLocalMap` 中。线程不退出，引用就一直存在。使用线程池时，若线程不销毁，`ThreadLocal` 未 `remove()` 会导致 value 无法回收，最终 OOM。

![ThreadLocal 内存泄漏原理](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/bat-interview/bat-interview-05-threadlocal-oom.png)


### 1.6 synchronized 与 volatile 区别

| 对比项     | volatile   | synchronized     |
| ---------- | ---------- | ---------------- |
| 原子性     | 不保证     | 保证             |
| 可见性     | 保证       | 保证             |
| 有序性     | 禁止重排序 | 保证（内存屏障） |
| 阻塞       | 不阻塞     | 可能阻塞         |
| 粒度       | 变量       | 变量/方法/类     |
| 编译器优化 | 禁止       | 允许             |

**synchronized 锁升级**（JDK 6+）：无锁 → 偏向锁 → 轻量级锁 → 重量级锁


### 1.7 死锁：条件与模拟

**死锁四个必要条件**（缺一不可）：

1. 互斥
2. 占有且等待
3. 不可抢占
4. 循环等待

```java
// 经典死锁示例
Object lockA = new Object();
Object lockB = new Object();

new Thread(() -> {
    synchronized (lockA) {
        Thread.sleep(100);
        synchronized (lockB) { /* ... */ }
    }
}).start();

new Thread(() -> {
    synchronized (lockB) {
        Thread.sleep(100);
        synchronized (lockA) { /* ... */ }
    }
}).start();
```

**排查**：`jstack` 查看 `Found one Java-level deadlock`


### 1.8 并发与并行的区别

- **并发（Concurrent）**：同一时间段内交替执行多个任务（单核也可）
- **并行（Parallel）**：同一时刻多个任务真正同时执行（多核）


### 1.9 常用并发工具类

| 类               | 作用                           |
| ---------------- | ------------------------------ |
| `CountDownLatch` | 一个或多个线程等待其他线程完成 |
| `CyclicBarrier`  | 一组线程互相等待到屏障点       |
| `Semaphore`      | 控制并发访问数量               |
| `Phaser`         | 可动态调整参与方数量的屏障     |

**BlockingQueue 方法区别**：

- `put` / `take`：阻塞
- `offer` / `poll`：非阻塞或限时阻塞


## 二、JVM 相关

### 2.1 JVM 内存模型

```
┌─────────────────────────────────────────┐
│                  堆 Heap                 │
│  ┌─────────────┐  ┌──────────────────┐  │
│  │  新生代      │  │     老年代        │  │
│  │ Eden|S0|S1  │  │                  │  │
│  └─────────────┘  └──────────────────┘  │
├─────────────────────────────────────────┤
│  元空间 Metaspace（JDK8+，本地内存）      │
├─────────────────────────────────────────┤
│  线程私有：虚拟机栈 | 本地方法栈 | PC寄存器  │
└─────────────────────────────────────────┘
```

**版本演进**：

| JDK 版本   | 永久代               | 运行时常量池 |
| ---------- | -------------------- | ------------ |
| 1.6 及之前 | 有                   | 方法区       |
| 1.7        | 逐步去除             | 移至堆       |
| 1.8+       | 移除，改用 Metaspace | Metaspace    |

![JVM 运行时数据区域（堆、栈、元空间等）](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/bat-interview/bat-interview-06-jvm-memory-model.jpeg)

**为什么用 Metaspace 替代 PermGen？**

- PermGen 大小固定，容易 `OutOfMemoryError: PermGen space`
- Metaspace 使用本地内存，可动态扩展
- 简化 Full GC，类元数据回收更高效


### 2.2 GC 机制

![JVM 分代垃圾回收模型（新生代 / 老年代）](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/bat-interview/bat-interview-07-jvm-gc-generational.jpeg)

#### Minor GC / Major GC / Full GC

| 类型     | 范围            | 触发条件                                      |
| -------- | --------------- | --------------------------------------------- |
| Minor GC | 新生代          | Eden 区满                                     |
| Major GC | 老年代          | 老年代空间不足                                |
| Full GC  | 整个堆 + 元空间 | `System.gc()`、晋升失败、CMS 失败、元空间满等 |

#### 垃圾收集算法

| 算法      | 原理                            | 优缺点               |
| --------- | ------------------------------- | -------------------- |
| 标记-清除 | 标记存活对象，清除未标记        | 产生碎片             |
| 复制      | 分两块，存活对象复制到另一块    | 浪费空间，适合新生代 |
| 标记-整理 | 标记后移动存活对象，消除碎片    | 适合老年代           |
| 分代收集  | 新生代用复制，老年代用标记-整理 | 综合方案             |

#### 常见垃圾收集器

- **Serial / Serial Old**：单线程，适合客户端
- **ParNew**：Serial 的多线程版，配合 CMS
- **Parallel Scavenge / Parallel Old**：吞吐量优先
- **CMS**：低延迟，标记-清除，有碎片
- **G1**：分区收集，可预测停顿（JDK 9 默认）
- **ZGC / Shenandoah**：超低延迟（JDK 11+ / 12+）

![常见垃圾收集器对比与组合](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/bat-interview/bat-interview-10-gc-collectors.png)

#### Class 文件结构

![Java Class 文件二进制结构](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/bat-interview/bat-interview-09-classfile-structure.png)

#### GC Roots 有哪些？

- 虚拟机栈中引用的对象
- 方法区中类静态属性引用的对象
- 方法区中常量引用的对象
- 本地方法栈中 JNI 引用的对象
- 活跃线程、同步锁持有的对象等

![GC Roots 可达性分析示意](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/bat-interview/bat-interview-24-gc-roots.png)


### 2.3 类加载器与双亲委派

#### 四种类加载器

| 加载器                  | 加载范围                      |
| ----------------------- | ----------------------------- |
| Bootstrap ClassLoader   | `lib` 下核心类库（rt.jar 等） |
| Extension ClassLoader   | `lib/ext` 扩展类库            |
| Application ClassLoader | CLASSPATH 用户类              |
| 自定义 ClassLoader      | 用户指定路径                  |

#### 双亲委派流程

```
收到加载请求 → 委派给父加载器 → 父加载器再向上委派
→ 顶层无法加载 → 子加载器自己尝试加载
```

**好处**：

1. **沙箱安全**：防止核心 API 被篡改（如自定义 `java.lang.String`）
2. **避免重复加载**

![双亲委派机制工作流程](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/bat-interview/bat-interview-08-classloader-delegation.jpeg)

#### 什么情况下破坏双亲委派？

| 场景                            | 原因                                                        |
| ------------------------------- | ----------------------------------------------------------- |
| **SPI 机制**（JDBC、JNDI）      | 核心类需要加载实现类，由线程上下文类加载器打破委派          |
| **OSGi / 模块化**               | 不同 Bundle 需要独立类加载                                  |
| **热部署**（Tomcat）            | 每个 Web 应用需要独立 ClassLoader，优先加载 WEB-INF/classes |
| **字节码增强**（Spring、CGLIB） | 需要加载动态生成的类                                        |

Tomcat 类加载器层次：`Bootstrap → System → Common → Webapp`，Webapp 优先加载自己的类。


### 2.4 JVM 调优

#### 常用工具

- **命令行**：`jps`、`jstat`、`jmap`、`jstack`、`jinfo`
- **可视化**：VisualVM、JProfiler、Arthas、MAT

#### 核心 JVM 参数

```bash
# 堆内存
-Xms4g -Xmx4g                    # 初始/最大堆，建议设相同避免动态扩容
-Xmn2g                           # 新生代大小
-XX:NewRatio=2                   # 老年代:新生代 = 2:1
-XX:SurvivorRatio=8              # Eden:S0:S1 = 8:1:1

# 元空间
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=512m

# 垃圾收集器（G1 示例）
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200          # 目标最大停顿时间
-XX:G1HeapRegionSize=16m

# GC 日志（JDK 9+）
-Xlog:gc*:file=gc.log:time,uptime,level,tags

# 栈与直接内存
-Xss256k
-XX:MaxDirectMemorySize=512m

# OOP 压缩（64 位 JVM 默认开启）
-XX:+UseCompressedOops
```

#### 调优思路

1. **定位问题**：GC 日志、堆 Dump、线程 Dump
2. **分析瓶颈**：频繁 Full GC？老年代增长快？元空间泄漏？
3. **针对性调整**：增大堆、换收集器、修复内存泄漏代码
4. **压测验证**：对比调优前后吞吐量和 P99 延迟

#### 内存泄漏常见场景

- 静态集合持有对象引用
- 未关闭的连接/流
- `ThreadLocal` 未 `remove()`
- 监听器/回调未注销
- 内部类持有外部类引用


### 2.5 堆与栈的区别

| 对比     | 栈                         | 堆               |
| -------- | -------------------------- | ---------------- |
| 存储内容 | 局部变量、引用、方法调用栈 | 对象实例、数组   |
| 线程     | 线程私有                   | 线程共享         |
| 生命周期 | 方法结束即释放             | GC 回收          |
| 异常     | StackOverflowError         | OutOfMemoryError |
| 速度     | 快（连续内存）             | 相对慢           |

**参数传递**：Java 只有值传递。基本类型传值，对象传引用的副本（引用的值传递）。


### 2.6 对象引用类型

| 类型   | 回收时机                     | 场景                           |
| ------ | ---------------------------- | ------------------------------ |
| 强引用 | 永不回收（除非 unreachable） | 普通 `new`                     |
| 软引用 | 内存不足时回收               | 缓存                           |
| 弱引用 | GC 时回收                    | `WeakHashMap`、ThreadLocal key |
| 虚引用 | 随时可能回收，用于跟踪回收   | 堆外内存管理                   |


## 三、Java 基础与集合

### 3.1 红黑树

自平衡二叉查找树，五条性质：

1. 节点非红即黑
2. 根节点为黑
3. 叶子节点（NIL）为黑
4. 红节点的子节点必须为黑
5. 任意节点到叶子的路径上黑节点数相同

**应用**：`TreeMap`/`TreeSet`、Linux CFS 调度、epoll、Nginx 定时器、HashMap 链表过长转红黑树


### 3.2 NIO

**核心组件**：`Channel`、`Buffer`、`Selector`

**特性**：I/O 多路复用 + 非阻塞 I/O

**适用场景**：

- 大量长连接、低频率通信（网关、推送服务）
- 高并发网络服务（Netty、ZooKeeper）

**与 BIO 对比**：

|          | BIO                          | NIO                |
| -------- | ---------------------------- | ------------------ |
| 模式     | 阻塞                         | 非阻塞             |
| 线程模型 | 一连接一线程                 | 少线程多连接       |
| API      | `InputStream`/`OutputStream` | `Channel`/`Buffer` |


### 3.3 Java 9 主要新特性

- **模块化**（JPMS）：`module-info.java`
- **JShell**：交互式 REPL
- **不可变集合工厂**：`List.of()`、`Set.of()`、`Map.of()`
- **HTTP Client**（`java.net.http`）：支持 HTTP/2
- **G1 成为默认 GC**
- **接口私有方法**
- **Try-with-resources 增强**


### 3.4 HashMap 底层实现

**JDK 8 结构**：数组 + 链表 + 红黑树

```
hash(key) → 数组下标 → 链表（长度≥8 且数组≥64 转红黑树）
```

**关键参数**：

- 默认容量 16，负载因子 0.75
- 扩容为 2 倍，rehash
- 树化阈值 8，退化阈值 6

**put 流程简述**：

1. 计算 hash，定位桶
2. 桶为空 → 直接放入
3. 桶非空 → 链表/红黑树查找，key 相同则覆盖，否则插入
4. 链表长度 ≥ 8 → 树化
5. 元素数 > 容量 × 0.75 → 扩容

![HashMap JDK8 底层数据结构（数组 + 链表 + 红黑树）](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/bat-interview/bat-interview-11-hashmap-structure.jpeg)


### 3.5 HashMap vs Hashtable vs ConcurrentHashMap

| 对比项    | HashMap          | Hashtable               | ConcurrentHashMap         |
| --------- | ---------------- | ----------------------- | ------------------------- |
| 线程安全  | 否               | 是（synchronized 方法） | 是                        |
| null      | 允许             | 不允许                  | 不允许                    |
| 性能      | 高               | 低（全表锁）            | 高                        |
| JDK8 实现 | 数组+链表+红黑树 | 同左                    | CAS + synchronized 锁桶头 |

**ConcurrentHashMap JDK7 vs JDK8**：

- **JDK7**：Segment 分段锁（16 个 Segment）
- **JDK8**：取消 Segment，CAS + synchronized 锁单个 Node，粒度更细

**HashMap 并发死循环**（JDK7）：多线程扩容时链表可能形成环。JDK8 改为尾插法解决。


### 3.6 反射

**用途**：框架（Spring IOC）、动态代理、注解处理、序列化

**获取 Class 三种方式**：

```java
Class<?> c1 = obj.getClass();
Class<?> c2 = Foo.class;
Class<?> c3 = Class.forName("com.example.Foo");
```

**缺点**：性能低（无法 JIT 优化）、破坏封装、安全问题

**建议**：框架初始化阶段可用，热点路径避免


### 3.7 自定义注解

**典型场景**：权限校验、日志、限流、数据加解密

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RequirePermission {
    String value();
}

// 配合 AOP
@Around("@annotation(requirePermission)")
public Object check(ProceedingJoinPoint pjp, RequirePermission requirePermission) {
    // 鉴权逻辑
    return pjp.proceed();
}
```


### 3.8 List / Set / Map 区别

| 接口 | 有序           | 重复       | null                 |
| ---- | -------------- | ---------- | -------------------- |
| List | 是（插入顺序） | 允许       | 允许多个             |
| Set  | 视实现而定     | 不允许     | 通常一个             |
| Map  | 视实现而定     | key 不重复 | key 一个，value 多个 |

**ArrayList vs LinkedList vs Vector**：

|          | ArrayList | LinkedList | Vector     |
| -------- | --------- | ---------- | ---------- |
| 底层     | 动态数组  | 双向链表   | 动态数组   |
| 随机访问 | O(1)      | O(n)       | O(1)       |
| 头尾插入 | 尾快      | 头尾快     | 尾快       |
| 线程安全 | 否        | 否         | 是（过时） |
| 扩容     | 1.5 倍    | 无         | 2 倍       |


## 四、Spring 生态

### 4.1 Spring AOP

**场景**：事务、日志、权限、缓存、异常处理

**实现方式**：

| 方式         | 时机          | 特点                        |
| ------------ | ------------- | --------------------------- |
| JDK 动态代理 | 运行时        | 基于接口，JDK 原生          |
| CGLIB        | 运行时        | 基于子类，不能代理 final 类 |
| AspectJ      | 编译时/加载时 | 功能最强，需特殊编译        |

**Spring AOP 默认**：有接口用 JDK 代理，无接口用 CGLIB

**执行流程**：

```
调用方法 → 代理拦截 → 前置通知 → 目标方法 → 后置/返回/异常通知
```

![Spring AOP 实现原理（JDK 动态代理 vs CGLIB）](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/bat-interview/bat-interview-12-spring-aop.jpeg)


### 4.2 Spring Bean 作用域与生命周期

#### 作用域

| 作用域            | 说明                          |
| ----------------- | ----------------------------- |
| singleton（默认） | 单例，IoC 容器内唯一          |
| prototype         | 每次获取创建新实例            |
| request           | 每个 HTTP 请求一个（Web）     |
| session           | 每个 HTTP Session 一个（Web） |
| application       | ServletContext 级别（Web）    |

#### 生命周期

```
实例化（构造器）
  → 属性赋值（@Autowired）
  → Aware 接口回调（BeanNameAware 等）
  → BeanPostProcessor.postProcessBeforeInitialization
  → @PostConstruct / InitializingBean.afterPropertiesSet
  → BeanPostProcessor.postProcessAfterInitialization
  → Bean 可用
  → 容器关闭
  → @PreDestroy / DisposableBean.destroy
  → 销毁
```

![Spring Bean 作用域与生命周期](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/bat-interview/bat-interview-13-spring-bean-lifecycle.jpeg)


### 4.3 Spring Boot 相比 Spring 的改进

- **自动配置**：`@EnableAutoConfiguration` + `spring.factories`
- **起步依赖**：简化 Maven 依赖管理
- **内嵌容器**：Tomcat/Jetty/Undertow，jar 包直接运行
- **无 XML 配置**：约定优于配置
- **Actuator**：健康检查、指标监控
- **外部化配置**：`application.yml` 统一配置

#### 自定义 Starter 步骤

1. 创建 `xxx-spring-boot-autoconfigure` 模块
2. 编写 `@Configuration` + `@ConditionalOnXxx` 自动配置类
3. 在 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 注册
4. 创建 `xxx-spring-boot-starter` 聚合依赖


### 4.4 Spring IOC

**控制反转**：对象的创建和依赖关系由容器管理，而非代码 `new`

**优点**：

- 降低耦合
- 便于单元测试（Mock）
- 统一管理对象生命周期
- 支持 AOP 等横切能力

**初始化过程**：

1. 加载配置（XML/注解/JavaConfig）
2. 解析 BeanDefinition
3. 注册到 BeanFactory
4. 实例化 → 依赖注入 → 初始化
5. 放入单例池

![Spring IOC 容器初始化与依赖注入原理](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/bat-interview/bat-interview-14-spring-ioc.jpeg)


### 4.5 Spring 事务

#### ACID

- **原子性**：全成功或全回滚
- **一致性**：数据完整性约束
- **隔离性**：并发事务互不干扰
- **持久性**：提交后永久保存

#### 隔离级别

| 级别                          | 脏读   | 不可重复读 | 幻读   |
| ----------------------------- | ------ | ---------- | ------ |
| READ UNCOMMITTED              | 可能   | 可能       | 可能   |
| READ COMMITTED                | 不可能 | 可能       | 可能   |
| REPEATABLE READ（MySQL 默认） | 不可能 | 不可能     | 可能*  |
| SERIALIZABLE                  | 不可能 | 不可能     | 不可能 |

*InnoDB 的 RR 通过 MVCC + 间隙锁在很大程度上避免幻读

#### 传播行为（常考）

| 传播行为         | 说明                           |
| ---------------- | ------------------------------ |
| REQUIRED（默认） | 有事务加入，无则新建           |
| REQUIRES_NEW     | 挂起当前，新建独立事务         |
| NESTED           | 嵌套事务，外层回滚则内层也回滚 |
| SUPPORTS         | 有则加入，无则非事务执行       |
| NOT_SUPPORTED    | 挂起当前，非事务执行           |

![Spring MVC / 事务 / AOP 整体架构](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/bat-interview/bat-interview-15-spring-mvc-transaction.png)

![Spring 事务实现方式（编程式 vs 声明式）](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/bat-interview/bat-interview-21-spring-transaction.jpeg)


### 4.6 Spring 常用设计模式

| 模式       | 应用                       |
| ---------- | -------------------------- |
| 工厂模式   | BeanFactory                |
| 单例模式   | Bean 默认单例              |
| 代理模式   | AOP                        |
| 模板方法   | JdbcTemplate、RestTemplate |
| 观察者模式 | ApplicationEvent           |
| 适配器模式 | HandlerAdapter             |
| 装饰器模式 | BeanWrapper                |


## 五、中间件（Dubbo / MQ / Spring Cloud）

### 5.1 Dubbo 一次调用链路

```
Consumer 代理
  → Cluster 容错（Failover 等）
  → LoadBalance 负载均衡
  → Filter 链（上下文、限流等）
  → Protocol 序列化（Hessian2 等）
  → Netty 网络传输
  → Provider 反序列化
  → Filter 链
  → 目标 Service 方法
```

![Dubbo 扩展点与一次完整调用链路](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/bat-interview/bat-interview-16-dubbo-architecture.jpeg)

### 5.2 Dubbo 负载均衡策略

| 策略           | 说明         | 适用             |
| -------------- | ------------ | ---------------- |
| Random         | 加权随机     | 默认，通用       |
| RoundRobin     | 加权轮询     | 注意慢提供者堆积 |
| LeastActive    | 最少活跃调用 | 慢服务隔离       |
| ConsistentHash | 一致性哈希   | 有状态服务       |

![Dubbo 一致性 Hash 负载均衡原理](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/bat-interview/bat-interview-17-dubbo-consistent-hash.jpeg)

### 5.3 Dubbo 并发限流

```xml
<!-- 服务端：限制并发执行数 -->
<dubbo:service interface="com.foo.BarService" executes="10" />

<!-- 客户端：限制并发请求数 -->
<dubbo:reference interface="com.foo.BarService" actives="10" />
```

### 5.4 Dubbo 配置方式

XML、注解、API、Properties 四种，推荐注解 + YAML

### 5.5 消息中间件对比

| 产品     | 优点                       | 缺点                   |
| -------- | -------------------------- | ---------------------- |
| Kafka    | 超高吞吐、持久化、生态好   | 延迟相对较高，运维复杂 |
| RocketMQ | 低延迟、事务消息、顺序消息 | 社区相对小             |
| RabbitMQ | 功能丰富、路由灵活、易用   | 吞吐不如 Kafka         |
| ActiveMQ | 老牌、JMS 标准             | 性能一般，逐渐式微     |

### 5.6 消息一致性保障

![消息中间件保证消息一致性的流程](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/bat-interview/bat-interview-18-mq-consistency.jpeg)

**本地消息表**：业务操作与消息写入同一本地事务 → 定时任务扫描发送 → 消费者幂等处理

**事务消息**（RocketMQ）：Half 消息 → 本地事务 → Commit/Rollback

**重试机制**：消费失败 → 延迟重试（阶梯退避）→ 死信队列

### 5.7 Spring Cloud 熔断（Hystrix / Sentinel）

- 监控服务调用失败率
- 达到阈值（如 5 秒 20 次失败）→ 熔断开路
- 降级返回默认值或友好提示
- 半开状态试探恢复

### 5.8 Spring Cloud vs Dubbo

| 对比     | Dubbo                | Spring Cloud             |
| -------- | -------------------- | ------------------------ |
| 定位     | RPC 框架             | 微服务一站式方案         |
| 通信     | 自定义协议（高性能） | HTTP/REST                |
| 注册中心 | ZooKeeper/Nacos      | Eureka/Nacos/Consul      |
| 适用     | 内部高性能 RPC       | 云原生、多语言、快速迭代 |

![Spring Cloud 与 Dubbo 对比](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/bat-interview/bat-interview-19-springcloud-vs-dubbo.jpeg)


## 六、数据库与 SQL

### 6.1 MySQL 锁机制

| 锁类型      | 说明                          |
| ----------- | ----------------------------- |
| 表锁        | 锁整张表，MyISAM 默认         |
| 行锁        | 锁单行，InnoDB 支持（需索引） |
| 共享锁（S） | 读锁，可多个事务同时持有      |
| 排他锁（X） | 写锁，独占                    |
| 意向锁      | 表级，表示事务即将锁行        |

### 6.2 乐观锁 vs 悲观锁

**乐观锁**：版本号/CAS，更新时 `WHERE version = ?`，适合读多写少

```sql
UPDATE account SET balance = balance - 100, version = version + 1
WHERE id = 1 AND version = 5;
```

**悲观锁**：`SELECT ... FOR UPDATE`，适合写多读少

### 6.3 事务隔离级别与并发问题

- **脏读**：读到未提交数据
- **不可重复读**：同一事务两次读结果不同（被其他事务修改）
- **幻读**：同一事务两次范围查询结果行数不同（被其他事务插入/删除）

### 6.4 Binlog 三种格式

| 格式      | 内容       | 优点     | 缺点                         |
| --------- | ---------- | -------- | ---------------------------- |
| STATEMENT | SQL 语句   | 日志量小 | 可能主从不一致（UDF、sleep） |
| ROW       | 行变更记录 | 精确     | 日志量大                     |
| MIXED     | 混合       | 折中     | 规则复杂                     |

### 6.5 分布式事务方案

| 方案         | 说明                           |
| ------------ | ------------------------------ |
| 2PC/XA       | 强一致，性能差，数据库支持     |
| TCC          | Try-Confirm-Cancel，业务侵入大 |
| 本地消息表   | 最终一致，实现简单             |
| Saga         | 长事务拆分 + 补偿              |
| 最大努力通知 | 允许丢失，重试通知             |

#### 2PC 两阶段提交

1. **准备阶段**：协调者询问所有参与者是否可以提交
2. **提交阶段**：全部同意则提交，任一拒绝则回滚

**问题**：同步阻塞、单点故障、数据不一致（网络分区）

![分布式事务两阶段提交（2PC）原理](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/bat-interview/bat-interview-20-distributed-tx-2pc.jpeg)

### 6.6 SQL 执行过程

![MySQL SQL 执行整体架构](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/bat-interview/bat-interview-22-mysql-sql-architecture.jpeg)

![SQL 解析与执行过程](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/bat-interview/bat-interview-23-sql-parse-execute.jpeg)

```
连接器（认证）→ 查询缓存（8.0 已移除）→ 分析器（词法/语法）
→ 优化器（执行计划）→ 执行器（调用存储引擎）→ 返回结果
```

### 6.7 JDBC 事务

```java
conn.setAutoCommit(false);
try {
    // 执行 SQL
    conn.commit();
} catch (Exception e) {
    conn.rollback();
}
```


## 七、Redis

### 7.1 为什么 Redis 快？

1. **纯内存操作**
2. **单线程**（6.0+ 多线程仅用于网络 I/O），避免上下文切换和锁竞争
3. **I/O 多路复用**（epoll），单线程处理多连接
4. **高效数据结构**（SDS、跳表、压缩列表等）
5. **简单协议**（RESP）

### 7.2 五种基本数据类型

| 类型   | 底层实现          | 场景                   |
| ------ | ----------------- | ---------------------- |
| String | SDS               | 缓存、计数器、分布式锁 |
| Hash   | 压缩列表/哈希表   | 对象存储               |
| List   | 压缩列表/双向链表 | 消息队列、时间线       |
| Set    | 整数集合/哈希表   | 标签、去重             |
| ZSet   | 跳表 + 哈希表     | 排行榜、延迟队列       |

### 7.3 跳跃表

Redis 跳表：多层索引的有序链表，查找/插入/删除平均 O(log N)

用于：ZSet 有序集合、集群节点内部结构

### 7.4 持久化

| 方式 | 原理           | 优点             | 缺点           |
| ---- | -------------- | ---------------- | -------------- |
| RDB  | 快照           | 恢复快、文件紧凑 | 可能丢数据     |
| AOF  | 写命令日志     | 数据安全         | 文件大、恢复慢 |
| 混合 | RDB + AOF 增量 | 兼顾             | 配置复杂       |

### 7.5 分布式锁（正确实现）

```java
// 加锁：SET key unique_value NX PX 30000
String result = jedis.set(lockKey, requestId, "NX", "PX", 30000);

// 解锁：Lua 脚本保证原子性（先判断 value 再删除）
String script = "if redis.call('get',KEYS[1]) == ARGV[1] then " +
                "return redis.call('del',KEYS[1]) else return 0 end";
```

**注意**：

- 必须设置过期时间，防死锁
- 解锁必须验证持有者，防误删
- 用 Lua 保证判断+删除原子性
- 生产环境推荐 Redisson（看门狗续期）

### 7.6 淘汰策略

- `volatile-lru` / `allkeys-lru`：LRU 淘汰
- `volatile-ttl`：优先淘汰即将过期
- `volatile-random` / `allkeys-random`：随机淘汰
- `noeviction`：不淘汰，写操作报错（默认）

### 7.3 集群原理

- 16384 个哈希槽，CRC16(key) % 16384
- 主从复制 + 故障转移
- 超过半数 Master 宕机 → 集群不可用


## 八、分布式与高并发设计

### 8.1 CAP 与 BASE

**CAP**（最多满足两个）：

- **C**onsistency：一致性
- **A**vailability：可用性
- **P**artition tolerance：分区容错

分布式系统必须容忍分区，实际在 CP（ZooKeeper）和 AP（Eureka）之间权衡。

**BASE**：

- Basically Available：基本可用
- Soft state：软状态
- Eventually consistent：最终一致

### 8.2 高并发优化手段

1. **缓存**：本地缓存（Caffeine）+ 分布式缓存（Redis），注意穿透/击穿/雪崩
2. **异步**：MQ 削峰填谷
3. **读写分离**：主写从读
4. **分库分表**：水平拆分
5. **CDN + 静态化**：减少动态请求
6. **限流降级熔断**：保护核心链路
7. **集群 + 负载均衡**：水平扩展

### 8.3 缓存三大问题

| 问题 | 解决方案                       |
| ---- | ------------------------------ |
| 穿透 | 布隆过滤器、缓存空值           |
| 击穿 | 互斥锁、逻辑过期               |
| 雪崩 | 过期时间随机化、多级缓存、熔断 |

### 8.4 Session 集群同步

- **粘性 Session**：负载均衡绑定节点（不推荐）
- **Session 复制**：Tomcat 集群同步（性能差）
- **集中存储**：Redis 存 Session（推荐）
- **无状态**：JWT Token（微服务主流）

### 8.5 限流算法

- **计数器**：简单，有临界突变
- **滑动窗口**：平滑
- **令牌桶**：允许突发流量
- **漏桶**：恒定速率


## 九、算法与数据结构

### 9.1 排序算法复杂度

| 算法 | 平均       | 最坏       | 空间     | 稳定 |
| ---- | ---------- | ---------- | -------- | ---- |
| 冒泡 | O(n²)      | O(n²)      | O(1)     | 是   |
| 选择 | O(n²)      | O(n²)      | O(1)     | 否   |
| 插入 | O(n²)      | O(n²)      | O(1)     | 是   |
| 快排 | O(n log n) | O(n²)      | O(log n) | 否   |
| 归并 | O(n log n) | O(n log n) | O(n)     | 是   |
| 堆排 | O(n log n) | O(n log n) | O(1)     | 否   |

> 快排不稳定原因：交换可能跳过相等元素，破坏相对顺序。

### 9.2 常考算法题

- 两数之和、三数之和
- 反转链表、判断环
- 二叉树遍历（前中后序、层序）
- LRU 缓存（HashMap + 双向链表）
- TopK（堆 / 分治）
- 海量数据 TopN IP（Hash 分治 + 堆）

### 9.3 设计题

**5 万人并发抢票**：

1. 页面静态化 + CDN
2. 按钮置灰 + 验证码（防刷）
3. Redis 预减库存
4. MQ 异步下单
5. 数据库乐观锁扣减
6. 限流（令牌桶）

**手机号归属地查询（5k QPS）**：

1. 号段表加载到内存/Redis
2. 前缀树或 Hash 查找 O(1)
3. 本地缓存 + 多级缓存
4. 5w QPS：集群 + 分片


## 十、网络与操作系统

### 10.1 TCP vs UDP

|        | TCP                | UDP           |
| ------ | ------------------ | ------------- |
| 连接   | 面向连接           | 无连接        |
| 可靠性 | 可靠（重传、确认） | 不可靠        |
| 顺序   | 保证               | 不保证        |
| 速度   | 较慢               | 快            |
| 场景   | HTTP、RPC          | DNS、视频直播 |

### 10.2 HTTP vs HTTPS

- HTTP：明文传输，端口 80
- HTTPS：HTTP + TLS/SSL，端口 443，加密 + 身份认证

**HTTPS 握手简述**：

1. Client Hello（支持的加密套件）
2. Server Hello + 证书
3. 客户端验证证书，生成对称密钥
4. 双方用对称密钥加密通信

### 10.3 GET vs POST

|      | GET            | POST               |
| ---- | -------------- | ------------------ |
| 语义 | 获取资源       | 提交/修改资源      |
| 参数 | URL 查询字符串 | Body               |
| 缓存 | 可缓存         | 一般不缓存         |
| 幂等 | 幂等           | 非幂等             |
| 长度 | URL 长度限制   | 无限制（看服务器） |

### 10.4 浏览器访问 URL 全过程

```
DNS 解析 → TCP 三次握手 → TLS 握手（HTTPS）
→ 发送 HTTP 请求 → 服务器处理 → 返回响应
→ 浏览器解析渲染 → TCP 四次挥手
```

### 10.5 OSI 七层模型

| 层         | 协议/设备        |
| ---------- | ---------------- |
| 应用层     | HTTP、DNS        |
| 表示层     | SSL/TLS          |
| 会话层     | -                |
| 传输层     | TCP、UDP         |
| 网络层     | IP、ICMP、路由器 |
| 数据链路层 | MAC、交换机      |
| 物理层     | 网线、光纤       |

### 10.6 Linux 常考点

- **epoll**：I/O 多路复用，边缘触发/水平触发
- **进程通信**：管道、消息队列、共享内存、信号量、Socket
- **硬链接 vs 软链接**：硬链接同 inode，不能跨分区；软链接是快捷方式
- **僵尸进程**：子进程退出父进程未 wait


## 附录：面试技巧

### A.1 源码阅读建议

面试官常问「看过哪些源码」，建议准备 1~2 个深入点：

- `HashMap` / `ConcurrentHashMap` 扩容与树化
- `ThreadPoolExecutor` 执行流程
- `Spring` Bean 生命周期 / AOP 代理创建
- `MyBatis` 插件与 SQL 执行链
- `Netty` 线程模型与 Pipeline

### A.2 回答框架（STAR + 技术深度）

1. **是什么**：一句话定义
2. **为什么**：解决什么问题
3. **怎么做**：原理 + 关键步骤
4. **优缺点**：对比方案
5. **项目经验**：结合真实场景

### A.3 不要踩的坑

- 只说概念不说原理
- 背八股文但无法举一反三
- 项目描述含糊（「参与了」「负责部分」）
- 源码完全没看过却声称精通
