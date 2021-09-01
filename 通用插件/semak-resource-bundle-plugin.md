`semak-resource-bundle-plugin`是一款提供了资源绑定功能的Maven插件，其特性主要包括：


1. 基于资源文件生成资源类。
2. 支持多个资源文件。



## 1. 依赖


### 1.1. 环境配置


1. Open JDK 1.8+，并已配置有效的环境变量。
2. Maven 3.3.x+，并已配置有效的环境变量。



### 1.2. Maven依赖配置


```xml
<plugin>
    <groupId>com.github.semak.plugin</groupId>
    <artifactId>semak-resource-bundle-maven-plugin</artifactId>
    <version>最新RELEASE版本</version> 
</plugin>
```


## 2. 插件配置


在需要做资源处理的模块POM中，配置如下插件：


> 占位符根据项目模块路径进行替换



```xml
<properties>
    <bundle.sourceBundle.one>bundle/message</bundle.sourceBundle.one>
    <bundle.targetClass>com.example.demo.support.bundle.MessageCode</bundle.targetClass>
</properties>

<build>
    <plugins>
        <plugin>
            <groupId>com.github.semak.plugin</groupId>
            <artifactId>semak-resource-bundle-maven-plugin</artifactId>
            <version>最新RELEASE版本</version>
            <inherited>false</inherited>
            <configuration>
                <sourceBundles>
                    <sourceBundle>${bundle.sourceBundle.one}</sourceBundle>
                </sourceBundles>
                <targetClass>${bundle.targetClass}</targetClass>
            </configuration>
            <executions>
                <execution>
                    <id>bundles</id>
                    <goals>
                        <goal>bundle-process</goal>
                    </goals>
                    <phase>compile</phase>
                    <configuration>
                        <sourceBundles>
                            <sourceBundle>${bundle.sourceBundle.one}</sourceBundle>
                        </sourceBundles>
                        <targetClass>${bundle.targetClass}</targetClass>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```


注意：请设置`inherited`属性为`false`。


## 3. 插件配置描述


插件包含的执行目标（Goal）：

| 目标名称 | 执行目标 | 描述 |
| :--- | :--- | :--- |
| **bundle-process** | **semak-resource-bundle:bundle-process** | 将资源文件转换为资源类 |



目标**semak-resource-bundle:bundle-process**包含的参数：

| 参数名称 | 描述 |
| :--- | :--- |
| **sourceBundles** | 资源文件列表，由子节点**sourceBundle**构成 |
| **sourceBundles.sourceBundle** | 资源文件的classpath路径，如：`bundle/message`代表了`classpath:bundle/message.properties`资源文件 |
| **targetClass** | 生成目标类的全名，包名如果不存在则会自动创建 |



## 4. 资源文件配置


> **classpath:bundle/message.properties**



```yaml
999999=系统处理异常！：{0}
-999998=数据库处理异常！：{0}
-12321=测试业务异常：{0}
```


## 5. 运行插件


### 5.1. 命令行方式


进入插件所在模块，执行下面命令：


```bash
mvn semak-resource-bundle:bundle-process
```


控制台输出：


```
$ mvn semak-resource-bundle:bundle-process
[INFO] Scanning for projects...
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] Building demo 0.0.1-SNAPSHOT
[INFO] ------------------------------------------------------------------------
[INFO] 
[INFO] --- semak-resource-bundle-maven-plugin:1.0.0-SNAPSHOT:bundle-process (default-cli) @ demo ---
[INFO] ######################### Generating Message Code [Author: caobin]#########################
[INFO] ****** 1. Initializing parameters of template ******
[INFO] >>> sourceBundles: [bundle/message]
[INFO] >>> targetClass: com.example.demo.support.bundle.MessageCode
[INFO] ****** 2. Generating message code file ******
[info] >>> Source bundles: [bundle/message]
[info] >>> Reading resource bundle: 
[info] =======================================================
[info] -12321=测试业务异常：{0}
[info] 999999=系统处理异常！：{0}
[info] -999998=数据库处理异常！：{0}
[info] =======================================================
[info] >>> Outputting message code file...
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 0.619 s
[INFO] Finished at: 2019-08-05T13:51:41+08:00
[INFO] Final Memory: 18M/491M
[INFO] ------------------------------------------------------------------------
```


### 5.2. 在指定的Maven生命周期中运行


在插件中配置`executions`，指定其中`execution`的目标 → goal和执行阶段 → `phase`，这里建议`phase`为`compile`，然后在项目的编译、打包、发布等阶段均可自动执行到插件。


```xml
....
<executions>
     <execution>
         <id>bundles</id>
         <goals>
             <goal>bundle-process</goal>
         </goals>
         <phase>compile</phase>
         <configuration>
             <sourceBundles>
                 <sourceBundle>${bundle.sourceBundle.one}</sourceBundle>
             </sourceBundles>
             <targetClass>${bundle.targetClass}</targetClass>
         </configuration>
     </execution>
 </executions>
```


### 5.3. 输出资源类


在执行完插件后，会生成如下的插件类：


```java
package com.example.demo.support.bundle;

/**
 * Message Source Codes
 * <p>!!此类自动生成，请勿手动修改!!</p>
 *
 * @author AutoGen
 */
public class MessageCode {

	/**
	 * [-12321] - 测试业务异常：{0}
	 */
	public static final Long C_12321 = -12321L;

	/**
	 * [-999998] - 数据库处理异常！：{0}
	 */
	public static final Long C_999998 = -999998L;

	/**
	 * [999999] - 系统处理异常！：{0}
	 */
	public static final Long C999999 = 999999L;

}
```
