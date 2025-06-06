SpringBoot整合Druid，如果希望自动注入的方式同时使配置生效

1. **<font style="color:rgb(36, 41, 46);">首先引入</font>**<font style="color:rgb(56, 58, 66);background-color:rgb(246, 248, 250);">starter</font>

```yaml
<!--引入druid数据源-->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid</artifactId>
            <version>1.2.6</version>
        </dependency>
```

2. **<font style="color:rgb(36, 41, 46);">yml 配置文件</font>**

**<font style="color:rgb(36, 41, 46);">需要加drudi才会生效</font>**

例子：

```yaml
spring:
  datasource:
    driver-class-name: XXXXX
    username: XXX
    password: XXX
    url: XXXX
    type: com.alibaba.druid.pool.DruidDataSource
    druid:
      validationQuery: SELECT 1 FROM DUAL
      keepAlive: true
      initialSize: 5
      minIdle: 5
      maxActive: 20
      maxWait: 60000
```

不生效例子：

```yaml
spring:
  datasource:
    driver-class-name: XXXXX
    username: XXX
    password: XXX
    url: XXXX
    type: com.alibaba.druid.pool.DruidDataSource
    validationQuery: SELECT 1 FROM DUAL
    keepAlive: true
    initialSize: 5
    minIdle: 5
    maxActive: 20
    maxWait: 60000
    druid:
```