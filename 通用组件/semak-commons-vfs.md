# semak-commons-vfs

`semak-commons-vfs`是基于`apache commons vfs2`构建的组件，通过对原有类库的功能裁剪、抽象，提供一套独立而简约的API接口来访问不同的虚拟文件系统。其主要特性包括：

1. 用于访问虚拟文件系统上不同类型文件的简约且易操作的API。
2. 呈现各种不同来源文件的统一视图，这些文件和来源包括：ZIP、TAR、TARGZ、SFTP、FTP等。
3. 通过统一接口为不同来源的文件提供相应语义的操作，比如说，**putFile**这个方法，对于存档文件来说是将文件压缩到存档文件中；而对于FTP来说，则是将本地文件上传到FTP服务器的指定位置。



## 1. 先决条件

### 1.1. 环境配置


1. Open JDK 1.8+，并已配置有效的环境变量。
1. Maven 3.3.x+，并已配置有效的环境变量。



### 1.2. Maven依赖配置

```xml
<dependency>
   <groupId>com.github.semak.commons</groupId>
   <artifactId>semak-commons-vfs-spring-boot-starter</artifactId>
   <version>最新RELEASE版本</version>
</dependency>
```



# 2. 不同文件系统支持的操作

虽然组件提供了`VirtualFileObject`的接口来操作不同文件系统，但其支持的操作根据文件系统类型不同会有一定差别，请参照下表执行：

| 文件系统 | 读操作 | 写操作 | 创建/删除操作 | 拷贝操作 |
| -------- | ------ | ------ | ------------- | -------- |
| **ZIP**  | 支持   | 支持   | 不支持        | 不支持   |
| **TAR**  | 支持   | 支持   | 不支持        | 不支持   |
| **TGZ**  | 支持   | 支持   | 不支持        | 不支持   |
| **FTP**  | 支持   | 支持   | 支持          | 支持     |
| **SFTP** | 支持   | 支持   | 支持          | 支持     |



# 3. 使用方式

## 3.1. 初始化

通过声明虚拟文件对象接口`com.github.semak.commons.vfs.core.VirtualFileObject`，并搭配以不同实现类来实现初始化。下面会列出几类文件系统的初始化代码片段。

1. ZIP文件对象初始化

   ```java
   //1.装配fileSystemManager
   @Autowired
   private FileSystemManager fileSystemManager;
   
   //2.将接口VirtualFileObject指向实例化的ZipVFO
   VirtualFileObject virtualFileObject = new ZipVFO(fileSystemManager, StandardCharsets.UTF_8);
   ```

2. TAR文件对象初始化

   ```java
   //1.装配fileSystemManager
   @Autowired
   private FileSystemManager fileSystemManager;
   
   //2.将接口VirtualFileObject指向实例化的TarVFO
   VirtualFileObject virtualFileObject = new TarVFO(fileSystemManager, StandardCharsets.UTF_8);
   ```

3. TAR.GZ文件对象初始化

   ```java
   //1.装配fileSystemManager
   @Autowired
   private FileSystemManager fileSystemManager;
   
   //2.将接口VirtualFileObject指向实例化的TarGzVFO
   VirtualFileObject virtualFileObject
   VirtualFileObject virtualFileObject = new TarGzVFO(fileSystemManager, StandardCharsets.UTF_8);
   ```

4. SFTP文件对象初始化（配置属性参照代码注释）

   ```java
   //1.装配fileSystemManager
   @Autowired
   private FileSystemManager fileSystemManager;
   
   //2.SFTP配置项
   /** 2.1.用户名密码模式 **/
   SftpConfiguration sftpConfiguration = SftpConfiguration.builder()
                   .userAuthType(UserAuthType.PASSWORD)
                   .domain("192.168.3.61")
                   .username("root")
                   .password("12345678")
                   .build();
           
   /** 2.2.密钥对模式 **/
   SftpConfiguration sftpConfiguration = SftpConfiguration.builder()
                   .userAuthType(UserAuthType.PUBLICKEY)
                   .domain("192.168.3.61")
                   .username("root")
                   .privateKey(new File( System.getProperty("user.home").concat(File.separator).concat(".ssh/id_rsa")))
                   .build();
   
   //3.将接口VirtualFileObject指向实例化的SftpVFO
   virtualFileObject = new SftpVFO(fileSystemManager, sftpConfiguration);
   ```

5. FTP文件对象初始化（配置属性参照代码注释）

   ```java
   //1.装配fileSystemManager
   @Autowired
   private FileSystemManager fileSystemManager;
   
   //2.FTP配置项
   FtpConfiguration ftpConfiguration = FtpConfiguration.builder()
                   .domain("192.168.3.61")
                   .username("ftpuser")
                   .password("123456")
                   .userDirIsRoot(false)
                   .build();
   
   //3.将接口VirtualFileObject指向实例化的tpVFO
   virtualFileObject = new FtpVFO(fileSystemManager, ftpConfiguration);
   ```

   

## 3.2. 接口操作

下面将详细介绍虚拟文件对象接口`com.github.semak.commons.vfs.core.VirtualFileObject`的各个方法使用方式与作用范围。

### 3.2.1. createFile

* 支持**FTP**、**SFTP**，语义为**在FTP/SFTP服务器端的指定路径创建文件**

  ```java
  virtualFileObject.createFile(new File("/home/vfs/a.txt"));
  ```

  * 不存在的目标目录会一起被创建出来

  * 通过指定配置值`userDirIsRoot`来判定远程文件的根路径起始点为**真实根路径**或是**以当前用户目录的起始路径为根路径**。在FTP中，如目录被限制无法切换，则此属性失效

    

### 3.2.2. createFolder

* 支持**FTP**、**SFTP**，语义为**在FTP/SFTP服务器端的指定路径创建文件夹**

  ```java
  virtualFileObject.createFolder(new File("/home/vfs/c"));
  ```

  * 不存在的目标目录会一起被创建出来
  * 通过指定配置值`userDirIsRoot`来判定远程文件的根路径起始点为**真实根路径**或是**以当前用户目录的起始路径为根路径**。在FTP中，如目录被限制无法切换，则此属性失效

  

### 3.2.3. copyTo

* 支持**FTP**、**SFTP**，语义为**在FTP/SFTP服务器端将源文件（夹）拷贝到目标文件（夹）**。需要注意的是，将源文件（夹）和目标文件（夹）均为FTP/SFTP服务器端的远程文件（夹）

  ```java
  virtualFileObject.copyTo(new File("/home/vfs/a.txt"), new File("/home/vfs/c/a1.txt"));        
  ```

  * 不存在的目标目录会一起被创建出来
  * 通过指定配置值`userDirIsRoot`来判定远程文件的根路径起始点为**真实根路径**或是**以当前用户目录的起始路径为根路径**。在FTP中，如目录被限制无法切换，则此属性失效



### 3.2.4. exists

* 支持**FTP**、**SFTP**，语义为**在FTP/SFTP服务器端判断目标文件（夹）是否存在**。

  ```java
  boolean exists = virtualFileObject.exists(new File("/home/vfs/c/a1.txt"));
  ```

  * 目标文件（夹）存在则返回`true`，反之则返回`false`



### 3.2.5. removeFile

* 支持**FTP**、**SFTP**，语义为**在FTP/SFTP服务器端删除目标文件（夹）**。

  ```java
  virtualFileObject.removeFile(new File("/home/vfs/c/b1.txt"));
  ```

  * 只会删除指定文件和目录，不会影响其父目录

  * 通过指定配置值`userDirIsRoot`来判定远程文件的根路径起始点为**真实根路径**或是**以当前用户目录的起始路径为根路径**。在FTP中，如目录被限制无法切换，则此属性失效

    

### 3.2.6. putFile

* 支持**ZIP**、**TAR**、**TARGZ**，语义为**将指定的本地路径上的文件或文件夹下的文件打包到指定路径**

  ```java
  virtualFileObject.putFile(new File(userPath + "A/图片/29空格.png"), new File(targetPath + "targetF1.zip"));
  ```

  * 不存在的目标目录会一起被创建出来

  * 如果源文件为文件夹，则将文件夹下所有文件进行打包

    

* 支持**FTP**、**SFTP**，语义为**将指定的本地路径上的文件（夹）上传到FTP/SFTP服务器端指定的远程路径上**

  ```java
  File localFile = new File(localPath + "01.zip");
  File remoteFile = new File("/home/vfs/01_t.zip");
  virtualFileObject.putFile(localFile, remoteFile);
  ```

  * 源文件（夹）为本地文件（夹），目标文件（夹）为FTP/SFTP服务器上的远程文件（夹）
  * 不存在的目录会一起被创建出来
  * 通过指定配置值`userDirIsRoot`来判定远程文件的根路径起始点为**真实根路径**或是**以当前用户目录的起始路径为根路径**。在FTP中，如目录被限制无法切换，则此属性失效



### 3.2.7. retrieveFile

* 支持**ZIP**、**TAR**、**TARGZ**，语义为**将指定的本地路径上的已打包文件（或包内文件）解压到到指定路径**

  ```java
  //zip包解压
  virtualFileObject.retrieveFile(new File(userPath + "02.zip"), new File(targetPath));
  
  //zip包内指定文件解压
  virtualFileObject.retrieveFile(new File(userPath + "02.zip!/dist/tinymce/license.txt"), new File(targetPath + "license.txt"));
  ```

  * 不存在的目标目录会一起被创建出来
  * 包内文件以此格式表达：`包文件全路径!包内文件绝对路径`



* 支持**FTP**、**SFTP**，语义为**将FTP/SFTP服务器端指定的文件（夹）下载到指定的本地路径上**

  ```java
  File remoteFile = new File("/home/vfs/01_t.zip");
  File localFile = new File(localPath + "/temp/01.zip");
  virtualFileObject.retrieveFile(remoteFile, localFile);
  ```

  * 源文件（夹）为FTP/SFTP服务器上的远程文件（夹），目标文件（夹）为本地文件（夹）
  * 不存在的目录会一起被创建出来
  * 通过指定配置值`userDirIsRoot`来判定远程文件的根路径起始点为**真实根路径**或是**以当前用户目录的起始路径为根路径**。在FTP中，如目录被限制无法切换，则此属性失效



## 3.3. 文件选择器

我们可以看到，在虚拟文件对象接口`com.github.semak.commons.vfs.core.VirtualFileObject`中的一些方法支持`org.apache.commons.vfs2.FileSelector`类型的参数，其主要作用在于按自定义规则筛选指定文件，再进行操作。比如：

```java
virtualFileObject.retrieveFile(new File(userPath + "02.zip"), new FileSelector() {
    @Override
    public boolean includeFile(FileSelectInfo fileInfo) throws Exception {
        return fileInfo.getFile().getName().getExtension().equals("js");
    }

    @Override
    public boolean traverseDescendents(FileSelectInfo fileInfo) throws Exception {
        return true;
    }
}, new File(targetPath));
```

* **includeFile**: 会将zip包内的所有文件扩展名为js的文件解压出来；如果是FTP/SFTP，则会将所有文件扩展名为js的文件下载下来
* **traverseDescendents**：是否进行递归遍历
