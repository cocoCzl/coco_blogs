## 技术复盘报告：容器环境下网关下游服务不可达问题
### 1. 问题现象
+ **场景**：在 Docker/Rancher 容器环境中，下游服务（注册名为域名）重启后，IP 地址发生变更。
+ **故障**：网关服务（Spring Cloud Gateway）在数分钟内持续报错 `No route to host` 或 `Connection refused`，指向的依然是旧容器的 IP。
+ **无效尝试**：仅禁用 JVM DNS 缓存（`-Dnetworkaddress.cache.ttl=0`）和调整 Nacos 实时查询（`subscribe=false`）后，问题仍未解决。

### 2. 根因分析 (Root Cause Analysis)
这个问题的根源在于 **多层缓存机制的叠加**，特别是 **Reactor Netty 隐蔽的 DNS 缓存**。

我们在排查中剥开了四层“洋葱”：

1. **服务发现层（Nacos）**：
    - **问题**：Nacos 客户端默认缓存实例列表，且 UDP 推送在容器网络中不可靠。
    - **现状**：已通过 `subscribe=false` 解决，强制网关实时获取最新**域名**。
2. **Java 虚拟机层（JVM）**：
    - **问题**：JDK 默认无限缓存 DNS 解析结果。
    - **现状**：已通过 `-Dnetworkaddress.cache.ttl=0` 解决，要求 JVM 每次都向操作系统查询。
3. **连接传输层（Connection Pool）**：
    - **问题**：HTTP 长连接（Keep-Alive）复用了旧 TCP 连接，导致根本不会触发新的 DNS 解析。
    - **现状**：已通过 `max-life-time` 和 `type: elastic` 强制定期断连，触发重连。
4. **底层网络层（Reactor Netty）——【核心元凶】**：
    - **问题**：Spring Cloud Gateway 底层使用的 Reactor Netty 为了追求极致性能，**默认内置了一套独立的 DNS 解析器（非 JDK 实现）**。
    - **关键点**：这套内置解析器**忽略了** JVM 的 `ttl=0` 参数，并且自己维护了一份 DNS 缓存。
    - **结果**：即使 Nacos 给了新域名，JVM 也准备好了解析，Netty 却直接从自己的“私有小本本”里拿出了旧 IP，导致请求发往死胡同。

### 3. 问题定性
**这是一个“默认行为不匹配”的问题。**

+ **是 Netty 的问题吗？**
    - **不是 Bug**。Netty 设计内置 DNS 解析器是为了在高并发下避免阻塞，提升性能。在 IP 固定的物理机时代，这很完美。但在 IP 频繁变动的 Docker 时代，它的默认缓存策略显得“太固执”。
+ **是 Spring Cloud Gateway 的问题吗？**
    - **是适配问题**。SCG 默认集成了 Netty，但没有在默认配置中针对“动态容器环境”做 DNS 策略的自动调整，导致开发者必须手动介入配置。

### 4. 最终解决方案 (Solution)
为了彻底解决 IP 漂移问题，我们实施了 **“全链路去缓存”** 策略：

1. **强制 Netty 使用 JDK 解析器（核心修复）**： 注入 `HttpClientCustomizer`，将 Netty 的解析器替换为 `DefaultAddressResolverGroup.INSTANCE`。
    - _作用_：打通 Netty 与 JVM 的屏障，让 Netty 乖乖听从 JVM 的 `ttl=0` 配置。

```java
@Configuration
public class GatewayNettyConfig {

    // 强制 Netty 使用 JDK 默认的 DNS 解析器
    // -Dnetworkaddress.cache.ttl=0 才会对 Netty 生效！
    @Bean
    public HttpClientCustomizer useJdkDnsResolver() {
        return httpClient -> httpClient.resolver(DefaultAddressResolverGroup.INSTANCE);
    }
}
```

2. **禁用 JVM DNS 缓存**： 配置 `-Dnetworkaddress.cache.ttl=0`。
    - _作用_：让 Java 每次解析域名都去询问 Docker 的 DNS 服务器。

```bash
JAVA_OPTS: -Xms1024m -Xmx1024m -Dsun.net.inetaddr.ttl=0 -Dnetworkaddress.cache.ttl=0
```

3. **主动管理连接池**： 配置 `max-life-time: 15s` 和 `evict-background-interval: 5s`。
    - _作用_：防止 TCP 长连接“占着茅坑不拉屎”，强制定期刷新连接以触发 DNS 解析。

```yaml
spring:
  cloud:
    gateway:
      httpclient:
        pool:
          type: elastic
          max-idle-time: 5000   # 5秒不用就关
          max-life-time: 15000  # 15秒强制关，触发重连
          evict-background-interval: 5s
```

4. **应用层重试**： 代码中增加 `Retry.backoff` 指数退避重试。
    - _作用_：对抗 Docker 内部 DNS 服务器本身的同步延迟（Propagation Delay）。

### 5. 经验总结 (Lessons Learned)
在 Kubernetes/Docker 等**IP 易变**的容器化环境中，使用 Spring Cloud Gateway + Nacos（域名注册模式）时，**必须显式接管 Netty 的 DNS 解析逻辑**。

不要相信默认配置能处理好 Docker 的动态 IP，**“高性能”往往意味着“强缓存”**，而这正是动态环境的天敌。
