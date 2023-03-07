# semak-oss

`semak-oss`组件是基于**MinIO** 高性能对象存储构建的客户端组件，其特性主要包括：

1. 完全兼容Amazon **S3**接口，切换云上存储服务无压力。
2. 客户端与存储服务器之间采用**http/https**通信协议。
3. 提供丰富的对象和桶操作接口。
4. 安全策略的应用。
5. 基于SpringBoot组件规范进行开发，通过依赖starter和一些简单的属性配置（properties或yaml）即可启用客户端。



## 1. 先决条件


### 1.1. 环境配置


1. Open JDK 1.8+，并已配置有效的环境变量。
2. Maven 3.3.x+，并已配置有效的环境变量。



### 1.2. Maven依赖配置

```xml
<dependency>
    <groupId>com.github.semak.oss</groupId>
    <artifactId>semak-oss-spring-boot-starter</artifactId>
    <version>最新RELEASE版本</version>
</dependency>
```



## 2. 属性定义

```yaml
spring:
  oss:
    enabled: true
    endpoint: http://api.minio.polarwin.cc
    access-key: user
    secret-key: t5jOfEPNpmx49HbXQUfovGR3EdzS0WV9
    simple-error-response: true
```



### 2.1. 属性描述

| **属性**                             | **数据类型** | **必填** | **默认值** | **描述**                   |
| :----------------------------------- | :----------- | :------- | :--------- | :------------------------- |
| **spring.oss.enabled**               | boolean      | 否       | true       | 是否启用oss的客户端        |
| **spring.oss.endpoint**              | String       | 是       |            | oss服务端点                |
| **spring.oss.access-key**            | String       | 是       |            | oss服务端申请的access key  |
| **spring.oss.secret-key**            | String       | 是       |            | oss服务端申请的secret key  |
| **spring.oss.simple-error-response** | boolean      | 否       | true       | 是否启用简洁的错误响应格式 |



## 3. 使用方式

> 后续示例代码中涉及的自定义方法如下：

```java
protected String defaultBucket = "demo.user-api";

/**
 * 获取资源文件路径
 *
 * @param simpleFilename
 * @return
 */
protected String getResourceAsPath(String simpleFilename) {
    return Thread.currentThread().getContextClassLoader().getResource("files/" + simpleFilename).getPath();
}

/**
 * 获取资源文件流
 *
 * @param simpleFilename
 * @return
 */
protected InputStream getResourceAsStream(String simpleFilename) {
    return Thread.currentThread().getContextClassLoader().getResourceAsStream("files/" + simpleFilename);
}

/**
 * 获取输出文件
 *
 * @param simpleFilename
 * @return
 */
protected File getOutputFile(String simpleFilename) {
    return new File(System.getProperty("user.dir") + "/src/test/resources/files/output/" + simpleFilename);
}
```



### 3.1. 对象操作

> 1. 对象操作的所有接口可以参考：`com.github.semak.oss.core.ops.ObjectOperations`。
> 2. 下面操作例举部分接口用法，并非全部。同类动作的接口，如putObject，uploadObject等，几乎都具备构建复杂参数的用法，不做一一举例。

#### 3.1.1. 通过流的方式上传对象

接口：

```java
/**
 * 通过流上传对象
 *
 * <p>输入流会在对象上传完毕后进行自动关闭</p>
 *
 * @param bucketName  指定的桶名称
 * @param objectName  定义的对象名称
 * @param inputStream 上传对象的输入流
 */
void putObject(String bucketName, String objectName, InputStream inputStream);
```



示例：

```java
@Autowired
private OssClient ossClient;

/**
 * 以流的方式上传对象
 */
@Test
public void putObject1() {
    ossClient.opsForObject().putObject(
            defaultBucket,
            "02.jpg",
            getResourceAsStream("02.jpg")
    );
}
```



#### 3.1.2. 通过流的方式上传对象并设置Content-Type

接口：

```java
/**
 * 通过流上传对象
 *
 * <p>输入流会在对象上传完毕后进行自动关闭</p>
 *
 * @param bucketName  指定的桶名称
 * @param objectName  定义的对象名称
 * @param inputStream 上传对象的输入流
 * @param contentType 对象的类型（同HTTP Header: content-type）
 */
void putObject(String bucketName, String objectName, InputStream inputStream, String contentType);
```



示例：

```java
@Autowired
private OssClient ossClient;

/**
 * 以流的方式上传对象并设置Content-Type
 */
@Test
public void putObject2() {
    ossClient.opsForObject().putObject(
            defaultBucket,
            "02_E.jpg",
            getResourceAsStream("02.jpg"),
            "image/jpeg"
    );
}
```

* 当设置**Content-Type**时，在浏览器支持它的情况下，可以直接展示在浏览器中而非直接下载。
* 在不设置**Content-Type**时，其默认值为：`application/octet-stream`。



#### 3.1.3. 通过流的方式上传对象(复杂参数)

接口：

```java
/**
 * 通过流上传对象
 *
 * <p>输入流会在对象上传完毕后进行自动关闭</p>
 *
 * @param putObjectArgs 可构造基于输入流的上传参数
 */
void putObject(PutObjectArgs putObjectArgs);
```



示例：

```java
@Autowired
private OssClient ossClient;   

/**
 * 以流的方式上传对象(复杂参数构造)
 */
@Test
public void putObject3() {
    InputStream inputStream = getResourceAsStream("02.jpg");
    try {
        ossClient.opsForObject().putObject(
                PutObjectArgs
                        .builder()
                        .bucket(defaultBucket)
                        .object("02_F.jpg")
                        .stream(inputStream, inputStream.available(),-1)
                        .build()
        );
    } catch (IOException e) {
        Assert.fail(e.getMessage());
    }
}
```



#### 3.1.4. 通过文件上传对象

接口：

```java
/**
 * 通过文件上传对象
 *
 * <p>对象名称同文件名称</p>
 *
 * @param bucketName 指定的桶名称
 * @param file       上传的文件
 */
void uploadObject(String bucketName, File file);
```



示例：

```java
@Autowired
private OssClient ossClient;

/**
 * 以文件方式上传对象
 */
@Test
public void uploadObject1() {
    ossClient.opsForObject().uploadObject(
            defaultBucket,
            new File(getResourceAsPath("01.jpg"))
    );
}
```



#### 3.1.5. 通过文件方式上传对象并设置Content-Type & 对象名称

接口：

```java
/**
 * 通过文件上传对象
 *
 * @param bucketName  指定的桶名称
 * @param objectName  定义的对象名称
 * @param file        上传的文件
 * @param contentType 对象的类型（同HTTP Header: content-type）
 */
void uploadObject(String bucketName, String objectName, File file, String contentType);
```



示例：

```java
@Autowired
private OssClient ossClient;

@Test
public void uploadObject2() {
    ossClient.opsForObject().uploadObject(
            defaultBucket,
            "01_E1.jpg",
            new File(getResourceAsPath("01.jpg")),
            "image/jpeg"
    );
}
```



#### 3.1.6. 通过字节数组的方式获取对象

接口：

```java
/**
 * 以字节数组的方式获取对象
 *
 * @param bucketName 指定的桶名称
 * @param objectName 指定的对象名称
 * @return
 */
byte[] getObject(String bucketName, String objectName);
```



示例：

```java
@Autowired
private OssClient ossClient;

/**
 * 以字节数组的方式获取对象
 */
@Test
public void getObject() {
    try {
        byte[] object = ossClient.opsForObject().getObject(defaultBucket, "01.jpg");
        File output = getOutputFile("01_OP1.jpg");
        log.info(">>> output: {}", output.getAbsolutePath());
        FileUtils.writeByteArrayToFile(getOutputFile("01_OP1.jpg"), object);
    } catch (Exception e) {
        Assert.fail(e.getMessage());
    }
}
```



#### 3.1.7. 通过文件方式获取对象

接口：

```java
/**
 * 将对象下载为文件
 *
 * @param bucketName 指定的桶名称
 * @param objectName 指定的对象名称
 * @param targetFile 目标文件
 * @param overwrite  目标文件若存在，是否进行覆盖
 */
void downloadObject(String bucketName, String objectName, File targetFile, boolean overwrite);
```



示例：

```java
@Autowired
private OssClient ossClient;

/**
 * 以文件方式获取对象
 */
@Test
public void downloadObject() {
    ossClient.opsForObject().downloadObject(
            defaultBucket,
            "02.jpg",
            getOutputFile("02_OP1.jpg"),
            true
    );
}
```



#### 3.1.8. 获取预签名对象的URL

接口：

```java
/**
 * 获取预签名对象的URL
 *
 * <p>用于私有桶临时开放的用于下载的URL</p>
 *
 * @param bucketName 指定的桶名称
 * @param objectName 指定的对象名称
 * @param expiry     URL过期时间(不得超过7天)
 * @param httpMethod 访问URL使用的Http Method
 * @return
 */
String getPresignedObjectUrl(String bucketName, String objectName, Duration expiry, Method httpMethod);
```



示例：

```java
@Autowired
private OssClient ossClient;

/**
 * 获取预签名对象的URL - 仅时效限制
 *
 * <p>注：chrome浏览器可能因为缓存问题，会造成URL过期依旧能够下载的情况，重开会话或启用无痕浏览可以避免此类问题</p>
 *
 */
@Test
public void getPresignedObjectUrl1() {
    String url = ossClient.opsForObject().getPresignedObjectUrl(
            defaultBucket,
            "01.jpg",
            Duration.ofSeconds(10),
            Method.GET
    );
    log.info(">>> {}", url);
    Assert.assertNotNull(url);
}
```



示例输出：

```
>>> http://api.minio.polarwin.cc/demo.user-api/01.jpg?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=user%2F20230307%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20230307T014832Z&X-Amz-Expires=10&X-Amz-SignedHeaders=host&X-Amz-Signature=09494cb41fed627bf8ef2e190cbca7b645cfe74299d4585cfb6158b8482960e4
```



#### 3.1.9. 获取预签名对象的URL并附加额外访问头限制

接口：

```java
/**
 * 获取预签名对象的URL
 *
 * <p>用于私有桶临时开放的用于下载的URL</p>
 *
 * @param bucketName   指定的桶名称
 * @param objectName   指定的对象名称
 * @param expiry       URL过期时间(不得超过7天)
 * @param extraHeaders 访问URL必须附加的访问头(Http Headers)
 * @param httpMethod   访问URL使用的Http Method
 * @return
 */
String getPresignedObjectUrl(String bucketName, String objectName, Duration expiry, Map<String, String> extraHeaders, Method httpMethod);
```



示例：

```java
@Autowired
private OssClient ossClient;

/**
 * 获取预签名对象的URL - 时效 & 请求头 限制
 */
@Test
public void getPresignedObjectUrl2() {
    Map<String, String> extraHeaders = new HashMap(1){{
        put("mykey", "d41d8cd98f00b204e9800998ecf8427e");
    }};
    String url = ossClient.opsForObject().getPresignedObjectUrl(
            defaultBucket,
            "01_E1.jpg",
            Duration.ofSeconds(20),
            extraHeaders,
            Method.GET
    );
    log.info(">>> {}", url);
    log.info(">>> 使用curl验证： curl --location --request GET '{}' --header '{}: {}'", url, "mykey", "d41d8cd98f00b204e9800998ecf8427e");
    Assert.assertNotNull(url);
}
```



输出：

```
>>> http://api.minio.polarwin.cc/demo.user-api/01_E1.jpg?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=user%2F20230307%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20230307T015554Z&X-Amz-Expires=20&X-Amz-SignedHeaders=host%3Bmykey&X-Amz-Signature=b46248c1b502766a626c594d1bdc43b2e57e06dca02b1bb0dfa2fdd218e3f86d

>>> 使用curl验证： curl --location --request GET 'http://api.minio.polarwin.cc/demo.user-api/01_E1.jpg?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=user%2F20230307%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20230307T015554Z&X-Amz-Expires=20&X-Amz-SignedHeaders=host%3Bmykey&X-Amz-Signature=b46248c1b502766a626c594d1bdc43b2e57e06dca02b1bb0dfa2fdd218e3f86d' --header 'mykey: d41d8cd98f00b204e9800998ecf8427e'
```



#### 3.1.10. 删除单个对象

接口：

```java
/**
 * 删除单个对象
 *
 * @param bucketName 指定的桶名称
 * @param objectName 定义的对象名称
 */
void removeObject(String bucketName, String objectName);
```



示例：

```java
@Autowired
private OssClient ossClient;

/**
 * 删除单个对象
 */
@Test
public void removeObject() {
    ossClient.opsForObject().removeObject(defaultBucket, "02_E.jpg");
}
```



#### 3.1.11. 删除多个对象

接口：

```java
/**
 * 删除多个对象
 *
 * @param bucketName  指定的桶名称
 * @param objectNames 定义的多个对象名称
 */
void removeObjects(String bucketName, String... objectNames);
```



示例：

```java
@Autowired
private OssClient ossClient;

/**
 * 删除多个对象
 */
@Test
public void removeObjects() {
    ossClient.opsForObject().removeObjects(defaultBucket, "01_E1.jpg", "02_E.jpg");
}
```



#### 3.1.12. 组合对象

接口：

```java
/**
 * 通过使用服务器端副本组合来自不同源对象的数据来创建对象(比如可以将文件分片上传，然后将他们合并为一个文件)
 *
 * <p>注意：每个分片都不能小于5MB，否则会报错：source xxx: size xxx must be greater than 5242880</p>
 *
 * <pre>Example:
 *      List<ComposeSource> composeSources = new ArrayList(2) {{
 *           add(ComposeSource.builder().bucket("bucket1").object("object1").build());
 *           add(ComposeSource.builder().bucket("bucket2").object("object2").build());
 *      }};
 *
 *      composeObject(composeSources, "bucket3", "object3");
 * </pre>
 *
 * @param composeSources 来源(待组合)对象集合(至少2个对象以上)
 * @param targetBucket   指定的目标桶名称(来源对象会合并至该桶)
 * @param objectName     定义的目标对象名称
 */
void composeObject(List<ComposeSource> composeSources, String targetBucket, String objectName);
```



示例：

```java
@Autowired
private OssClient ossClient;

/**
 * 组合对象
 */
@Test
public void composeObjects() {
    //上传分卷/分片文件
    Stream.of(new String[]{"wz.zip.001", "wz.zip.002"}).forEach(z -> ossClient.opsForObject().uploadObject(
            defaultBucket,
            new File(getResourceAsPath(z))
    ));
    //组合分卷/分片文件
    List<ComposeSource> composeSources = Stream
            .of(new String[]{"wz.zip.001", "wz.zip.002"})
            .map(o -> ComposeSource.builder().bucket(defaultBucket).object(o).build())
            .collect(Collectors.toList());
    ossClient.opsForObject().composeObject(composeSources, defaultBucket, "wz.zip");
    //输出组合后文件
    ossClient.opsForObject().downloadObject(
            defaultBucket,
            "wz.zip",
            getOutputFile("wz_op.zip"),
            true
    );
}
```

* 组合后的对象仍然在服务端存储



#### 3.1.13. 拷贝对象

接口：

```java
/**
 * 通过服务器端从一个对象复制数据来创建另一个对象
 *
 * @param sourceBucket     源桶名称
 * @param sourceObjectName 源对象名称
 * @param targetBucket     目标桶名称
 * @param targetObjectName 目标对象名称
 */
void copyObject(String sourceBucket, String sourceObjectName, String targetBucket, String targetObjectName);
```



示例：

```java
@Autowired
private OssClient ossClient;

/**
 * 拷贝对象
 */
@Test
public void copyObject() {
    ossClient.opsForObject().copyObject(defaultBucket, "01.jpg", defaultBucket, "01_copied.jpg");
}
```

* 拷贝后的对象仍然在服务端存储



#### 3.1.14. 设置对象标签

接口：

```java
/**
 * 设置对象标签
 *
 * @param bucket     指定的桶名称
 * @param objectName 指定的对象名称
 * @param tags       需要设置的标签
 */
void setObjectTags(String bucket, String objectName, Tag... tags);
```



示例：

```java
@Autowired
private OssClient ossClient;

@Test
public void setObjectTags() {
    ossClient.opsForObject().setObjectTags(
            defaultBucket,
            "01.jpg",
            Tag.builder().name("tk1").value("test1").build(),
            Tag.builder().name("tk2").value("test2").build(),
            Tag.builder().name("tk3").value("test3").build()
    );
}
```



#### 3.1.15. 获取对象标签

接口：

```java
/**
 * 获取对象标签
 *
 * @param bucket     指定的桶名称
 * @param objectName 指定的对象名称
 * @return
 */
List<Tag> getObjectTags(String bucket, String objectName);
```



示例：

```java
@Autowired
private OssClient ossClient;

/**
 * 获取对象标签
 */
@Test
public void getObjectTags(){
    List<Tag> tags = ossClient.opsForObject().getObjectTags(defaultBucket, "01.jpg");
    log.info(">>> tags: {}", tags);
    Assert.assertTrue(tags.size() == 3);

    Map<String, String> expectTags = new HashMap(3){{
        put("tk1", "test1");
        put("tk2", "test2");
        put("tk3", "test3");
    }};

    expectTags.entrySet().stream().forEach(e -> {
        log.info("validate tag with key: {}", e.getKey());
        Assert.assertTrue(tags.stream().filter(tag -> e.getKey().equals(tag.getName()) && e.getValue().equals(tag.getValue())).count() == 1);
    });
}
```



输出：

```
>>> tags: [Tag(name=tk2, value=test2), Tag(name=tk1, value=test1), Tag(name=tk3, value=test3)]
validate tag with key: tk3
validate tag with key: tk2
validate tag with key: tk1
```



#### 3.1.16. 删除对象标签

接口：

```java
/**
 * 删除对象标签
 *
 * @param bucket     指定的桶名称
 * @param objectName 指定的对象名称
 */
void deleteObjectTags(String bucket, String objectName);
```



示例：

```java
@Autowired
private OssClient ossClient;

/**
 * 删除对象标签
 */
@Test
public void deleteObjectTags() {
    ossClient.opsForObject().deleteObjectTags(defaultBucket, "01.jpg");
}
```

* 删除对象标签时，会删除对象所有的标签



#### 3.1.17. 获取对象的对象信息和元数据

接口：

```java
/**
 * 获取对象的对象信息和元数据
 *
 * @param bucket     指定的桶名称
 * @param objectName 指定的对象名称
 * @return
 */
StatObjectResponse statObject(String bucket, String objectName);
```



示例：

```java
@Autowired
private OssClient ossClient;

/**
 * 获取对象的对象信息和元数据
 */
@Test
public void statObject() {
    StatObjectResponse statObjectResponse = ossClient.opsForObject().statObject(defaultBucket, "01.jpg");
    log.info(">>> Object stat: {}", statObjectResponse);
}
```



输出：

```
>>> Object stat: ObjectStat{bucket=demo.user-api, object=01.jpg, last-modified=2023-01-31T07:37:46Z, size=2982554}
```



### 3.2. 桶操作

>  API不提供桶的创建、更新与删除，请到OSS服务端控制台进行操作。

#### 3.2.1. 判断桶是否存在

接口：

```java
/**
 * 判断桶是否存在
 *
 * @param bucketName 指定桶的名称
 * @return
 */
boolean bucketExists(String bucketName);
```



示例：

```java
@Autowired
private OssClient ossClient;

/**
 * 判断桶是否存在
 */
@Test
public void bucketExists() {
    Assert.assertTrue(ossClient.opsForBucket().bucketExists("demo.public"));
    Assert.assertTrue(ossClient.opsForBucket().bucketExists("demo.user-api"));
    Assert.assertFalse(ossClient.opsForBucket().bucketExists("demo.not-exists"));
}
```



#### 3.2.2. 设置桶策略

> 建议在OSS服务端进行手动设置，代码级别的调整可以作为临时应用，不建议作为标准使用

接口：

```java
/**
 * 设置桶策略
 *
 * @param bucketName   指定桶的名称
 * @param bucketPolicy 桶策略
 */
void setBucketPolicy(String bucketName, BucketPolicy bucketPolicy);
```



示例：

```java
@Autowired
private OssClient ossClient;

/**
 * 设置桶策略
 */
@Test
public void setBucketPolicy() {
    BucketPolicy p1Policy = BucketPolicyFactory.getBucketPolicy(BucketPolicyType.PUBLIC, "demo.p1");
    ossClient.opsForBucket().setBucketPolicy("demo.p1", p1Policy);

    BucketPolicy p2Policy = BucketPolicyFactory.getBucketPolicy(BucketPolicyType.READONLY, "demo.p2");
    ossClient.opsForBucket().setBucketPolicy("demo.p2", p2Policy);
}
```



#### 3.2.3. 列出桶内对象信息

接口：

```java
/**
 * 列出桶内对象信息
 *
 * @param listObjectsArgs 可构造基于桶列表对象设置的参数
 * @return
 */
Iterable<Result<Item>> listObjects(ListObjectsArgs listObjectsArgs);
```



示例：

```java
@Autowired
private OssClient ossClient;

/**
 * 列出桶内对象信息
 */
@Test
public void listObjects() {
    Iterable<Result<Item>> resultIterable = ossClient.opsForBucket().listObjects(ListObjectsArgs.builder()
            .bucket(defaultBucket)
            //带路径对象可进行递归
            .recursive(true)
            .prefix("wz")
            //.maxKeys(2) //在MinIO中无效，有可能是服务端问题，未深入调试
            .build());
    resultIterable.forEach(r -> {
        try {
            log.info(">>> item: {} -> {}", r.get().objectName(), r.get().size());
        } catch (Exception e) {
            Assert.fail(e.getMessage());
        }
    });
}
```



## 4. 关于不同AccessKey策略对于桶访问的影响验证

### 4.1. 示例1

示例：

```java
/**
 * 不同AccessKey策略对于桶访问的影响验证1
 *
 * <p>user001，控制台创建AccessKey时，开启Restrict beyond user policy功能，并进行配置。限制仅可访问操作demo.user-api的桶</p>
 *
 * <pre>
 * {
 *  "Version": "2012-10-17",
 *  "Statement": [
 *   {
 *    "Effect": "Allow",
 *    "Action": [
 *     "s3:*"
 *    ],
 *    "Resource": [
 *     "arn:aws:s3:::demo.user-api"
 *    ]
 *   },
 *   {
 *    "Effect": "Allow",
 *    "Action": [
 *     "s3:*"
 *    ],
 *    "Resource": [
 *     "arn:aws:s3:::demo.user-api/*"
 *    ]
 *   }
 *  ]
 * }
 * </pre>
 */
@Test
public void policyValidation1() {
    OssClient ossClient001 = new OssClient(
            "http://api.minio.polarwin.cc",
            "user001",
            "8X9EkYqkS3ZJG0wVV85eKXG7i96NiveO"

    );

    //01. 授权的私有桶
    StatObjectResponse statObjectResponse = ossClient001.opsForObject().statObject(defaultBucket, "01.jpg");
    log.info(">>> 授权私有桶： {}", statObjectResponse);

    //02. 未授的权私有桶
    try {
        statObjectResponse = ossClient001.opsForObject().statObject("demo.user-policy", "01.jpg");
        Assert.fail();
    } catch (Exception e) {
        log.error("未授权的私有桶： " + e.getMessage());
    }

    //03. 未授权的公有桶
    try {
        statObjectResponse = ossClient001.opsForObject().statObject("demo.public", "01.jpg");
    } catch (Exception e) {
        log.error("未授权的公有桶： " + e.getMessage());
    }

}
```



输出：

```
>>> 授权私有桶： ObjectStat{bucket=demo.user-api, object=01.jpg, last-modified=2023-01-31T07:37:46Z, size=2982554}

未授权的私有桶： error occurred
ErrorResponse(code = AccessDenied, message = Access Denied., bucketName = demo.user-policy, objectName = null, resource = /demo.user-policy, requestId = 174A0263DAA85B06, hostId = 1375be63-b345-4abc-bb7d-1a6176d123e0)

未授权的公有桶： error occurred
ErrorResponse(code = AccessDenied, message = Access Denied., bucketName = demo.public, objectName = null, resource = /demo.public, requestId = 174A0263DDBF8114, hostId = 1375be63-b345-4abc-bb7d-1a6176d123e0)
```



