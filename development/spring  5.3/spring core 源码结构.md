# Spring Core源码结构

## 概述
spring-core是spring框架的基石，它为spring框架提供了基础的支持。  
共有6个模块，分别是 asm、cglib、core、lang、objenesis、util。
```
asm          用来操作字节码，动态生成类或者增强既有类的功能
cglib        代码生成库，用它来实现动态代理，生成字节码文件
core         核心包
lang         注解(NonNull,NonNullApi,Nullable, etc.)
objenesis    用于实例化一些特殊java对象的工具
util         工具类(ClassUtils,ObjectUtils,StringUtils, etc.)
```

## ASM
ASM是一个通用的Java字节码操作工具。它可以直接以二进制形式修改现有类或动态生成类。
在ASM的核心实现中，它主要有以下几个类、接口：  

- ClassReader：字节码的读取与分析引擎
每当有事件发生时，调用注册的ClassVisitor、AnnotationVisitor、FieldVisitor、MethodVisitor。  

- ClassVisitor: 
定义在读取Class字节码时会触发的事件，如类头解析完成、注解解析、字段解析、方法解析等。  

- AnnotationVisitor：
定义在解析注解时会触发的事件，如解析到一个基本值类型的注解、enum值类型的注解、Array值类型的注解、注解值类型的注解等。  

- FieldVisitor：  
定义在解析字段时触发的事件，如解析到字段上的注解、解析到字段相关的属性等。

- MethodVisitor：  
定义在解析方法时触发的事件，如方法上的注解、属性、代码等。  

- ClassWriter：  
它实现了ClassVisitor接口，用于拼接字节码。  

- AnnotationWriter：  
它实现了AnnotationVisitor接口，用于拼接注解相关字节码。  

- FieldWriter：  
它实现了FieldVisitor接口，用于拼接字段相关字节码。  

- MethodWriter：  
它实现了MethodVisitor接口，用于拼接方法相关字节码。  

- SignatureReader：  
对类定义、字段定义、方法定义、本地变量定义的签名的解析。Signature因范型引入，用于存储范型定义时的元数据（因为这些元数据在运行时会被擦除）。  

- SignatureVisitor：  
定义在解析Signature时会触发的事件，如正常的Type参数、类或接口的边界等。  

- SignatureWriter：实现了SignatureVisitor接口，用于拼接范型相关字节码。
- Attribute：字节码中属性的类抽象
- ByteVector：字节码二进制存储的容器
- Opcodes：字节码指令的一些常量定义
- Type：类型相关的常量定义以及一些基于其上的操作

## core核心包：
annotation目录：注解、元注解、合并的注解等  

- AliasFor：别名，通常是互相别名
- AnnotationsProcessor：处理注解以后的回调
- Order：注解用于排序
- OrderUtils：Spring提供了OrderUtils来获取Class的Order注解排序信息  

codec目录：encode和decode输入流  

convert目录：主要是转换器服务，将一个类型转换位另外一个类型。  
定义了很多接口，有ConversionService、Converter、ConverterFactory、ConverterRegistry、GenericConverter，就是用来给给框架或者用户去实现转换功能的。  

env目录：  
环境目录，主要是profile，Environment，AbstractEnvironment，StandardEnvironment  

io目录：一些读取资源的类  
主要是输入流接口，抽象资源类以及子类。另外还有默认的资源加载器，子类文件系统资源加载器、类相对路径加载器。  

log目录：日志类  
日志处理目录，委托给Apache Commons log处理   

serializer目录：序列化、反序列化类   
序列化和反序列化接口Serializer，Deserializer，以及对应的默认实现DefaultSerializer，DefaultDeserializer，默认采用JDK序列化机制，ObjectOutputStream和ObjectInputStream。    

style目录：代码风格  
转为string字符串的风格格式  

task目录：可执行任务类  
任务执行器，继承了JDK的Executor，用来执行Runnable类型的task  

type目录：Class元数据、注解元数据等  
包含被注解的类型的元数据、注解元数据、Class元数据、方法元数据  

其它： 

- AliasRegistry：是个接口，定义对别名的简单增删等操作。
- SimpleAliasRegistry：是AliasRegistry的实现，主要是用了一个map对别名进行缓存。  

在Spring的bean管理容器中，因为bean可以取很多别名。为此，Spring专门提供了一个接口AliasRegistry来实现别名注册的功能。

- AttributeAccessor：对象元数据的访问接口
- AttributeAccessorSupport：是实现AttributeAccessor的抽象类,内部由LinkedHashMap实现

- Ordered：排序功能
- OrderComparator：实现了Comparator的一个比较器  

Spring框架是一个大量使用策略设计模式的框架，这意味着有很多相同接口的实现类，那么必定会有优先级的问题。于是Spring提供了Ordered这个接口，来处理相同接口实现类的优先级问题。

## Objenesis
SpringObjenesis封装了Objenesis，涉及了一个spring封装的数据结构org.springframework.util.ConcurrentReferenceHashMap，可以做缓存使用，不保证命中率。

