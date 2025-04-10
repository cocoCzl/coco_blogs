## SpringBoot启动异常
SpringBoot启动报错：java.lang.ClassNotFoundException：ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP异常
SpringBoot 3.3.7
```xml
<dependency>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-parent</artifactId>
<version>3.3.7</version>
</dependency>
```
使用的默认logback：
```xml
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-parent</artifactId>
    <version>1.5.12</version>
</dependency>
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-parent</artifactId>
    <version>1.5.12</version>
</dependency>
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