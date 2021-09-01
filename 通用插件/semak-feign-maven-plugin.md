一般来说，作为服务提供方，特别是作为服务治理生态中的服务提供方，除了本身开放的服务之外，还需要提供给**调用方**有效的服务调用方式。在基于Spring Cloud的生态中，我们会使用`Feign`作为服务调用时的客户端，它会对接口进行代理并调用**服务方**。


原则上，类似这种**伪RPC**（把REST风格的调用伪装成RPC模式的调用）服务，我们这里的**服务方**仅需要提供**Facade**接口和**DTO**即可。按照`Feign`的代理规则，如果**Facade**接口没有附加足够的Rest注解，会无法进行代理。但是手动去添加注解，必然带来很多冗余的工作量。


所以，为方便大家提供给**调用方**经过注解渲染的`Facade`、`DTO`，并提供开箱即用的配置，这里提供了`semak-feign-maven-plugin`插件来一键生成。


插件提供的特性主要包括：


1. 对生成的Facade接口进行注解渲染，支持SpringRest提供的全部注解，支持Jackson提供的部分常用注解。
2. 对生成的客户端调用组件进行自动配置，基于SpringBoot可以做到开箱即用，且无需任何属性配置。
3. 通过Doclet对生成的源码自动进行Javadoc注释渲染。



## 1. 依赖


### 1.1. 环境配置


1. Open JDK 1.8+，并已配置有效的环境变量。
2. Maven 3.3.x+，并已配置有效的环境变量。



### 1.2. Maven依赖配置


**注意：** plugin必须配置在项目的`根POM`中。


```xml
<plugin>
    <groupId>com.github.semak.plugin</groupId>
    <artifactId>semak-feign-maven-plugin</artifactId>
    <version>最新RELEASE版本</version>
</plugin>
```


## 2. 插件配置


```xml
<plugin>
    <groupId>com.github.semak.plugin</groupId>
    <artifactId>semak-feign-maven-plugin</artifactId>
    <version>最新RELEASE版本</version>
    <inherited>false</inherited>
    <configuration>
        <!-- 和spring.application.name对应 -->
        <applicationName>test.nacos.server</applicationName>
        <modules>
            <!-- facade module -->
            <module>
                <name>semak-rest-test-cloud-nacos</name>
                <type>FACADE</type>
                <facadePackage>com.github.semak.rest.test.nacos.server.facade</facadePackage>
            </module>
            <!-- dto module -->
            <module>
                <name>semak-rest-test-cloud-nacos</name>
                <type>DTO</type>
                <dtoPackage>com.github.semak.rest.test.nacos.server.model.dto</dtoPackage>
            </module>
            <!-- destination module -->
            <module>
                <name>semak-rest-test-cloud-nacos-facade</name>
                <type>DESTINATION</type>
            </module>
        </modules>
    </configuration>
</plugin>
```


为了构建客户端自动代理接口，需要从标识为`FACADE`的模块提取接口类及实现类（注解渲染相关），从标识为`DTO`的模块提取数据传输对象类，经处理配置后，将源代码生成到标识为`DESTINATION`的目标模块，最终编译打包，提供给客户端使用（自动配置）。


## 3. 插件配置描述


插件包含的执行目标（Goal）：

| **目标名称** | **执行目标** | **描述** |
| :--- | :--- | :--- |
| **generate** | **semak-feign:generate** | 提取指定模块的Facade、DTO，处理后输出到目标模块 |



**`注意`**:


- **插件执行前，整个项目需要进行一次完整的安装`mvn install`**。
- **配置插件时，将`inherited`节点设置为`false`**。



目标**semak-feign:generate**包含的参数：

| **参数名称** | **描述** |
| :--- | :--- |
| **applicationName** | 同`spring.application.name`的值，建议由`产线名称.应用名称`构成 |
| **modules.module.name** | 项目模块名称，即Maven Module名称 |
| **modules.module.type** | 主要由三个类型构成：
 **FACADE**: 标识为服务接口（Facade）的来源模块 
 **DTO**: 标识为数据传输对象（DTO）的来源模块 
 **DESTINATION**: 标识为生成客户端代理接口（含自动配置）的模块 |
| **modules.module.facadePackage** | 当`modules.module.type`为`FACADE`时，使用此属性。标识了模块中FACADE接口的来源包名。 |
| **modules.module.dtoPackage** | 当`modules.module.type`为`DTO`时，使用此属性。标识了模块中DTO的来源包名。 |



## 4. 运行插件


### 4.1. 命令行方式


执行插件前，如果没有对工程进行过安装操作，需要对工程先做一次完整的安装（`mvn install`）。


切换到项目**根目录**，执行命令: `mvn semak-feign:generate`，可以看到生成的日志如下:


```bash
$ mvn semak-feign:generate
[INFO] Scanning for projects...
[INFO] ------------------------------------------------------------------------
[INFO] Reactor Build Order:
[INFO] 
[INFO] semak-rest
[INFO] semak-rest-core
[INFO] semak-rest-spring-boot
[INFO] semak-rest-spring-boot-starter
[INFO] semak-rest-spring-cloud
[INFO] semak-rest-spring-cloud-starter
[INFO] semak-rest-spring-cloud-nacos
[INFO] semak-rest-spring-cloud-nacos-starter
[INFO] semak-rest-test
[INFO] semak-rest-test-boot
[INFO] semak-rest-test-cloud
[INFO] semak-rest-test-cloud-nacos
[INFO] semak-rest-test-cloud-client
[INFO] semak-rest-test-cloud-nacos-facade
[INFO] semak-rest-test-cloud-nacos-client
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] Building semak-rest 1.0.0-SNAPSHOT
[INFO] ------------------------------------------------------------------------
[INFO] 
[INFO] --- semak-feign-maven-plugin:1.0.0-SNAPSHOT:generate (default-cli) @ semak-rest ---
[INFO] ######################### Generating Feign Facade [Author: caobin]#########################
[INFO] ****** 1. Loading Modules ******
[INFO] >>> Loaded module: semak-rest-test-cloud-nacos with type [FACADE]
[INFO] >>> Loaded module: semak-rest-test-cloud-nacos with type [DTO]
[INFO] >>> Loaded module: semak-rest-test-cloud-nacos-facade with type [DESTINATION]
[INFO] ****** 2. Cleaning sources and resources of destination project ******
[INFO] >>> Sources was cleaned
[INFO] >>> Resources was cleaned
[INFO] ****** 3. Processing & Copying dto(s) to destination ******
[info] >>> Located directory of dto classes: /Users/brevy/Project/Github/semak/semak-rest/semak-rest-test/semak-rest-test-cloud-nacos/target/classes/com/github/semak/rest/test/nacos/server/model/dto
[info] >>> Found dto class: com.github.semak.rest.test.nacos.server.model.dto.UserResponse
[info] >>> Found dto class: com.github.semak.rest.test.nacos.server.model.dto.UserRequest
[info] >>> Found dto class: com.github.semak.rest.test.nacos.server.model.dto.Contact
正在加载源文件/Users/brevy/Project/Github/semak/semak-rest/semak-rest-test/semak-rest-test-cloud-nacos/src/main/java/com/github/semak/rest/test/nacos/server/model/dto/UserResponse.java...
正在构造 Javadoc 信息...
正在加载源文件/Users/brevy/Project/Github/semak/semak-rest/semak-rest-test/semak-rest-test-cloud-nacos/src/main/java/com/github/semak/rest/test/nacos/server/model/dto/UserResponse.java...
正在构造 Javadoc 信息...
正在加载源文件/Users/brevy/Project/Github/semak/semak-rest/semak-rest-test/semak-rest-test-cloud-nacos/src/main/java/com/github/semak/rest/test/nacos/server/model/dto/UserResponse.java...
正在构造 Javadoc 信息...
正在加载源文件/Users/brevy/Project/Github/semak/semak-rest/semak-rest-test/semak-rest-test-cloud-nacos/src/main/java/com/github/semak/rest/test/nacos/server/model/dto/UserRequest.java...
正在构造 Javadoc 信息...
正在加载源文件/Users/brevy/Project/Github/semak/semak-rest/semak-rest-test/semak-rest-test-cloud-nacos/src/main/java/com/github/semak/rest/test/nacos/server/model/dto/UserRequest.java...
正在构造 Javadoc 信息...
正在加载源文件/Users/brevy/Project/Github/semak/semak-rest/semak-rest-test/semak-rest-test-cloud-nacos/src/main/java/com/github/semak/rest/test/nacos/server/model/dto/UserRequest.java...
正在构造 Javadoc 信息...
正在加载源文件/Users/brevy/Project/Github/semak/semak-rest/semak-rest-test/semak-rest-test-cloud-nacos/src/main/java/com/github/semak/rest/test/nacos/server/model/dto/Contact.java...
正在构造 Javadoc 信息...
正在加载源文件/Users/brevy/Project/Github/semak/semak-rest/semak-rest-test/semak-rest-test-cloud-nacos/src/main/java/com/github/semak/rest/test/nacos/server/model/dto/Contact.java...
正在构造 Javadoc 信息...
正在加载源文件/Users/brevy/Project/Github/semak/semak-rest/semak-rest-test/semak-rest-test-cloud-nacos/src/main/java/com/github/semak/rest/test/nacos/server/model/dto/Contact.java...
正在构造 Javadoc 信息...
[info] >>> Outputting sources...
[info] >>> Generating Dto: /Users/brevy/Project/Github/semak/semak-rest/semak-rest-test/semak-rest-test-cloud-nacos-facade/src/main/java/com/github/semak/rest/test/nacos/server/model/dto/Contact.java
[info] >>> Generating Dto: /Users/brevy/Project/Github/semak/semak-rest/semak-rest-test/semak-rest-test-cloud-nacos-facade/src/main/java/com/github/semak/rest/test/nacos/server/model/dto/UserRequest.java
[info] >>> Generating Dto: /Users/brevy/Project/Github/semak/semak-rest/semak-rest-test/semak-rest-test-cloud-nacos-facade/src/main/java/com/github/semak/rest/test/nacos/server/model/dto/UserResponse.java
[INFO] ****** 4. Processing & Copying facade(s) to destination ******
[info] >>> Located directory of facade interfaces: /Users/brevy/Project/Github/semak/semak-rest/semak-rest-test/semak-rest-test-cloud-nacos/target/classes/com/github/semak/rest/test/nacos/server/facade
[info] >>> Found facade interface: com.github.semak.rest.test.nacos.server.facade.DemoFacade
[info] >>> Located directory of facade providers: /Users/brevy/Project/Github/semak/semak-rest/semak-rest-test/semak-rest-test-cloud-nacos/target/classes/com/github/semak/rest/test/nacos/server/facade
[info] >>> Found facade provider: com.github.semak.rest.test.nacos.server.facade.provider.DefaultDemoFacade
[info] >>> Facade Mapping: DemoFacade -> DefaultDemoFacade
正在加载源文件/Users/brevy/Project/Github/semak/semak-rest/semak-rest-test/semak-rest-test-cloud-nacos/src/main/java/com/github/semak/rest/test/nacos/server/facade/DemoFacade.java...
正在构造 Javadoc 信息...
正在加载源文件/Users/brevy/Project/Github/semak/semak-rest/semak-rest-test/semak-rest-test-cloud-nacos/src/main/java/com/github/semak/rest/test/nacos/server/facade/DemoFacade.java...
正在构造 Javadoc 信息...
[info] >>> Outputting sources...
[info] >>> Generating Facade: /Users/brevy/Project/Github/semak/semak-rest/semak-rest-test/semak-rest-test-cloud-nacos-facade/src/main/java/com/github/semak/rest/test/nacos/server/facade/DemoFacade.java
[INFO] ****** 5. Generating & Copying Configurations to destination ******
[info] >>> Located directory of java configuration: /Users/brevy/Project/Github/semak/semak-rest/semak-rest-test/semak-rest-test-cloud-nacos-facade/src/main/java/com/github/semak/rest/test/nacos/server/facade
[info] >>> Located directory of spring configuration: /Users/brevy/Project/Github/semak/semak-rest/semak-rest-test/semak-rest-test-cloud-nacos-facade/src/main/resources/META-INF
[info] >>> Outputting sources...
[info] >>> Generating Configuration: /Users/brevy/Project/Github/semak/semak-rest/semak-rest-test/semak-rest-test-cloud-nacos-facade/src/main/java/com/github/semak/rest/test/nacos/server/facade/FeignScannerConfiguration.java
[info] >>> Generating Configuration: /Users/brevy/Project/Github/semak/semak-rest/semak-rest-test/semak-rest-test-cloud-nacos-facade/src/main/resources/META-INF/spring.factories
[INFO] ------------------------------------------------------------------------
[INFO] Reactor Summary:
[INFO] 
[INFO] semak-rest ......................................... SUCCESS [  2.102 s]
[INFO] semak-rest-core .................................... SKIPPED
[INFO] semak-rest-spring-boot ............................. SKIPPED
[INFO] semak-rest-spring-boot-starter ..................... SKIPPED
[INFO] semak-rest-spring-cloud ............................ SKIPPED
[INFO] semak-rest-spring-cloud-starter .................... SKIPPED
[INFO] semak-rest-spring-cloud-nacos ...................... SKIPPED
[INFO] semak-rest-spring-cloud-nacos-starter .............. SKIPPED
[INFO] semak-rest-test .................................... SKIPPED
[INFO] semak-rest-test-boot ............................... SKIPPED
[INFO] semak-rest-test-cloud .............................. SKIPPED
[INFO] semak-rest-test-cloud-nacos ........................ SKIPPED
[INFO] semak-rest-test-cloud-client ....................... SKIPPED
[INFO] semak-rest-test-cloud-nacos-facade ................. SKIPPED
[INFO] semak-rest-test-cloud-nacos-client ................. SKIPPED
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 2.712 s
[INFO] Finished at: 2019-09-12T17:55:45+08:00
[INFO] Final Memory: 138M/837M
```


### 4.2. 生成的目标模块结构


自动生成后的**Facade**模块结构如下：


![](https://cdn.nlark.com/yuque/0/2020/jpeg/1241873/1590744470967-e64a601a-3786-44c0-9b1a-183beddd2512.jpeg#align=left&display=inline&height=340&margin=%5Bobject%20Object%5D&originHeight=340&originWidth=372&status=done&style=none&width=372)


### 4.3. 生成的Facade接口一览


```java
/**
 * HelloFacade
 *
 * @author caobin
 * @version 1.0 2019.08.16
 */
@FeignClient("test.nacos.server")
@RequestMapping(value="/demo", path="/demo", produces="application/json")
public interface DemoFacade {


	/**
	 * 添加用户
	 *
	 * @param arg0
	 * @return
	 */
	@PostMapping(value="/addUser", path="/addUser") 
	UserResponse addUser(UserRequest arg0);

	/**
	 * 分页获取用户列表
	 *
	 * @param arg0
	 * @param arg1
	 * @return
	 */
	@GetMapping(value="/getUsers", path="/getUsers") 
	SimplePageInfo<UserResponse> getUserList(@RequestParam(name="pageNum", value="pageNum") int arg0, @RequestParam(name="pageSize", value="pageSize") int arg1);

	/**
	 * 修改用户
	 *
	 * @param arg0
	 * @return
	 */
	@PutMapping(value="/modifyUser", path="/modifyUser") 
	UserResponse modifyUser(UserRequest arg0);

	/**
	 * 批量添加用户
	 *
	 * @param arg0
	 * @return
	 */
	@PostMapping(value="/addUsers", path="/addUsers") 
	List<UserResponse> addUsers(List<UserRequest> arg0);

	/**
	 * 获取当前日期
	 *
	 * @return
	 */
	@GetMapping(value="/getCurrentDate", path="/getCurrentDate") 
	Date getCurrentDate();

	/**
	 * 删除用户
	 *
	 * @param arg0
	 * @return
	 */
	@DeleteMapping(value="/deleteUser", path="/deleteUser") 
	boolean deleteUser(String arg0);

	/**
	 * Hello
	 *
	 * @param arg0
	 * @return
	 */
	@GetMapping(value="/hello/{name}", path="/hello/{name}") 
	String hello(@PathVariable(name="name", value="name") String arg0);

	/**
	 * 执行异常
	 *
	 * @param arg0
	 * @return
	 */
	@RequestMapping(value="/doException/{ex}", path="/doException/{ex}", method=RequestMethod.OPTIONS) 
	UserResponse doException(@PathVariable(name="ex", value="ex") String arg0);

	/**
	 * Null返回
	 *
	 * @return
	 */
	@RequestMapping(value="/getNull", path="/getNull", method=RequestMethod.GET) 
	UserResponse getNull();
}
```


我们可以看到插件对接口进行了注解渲染，以满足Feign客户端接口代理的需要。。


### 4.4. 生成的DTO接口一览


```java
/**
 * User Request
 *
 * @author caobin
 * @version 1.0 2017.10.07
 */
public class UserRequest extends BaseRequest {


	/**
	 * 用户名
	 */
	private String username;

	/**
	 * 年龄
	 */
	private Integer age;

	//此处略...


	/**
	 * Sets new 年龄.
	 *
	 * @param age New value of 年龄.
	 */
	public void setAge(Integer age){
		this.age = age;
	}

	/**
	 * Gets 年龄.
	 *
	 * @return Value of 年龄.
	 */
	public Integer getAge(){
		return age;
	}

	/**
	 * Sets new 用户名.
	 *
	 * @param username New value of 用户名.
	 */
	public void setUsername(String username){
		this.username = username;
	}

	/**
	 * Gets 用户名.
	 *
	 * @return Value of 用户名.
	 */
	public String getUsername(){
		return username;
	}
	
	//此处略...
}
```


## 5. 插件发布


最后，我们将自动生成的Facade模块进行打包发布后，二方/三方通过Maven配置完Facade坐标后，即可以基于此Facade进行Feign的接口代理调用。
