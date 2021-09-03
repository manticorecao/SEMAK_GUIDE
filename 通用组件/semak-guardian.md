# semak-guardian(迭代中)

`semak-guardian` 组件是一款基于Apache Shiro构建的安全组件，提供多应用、多端（设备）的中心化身份认证与鉴权（RBAC）功能。整个组件分为如下几个部分：


**守护者平台（中心化身份认证与鉴权平台）：**

1. 中心化且单独提供服务的身份认证与鉴权的微服务应用。
1. 可以单独使用，亦可整合服务注册与发现来使用。
1. 基于JWT令牌认证机制，提供临时令牌、刷新令牌与访问令牌。
   1. 临时令牌：身份认证通过，但未选择指定应用进入前使用，时效非常短（5分钟左右）。
   1. 刷新令牌：指定应用且身份认证通过后，发放的正式令牌，仅用于交换访问令牌使用，时效较长（24小时 - 7天不等）。
   1. 访问令牌：使用刷新令牌交换，用于访问应用服务的令牌，时效较短（15-20分钟左右）。
4. 提供完善的接口管理服务（参考Swagger）。
4. 基于Redis的中心化权限缓存机制。
4. 提供操作级别的权限控制方式。



**守护者客户端（为应用提供身份认证与鉴权的客户端）：**

1. 提供服务端接口的代理服务。
1. 通过简单配置为应用构建基于多端（设备）的中心化身份认证与鉴权（RBAC）功能。


**守护者异步处理平台（可选）：**

> 提供了网络隔离环境下服务的正常通信。（满足此类网络拓扑：公网区域 [普通应用] <- 双向通信 -> DMZ隔离区域 [中间件服务] <- 单向通信 - 内网区域 [守护者平台]）

1. 提供了守护者平台的所有功能。
1. 结合MQ与Redis来实现客户端接口的异步通信。



**守护者异步处理客户端（可选）：**

1. 提供了守护者客户端的所有功能。
1. 结合MQ与Redis来实现平台端接口的异步通信。
1. 基于长轮询机制改造接口响应方式，免除前端改造工作量。



## 1. 先决条件
### 1.1. 环境配置

1. Open JDK 1.8+，并已配置有效的环境变量。
1. Maven 3.3.x+，并已配置有效的环境变量。
1. Redis服务。
1. RocketMQ服务（仅异步处理需要）。



### 1.2. Maven依赖配置
> 以下Maven依赖二选一，根据实际项目需要进行选择。

#### 1.2.1. 守护者客户端依赖
```xml
<dependency>
  <groupId>com.github.semak.guardian</groupId>
  <artifactId>semak-guardian-spring-boot-starter</artifactId>
  <version>最新RELEASE版本</version>
</dependency>
```



#### 1.2.2. 守护者异步处理客户端依赖

```xml
<dependency>
  <groupId>com.github.semak.guardian</groupId>
  <artifactId>semak-guardian-async-spring-boot-starter</artifactId>
  <version>最新RELEASE版本</version>
</dependency>
```



## 2. 平台端功能


暂略。包括：应用配置，设备配置，角色配置，权限配置，菜单配置等等。



## 3. 客户端功能

### 3.1. 属性定义
```yaml
#守护者客户端基本配置
spring:
  guardian:
    client:
      anon-uris:
        - /demo/getCountry

#客户端负载均衡配置
ribbon:
  eager-load:
    enabled: true
    clients: semak-guardian
  ConnectTimeout: 5000
  ReadTimeout: 15000

#平台端域名或IP列表（使用服务注册与发现时无需配置）
semak-guardian:
  ribbon:
    listOfServers: 192.168.1.216:8090
```



#### 3.1.1. 属性描述

| **属性** | **数据类型** | **必填** | **默认值** | **描述** |
| :---: | :---: | :---: | :---: | :---: |
| **spring.guardian.client.anon-uris** | List | 否 | N/A | 无需守护者客户端进行认证的URI |
| **ribbon.eager-load.enabled** | boolean | 否 | fasle | 启用Ribbon客户端负载均衡的饥饿加载模式 |
| **ribbon.eager-load.clients** | String | 否 | N/A | 饥饿加载模式应用到的客户端名称，以英文逗号分隔 |
| **ribbon.ConnectTimeout** | int | 否 | 2000 | Ribbon客户端全局连接超时（单位：毫秒） |
| **ribbon.ReadTimeout** | int | 否 | 5000 | Ribbon客户端全局读取超时（单位：毫秒） |
| **semak-guardian.ribbon.listOfServers** | String | 是 | N/A | 守护者平台域名或IP:PORT列表，（使用服务注册与发现时无需配置），以英文逗号分隔 |



### 3.2. 使用方式


在配置完成上面章节的属性之后，整个应用程序就进入受保护状态，任意未经授权的访问都将被禁止。要对应用内资源进行访问，需要按照下面步骤进行操作。


#### 3.2.1. 用户登录
通过配套的登录页面进行登录。



#### 3.2.2. 普通资源的访问

登录成功后，对于一般的服务端点，即未显式声明权限注解的服务方法，用户可以直接访问。



#### 3.2.3. 授权资源的访问

对于以下使用 `@RequiresPermissions` 的服务方法，用户需要有对应的操作权限方可访问。
```java
@RequiresPermissions("demo:api:country:add")
@PostMapping("/addCountry")
@Override
public Response addCountry(@RequestBody @Valid CountryRequest countryRequest) {
    DemoCountry demoCountry = new DemoCountry();
    BeanCopyUtil.copyProperties(countryRequest, demoCountry, BeanCopyUtil.OverridePolicy.INCREMENTAL);
    demoCountryService.addCountry(demoCountry);
    return Response.ofSuccess();
}
```
用户相关权限配置方式可以参考平台端功能。
