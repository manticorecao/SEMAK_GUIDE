# semak-parent

`semak-parent`组件是`semak`相关项目的顶级POM组件，用于统一管理类库版本。根据POM的功能类型可分为必要依赖和可选依赖。



## 1. 先决条件


### 1.1. 环境配置


1. Open JDK 1.8+，并已配置有效的环境变量。
2. Maven 3.3.x+，并已配置有效的环境变量。



## 2. 必要依赖


作为必要依赖的POM，位于项目的根目录，为semak组件及框架必须继承的POM，包含一些常用且较为通用的类库和插件。


继承方式为下例中`parent`节点部分：


```xml
<parent>
    <artifactId>semak-parent</artifactId>
    <groupId>com.github.semak.parent</groupId>
    <version>最新RELEASE版本</version>
</parent>

<groupId>com.github.semak.xxx</groupId>
<artifactId>semak-xxx</artifactId>
<version>1.0.0-SNAPSHOT</version>
<packaging>pom</packaging>
```



## 3. 可选依赖


作为可选依赖的POM，以Maven的模块形式存在，为semak组件及框架提供功能性的POM，包含一些功能性的类库和插件。


继承方式使用Maven的import方式，并在`dependencyManagement`节点下进行管理，具体如下：

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.github.semak.parent</groupId>
            <artifactId>semak-base-dependencies</artifactId>
            <version>最新RELEASE版本</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>com.github.semak.parent</groupId>
            <artifactId>semak-data-dependencies</artifactId>
            <version>最新RELEASE版本</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```


当前定义的依赖模块分别为：


1. **semak-base-dependencies**: 基础类库依赖，必选依赖。

2. **semak-cloud-dependencies**: 云类库依赖，包含云原生类库的依赖，如：spring-cloud、spring-cloud-alibaba等。

3. **semak-data-dependencies**: 数据相关类库依赖，包含数据操作相关类库的依赖，如：hikaricp、mybatis、redis等。

4. **semak-engine-dependencies**: 引擎类库依赖，包含引擎类类库的依赖，如：调度引擎、工作流引擎、规则引擎等。

5. **semak-plugin-security**: 安全类依赖库，包含应用安全、数据安全相关的类库依赖，如：shiro、jwt等。

6. **semak-plugin-dependencies**: Maven插件类库依赖，包含插件开发所需类库。

   