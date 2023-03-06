# semak-tsdb-tdengine

`semak-tsdb-tdengine`是一款基于时序数据库[tdengine](https://www.taosdata.com/)的客户端组件，其主要特性包括：

1. 可以直接引用JDBC数据源（按照tdengine的配置规则），包括对各类连接池的良好兼容。
2. 以SQL-Like语句对时序数据库进行操作，简单方便。
3. 基于SpringBoot组件规范进行开发，通过依赖starter和一些简单的属性配置（properties或yaml）即可启用客户端。
4. 支持Native和Restful两种模式的JDBC驱动（由于性能上的差异，推荐使用Native模式）。



## 1. 先决条件


### 1.1. 环境配置


1. Open JDK 1.8+，并已配置有效的环境变量。
2. Maven 3.3.x+，并已配置有效的环境变量。
3. 使用Native模式的JDBC驱动时，需要在操作系统层面预装客户端驱动程序，具体参考：https://docs.taosdata.com/connector/



### 1.2. Maven依赖配置

```xml
<dependency>
    <groupId>com.github.semak.tsdb</groupId>
    <artifactId>semak-tsdb-tdengine-spring-boot-starter</artifactId>
    <version>最新RELEASE版本</version>
</dependency>
```



## 2. 数据源引用

### 2.1. 基于semak-hikaricp构建的数据源

```yaml
spring:
  datasource:
    datasource-type: hikari
    datasource-names:
      - "pri-ds"
    base-datasource-configure:
      minimum-idle: 3
      maximum-pool-size: 5
      auto-commit: true
      idle-timeout: 60000
      max-lifetime: 600000
      connection-timeout: 12000
      leak-detection-threshold: 5000
    pri-ds:
      connection-properties:
        useUnicode: true
        characterEncoding: utf8
        autoReconnect: true
        failOverReadOnly: false
        useSSL: false
        serverTimezone: Asia/Shanghai
        zeroDateTimeBehavior: CONVERT_TO_NULL
      default-candidate: true
      driver-class-name: com.taosdata.jdbc.TSDBDriver
      jdbc-url: jdbc:TAOS://tdengine-0.polarwin.cc:6030/test
      username: root
      password: taosdata
      connection-test-query: SELECT 1
```

* 基于Native模式的JDBC驱动按照格式（**推荐**）：`jdbc:[TAOS]://[host_name]:[port]/[database_name]`
* 基于Restful模式的JDBC驱动按照格式：`jdbc:[TAOS-RS]://[host_name]:[port]/[database_name]`
* **需要特别注意的是**，当存在多个数据源的情况下，请将时序数据库声明的数据源对象的`default-candidate`设置为`false`



### 2.2. 引用时序数据库的数据源

```yaml
spring:
	...
  tsdb:
    tdengine:
      enabled: true
      datasource-name: "pri-ds"
```



### 2.3. tsdb属性描述

| 属性                | 是否必填 | 默认值 | 描述                     |
| :------------------ | :------- | :----- | :----------------------- |
| **enabled**         | 否       | true   | 是否启用tdengine客户端   |
| **datasource-name** | 是       |        | 定义好的数据源的BeanName |



## 3. 使用方式

> 对于如何创建超级表、表等基础知识，请参考：https://docs.taosdata.com/taos-sql/

### 3.1. TsdbTemplate

组件配置完成后，可以直接引用Bean：`com.github.semak.tsdb.tdengine.TsdbTemplate`。由于其继承自JdbcTemplate，所以对于TsdbTemplate，其方法的应用与前者并无二致。但基于时序数据库操作的一些特性（主要集中于查询和插入动作），对于其方法的应用以及扩展，下面会进行具体的描述。



### 3.2. 数据查询

#### 3.2.1. 查询反序列化为实体对象的集合

定义对象

```java
@Data
public class ANode {

    private long ts;

    private int val;

    private long device;

    private String name;

    private int groupId;
}
```

定义对象映射

```java
public class ANodeRowMapper extends AbstractCustomRowMapper implements RowMapper<ANode> {

    @Override
    public ANode mapRow(ResultSet rs, int rowNum) throws SQLException {
        return doRowMapping(rs, ANode.class);
    }
}
```

执行查询

```java
@Autowired
private TsdbTemplate tsdbTemplate;

List<ANode> aNodes = tsdbTemplate.query("select * from anode where groupId = 1", new ANodeRowMapper());
log.info(">>> Result: {}", aNodes);
```

查询结果：

```
>>> Result: [ANode(ts=1543716081000, val=32957, device=99951, name=t, groupId=1), ANode(ts=1543716081000, val=213, device=99951, name=b, groupId=1), ANode(ts=1543716081000, val=32937, device=99951, name=a, groupId=1), ANode(ts=1543716081022, val=32938, device=99951, name=a, groupId=1), ANode(ts=1543716081024, val=32940, device=99951, name=a, groupId=1)]
```



#### 3.2.2. 查询反序列化为二维数组的集合

二维数组结构的返回更适合于前端图表的展示，如使用EChart来快速构建图表。

这里直接使用`com.github.semak.tsdb.tdengine.ArrayRowMapper`作为二维数组结构的反序列化来执行查询

```java
@Autowired
private TsdbTemplate tsdbTemplate;

List<Object[]> aNodes = tsdbTemplate.query("select * from anode where groupId = 1", new ArrayRowMapper());
log.info(">>> Result: {}", JSON.toJSON(aNodes));
```

查询结果：

```
>>> Result: [[1543716081000,32957,99951,"t",1],[1543716081000,213,99951,"b",1],[1543716081000,32937,99951,"a",1],[1543716081022,32938,99951,"a",1],[1543716081024,32940,99951,"a",1]]
```



#### 3.2.3. 使用PrepareStatment模式进行查询操作

```java
@Autowired
private TsdbTemplate tsdbTemplate;

String psSql = "select * from anode where groupId = ?";
int groupId = 1;
List<Object[]> aNodes = tsdbTemplate.query(connection -> {
    PreparedStatement ps = connection.prepareStatement(psSql);
    ps.setInt(1, groupId);
    return ps;
}, new ArrayRowMapper());
log.info(">>> Result: {}", JSON.toJSON(aNodes));
```



### 3.3. 数据插入

> tdengine中，除了超级表需要定义外，普通表如果引用了超级表作为模板，在插入时若不存在此表，则可以自动创建而无需额外定义。

```java
@Autowired
private TsdbTemplate tsdbTemplate;

String insertSql = "INSERT INTO current_c USING anode TAGS (?, ?, ?) VALUES (?, ?)";
long device = 99952L;
String name = "c";
int groupId = 2;
long ts = 1543716081000L;
int val = 219;

int inserted = tsdbTemplate.update(connection -> {
    PreparedStatement ps = connection.prepareStatement(insertSql);
    ps.setLong(1, device);
    ps.setString(2, name);
    ps.setInt(3, groupId);
    ps.setLong(4, ts);
    ps.setInt(5, val);
    return ps;
});
```



