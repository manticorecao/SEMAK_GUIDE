# semak-commons-model

`semak-commons-model`组件主要提供Model相关的功能，其特性主要包括：


1. 通用Model基类。
2. 支持分页结果转换器。
3. 支持基于掩码和加解密方式的数据脱敏，加解密方式支持SPI扩展。



## 1. 先决条件


### 1.1. 环境配置


1. Open JDK 1.8+，并已配置有效的环境变量。
2. Maven 3.3.x+，并已配置有效的环境变量。



### 1.2. Maven依赖配置


```xml
<dependency>
    <groupId>com.github.semak.commons</groupId>
    <artifactId>semak-commons-model</artifactId>
    <version>最新RELEASE版本</version>
</dependency>
```



## 2. 通用Model基类定义


**Model**这个词涵盖很多的定义，如：JavaBean、POJO（Plain Old Java Object）、Domain Object等，它更多的是描述一种倾向于做数据载体的Java对象。


这里为了简单起见，统一称为**Model**，其特性定义为：


- 对象中必须有**Getter**和**Setter**方法。
- 作为数据的载体。

下面我们按照Model的使用场景分别进行阐述。


**P.S.: 但凡项目中定义的Model对象，必须按以下场景进行基类继承，否则可能会有部分功能缺失。**



### 2.1. 持久对象 - PO


持久对象(Persistence Object)。与**表结构**一一对应，最形象的理解就是一个PO对象对应数据库中的一条记录。


持久对象一般会由MyBatis Generator（MBG）插件生成，并拷贝到`model`模块的对应包中。


生成的PO类会默认继承`com.github.semak.commons.model.po.BasePo`。



### 2.2. 业务对象 - BO


业务对象(Business Object)。把业务逻辑使用的要素封装为一个对象，这个对象可以包括一个或多个PO，同时也可以添加额外属性。


BO类默认继承`com.github.semak.commons.model.bo.BaseBo`。



### 2.3. 数据传输对象 - DTO


数据传输对象(Data Transfer Object)。用于服务交互(如：Restful、RPC等)时使用的对象。


DTO类必须**继承**默认基类，但由于一组DTO对象由**请求对象**和**响应对象**构成，故继承的基类有所不同。


- Request DTO默认继承`com.github.semak.commons.model.dto.BaseRequest`。
- Response DTO默认继承`com.github.semak.commons.model.dto.BaseResponse`。



### 2.4. 视图对象 - VO


视图对象(View Object)。用于视图层的数据展现。


VO类默认继承`com.github.semak.commons.model.vo.BaseVo`。



### 2.5. 创建一个Model


这是个简单的活，只要set..set...set....，好累😫  ~~


如果你熟知**Builder设计模式**，那你一定非常喜欢它构造对象的方式。但是，我们的Model如果去手写Builder模式势必增加不少工作量。


所以，组件提供了两中方式来快速创建一个Model。当然，这只是额外的Model构建方式，如果你不喜欢，可以不用。



#### 2.5.1. 基于Lombok注解的Builder


这是一种比较简单的方式，在你的项目支出Lombok注解的情况下，需要添加几个类注解即可，它们分别是：


```java
@AllArgsConstructor
@NoArgsConstructor
@Builder
```


> **注：**
> 1. 上面除`@Builder`以外的注解是为了保证Model能够以无参构造的方式创建，便于后续一些框架机制的处理。
> 2. PO是由MyBatis Generator（MBG）插件生成的，经扩展后，已对上面的注解进行了支持。



这里是一个例子：


```java
@AllArgsConstructor
@NoArgsConstructor
@Builder
@Getter
@Setter
public class DemoUser extends BasePo {
    /**
     * 编号
     */
    private BigDecimal id;

    /**
     * 用户名
     */
    private String username;

    /**
     * 年龄
     */
    private Integer age;

    /**
     * 性别： M - 男 F - 女
     */
    private String sex;
}
```


```java
//使用builder模式构建对象
DemoUser demoUser = DemoUser.builder()
                .username("George Zold")
                .age(36)
                .sex("M")
                .build();
```



#### 2.5.2. 基于Lambda的Builder


如果你没有也不想使用Lombok注解，组件也提供了基于Lambda的Builder。它不需要对Model做任何修饰，只需要像下面这样做：


```java
DemoUser demoUser = GenericBuilder
         .of(DemoUser::new)
         .with(DemoUser::setAge, 33)
         .with(DemoUser::setUsername, "Geroge")
         .with(DemoUser::setSex, "M")
         .build();
```


相比Lombok注解，此种Builder方式的精简程度略显欠缺，优点是没有任何注解的污染。



## 3. 分页结果转换


为了方便分页对象`Page<PO>`到`DTO`对象的快速转换，这里提供了`com.github.semak.commons.model.dto.SimplePageInfo`类来进行处理。


### 3.1. SimplePageInfo的结构


```java
/**
 * 通用PageInfo
 *
 * <p>用于基于PageHelper的分页PO到分页DTO的转换</p>
 *
 * @author caobin
 * @version 1.0 2019.06.10
 */
@Data
public class SimplePageInfo<D> extends BaseResponse {

    /**
     * 当前页
     */
    private int currentPage;

    /**
     * 每页的数量
     */
    private int pageSize;

    /**
     * 总页数
     */
    private int totalPage;

    /**
     * 总记录数
     */
    private long totalNum;

    /**
     * (转换后的)结果集
     */
    private List<D> list;


    public SimplePageInfo(){}


    public SimplePageInfo(Page<? extends Serializable> page, Class<D> targetClass){
        this(page, poList -> {
            if (targetClass != null && !CollectionUtils.isEmpty(poList)) {
                return poList.stream().map(p -> {
                    Object targetObject = newInstance(targetClass);
                    //BeanCopier本身已有缓存机制，故这里不用再做缓存
                    BeanCopier beanCopier = BeanCopier.create(p.getClass(), targetObject.getClass(), false);
                    beanCopier.copy(p, targetObject, null);
                    return (D) targetObject;
                }).collect(Collectors.toList());
            }
            return null;
        });
    }


    public SimplePageInfo(Page<? extends Serializable> page, Transformer<D> transformer){
        if(!(page == null || page.isEmpty())){
            currentPage = page.getPageNum();
            pageSize = page.getPageSize();
            totalPage = page.getPages();
            totalNum = page.getTotal();
            list = transformer.transform(page.getResult());
        }
    }


    private static Object newInstance(Class<?> clazz){
        try {
            return clazz.newInstance();
        } catch (InstantiationException | IllegalAccessException e) {
            throw new IllegalArgumentException(e);
        }
    }

    @FunctionalInterface
    public interface Transformer<D> {

        /**
         * 结果集转换
         * @param poList
         * @return
         */
        List<D> transform(List<? extends Serializable> poList);
    }
}
```


关于分页的字段名称定义，请尽量与前端按此标准进行协商。如有不一致，亦可根据此类进行扩展。



### 3.2. 使用方式


当PO和DTO中**需要映射的字段名一致**，只是数量不一致，我们可以这样进行转换：


> 下面例子中的`DemoUser`为PO类，`UserResponse`为DTO类

```java
Page<DemoUser> result = demoUserDao.selectAllPaged(new RowBounds(1, 4));
SimplePageInfo<UserResponse> simplePageInfo = new SimplePageInfo<>(result, UserResponse.class);
```



### 3.3. 自定义转换的使用方式


当PO和DTO中**需要映射的字段名不一致**，我们可以这样进行转换：


> 例子中的`DemoUser`为PO类，`UserCustomResponse`为DTO类

```java
Page<DemoUser> result = demoUserDao.selectAllPaged(new RowBounds(1, 4));
        SimplePageInfo<UserCustomResponse> simplePageInfo = new SimplePageInfo<>(result, poList -> poList.stream().map(po -> (DemoUser)po).map(po ->
           UserCustomResponse.builder()
                    .id(po.getId().longValue())
                    .user(po.getUsername())
                    .age(po.getAge())
                    .sex(po.getSex())
                    .build()
        ).collect(Collectors.toList()));
```



## 4. 数据脱敏处理


数据脱敏属于安全控制的范畴。对互联网公司、传统行业来说，数据安全一直是极为重视和敏感的话题。数据脱敏是指对某些敏感信息通过脱敏规则进行数据的变形，实现敏感隐私数据的可靠保护。涉及客户安全数据或者一些商业性敏感数据，如身份证号、手机号、卡号、客户号等个人信息按照相关部门规定，都需要进行数据脱敏。


组件提供了2中不同类型的脱敏方式。



### 4.1. 字段掩码处理


字段掩码处理：将指定字段的值用符号进行部分遮蔽，以起到脱敏作用。这是一种单向处理方式，经脱敏处理后无法再恢复原有的值。


当你继承了Model基类时，默认就启用了基于**字段掩码**的脱敏功能。


首先，你需要了解`com.github.semak.commons.model.desen.Mask`注解：


```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface Mask {

    int preReservedLen() default 0;

    int postReservedLen() default 0;

    String mark() default "*";
}
```
| 字段属性 | 默认值 | 描述 |
| :--- | :--- | :--- |
| **preReservedLen** | 0 | 前置保留（不需要掩盖）的长度 |
| **postReservedLen** | 0 | 后置保留（不需要掩盖）的长度 |
| **mark** | `*` | 掩码符号 |



对当前Model的字段添加`@Mask`注解时，其`toString()`方法的输出会针对这个字段进行数据遮蔽。


**这里是一个例子：**


我们先为`DemoUser`类的`username`字段添加`@Mask`注解。


```java
@AllArgsConstructor
@NoArgsConstructor
@Builder
@Getter
@Setter
public class DemoUser extends BasePo {
    private BigDecimal id;
    
    @Mask(preReservedLen = 2, postReservedLen = 3)
    private String username;

    private Integer age;
    
    private String sex;
}
```


构造`DemoUser`，并以`toString()`的方式输出。


```java
DemoUser demoUser = DemoUser.builder()
                .username("George Zold")
                .age(36)
                .sex("M")
                .build();
log.info(">>> Masked result: {}", demoUser);
log.info(">>> Real field (username): {}", demoUser.getUsername());
log.info(">>> Masked field (username): {}", demoUser.getMaskedField("username"));
```


日志输出如下：


```
17:18:22.294 [main] INFO com.github.semak.commons.model.MaskTestcase - >>> Masked result: DemoUser[id=<null>,username=Ge******ld,age=36,sex=M]
17:18:22.318 [main] INFO com.github.semak.commons.model.MaskTestcase - >>> Real field (username): George Zold
17:18:22.384 [main] INFO com.github.semak.commons.model.MaskTestcase - >>> Masked field (username): Ge*****old
```


- 我们可以看到`username`以前2后3的方式保留了值的明文部分，其他部分全部以`*`进行遮蔽。
- 当我们需要获取指定的经掩码处理过字段值，可以通过`demoUser.getMaskedField("username")`来获取。

**P.S.: 如果有对象嵌套，请保证嵌套的对象也继承了Model基类。**



### 4.2. 字段加解密处理


字段加解密处理：将指定字段的值用加密算法进行加/解密，以起到脱敏作用。这是一种双向的处理方式，经脱敏处理后的字段可以通过解密恢复原有的值。


组件默认使用`AES128 + BASE64`的方式进行加/解密。


首先，你需要了解`com.github.semak.commons.model.desen.Crypt`注解：


```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface Crypt {

    boolean encrypt() default true;

    boolean decrypt() default true;
}
```
| 字段属性 | 默认值 | 描述 |
| :--- | :--- | :--- |
| **encrypt** | true | 启用字段加密处理 |
| **decrypt** | true | 启用字段解密处理 |



如果有嵌套需求，你还需要了解`com.github.semak.commons.model.desen.NestedCrypt`注解：


```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface NestedCrypt {
}
```


下面，我们分几种方式来讲解如何使用。



#### 4.2.1. 平铺方式


**平铺方式**是指只会在当前Model中，以`@Crypt`注解标注**String类型**和**基于String泛型的集合类型**的字段。


**String类型的加解密**


我们先为`DemoUser`类的`username`字段添加`@Crypt`注解。


```java
@AllArgsConstructor
@NoArgsConstructor
@Builder
@Getter
@Setter
public class DemoUser extends BasePo {

    private BigDecimal id;

    @Crypt
    private String username;

    private Integer age;

    private String sex;
}
```


构造`DemoUser`，当需要获取加密对象时，使用`demoUser.getEncryptObject`获取原对象的加密对象，加密对象深度克隆自原对象，数据间不会有串扰。


```java

/** 加密 **/

DemoUser demoUser = DemoUser.builder()
               .username("George Zold")
               .age(36)
               .sex("M")
               .build();
log.info(">>> Encrypted result: {}", demoUser.getEncryptObject(DemoUser.class));
log.info(">>> Real field (username): {}", demoUser.getUsername());
log.info(">>> Encrypted field (username): {}", demoUser.getEncryptObject(DemoUser.class).getUsername());

//-------------------------------------------------------------------------------------------------------------------------------------

/** 解密 **/

DemoUser demoUser = DemoUser.builder()
                .username("CPF1ojl2JZnSgMY/5UQ8Uw==")
                .age(36)
                .sex("M")
                .build();
log.info(">>> Decrypted result: {}", demoUser.getDecryptObject(DemoUser.class));
log.info(">>> Real field (username): {}", demoUser.getUsername());
log.info(">>> Decrypted field (username): {}", demoUser.getDecryptObject(DemoUser.class).getUsername());
```


日志输出


```
17:13:11.502 [main] INFO com.github.semak.commons.model.CryptTestcase - >>> Encrypted result: DemoUser[id=<null>,username=CPF1ojl2JZnSgMY/5UQ8Uw==,age=36,sex=M]
17:13:11.523 [main] INFO com.github.semak.commons.model.CryptTestcase - >>> Real field (username): George Zold
17:13:11.526 [main] INFO com.github.semak.commons.model.CryptTestcase - >>> Encrypted field (username): CPF1ojl2JZnSgMY/5UQ8Uw==
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
17:30:46.308 [main] INFO com.github.semak.commons.model.CryptTestcase - >>> Decrypted result: DemoUser[id=<null>,username=George Zold,age=36,sex=M]
17:30:46.324 [main] INFO com.github.semak.commons.model.CryptTestcase - >>> Real field (username): CPF1ojl2JZnSgMY/5UQ8Uw==
17:30:46.326 [main] INFO com.github.semak.commons.model.CryptTestcase - >>> Decrypted field (username): George Zold
```


- 我们可以看到`username`已经加密成的密文，而原对象的`username`依然为明文，互不影响。
- 和加密类似，获取解密对象时，通过`demoUser.getDecryptObject`方法即可。

---

**基于String泛型的集合类型的加解密**


我们先为`DemoUser`类的`memo`字段添加`@Crypt`注解。


```java
@AllArgsConstructor
@NoArgsConstructor
@Builder
@Getter
@Setter
public class DemoUser extends BasePo {

    private BigDecimal id;

    private String username;

    private Integer age;

    private String sex;

    @Crypt
    private List<String> memo;
}
```


构造`DemoUser`，当需要获取加密对象时，使用`demoUser.getEncryptObject`获取原对象的加密对象，加密对象深度克隆自原对象，数据间不会有串扰。


```java

/** 加密 **/

List<String> memos = new ArrayList(2){{
    add("1234567890备忘录");
    add("ABCdefg提醒事项");
}};


DemoUser demoUser = DemoUser.builder()
        .username("George Zold")
        .age(36)
        .sex("M")
        .memo(memos)
        .build();
        
log.info(">>> Encrypted result: {}", demoUser.getEncryptObject(DemoUser.class));
log.info(">>> Real field (memo): {}", demoUser.getMemo());
log.info(">>> Encrypted field (memo): {}", demoUser.getEncryptObject(DemoUser.class).getMemo());

//-------------------------------------------------------------------------------------------------------------------------------------
                
/** 解密 **/

List<String> memos = new ArrayList(2){{
    add("GHFpK5Ak9LFBhicNalpz6yLflyz8CHbStRzd9K6yE10=");
    add("91GK6n0Xrjl8kEUAeUj0igNseTAnnGtBNjZEiHBJ6vI=");
}};

DemoUser demoUser = DemoUser.builder()
        .username("CPF1ojl2JZnSgMY/5UQ8Uw==")
        .age(36)
        .sex("M")
        .memo(memos)
        .build();

log.info(">>> Decrypted result: {}", demoUser.getDecryptObject(DemoUser.class));
log.info(">>> Real field (memo): {}", demoUser.getMemo());
log.info(">>> Encrypted field (memo): {}", demoUser.getDecryptObject(DemoUser.class).getMemo());
```


日志输出


```
17:29:54.382 [main] INFO com.github.semak.commons.model.CryptTestcase - >>> Encrypted result: DemoUser[id=<null>,username=George Zold,age=36,sex=M,memo=[GHFpK5Ak9LFBhicNalpz6yLflyz8CHbStRzd9K6yE10=, 91GK6n0Xrjl8kEUAeUj0igNseTAnnGtBNjZEiHBJ6vI=]]
17:29:54.403 [main] INFO com.github.semak.commons.model.CryptTestcase - >>> Real field (memo): [1234567890备忘录, ABCdefg提醒事项]
17:29:54.415 [main] INFO com.github.semak.commons.model.CryptTestcase - >>> Encrypted field (memo): [GHFpK5Ak9LFBhicNalpz6yLflyz8CHbStRzd9K6yE10=, 91GK6n0Xrjl8kEUAeUj0igNseTAnnGtBNjZEiHBJ6vI=]
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
17:30:46.308 [main] INFO com.github.semak.commons.model.CryptTestcase - >>> Decrypted result: DemoUser[id=<null>,username=George Zold,age=36,sex=M,memo=[1234567890备忘录, ABCdefg提醒事项]]
17:30:46.324 [main] INFO com.github.semak.commons.model.CryptTestcase - >>> Real field (memo): [GHFpK5Ak9LFBhicNalpz6yLflyz8CHbStRzd9K6yE10=, 91GK6n0Xrjl8kEUAeUj0igNseTAnnGtBNjZEiHBJ6vI=]
17:30:46.335 [main] INFO com.github.semak.commons.model.CryptTestcase - >>> Decrypted field (memo): [1234567890备忘录, ABCdefg提醒事项]
```


- 我们可以看到`memo`集合中的String对象都已经加密成的密文，而原对象的`memo`集合中的String对象依然为明文，互不影响。
- 和加密类似，获取解密对象时，通过`demoUser.getDecryptObject`方法即可。



#### 4.2.2. 嵌套方式


**Model类型的加解密**


我们先为`DemoUserInfo`类的`bankcard`和`idcard`字段添加`@Crypt`注解。


```java
@AllArgsConstructor
@NoArgsConstructor
@Builder
@Getter
@Setter
public class DemoUserInfo extends BasePo{

    @Crypt
    private String bankcard;

    @Crypt
    private String idcard;

    private String mobile;
}
```


然后，我们再为`DemoUser`类的`demoUserInfo`字段添加`@NestedCrypt`注解，标记为内嵌加密类。


```java
@AllArgsConstructor
@NoArgsConstructor
@Builder
@Getter
@Setter
public class DemoUser extends BasePo {

    private BigDecimal id;

    private String username;

    @NestedCrypt
    private DemoUserInfo demoUserInfo;
}
```


分别构造`DemoUserInfo`和`DemoUser`，当需要获取加密对象时，使用`demoUser.getEncryptObject`获取原对象的加密对象，加密对象深度克隆自原对象，数据间不会有串扰。


```java

/** 加密 **/

DemoUserInfo demoUserInfo = DemoUserInfo.builder()
        .bankcard("4425178934561222")
        .idcard("310101198011112345")
        .mobile("13577898098")
        .build();

DemoUser demoUser = DemoUser.builder()
        .username("George Zold")
        .demoUserInfo(demoUserInfo)
        .build();


log.info(">>> Encrypted result: {}", demoUser.getEncryptObject(DemoUser.class));
log.info(">>> Real field (bankcard): {}", demoUser.getDemoUserInfo().getBankcard());
log.info(">>> Real field (idcard): {}", demoUser.getDemoUserInfo().getIdcard());
log.info(">>> Encrypted field (bankcard): {}", demoUser.getEncryptObject(DemoUser.class).getDemoUserInfo().getBankcard());
log.info(">>> Encrypted field (idcard): {}", demoUser.getEncryptObject(DemoUser.class).getDemoUserInfo().getIdcard());

//-------------------------------------------------------------------------------------------------------------------------------------
                
/** 解密 **/

DemoUserInfo demoUserInfo = DemoUserInfo.builder()
        .bankcard("5oklv3sTHbXCOT5yo7LxhEnRSv+ciLzNS70L25rcplI=")
        .idcard("l++iuA0q2nrmBbHA3xQ7CFFCvSP6M+m20wM+pZSDFeE=")
        .mobile("13577898098")
        .build();

DemoUser demoUser = DemoUser.builder()
        .username("George Zold")
        .demoUserInfo(demoUserInfo)
        .build();


log.info(">>> Decrypted result: {}", demoUser.getDecryptObject(DemoUser.class));
log.info(">>> Real field (bankcard): {}", demoUser.getDemoUserInfo().getBankcard());
log.info(">>> Real field (idcard): {}", demoUser.getDemoUserInfo().getIdcard());
log.info(">>> Decrypted field (bankcard): {}", demoUser.getDecryptObject(DemoUser.class).getDemoUserInfo().getBankcard());
log.info(">>> Decrypted field (idcard): {}", demoUser.getDecryptObject(DemoUser.class).getDemoUserInfo().getIdcard());
```


日志输出


```
17:56:35.015 [main] INFO com.github.semak.commons.model.CryptTestcase - >>> Encrypted result: DemoUser[id=<null>,username=George Zold,demoUserInfo=DemoUserInfo[bankcard=5oklv3sTHbXCOT5yo7LxhEnRSv+ciLzNS70L25rcplI=,idcard=l++iuA0q2nrmBbHA3xQ7CFFCvSP6M+m20wM+pZSDFeE=,mobile=13577898098]]
17:56:35.034 [main] INFO com.github.semak.commons.model.CryptTestcase - >>> Real field (bankcard): 4425178934561222
17:56:35.034 [main] INFO com.github.semak.commons.model.CryptTestcase - >>> Real field (idcard): 310101198011112345
17:56:35.038 [main] INFO com.github.semak.commons.model.CryptTestcase - >>> Encrypted field (bankcard): 5oklv3sTHbXCOT5yo7LxhEnRSv+ciLzNS70L25rcplI=
17:56:35.040 [main] INFO com.github.semak.commons.model.CryptTestcase - >>> Encrypted field (idcard): l++iuA0q2nrmBbHA3xQ7CFFCvSP6M+m20wM+pZSDFeE=
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
17:57:01.229 [main] INFO com.github.semak.commons.model.CryptTestcase - >>> Decrypted result: DemoUser[id=<null>,username=George Zold,demoUserInfo=DemoUserInfo[bankcard=4425178934561222,idcard=310101198011112345,mobile=13577898098]]
17:57:01.246 [main] INFO com.github.semak.commons.model.CryptTestcase - >>> Real field (bankcard): 5oklv3sTHbXCOT5yo7LxhEnRSv+ciLzNS70L25rcplI=
17:57:01.246 [main] INFO com.github.semak.commons.model.CryptTestcase - >>> Real field (idcard): l++iuA0q2nrmBbHA3xQ7CFFCvSP6M+m20wM+pZSDFeE=
17:57:01.254 [main] INFO com.github.semak.commons.model.CryptTestcase - >>> Decrypted field (bankcard): 4425178934561222
17:57:01.257 [main] INFO com.github.semak.commons.model.CryptTestcase - >>> Decrypted field (idcard): 310101198011112345
```


- 我们可以看到内嵌对象`demoUserInfo`中的`bankcard`和`idcard`都已经加密成的密文，而原对象的内嵌对象`demoUserInfo`中的`bankcard`和`idcard`依然为明文，互不影响。
- 和加密类似，获取解密对象时，通过`demoUser.getDecryptObject`方法即可。

---



**基于Model泛型的集合类型的加解密**


我们先为`DemoAddr`类的`detail`字段添加`@Crypt`注解。


```java
@AllArgsConstructor
@NoArgsConstructor
@Builder
@Getter
@Setter
public class DemoAddr extends BasePo {

    @Crypt
    private String detail;

    private String zipcode;
}
```


然后，我们再为`DemoUser`类的`demoAddrs`字段添加`@NestedCrypt`注解，标记为内嵌加密集合类。


```java
@AllArgsConstructor
@NoArgsConstructor
@Builder
@Getter
@Setter
public class DemoUser extends BasePo {

    private BigDecimal id;

    private String username;

    @NestedCrypt
    private List<DemoAddr> demoAddrs;
}
```


分别构造`DemoAddr`和`DemoUser`，当需要获取加密对象时，使用`demoUser.getEncryptObject`获取原对象的加密对象，加密对象深度克隆自原对象，数据间不会有串扰。


```java

/** 加密 **/

List<DemoAddr> demoAddrs = new ArrayList<>(2);

demoAddrs.add(DemoAddr.builder()
        .zipcode("200135")
        .detail("No.1225 Hunan Road Pudong")
        .build());

demoAddrs.add(DemoAddr.builder()
        .zipcode("232111")
        .detail("No.1111 Zhaojiabang Road Xuhui")
        .build());

DemoUser demoUser = DemoUser.builder()
        .username("George Zold")
        .demoAddrs(demoAddrs)
        .build();

log.info(">>> Encrypted result: {}", demoUser.getEncryptObject(DemoUser.class));
log.info(">>> Real field (addr): {}", demoUser.getDemoAddrs());
log.info(">>> Encrypted field (addr): {}", demoUser.getEncryptObject(DemoUser.class).getDemoAddrs());

//-------------------------------------------------------------------------------------------------------------------------------------
                
/** 解密 **/

List<DemoAddr> demoAddrs = new ArrayList<>(2);

demoAddrs.add(DemoAddr.builder()
        .zipcode("200135")
        .detail("uj8gIIvReesDPnO2K9v9LaQ5TUH1/Pryobiu6SGvZDI=")
        .build());

demoAddrs.add(DemoAddr.builder()
        .zipcode("232111")
        .detail("CD7JXXH/FQEon0svqr7dLnR8wXs3wIDXXBkUTmRNJLA=")
        .build());

DemoUser demoUser = DemoUser.builder()
        .username("George Zold")
        .demoAddrs(demoAddrs)
        .build();

log.info(">>> Decrypted result: {}", demoUser.getDecryptObject(DemoUser.class));
log.info(">>> Real field (addr): {}", demoUser.getDemoAddrs());
log.info(">>> Decrypted field (addr): {}", demoUser.getDecryptObject(DemoUser.class).getDemoAddrs());
```


日志输出


```
18:12:25.192 [main] INFO com.github.semak.commons.model.CryptTestcase - >>> Encrypted result: DemoUser[id=<null>,username=George Zold,demoAddrs=[DemoAddr[detail=uj8gIIvReesDPnO2K9v9LaQ5TUH1/Pryobiu6SGvZDI=,zipcode=200135], DemoAddr[detail=CD7JXXH/FQEon0svqr7dLnR8wXs3wIDXXBkUTmRNJLA=,zipcode=232111]]]
18:12:25.209 [main] INFO com.github.semak.commons.model.CryptTestcase - >>> Real field (addr): [DemoAddr[detail=No.1225 Hunan Road Pudong,zipcode=200135], DemoAddr[detail=No.1111 Zhaojiabang Road Xuhui,zipcode=232111]]
18:12:25.220 [main] INFO com.github.semak.commons.model.CryptTestcase - >>> Encrypted field (addr): [DemoAddr[detail=uj8gIIvReesDPnO2K9v9LaQ5TUH1/Pryobiu6SGvZDI=,zipcode=200135], DemoAddr[detail=CD7JXXH/FQEon0svqr7dLnR8wXs3wIDXXBkUTmRNJLA=,zipcode=232111]]
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
18:13:09.408 [main] INFO com.github.semak.commons.model.CryptTestcase - >>> Decrypted result: DemoUser[id=<null>,username=George Zold,demoAddrs=[DemoAddr[detail=No.1225 Hunan Road Pudong,zipcode=200135], DemoAddr[detail=No.1111 Zhaojiabang Road Xuhui,zipcode=232111]]]
18:13:09.426 [main] INFO com.github.semak.commons.model.CryptTestcase - >>> Real field (addr): [DemoAddr[detail=uj8gIIvReesDPnO2K9v9LaQ5TUH1/Pryobiu6SGvZDI=,zipcode=200135], DemoAddr[detail=CD7JXXH/FQEon0svqr7dLnR8wXs3wIDXXBkUTmRNJLA=,zipcode=232111]]
18:13:09.436 [main] INFO com.github.semak.commons.model.CryptTestcase - >>> Decrypted field (addr): [DemoAddr[detail=No.1225 Hunan Road Pudong,zipcode=200135], DemoAddr[detail=No.1111 Zhaojiabang Road Xuhui,zipcode=232111]]
```


- 我们可以看到内嵌集合对象`demoAddrs`中的`detail`已经加密成的密文，而原对象的内嵌集合对象`demoAddrs`中的`detail`依然为明文，互不影响。
- 和加密类似，获取解密对象时，通过`demoUser.getDecryptObject`方法即可。



#### 4.2.3. 其他方式


在某些情况下，比如方法的返回值返回了个集合（如：`List<DemoUser>`）。这时，我们可能需要另一种处理方式，这时我们可以直接使用工具类`com.github.semak.commons.model.desen.CryptUtil`进行加解密处理。


```java
//加密
CryptUtil.encryptObject(object);
//解密
CryptUtil.decryptObject(object);
```


上述方法支持的入参类型是：


- String
- JavaBean（继承自`com.github.semak.commons.model.BaseModel`及其子类）
- Collection （以及java.util.Collection接口的实现类）



### 4.3. 字段加解密算法扩展


由于组件默认使用`AES128 + BASE64`的方式进行加/解密。如果你有自己的加密方法，组件提供了SPI的接入方式。


你可以实现`com.github.semak.commons.model.spi.CryptFiledAlgorithm`接口的加解/密方法。然后，在`classpath`的`META-INF/services`目录下创建`com.github.semak.commons.model.spi.CryptFiledAlgorithm`文件，并将实现类的全名输入到此文件中即可。
