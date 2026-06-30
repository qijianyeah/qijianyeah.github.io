# 跳槽涨薪必备精选面试题

---

## 目录

- [一、Java 基础与代码题](#一java-基础与代码题)
- [二、Java 集合](#二java-集合)
- [三、Java 并发](#三java-并发)
- [四、JVM](#四jvm)
- [五、Web 与网络基础](#五web-与网络基础)
- [六、Spring 生态](#六spring-生态)
- [七、MyBatis](#七mybatis)
- [八、分布式基础理论](#八分布式基础理论)
- [九、Zookeeper 与 Dubbo](#九zookeeper-与-dubbo)
- [十、Spring Cloud 与微服务治理](#十spring-cloud-与微服务治理)
- [十一、Netty 与 IO 模型](#十一netty-与-io-模型)
- [十二、Redis](#十二redis)
- [十三、MySQL](#十三mysql)
- [十四、消息队列](#十四消息队列)
- [十五、TCP 协议](#十五tcp-协议)

---

## 一、Java 基础与代码题

### 1.1 看以下代码回答问题（一）

```java
public static void main(String[] args) {
    String s = new String("abc");
    // 在这中间可以添加 N 行代码，但必须保证 s 引用的指向不变，最终将输出变成 abcd
    System.out.println(s);
}
```

**思路**：`String` 内部字符数组在 JDK 8 及以前可通过反射修改；JDK 9+ 内部实现改为 `byte[]` + `coder`，反射字段名已变化，面试中通常按 JDK 8 思路作答。

```java
Field value = s.getClass().getDeclaredField("value");
value.setAccessible(true);
value.set(s, "abcd".toCharArray());
```

> 注意：反射修改 `String` 仅用于理解不可变性的边界，生产代码严禁这样做。

### 1.2 看以下代码回答问题（二）

```java
public static void main(String[] args) {
    String s1 = new String("abc");
    String s2 = "abc";
    // s1 == s2?  → false
    String s3 = s1.intern();
    // s2 == s3?  → true
}
```

**解析**：

| 比较 | 结果 | 原因 |
|------|------|------|
| `s1 == s2` | `false` | `s1` 在堆中，`s2` 指向字符串常量池 |
| `s2 == s3` | `true` | `intern()` 将 `"abc"` 放入常量池并返回池中引用 |

`intern()` 流程：先在常量池查找 `"abc"`，存在则返回池中引用；不存在则把堆中字符串放入常量池（JDK 7+ 不再在池中复制新对象，而是把堆中引用登记进常量池）并返回。

### 1.3 看以下代码回答问题（三）

```java
public static void main(String[] args) {
    Integer i1 = 100;
    Integer i2 = 100;
    // i1 == i2?  → true
    Integer i3 = 128;
    Integer i4 = 128;
    // i3 == i4?  → false
}
```

**解析**：`Integer` 内部类 `IntegerCache` 在类加载时缓存 **-128 ~ 127** 的 `Integer` 对象。`Integer.valueOf()` 在此范围内直接返回缓存对象，超出范围则 `new Integer()`。

---

## 二、Java 集合

### 2.1 String、StringBuffer、StringBuilder 的区别

| 特性 | String | StringBuffer | StringBuilder |
|------|--------|--------------|---------------|
| 可变性 | 不可变 | 可变 | 可变 |
| 线程安全 | 天然线程安全（不可变） | 线程安全（方法加 `synchronized`） | 非线程安全 |
| 性能 | 频繁拼接产生大量中间对象 | 多线程安全场景 | 单线程场景最快 |

**选型建议**：少量拼接用 `String`；多线程共享缓冲区用 `StringBuffer`；单线程大量拼接用 `StringBuilder`。

### 2.2 ArrayList 和 LinkedList 有哪些区别

| 维度 | ArrayList | LinkedList |
|------|-----------|------------|
| 底层结构 | 动态数组 | 双向链表 |
| 随机访问 | `get(i)` 为 O(1) | O(n) |
| 尾部插入 | 均摊 O(1) | O(1) |
| 中间插入/删除 | 需要搬移元素，O(n) | 定位 O(n) + 插入/删除 O(1) |
| 内存 | 连续，缓存友好 | 每个节点额外指针开销 |
| 额外能力 | 仅 `List` | 同时实现 `Deque`，可作队列 |

**常见误区**：不能说「LinkedList 一定更适合增删」——若需要按索引频繁访问或中间大量插入，ArrayList 往往更优；LinkedList 适合头尾队列操作、不需要随机访问的场景。

### 2.3 CopyOnWriteArrayList 的底层原理

1. 内部基于数组存储元素。
2. **写操作**（add/set/remove）：加锁 → 复制一份新数组 → 在新数组上修改 → 将 `array` 引用指向新数组。
3. **读操作**：无锁，直接读当前数组快照。
4. 适合**读多写少**；缺点是占内存、写时复制开销大，且读到的可能是短暂过期数据，不适合强实时场景。

### 2.4 HashMap 的扩容机制

**触发条件**：元素个数超过 `capacity × loadFactor`（默认 16 × 0.75 = 12）。

**JDK 1.7**：先建 newTable → 逐桶转移链表元素（头插法，多线程扩容可能形成环形链表）→ 替换 `table`。

**JDK 1.8**：

1. 先建 2 倍大小的新数组。
2. 遍历旧数组每个桶的链表或红黑树。
3. 链表：按 `(e.hash & oldCap) == 0` 拆成低位链、高位链，分别挂到新下标 `j` 或 `j + oldCap`（不需重新取模）。
4. 红黑树：按同样规则拆分或退化为链表。
5. 全部转移完成后替换 `table`。

### 2.5 ConcurrentHashMap 的扩容机制

**JDK 1.7**：基于 `Segment` 分段锁，每个 Segment 内部扩容逻辑类似 HashMap，各段独立判断阈值。

**JDK 1.8**：

1. 取消 Segment，采用 **Node 数组 + CAS + synchronized（锁桶头节点）**。
2. 扩容时设置 `sizeCtl` 为负数标识「扩容中」，其他线程可协助 `helpTransfer`。
3. 将旧数组按段（stride）划分给多个线程并行迁移。
4. 迁移规则与 HashMap 1.8 相同（低位/高位链拆分）。
5. 支持多线程协同扩容，吞吐量显著高于 1.7。

---

## 三、Java 并发

### 3.1 ThreadLocal 的底层原理

1. **作用**：为每个线程提供独立的变量副本，实现线程隔离。
2. **结构**：每个 `Thread` 持有 `ThreadLocalMap`；Map 的 **key 是弱引用的 `ThreadLocal`**，value 是强引用。
3. **内存泄漏**：key 被 GC 后，Entry 的 key 变 null，但 value 仍被 Entry 强引用，而 Entry 又被 Thread 强引用 → 线程池场景下线程不销毁，value 无法回收。**解决**：`finally` 中调用 `ThreadLocal.remove()`。
4. **典型场景**：数据库连接绑定、用户上下文 `UserContext`、SimpleDateFormat 线程隔离。

### 3.2 如何理解 volatile 关键字

`volatile` 保证两件事：

1. **可见性**：写操作会刷回主内存；读操作从主内存读取，不走 CPU 缓存。
2. **有序性**：通过内存屏障禁止指令重排序。

**不保证原子性**：`count++` 这类复合操作仍需 `synchronized` 或 `AtomicInteger`。

底层：写后插入 StoreStore + StoreLoad 屏障；读后插入 LoadLoad + LoadStore 屏障。

### 3.3 ReentrantLock 公平锁与非公平锁

两者底层都基于 **AQS** 队列：

- **非公平锁**：线程调用 `lock()` 时先 CAS 抢锁，失败再入队。
- **公平锁**：先检查 AQS 队列是否有等待线程，有则排队，无才 CAS。

锁释放时，都唤醒队列头节点。非公平锁的「不公平」体现在**加锁阶段**允许插队，唤醒阶段仍是 FIFO。

![ReentrantLock 公平锁与非公平锁](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/salary-interview/salary-interview-005-reentrantlock-fair-unfair.png)

### 3.4 tryLock() 和 lock() 的区别

| 方法 | 行为 | 返回值 |
|------|------|--------|
| `tryLock()` | 尝试加锁，不阻塞 | 成功 `true`，失败 `false` |
| `tryLock(timeout, unit)` | 限时等待 | 超时前成功则 `true` |
| `lock()` | 阻塞直到获取锁 | 无返回值 |

可配合 `tryLock` 实现**避免死锁**（获取不到则释放已持有的锁并重试）。

### 3.5 CountDownLatch 和 Semaphore

| 组件 | 作用 | 典型场景 |
|------|------|----------|
| `CountDownLatch` | 一个或多个线程等待计数归零 | 主线程等待多个子任务完成 |
| `Semaphore` | 控制同时访问资源的线程数 | 限流、连接池 |

底层均基于 AQS：`CountDownLatch` 的 state 倒数到 0 时唤醒等待线程；`Semaphore` 的 state 表示剩余许可数。

### 3.6 synchronized 的偏向锁、轻量级锁、重量级锁

| 锁状态 | 触发条件 | 特点 |
|--------|----------|------|
| 偏向锁 | 只有一个线程访问 | 对象头记录线程 ID，同线程再次进入无需 CAS |
| 轻量级锁 | 第二个线程竞争 | CAS 自旋尝试获取，不阻塞 |
| 重量级锁 | 自旋多次失败 | 升级为 Monitor，线程阻塞，涉及内核态切换 |

**自旋锁**：通过 CAS 循环尝试获取锁，避免线程阻塞/唤醒的内核开销，适合锁持有时间短的场景。

**锁升级路径**（JDK 15+ 偏向锁默认关闭）：无锁 → 偏向锁 → 轻量级锁 → 重量级锁；**不会降级**。

### 3.7 synchronized 和 ReentrantLock 的区别

| 维度 | synchronized | ReentrantLock |
|------|--------------|---------------|
| 层面 | JVM 内置关键字 | JDK API |
| 释放 | 自动释放 | 必须 `unlock()`（建议 `try-finally`） |
| 公平性 | 非公平 | 可选公平/非公平 |
| 功能 | 基础互斥 | 可中断、超时、`Condition` 多条件队列 |
| 锁信息 | 对象头 Mark Word | AQS 的 `state` 字段 |
| 优化 | 锁升级（偏向/轻量/重量） | 无锁升级 |

### 3.8 线程池的底层工作原理

`ThreadPoolExecutor` 执行流程：

```
提交任务
  → 核心线程数未满？创建核心线程执行
  → 工作队列未满？入队
  → 线程数 < maximumPoolSize？创建非核心线程执行
  → 触发拒绝策略（AbortPolicy / CallerRunsPolicy / DiscardPolicy / DiscardOldestPolicy）
```

非核心线程空闲超过 `keepAliveTime` 会被回收。合理配置核心参数是面试高频追问点：`corePoolSize`、`maximumPoolSize`、`workQueue` 类型与容量、`RejectedExecutionHandler`。

---

## 四、JVM

### 4.1 JVM 中哪些是线程共享区

| 区域 | JDK 8 说明 | 线程共享 |
|------|-----------|----------|
| 堆（Heap） | 对象实例、数组 | 是 |
| 方法区 / 元空间（Metaspace） | 类元信息、常量、静态变量 | 是 |
| 直接内存（Direct Memory） | NIO 堆外内存 | 逻辑共享 |
| 虚拟机栈 | 方法栈帧 | 否（线程私有） |
| 本地方法栈 | Native 方法 | 否 |
| 程序计数器 | 当前字节码行号 | 否 |

### 4.2 JVM 中哪些可以作为 GC Root

GC 从 GC Roots 出发标记存活对象，未被标记的即为垃圾。常见 GC Root：

- 虚拟机栈中引用的对象（局部变量）
- 方法区中静态属性引用的对象
- 方法区中常量引用的对象
- 本地方法栈中 JNI 引用的对象
- 所有被 synchronized 持有的对象
- JVM 内部引用（基本类型 Class、系统类加载器等）
- 跨代引用 remembered set 中的对象

### 4.3 项目如何排查 JVM 问题

**正常运行时**：

| 工具 | 用途 |
|------|------|
| `jmap -heap` / `jmap -histo` | 堆内存分布、对象统计 |
| `jstack` | 线程栈、死锁检测 |
| `jstat -gcutil` | GC 频率与耗时，关注 Full GC |
| `jvisualvm` / Arthas | 可视化分析、在线诊断 |

**排查思路**：频繁 Full GC 但无 OOM → 可能是大对象直接进老年代或 Survivor 过小，尝试调大年轻代；CPU 飙高 → `jstack` 找热点线程定位代码。

**已发生 OOM**：

```bash
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/path/to/dump.hprof
```

用 MAT / jvisualvm 分析 dump，定位大对象、泄漏引用链。

### 4.4 类加载器双亲委派模型

默认三层类加载器：

1. `BootstrapClassLoader`（C++ 实现，加载 `JAVA_HOME/lib`）
2. `ExtClassLoader`（加载 `ext` 目录）
3. `AppClassLoader`（加载 classpath）

**委派流程**：`AppClassLoader.loadClass()` → 委派 `ExtClassLoader` → 委派 `BootstrapClassLoader`；父加载器加载失败，子加载器才自己加载。

**目的**：防止核心类被篡改（如自定义 `java.lang.String`），保证类加载唯一性。

**打破双亲委派**：SPI（`Thread.contextClassLoader`）、Tomcat `WebAppClassLoader`、OSGi 模块化。

### 4.5 Tomcat 为什么要使用自定义类加载器

一个 Tomcat 可部署多个 Web 应用，不同应用可能有相同全限定名的类（如 `com.example.User`）。若共用 `AppClassLoader`，后加载的会覆盖先加载的。

Tomcat 为每个应用创建 `WebAppClassLoader`，实现**应用级类隔离**；同时支持**热部署**（销毁旧 ClassLoader 并重建）。

### 4.6 Tomcat 如何进行优化

1. **JVM 参数**：堆大小、GC 选择（G1/ZGC）、元空间。
2. **连接器**：调整 `maxThreads`、`minSpareThreads`、连接超时。
3. **IO 模型**：NIO / APR 替代 BIO。
4. **缓存**：`appReadBufSize`、`bufferPoolSize` 等缓冲区调优。

---

## 五、Web 与网络基础

### 5.1 浏览器发出请求到收到响应经历了哪些步骤

1. 解析 URL，检查缓存（强缓存 / 协商缓存）。
2. DNS 解析（浏览器缓存 → hosts → 本地 DNS → 递归查询）。
3. 建立 TCP 连接（三次握手），HTTPS 还需 TLS 握手。
4. 发送 HTTP 请求报文。
5. 经过路由器、交换机到达服务器。
6. 服务器（如 Tomcat）按 HTTP 协议解析请求，路由到 Servlet / Controller。
7. 业务处理，生成响应。
8. 封装 HTTP 响应，经网络返回浏览器。
9. 浏览器解析 HTML、加载 CSS/JS、渲染页面。

### 5.2 跨域请求是什么？怎么解决？

**同源策略**：协议、域名、端口三者相同才为同源。Ajax 跨域会被浏览器拦截（`<img>`、`<script>` 的 src 不受限）。

| 方案 | 说明 |
|------|------|
| CORS | 服务端响应头 `Access-Control-Allow-Origin` |
| 反向代理 | Nginx 将前端与 API 代理到同源 |
| JSONP | 利用 `<script>` 不受同源限制（仅 GET，已过时） |
| 网关统一转发 | 后端 BFF / Gateway 代为请求 |

---

## 六、Spring 生态

### 6.1 Spring 中 Bean 创建的生命周期

核心步骤：

1. 实例化前：`InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation`
2. 推断构造方法并实例化
3. 实例化后：`postProcessAfterInstantiation`
4. 属性填充（依赖注入）
5. `Aware` 回调（`BeanNameAware`、`BeanFactoryAware` 等）
6. `BeanPostProcessor.postProcessBeforeInitialization`
7. 初始化：`@PostConstruct` → `InitializingBean.afterPropertiesSet` → 自定义 `init-method`
8. `BeanPostProcessor.postProcessAfterInitialization`（**AOP 代理在此生成**）
9. 使用 Bean
10. 销毁：`@PreDestroy` → `DisposableBean.destroy` → 自定义 `destroy-method`

![Spring Bean 生命周期](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/salary-interview/salary-interview-001-spring-bean-lifecycle.png)

### 6.2 Spring 中 Bean 是线程安全的吗

Spring **本身不保证** Bean 线程安全：

- **无状态 Bean**（只有方法、无共享可变字段）：线程安全。
- **有状态 Bean**（含成员变量）：非线程安全，需自行加锁或改为原型作用域。

与作用域无关：单例 / 原型只是生命周期不同，单例多线程共享同一实例，更需注意并发。

### 6.3 ApplicationContext 和 BeanFactory 的区别

| 特性 | BeanFactory | ApplicationContext |
|------|-------------|-------------------|
| 定位 | 基础 IoC 容器 | 增强版容器 |
| Bean 实例化 | 延迟加载（getBean 时） | 默认启动时预实例化单例 |
| 扩展能力 | 基础 | 国际化、事件发布、环境抽象（`Environment`）、AOP 自动集成 |

实际开发几乎都用 `ApplicationContext`（`AnnotationConfigApplicationContext`、`SpringApplication`）。

### 6.4 Spring 中的事务是如何实现的

1. 基于 **AOP + 数据库事务**。
2. `@Transactional` 标注的方法由事务代理对象拦截。
3. 开启事务：`Connection.setAutoCommit(false)`。
4. 执行业务 SQL。
5. 无异常 → `commit`；有回滚异常 → `rollback`。
6. **传播行为**由 `TransactionManager` 管理多个连接（如 `REQUIRES_NEW` 新建连接）。
7. **隔离级别**直接映射数据库隔离级别。

### 6.5 Spring 中什么时候 @Transactional 会失效

| 场景 | 原因 |
|------|------|
| 同类内部自调用 | 绕过了代理对象，AOP 不生效 |
| 方法非 `public` | Spring AOP 默认不代理非 public 方法 |
| 异常被吞掉 | 未抛出异常，无法触发回滚 |
| 异常类型不匹配 | 默认只回滚 `RuntimeException` 和 `Error` |
| 数据库引擎不支持事务 | 如 MyISAM |
| 未被 Spring 管理 | 对象不是 Spring Bean |

**解决自调用**：注入自身代理、`AopContext.currentProxy()`、拆分到另一个 Bean。

### 6.6 Spring 容器启动流程

![Spring 容器启动流程](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/salary-interview/salary-interview-002-spring-container-startup.png)

1. 扫描组件，生成 `BeanDefinition` 存入 `beanDefinitionMap`。
2. 筛选非懒加载的单例 `BeanDefinition` 进行创建（原型 Bean 在 `getBean` 时创建）。
3. 单例 Bean 创建过程：合并 `BeanDefinition` → 实例化 → 属性填充 → 初始化（含 AOP）。
4. 发布 `ContextRefreshedEvent` 等容器事件。
5. 启动完成。

扩展点：`BeanFactoryPostProcessor`（修改 BeanDefinition，如 `@Configuration` 解析）、`BeanPostProcessor`（依赖注入 `@Autowired`、AOP 织入）。

### 6.7 Spring 用到了哪些设计模式

| 模式 | Spring 中的应用 |
|------|----------------|
| 工厂模式 | `BeanFactory`、`ApplicationContext` 创建 Bean |
| 单例模式 | 默认 Bean 作用域为 singleton |
| 代理模式 | AOP、`@Transactional` 事务代理 |
| 模板方法 | `JdbcTemplate`、`RestTemplate`、`RedisTemplate` |
| 观察者模式 | `ApplicationEvent` / `ApplicationListener` |
| 适配器模式 | `HandlerAdapter` 适配不同 Controller |
| 策略模式 | `Resource` 加载策略、`InstantiationStrategy` |
| 装饰器模式 | `BeanWrapper` 对 Bean 属性访问增强 |
| 责任链模式 | `FilterChain`、`HandlerInterceptor` 链 |

### 6.8 SpringMVC 的底层工作流程

![SpringMVC 请求处理流程](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/salary-interview/salary-interview-006-springmvc-flow.png)

1. 请求到达 `DispatcherServlet`（前端控制器）。
2. `HandlerMapping` 根据 URL 找到 `HandlerExecutionChain`（Handler + 拦截器）。
3. `HandlerAdapter` 适配并调用 Controller 方法。
4. Controller 返回 `ModelAndView` 或直接 `@ResponseBody` 数据。
5. `ViewResolver` 解析视图名。
6. `View` 渲染（或 `HttpMessageConverter` 序列化 JSON）。
7. `DispatcherServlet` 写回响应。

### 6.9 SpringBoot 常用注解及其底层实现

**`@SpringBootApplication`** = 三个注解组合：

| 注解 | 作用 |
|------|------|
| `@SpringBootConfiguration` | 本质是 `@Configuration`，启动类也是配置类 |
| `@EnableAutoConfiguration` | 导入 `AutoConfigurationImportSelector`，加载 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` |
| `@ComponentScan` | 默认扫描启动类所在包及子包 |

**其他常用注解**：

| 注解 | 底层 |
|------|------|
| `@Bean` | `@Configuration` 类中被 `@Bean` 标注的方法由 `ConfigurationClassPostProcessor` 解析，注册 `BeanDefinition` |
| `@Controller` / `@RestController` | `@Component` 衍生，配合 `@RequestMapping` 注册 `RequestMappingHandlerMapping` |
| `@Service` | `@Component` 衍生，语义化分层 |
| `@Autowired` | `AutowiredAnnotationBeanPostProcessor` 实现依赖注入 |
| `@Value` | 同上，解析 `${}` 占位符 |
| `@ConfigurationProperties` | 绑定配置文件到 POJO |

### 6.10 SpringBoot 是如何启动 Tomcat 的

1. `SpringApplication.run()` 创建 `ApplicationContext`。
2. `ServletWebServerApplicationContext` 在 `onRefresh()` 中调用 `createWebServer()`。
3. 通过 `ServletWebServerFactory`（默认 `TomcatServletWebServerFactory`）创建嵌入式 Tomcat。
4. `@ConditionalOnClass` 检测 classpath 存在 Tomcat 相关类才自动配置。
5. 注册 `DispatcherServlet` 并启动 Tomcat 监听端口。

### 6.11 SpringBoot 配置文件的加载顺序

优先级从高到低（高覆盖低）：

1. 命令行参数
2. `java` 系统属性（`System.getProperties()`）
3. 操作系统环境变量
4. jar 包外 `application-{profile}.properties/yml`
5. jar 包内 `application-{profile}.properties/yml`
6. jar 包外 `application.properties/yml`
7. jar 包内 `application.properties/yml`
8. `@PropertySource` 指定的配置

---

## 七、MyBatis

### 7.1 MyBatis 存在哪些优点和缺点

**优点**：

- SQL 与 Java 代码解耦，灵活可控
- 比 JDBC 减少样板代码，无需手动开关连接
- 兼容多种数据库（基于 JDBC）
- 与 Spring 深度集成
- 支持结果映射、动态 SQL、插件扩展

**缺点**：

- SQL 需手写，字段多时工作量大
- SQL 与数据库方言绑定，移植需改动
- 复杂关联查询不如 JPA 便捷

### 7.2 MyBatis 中 #{} 和 ${} 的区别

| 特性 | `#{}` | `${}` |
|------|-------|-------|
| 处理方式 | 预编译占位符 `?` | 字符串拼接 |
| 安全性 | 防 SQL 注入 | 有注入风险 |
| 适用场景 | 参数值 | 动态表名、列名、ORDER BY |

```sql
-- name="zhouyu", password="1 or 1=1"
-- #{} 生成：
SELECT * FROM user WHERE name = 'zhouyu' AND password = '1 or 1=1'
-- ${} 生成（危险）：
SELECT * FROM user WHERE name = zhouyu AND password = 1 or 1=1
```

---

## 八、分布式基础理论

### 8.1 什么是 CAP 理论

- **C（Consistency）**：强一致性，所有节点同一时刻数据相同。
- **A（Availability）**：可用性，请求总能得到响应（不保证最新）。
- **P（Partition Tolerance）**：分区容错，网络分区时系统仍能运行。

分布式系统必须容忍分区（P），因此只能在 **CP** 和 **AP** 之间权衡，无法同时满足三者。

### 8.2 什么是 BASE 理论

CAP 无法同时满足时的工程妥协：

- **BA（Basically Available）**：基本可用，允许损失部分功能或响应变慢。
- **S（Soft state）**：软状态，允许中间态数据不一致。
- **E（Eventually consistent）**：最终一致性，经过一段时间后达到一致。

### 8.3 什么是 RPC

远程过程调用：像调用本地方法一样调用远程服务。通信可基于 TCP 自定义协议或 HTTP，序列化常用 Hessian、Protobuf、JSON。Dubbo、gRPC 均为 RPC 框架。

与 HTTP REST 的区别：RPC 侧重**高性能、强类型接口**；REST 侧重**资源语义、跨语言通用性**。

### 8.4 分布式 ID 有哪些解决方案

| 方案 | 优点 | 缺点 |
|------|------|------|
| UUID | 简单、本地生成 | 无序、占空间、不适合做主键索引 |
| 数据库自增 | 简单有序 | 单点瓶颈 |
| Redis INCR | 性能较好 | 依赖 Redis 高可用 |
| ZooKeeper 顺序节点 | 全局有序 | 性能一般 |
| 雪花算法（Snowflake） | 趋势递增、高性能 | 依赖时钟、需处理机器 ID 分配 |

业界开源：美团 Leaf、百度 UidGenerator、滴滴 TinyID。

### 8.5 分布式锁的使用场景与实现

**场景**：跨 JVM 进程对共享资源互斥（库存扣减、订单幂等）。

| 实现 | 特点 |
|------|------|
| Redis（`SET NX EX` + Lua） | 高性能，AP 模型，极端情况可能重复加锁 |
| Redisson | 封装可重入锁、看门狗续期、红锁 |
| ZooKeeper 临时顺序节点 | 强一致（CP），性能较低 |
| 数据库唯一索引 | 简单但性能差 |

### 8.6 什么是分布式事务？有哪些实现方案

**场景**：跨服务操作需保证全部成功或全部失败。

| 方案 | 说明 |
|------|------|
| 本地消息表 | 业务与消息同事务写入，异步投递 + 重试 |
| 事务消息（RocketMQ） | Half 消息 → 本地事务 → commit/rollback |
| TCC | Try-Confirm-Cancel 三阶段，业务侵入大 |
| Saga | 长事务拆为本地事务 + 补偿 |
| Seata AT 模式 | 自动代理数据源，两阶段提交，对业务透明 |

---

## 九、Zookeeper 与 Dubbo

### 9.1 什么是 ZAB 协议

Zookeeper 原子广播协议，三阶段：

1. **Leader 选举**：选出一个 Leader 处理写请求。
2. **数据同步**：Follower 与 Leader 对齐数据。
3. **请求广播**：Leader 用两阶段提交广播写请求，Follower ACK 过半后提交。

Zookeeper 追求**顺序一致性**，实际是强一致倾向的**最终一致**。

### 9.2 为什么 Zookeeper 可以作为注册中心

- 临时节点 + Watcher 实现服务自动注册与下线感知。
- 数据在内存中，读性能高。
- 顺序节点可做分布式 ID、选主。

**局限**：Zookeeper 是 CP 系统，Leader 选举期间可能不可用；注册中心更侧重 AP 时可选 Nacos、Eureka。

### 9.3 Zookeeper 领导者选举流程

1. 各节点初始状态 `LOOKING`，投票给自己。
2. 交换选票，PK 规则：**zxid 大者优先**，zxid 相同则 **myid 大者优先**。
3. 收到更优选票则改票并广播。
4. 某选票获得**过半**投票则当选 Leader，其余为 Follower / Observer。

### 9.4 Zookeeper 集群数据同步

1. Leader 处理所有写请求。
2. 写操作以事务日志形式发送给 Follower，过半 ACK 后提交。
3. 读请求可由任意节点处理（可能读到旧数据）。
4. 新节点加入时通过快照 + 增量日志同步。

### 9.5 Dubbo 支持哪些负载均衡策略

| 策略 | 说明 |
|------|------|
| Random | 随机，支持权重 |
| RoundRobin | 加权平滑轮询 |
| LeastActive | 最少活跃调用数 |
| ConsistentHash | 一致性哈希，同参数总是同一提供者 |

### 9.6 Dubbo 服务导出与引入

**导出（Provider）**：

1. 解析 `@DubboService` 得到 `ServiceConfig`。
2. 调用 `export()` 打开服务端口。
3. 注册地址到注册中心。
4. 启动 Netty 等协议服务器。

**引入（Consumer）**：

1. 解析 `@DubboReference` 得到 `ReferenceConfig`。
2. 从注册中心订阅提供者列表。
3. 生成接口代理对象（JDK 动态代理或 Javassist）。
4. 调用代理方法 → 负载均衡选提供者 → 网络发送请求。

### 9.7 Dubbo 的架构设计

![Dubbo 架构分层](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/salary-interview/salary-interview-003-dubbo-architecture.jpeg)

| 层次 | 职责 |
|------|------|
| config | 配置层，`ServiceConfig` / `ReferenceConfig` |
| proxy | 服务代理层，透明代理 |
| registry | 注册中心层，地址注册与发现 |
| cluster | 集群层，路由 + 负载均衡 |
| monitor | 监控层，调用统计 |
| protocol | 远程调用层，RPC 协议 |
| exchange | 信息交换层，Request-Response 语义 |
| transport | 网络传输层，Netty |
| serialize | 序列化层 |

**核心**：`Protocol + Invoker + Exporter` 即可完成 RPC；`Cluster` 将多 Invoker 伪装为一个。

---

## 十、Spring Cloud 与微服务治理

### 10.1 Spring Cloud 常用组件

| 组件 | 作用 |
|------|------|
| Nacos / Eureka / Consul | 注册中心 |
| Spring Cloud Config / Nacos Config | 配置中心 |
| OpenFeign | 声明式 HTTP 客户端 |
| Spring Cloud Gateway / Zuul | API 网关 |
| Ribbon / Spring Cloud LoadBalancer | 客户端负载均衡 |
| Sentinel / Hystrix | 熔断降级 |
| Sleuth + Zipkin | 链路追踪 |
| Seata | 分布式事务 |

### 10.2 Spring Cloud 和 Dubbo 的区别

| 维度 | Spring Cloud | Dubbo |
|------|--------------|-------|
| 定位 | 微服务全家桶 | 高性能 RPC 框架 |
| 通信 | 以 HTTP/REST 为主 | 自定义二进制协议，性能更高 |
| 生态 | 组件齐全 | 聚焦服务调用，3.x 也扩展了注册配置 |
| 关系 | 互补，可组合使用 | 可作 Spring Cloud 的 RPC 实现 |

### 10.3 服务雪崩、限流、熔断、降级

| 概念 | 说明 |
|------|------|
| 服务雪崩 | 下游故障导致上游资源耗尽，层层扩散 |
| 限流 | 控制单位时间请求量（令牌桶、漏桶、Sentinel） |
| 熔断 | 下游故障时快速失败，停止调用（如 Sentinel 熔断规则） |
| 降级 | 系统过载时关闭非核心功能或返回兜底数据 |

**熔断 vs 降级**：熔断由**下游故障触发**；降级为**主动保护**系统而削减功能。二者都防止系统崩溃。

### 10.4 SOA、分布式、微服务的关系

- **分布式**：多台机器协同完成系统功能。
- **SOA**：面向服务架构，通过 ESB 总线集成各服务。
- **微服务**：更细粒度的 SOA，每个服务独立部署、独立数据库，通过轻量 HTTP/RPC 通信，去中心化治理。

---

## 十一、Netty 与 IO 模型

### 11.1 BIO、NIO、AIO

| 模型 | 全称 | 特点 |
|------|------|------|
| BIO | Blocking IO | 一连接一线程，阻塞读写 |
| NIO | Non-blocking IO | 多路复用（Selector），单线程管理多连接 |
| AIO（NIO.2） | Asynchronous IO | 操作系统完成 IO 后回调通知 |

Java 网络编程主流用 **NIO（Netty）**；AIO 在 Linux 上底层仍是 epoll，实际采用较少。

### 11.2 零拷贝是什么

减少数据在内核态与用户态之间的拷贝次数：

| 技术 | 说明 |
|------|------|
| `mmap` | 内核缓冲区映射到用户空间，减少一次拷贝 |
| `sendfile` | 数据直接从页缓存发到网卡，不经用户空间 |
| `DirectByteBuffer` | 堆外内存，避免 Java 堆与 Native 堆互拷 |

Netty 的 `FileRegion` 封装了 `sendfile`。

### 11.3 Netty 是什么？和 Tomcat 的区别

- **Netty**：基于 NIO 的异步网络框架，专注高性能网络 IO，协议无关。
- **Tomcat**：Servlet 容器，处理 HTTP 请求，面向 Web 应用。

Netty 适合做 RPC 框架底层、IM、网关；Tomcat 适合标准 Web 应用。

### 11.4 Netty 的线程模型

![Netty 线程模型](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/salary-interview/salary-interview-004-netty-thread-model.png)

默认 **Reactor 主从多线程**：

- `bossGroup`：接收连接（一般 1 个线程）。
- `workerGroup`：处理 IO 读写与业务（默认 `CPU核数 × 2`）。

可通过 `ServerBootstrap` 配置为单线程、多线程等模型。

### 11.5 Netty 的高性能体现

1. IO 多路复用（epoll）。
2. 零拷贝（`DirectBuffer`、`FileRegion`）。
3. 内存池（PooledByteBufAllocator）。
4. 单线程串行化处理 Channel，避免锁竞争。
5. 高效序列化（Protobuf 等）。
6. 无锁化设计（CAS、volatile）。

---

## 十二、Redis

### 12.1 Redis 数据结构及典型场景

| 类型 | 典型场景 |
|------|----------|
| String | 缓存、计数器、分布式锁、Session |
| Hash | 存储对象字段 |
| List | 消息流、栈/队列 |
| Set | 去重、交集（共同关注） |
| ZSet | 排行榜、延时队列 |

### 12.2 Redis 分布式锁实现

1. `SET key value NX EX timeout` 保证互斥与过期。
2. **Lua 脚本**保证「判断 + 删除」原子性（防止误删他人锁）。
3. **看门狗**（Redisson）：后台线程自动续期。
4. **红锁**（RedLock）：向 N/2+1 个独立节点加锁，提高可靠性（仍有争议）。

### 12.3 Redis 主从复制原理

1. 从库发送 `PSYNC` 请求。
2. 首次全量同步：主库 `BGSAVE` 生成 RDB → 发送从库 → 从库加载。
3. 同步期间新写命令写入 **replication buffer**，RDB 后发增量命令。
4. 后续增量同步：主库实时传播写命令。
5. 断线重连：尝试 **部分重同步**（`PSYNC` + `repl_backlog`）。

### 12.4 缓存穿透、击穿、雪崩

| 问题 | 现象 | 解决方案 |
|------|------|----------|
| 穿透 | 查询不存在的数据，绕过缓存直击 DB | 布隆过滤器、缓存空值 |
| 击穿 | 热点 key 过期瞬间大量请求打到 DB | 互斥锁重建、逻辑过期 |
| 雪崩 | 大量 key 同时过期 | 过期时间加随机值、多级缓存、集群高可用 |

### 12.5 Redis 和 MySQL 如何保证数据一致

| 策略 | 说明 |
|------|------|
| 先更新 DB 再删缓存 | 简单，但有短暂不一致窗口 |
| 先删缓存再更新 DB | 并发读可能回填旧值 |
| 延时双删 | 更新后延迟再删一次缓存 |
| 订阅 Binlog | Canal 等监听 MySQL 变更异步更新缓存 |
| 读写走 Redis 为主 | 强一致场景用 Redisson 读写锁 |

没有银弹，根据业务容忍度选择；高一致场景推荐 **Canal + 消息队列** 异步刷新。

---

## 十三、MySQL

### 13.1 EXPLAIN 各字段含义

| 字段 | 含义 |
|------|------|
| `id` | 查询序号，子查询或 UNION 中可能不同 |
| `select_type` | 查询类型（SIMPLE、PRIMARY、SUBQUERY 等） |
| `table` | 访问的表 |
| `partitions` | 匹配的分区 |
| `type` | 访问类型，性能从好到差：`system > const > eq_ref > ref > range > index > ALL` |
| `possible_keys` | 可能用到的索引 |
| `key` | 实际使用的索引 |
| `key_len` | 索引使用字节数 |
| `ref` | 索引等值匹配的对象 |
| `rows` | 预估扫描行数 |
| `filtered` | 按条件过滤后剩余行数百分比 |
| `Extra` | 额外信息（`Using index`、`Using filesort`、`Using temporary` 等） |

### 13.2 索引覆盖是什么

查询列全部包含在索引中，无需回表查聚簇索引，Extra 显示 `Using index`。

### 13.3 最左前缀原则

联合索引 `(a, b, c)` 的 B+ 树按 a → b → c 排序。查询必须从**最左列**开始匹配才能利用索引；跳过中间列则后续列无法用索引（如 `WHERE b=1 AND c=2` 不走 `(a,b,c)` 索引）。

### 13.4 InnoDB 如何实现事务

1. 修改页先写入 **Buffer Pool**。
2. 写 **Redo Log**（物理日志，WAL，保证持久性）。
3. 写 **Undo Log**（逻辑日志，保证原子性与 MVCC）。
4. 提交时 Redo Log 刷盘（`innodb_flush_log_at_trx_commit` 控制策略）。
5. 后台线程将脏页异步刷盘。
6. 回滚时用 Undo Log 恢复。

### 13.5 B 树和 B+ 树的区别，为什么 MySQL 用 B+ 树

| 特性 | B 树 | B+ 树 |
|------|------|-------|
| 非叶子节点存数据 | 是 | 否（仅存索引） |
| 叶子节点链表 | 无 | 有，支持范围扫描 |
| 单节点元素 | 有 | 非叶子节点冗余索引 |

MySQL InnoDB 选 B+ 树原因：

- 非叶子节点不存数据，单页容纳更多索引项，树高低（三层约 2000 万行）。
- 叶子节点有序链表，范围查询高效。
- 查询稳定走叶子节点，磁盘 IO 可预测。

### 13.6 MySQL 锁有哪些

**按粒度**：

| 锁 | 说明 |
|----|------|
| 行锁 | 锁定一行，并发度高 |
| 表锁 | 锁定整表 |
| 间隙锁（Gap Lock） | 锁定索引记录之间的间隙，防幻读 |
| 临键锁（Next-Key Lock） | 行锁 + 间隙锁 |

**按模式**：共享锁（S）、排他锁（X）。

**按思想**：乐观锁（版本号 / CAS）、悲观锁（上述锁）。

### 13.7 MySQL 慢查询优化

1. `EXPLAIN` 分析，确保走索引。
2. 检查是否索引覆盖、是否回表过多。
3. 避免 `SELECT *`，减少数据传输。
4. 数据量过大考虑分库分表。
5. 检查硬件与 MySQL 参数（`innodb_buffer_pool_size` 等）。
6. 开启慢查询日志：`slow_query_log = 1`，`long_query_time` 设阈值。

---

## 十四、消息队列

### 14.1 消息队列有哪些作用

1. **解耦**：生产与消费独立演进。
2. **异步**：主流程快速返回，后台消费。
3. **削峰填谷**：缓冲流量高峰。

### 14.2 死信队列与延时队列

- **死信队列（DLQ）**：消费多次失败或 TTL 过期的消息转入死信队列，供人工处理或重试。
- **延时队列**：消息在指定时间后才可被消费，用于订单超时取消、定时通知。RocketMQ 支持 18 个延时级别；RabbitMQ 用死信 + TTL 实现。

### 14.3 Kafka 为什么比 RocketMQ 吞吐量高

1. **顺序写磁盘** + Page Cache，充分利用磁盘顺序 IO。
2. **零拷贝**（`sendfile`）发送消息。
3. **批量发送与压缩**，减少网络往返。
4. **分区并行**，水平扩展能力强。

代价：Kafka 偏向吞吐，同步刷盘配置下可靠性可调，但默认异步发送会牺牲一定可靠性。

### 14.4 Kafka Pull 和 Push

| 模式 | 优点 | 缺点 |
|------|------|------|
| Pull（Kafka） | 消费者按能力拉取，可批量 | 可能空轮询，需协调 |
| Push | 实时性好 | 易造成消费者压力过大 |

Kafka 选择 Pull，由消费者控制速率。

### 14.5 RocketMQ 底层实现原理

1. `NameServer` 轻量注册中心，Broker 每 30s 心跳。
2. Producer 从 NameServer 获取 Broker 地址，按队列选择发送。
3. Consumer 主动 Pull 消息（Push 本质是长轮询 Pull）。
4. 消息存储：CommitLog 顺序写 + ConsumeQueue 索引。

### 14.6 消息队列如何保证可靠传输

**不丢消息**：

| 阶段 | 措施 |
|------|------|
| 生产 | 同步发送 + ACK（Kafka `acks=all`，RocketMQ 同步刷盘） |
| 存储 | 多副本同步 |
| 消费 | 手动 ACK，消费成功后再提交 offset |

**不重复**：消费者实现**幂等**（唯一键、状态机、Redis 去重）。

---

## 十五、TCP 协议

### 15.1 三次握手

1. 客户端 → `SYN`（seq=x）
2. 服务端 → `SYN+ACK`（seq=y, ack=x+1）
3. 客户端 → `ACK`（ack=y+1）

**目的**：双方确认收发能力，同步初始序列号，防止历史连接干扰。

### 15.2 四次挥手

1. 客户端 → `FIN`
2. 服务端 → `ACK`（可能还有数据要发）
3. 服务端 → `FIN`
4. 客户端 → `ACK`（等待 2MSL 后关闭）

**为什么四次**：TCP 全双工，双方需各自关闭发送通道；服务端收到 FIN 后可能仍有数据要传输，ACK 与 FIN 不能合并。

**2MSL 等待**：确保最后的 ACK 能到达服务端，并让旧连接报文在网络中消散。
