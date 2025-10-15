## SpringBoot启动异常
SpringBoot启动报错：java.lang.ClassNotFoundException：ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP异常

SpringBoot 3.3.7

```xml
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-parent</artifactId>
<version>3.3.7</version>
```

使用的默认logback：

```xml
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-parent</artifactId>
    <version>1.5.12</version>

    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-parent</artifactId>
    <version>1.5.12</version>
```

解决方式：在logback的xml将SizeAndTimeBasedFNATP替换为SizeAndTimeBasedFileNamingAndTriggeringPolicy



## Spring Boot + JPA主键生成问题
问题：

项目使用的是SpringBoot JAP和Mysql，有很多表的实体类都写了

```java
@Id
@GeneratedValue(strategy = GenerationType.AUTO)
@Column(name = "id")
```

这些主键ID的值好像被共享了，如果我新建一个表然后使用JAP插入一条数据，ID会直接从一个很大的数据开始。

原因：

```java
@Id
@GeneratedValue(strategy = GenerationType.AUTO)
@Column(name = "id")
private int id;
```

这个配置告诉 JPA 自动生成主键，**但具体的主键生成策略依赖于数据库和 JPA 实现（Hibernate）如何解释 **`**GenerationType.AUTO**`。

在 MySQL 中，Hibernate 通常会将 `AUTO` 映射为 `GenerationType.IDENTITY`，即使用数据库的 `AUTO_INCREMENT` 功能。但 Hibernate 也可能使用一个**全局的序列生成器（尤其是在一些 JPA provider 配置或者使用某些 dialect 时）**，从而导致所有表的主键 ID 是共享的。

解决办法：

#### 方案一：显式使用 `GenerationType.IDENTITY`（推荐 MySQL）
```plain
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
@Column(name = "id")
private int id;
```

`IDENTITY` 直接使用 MySQL 的 `AUTO_INCREMENT`，每个表的主键互不影响。

这种情况下，每个表的数据从 1 开始，自增，不会“共享 ID”。

#### 方案二：使用 `GenerationType.SEQUENCE` + 每个表独立序列（不推荐 MySQL）
这种方式通常用于 PostgreSQL 或 Oracle 等支持序列的数据库，在 MySQL 中要实现这个，需要手动创建序列表，不太适合。

### 能不能让 JPA 的 Hibernate 在使用 MySQL + `GenerationType.AUTO` 时，不使用全局序列生成器？
**在 MySQL 下，Hibernate 默认并不会用全局序列生成器，而是根据配置和版本不同，有时会退化为 **`**IDENTITY**`**，但它的行为并不一定可预测，特别是 AUTO 的语义是“由 JPA provider 自行决定”。**

**你不能“强制” Hibernate 在 **`**AUTO**`** 下使用某种特定策略（比如每表独立），除非你自己指定具体的策略，如 **`**IDENTITY**`**。**

#### `**GenerationType.AUTO**`** 背后的行为**
+ **它是个“智能”策略：由 JPA 实现（比如 Hibernate）****根据底层数据库能力自动选择合适的主键生成方式****。**
+ **在 MySQL 中：**
    - **Hibernate 早期版本会将 **`**AUTO**`** 映射为 **`**IDENTITY**`**。**
    - **但在部分 Hibernate 版本 + Spring Boot 中，也可能采用一种叫 **`**enhanced-sequence**`** 的策略，这种策略使用一个表（hibernate_sequence）来生成所有实体的主键值，导致“全局共享序列”。**

### **其他建议（可选）**
#### **修改 application 配置文件，设置 Hibernate 不要使用 sequence-style generator：**
```plain
spring:
  jpa:
    properties:
      hibernate:
        id:
          new_generator_mappings: false
```

`**hibernate.id.new_generator_mappings=false**`** 这个配置会让 Hibernate 回退为老式的主键生成策略，在某些 Hibernate 版本中能阻止它使用 **`**hibernate_sequence**`**。**

**但是这个方法不是很推荐，因为它依赖 Hibernate 内部行为，不如直接使用 **`**IDENTITY**`** 明确可靠。**

### **需要同时支持PG和Mysql**
**项目需要同时支持 MySQL 和 PostgreSQL**，并且你希望统一使用 `@GeneratedValue(strategy = GenerationType.AUTO)`，但又不想出现 ID 全局共享的问题。

那我们就要兼顾：

+ MySQL 用 `AUTO_INCREMENT`
+ PostgreSQL 用 `SEQUENCE`
+ 避免 Hibernate 在 `AUTO` 下使用全局 `hibernate_sequence`

#### 建议的做法：**使用 **`**GenerationType.AUTO**`** + 每个实体单独指定序列（适配 PG）+ MySQL 自动忽略序列**
步骤如下：

第一步：给每个实体添加独立的序列生成器

```plain
@Id
@GeneratedValue(strategy = GenerationType.AUTO, generator = "your_entity_seq")
@SequenceGenerator(name = "your_entity_seq", sequenceName = "your_entity_seq", allocationSize = 1)
@Column(name = "id")
private Long id;
```

+ `sequenceName` 是 PG 下用的序列名
+ `allocationSize=1` 是为了让它和 `AUTO_INCREMENT` 的行为一致（每次递增1）
+ 对于 MySQL，`@SequenceGenerator` 不起作用，会被忽略，依然用 `AUTO_INCREMENT`（只要 `AUTO` 被映射成 `IDENTITY`）

---

第二步：配置 `hibernate.id.new_generator_mappings=false`

这个配置可以让 Hibernate 对 `AUTO` 的行为更“老派”，在 PG 下避免使用全局的 `hibernate_sequence`：

```plain
spring:
  jpa:
    properties:
      hibernate:
        id:
          new_generator_mappings: false
```

## JAP事务Can't call commit when autocommit=true异常
### 环境
SpringBoot+JPA+Druid+Mysql

Mysql连接串：jdbc:mysql://ip:port/ops_dev?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=Asia/Shanghai

### 问题
Druid没有设置default-auto-commit: false，所以是事务是自动提交的，然后有些方法加上了@Transactional注解，使用了Spring的事务注解，开启了事务，但是偶发性的出现：Caused by: java.sql.SQLException: Can't call commit when autocommit=true异常。而且MySQL 5 没问题，MySQL 8 就有问题。

### 结论
autoReconnect=true导致

#### 为什么 MySQL 5 没问题，MySQL 8 就有问题？
##### **1. **`**autoReconnect=true**`** 的“黑历史”**
这个参数在 MySQL 的 JDBC 驱动中一直是一个“不推荐使用”的选项。它的设计初衷是让驱动本身在发现连接断开时，能悄悄地帮你重新连上。

+ **在老的 MySQL 5.x 驱动中**：它的行为相对“可预测”一些。虽然不推荐，但在某些简单场景下，它可能“恰好”能工作，或者它在失败时抛出的异常能被上层框架（如 Druid）正确捕获和处理。
+ **在新的 MySQL 8.x 驱动 (**`**mysql-connector-java-8.x**`**) 中**：
    - `autoReconnect=true`**已被正式废弃，并且其行为变得非常不可靠**。
    - 当连接因为超时被服务器切断后，驱动可能会尝试重连。**关键在于**：重连成功后，新的数据库会话（Session）会恢复到**数据库的默认设置**。
    - 数据库的默认设置通常是 `autocommit=true`。
    - 这就完美解释了你的问题：Spring 事务管理器以为它已经把连接设置成了 `setAutoCommit(false)`，但中途连接断了，驱动悄悄重连了，这个连接的状态被重置回了 `autocommit=true`。当 Spring 最后执行 `commit()` 时，就收到了致命的 `Can't call commit when autocommit=true` 异常。

##### **2. 现代连接池 vs. **`**autoReconnect**`
现代的数据库连接池（如 Druid, HikariCP）的设计理念是：**由连接池全权负责管理连接的生命周期**。

+ **连接池的职责**：创建连接、租借连接、回收连接、**检测连接的有效性**、销毁无效连接。
+ `**autoReconnect=true**`** 的问题**：它让 JDBC 驱动本身也去插手连接的管理。这就好比一个团队里有两个老板，都想说了算，最终一定会导致混乱。

当 `autoReconnect=true` 和 Druid 一起工作时：

+ Druid 以为自己掌控着连接，但驱动可能在“背后”偷偷地把连接给换掉了（重连）。
+ Druid 无法感知到这种“偷梁换柱”，也就无法知道连接的内部状态（比如 `autocommit` 模式）已经被重置了。
+ 最终，当应用程序（通过 Spring）使用这个状态不一致的连接时，就会爆出各种奇怪的错误。

**结论：**`**autoReconnect=true**`** 和 Druid（以及其他任何现代连接池）是天生冲突的。你不应该同时使用它们。**

#### **Spring事务**
### 一个事务操作的真实过程
以 Spring `@Transactional` 方法为例：

1. 方法进入事务拦截器 → Spring 开启事务；
2. Spring 向 DataSource 要一个连接；
3. Spring 调用 `conn.setAutoCommit(false)`；
4. 你执行各种 SQL；
5. Spring 调用 `conn.commit()`；
6. Spring 调用 `conn.setAutoCommit(true)`；
7. Spring 把连接交还给 Druid 连接池。

---

#### 举例
##### 代码
```java
@Configuration
@EnableTransactionManagement
@EnableJpaRepositories(entityManagerFactoryRef = "entityManagerFactoryPrimary", transactionManagerRef = "transactionManagerPrimary", basePackages = {
    "xxx.xxx.xx.repo",
    "xxx.xxx.repo"}) //设置Repository所在位置
public class PrimaryConfig {

    @Autowired
    @Qualifier("druidDataSource")
    private DataSource druidDataSource;
    @Resource
    private JpaProperties jpaProperties;
    @Resource
    private HibernateProperties hibernateProperties;

    private Map<String, Object> getVendorProperties() {

        return hibernateProperties.determineHibernateProperties( jpaProperties.getProperties(), new HibernateSettings());
    }

    @Primary
    @Bean(name = "entityManagerFactoryPrimary")
    public LocalContainerEntityManagerFactoryBean entityManagerFactoryPrimary(
        EntityManagerFactoryBuilder builder) {
        return builder.dataSource(druidDataSource)
        .packages("xxx.xxx.entity",
                  "xxx.xxx.xx.entity") // 设置实体类所在位置
        .persistenceUnit("primaryPersistenceUnit")
        .properties(getVendorProperties())
        .build();
    }

    @Primary
    @Bean(name = "transactionManagerPrimary")
    public PlatformTransactionManager transactionManagerPrimary(EntityManagerFactoryBuilder builder) {
        return new JpaTransactionManager(
            Objects.requireNonNull(entityManagerFactoryPrimary(builder).getObject()));
    }

}
```

注册 `EntityManagerFactory`、`EntityManager`、和 `PlatformTransactionManager`，并告诉 Spring Data JPA 哪些包下的 `Repository` 使用这个工厂/事务管理器。

---

```java
@Configuration
@EnableTransactionManagement
@EnableJpaRepositories(
  entityManagerFactoryRef = "entityManagerFactoryPrimary",
  transactionManagerRef = "transactionManagerPrimary",
  basePackages = { ... }
)
```

+ `@Configuration`：声明这是一个 Spring 配置类（等同于 XML 配置）。
+ `@EnableTransactionManagement`：启用注解事务支持（使 `@Transactional` 生效）。
+ `@EnableJpaRepositories(...)`：启用 Spring Data JPA 仓库，重要参数：
+ `entityManagerFactoryRef = "entityManagerFactoryPrimary"`：指明这些 Repository 使用名为 `entityManagerFactoryPrimary` 的 `EntityManagerFactory`。
+ `transactionManagerRef = "transactionManagerPrimary"`：指明使用名为 `transactionManagerPrimary` 的事务管理器（`PlatformTransactionManager`）。
+ `basePackages`：扫描这些包下的 interface（`JpaRepository` / `CrudRepository`）并创建实现 bean。**这一步非常关键**，如果某个 repository 不在这些包内，就不会使用这个配置（可能导致事务/数据源不匹配）。

---

```java
private Map<String, Object> getVendorProperties() {
    return hibernateProperties.determineHibernateProperties(jpaProperties.getProperties(), new HibernateSettings());
}
```

+ 作用：把 Spring Boot 的 `spring.jpa.*` 和 `spring.jpa.properties.*` 等配置解析成给 Hibernate 的属性 `Map<String,Object>`，之后会传给 `LocalContainerEntityManagerFactoryBean`。
+ 注意：如果你需要额外设置 Hibernate 属性（比如 `hibernate.hbm2ddl.auto`、`hibernate.dialect`、`hibernate.format_sql` 等），可以在这里 `put` 进去或在 `application.yml` 里配置。

---

创建 EntityManagerFactory

```java
@Primary
@Bean(name = "entityManagerFactoryPrimary")
public LocalContainerEntityManagerFactoryBean entityManagerFactoryPrimary(EntityManagerFactoryBuilder builder) {
    return builder
            .dataSource(druidDataSource)
            .packages( /* 实体类包 */ )
            .persistenceUnit("primaryPersistenceUnit")
            .properties(getVendorProperties())
            .build();
}
```

+ `EntityManagerFactoryBuilder`：Spring Boot 提供的辅助构造器，封装了 `JpaVendorAdapter`、数据源、属性等，最终返回 `LocalContainerEntityManagerFactoryBean`。
+ `.dataSource(druidDataSource)`：指定底层数据源（你传入的 Druid）。
+ `.packages(...)`：扫描这些包下的 `@Entity`，将它们交给这个 `EntityManagerFactory` 管理。**实体类必须在这些包下**，否则 JPA 无法映射表。
+ `.persistenceUnit("primaryPersistenceUnit")`：设定持久单元名，通常用于多持久单元/多数据源场景以区分。
+ `.properties(getVendorProperties())`：传入 Hibernate 的配置项。
+ 返回类型 `LocalContainerEntityManagerFactoryBean`：这是一个 Bean 工厂（FactoryBean），Spring 会通过它创建实际的 `EntityManagerFactory`（`getObject()`）。

**注意点**：

+ `@Primary` 标注意味着在存在多个 `EntityManagerFactory` 时，这个 bean 会优先注入（如果注入点没有指定 bean 名称）。
+ `packages(...)` 与 `EnableJpaRepositories.basePackages` 要对应好：Repository 所在包应管理的实体在这个 factory 的 `packages` 中，否则查询可能失败或抛错找不到实体。

---

事务管理器

```java
@Primary
@Bean(name = "transactionManagerPrimary")
public PlatformTransactionManager transactionManagerPrimary(EntityManagerFactoryBuilder builder) {
    return new JpaTransactionManager(
        Objects.requireNonNull(entityManagerFactoryPrimary(builder).getObject()));
}
```

+ 创建 `JpaTransactionManager`，并注入对应的 `EntityManagerFactory`。
+ `JpaTransactionManager` 是基于 JDBC 的本地事务管理器（对 JPA 提供事务边界）。当 `@Transactional` 生效时，Spring 会通过这个 `PlatformTransactionManager`：
    - 获取连接（或使用 `EntityManagerFactory` 的 `EntityManager`）；
    - 将 `autoCommit` 设为 `false`（以开始事务）；
    - 在方法退出时 `commit()` 或 `rollback()`。
+ `@Primary`：当多个 `PlatformTransactionManager` 存在且 `@Transactional` 没有显式指定 `transactionManager` 时，将使用这个作为默认事务管理器。

**注意：**

+ 如果存在多个数据源/事务管理器，必须在 `@Transactional` 上指定 `transactionManager = "transactionManagerPrimary"`（否则可能被错误的事务管理器管理）。
+ `JpaTransactionManager` 内部会在 commit 时调用底层 JDBC 的 `Connection.commit()`。

---

###### 整体运行时关系（流程）
1. Spring 容器创建 `druidDataSource`（别处配置）。
2. Spring 使用这个配置类构建 `LocalContainerEntityManagerFactoryBean`，并最终得到 `EntityManagerFactory`。
3. Spring 创建 `JpaTransactionManager`，绑定上面得到的 `EntityManagerFactory`。
4. 被 `@EnableJpaRepositories` 扫描到的 Repository 会使用上面的 `EntityManagerFactory` 与 `transactionManagerPrimary`。
5. 当某个带 `@Transactional`（或 Repository 接口方法）被调用时：
    - `JpaTransactionManager` 获取/绑定一个 `EntityManager`（或 Connection）到当前线程；
    - 设置 JDBC Connection 的 `autoCommit=false`（开始事务）；
    - 执行方法体；
    - 最后 commit/rollback 并释放连接。



| 组件 | 主要职责 | 典型Bean名称 | 关系 |
| --- | --- | --- | --- |
| `DataSource` | 负责连接数据库 | `primaryDataSource`<br/> / `secondaryDataSource` | 一个数据库一个数据源 |
| `EntityManagerFactory` | 负责管理实体和数据库交互 | `entityManagerFactoryPrimary`<br/> / `entityManagerFactorySecondary` | 依赖对应的 DataSource |
| `TransactionManager` | 管理事务提交与回滚 | `transactionManagerPrimary`<br/> / `transactionManagerSecondary` | 依赖对应的 EntityManagerFactory |


| 层级 | Spring/JPA对象 | 职责 | 创建时机 |
| --- | --- | --- | --- |
| ① | `LocalContainerEntityManagerFactoryBean` | Spring 的 **工厂 Bean**，是 `EntityManagerFactory`<br/> 的配置载体 | Spring 容器启动时创建 |
| ② | `EntityManagerFactory` | **实体管理器工厂**，负责生产 `EntityManager` | Spring 启动时由上面的 Bean 初始化 |
| ③ | `EntityManager` | **真正操作数据库的对象**，执行增删改查、事务等 | 每次请求/事务中按需创建 |


```plain
Spring 容器
│
│---> LocalContainerEntityManagerFactoryBean (Spring 工厂Bean)
│           │
│           ▼
│    EntityManagerFactory (Hibernate SessionFactory)
│           │
│           ▼
│    EntityManager (Hibernate Session)
│
│---> JpaTransactionManager (使用 EntityManagerFactory 进行事务控制)
```

---

##### 背后机制
**每次进入一个 **`**@Transactional**`** 方法时，Spring 会从 **`**EntityManagerFactory**`** 拿到一个“事务级别专属的 EntityManager”**。  
在同一个事务中（无论调用多少次 Repository 方法）使用的都是同一个 `EntityManager`。  
但不同事务之间使用的 `EntityManager` 是不同的（不会共享）。

Spring 在启动时，会在容器里注册一个代理：

```plain
SharedEntityManagerCreator.createSharedEntityManager(EntityManagerFactory emf)
```

这个代理就是我们通过 `@Autowired EntityManager` 或 `JpaRepository` 间接使用的那个对象。  
它并不是一个真实的 `EntityManager`，而是一个 **线程绑定（ThreadLocal）代理对象**。

它的行为大概是这样的：

1. **进入 @Transactional 方法时**
    - Spring 的 `TransactionInterceptor` 启动事务。
    - `JpaTransactionManager` 会从 `EntityManagerFactory` 创建一个真正的 `EntityManager`。
    - 这个 `EntityManager` 会绑定到当前线程（ThreadLocal）。
2. **事务中的所有 Repository 操作**
    - 都通过代理拿到同一个 `EntityManager` 实例。
    - 因此同一个事务内的操作共享一级缓存（Persistence Context）。
3. **事务提交或回滚时**
    - `JpaTransactionManager` 会在事务结束后关闭该 `EntityManager`。
    - 同时从 `ThreadLocal` 移除绑定关系。

```plain
[EntityManagerFactory]  ——>  [EntityManager #1]  <——  @Transactional(事务1)
                               ↑
                               │同一事务内所有Repo共享
                               └──> persistence context 缓存

[EntityManagerFactory]  ——>  [EntityManager #2]  <——  @Transactional(事务2)
```
