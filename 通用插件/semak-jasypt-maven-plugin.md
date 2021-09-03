# semak-jasypt-maven-plugin

`semak-jasypt-maven-plugin`插件是一款基于jasypt的加解密Maven插件，其特性主要包括：


1. 基于命令行的加密功能。
2. 基于命令行的解密功能。
3. 基于命令行的可选的盐（salt）参数。



## 1. 依赖


### 1.1. 环境配置


1. Open JDK 1.8+，并已配置有效的环境变量。
2. Maven 3.3.x+，并已配置有效的环境变量。



### 1.2. Maven依赖配置


```xml
<plugin>
    <groupId>com.github.semak.plugin</groupId>
    <artifactId>semak-jasypt-maven-plugin</artifactId>
    <version>最新RELEASE版本</version> 
</plugin>
```



## 2. 插件配置


在项目的根POM中，配置如下插件：


```xml
<build>
    <plugins>
        <plugin>
            <groupId>com.github.semak.plugin</groupId>
            <artifactId>semak-jasypt-maven-plugin</artifactId>
            <version>最新RELEASE版本</version>
        </plugin>
    </plugins>
</build>
```



## 3. 插件配置描述


插件包含的执行目标（Goal）：


| 目标名称 | 描述 |
| :--- | :--- |
| **semak-jasypt:encrypt** | 加密明文 |
| **semak-jasypt:decrypt** | 解密密文 |




## 4. 命令行的调用方式


### 4.1. 加密


```bash
#使用指定salt
mvn semak-jasypt:encrypt -Djasypt.salt=2M7VHgB8PsaXqHi -Djasypt.plain-password=psfp

#使用默认salt，与semak组件匹配
mvn semak-jasypt:encrypt -Djasypt.plain-password=psfp
```


控制台输出：


```
[INFO] --- semak-jasypt-maven-plugin:1.0.0-SNAPSHOT:encrypt (default-cli) @ semak-data-datasource ---
[INFO] ========注意：同一明文每次转换的结果不同 ========
[INFO] >>>> 明文密码 [psfp] 转换为密文密码 [LBThk+fyhd3llwiv1tIdVA==] <<<<
```



### 4.2. 加密参数描述


| 参数名称 | 必填 | 描述 |
| :--- | :--- | :--- |
| **jasypt.salt** | 否 | 加解密用到的盐值，默认使用组件内置提供的盐值 |
| **jasypt.plain-password** | 是 | 密码明文 |




### 4.3. 解密


```bash
#使用指定salt
mvn semak-jasypt:decrypt -Djasypt.salt=2M7VHgB8PsaXqHi -Djasypt.enc-password=409XzuLG2vmRexN08yIvKg==

#使用默认salt，与semak组件匹配
mvn semak-jasypt:decrypt -Djasypt.enc-password=409XzuLG2vmRexN08yIvKg==
```


控制台输出：


```
[INFO] --- semak-jasypt-maven-plugin:1.0.0-SNAPSHOT:decrypt (default-cli) @ semak-data-datasource ---
[INFO] >>>> 密文密码 [409XzuLG2vmRexN08yIvKg==] 转换为明文密码 [psfp] <<<<
```



### 4.4. 解密参数描述


| 参数名称 | 必填 | 描述 |
| :--- | :--- | :--- |
| **jasypt.salt** | 否 | 加解密用到的盐值，默认使用组件内置提供的盐值 |
| **jasypt.enc-password** | 是 | 密码密文 |

