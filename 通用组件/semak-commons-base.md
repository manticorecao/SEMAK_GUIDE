# semak-commons-base

`semak-commons-base`组件主要提供通用基础功能，主要的功能和工具包括：


1. 无侵入Builder构建器。
1. 基础加解密支持。
1. 字节码增强工具。
1. 日志工具（目前仅支持logback）。
1. SPI加载工具。
1. CSV工具类。
1. 网络工具类（如：获取本机IP，非环回地址）。



## 1. 先决条件


### 1.1. 环境配置


1. Open JDK 1.8+，并已配置有效的环境变量。
1. Maven 3.3.x+，并已配置有效的环境变量。



### 1.2. Maven依赖配置


```xml
<dependency>
    <groupId>com.github.semak.commons</groupId>
    <artifactId>semak-commons-base</artifactId>
    <version>最新RELEASE版本</version>
</dependency>
```



## 2. 功能或工具的使用


### 2.1. 无侵入Builder构建器


一种类似于Builder模式的Model设置方式，将常规的setter方式变的更为连贯，而无需对代码进行任何侵入。


如果，原来我们队Model的设置方式是这样的：
```java
DemoUser demoUser = new DemoUser();
demoUser.setAge(33);
demoUser.setUsername("George");
demoUser.setSex("M");
```


那么，使用Builder构建器的方式来做是这样的：
```java
DemoUser demoUser = GenericBuilder
         .of(DemoUser::new)
         .with(DemoUser::setAge, 33)
         .with(DemoUser::setUsername, "Geroge")
         .with(DemoUser::setSex, "M")
         .build();
```



### 2.2. 字节码增强工具


一种基于Javassist进行简单封装的工具类，将常规的前置操作统一屏蔽，对外提供更好用的接口。


如果，我们需要增强的类如下：
```java
package com.github.semak.commons.base.enhancer;

/**
 * demo
 *
 * @author caobin
 * @version 1.0
 * @date 2020.06.01
 */
public class Demo {

    public void sayHello(){
        System.out.println(">> Hello, George");
    }
}
```


现在需要做一个sayHello方法的前置处理:
```java
try{
    //增强方法
    ClassEnhancer.enhance("com.github.semak.commons.base.enhancer.Demo", this.getClass(), ctClass -> {
        //基于Javassist的字节码实现
        CtMethod sayHello = ctClass.getDeclaredMethod("sayHello");
        sayHello.insertAt(1, "{System.out.println(\">>> 前置操作打印。。。\");}");
        log.info("sayHello方法已增强");
    });
    //初始化对象并调用方法
    Demo demo = new Demo();
    demo.sayHello();
} catch (Exception e){
    log.error(e.getMessage(), e);
}
```


输出结果如下：


```
15:04:45.264 [main] INFO com.github.semak.commons.base.enhancer.DemoTestCase - sayHello方法已增强
>>> 前置操作打印。。。
>> Hello, George
```



### 2.3. 日志工具


#### 2.3.1. 本机IP转换器


将本机IP（非环回地址）作为日志模式的有效占位符。


具体配置如下：
```xml
<configuration>
    <!-- ...... -->

    <conversionRule conversionWord="LIP" converterClass="com.github.semak.commons.base.logging.LogIpConverter" />

    <property name="CONSOLE_LOG_PATTERN" value="%d{yyyy-MM-dd HH:mm:ss.SSS} %clr(%-5level) [%magenta(%LIP),%blue(%X{X-B3-TraceId:&#45;&#45;})] %clr(&#45;&#45;&#45;) [%clr(%t){faint}] %cyan(%logger{80}) :%msg%n" />

    <!-- ...... -->
</configuration>
```


输出的日志样式：
```
2020-06-01 15:11:49.947 INFO  [10.134.56.201,-] --- [main] com.zaxxer.hikari.HikariDataSource :HikariPool-1 - Starting...
```



### 2.4. SPI加载工具


基于Java SPI提供的工具类，提供SPI接口的注册，并可以基于接口获取单个或多个实现对象。


1. 定义接口`com.github.semak.commons.base.spi.DemoSpi`。

   ```java
   public interface DemoSpi {
   
       String hello(String text);
   }
   ```

2. 定义接口`com.github.semak.commons.base.spi.DemoSpi`的实现类`com.github.semak.commons.base.spi.DefaultDemoSpi`。

   ```java
   public class DefaultDemoSpi implements DemoSpi {
   
       @Override
       public String hello(String text) {
           return "Hello, " + text;
       }
   }
   ```

3. 在`classpath:/META-INF/services/`下定义`com.github.semak.commons.base.spi.DemoSpi`文件，并列出实现类全名。

   ```
   com.github.semak.commons.base.spi.DefaultDemoSpi
   ```

4. 通过SPI加载工具类注册接口，并获取实现。

   ```java
   TypeBasedServiceLoader.register(DemoSpi.class);
   //获取所有罗列的实现
   Collection<DemoSpi> collection = TypeBasedServiceLoader.newServiceInstances(DemoSpi.class);
   log.info(">>> Collection: {}", collection);
   //获取排在第一位的实现
   DemoSpi demoSpi = TypeBasedServiceLoader.newServiceInstance(DemoSpi.class);
   log.info(">>> Result: {}", demoSpi.hello("George"));
   ```

   输出如下：

   ```
   15:22:53.287 [main] INFO com.github.semak.commons.base.spi.SpiTestCase - >>> Collection: [com.github.semak.commons.base.spi.DefaultDemoSpi@5fe5c6f]
   15:22:53.293 [main] INFO com.github.semak.commons.base.spi.SpiTestCase - >>> Result: Hello, George
   ```

   


### 2.5. CSV工具类


提供CSV文件的基本读写功能，由于是一次性加载到内存进行操作，所以需要对实际数据量进行评估。


#### 2.5.1. 写入CSV


```java
private static final String CSV_FILE = System.getProperty("user.home").concat(File.separator).concat("TMP_FILE.csv");


public void writeCsv(){
    List<String[]> content = new ArrayList<>(2);
    content.add(new String[]{"1", "George", "26.45"});
    content.add(new String[]{"2", "Mary", "56.70"});
    try {
        CsvUtils.writeCsv(new File(CSV_FILE), content, "utf-8");
    } catch (IOException e) {
        log.error(e.getMessage(), e);
    }
}
```



#### 2.5.2. 读取CSV


```java
public void readCsv(){
    List<String[]> content = CsvUtils.readCsv(new File(CSV_FILE), false, "utf-8");
    content.stream().forEach(line -> log.info(">> {}", Arrays.toString(line)));
}
```


输出结果：


```
15:37:30.485 [main] INFO com.github.semak.commons.base.csv.CsvTestCase - >> [1, George, 26.45]
15:37:30.490 [main] INFO com.github.semak.commons.base.csv.CsvTestCase - >> [2, Mary, 56.70]
```



### 2.6. 网络工具类


提供基于Java扩展的网络工具。


#### 2.6.1. 获取本机非环回IP地址


```java
public void getLoacalIp(){
    //需要忽略的网卡名称，用正则表达式表示
    List<String> ignores = new ArrayList(1){{
       add("docker.*");
    }};
    InetAddress inetAddress = NetUtils.findFirstNonLoopbackAddress(ignores, true);
    log.info(">> Local ip: {}", inetAddress.getHostAddress());
}
```


输出结果：


```
17:31:26.233 [main] INFO com.github.semak.commons.base.utility.NetUtils - Testing interface: utun1
17:31:26.235 [main] INFO com.github.semak.commons.base.utility.NetUtils - Testing interface: utun0
17:31:26.235 [main] INFO com.github.semak.commons.base.utility.NetUtils - Testing interface: llw0
17:31:26.235 [main] INFO com.github.semak.commons.base.utility.NetUtils - Testing interface: awdl0
17:31:26.236 [main] INFO com.github.semak.commons.base.utility.NetUtils - Testing interface: en5
17:31:26.236 [main] INFO com.github.semak.commons.base.utility.NetUtils - Testing interface: en0
17:31:26.236 [main] INFO com.github.semak.commons.base.utility.NetUtils - Found non-loopback interface: en0
17:31:26.236 [main] INFO com.github.semak.commons.base.utility.NetUtils - Testing interface: lo0
17:31:26.236 [main] INFO com.github.semak.commons.base.net.NetTestCase - >> Local ip: 10.134.56.201
```



### 2.7. 文件魔数检测工具


通过读取文件的十六进制的前几位来判断文件真实类型。
```java
URL url = this.getClass().getClassLoader().getResource("files");
File classpathDir = new File(url.getPath());
File[] files = classpathDir.listFiles();
for(File file : files){
	MimeType mimeType = FileMagicNumberDetector.detectFileType(file);
	log.info(String.format("File: [%s], MimeType: [%s].", file.getName(), mimeType == null ? "unknown" : mimeType.getDesc()));
}
```
输出结果：
```
File: [演示文稿1.pptx], MimeType: [application/vnd.openxmlformats-officedocument.presentationml.presentation].
File: [错误编码规则.doc], MimeType: [application/msword].
File: [1102113100000028.xls], MimeType: [application/vnd.ms-excel].
File: [799报文.docx], MimeType: [application/vnd.openxmlformats-officedocument.wordprocessingml.document].
File: [支付宝架构与技术-阿里金融交流.ppt], MimeType: [application/vnd.ms-powerpoint].
File: [ETL脚本变量配置表.xlsx], MimeType: [application/vnd.openxmlformats-officedocument.spreadsheetml.sheet].
File: [10.zip], MimeType: [application/zip].
File: [analyzerspecification.pdf], MimeType: [application/pdf].
File: [版本管理流程.vsd], MimeType: [application/vnd.visio].
File: [基于消息系统的分布式事务.vsdx], MimeType: [application/vnd.ms-visio.drawing].
File: [牺牲阳极展柜.jpg], MimeType: [image/jpeg].
File: [warn.png], MimeType: [image/png].
File: [bankofshanghai.rar], MimeType: [application/x-rar-compressed].
```
也可以通过工具中的 `valicateFileType` 的方法指定MimeType（枚举类）来对比文件。
