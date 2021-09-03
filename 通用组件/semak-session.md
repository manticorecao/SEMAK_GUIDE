# semak-session

`semak-session` 提供了会话（session）中心化的功能，基于 `spring-session` 项目进行了扩展。其主要特性包括：


1. 支持基于 `semak-redis` 组件的分布式会话功能（建议配合`semak-redis`组件使用）。
1. 支持Redis Key的命名空间（前缀）设置。
1. 支持会话的超时设置。
1. 支持基于其他存储的分布式会话功能，包括MongoDB、JDBC、HAZELCAST等。



## 1. 先决条件


### 1.1. 环境配置


1. Open JDK 1.8+，并已配置有效的环境变量。
1. Maven 3.3.x+，并已配置有效的环境变量。
1. 配合 `semak-session` 组件的有效的中心化存储服务，如：Redis，并预先在配置文件中设置好配置内容。



### 1.2. Maven依赖配置
```xml
<dependency>
    <groupId>com.github.semak.session</groupId>
    <artifactId>semak-session-spring-boot-starter</artifactId>
    <version>最新RELEASE版本</version>
</dependency>
```



## 2. 属性定义


### 2.1. 基于Redis的属性定义
```yaml
spring:
  session:
    store-type: redis
    redis:
      namespace: pw:demo:session
    timeout: "PT120M"
```

### 2.2. 基于Redis的属性描述
以下属性为 `spring.session` 的子节点属性。

| **属性** | **是否必填** | **默认值** | **描述** |
| --- | --- | --- | --- |
| store-type | 否 |  | 中心化存储类型（参照枚举类 `org.springframework.boot.autoconfigure.session.StoreType` ） |
| redis.namespace | 否 | spring:session | 用来存储会话的键的命名空间（前缀） |
| redis.flush-mode | 否 | on-save | 会话刷新模式。确定何时将回回更改写入会话存储（参照枚举类 `org.springframework.session.FlushMode` ） |
| redis.save-mode | 否 | on-set-attribute | 会话保存模式。确定如何跟踪会话更改并将其保存到会话存储。（参照枚举类 `org.springframework.session.SaveMode` ） |
| redis.configure-action | 否 | notify-keyspace-events | 当不存在用户定义的ConfigureRedisAction Bean时，要应用的configure操作。（参照枚举类 `org.springframework.boot.autoconfigure.session.RedisSessionProperties.ConfigureAction` ） |
| redis.cleanup-cron | 否 | 0 * * * * * | 过期的会话清理作业的Cron表达式 |
| timeout | 否 | PT30M | 会话超时时间，参考 `java.time.Duration` 类的注释举例来定义标准值。 |



## 3. 验证方式

结合`seamak-redis`组件创建一个`application.yml`

```yaml
spring:
  application:
    name: pw.session
  redis:
    client-names:
      - "cluster-client"
    base-redis-configure:
      use-jackson-as-default-serializer: true
      key-prefix: ${spring.application.name}
      ssl: false
      connect-timeout-in-millis: 1500
      read-timeout-in-millis: 5000
      database: 0
      ignore-error: true
      pool:
        max-active: 45
        min-idle: 1
        max-idle: 4
        max-wait-in-millis: 35000
        time-between-eviction-runs-in-millis: 35000
        test-while-idle: true
      retry-policy:
        retry-times: 3
        retry-interval-in-millis: 3000
        exponential-back-off-policy:
          initial-interval-in-millis: 2000
          max-interval-in-millis: 60000
          multiplier: 2
    cluster-client:
      default-candidate: true
      type: cluster
      nodes:
        - "192.168.3.21:7000"
        - "192.168.3.22:7000"
        - "192.168.3.31:7000"
        - "192.168.3.32:7000"
        - "192.168.3.41:7000"
        - "192.168.3.42:7000"
      max-redirects: 5
      password: 123456
  session:
    store-type: redis
    redis:
      namespace: pw:demo:session
    timeout: "PT120M"
```

按照下例编写一个Controller示例。

```java
@Slf4j
@Controller
public class LoginController {

    @RequestMapping("/test/login")
    public String login(@RequestParam(value = "username", required = false) String username, HttpServletRequest request, HttpServletResponse response, HttpSession session) {
        log.info(">>> SESSION ID: {}", session.getId());
        Boolean isLogin = (Boolean) session.getAttribute("isLogin");

        if (isLogin != null && isLogin) {
            log.info("用户[{}]已登录", session.getAttribute("username"));
            return "index";
        } else {
            if ("caobin".equals(username)) {
                session.setAttribute("username", username);
                session.setAttribute("isLogin", true);
                return "index";
            } else {
                return "authFailed";
            }
        }
    }
}
```
创建2个服务端点，端口分别为`8080`和`8090`。请求 `http://localhost:8080/test/login?username=caobin` ，然后接着请求`http://localhost:8090/test/login` 发现打印日志**用户xx已登录**。此时去查看Redis中 `pw:demo:session` 开头，以**session id**结尾的**Redis Key**值，如能找到，说明会话中心化存储已生效。

