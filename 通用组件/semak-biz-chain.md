# semak-biz-chain

`semak-biz-chain`组件是一款将责任链模式和命令模式应用与业务开发的组件，可以为业务功能的模块化、链式执行、异常处理等提供强有力的支撑。其特性主要如下：


1. 通过配置方式来实现业务单元定义。

1. 支持完整链路描述。
1. 支持业务单元级别的异常处理（异常处理链与正常执行链执行方向相反）。



## 1. 先决条件
### 1.1. 环境配置

1. Open JDK 1.8+，并已配置有效的环境变量。
1. Maven 3.3.x+，并已配置有效的环境变量。



### 1.2. Maven依赖配置
```xml
<dependency>
    <groupId>com.github.semak.biz</groupId>
    <artifactId>semak-biz-chain-spring-boot-starter</artifactId>
    <version>最新RELEASE版本</version>
</dependency>
```




## 2. 定义业务单元
```java
/**
 * 业务1
 * @author caobin
 * @version 1.0 2020.07.02
 */
@Slf4j
@Component
public class BizUnit1 extends BizUnit {

    @Autowired
    private DemoDep demoDep;

    @Override
    protected ProcessResult process(BizContext context) throws Exception {
        log.info("a: {}", context.getContextValue("a", String.class).get());
        log.info("Does b value exist?: {}", context.getContextValue("b", String.class).isPresent());
        context.put("c", "测试Biz1");
        return ProcessResult.CONTINUE_PROCESSING;
    }

    @Override
    protected PostProcessResult postprocess(BizContext context, Exception exception) {
        log.info("post process 1");
        return PostProcessResult.RETHROWN_EXCEPTION;
    }
}
```

- 业务单元类必须继承`com.github.semak.biz.chain.core.BizUnit`抽象类。

- 业务单元必须以`@Component`注解进行标注，托管到Spring IOC容器。

- `process`方法是业务`正向`处理单元，返回结果通过枚举`com.github.semak.biz.chain.core.result.ProcessResult`进行设置。
   - `CONTINUE_PROCESSING`: 继续处理下一业务单元。

   - `PROCESSING_COMPLETE`: 本业务单元处理完成后结束，不再继续传递。

   - 返回`null`: 默认继续处理下一单元，但建议显式设置。

- `postprocess`方法是业务`反向`处理单元，一般建议做资源释放、脏数据清理、异常处理等逻辑。正向业务单元`执行完成`或`抛出异常`时，整条业务链会反向调用`postprocess`方法进行后处理(异常时，从抛出异常的业务单元对应的`postprocess`开始进行反向调用)。返回结果通过枚举`com.github.semak.biz.chain.core.result.PostProcessResult`进行设置。
   - `HANDLED_EXCEPTION`: 异常被后处理方法处理，不再向外层抛出。

   - `RETHROWN_EXCEPTION`: 异常被后处理方法处理，继续往上层抛出。当需要整条业务链进行异常抛出时，在反向的`每一个业务单元`中，都需要返回`RETHROWN_EXCEPTION`，才可以保证异常链路的正常传递。

   - `CONTINUE_PROCESSING`: 清理、资源释放等非异常处理操作完成后的返回值。

   - 返回`null`: 默认`CONTINUE_PROCESSING`，但建议显式设置。

- 通过`context.getContextValue`获取执行链上下文中的参数，返回值通过Java 8的`Optional`进行包装，提供对null值的优雅处理。`Optional`对象常用方法有:
   - `get`: 获取包装对象的真实值，当为null时则抛错。

   - `isPresent`: 判断当前要获取的包装对象的真实值是否为null，非null则返回true。

   - `orElse`: 当包装对象的真实值为null时，返回一个指定的默认值。



## 3. 配置业务单元
```yaml
spring:
  biz:
    chain:
      enabled: true
      biz-chains:
        bizChainC1:
          - com.github.semak.biz.chain.unit.BizUnit1
          - com.github.semak.biz.chain.unit.BizUnit3
        bizChainC2:
          - com.github.semak.biz.chain.unit.BizUnit1
          - com.github.semak.biz.chain.unit.BizUnit2
          - com.github.semak.biz.chain.unit.BizUnit3
```
**配置描述**

| **属性** | **数据类型** | **必填** | **默认值** | **描述** |
| --- | --- | --- | --- | --- |
| **spring.biz.chain.enabled** | boolean | 否 | true | 启用业务链功能 |
| **spring.biz.chain.biz-chains.&lt;chainName&gt;** | String | 否 |  | 业务链名称（**也是BeanName**） |
| **spring.biz.chain.biz-chains.&lt;chainName&gt;.<units>** | List<String> | 否 |  | 业务单元类的全名 |


## 4. 执行业务链
在完成以上的定义和配置后，我们可以这样开始执行一条业务链。
```java
@Resource
private BizChain bizChainC1;

@Test
public void bizChain1(){
    log.info("chain desc: {}", new Object[]{bizChainC1.getChainDescription()});
    BizContext bizContext = new BizContextBase();
    bizContext.put("a", "ffff");
    bizContext.put("b", "a");
    try {
        bizChainC1.executeChain(bizContext);
    } catch (Exception e) {
        Assert.fail(e.getMessage());
    }
}
```

- 使用`@Resource`通过Bean Name来引用业务链。

- 使用`com.github.semak.biz.chain.core.context.BizContextBase`类来初始化业务链的上下文，并初始化参数。

- 通过调用业务链的`executeChain`方法，并传入上下文参数开始启动整个业务链。

- 通过`BizChain`的`getChainDescription`方法，我们可以看到整条业务链中调用的单元及顺序，如: `BizUnit1 -> BizUnit3`

