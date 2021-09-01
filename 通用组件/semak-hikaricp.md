# semak-hikaricp

`semak-hikaricp`组件是一款基于Hikaricp连接池来定义数据源的组件，其特性主要包括：


1. 通过简单的属性配置（properties或yaml）即可动态扩展数据源个数。
2. 可继承的基础属性。
3. 基于Jasypt的加密方式。



## 1. 先决条件


### 1.1. 环境配置


1. Open JDK 1.8+，并已配置有效的环境变量。
2. Maven 3.3.x+，并已配置有效的环境变量。



### 1.2. Maven依赖配置


```xml
<dependency>
    <groupId>com.github.semak.hikaricp</groupId>
    <artifactId>semak-hikaricp-spring-boot-starter</artifactId>
    <version>最新RELEASE版本</version>
</dependency>
```


**重要：请依赖对应的jdbc驱动。**



## 2. 数据源定义


本组件对数据源的定义主要是在`spring.datasource`属性下进行扩展，分为四个部分：


### 2.1. 数据源类型定义


`spring.datasource.datasource-type`属性主要定义了数据源所选用的连接池，这里需要显式设置为`hikari`。


```yaml
spring:
  datasource:
    datasource-type: hikari
```



### 2.2. 数据源名称定义


`spring.datasource.datasource-names.*`属性主要指定了声明的数据源名称，也作为Spring Bean Name存在，作为数据源详细属性配置的前提。


```yaml
spring:
  datasource:
    datasource-names:
      - "pri-ds"
      - "sec-ds"
```


如上配置，`pri-ds`即表示指定了一个Spring Bean Name为`pri-ds`的Bean，会在后续数据源详细属性定义中进行匹配。



### 2.3. 数据源基本（父）属性定义


`spring.datasource.base-datasource-configure.*`属性大部分来源于Hikaricp的原生配置属性，在`base-datasource-configure`节点下，作为所有定义的数据源的父配置，即可被子配置所继承和覆盖，其作用主要是简化子配置。


```yaml
spring:
  datasource:
    base-datasource-configure:
      minimum-idle: 2
      maximum-pool-size: 5
      auto-commit: true
      idle-timeout: 60000
      max-lifetime: 600000
      connection-timeout: 12000
      leak-detection-threshold: 5000
```



### 2.4. 数据源详细属性定义


`spring.datasource.<datasource-name>.*`属性大部分来源于Hikaricp的原生配置属性，且可继承或覆盖`spring.datasource.base-datasource-configure`节点下的属性。`<datasource-name>`节点为`spring.datasource.datasource-names.*`节点中所引用的节点名称。


```yaml
spring:
  datasource:
    pri-ds:
      #enabled: true
      default-candidate: true
      jdbc-url: jdbc:oracle:thin:@192.168.23.35:1521:xxx
      username: xxx
      password: ENC(409XzuLG2vmRexN08yIvKg==)
      minimum-idle: 1
      maximum-pool-size: 3
      connection-test-query: SELECT 1 FROM DUAL
    sec-ds:
      #enabled: true
      default-candidate: false
      data-source-properties:
        useUnicode: true
        characterEncoding: utf8
        autoReconnect: true
        failOverReadOnly: false
        useSSL: false
        serverTimezone: Asia/Shanghai
      jdbc-url: jdbc:mysql://192.168.23.149:3306/xxx
      username: root
      password: Aa111111
      minimum-idle: 2
      maximum-pool-size: 5
      connection-test-query: SELECT 1
```



### 2.5. 一个完整的多数据源定义


结合以上四项，可以得出一个完整的数据源定义如下：


```yaml
spring:
  datasource:
    #hikari / druid
    datasource-type: hikari
    datasource-names:
      - "pri-ds"
      - "sec-ds"
    base-datasource-configure:
      minimum-idle: 2
      maximum-pool-size: 5
      auto-commit: true
      idle-timeout: 60000
      max-lifetime: 600000
      connection-timeout: 12000
      leak-detection-threshold: 5000
    pri-ds:
      #enabled: true
      default-candidate: true
      jdbc-url: jdbc:oracle:thin:@192.168.23.35:1521:xxx
      username: xxx
      password: ENC(409XzuLG2vmRexN08yIvKg==)
      minimum-idle: 1
      maximum-pool-size: 3
      connection-test-query: SELECT 1 FROM DUAL
    sec-ds:
      #enabled: true
      default-candidate: false
      data-source-properties:
        useUnicode: true
        characterEncoding: utf8
        autoReconnect: true
        failOverReadOnly: false
        useSSL: false
        serverTimezone: Asia/Shanghai
      jdbc-url: jdbc:mysql://192.168.23.149:3306/xxx
      username: root
      password: Aa111111
      minimum-idle: 2
      maximum-pool-size: 5
      connection-test-query: SELECT 1
```



### 2.6. Hikaricp属性描述


这里对Hikaricp原生必要、常用属性和非原生扩展属性进行描述，其他属性可参考[官方属性说明](https://github.com/brettwooldridge/HikariCP)，做进一步了解。

| 属性 | 是否必填 | 默认值 | 描述 |
| :--- | :--- | :--- | :--- |
| **enabled** | 否 | true | 是否启用连接池 |
| **default-candidate** | 否 | true | 作为默认的候选数据源，多个数据源仅能有一个默认的候选数据源设置为true |
| **jdbc-url** | 是 | none | jdbc url 地址，驱动可通过jdbc-url来自动侦测 |
| **data-source-properties** | 否 | none | 数据源的属性，原来在jdbc-url后面追加的属性可以配置在此（注意：此节点下的属性必须为驼峰形式） |
| **username** | 是 | none | 连接数据库的用户名 |
| **password** | 是 | none | 连接数据库的密码 |
| **auto-commit** | 否 | true | 控制连接池中连接的自动提交属性 |
| **connection-timeout** | 否 | 30000 (ms) | 等待池中连接的最大毫秒数。 如果在没有连接可用的情况下超过此时间，则将抛出SQLException。 最低可接受的连接超时为250毫秒 |
| **minimum-idle** | 否 | 和`maximum-pool-size`配置的值一致 | 连接池中维护的最小空闲连接数 |
| **idle-timeout** | 否 | 600000 (ms) | 连接在连接池中空闲的最长时间 |
| **max-lifetime** | 否 | 1800000 (ms) | 连接池中连接的最长生命周期。强烈建议设置此值，值0表示没有最大生命周期（无限生命周期） |
| **connection-test-query** | 否 | none | 从连接池中获取连接之前执行的查询，以验证与数据库的连接是否仍然有效 |
| **maximum-pool-size** | 否 | 10 | 连接池允许的最大连接数，包括空闲和正在使用的连接。 基本上，此值将确定数据库后端的最大实际连接数 |
| **leak-detection-threshold** | 否 | 0 | 可能的连接泄漏检测。 值为0表示禁用泄漏检测。 启用泄漏检测的最低可接受值是2000（ms） |



## 3. 密码加密方式


密码是以jasypt的方式进行加解密的，如：


```yaml
password: ENC(409XzuLG2vmRexN08yIvKg==)
```


使用`ENC(已加密密码)`的方式。如果显式定义了盐，则使用显式定义的盐进行加解密，否则采用系统默认设置的盐进行加解密：


```yaml
jasypt:
  encryptor:
    password: 12345
```


这里的`password`属性类似于盐。



### 3.1. 加解密插件


为便于对密码进行加解密的处理，可以使用插件`semak-jasypt-maven-plugin`提供的maven命令进行加解密。具体操作请参考相关插件文档。
