# MyBatis 框架核心面试知识梳理

## 1. MyBatis 是什么

- MyBatis 是一个**半自动 ORM 持久层框架**。
- 它的核心能力是：**把 Java 对象与 SQL 执行过程做映射**，帮助开发者更方便地访问数据库。
- 和 Hibernate/JPA 这类“全自动 ORM”相比，MyBatis 更强调：
  - SQL 由开发者自己编写
  - 映射关系由框架负责组织
  - 灵活、可控、容易做复杂查询优化

一句话理解：

> MyBatis 不是替你“生成 SQL”的框架，而是帮你把“写好的 SQL”优雅地执行出来，并映射成 Java 对象。

---

## 2. 为什么项目里常用 MyBatis

### 2.1 优点

- SQL 可控，适合复杂查询、性能优化、分页、分库分表等场景。
- 学习成本相对较低，尤其适合熟悉 SQL 的后端开发。
- 支持 XML 和注解两种方式定义 SQL。
- 与 Spring/Spring Boot 集成成熟。
- 支持动态 SQL，能较方便地拼接条件查询。
- 支持一级缓存、二级缓存、插件扩展。

### 2.2 缺点

- 需要自己维护 SQL，表多、查询多时工作量较大。
- SQL 与 Java 代码分散时，可维护性依赖团队规范。
- 复杂映射、动态 SQL 写多了以后，XML 容易膨胀。
- 对象关系映射能力不如 Hibernate/JPA 那么“自动化”。

### 2.3 适用场景

MyBatis 更适合：

- 业务 SQL 比较复杂的系统
- 对 SQL 性能要求高的系统
- 需要精细控制查询逻辑的场景
- 互联网业务中常见的 CRUD + 复杂报表 + 条件查询场景

不太适合：

- 希望完全由框架托管对象关系映射
- 大量简单实体，希望极度减少 SQL 编写工作

---

## 3. MyBatis 的核心定位

很多面试会问：**MyBatis 到底算什么框架？**

建议回答：

- 它是一个持久层框架。
- 它本质上是 JDBC 的封装。
- 它不是数据库连接池，不是事务框架，也不是完整 ORM。
- 它重点解决的是：
  - SQL 与 Java 方法之间的映射
  - 参数绑定
  - 结果集映射
  - 重复 JDBC 模板代码的消除

### 3.1 MyBatis 和 JDBC 的关系

JDBC 需要开发者手动处理：

- 获取连接
- 创建 `PreparedStatement`
- 设置参数
- 执行 SQL
- 遍历 `ResultSet`
- 组装对象
- 关闭资源

MyBatis 则把这些模板代码封装起来，开发者主要关心：

- SQL 是什么
- 入参是什么
- 返回对象映射成什么

---

## 4. MyBatis 核心架构

面试里常见高频问题：**MyBatis 的核心组件有哪些？**

### 4.1 核心组件总览

- `SqlSessionFactoryBuilder`
- `SqlSessionFactory`
- `SqlSession`
- `Executor`
- `MappedStatement`
- `ParameterHandler`
- `StatementHandler`
- `ResultSetHandler`
- `MapperProxy`

可以理解成下面这条链路：

```text
配置文件 -> SqlSessionFactory -> SqlSession -> Executor -> JDBC -> 数据库
                                           |
                                      Mapper 代理调用
```

### 4.2 `SqlSessionFactoryBuilder`

- 作用：根据配置构建 `SqlSessionFactory`
- 一般只在启动阶段使用一次
- 不建议长期持有

### 4.3 `SqlSessionFactory`

- 作用：生产 `SqlSession`
- 生命周期通常较长
- 一个数据库环境通常对应一个 `SqlSessionFactory`

### 4.4 `SqlSession`

- 类似 MyBatis 操作数据库的会话对象
- 可以执行增删改查、提交事务、回滚事务
- 默认不是线程安全的，不能多线程共享

### 4.5 `Executor`

- 真正执行 SQL 的核心组件
- 它会调用 StatementHandler 去准备 SQL，再执行 JDBC 操作
- MyBatis 提供三种执行器：
  - `SimpleExecutor`
  - `ReuseExecutor`
  - `BatchExecutor`

### 4.6 `MappedStatement`

- 每一条 SQL 在 MyBatis 中最终都会被解析成一个 `MappedStatement`
- 它里面保存了：
  - SQL 配置
  - 入参类型
  - 返回值类型
  - 缓存配置
  - SQL 命令类型等信息

可以把它理解成：

> Mapper 方法对应的“元数据对象”。

### 4.7 四大对象

MyBatis 面试高频：**四大对象是什么？**

- `Executor`
- `StatementHandler`
- `ParameterHandler`
- `ResultSetHandler`

它们是插件机制主要拦截的目标。

---

## 5. MyBatis 执行流程

### 5.1 从启动到执行的过程

```text
读取全局配置 -> 解析 mapper.xml / 注解 SQL -> 封装为 MappedStatement
-> 创建 SqlSessionFactory -> 打开 SqlSession -> 获取 Mapper 代理对象
-> 调用 Mapper 方法 -> MapperProxy 拦截 -> Executor 执行 SQL
-> ParameterHandler 设置参数 -> StatementHandler 执行语句
-> ResultSetHandler 映射结果 -> 返回 Java 对象
```

### 5.2 面试表达版

可以这样答：

1. MyBatis 启动时会解析全局配置和 Mapper 映射文件。
2. 每条 SQL 会被封装成 `MappedStatement`。
3. 业务调用 Mapper 接口时，实际上调用的是 JDK 动态代理生成的代理对象。
4. 代理对象会根据方法信息找到对应的 `MappedStatement`。
5. 然后交给 `Executor` 执行。
6. 执行过程中由 `ParameterHandler` 设置参数、`StatementHandler` 处理 JDBC 语句、`ResultSetHandler` 做结果集映射。
7. 最终把查询结果转成 Java 对象返回。

---

## 6. Mapper 接口为什么可以直接调用

这是面试中的经典题。

### 6.1 原理

- MyBatis 会为 Mapper 接口创建代理对象。
- 代理技术通常是 **JDK 动态代理**。
- 调用接口方法时，会进入代理逻辑。
- 代理逻辑根据：
  - 接口全限定名
  - 方法名
  - 参数信息
  - 返回值类型
  去找到对应的 SQL 配置并执行。

### 6.2 本质

你调用的不是接口本身，而是接口对应的代理对象。

### 6.3 面试回答模板

> Mapper 接口之所以不用写实现类，是因为 MyBatis 在运行期通过 JDK 动态代理为接口生成代理对象。调用接口方法时，代理对象会根据 namespace、methodId 和参数信息找到对应的 `MappedStatement`，然后交给 `SqlSession` 和 `Executor` 去执行 SQL。

---

## 7. `#{}` 和 `${}` 的区别

这是 MyBatis 面试必问题。

### 7.1 `#{}`

- 表示**预编译占位符**
- 最终会转换为 `?`
- 通过 `PreparedStatement` 设置参数
- 能防止 SQL 注入
- 推荐日常开发优先使用

示例：

```sql
select * from user where id = #{id}
```

最终类似于：

```sql
select * from user where id = ?
```

### 7.2 `${}`

- 表示**字符串直接拼接**
- 不会进行预编译
- 有 SQL 注入风险
- 常用于：
  - 动态表名
  - 动态字段名
  - `order by` 字段名拼接

示例：

```sql
select * from user order by ${column}
```

### 7.3 核心区别总结

| 对比项 | `#{}` | `${}` |
|---|---|---|
| 底层方式 | 预编译参数绑定 | 字符串拼接 |
| 是否防 SQL 注入 | 是 | 否 |
| 使用场景 | 普通参数值 | 表名、列名、排序字段 |
| 推荐程度 | 强烈推荐 | 谨慎使用 |

### 7.4 面试标准答法

> `#{}` 会被 MyBatis 解析为 JDBC 的 `?` 占位符，通过 `PreparedStatement` 绑定参数，能够避免 SQL 注入；`${}` 是直接做字符串拼接，不会预编译，存在注入风险。一般查询条件用 `#{}`，只有动态表名、动态列名、排序字段这类不能作为预编译参数的场景才考虑 `${}`。

---

## 8. MyBatis 动态 SQL

### 8.1 常见标签

- `<if>`
- `<where>`
- `<trim>`
- `<set>`
- `<choose>` / `<when>` / `<otherwise>`
- `<foreach>`

### 8.2 动态 SQL 解决什么问题

主要解决：

- 条件查询拼接麻烦
- 批量查询 / 批量插入拼接麻烦
- 更新时只更新有值字段
- 避免手写大量字符串拼接

### 8.3 高频标签说明

#### 8.3.1 `<if>`

根据条件动态追加 SQL 片段。

#### 8.3.2 `<where>`

- 自动补 `where`
- 自动去掉多余的 `and` / `or`

#### 8.3.3 `<set>`

- 常用于更新语句
- 自动去掉最后多余的逗号

#### 8.3.4 `<foreach>`

- 常用于 `in` 查询
- 批量插入
- 批量删除

### 8.4 动态 SQL 的风险

- XML 容易越来越复杂
- 逻辑过重时可读性差
- if/foreach 嵌套多了不利于维护

实践建议：

- 复杂 SQL 适当拆分
- 把业务逻辑放在 Java 层，不要把 XML 写成“脚本语言”
- 动态排序、动态表名要重点防注入

---

## 9. `resultType` 和 `resultMap` 的区别

### 9.1 `resultType`

- 适合字段名和对象属性名基本一致的简单映射
- 使用简单，配置量少

### 9.2 `resultMap`

- 适合复杂映射
- 支持：
  - 字段名与属性名不一致
  - 一对一关联
  - 一对多关联
  - 嵌套对象映射

### 9.3 怎么选

- 简单单表查询：优先 `resultType`
- 复杂对象映射：使用 `resultMap`

### 9.4 面试高频点

如果问：**为什么还需要 `resultMap`？**

可以答：

> 因为数据库字段名和 Java 属性名并不一定完全一致，简单场景下 `resultType` 就够用，但遇到字段别名、关联对象、集合映射时，就需要 `resultMap` 做更精细的映射控制。

---

## 10. MyBatis 一级缓存和二级缓存

### 10.1 一级缓存

- 默认开启
- 作用域是 `SqlSession`
- 同一个 `SqlSession` 中，相同查询条件可能直接命中缓存

一级缓存失效常见场景：

- 不同 `SqlSession`
- 同一个 `SqlSession` 中执行了增删改
- 手动清空缓存
- 查询条件不同

### 10.2 二级缓存

- 作用域是 `Mapper` 级别 / namespace 级别
- 多个 `SqlSession` 可共享
- 默认不开启，需要显式配置

### 10.3 为什么很多项目不用二级缓存

- 缓存一致性难控制
- 分布式环境下问题更多
- 更新频繁业务命中率不高
- 通常更倾向使用 Redis 这类统一缓存体系

### 10.4 面试怎么答

> MyBatis 一级缓存默认开启，作用域是 `SqlSession`，本质是会话级缓存；二级缓存是 namespace 级别，需要手动开启。实际项目里一级缓存比较常见，二级缓存因为一致性和维护成本问题，很多团队不会重度依赖，而是使用 Redis 等更通用的缓存方案。

---

## 11. MyBatis 插件机制

### 11.1 插件本质

MyBatis 支持通过插件机制增强核心流程，本质是对四大对象做拦截。

可拦截对象：

- `Executor`
- `StatementHandler`
- `ParameterHandler`
- `ResultSetHandler`

### 11.2 常见应用场景

- 分页插件
- SQL 日志打印
- 慢 SQL 监控
- 多租户拦截
- 数据权限拦截
- 自动字段填充

### 11.3 分页插件 PageHelper 为什么能工作

本质上就是通过拦截 SQL 执行过程，对原 SQL 做改写，增加分页语句。

---

## 12. MyBatis 与 Spring 的集成

### 12.1 集成后谁管理 `SqlSession`

在 Spring 环境中，一般不是开发者手动管理 `SqlSession`，而是交给 Spring 和 MyBatis-Spring 整合模块管理。

### 12.2 Mapper 为什么能注入到 Spring 容器

- 因为配置了 Mapper 扫描
- Spring 启动时会把 Mapper 接口注册成 Bean
- 实际注入的是代理对象

常见方式：

- `@Mapper`
- `@MapperScan`

### 12.3 Spring 事务和 MyBatis 怎么配合

- 事务主要由 Spring 管理
- MyBatis 使用的连接会参与 Spring 事务
- 在同一个事务中，多个 Mapper 操作可以共用同一连接

面试表达：

> 在 Spring 项目中，一般由 Spring 接管事务和连接管理，MyBatis 负责 SQL 映射与执行。MyBatis-Spring 会把 `SqlSession` 与当前线程、当前事务绑定起来，所以在 `@Transactional` 方法里多个 Mapper 调用可以处于同一个事务上下文。

---

## 13. MyBatis 常见执行器

### 13.1 `SimpleExecutor`

- 默认执行器
- 每次执行都会创建新的 Statement

### 13.2 `ReuseExecutor`

- 会复用 Statement
- 适合重复执行相同 SQL 的场景

### 13.3 `BatchExecutor`

- 批量执行器
- 适合批量插入、批量更新
- 最终统一提交

### 13.4 面试注意点

- 批处理不等于一定高性能，还要看 JDBC 驱动、数据库支持、批次大小
- 批量操作要注意内存占用和事务大小

---

## 14. MyBatis 懒加载

### 14.1 是什么

- 在关联对象查询时，不是一次性全部加载
- 只有真正访问关联属性时才触发查询

### 14.2 优点

- 减少无意义查询
- 提升部分场景性能

### 14.3 缺点

- 可能导致 N+1 查询问题
- 调试时不容易第一时间看出 SQL 数量

### 14.4 面试表达

> MyBatis 支持懒加载，适用于关联对象不一定会被访问的场景，可以减少初次查询压力。但如果关联数据很多、访问频繁，或者一层层触发子查询，就容易形成 N+1 问题，所以要结合实际场景使用。

---

## 15. MyBatis 中常见面试问题

### 15.1 MyBatis 和 Hibernate/JPA 有什么区别

| 对比项 | MyBatis | Hibernate / JPA |
|---|---|---|
| SQL 控制权 | 开发者手写，控制力强 | 框架自动生成较多 |
| 学习重点 | SQL + 映射配置 | ORM 思想 + 实体关系 |
| 复杂查询支持 | 更灵活 | 复杂 SQL 通常不如手写直接 |
| 开发效率 | 简单场景一般 | 纯 CRUD 往往更高 |
| 性能调优 | 更直接 | 需要理解 ORM 行为 |

一句话：

- **MyBatis 更适合“SQL 驱动型”项目**
- **Hibernate/JPA 更适合“对象模型驱动型”项目**

### 15.2 MyBatis 会不会有 SQL 注入风险

- 用 `#{}` 时风险较低，因为底层走预编译
- 用 `${}` 时有明显风险
- 动态排序、动态表名是高风险点

### 15.3 MyBatis 为什么比 JDBC 开发效率高

- 去掉了大量模板代码
- 自动做参数绑定和结果映射
- 支持 Mapper 接口代理
- 支持动态 SQL 和插件扩展

### 15.4 MyBatis 为什么说是半自动 ORM

- 自动的是：参数映射、结果映射、对象封装
- 非自动的是：SQL 仍然主要由开发者自己写

### 15.5 MyBatis 有哪些不足

- SQL 多时维护成本上升
- 复杂 XML 可读性下降
- 强依赖开发者 SQL 能力
- 对复杂对象关系没有 Hibernate 那样强的自动管理能力

---

## 16. 项目实践中的高频问题

### 16.1 如何避免 XML 过于臃肿

- 按模块拆分 Mapper
- 抽取公共 SQL 片段
- 复杂查询适度用 SQL Builder 或代码生成工具
- 控制动态 SQL 复杂度

### 16.2 如何防止慢 SQL 问题

- 不要只关注 MyBatis，要看真正执行到数据库的 SQL
- 打印完整 SQL 和参数
- 配合执行计划分析索引命中情况
- 避免在 XML 中写过度复杂的动态查询

### 16.3 MyBatis 分页要注意什么

- 大分页性能差，本质是数据库问题，不是框架问题
- 深分页可以考虑游标分页、延迟关联、覆盖索引等优化
- 分页插件只是改写 SQL，不会自动解决分页性能问题

### 16.4 批量插入为什么可能不快

- 单次批量过大，SQL 太长
- JDBC 驱动未正确开启批处理优化
- 事务过大导致锁和日志压力上升
- 索引过多影响写入性能

---

## 17. 面试回答模板

### 17.1 什么是 MyBatis

> MyBatis 是一个半自动 ORM 持久层框架，本质上是对 JDBC 的封装。它允许开发者自己编写 SQL，再由框架完成参数绑定、SQL 执行和结果集映射。它的优点是 SQL 可控、灵活，特别适合复杂查询和性能调优场景。

### 17.2 MyBatis 的执行流程

> MyBatis 启动时先解析全局配置和 Mapper 映射文件，把每条 SQL 封装成 `MappedStatement`。业务调用 Mapper 接口时，实际调用的是动态代理对象。代理对象根据方法找到对应 SQL，交给 `SqlSession` 和 `Executor` 去执行。执行过程中会完成参数绑定、Statement 处理和结果集映射，最终返回 Java 对象。

### 17.3 `#{}` 和 `${}` 的区别

> `#{}` 是预编译参数占位符，底层会变成 `?`，通过 `PreparedStatement` 绑定参数，能够防止 SQL 注入；`${}` 是字符串直接拼接，不会预编译，存在注入风险。通常普通参数都用 `#{}`，只有动态表名、列名、排序字段这类场景才谨慎使用 `${}`。

### 17.4 一级缓存和二级缓存

> MyBatis 一级缓存是 `SqlSession` 级别，默认开启；二级缓存是 namespace 级别，需要手动配置。一级缓存更像会话内优化，二级缓存能跨会话共享，但实际项目里因为一致性和维护成本问题，很多团队更倾向使用 Redis 这类统一缓存体系。

### 17.5 MyBatis 和 Hibernate 的区别

> MyBatis 更偏向 SQL 驱动，开发者自己掌控 SQL，适合复杂查询和性能调优；Hibernate/JPA 更偏向对象关系映射，适合通过实体模型驱动开发。简单 CRUD 场景下 Hibernate/JPA 开发效率可能更高，但复杂 SQL 场景下 MyBatis 往往更灵活。

### 17.6 Mapper 接口为什么不需要实现类

> 因为 MyBatis 会基于 Mapper 接口生成动态代理对象。我们调用接口方法时，实际进入的是代理逻辑，代理再根据接口名、方法名和参数信息找到对应的 SQL，也就是对应的 `MappedStatement`，最后交给 `SqlSession` 和 `Executor` 去执行。

### 17.7 MyBatis 为什么是半自动 ORM

> 因为它只帮我们做了对象和 SQL 执行过程之间的映射，比如参数绑定、结果集封装、Mapper 代理等；但 SQL 本身仍然需要开发者自己编写，所以它不是全自动 ORM，而是半自动 ORM。

### 17.8 `resultType` 和 `resultMap` 的区别

> `resultType` 适合简单结果映射，要求查询结果字段和 Java 对象属性基本能对应上；`resultMap` 适合复杂映射，能处理字段名不一致、嵌套对象、一对多等场景。简单查询优先用 `resultType`，复杂对象映射用 `resultMap`。

### 17.9 MyBatis 插件机制原理

> MyBatis 的插件机制本质上是责任链加动态代理，它允许我们对四大对象进行拦截，分别是 `Executor`、`StatementHandler`、`ParameterHandler`、`ResultSetHandler`。常见场景包括分页、SQL 监控、多租户、数据权限等。

### 17.10 Spring 事务和 MyBatis 是怎么整合的

> 在 Spring 项目中，一般由 Spring 管理事务和数据库连接，MyBatis 负责 SQL 映射与执行。通过 MyBatis-Spring 整合后，`SqlSession` 会和当前线程、当前事务绑定，所以在同一个 `@Transactional` 方法里多个 Mapper 调用通常处于同一个事务上下文。

### 17.11 为什么很多项目不太用 MyBatis 二级缓存

> 因为二级缓存虽然能跨 `SqlSession` 共享数据，但一致性控制比较麻烦，尤其在分布式环境、更新频繁场景下容易出问题。相比之下，很多团队更愿意把缓存统一交给 Redis 之类的中间件处理，所以 MyBatis 二级缓存在真实项目里不算特别常用。

### 17.12 MyBatis 的优缺点怎么答

> MyBatis 的优点是 SQL 可控、灵活、适合复杂查询和性能优化，也很适合和 Spring 体系整合；缺点是 SQL 需要自己维护，Mapper 和 XML 多了以后维护成本会升高，而且对开发者 SQL 能力要求比较高。

---

## 18. 高频易错点 / 面试陷阱

- 不要把 MyBatis 说成“数据库连接池”。
- 不要把 MyBatis 说成“全自动 ORM”。
- 不要说 `${}` 也能防 SQL 注入。
- 一级缓存是 `SqlSession` 级别，不是整个应用级别。
- 二级缓存不是默认就能安全高效使用。
- 分页慢的根因通常在 SQL 和数据库，不在 MyBatis 本身。
- 懒加载能优化查询，但也可能引入 N+1 问题。
- 批处理能提效，但不是无脑越大越好。

---

## 19. 面试八股版速答清单

这一部分适合面试前集中背诵。

### 19.1 一问一答版

#### 1）MyBatis 是什么？

答：

- 持久层框架
- JDBC 的封装
- 半自动 ORM
- 核心价值是 SQL 映射、参数绑定、结果封装

#### 2）为什么说 MyBatis 是半自动 ORM？

答：

- 自动：参数映射、结果映射、对象封装
- 非自动：SQL 需要开发者手写

#### 3）MyBatis 的核心组件有哪些？

答：

- `SqlSessionFactory`
- `SqlSession`
- `Executor`
- `MappedStatement`
- `MapperProxy`

#### 4）MyBatis 四大对象是什么？

答：

- `Executor`
- `StatementHandler`
- `ParameterHandler`
- `ResultSetHandler`

#### 5）Mapper 接口为什么可以直接调用？

答：

- 因为 MyBatis 使用 JDK 动态代理生成 Mapper 代理对象
- 调用方法时实际执行的是代理逻辑，不是接口本身

#### 6）`#{}` 和 `${}` 的区别？

答：

- `#{}`：预编译，占位符，防 SQL 注入
- `${}`：字符串拼接，不预编译，有注入风险

#### 7）一级缓存和二级缓存的区别？

答：

- 一级缓存：`SqlSession` 级别，默认开启
- 二级缓存：namespace 级别，需要手动开启

#### 8）为什么很多项目不用二级缓存？

答：

- 一致性难控制
- 分布式场景复杂
- 很多项目统一用 Redis

#### 9）`resultType` 和 `resultMap` 的区别？

答：

- `resultType` 适合简单映射
- `resultMap` 适合复杂映射和关联对象映射

#### 10）MyBatis 插件拦截的是什么？

答：

- 拦截四大对象：`Executor`、`StatementHandler`、`ParameterHandler`、`ResultSetHandler`

#### 11）MyBatis 和 Hibernate/JPA 的区别？

答：

- MyBatis：SQL 驱动，灵活，可控
- Hibernate/JPA：对象驱动，自动化程度更高

#### 12）MyBatis 适合什么场景？

答：

- 复杂查询多
- 性能优化要求高
- 希望精细控制 SQL

### 19.2 快问快答版

- MyBatis 本质上是什么？→ JDBC 封装。
- MyBatis 是全自动 ORM 吗？→ 不是，是半自动 ORM。
- Mapper 为什么不用实现类？→ 动态代理。
- `#{}` 底层是什么？→ `PreparedStatement` 占位符 `?`。
- `${}` 为什么危险？→ 字符串直拼，存在 SQL 注入。
- 一级缓存作用域？→ `SqlSession`。
- 二级缓存作用域？→ namespace / Mapper。
- 分页插件本质？→ 拦截 SQL 并改写。
- MyBatis 优势？→ SQL 可控、适合复杂查询。
- MyBatis 劣势？→ SQL 维护成本高。

### 19.3 背诵浓缩版

> MyBatis 是一个半自动 ORM 持久层框架，本质是对 JDBC 的封装。它通过 Mapper 动态代理、参数绑定和结果映射，简化了数据库访问流程。它的优势是 SQL 可控、灵活，适合复杂查询和性能优化；高频考点包括执行流程、`#{}` 和 `${}`、缓存机制、动态 SQL、插件机制以及与 Spring 的整合。

---

## 20. 一页速记版

### 20.1 核心定位

- MyBatis = **持久层框架 + JDBC 封装 + 半自动 ORM**
- 特点：**SQL 自己写，映射交给框架**

### 20.2 核心组件

- `SqlSessionFactory`
- `SqlSession`
- `Executor`
- `MappedStatement`
- `MapperProxy`

### 20.3 四大对象

- `Executor`
- `StatementHandler`
- `ParameterHandler`
- `ResultSetHandler`

### 20.4 高频考点

- `#{}` vs `${}`
- 一级缓存 vs 二级缓存
- `resultType` vs `resultMap`
- MyBatis 执行流程
- Mapper 动态代理原理
- MyBatis vs Hibernate/JPA
- 动态 SQL 标签
- 插件机制

### 20.5 面试关键词

- 半自动 ORM
- JDBC 封装
- 动态代理
- 预编译
- 参数绑定
- 结果映射
- 动态 SQL
- 一级缓存
- 插件拦截
- SQL 可控

---

## 21. 结论

如果让你用一句话总结 MyBatis：

> MyBatis 是一个以 SQL 为中心的持久层框架，适合对数据库访问过程有较强控制需求的业务系统，核心优势是灵活、可控、易于做复杂查询和性能优化。

如果让你再补一句项目经验：

> 实际项目里我们通常把 MyBatis 和 Spring 事务、连接池、分页插件、Redis 缓存等一起配合使用，MyBatis 负责 SQL 映射和执行，真正的性能瓶颈往往还是 SQL 本身和数据库设计。
