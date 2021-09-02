# semak-elastic-job

> 已升级到3.0.0-RC1 - 当前文档不正确，需迭代

`semak-elastic-job`组件是一款基于当当网Elastic-Job-Lite进行定制的去中心化的分布式调度组件，其主要特性包括：


1. 支持丰富的作业类型（SimpleJob, DataflowJob, ScriptJob）。
1. 支持基于数据库存储的作业事件追踪。
1. 支持分布式调度。
1. 支持弹性扩容缩容。
1. 支持失效转移。
1. 支持错过执行作业重触发。
1. 支持作业分片一致性，保证同一分片在分布式环境中仅一个执行实例。
1. 支持基于分片的并行调度。
1. 更简单的JOB配置方式。
1. 运维平台。



## 1. 先决条件


### 1.1. 环境配置


1. Open JDK 1.8+，并已配置有效的环境变量。
1. Maven 3.3.x+，并已配置有效的环境变量。
1. 安装Zookeeper 3.4.6或以上版本，并以集群(2n+1个节点)方式提供高可用的服务。



### 1.2. Maven依赖配置


```xml
<dependency>
    <groupId>com.github.semak.elastic.job</groupId>
    <artifactId>semak-elastic-job-spring-boot-starter</artifactId>
    <version>最新RELEASE版本</version>
</dependency>
```


## 2. 核心理念


### 2.1. 分布式调度


`semak-elastic-job`组件并无作业调度中心节点，而是基于部署作业框架的程序在到达相应时间点时各自触发调度。


注册中心仅用于作业注册和监控信息存储。而主作业节点仅用于处理分片和清理等功能。


### 2.2. 作业高可用


`semak-elastic-job`组件提供最安全的方式执行作业。将分片总数设置为1，并使用多于1台的服务器执行作业，作业将会以1主n从的方式执行。


一旦执行作业的服务器崩溃，等待执行的服务器将会在下次作业启动时替补执行。开启失效转移功能效果更好，可以保证在本次作业执行时崩溃，备机立即启动替补执行。


### 2.3. 最大限度利用资源


`semak-elastic-job`组件也提供最灵活的方式，最大限度的提高执行作业的吞吐量。将分片项设置为大于服务器的数量，最好是大于服务器倍数的数量，作业将会合理的利用分布式资源，动态的分配分片项。


例如：3台服务器，分成10片，则分片项分配结果为服务器A=0,1,2;服务器B=3,4,5;服务器C=6,7,8,9。 如果服务器C崩溃，则分片项分配结果为服务器A=0,1,2,3,4;服务器B=5,6,7,8,9。在不丢失分片项的情况下，最大限度的利用现有资源提高吞吐量。


## 3. 整体架构


![image.png](https://cdn.nlark.com/yuque/0/2020/png/1241873/1591868172196-922be386-7c90-4b86-b095-050898fc31f8.png#align=left&display=inline&height=613&margin=%5Bobject%20Object%5D&name=image.png&originHeight=613&originWidth=1037&size=200187&status=done&style=none&width=1037)


## 4. 作业定义


当前支持3种类型的作业，下面分别进行说明。需要注意的是，下面的作业，除了ScriptJob外，定义时除了实现必要的接口之外，还需要添加`@Component`注解来声明为一个Spring的Bean。


### 4.1. Simple类型作业


意为简单实现，未经任何封装的类型。需实现`com.dangdang.ddframe.job.api.simple.SimpleJob`接口。该接口仅提供单一方法用于覆盖，此方法将定时执行。与Quartz原生接口相似，但提供了弹性扩缩容和分片等功能。


```java
@Component
public class DemoSimpleJob implements SimpleJob {
    
    @Override
    public void execute(ShardingContext context) {
        switch (context.getShardingItem()) {
            case 0: 
                // do something by sharding item 0
                break;
            case 1: 
                // do something by sharding item 1
                break;
            case 2: 
                // do something by sharding item 2
                break;
            // case n: ...
        }
    }
}
```


### 4.2. Dataflow类型作业


Dataflow类型用于处理数据流，需实现`com.dangdang.ddframe.job.api.dataflow.DataflowJob`接口。该接口提供2个方法可供覆盖，分别用于抓取(fetchData)和处理(processData)数据。


```java
@Component
public class DemoDataFlowJob implements DataflowJob<Foo> {
    
    @Override
    public List<Foo> fetchData(ShardingContext context) {
        switch (context.getShardingItem()) {
            case 0: 
                List<Foo> data = // get data from database by sharding item 0
                return data;
            case 1: 
                List<Foo> data = // get data from database by sharding item 1
                return data;
            case 2: 
                List<Foo> data = // get data from database by sharding item 2
                return data;
            // case n: ...
        }
    }
    
    @Override
    public void processData(ShardingContext shardingContext, List<Foo> data) {
        // process data
        // ...
    }
}
```


#### 4.2.1. 流式处理


可通过设置配置项`spring.lite-job.jobs.<jobName>.streaming-process`为`true`来开启流式处理。


流式处理数据只有**fetchData**方法的返回值为null或集合长度为空时，作业才停止抓取，否则作业将一直运行下去。


非流式处理数据则只会在每次作业执行过程中执行一次**fetchData**方法和**processData**方法，随即完成本次作业。


如果采用流式作业处理方式，建议**processData**处理数据后更新其状态，避免**fetchData**再次抓取到，从而使得作业永不停止。 流式数据处理参照**TbSchedule**设计，适用于不间歇的数据处理。


### 4.3. Script类型作业


Script类型作业意为脚本类型作业，支持shell，python，perl等所有类型脚本。只需通过控制台或代码配置scriptCommandLine即可，无需编码。执行脚本路径可包含参数，参数传递完毕后，作业框架会自动追加最后一个参数为作业运行时信息。


```bash
#!/bin/bash
echo sharding execution context is $*
```


作业运行时输出


```
sharding execution context is {“jobName”:“scriptElasticDemoJob”,“shardingTotalCount”:10,“jobParameter”:“”,“shardingItem”:0,“shardingParameter”:“A”}
```


## 5. 作业配置


定义完作业类或脚本后，我们需要进行一下简单配置，以启用作业。


```yaml
spring:
  application:
    name: job-test
  lite-job:
    zk:
      connect-string: 127.0.0.1:2181
      namespace: lite-job-test/${spring.application.name}
    base-job-definition:
      disabled: false
      # 开启事件追踪
      enable-trace-rdb: false
      job-type: simple
      overwrite: true
      sharding-total-count: 1
      misfire: true
      job-listener-class:
        - com.github.semak.elastic.job.test.listener.JobListener
    jobs:
      job-a:
        alias: 作业A
        job-class: com.github.semak.elastic.job.test.job.DemoSimpleJob
        #sharding-total-count: 3
        #sharding-item-parameters: 0=shanghai,1=beijing,2=guangzhou
        cron: "0/10 * * * * ?"
        job-parameter: "作业A-测试参数1"
        description: "DEMO A 描述"
      job-b:
        alias: 作业B
        job-type: dataflow
        job-class: com.github.semak.elastic.job.test.job.DemoDataflowJob
        cron: "0/20 * * * * ?"
        job-parameter: "作业B-测试参数1"
        description: "DEMO B 描述"
        streaming-process: true
      job-c:
        alias: 作业C
        job-type: script
        cron: "0/30 * * * * ?"
        description: "DEMO C 描述"
        script-command-line: "/Users/brevy/Project/Github/semak/semak-elastic-job/semak-elastic-job-test/src/main/resources/demo.sh"
```


### 5.1. 配置描述
| **属性** | **必填** | **默认值** | **描述** |
| :--- | :--- | :--- | :--- |
| **spring.lite-job.enabled** | 否 | true | 开启分布式任务调度功能 |
| **spring.lite-job.zk.connect-string** | 是 |   | zookeeper地址连接串 |
| **spring.lite-job.zk.namespace** | 是 |   | zookeeper命名空间 |
| **spring.lite-job.base-job-definition.*** | 否 |   | 作业通用配置，可被具体作业所继承并覆盖，包含属性同具体作业一致 |
| **spring.lite-job.jobs.<jobName>.disabled** | 否 | false | 作业是否启动时禁止 |
| **spring.lite-job.jobs.<jobName>.alias** | 否 |   | Job别名 |
| **spring.lite-job.jobs.<jobName>.job-class** | 是 |   | 作业实现类，需实现 `com.dangdang.ddframe.job.api.ElasticJob`接口，**script类型的作业不需要填写此属性** |
| **spring.lite-job.jobs.<jobName>.cron** | 是 |   | CRON表达式，定义作业执行时间 |
| **spring.lite-job.jobs.<jobName>.enable-trace-rdb** | 否 | false | 启用作业事件追踪 |
| **spring.lite-job.jobs.<jobName>.job-parameter** | 否 |   | 作业自定义参数 |
| **spring.lite-job.jobs.<jobName>.description** | 否 |   | 作业描述 |
| **spring.lite-job.jobs.<jobName>.streaming-process** | 否 | false | **dataflow类型的作业专用**，开启流式处理 |
| **spring.lite-job.jobs.<jobName>.script-command-line** | 是 |   | **script类型的作业专用**， 执行脚本的绝对路径 |
| **spring.lite-job.jobs.<jobName>.job-type** | 是 |   | Job类型分为：**SIMPLE**, **DATAFLOW**, **SCRIPT** |
| **spring.lite-job.jobs.<jobName>.overwrite** | 否 | true | 本地配置是否可覆盖注册中心配置 |
| **spring.lite-job.jobs.<jobName>.misfire** | 否 | true | 是否开启错过任务重新执行 |
| **spring.lite-job.jobs.<jobName>.job-listener-class** | 否 |   | 作业监听器实现类（可以列表的形式配置多个） |
| **spring.lite-job.jobs.<jobName>.job-sharding-strategy-class** | 否 | com.dangdang.ddframe.job.lite.api.strategy.impl.AverageAllocationJobShardingStrategy | 设置作业分片策略实现类全路径 |
| **spring.lite-job.jobs.<jobName>.sharding-total-count** | 否 | 1 | 作业分片总数 |
| **spring.lite-job.jobs.<jobName>.sharding-item-parameters** | 否 |   | 设置分片序列号和个性化参数对照表 |
| **spring.lite-job.jobs.<jobName>.max-time-diff-seconds** | 否 | -1 | 最大允许的本机与注册中心的时间误差秒数。如果时间误差超过配置秒数则作业启动时将抛异常，配置为-1表示不校验时间误差修复作业服务器不一致状态服务调度间隔时间，配置为小于1的任意值表示不执行修复(默认：10) |
| **spring.lite-job.jobs.<jobName>.reconcile-interval-minutes** | 否 | 10 | 修复作业服务器不一致状态服务调度间隔时间，配置为小于1的任意值表示不执行修复，单位：分钟 |

### 
### 5.2. 自诊断修复


在分布式的场景下由于网络、时钟等原因，可能导致Zookeeper的数据与真实运行的作业产生不一致，这种不一致通过正向的校验无法完全避免。需要另外启动一个线程定时校验注册中心数据与真实作业状态的一致性，即维持Job的最终一致性。


在网络不稳定的环境下可能有的作业分片并未执行，可以重启修复。但在提供了`reconcile-interval-minutes`属性配置后，可以设置修复状态服务执行间隔分钟数，修复作业服务器不一致状态，默认每10分钟检测并修复一次。


## 6. 分片策略


`semak-elastic-job`组件默认提供了多种作业的分片策略。


### 6.1. AverageAllocationJobShardingStrategy


#### 6.1.1. 策略类全路径


**com.dangdang.ddframe.job.lite.api.strategy.impl.AverageAllocationJobShardingStrategy**


#### 6.1.2. 策略说明


基于平均分配算法的分片策略，也是**默认的分片策略**。


如果分片不能整除，则不能整除的多余分片将依次追加到序号小的服务器。如：


如果有3台服务器，分成9片，则每台服务器分到的分片是：**1=[0,1,2], 2=[3,4,5], 3=[6,7,8]**


如果有3台服务器，分成8片，则每台服务器分到的分片是：**1=[0,1,6], 2=[2,3,7], 3=[4,5]**


如果有3台服务器，分成10片，则每台服务器分到的分片是：**1=[0,1,2,9], 2=[3,4,5], 3=[6,7,8]**


### 6.2. OdevitySortByNameJobShardingStrategy


#### 6.2.1. 策略类全路径


**com.dangdang.ddframe.job.lite.api.strategy.impl.OdevitySortByNameJobShardingStrategy**


#### 6.2.2. 策略说明


根据作业名的哈希值奇偶数决定IP升降序算法的分片策略。


作业名的哈希值为奇数则IP升序。


作业名的哈希值为偶数则IP降序。


用于不同的作业平均分配负载至不同的服务器。


**AverageAllocationJobShardingStrategy**的缺点是，一旦分片数小于作业服务器数，作业将永远分配至IP地址靠前的服务器，导致IP地址靠后的服务器空闲。而**OdevitySortByNameJobShardingStrategy**则可以根据作业名称重新分配服务器负载。如：


如果有3台服务器，分成2片，作业名称的哈希值为奇数，则每台服务器分到的分片是：**1=[0], 2=[1], 3=[]**


如果有3台服务器，分成2片，作业名称的哈希值为偶数，则每台服务器分到的分片是：**3=[0], 2=[1], 1=[]**


### 6.3. RotateServerByNameJobShardingStrategy


#### 6.3.1. 策略类全路径


**com.dangdang.ddframe.job.lite.api.strategy.impl.RotateServerByNameJobShardingStrategy**


#### 6.3.2. 策略说明


根据作业名的哈希值对服务器列表进行轮转的分片策略。


### 6.4. 自定义分片策略


实现JobShardingStrategy接口并实现sharding方法，接口方法参数为作业服务器IP列表和分片策略选项，分片策略选项包括作业名称，分片总数以及分片序列号和个性化参数对照表，可以根据需求定制化自己的分片策略。


### 6.5. 配置分片策略


与配置通常的作业属性相同，在spring命名空间或者JobConfiguration中配置jobShardingStrategyClass属性，属性值是作业分片策略类的全路径。


## 7. 作业监听器


作业可通过配置多个任务监听器，在任务执行前和执行后执行监听的方法。监听器分为每台作业节点均执行和分布式场景中仅单一节点执行2种。


### 7.1. 每台作业节点均执行的监听


若作业处理作业服务器的文件，处理完成后删除文件，可考虑使用每个节点均执行清理任务。此类型任务实现简单，且无需考虑全局分布式任务是否完成，请尽量使用此类型监听器。


#### 7.1.1. 定义监听器


监听器需实现接口`com.dangdang.ddframe.job.lite.api.listener.ElasticJobListener`。


```java
public class MyElasticJobListener implements ElasticJobListener {
    
    @Override
    public void beforeJobExecuted(ShardingContexts shardingContexts) {
        // do something ...
    }
    
    @Override
    public void afterJobExecuted(ShardingContexts shardingContexts) {
        // do something ...
    }
}
```


#### 7.1.2. 配置监听器


通过`spring.lite-job.jobs.<jobName>.job-listener-class`来添加监听器。


```yaml
spring:
  lite-job:
    jobs:
      job-a:
        alias: 作业A
        job-class: com.github.semak.elastic.job.test.job.DemoSimpleJob
        cron: "0/10 * * * * ?"
        job-listener-class:
          - com.github.semak.elastic.job.test.listener.JobListener
```


### 7.2. 分布式场景中仅单一节点执行的监听


若作业处理数据库数据，处理完成后只需一个节点完成数据清理任务即可。此类型任务处理复杂，需同步分布式环境下作业的状态，提供了超时设置来避免作业不同步导致的死锁，**请谨慎使用**。


#### 7.2.1. 定义监听器


监听器需继承抽象类`com.dangdang.ddframe.job.lite.api.listener.AbstractDistributeOnceElasticJobListener`。


```java
public class TestDistributeOnceElasticJobListener extends AbstractDistributeOnceElasticJobListener {
    
    public TestDistributeOnceElasticJobListener(long startTimeoutMills, long completeTimeoutMills) {
        super(startTimeoutMills, completeTimeoutMills);
    }
    
    @Override
    public void doBeforeJobExecutedAtLastStarted(ShardingContexts shardingContexts) {
        // do something ...
    }
    
    @Override
    public void doAfterJobExecutedAtLastCompleted(ShardingContexts shardingContexts) {
        // do something ...
    }
}
```


通过`spring.lite-job.jobs.<jobName>.job-listener-class`来添加监听器。


```yaml
spring:
  lite-job:
    jobs:
      job-a:
        alias: 作业A
        job-class: com.github.semak.elastic.job.test.job.DemoSimpleJob
        cron: "0/10 * * * * ?"
        job-listener-class:
          - com.github.semak.elastic.job.test.listener.JobOnceListener
```


## 8. 事件追踪（可选功能）


`semak-elastic-job`组件提供了事件追踪功能，可通过事件订阅的方式处理调度过程的重要事件，用于查询、统计和监控。目前主要提供基于关系型数据库的两种事件订阅方式来记录事件。


### 8.1.  开启事件追踪


```yaml
spring:
  lite-job:
    base-job-definition:
      disabled: false
      # 开启事件追踪
      enable-trace-rdb: true
```


开启事件追踪的同时，需要配置对应的数据源：


```yaml
spring:
  application:
    name: job-test
  datasource:
    url: jdbc:mysql://localhost:3306/test?useUnicode=true&characterEncoding=UTF-8&useSSL=false
    username: root
    password: 12345678
    hikari:
      minimum-idle: 2
      maximum-pool-size: 5
      auto-commit: true
      idle-timeout: 60000
      pool-name: EjlRdbHikariCP
      max-lifetime: 600000
      connection-timeout: 12000
      connection-test-query: SELECT 1
      leak-detection-threshold: 5000
  lite-job:
    zk:
      connect-string: 127.0.0.1:2181
      namespace: lite-job-test/${spring.application.name}
    base-job-definition:
      disabled: false
      # 开启事件追踪
      enable-trace-rdb: true
```


- 用于分布式调度功能的数据源一般为候选（Candidate）数据源。
- 以上例子是基于Spring-JDBC定义的数据源，建议配置semak系列组件中的HikariCP或Druid数据源组件来使用。



### 8.2. 事件追踪表


时间追踪主要涉及2张表，`JOB_EXECUTION_LOG`和`JOB_STATUS_TRACE_LOG`，请提前在MySQL数据库中建立好。原始SQL脚本如下：


```sql
CREATE TABLE `JOB_EXECUTION_LOG`
(
    `id`               varchar(40)  NOT NULL,
    `job_name`         varchar(100) NOT NULL,
    `task_id`          varchar(255) NOT NULL,
    `hostname`         varchar(255) NOT NULL,
    `ip`               varchar(50)  NOT NULL,
    `sharding_item`    int(11)      NOT NULL,
    `execution_source` varchar(20)  NOT NULL,
    `failure_cause`    varchar(4000)         DEFAULT NULL,
    `is_success`       int(11)      NOT NULL,
    `start_time`       timestamp    NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    `complete_time`    timestamp    NULL     DEFAULT NULL,
    PRIMARY KEY (`id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;

CREATE TABLE `JOB_STATUS_TRACE_LOG`
(
    `id`               varchar(40)  NOT NULL,
    `job_name`         varchar(100) NOT NULL,
    `original_task_id` varchar(255) NOT NULL,
    `task_id`          varchar(255) NOT NULL,
    `slave_id`         varchar(50)  NOT NULL,
    `source`           varchar(50)  NOT NULL,
    `execution_type`   varchar(20)  NOT NULL,
    `sharding_item`    varchar(100) NOT NULL,
    `state`            varchar(20)  NOT NULL,
    `message`          varchar(4000)     DEFAULT NULL,
    `creation_time`    timestamp    NULL DEFAULT NULL,
    PRIMARY KEY (`id`),
    KEY `TASK_ID_STATE_INDEX` (`task_id`, `state`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;
```


#### 8.2.1. JOB_EXECUTION_LOG字段含义
| **字段名称** | **字段类型** | **是否必填** | **描述** |
| :--- | :--- | :--- | :--- |
| id | VARCHAR(40) | 是 | 主键 |
| job_name | VARCHAR(100) | 是 | 作业名称 |
| task_id | VARCHAR(1000) | 是 | 任务名称,每次作业运行生成新任务 |
| hostname | VARCHAR(255) | 是 | 主机名称 |
| ip | VARCHAR(50) | 是 | 主机IP |
| sharding_item | INT | 是 | 分片项 |
| execution_source | VARCHAR(20) | 是 | 作业执行来源。可选值为NORMAL_TRIGGER, MISFIRE, FAILOVER |
| failure_cause | VARCHAR(2000) | 否 | 执行失败原因 |
| is_success | BIT | 是 | 是否执行成功 |
| start_time | TIMESTAMP | 是 | 作业开始执行时间 |
| complete_time | TIMESTAMP | 否 | 作业结束执行时间 |



**JOB_EXECUTION_LOG**记录每次作业的执行历史。分为两个步骤：


- 作业开始执行时向数据库插入数据，除`failure_cause`和`complete_time`外的其他字段均不为空。
- 作业完成执行时向数据库更新数据，更新`is_success`, `complete_time`和`failure_cause`(如果作业执行失败)。



#### 8.2.2. JOB_STATUS_TRACE_LOG字段含义
| **字段名称** | **字段类型** | **是否必填** | **描述** |
| :--- | :--- | :--- | :--- |
| id | VARCHAR(40) | 是 | 主键 |
| job_name | VARCHAR(100) | 是 | 作业名称 |
| original_task_id | VARCHAR(1000) | 是 | 原任务名称 |
| task_id | VARCHAR(1000) | 是 | 任务名称 |
| slave_id | VARCHAR(1000) | 是 | 执行作业服务器的名称，Lite版本为服务器的IP地址，Cloud版本为Mesos执行机主键 |
| source | VARCHAR(50) | 是 | 任务执行源，可选值为CLOUD_SCHEDULER, CLOUD_EXECUTOR, LITE_EXECUTOR |
| execution_type | VARCHAR(20) | 是 | 任务执行类型，可选值为NORMAL_TRIGGER, MISFIRE, FAILOVER |
| sharding_item | VARCHAR(255) | 是 | 分片项集合，多个分片项以逗号分隔 |
| state | VARCHAR(20) | 是 | 任务执行状态，可选值为TASK_STAGING, TASK_RUNNING, TASK_FINISHED, TASK_KILLED, TASK_LOST, TASK_FAILED, TASK_ERROR |
| message | VARCHAR(2000) | 是 | 相关信息 |
| creation_time | TIMESTAMP | 是 | 记录创建时间 |



**JOB_STATUS_TRACE_LOG**记录作业状态变更痕迹表。可通过每次作业运行的`task_id`查询作业状态变化的生命周期和运行轨迹。


## 9. 运维平台


### 9.1. 部署方式


1. 通过对`semak-elastic-job-console`进行打包，在target目录下会生成一个`semak-elastic-job-console-xxx-assembly.tar.gz`的压缩包。
1. 将压缩包放在需要运行的位置，解压压缩包，进入bin目录执行`start.sh`启动即可。默认服务端口为`8899`，如果需要改变端口，可以这样启动`./start.sh -p 8888`。
1. 如需要修改访问鉴权，请提前通过`conf/auth.properties`进行配置。



### 9.2. 登录


提供两种账户，管理员及访客，管理员拥有全部操作权限，访客仅拥有察看权限。


默认管理员用户名和密码是`root/root`，访客用户名和密码是`guest/guest`。


### 9.3. 功能列表


1. 登录安全控制。
1. 注册中心、事件追踪数据源管理。
1. 快捷修改作业设置。
1. 作业和服务器维度状态查看。
1. 操作作业禁用\启用、停止和删除等生命周期。
1. 事件追踪查询。



### 9.4. 设计理念


运维平台和Job本身并无直接关系，是通过读取作业注册中心数据展现作业状态，或更新注册中心数据修改全局配置。


控制台只能控制作业本身是否运行，但不能控制作业进程的启动，因为控制台和作业本身服务器是完全分离的，控制台并不能控制作业服务器。


### 9.5. 不支持项


- 添加作业：作业在首次运行时将自动添加，并无作业分发功能。



### 9.6. 我们如何观察和调整作业


到Elastic Job Lite Console查看你注册的Job。


在**全局配置**中添加你的注册中心：


![image.png](https://cdn.nlark.com/yuque/0/2020/png/1241873/1591868282213-27a3c628-3c41-46d7-b031-1870dd0bbbb5.png#align=left&display=inline&height=458&margin=%5Bobject%20Object%5D&name=image.png&originHeight=458&originWidth=600&size=20221&status=done&style=none&width=600)


![image.png](https://cdn.nlark.com/yuque/0/2020/png/1241873/1591868292520-abe6294c-ffcc-436b-8187-514f7868c760.png#align=left&display=inline&height=281&margin=%5Bobject%20Object%5D&name=image.png&originHeight=281&originWidth=2318&size=42900&status=done&style=none&width=2318)


默认命名空间定义规则：`lite-job-环境/应用名称`，如：xj.demo + dev环境，应为：`lite-job-dev/xj.demo`


然后，我们可以切换到`作业维度`查看和修改作业：


![image.png](https://cdn.nlark.com/yuque/0/2020/png/1241873/1591868303539-75802451-920a-43ed-b932-0d12c88709c8.png#align=left&display=inline&height=479&margin=%5Bobject%20Object%5D&name=image.png&originHeight=479&originWidth=2546&size=81632&status=done&style=none&width=2546)


![image.png](https://cdn.nlark.com/yuque/0/2020/png/1241873/1591868313282-f2ff51d7-556c-48a9-8e3e-8a13e07862f9.png#align=left&display=inline&height=685&margin=%5Bobject%20Object%5D&name=image.png&originHeight=685&originWidth=1251&size=85891&status=done&style=none&width=1251)
