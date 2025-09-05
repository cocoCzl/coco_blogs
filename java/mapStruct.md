## 实践
### 依赖
```xml
<!-- https://mvnrepository.com/artifact/org.mapstruct/mapstruct -->
<dependency>
  <groupId>org.mapstruct</groupId>
  <artifactId>mapstruct</artifactId>
  <version>1.6.3</version>
</dependency>

<dependency>
  <groupId>org.mapstruct</groupId>
  <artifactId>mapstruct-processor</artifactId>
  <version>1.6.3</version>
  <scope>provided</scope>
</dependency>

插件
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.8.1</version>
        <configuration>
          <source>1.8</source>
          <target>1.8</target>
          <annotationProcessorPaths>
            <!-- Lombok -->
            <path>
              <groupId>org.projectlombok</groupId>
              <artifactId>lombok</artifactId>
              <version>1.18.22</version>
            </path>
            <!-- MapStruct -->
            <path>
              <groupId>org.mapstruct</groupId>
              <artifactId>mapstruct-processor</artifactId>
              <version>1.6.3</version>
            </path>
          </annotationProcessorPaths>
        </configuration>
      </plugin>
```

需要注意Lombok插件必须放在MapStruct前面

#### 为什么用这个顺序
+ 编译阶段，`annotationProcessorPaths` 里的处理器会一起跑。
+ **但是** Lombok 是 _修改 AST_（在编译过程中生成 getter/setter、构造器等方法），  
  MapStruct 是 _分析 AST_（需要看到 getter/setter 才能生成映射代码）。
+ 如果 Lombok 没先把方法“补上”，MapStruct 在分析时就会发现 **DTO/Entity 上没有对应的 getter/setter**，于是生成的实现类就只会 `new DTO()` 而没有赋值。
+ 所以 Lombok 要放在前面，保证它先把代码补全，再轮到 MapStruct 分析。

### 类
可以建一个Base接口

```java
public interface BaseMapper<S, T> {

    // 单对象映射
    T toDto(S entity);

    // 反向映射
    S toEntity(T dto);

    // 列表映射
    List<T> toDtoList(List<S> entityList);

    List<S> toEntityList(List<T> dtoList);
}
```
