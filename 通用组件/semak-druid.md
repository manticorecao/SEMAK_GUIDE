# semak-druid

`semak-druid`组件是一款基于Alibaba Druid连接池来定义数据源的组件，其特性主要包括：


1. 通过简单的属性配置（properties或yaml）即可动态扩展数据源个数。
2. 可继承的基础属性。
3. Druid的专有属性定义（如：Filter、Stat，且不可继承）。
4. 基于Jasypt的加密方式(Druid原有加密方式予以屏蔽)。



## 1. 先决条件


### 1.1. 环境配置


1. Open JDK 1.8+，并已配置有效的环境变量。
2. Maven 3.3.x+，并已配置有效的环境变量。



### 1.2. Maven依赖配置


```xml
<dependency>
    <groupId>com.github.semak.druid</groupId>
    <artifactId>semak-druid-spring-boot-starter</artifactId>
    <version>最新RELEASE版本</version>
</dependency>
```


**重要：请依赖对应的jdbc驱动。**



## 2. 数据源定义


本组件对数据源的定义主要是在`spring.datasource`属性下进行扩展，分为四个部分：


### 2.1. 数据源类型定义


`spring.datasource.datasource-type`属性主要定义了数据源所选用的连接池，这里需要显式设置为`druid`。


```yaml
spring:
  datasource:
    datasource-type: druid
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


`spring.datasource.base-datasource-configure.*`属性大部分来源于Druid的原生配置属性，在`base-datasource-configure`节点下，作为所有定义的数据源的父配置，即可被子配置所继承和覆盖，其作用主要是简化子配置。


```yaml
spring:
  datasource:
    base-datasource-configure:
      #连接池中初始连接数量
      initial-size: 5
      #连接池中最小连接数量
      min-idle: 2
      #连接池中最大连接数量
      max-active: 20
      validation-query-timeout: 10
      #建议配置为true，不影响性能，并且保证安全性。申请连接的时候检测，如果空闲时间大于timeBetweenEvictionRunsMillis，执行validationQuery检测连接是否有效。
      test-while-idle: true
      #申请连接时执行validationQuery检测连接是否有效，做了这个配置会降低性能
      test-on-borrow: false
      #归还连接时执行validationQuery检测连接是否有效，做了这个配置会降低性能
      test-on-return: false
      #获取连接时最大等待时间(ms)
      max-wait: 30000
      #关闭公平锁，提升性能
      use-unfair-lock: true
      #间隔多久才进行一次检测，检测需要关闭的空闲连接(ms)
      time-between-eviction-runs-millis: 60000
      #一个空闲连接在池中最小生存的时间(ms)
      min-evictable-idle-time-millis: 300000
      #一个空闲连接在池中最大生存的时间(ms)
      max-evictable-idle-time-millis: 600000
      #打开PSCache
      pool-prepared-statements: true
      #关闭申请却忘记关闭的连接（怀疑内存泄漏时使用，有一定性能影响）
      remove-abandoned: false
      #一个连接从申请到被关闭之间的最大生命周期(ms)
      remove-abandoned-timeout-millis: 300000
      #强制关闭连接时是否记录日志
      log-abandoned: true
      #执行查询的超时时间(秒)，执行Statement对象时如果超过此时间，则抛出SQLException
      query-timeout: 60
      #执行一个事务的超时时间(秒)，执行Statement对象时判断是否为事务，如果此项未设置，则使用queryTimeout
      transaction-query-timeout: 120
```



### 2.4. Druid的Filter专有属性定义


`spring.datasource.druid-filter-configure`属性大部分来源于Druid的filter配置，仅作为通用配置，在所有数据源中生效，无需添加在基本（父）的配置中，也无法被子配置所修改。这里移除了Druid的config支持，使用jasypt来进行数据库密码的加解密。


更多的Filter配置可以参考 [官方文档](https://github.com/alibaba/druid/tree/master/druid-spring-boot-starter#%E5%A6%82%E4%BD%95%E9%85%8D%E7%BD%AE-filter)


```yaml
spring:
  datasource:   
    #druid的filter配置，仅作为通用配置，无需添加在基础的配置中。移除config支持，使用jasypt
    druid-filter-configure:
      #SQL日志
      slf4j:
        enabled: true
        statement-log-enabled: true
        statement-executable-sql-log-enable: true
      #统计
      stat:
        enabled: true
        merge-sql: true
        log-slow-sql: true
        slow-sql-millis: 300
      #防止SQL注入
      wall:
        enabled: true
```



### 2.5. Druid的Stat专有属性定义


`spring.datasource.druid-stat-configure`属性大部分来源于Druid的stat监控配置，仅作为通用配置，在所有数据源中生效，无需添加在基本（父）的配置中，也无法被子配置所修改。


更多的Stat配置可以参考 [官方文档](https://github.com/alibaba/druid/tree/master/druid-spring-boot-starter#%E9%85%8D%E7%BD%AE%E5%B1%9E%E6%80%A7) 中的 **监控配置** 部分


```yaml
spring:
  datasource: 
    #druid的stat监控配置，仅作为通用配置，无需添加在基础的配置中
    druid-stat-configure:
      stat-view-servlet:
        enabled: true
        url-pattern: '/druid/*'
        login-username: druid
        login-password: druid
        reset-enable: false
      web-stat-filter:
        enabled: true
        url-pattern: '/*'
        exclusions: '*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*'
```



### 2.6. 数据源详细属性定义


`spring.datasource.<datasource-name>.*`属性大部分来源于Druid的原生配置属性，且可继承或覆盖`spring.datasource.base-datasource-configure`节点下的属性。`<datasource-name>`节点为`spring.datasource.datasource-names.*`节点中所引用的节点名称。


```yaml
spring:
  datasource: 
    pri-ds:
      #enabled: true
      default-candidate: true
      url: jdbc:oracle:thin:@192.168.23.35:1521:xxx
      username: xxx
      password: ENC(409XzuLG2vmRexN08yIvKg==)
      validation-query: SELECT 1 FROM DUAL
    sec-ds:
      #enabled: true
      default-candidate: false
      #连接属性
      connection-properties:
        useUnicode: true
        characterEncoding: utf8
        autoReconnect: true
        failOverReadOnly: false
        useSSL: false
      url: jdbc:mysql://192.168.23.149:3306/xxx
      username: root
      password: Aa111111
      validation-query: SELECT 1
```



### 2.7. 一个完整的多数据源定义


结合以上几项，可以得出一个完整的数据源定义如下：


```yaml
spring:
  datasource:
    #hikari / druid
    datasource-type: druid
    datasource-names:
      - "pri-ds"
      - "sec-ds"
    base-datasource-configure:
      #连接池中初始连接数量
      initial-size: 5
      #连接池中最小连接数量
      min-idle: 2
      #连接池中最大连接数量
      max-active: 20
      validation-query-timeout: 10
      #建议配置为true，不影响性能，并且保证安全性。申请连接的时候检测，如果空闲时间大于timeBetweenEvictionRunsMillis，执行validationQuery检测连接是否有效。
      test-while-idle: true
      #申请连接时执行validationQuery检测连接是否有效，做了这个配置会降低性能
      test-on-borrow: false
      #归还连接时执行validationQuery检测连接是否有效，做了这个配置会降低性能
      test-on-return: false
      #获取连接时最大等待时间(ms)
      max-wait: 30000
      #关闭公平锁，提升性能
      use-unfair-lock: true
      #间隔多久才进行一次检测，检测需要关闭的空闲连接(ms)
      time-between-eviction-runs-millis: 60000
      #一个空闲连接在池中最小生存的时间(ms)
      min-evictable-idle-time-millis: 300000
      #一个空闲连接在池中最大生存的时间(ms)
      max-evictable-idle-time-millis: 600000
      #指定每个连接上PSCache的大小
      max-pool-prepared-statement-per-connection-size: 100
      #关闭申请却忘记关闭的连接（怀疑内存泄漏时使用，有一定性能影响）
      remove-abandoned: false
      #一个连接从申请到被关闭之间的最大生命周期(ms)
      remove-abandoned-timeout-millis: 300000
      #强制关闭连接时是否记录日志
      log-abandoned: true
      #执行查询的超时时间(秒)，执行Statement对象时如果超过此时间，则抛出SQLException
      query-timeout: 60
      #执行一个事务的超时时间(秒)，执行Statement对象时判断是否为事务，如果此项未设置，则使用queryTimeout
      transaction-query-timeout: 120
    #druid的filter配置，仅作为通用配置，无需添加在基础的配置中。移除config支持，使用jasypt
    druid-filter-configure:
      #SQL日志
      slf4j:
        enabled: true
        statement-log-enabled: true
        statement-executable-sql-log-enable: true
      #统计
      stat:
        enabled: true
        merge-sql: true
        log-slow-sql: true
        slow-sql-millis: 300
      #防止SQL注入
      wall:
        enabled: true
    #druid的stat监控配置，仅作为通用配置，无需添加在基础的配置中
    druid-stat-configure:
      stat-view-servlet:
        enabled: true
        url-pattern: '/druid/*'
        login-username: druid
        login-password: druid
        reset-enable: false
      web-stat-filter:
        enabled: true
        url-pattern: '/*'
        exclusions: '*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*'
    pri-ds:
      #enabled: true
      default-candidate: true
      url: jdbc:oracle:thin:@192.168.23.35:1521:xxx
      username: psfp
      password: ENC(409XzuLG2vmRexN08yIvKg==)
      validation-query: SELECT 1 FROM DUAL
      #打开PSCache
      pool-prepared-statements: true
      #指定每个连接上PSCache的大小
      max-pool-prepared-statement-per-connection-size: 100
    sec-ds:
      #enabled: true
      default-candidate: false
      #连接属性
      connection-properties:
        useUnicode: true
        characterEncoding: utf8
        autoReconnect: true
        failOverReadOnly: false
        useSSL: false
      url: jdbc:mysql://192.168.23.149:3306/xxx
      username: root
      password: Aa111111
      validation-query: SELECT 1
```



### 2.8. Druid属性描述


这里对Druid原生必要、常用属性和非原生扩展属性进行描述，其他属性可参考[官方属性说明](https://github.com/alibaba/druid/wiki/DruidDataSource%E9%85%8D%E7%BD%AE%E5%B1%9E%E6%80%A7%E5%88%97%E8%A1%A8)，做进一步了解。


| 属性 | 是否必填 | 默认值 | 描述 |
| :--- | :--- | :--- | :--- |
| **enabled** | 否 | true | 是否启用连接池 |
| **default-candidate** | 否 | true | 作为默认的候选数据源，多个数据源仅能有一个默认的候选数据源设置为true |
| **url** | 是 | none | jdbc url 地址，驱动可通过jdbc-url来自动侦测 |
| **connection-properties** | 否 | none | 数据源的属性，原来在jdbc-url后面追加的属性可以配置在此（注意：此节点下的属性必须为驼峰形式） |
| **username** | 是 | none | 连接数据库的用户名 |
| **password** | 是 | none | 连接数据库的密码 |
| **initial-size** | 否 | 0 | 初始化时建立物理连接的个数 |
| **max-active** | 否 | 8 | 最大连接池数量 |
| **min-idle** | 否 | 0 | 最小连接池数量 |
| **max-wait** | 否 | -1 | 获取连接时最大等待时间，单位毫秒。配置了maxWait之后，缺省启用公平锁，并发效率会有所下降，如果需要可以通过配置useUnfairLock属性为true使用非公平锁 |
| **pool-prepared-statements** | 否 | false | 是否缓存preparedStatement，也就是PSCache。PSCache对支持游标的数据库性能提升巨大，比如说oracle。在mysql下建议关闭 |
| **validation-query** | 否 | none | 用来检测连接是否有效的sql，要求是一个查询语句，常用select 'x'。如果validationQuery为null，testOnBorrow、testOnReturn、testWhileIdle都不会起作用 |
| **validation-query-timeout** | 否 | -1 | 单位：秒，检测连接是否有效的超时时间。底层调用jdbc Statement对象的void setQueryTimeout(int seconds)方法 |
| **test-on-borrow** | 否 | true | 申请连接时执行validationQuery检测连接是否有效，做了这个配置会降低性能 |
| **test-on-return** | 否 | false | 归还连接时执行validationQuery检测连接是否有效，做了这个配置会降低性能 |
| **test-while-idle** | 否 | false | 建议配置为true，不影响性能，并且保证安全性。申请连接的时候检测，如果空闲时间大于timeBetweenEvictionRunsMillis，执行validationQuery检测连接是否有效 |
| **time-between-eviction-runs-millis** | 否 | 60000 (ms) | 1) Destroy线程会检测连接的间隔时间，如果连接空闲时间大于等于minEvictableIdleTimeMillis则关闭物理连接<br/>2) testWhileIdle的判断依据，详细看testWhileIdle属性的说明 |
| **min-evictable-idle-time-millis** | 否 | 1800000 (ms) | 连接保持空闲而不被驱逐的最小时间 |



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


为便于对密码进行加解密的处理，可以使用插件`semak-jasypt-maven-plugin`提供的maven命令进行加解密。具体操作参考相关插件文档。
