# sun.misc.Unsafe 方法分类与 JDK 源码使用总结

`sun.misc.Unsafe` 是 JDK 暴露给受信代码使用的一组“越过 Java 语言安全边界”的底层能力。它的 API 很杂，但从职责上可以分成几大类。理解它的关键不是逐个背方法，而是按**功能**、**内存语义**、**JDK 使用场景**来归类。

> 说明：
> - `sun.misc.Unsafe` 主要出现在 JDK 8 及更早版本；JDK 9+ 大量实现迁移到 `jdk.internal.misc.Unsafe`。
> - 两者能力高度相近，后者还加入了更细粒度的 acquire/release/opaque/fence 等方法。
> - 下文主线以 `sun.misc.Unsafe` 为中心，必要时补充 `jdk.internal.misc.Unsafe` 来说明现代 JDK 源码中的实际使用方式。

---

## 1. 总体分类图

按功能可分为：

1. **对象/字段/数组的定位与访问**
2. **堆外内存（native memory）管理与拷贝**
3. **原子操作 / CAS / 获取并更新**
4. **可见性、有序性与内存屏障**
5. **线程阻塞与唤醒（park/unpark）**
6. **类、对象与初始化绕过能力**
7. **数组布局、地址、页大小等 VM 信息查询**
8. **异常、监视器、历史兼容 API**

这些能力分别服务于 JDK 的几类核心实现：

- 并发容器与原子类
- 锁、线程调度与 ForkJoin 框架
- NIO 直接内存 / DirectByteBuffer
- 序列化、反射、对象实例化
- JVM 内部布局感知型优化

---

## 2. 对象/字段/数组访问类

### 2.1 普通读写

典型方法：

- `getInt(Object o, long offset)` / `putInt(...)`
- `getLong` / `putLong`
- `getObject` / `putObject`（JDK 9+ 对应 `getReference` / `putReference`）
- `getBoolean/getByte/getShort/getChar/getFloat/getDouble`
- 同名 `putXxx`

### 作用

绕过 Java 访问控制，直接通过“对象 + 偏移量”读写字段或数组元素。

### 适用场景

- 已知字段内存偏移量，希望避免反射开销
- 并发类内部直接访问字段
- 需要基于数组基址和元素步长做手工寻址

### JDK 源码中的使用

#### 1）`AtomicLong`

源码中先拿到字段偏移量：

- `java/util/concurrent/atomic/AtomicLong.java`

```java
private static final long VALUE = U.objectFieldOffset(AtomicLong.class, "value");
```

之后读写 `value` 字段时直接走 Unsafe：

- `set()` 使用 `putLongVolatile`
- `getPlain()` 使用 `getLong`
- `setPlain()` 使用 `putLong`

这说明 Unsafe 最常见用途之一，就是让 JDK 自己实现“比反射更低层、比字节码更灵活”的字段访问。

#### 2）`LockSupport`

- `java/util/concurrent/locks/LockSupport.java`

```java
private static final long PARKBLOCKER = U.objectFieldOffset(Thread.class, "parkBlocker");
U.putReferenceOpaque(t, PARKBLOCKER, arg);
U.getReferenceOpaque(t, PARKBLOCKER);
```

这里的场景不是普通业务字段访问，而是**直接操纵 Thread 内部状态字段**，为诊断和阻塞原因跟踪服务。

---

## 3. volatile / opaque / acquire / release 访问类

### 3.1 volatile 语义访问

典型方法：

- `getObjectVolatile` / `putObjectVolatile`
- `getIntVolatile` / `putIntVolatile`
- `getLongVolatile` / `putLongVolatile`

### 作用

以 volatile 读写语义访问字段，保证可见性和部分有序性。

### 使用场景

- 无锁/低锁并发结构
- 原子类的基础读写
- 状态位、版本号、队列头尾等共享变量

### JDK 源码中的使用

#### `AtomicLong`

- `set(long newValue)` → `U.putLongVolatile(this, VALUE, newValue)`

这里表示“普通 set 方法也必须具备 volatile store 语义”，因为 `AtomicLong` 的规范就是并发可见。

---

### 3.2 JDK 9+ 更细粒度内存语义

典型方法（在 `jdk.internal.misc.Unsafe` 中更常见）：

- `getReferenceOpaque` / `putReferenceOpaque`
- `getLongAcquire` / `putLongRelease`
- `putReferenceRelease`
- `getReferenceAcquire`

### 作用

提供比 volatile 更细粒度的内存序：

- **plain**：近似普通读写
- **opaque**：保证原子性，但排序更弱
- **acquire**：读取后面的操作不能重排到前面
- **release**：写入前面的操作不能重排到后面
- **volatile**：最强的一般访问语义

### 使用场景

- 提升并发框架性能
- 避免不必要的全 volatile 成本
- 实现接近 `VarHandle` 的底层能力

### JDK 源码中的使用

#### 1）`ConcurrentHashMap`

- `java/util/concurrent/ConcurrentHashMap.java`

```java
U.getReferenceAcquire(tab, ((long)i << ASHIFT) + ABASE)
U.compareAndSetReference(tab, ((long)i << ASHIFT) + ABASE, c, v)
U.putReferenceRelease(tab, ((long)i << ASHIFT) + ABASE, v)
```

用途：

- `tabAt`：以 acquire 语义读取桶元素
- `casTabAt`：CAS 更新桶位
- `setTabAt`：以 release 语义发布新节点

这是现代 JDK 中 Unsafe 的代表性用法：**不直接给整个字段加 volatile，而是按操作点精确选择内存语义**。

#### 2）`LockSupport`

- `putReferenceOpaque`
- `getReferenceOpaque`

这里只需要“可诊断可观测”，不要求最强 volatile 语义，因此 opaque 足够。

---

## 4. 字段偏移、静态字段基址、数组布局查询类

典型方法：

- `objectFieldOffset(Field f)`
- `staticFieldOffset(Field f)`
- `staticFieldBase(Field f)`
- `arrayBaseOffset(Class<?> arrayClass)`
- `arrayIndexScale(Class<?> arrayClass)`

JDK 9+ 常见还有按名称取偏移的内部便捷版本，如：

- `objectFieldOffset(Class<?> c, String name)`

### 作用

这些方法本身不读写数据，而是提供“寻址所需元数据”：

- 某字段在对象中的偏移量
- 某静态字段的存储基址和偏移
- 数组第一个元素的起始偏移
- 数组每个元素的步长

### 使用场景

- 给后续 `get/put/CAS` 提供参数
- 手写高性能数组/表结构寻址逻辑
- JVM 内部对象布局敏感代码

### JDK 源码中的使用

#### 1）`AtomicLong`

`objectFieldOffset(AtomicLong.class, "value")` 用于得到 `value` 字段地址。

#### 2）`LockSupport`

`objectFieldOffset(Thread.class, "parkBlocker")` 直接定位 `Thread` 内部字段。

#### 3）`ConcurrentHashMap`

通过 `ABASE`、`ASHIFT` 配合数组访问：

```java
((long)i << ASHIFT) + ABASE
```

本质上就是：

- `ABASE` = 数组首元素偏移
- `ASHIFT` = 元素步长对应的位移量

这是一种非常典型的 Unsafe 用法：**把 Java 数组当成“可手工寻址的连续内存”来访问**。

---

## 5. CAS 与原子更新类

### 5.1 经典 CAS

典型方法：

- `compareAndSwapObject`
- `compareAndSwapInt`
- `compareAndSwapLong`

JDK 9+ 对应/扩展：

- `compareAndSetReference`
- `compareAndSetLong`
- `compareAndExchangeLong`
- `weakCompareAndSetLongPlain/Acquire/Release/...`

### 作用

实现无锁并发的核心原语：

> 如果内存中的旧值 == 期望值，则原子替换为新值。

### 使用场景

- 原子类
- 并发队列/栈
- 并发哈希表桶位安装
- 锁状态更新

### JDK 源码中的使用

#### 1）`AtomicLong`

- `compareAndSet` → `U.compareAndSetLong(this, VALUE, expectedValue, newValue)`
- `getAndAddLong`、`getAndSetLong` 等也是基于底层原子更新实现

这是最标准的 CAS 场景：**用户可见原子类 API 的底层实现**。

#### 2）`ConcurrentHashMap`

- `casTabAt(...)`

用于在空桶上原子安装节点，避免全表锁。

#### 3）JDK 8 `sun.misc.Unsafe`

`compareAndSwapXxx` 是很多 `java.util.concurrent` 包中无锁算法的基础接口。

---

### 5.2 获取并更新（fetch-and-add / get-and-set）

典型方法（主要见于 JDK 9+ 内部 Unsafe）：

- `getAndAddLong`
- `getAndSetLong`
- `getAndSetReference`

### 作用

以单条原子语义完成“读取旧值 + 写入新值”。

### 使用场景

- 序号递增
- 计数器
- 原子交换引用

### JDK 源码中的使用

#### `AtomicLong`

- `getAndIncrement()` → `U.getAndAddLong(this, VALUE, 1L)`
- `getAndDecrement()` → `U.getAndAddLong(this, VALUE, -1L)`
- `getAndSet(long)` → `U.getAndSetLong(this, VALUE, newValue)`

属于“原子类包装底层原语”的典型模式。

---

## 6. 堆外内存访问与管理类

典型方法：

- `allocateMemory(long bytes)`
- `reallocateMemory(long address, long bytes)`
- `freeMemory(long address)`
- `setMemory(...)`
- `copyMemory(...)`
- `copySwapMemory(...)`（JDK 9+）
- `getByte(long address)` / `putByte(long address, byte x)`
- `getInt(long address)` / `putInt(long address, int x)`
- `getAddress(long address)` / `putAddress(long address, long x)`

### 作用

直接操作 C heap / native memory，不受 Java GC 管理。

### 使用场景

- DirectByteBuffer
- NIO / 网络 / 文件 I/O
- 与 JNI、本地库共享内存
- 大块内存拷贝和初始化

### JDK 源码中的使用

#### 1）`Bits`

- `java/nio/Bits.java`

```java
PAGE_SIZE = UNSAFE.pageSize();
UNSAFE.setMemory(srcAddr + offset, len, value);
```

用途：

- 获取系统页大小
- 对直接内存做批量清零/填充

#### 2）`Direct-X-Buffer.java.template`

JDK 直接缓冲区模板中最典型：

```java
base = UNSAFE.allocateMemory(size);
UNSAFE.freeMemory(address);
Bits.setMemory(base, size, (byte) 0);
```

用途：

- 分配 direct buffer 背后的 native memory
- 初始化为 0
- 通过 Cleaner 注册释放逻辑

这类代码说明 Unsafe 的另一个核心身份：**JDK 自己的“手工 malloc/free”接口**。

---

## 7. 内存屏障与有序性控制类

典型方法：

- `loadFence()`
- `storeFence()`
- `fullFence()`

以及通过访问语义隐式表达顺序的：

- `putOrderedObject`（JDK 8）
- `putLongRelease` / `getLongAcquire`（JDK 9+）

### 作用

手动插入内存屏障，约束 CPU 与编译器重排。

### 使用场景

- 实现 lock-free 算法
- 发布-订阅模式中的安全发布
- 细粒度替代 volatile

### JDK 源码中的实际情况

现代 JDK 更常用的是 **acquire/release/opaque** 风格，而不是在业务代码里大量直接写 `fullFence()`。

#### 例子：`ConcurrentHashMap`

- 读桶元素：`getReferenceAcquire`
- 写桶元素：`putReferenceRelease`

本质上等价于“把 fence 融合进具体访问操作里”，可读性和性能都更好。

#### 例子：`AtomicLong.lazySet()`

- `U.putLongRelease(this, VALUE, newValue)`

这是 JDK 9+ 对 JDK 8 `putOrderedLong` 语义的现代化表达。

---

## 8. 线程挂起/恢复类

典型方法：

- `park(boolean isAbsolute, long time)`
- `unpark(Object thread)`

### 作用

提供底层线程阻塞/唤醒机制，是很多同步器的基础。

### 使用场景

- `LockSupport`
- AQS
- ForkJoinPool
- 各类锁、同步器、线程池等待机制

### JDK 源码中的使用

#### 1）`LockSupport`

- `java/util/concurrent/locks/LockSupport.java`

```java
U.unpark(thread);
U.park(false, 0L);
U.park(false, nanos);
U.park(true, deadline);
```

`LockSupport` 基本就是对 Unsafe park/unpark 的安全包装与语义解释层。

#### 2）`ForkJoinPool`

- `java/util/concurrent/ForkJoinPool.java`

文件本身大量依赖 `LockSupport`，而 `LockSupport` 底层再落到 Unsafe 的 `park/unpark`。

所以从分层上看：

> `ForkJoinPool / AQS / 锁实现` → `LockSupport` → `Unsafe.park/unpark`

---

## 9. 类定义、对象创建、类初始化控制类

典型方法：

- `defineClass(...)`
- `defineAnonymousClass(...)`（历史接口）
- `allocateInstance(Class<?> cls)`
- `shouldBeInitialized(Class<?> c)`
- `ensureClassInitialized(Class<?> c)`

### 作用

绕过正常 Java 语言路径来：

- 定义类
- 分配对象但不执行构造函数
- 控制类初始化时机

### 使用场景

- 反射/序列化/代理框架
- 运行时代码生成
- JVM 内部启动与类加载细节控制

### JDK 源码中的使用场景说明

#### 1）`allocateInstance`

最经典的语义是：

> 只分配对象，不调用构造函数。

它是序列化体系、某些反射工厂、代理/框架“绕过构造器恢复对象状态”的基础能力。

在现代 JDK 中，序列化路径更多通过 `ReflectionFactory` 等封装间接使用，但底层思想与 Unsafe 一致：

- 反序列化时对象需要“先拿到一块已分配内存的实例”
- 然后再按流里的字段值恢复状态
- 而不是走业务构造函数逻辑

`ObjectInputStream` / `ObjectStreamClass` 相关代码体现了这一使用场景：对象恢复与普通 `new` 语义是分离的。

#### 2）`defineClass` / `defineAnonymousClass`

这类方法常用于动态生成类、Lambda/代理等历史实现路径。现代 JDK 中很多场景已迁移到 `MethodHandles.Lookup#defineClass` 等更安全 API。

#### 3）`ensureClassInitialized`

配合 `staticFieldBase/staticFieldOffset` 使用，确保静态字段访问前类已经初始化。

---

## 10. VM 信息查询类

典型方法：

- `addressSize()`
- `pageSize()`
- `arrayBaseOffset()`
- `arrayIndexScale()`
- `unalignedAccess()`（现代 JDK 内部 Unsafe 常见）
- `isWritebackEnabled()`、`dataCacheLineFlushSize()` 等较底层平台能力查询

### 作用

让 Java 代码感知底层平台布局和硬件能力。

### 使用场景

- direct memory 对齐
- NIO 优化
- 数组寻址优化
- 硬件相关能力判断

### JDK 源码中的使用

#### `Bits`

```java
PAGE_SIZE = UNSAFE.pageSize();
private static boolean UNALIGNED = UNSAFE.unalignedAccess();
```

说明 `Bits` 会根据平台页大小、是否允许非对齐访问来选择优化路径。

---

## 11. 异常抛出与特殊控制类

典型方法：

- `throwException(Throwable ee)`

### 作用

绕过 Java 编译期 checked exception 检查，直接抛异常。

### 使用场景

- JDK/框架内部把底层异常透传出去
- 某些反射、代理或桥接代码

### 评价

这个方法体现的不是并发或内存能力，而是“绕过语言层限制”。因此它通常归入“特殊控制能力”。

---

## 12. 监视器与历史兼容类

典型方法：

- `monitorEnter(Object o)`
- `monitorExit(Object o)`
- `tryMonitorEnter(Object o)`

### 作用

直接操作对象监视器。

### 状态

这些 API 已被标记为 `@Deprecated`，现代 JDK 基本不推荐使用。

### 使用场景

- 主要是历史兼容
- 现代 JDK 同步体系更多走 `synchronized`、AQS、LockSupport 等路径

---

## 13. 各类方法与典型使用场景对照表

| 功能分类 | 代表方法 | 主要作用 | JDK 典型场景 |
|---|---|---|---|
| 对象字段普通访问 | `getInt/putInt/getObject/putObject` | 直接按偏移读写对象字段 | `AtomicLong`、线程内部字段访问 |
| volatile 访问 | `getLongVolatile/putLongVolatile` | 并发可见性读写 | 原子类状态字段 |
| acquire/release/opaque | `getReferenceAcquire/putReferenceRelease/getReferenceOpaque` | 更细粒度内存序控制 | `ConcurrentHashMap`、`LockSupport` |
| 字段定位 | `objectFieldOffset/staticFieldOffset/staticFieldBase` | 获取字段地址元数据 | 原子类、线程字段、静态字段访问 |
| 数组布局 | `arrayBaseOffset/arrayIndexScale` | 计算数组元素地址 | `ConcurrentHashMap` 桶数组寻址 |
| CAS | `compareAndSwapXxx` / `compareAndSetXxx` | 无锁原子更新 | 原子类、并发容器 |
| 获取并更新 | `getAndAddLong/getAndSetLong` | 单操作原子读改写 | `AtomicLong` |
| 堆外内存分配 | `allocateMemory/freeMemory/reallocateMemory` | 手工管理 native memory | DirectByteBuffer |
| 堆外内存批量操作 | `setMemory/copyMemory/copySwapMemory` | 批量初始化、复制、字节交换 | `Bits`、NIO |
| 地址/平台信息 | `getAddress/putAddress/pageSize/addressSize` | 指针和系统布局查询 | NIO、直接内存管理 |
| 线程阻塞唤醒 | `park/unpark` | 底层阻塞机制 | `LockSupport`、锁、线程池 |
| 对象绕过构造创建 | `allocateInstance` | 分配对象不执行构造器 | 序列化、反射工厂 |
| 类定义/初始化 | `defineClass/ensureClassInitialized` | 动态定义类与控制初始化 | 动态类加载、内部运行时 |
| 异常绕过 | `throwException` | 绕过 checked exception 限制 | 反射/桥接内部逻辑 |
| 历史监视器接口 | `monitorEnter/monitorExit` | 直接操作 monitor | 历史兼容，现少用 |

---

## 14. 为什么 JDK 大量使用 Unsafe

JDK 使用 Unsafe 的核心原因有三点：

### 14.1 Java 语言本身不提供这些底层原语

例如：

- CAS
- park/unpark
- 按偏移访问字段
- 手工分配 native memory

这些能力必须由 JVM 支持，再通过 Unsafe 暴露给 JDK 类库实现者。

### 14.2 JDK 需要“接近 JVM/硬件”的性能

例如 `ConcurrentHashMap`、`AtomicLong`、`ForkJoinPool`、`DirectByteBuffer` 都属于性能极敏感组件，不能完全依赖反射或重量级同步。

### 14.3 它是很多更高层 API 的底层基础

从源码层次看，很多你平时直接使用的标准库 API，本质上是 Unsafe 的“安全封装层”：

- `AtomicLong` / `AtomicInteger`
- `LockSupport`
- `ConcurrentHashMap`
- `DirectByteBuffer`
- 序列化与反射工厂的一部分能力

---

## 15. 现代替代方案与演进方向

Unsafe 并没有完全消失，但 JDK 在逐步用更明确的标准/内部 API 替代它。

### 15.1 并发字段访问 → `VarHandle`

`VarHandle` 以标准 API 的形式提供：

- plain / opaque / acquire / release / volatile
- compareAndSet / compareAndExchange
- getAndAdd / getAndSet

它本质上是在语言层正规化 Unsafe 的并发内存访问能力。

### 15.2 堆外内存 → Foreign Memory / `MemorySegment`

直接内存与本地内存操作正在逐步迁移到更安全的 Foreign Function & Memory API。

### 15.3 动态定义类 → `MethodHandles.Lookup#defineClass`

比 `Unsafe.defineClass` 更受控。

### 15.4 对象实例化绕过 → 仍以内部机制为主

像序列化这种场景仍然需要 JVM 级支持，但一般不会鼓励应用代码直接依赖 Unsafe。

---

## 16. 总结

如果按“最重要的使用面”来理解 `sun.misc.Unsafe`，可以抓住四条主线：

1. **并发原语**：CAS、volatile/acquire/release、fence
2. **线程调度原语**：park/unpark
3. **内存原语**：堆外内存分配、拷贝、清零、地址访问
4. **对象/类绕过原语**：字段偏移、绕过构造器实例化、动态定义类

在 JDK 源码中，它主要不是给业务代码用的，而是给标准库实现这些高性能、低层组件用的：

- `AtomicLong`：原子更新与字段偏移
- `ConcurrentHashMap`：数组槽位的 acquire/release/CAS
- `LockSupport`：park/unpark 与线程 blocker 字段访问
- `Bits` / `DirectByteBuffer`：native memory 分配、清零、释放
- 序列化/反射体系：绕过构造器创建对象

一句话概括：

> `Unsafe` 是 JDK 类库通往 JVM 底层能力的“后门接口”；标准库里的高性能并发、直接内存、线程阻塞与对象底层操作，很多都建立在它之上。

---

## 17. 参考源码

- OpenJDK 8 `sun.misc.Unsafe`
  - `jdk/src/share/classes/sun/misc/Unsafe.java`
- OpenJDK 主线 `jdk.internal.misc.Unsafe`
  - `src/java.base/share/classes/jdk/internal/misc/Unsafe.java`
- `java/util/concurrent/atomic/AtomicLong.java`
- `java/util/concurrent/ConcurrentHashMap.java`
- `java/util/concurrent/locks/LockSupport.java`
- `java/nio/Bits.java`
- `java/nio/Direct-X-Buffer.java.template`
- `java/io/ObjectInputStream.java`
- `java/io/ObjectStreamClass.java`
