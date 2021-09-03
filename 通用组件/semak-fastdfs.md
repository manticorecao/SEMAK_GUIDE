# semak-fastfds

`semak-fastdfs` 基于分布式文件存储服务FastDFS提供的客户端组件，其主要特性包括：


1. 文件的上传、下载和删除。
1. 获取文件的详细信息。
1. 以Http的方式获取文件下载的链接，并防止盗链。



## 1. 先决条件


### 1.1. 环境配置


1. Open JDK 1.8+，并已配置有效的环境变量。
1. Maven 3.3.x+，并已配置有效的环境变量。



### 1.2. Maven依赖配置


```xml
<dependency>
    <groupId>com.github.semak.fdfs</groupId>
    <artifactId>semak-fastdfs-spring-boot-starter</artifactId>
    <version>最新RELEASE版本</version>
</dependency>
```




## 2. 属性定义
```yaml
spring:
  fastdfs:
    enabled: true
    tracker-server:
      - 192.168.1.233:22122
      - 192.168.1.234:22122
    http:
      anti-steal-token: true
      secret-key: QXtEsrIBrh4g3yDiNcigMzx/VlC8beDvKQ/Wnl203BIkeJ3XxFLOAymUC1ldoidmyjrsq5AwB5GQiU0shQZeGQ==
```



### 2.1. 属性描述

| **属性** | **数据类型** | **必填** | **默认值** | **描述** |
| :---- | :---- | :---- | :---- | :---- |
| **spring.fastdfs.enabled** | boolean | 否 | true | 是否启用fastdfs的客户端 |
| **spring.fastdfs.connect-timeout-in-ms** | int | 否 | 5000 | 连接超时（单位：ms） |
| **spring.fastdfs.network-timeout-in-ms** | int | 否 | 30000 | 响应超时（单位：ms） |
| **spring.fastdfs.charset** | String | 否 | UTF-8 | 字符集 |
| **spring.fastdfs.tracker-server** | List | 是 | N/A | Tracker Server的地址，格式为IP:PORT |
| **spring.fastdfs.http.tracker-http-port** | int | 否 | 8080 | Tracker Server的Http服务端口（暂时无需关注） |
| **spring.fastdfs.http.storage-http-port** | int | 否 | 22090 | Storage Server的Http服务端口（暂时无需关注） |
| **spring.fastdfs.http.anti-steal-token** | boolean | 否 | false | 是否启用文件防盗链机制 |
| **spring.fastdfs.http.secret-key** | String | 是 | N/A | 防盗链使用的密钥，仅在启用文件防盗链机制时生效 |
| **spring.fastdfs.connection-pool.enabled** | boolean | 否 | true | 是否启用连接池 |
| **spring.fastdfs.connection-pool.max-count-per-entry** | int | 否 | 500 | 每一个IP:Port的最大连接数，0为没有限制 |
| **spring.fastdfs.connection-pool.max-idle-time-in-ms** | int | 否 | 3600000 | 每一个连接的最大空闲时间（单位：ms） |
| **spring.fastdfs.connection-pool.max-wait-time-in-ms** | int | 否 | 1000 | 达到最大连接数时候的最大等待时间（单位：ms） |
| **spring.fastdfs.storage-client-pool.max-total** | int | 否 | 30 | 实例化的StorageClinet的最大数量 |
| **spring.fastdfs.storage-client-pool.max-idle** | int | 否 | 10 | 最大空闲StorageClient实例数量 |
| **spring.fastdfs.storage-client-pool.min-idle** | int | 否 | 2 | 最小空闲StorageClient实例数量 |



## 3. 使用方式


### 3.1. 上传文件
```java
@Autowired
private FdfsClient fdfsClient;

@Test
public void upload() throws IOException {
    InputStream inputStream = Thread.currentThread().getContextClassLoader().getResourceAsStream("files/FX01.png");
    byte[] b = IOUtils.toByteArray(inputStream);
    FdfsFile fdfsFile = new FdfsFile();
    fdfsFile.setName("FX01");
    fdfsFile.setExtension("png");
    fdfsFile.setContent(b);
    PutFileResult result = fdfsClient.putFile(fdfsFile);
    log.info(">>> Result: {}", result);
    if(!result.isSuccess()){
        Assert.fail(result.getMessage());
    }
}
```
> 此处的返回值会给到2个关键值，需要记录下来，作为后续操作的基础参数。

执行结果：
```
2020-10-10 16:03:04.679  INFO 90802 --- [main] com.github.semak.fdfs.DefaultFdfsClient  : File upload costs:23 ms
2020-10-10 16:03:04.680  INFO 90802 --- [main] com.github.semak.fdfs.DefaultFdfsClient  : File upload successfully ==> groupName: group1, remoteFileName: M00/00/06/wKgB3F-BariAW0AHAPlhYnqCvD8713.png
2020-10-10 16:03:04.681  INFO 90802 --- [main] com.github.semak.fdfs.FastdfsTestcase    : >>> Result: PutFileResult(groupName=group1, remoteFileName=M00/00/06/wKgB3F-BariAW0AHAPlhYnqCvD8713.png)
```



### 3.2. 获取文件

```java
@Autowired
private FdfsClient fdfsClient;

@Test
public void getFile() throws Exception {
    String home = System.getProperty("user.home");
    GetFileResult result = fdfsClient.getFile("group1", "M00/00/06/wKgB3F-BariAW0AHAPlhYnqCvD8713.png");
    if(!result.isSuccess()){
        Assert.fail(result.getMessage());
    }
    IOUtils.write(result.getFileBytes(), new FileOutputStream(home + "/Downloads/fdfs/图片.png"));
}
```
此处获取成功文件后，会将文件写入到指定位置。



### 3.3. 获取文件的下载链接（防盗链）

```java
@Autowired
private FdfsClient fdfsClient;

@Test
public void getFileLink() {
    GetFileLinkResult result = fdfsClient.getFileLink("group1", "M00/00/06/wKgB3F-BariAW0AHAPlhYnqCvD8713.png");
    if(!result.isSuccess()){
        Assert.fail(result.getMessage());
    }
    log.info(">>> result: {}", result.getFileLink());
}
```
执行结果：
```
2020-10-10 16:06:39.809  INFO 90861 --- [           main] com.github.semak.fdfs.FastdfsTestcase    : >>> result: http://192.168.1.216:22090/group1/M00/00/06/wKgB3F-BariAW0AHAPlhYnqCvD8713.png?ts=1602317199&token=4425e0ffe2f1a927872ff1999143c473&attname=FX01.png
```
结果中的链接有3个参数，含义如下：

- ts - 创建链接的时间戳
- token - 访问文件使用的令牌，防止盗链
- attname - 下载文件时，通过此属性的值来重命名文件。当含有中文时，会给到编码后的值。



### 3.4. 获取文件信息
```java
@Autowired
private FdfsClient fdfsClient;

@Test
public void getFileInfo() {
    FileInfoResult result = fdfsClient.getFileInfo("group1", "M00/00/06/wKgB3F-BariAW0AHAPlhYnqCvD8713.png");
    log.info(">>> Result: {}", result);
    if(!result.isSuccess()){
        Assert.fail(result.getMessage());
    }
}
```
执行结果：
```
2020-10-10 16:10:12.258  INFO 90933 --- [main] com.github.semak.fdfs.FastdfsTestcase    : >>> Result: FileInfoResult(fileType=1, sourceIpAddr=192.168.1.220, fileSize=16343394, createTimestamp=Sat Oct 10 16:03:04 CST 2020, crc32=2055388223, metaInfo=[org.csource.common.NameValuePair@6c451c9c, org.csource.common.NameValuePair@31c269fd])
```



### 3.5. 删除文件

```java
@Autowired
private FdfsClient fdfsClient;

@Test
public void removeFile(){
    RemoveFileResult result = fdfsClient.removeFile("group1", "M00/00/06/wKgB3F-BariAW0AHAPlhYnqCvD8713.png");
    if(!result.isSuccess()){
        Assert.fail(result.getMessage());
    }
}
```



## 4. 结合SpringMVC的使用方式


> 这里简单的示例结合Spring的上传下载的写法

### 4.1. 上传文件
```java
@PostMapping("/upload")
public Response<String> uploadFile(@RequestParam MultipartFile file, @RequestParam String user) throws IOException {
    //全局限制大小后，如有需要，可以进行局部大小限制
    int maxSize = 30 * 1024 * 1024;
    if(file.getSize() > maxSize){
        throw new MaxUploadSizeExceededException(maxSize);
    }

    FdfsFile fdfsFile = new FdfsFile();
    fdfsFile.setName(FilenameUtils.removeExtension(file.getOriginalFilename()));
    fdfsFile.setExtension(FilenameUtils.getExtension(file.getOriginalFilename()));
    fdfsFile.setContent(file.getBytes());
    PutFileResult putFileResult = fdfsClient.putFile(fdfsFile);

    if(putFileResult.isSuccess()){
        log.info(">>> Result: {}", putFileResult);
        return Response.ofSuccess("Upload successfully.");
    }else{
        return Response.ofFailure(-99999L, putFileResult.getMessage());
    }
}
```


全局相关配置如下：


```yaml
  spring:
    servlet:
      multipart:
        enabled: true
        resolve-lazily: true
        #单文件最大限制(全局)
        max-file-size: 100MB
        location: ${java.io.tmpdir}
```



### 4.2. 下载文件

```java
@GetMapping("/download")
public Response<String> downloadFile() {
    //groupName: group1, remoteFileName: M00/00/07/wKgB2F-BW-yAaWTyAT-8KOOSHyo486.png
    GetFileLinkResult getFileLinkResult = fdfsClient.getFileLink("group1", "M00/00/07/wKgB2F-BW-yAaWTyAT-8KOOSHyo486.png");
    if(getFileLinkResult.isSuccess()){
        return Response.ofSuccess(getFileLinkResult.getFileLink());
    }else{
        return Response.ofFailure(-99999L, getFileLinkResult.getMessage());
    }
}
```
这里返回的是一个文件服务器的链接，提供了盗链机制，这里不再将文件下载到应用中，以明确职责、节省资源。
