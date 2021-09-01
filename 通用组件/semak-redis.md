`semak-redis`提供Key-Value风格的数据存储组件，基于`spring-data-redis`项目进行了扩展和增强。 其主要特性包括：


1. 支持Redis的Standalone、Sentinel、Cluster客户端的使用。
2. 支持Codis客户端的使用。
3. 基于Lettuce客户端来构建。
4. 支持客户端对CodisProxy的负载均衡
5. 提供SSL支持。
6. 通过简单的属性配置（properties或yaml）即可动态扩展客户端个数。
7. 可继承的基础属性。
8. 提供key前缀来区分应用。
9. 基于Jasypt的加密方式。
10. 提供客户端重试策略。
11. 提供客户端异常处理的模式。
12. 提供基于spring-cache的注解支持。



## 1. 先决条件


### 1.1. 环境配置


1. Open JDK 1.8+，并已配置有效的环境变量。
2. Maven 3.3.x+，并已配置有效的环境变量。



### 1.2. Maven依赖配置


```xml
<dependency>
    <groupId>com.github.semak.redis</groupId>
    <artifactId>semak-redis-spring-boot-starter</artifactId>
    <version>最新RELEASE版本</version>
</dependency>
```


## 2. 客户端定义


本组件对客户端的定义主要是在`spring.redis`属性下进行扩展，分为三个部分：客户端名称定义、客户端基本（父）属性定义 和 客户端专有属性定义


### 2.1. 客户端名称定义


`spring.redis.client-names.*`属性主要指定了声明的客户端名称，也作为Spring Bean Name存在，作为客户端详细属性配置的前提。


```yaml
spring:
  redis:
    client-names:
      - "pri-redis"
      - "sec-redis"
      - "ssl-redis"
      - "redis-cluster"
      - "codis"
```


如上配置，`pri-redis`即表示指定了一个Spring Bean Name为`pri-redis`的Bean，会在后续客户端详细属性定义中进行匹配。


### 2.2. 客户端基本（父）属性定义


`spring.redis.base-redis-configure.*`属性主要由基础属性、连接池属性和重试策略属性构成，在`base-redis-configure`节点下，作为所有定义的客户端的父配置，即可被子配置所继承和覆盖，其作用主要是简化子配置（详细配置）。


```yaml
spring:
  application:
    name: line.project
  redis:
    base-redis-configure:
      #enabled: true
      #default-candidate: true
      use-jackson-as-default-serializer: true
      key-prefix: ${spring.application.name}
      ssl: false
      connect-timeout-in-millis: 1500
      read-timeout-in-millis: 5000
      #Redis数据库索引（0-15）
      database: 0
      #password:
      #出错时不抛出异常，返回null
      ignore-error: true
      pool:
        #连接池可以分配的最大连接数
        max-active: 45
        #最小空闲连接数
        min-idle: 1
        #最大空闲连接数
        max-idle: 4
        #当连接池耗尽时，在抛出异常之前连接分配应阻塞的最长时间（毫秒）。使用负值则一直等待。
        max-wait-in-millis: 35000
        #空闲连接检测线程的周期（毫秒）。如果为负值，表示不运行检测线程。默认为-1.
        time-between-eviction-runs-in-millis: 35000
        test-while-idle: true
      #重试策略
      retry-policy:
        #重试次数
        retry-times: 3
        #重试周期（毫秒）
        retry-interval-in-millis: 3000
        #指数退避策略
        exponential-back-off-policy:
          #初始休眠周期（单位：毫秒）
          initial-interval-in-millis: 2000
          #退避周期最大值（单位：毫秒）
          max-interval-in-millis: 60000
          #乘数
          multiplier: 2
```


### 2.3. 客户端基本（父）属性描述


以下属性为`spring.redis.base-redis-configure`的子节点属性。

| **属性** | **是否必填** | **默认值** | **描述** |
| :--- | :--- | :--- | :--- |
| **enabled** | 否 | true | 是否启用客户端 |
| **default-candidate** | 否 | true | 作为默认的候选客户端，多个客户端仅能有一个默认的候选客户端设置为true |
| **use-jackson-as-default-serializer** | 否 | true | 使用Jackson作为默认序列化的方式 |
| **key-prefix** | 否 | none | 为key添加的前缀名称。注意：使用RedisConnection直接操作的情况下，此前缀不会在key上进行附加 |
| **type** | 是 | none | Redis客户端类型，支持的类型值为：standalone, sentinel, cluster, codis，
其具体含义分别为：独立模式（单节点）、哨兵模式、集群模式、Codis分布式缓存客户端类型 |
| **ssl** | 否 | false | 启用SSL支持（配合服务端Stunnel使用） |
| **keystore-path** | 否 | none | Keystore路径，支持`classpath:xxx/yyy.jks` |
| **keystore-password** | 否 | none | Keystore预设的密码 |
| **connect-timeout-in-millis** | 否 | 5000 | 连接超时时间（单位：毫秒） |
| **read-timeout-in-millis** | 否 | 30000 | 读取超时时间（单位：毫秒） |
| **shutdown-timeout-in-millis** | 否 | 100 | （基于Lettuce的）客户端关闭超时时间（单位：毫秒） |
| **database** | 否 | 0 | Redis数据库索引（默认跟随Redis服务端，为0-15），其中Redis客户端类型为cluster的配置此属性无效（cluster的database默认为0） |
| **password** | 否 | none | Redis服务访问密码，Codis 3+为其Proxy服务的访问密码 |
| **ignore-error** | 否 | true | 出错时是否忽略异常，有返回值的方法返回null |
| **pool.max-active** | 否 | -1 | 连接池可以分配的最大连接数。负值则无限制 |
| **pool.min-idle** | 否 | 0 | 最小空闲连接数 |
| **pool.max-idle** | 否 | 8 | 最大空闲连接数 |
| **pool.max-wait-in-millis** | 否 | -1 | 当连接池耗尽时，在抛出异常之前连接分配应阻塞的最长时间（毫秒）。使用负值则一直等待。 |
| **pool.time-between-eviction-runs-in-millis** | 否 | -1 | 空闲连接检测线程的周期（毫秒）。如果为负值，表示不运行检测线程。 |
| **pool.test-while-idle** | 否 | true | 空闲检测连接有效性 |
| **pool.test-on-borrow** | 否 | false | 获取连接时检测连接有效性 |
| **pool.test-on-return** | 否 | false | 返还连接时时检测连接有效性 |
| **retry-policy.retry-times** | 否 | 3 | 重试次数 |
| **retry-policy.retry-interval-in-millis** | 否 | 3000 | 重试周期（毫秒） |
| **retry-policy.exponential-back-off-policy.initial-interval-in-millis** | 否 | 100 | 指数退避策略：初始休眠周期（单位：毫秒），与默认的“乘数”值（multiplier）相结合，为多次重试提供了有效的暂停传播机制 |
| **retry-policy.exponential-back-off-policy.max-interval-in-millis** | 否 | 30000 | 退避周期最大值（单位：毫秒） |
| **retry-policy.exponential-back-off-policy.multiplier** | 否 | 2 | 乘数 |

### 
### 2.4. 客户端专有属性定义


除了可以使用通用客户端基本（父）属性进行属性值覆盖外，针对不同客户端类型，额外提供如下属性配置。


#### 2.4.1. 独立模式（Standalone）属性定义


```yaml
spring:
  redis:
    client-names:
      - "standalone-client"
    standalone-client:
      type: standalone
      host: 192.168.23.158
      port: 6379
      password: ENC(6tqzYkJGBbg+s5nK08DtEA==)
```


#### 2.4.2. 独立模式（Standalone）属性描述


以下属性为`spring.redis.<redis-client-name>`的子节点属性。

| 属性 | 是否必填 | 默认值 | 描述 |
| :--- | :--- | :--- | :--- |
| **host** | 是 | none | Redis服务域名或IP |
| **port** | 否 | 6379 | Redis服务端口（默认：6379） |



#### 2.4.3. 哨兵模式（Sentinel）属性定义


```yaml
spring:
  redis:
    client-names:
      - "sentinel-client"
    sentinel-client:
      type: sentinel
      master-node-name: redisMaster
      nodes:
        - "192.168.23.155:26379"
        - "192.168.23.157:26379"
        - "192.168.23.158:26379"
      password: ENC(6tqzYkJGBbg+s5nK08DtEA==)
```


#### 2.4.4. 哨兵模式（Sentinel）属性描述


以下属性为`spring.redis.<redis-client-name>`的子节点属性。

| **属性** | **是否必填** | **默认值** | **描述** |
| :--- | :--- | :--- | :--- |
| **master-node-name** | 是 | none | Redis服务的主节点名称（Sentinel中定义的指向Redis主节点的名称） |
| **nodes** | 是 | none | Sentinel节点（遵循host:port的定义模式） |



#### 2.4.5. 集群模式（Cluster）属性定义


```yaml
spring:
  redis:
    client-names:
      - "cluster-client"
    cluster-client:
      type: cluster
      nodes:
        - "192.168.23.155:8001"
        - "192.168.23.155:8002"
        - "192.168.23.157:8003"
        - "192.168.23.157:8004"
        - "192.168.23.158:8005"
        - "192.168.23.158:8006"
      max-redirects: 5
      password: ENC(6tqzYkJGBbg+s5nK08DtEA==)
```


#### 2.4.6. 集群模式（Cluster）属性描述


以下属性为`spring.redis.<redis-client-name>`的子节点属性。

| **属性** | **是否必填** | **默认值** | **描述** |
| :--- | :--- | :--- | :--- |
| **nodes** | 是 | none | 集群节点（遵循host:port的定义模式） |
| **max-redirects** | 否 | 5 | 跨集群执行命令时要遵循的最大重定向数 |



#### 2.4.7. Codis属性定义


```yaml
spring:
  redis:
    client-names:
      - "codis-client"
    codis-client:
      type: codis
      zookeeper:
        connect-string:
          - "192.168.23.155:2181"
          - "192.168.23.157:2181"
          - "192.168.23.158:2181"
        proxy-dir: /jodis/codis-dev
        session-timeout-in-millis: 30000
      password: ENC(mv+UiKUaQO9cC9Q0/w5AWQ==)
```


#### 2.4.8. Codis属性描述


以下属性为`spring.redis.<redis-client-name>`的子节点属性。

| **属性** | **是否必填** | **默认值** | **描述** |
| :--- | :--- | :--- | :--- |
| **zookeeper.connect-string** | 是 | none | 连接字符串（遵循host:port模式） |
| **zookeeper.session-timeout-in-millis** | 否 | 30000 | session超时时间（单位：毫秒） |
| **zookeeper.proxy-dir** | 是 | none | 代理目录，codis2: `/zk/codis/db_xxx/proxy`，codis3: `/jodis/xxx` （`xxx`：codis product name） |



### 2.5. 一个完整的多客户端定义样例


```yaml
spring:
  application:
    name: line.project
  redis:
    client-names:
      - "standalone-client"
      - "sentinel-client"
      - "ssl-based-client"
      - "cluster-client"
      - "codis-client"
    base-redis-configure:
      use-jackson-as-default-serializer: true
      key-prefix: ${spring.application.name}
      ssl: false
      connect-timeout-in-millis: 1500
      read-timeout-in-millis: 5000
      ignore-error: true
      pool:
        max-active: 45
        min-idle: 1
        max-idle: 4
        max-wait-in-millis: 35000
        time-between-eviction-runs-in-millis: 35000
      retry-policy:
        retry-times: 3
        retry-interval-in-millis: 3000
    standalone-client:
      #standalone/sentinel/cluster/codis
      type: standalone
      read-timeout-in-millis: 15000
      database: 1
      host: 192.168.23.158
      port: 6379
      password: ENC(6tqzYkJGBbg+s5nK08DtEA==)
      pool:
        max-active: 50
      retry-policy:
        retry-times: 3
        retry-interval-in-millis: 2000
        #exponential-back-off-policy:
        #  initial-interval-in-millis: 2000
        #  max-interval-in-millis: 60000
        #  multiplier: 2
    sentinel-client:
      enabled: true
      default-candidate: false
      #single/sentinel/cluster/codis
      type: sentinel
      database: 2
      master-node-name: redisMaster
      nodes:
        - "192.168.23.155:26379"
        - "192.168.23.157:26379"
        - "192.168.23.158:26379"
      password: ENC(6tqzYkJGBbg+s5nK08DtEA==)
    ssl-based-client:
      default-candidate: false
      type: standalone
      database: 3
      host: 192.168.23.155
      port: 16379
      password: ENC(6tqzYkJGBbg+s5nK08DtEA==)
      ssl: true
      keystore-path: "classpath:jks/keystore.jks"
      keystore-password: ENC(Xd7n2qQuM68TteVyGxU8QNX+XYg+mBd2)
    cluster-client:
      default-candidate: false
      type: cluster
      nodes:
        - "192.168.23.155:8001"
        - "192.168.23.155:8002"
        - "192.168.23.157:8003"
        - "192.168.23.157:8004"
        - "192.168.23.158:8005"
        - "192.168.23.158:8006"
      max-redirects: 5
      password: ENC(6tqzYkJGBbg+s5nK08DtEA==)
    codis-client:
      default-candidate: false
      type: codis
      database: 10
      zookeeper:
        connect-string:
          - "192.168.23.155:2181"
          - "192.168.23.157:2181"
          - "192.168.23.158:2181"
        proxy-dir: /jodis/codis-dev
        session-timeout-in-millis: 30000
      password: ENC(mv+UiKUaQO9cC9Q0/w5AWQ==)
```


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


## 4. 客户端使用方式


### 4.1. 客户端接口


虽然组件基于spring-data-redis进行开发，但并不建议直接使用其原生提供RedisTemplate类（组件中已屏蔽其初始化），而是提供了下面几个接口：


- RedisClient接口：用于连接Redis服务（standalone/sentinel/cluster），屏蔽了会造成服务端阻塞的方法、有执行风险的方法、sentinel服务命令和cluster服务命令。
- CodisClient接口：用于连接Codis集群服务，屏蔽了会造成服务端阻塞的方法、有执行风险的方法和[Codis不支持的命令](https://github.com/CodisLabs/codis/blob/release3.2/doc/unsupported_cmds.md)。
- UnsafeRedisClient接口：包含了接近于RedisTemplate的命令集，不建议使用。



接口的定义，摒除了大部分风险，也去除了部分冗余，使得Redis客户端使用起来更加轻便。


### 4.2. 客户端使用


```java
@slf4j
@Component
public class BizDemo1 {

    @Autowired
    private RedisClient<String, Object> redisStandaloneClient;

    /**
     * Set & Get操作
     */
    public void valueSetAndGet() {
        //key: demo:vs:01，如果配置了key-prefix，下面写法不变，但真实写入的key为"你的前缀名称:demo:vs:01"
        String key = KeyBuilder.create().name("demo").name("vs").name("01").build();
        redisStandaloneClient.delete(key);
        redisStandaloneClient.opsForValue().set(key, "测试01");
        log.info(redisStandaloneClient.hasKey(key));
        String result = (String) redisStandaloneClient.opsForValue().get(key);
        log.info("测试01: {}", result);
    }
}
```


- 绑定Redis客户端接口：RedisClient/CodisClient/UnsafeRedisClient(不建议)。
- 使用KeyBuilder类创建key，name之间默认以":"分割。



#### 4.2.1. 基础数据操作方法


如果你用过RedisTemplate，那么此类数据操作方法是基本一致的。


```java
redisClient.opsForValue().set(key, "测试02", Duration.ofSeconds(3));
redisClient.opsForValue().get(key);
//......
redisClient.opsForList().leftPush(key, "a");
redisClient.opsForList().leftPush(key, "b");
//......
```


- `opsForValue`：简单值的相关操作。
- `opsForList`：列表相关操作。
- `opsForHash`：哈希表相关操作。
- `opsForSet`：集合相关操作。
- `opsForZSet`：有序集合相关操作。
- `opsForGeo`：地理位置相关操作。



#### 4.2.2. 基于key绑定的基础数据操作方法


如果使用同一个key进行多项操作，那么可以使用**key绑定**的方式减少key的重复输入。


```java
BoundValueOperations<String, Object> boundValueOperations = redisClient.boundValueOps(key);
boundValueOperations.set("a");
//此处复用了绑定的key
String result1 = (String) boundValueOperations.get();
//......
BoundListOperations<String, Object> boundListOperations = redisClient.boundListOps(key);
boundListOperations.leftPush("k");
boundListOperations.leftPush("b");
boundListOperations.leftPush("c");
//此处复用了绑定的key
log.info("Size before pop: {}", boundListOperations.size());
String result = (String) boundListOperations.leftPop();
log.info("Size after pop: {}", boundListOperations.size());
```


#### 4.2.3. 基于Pipeline的数据操作方法


Pipeline的方式会将操作指令缓存在客户端，然后使用一个socket连接以批量提交的方式上送到服务端，并批量返回结果（在操作大量数据时，非常高效）。由于占用了客户端的内存，所以每批次的命令具体设置多少条需要有合理的预估。


这里使用接口中的`executePipelined`来执行批量命令，connection的开关本身受组件管理，无需手动调用。


```java
@Test
public void pipeline() {
    redisClient.executePipelined(new RedisCallback<Object>() {
        @Override
        public Object doInRedis(RedisConnection connection) throws DataAccessException {
            for (int i = 0; i < 100; i++) {
                connection.set(("demo:pipe:" + i).getBytes(), String.valueOf(new Random().nextLong()).getBytes());
            }
            return null;
        }
    });
    
    List<Object> results = redisClient.executePipelined(new RedisCallback<Object>() {
        @Override
        public Object doInRedis(RedisConnection connection) throws DataAccessException {
            //没必要调用
            //connection.openPipeline();
            for (int i = 0; i < 100; i++) {
                connection.get(("demo:pipe:" + i).getBytes());
            }
            //不要去调用
            //connection.closePipeline();
            return null;
        }
    });
    log.info("Pipelined values: {}", results);
}
```


- 使用RedisConnection直接操作的情况下，前缀属性（key-prefix）不会在key上进行附加。



输出日志：


```
Pipelined values: [1198692483339504511, 7517350612423976356, -234680408832128525, -1385247499640218210, and more...
```


#### 4.2.4. 事务操作方式


```java
/**
 * 事务操作
 */
public void executeWithTx(){
    List<Object> results = redisClient.execute(new SessionCallback<List<Object>>() {
        @Override
        public List<Object> execute(RedisOperations operations) throws DataAccessException {
            operations.multi();
            operations.opsForSet().add("demo:tx", "a");
            log.info("Get value in transaction: {}", operations.opsForSet().isMember("demo:tx", "a"));
            //返回包含了所有事务操作的结果
            return operations.exec();
        }
    });
    log.info("Results: {}", results);
    log.info("Get value after transaction: {}", redisClient.opsForSet().isMember("demo:tx", "a"));
}
```


输出日志：


```
Get value in transaction: null
Results: [1, true]
Get value after transaction: true
```


## 5. 使用注解操作缓存


通过整合spring-cache，组件也同时具备了基于注解的缓存操作能力。具体的缓存操作方式，可以参考[官方文档](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-caching.html)


### 5.1. 启用Caching相关注解


```yaml
spring:
  cache:
    enabled: true
    type: redis
```


通过设置`spring.cache.enabled`为`true`，`spring.cache.type`为`redis`来启用Redis Caching的相关注解。


### 5.2. Caching属性描述



| **属性** | **是否必填** | **默认值** | **描述** |
| :--- | :--- | :--- | :--- |
| **spring.cache.enabled** | 否 | false | 是否启用Caching相关注解 |
| **spring.cache.type** | 是 | none | 指定Caching类型，支持的类型请参考：`org.springframework.boot.autoconfigure.cache.CacheType`，由于这里是Redis相关组件，请使用`redis`作为`type`的值 |
| **spring.cache.redis.cache-null-values** | 否 | true | 运行缓存null值 |
| **spring.cache.redis.key-prefix** | 否 | none | key前缀 |
| **spring.cache.redis.use-key-prefix** | 否 | true | 启用key前缀 |
| **spring.cache.redis.time-to-live** | 否 | none | 缓存过期时间，参考 `java.time.Duration` 类的注释举例来定义标准值。 |

### 
### 5.3. 使用指定的cacheManager


当在使用Caching相关属性时，默认使用候选（`default-candidate`）客户端Bean使用的`RedisConnectionFactory`设置到CacheManager中。但如果需要指定非候选客户端Bean使用的`RedisConnectionFactory`，那么需要显式地指定`cacheManager`属性，`cacheManager`属性的值按照规则：`客户端Bean Name` + `.cacheManager`即可，如：使用的客户端的Bean是`sentinel-client`，那么它对应的cacheManager的Bean就是`sentinel-client.cacheManager`。


以下提供部分代码片段供参考：


```java
@Slf4j
@Component
public class CacheTemplate {
    @Cacheable(cacheNames = "test-cache100",sync = true)
    public String getFromStandalone(String id){
        log.info(">>> Get from standalone cache: ");
        return "standalone: ab";
    }

    @Cacheable(cacheNames = "test-cache200",sync = true, cacheManager = "sentinel-client.cacheManager")
    public String getFromSentinel(String id){
        log.info(">>> Get from sentinel cache: ");
        return "sentinel: ab";
    }

    @Cacheable(cacheNames = "test-cache300",sync = true, cacheManager = "codis-client.cacheManager")
    public String getFromCodis(String id){
        log.info(">>> Get from codis cache: ");
        return "codis: ab";
    }
}
```


特别说明，这里的`sync`属性，主要是在多线程环境下，某些操作可能使用相同参数同步调用。默认情况下，缓存不锁定任何资源，可能导致多次计算，而违反了缓存的目的。对于这些特定的情况，属性`sync`可以指示底层将缓存锁住，使只有一个线程可以进入计算，而其他线程堵塞，直到返回结果更新到缓存中。


## 6. 需要注意的问题


### 6.1. Redis多库使用建议


Redis的库（database）更多地是做为命名空间（namespace），且使用的是很不友好的数字命名（默认 0-15）。而且Redis作者本身也认为，当初设计database并不是个好主意。


以下为 Redis 作者的观点，引用自 [Google 网上论坛]（https://groups.google.com/d/topic/redis-db/vS5wX8X4Cjg/discussion）


> I understand how this can be useful, **but unfortunately I consider Redis multiple database errors my worst decision in Redis design at all**… without any kind of real gain, it makes the internals a lot more complex. The reality is that databases don't scale well for a number of reason, like active expire of keys and VM. If the DB selection can be performed with a string I can see this feature being used as a scalable O(1) dictionary layer, that instead it is not.



> With DB numbers, with a default of a few DBs, we are communication better what this feature is and how can be used I think. I hope that at some point we can drop the multiple DBs support at all, but I think it is probably too late as there is a number of people relying on this feature for their work.



所以，基于此原因，组件在设计的时候使用了`key-prefix`属性来为key添加前缀，而这里建议将`key-prefix`设置为你应用的名称（即`spring.application.name`）。之后，当你在通过提供的接口操作Redis命令时，会对key进行自动附加，比如：你添加了`demo:vs:one`，那么，实际存入到Redis服务端的可以是：`你的应用名称:demo:vs:one`，反之亦然。这里需要注意的是，当使用了RedisConnection时，此前缀无法附加在key上。


### 6.2. RedisCallback 与 SessionCallback 的使用建议


1. RedisCallback 建议用于非事务模式，且提供了原生的Redis Connection，权限较大，请谨慎使用。
2. SessionCallback 建议用于事务模式（同时提供了对@Transactional的支持），提供了封装的Redis操作。



### 6.3. Redis的发布与订阅


订阅与发布的功能，将以 [Spring Cloud Stream Binder](https://github.com/spring-cloud/spring-cloud-stream-binder-redis) 的形式，在消息中间件中提供。


### 6.4. Hash Tags


Hash Tags的作用是将一批含有其标记的Key设置到同一台Redis主机上，以便一些命令的运算需要。


在使用**Codis**时，建议在以下半支持（half-supported）的命令上使用（Redis Cluster不需要）。



| **命令类型** | **命令名称** |
| :---: | :--- |
| Lists | RPOPLPUSH |
| Sets | SDIFF |
|  | SINTER |
|  | SINTERSTORE |
|  | SMOVE |
|  | SUNION |
|  | SUNIONSTORE |
| Sorted Sets | ZINTERSTORE |
|  | ZUNIONSTORE |
| HyperLogLog | PFMERGE |
| Scripting | EVAL |
|  | EVALSHA |



具体写法：


```java
codisClient.opsForSet().add("demo:{vs}:15", "a");
codisClient.opsForSet().add("demo:{vs}:15", "f");
codisClient.opsForSet().add("demo:{vs}:15", "k");

codisClient.opsForSet().add("demo:{vs}:16", "n");
codisClient.opsForSet().add("demo:{vs}:16", "k");
codisClient.opsForSet().add("demo:{vs}:16", "c");

Set<String> d1Set = (Set) codisClient.opsForSet().difference("demo:{vs}:15", "demo:{vs}:16");
Assert.assertTrue(d1Set != null && d1Set.size() == 2 && d1Set.contains("a") && d1Set.contains("f"));

Set<String> d2Set = (Set) codisClient.opsForSet().difference("demo:{vs}:16", "demo:{vs}:15");
Assert.assertTrue(d2Set != null && d2Set.size() == 2 && d2Set.contains("n") && d2Set.contains("c"));
```


当一个key包含 `{}` 的时候，就不对整个key做hash，而`仅对 {} 包括的字符串做hash`。


假设hash算法为sha1(str)方法。对user:{user1}:ids和user:{user1}:tweets，其hash值都等同于`sha1(user1)`。


### 6.5. RedisClient的转换


在使用其他的Spring数据组件时，有依赖到Redis的时候，我们需要对组件定义的RedisClient进行类型转换，来进行适配。一般来说，需要适配的类为 `ReidsConnectionFactory` 和 `RedisTemplate` ，那么我们可以在配置中这样做：
```java
@Configuration
public class RedisXxxConfiguration {

    @Bean
    public RedisConnectionFactory redisConnectionFactory(RedisClient redisClient) {
        return ((RedisTemplate)redisClient).getConnectionFactory();
    }

    @Bean
    public RedisTemplate redisTemplate(RedisClient redisClient){
        return (RedisTemplate)redisClient;
    }
}
```


