# 类文件结构

Java 类文件是包含 Java 字节码的文件，该文件可以在JVM上执行。Java 类文件通常由Java编译器将 Java 编程语言源文件编译生成（其它JVM语言也可以创建类文件）。

## JVM 的“无关性”

谈论 JVM 的无关性，主要有以下两个：

- 平台无关性：任何操作系统都能运行 Java 代码
- 语言无关性： JVM 能运行除 Java 以外的其他代码

> Java 语言中的各种变量、关键字和运算符号的语义最终都是由多条字节码命令组合而成的， 因此字节码命令所能提供的语义描述能力肯定会比 Java 语言本身更加强大。 因此，有一些 Java 语言本身无法有效支持的语言特性，不代表字节码本身无法有效支持。

## Class 文件结构

Class文件中的所有内容被分为两种类型：无符号数、表。

- 无符号数 无符号数表示 Class 文件中的值，这些值没有任何类型，但有不同的长度。u1、u2、u4、u8 分别代表 1/2/4/8 字节的无符号数。
- 表 由多个无符号数或者其他表作为数据项构成的复合数据类型。

Class 文件具体由以下几个构成:

- 魔数（0xCAFEBABE）
- 版本信息
- 常量池
- 访问标志
- 类索引、父类索引、接口索引集合
- 字段表集合
- 方法表集合
- 属性表集合

```
ClassFile {
    u4             magic;
    u2             minor_version;
    u2             major_version;
    u2             constant_pool_count;
    cp_info        constant_pool[constant_pool_count-1];
    u2             access_flags;
    u2             this_class;
    u2             super_class;
    u2             interfaces_count;
    u2             interfaces[interfaces_count];
    u2             fields_count;
    field_info     fields[fields_count];
    u2             methods_count;
    method_info    methods[methods_count];
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```

### 魔数

Class 文件的头 4 个字节称为魔数，用于标识 Class 文件是否符合类文件格式。

### 版本信息

紧接着魔数的 4 个字节是版本信息，5-6 字节表示次版本号，7-8 字节表示主版本号，它们表示当前 Class 文件中使用的是哪个版本的 JDK。

高版本的 JDK 能向下兼容以前版本的 Class 文件，但不能运行以后版本的 Class 文件，即使文件格式并未发生任何变化，虚拟机也必需拒绝执行超过其版本号的 Class 文件。
```java
// ClassFile.java
    public enum Version {
        V45_3(45, 3), // base level for all attributes
        V49(49, 0),   // JDK 1.5: enum, generics, annotations
        V50(50, 0),   // JDK 1.6: stackmaps
        V51(51, 0),   // JDK 1.7
        V52(52, 0),   // JDK 1.8: lambda, type annos, param names
        V53(53, 0),   // JDK 1.9: modules, indy string concat
        V54(54, 0),   // JDK 10
        V55(55, 0),   // JDK 11: constant dynamic, nest mates
        V56(56, 0),   // JDK 12
        V57(57, 0),   // JDK 13
        V58(58, 0),   // JDK 14
        V59(59, 0),   // JDK 15
        V60(60, 0);   // JDK 16
        Version(int major, int minor) {
            this.major = major;
            this.minor = minor;
        }
        public final int major, minor;

        private static final Version MIN = values()[0];
        /** Return the least version supported, MIN */
        public static Version MIN() { return MIN; }

        private static final Version MAX = values()[values().length-1];
        /** Return the largest version supported, MAX */
        public static Version MAX() { return MAX; }
    }
```

### 常量池

版本信息之后就是常量池，常量池中存放两种类型的常量：

- 字面值常量

  字面值常量就是我们在程序中定义的字符串、被 final 修饰的值。

- 符号引用

  符号引用就是我们定义的各种名字：类和接口的全限定名、字段的名字和描述符、方法的名字和描述符。

#### 常量池的特点

- 常量池中常量数量不固定，因此常量池开头放置一个 u2 类型的无符号数，用来存储当前常量池的容量。
- 常量池的每一项常量都是一个表，表开始的第一位是一个 u1 类型的标志位（tag），代表当前这个常量属于哪种常量类型。

#### 常量池中常量类型

| 类型                             | tag | 描述　                 |
| -------------------------------- | --- | ---------------------- |
| CONSTANT_utf8_info               | 1   | UTF-8 编码的字符串     |
| CONSTANT_Integer_info            | 3   | 整型字面量             |
| CONSTANT_Float_info              | 4   | 浮点型字面量           |
| CONSTANT_Long_info               | 5   | 长整型字面量           |
| CONSTANT_Double_info             | 6   | 双精度浮点型字面量     |
| CONSTANT_Class_info              | 7   | 类或接口的符号引用     |
| CONSTANT_String_info             | 8   | 字符串类型字面量       |
| CONSTANT_Fieldref_info           | 9   | 字段的符号引用         |
| CONSTANT_Methodref_info          | 10  | 类中方法的符号引用     |
| CONSTANT_InterfaceMethodref_info | 11  | 接口中方法的符号引用   |
| CONSTANT_NameAndType_info        | 12  | 字段或方法的符号引用   |
| CONSTANT_MethodHandle_info       | 15  | 表示方法句柄           |
| CONSTANT_MethodType_info         | 16  | 标识方法类型           |
| CONSTANT_InvokeDynamic_info      | 18  | 表示一个动态方法调用点 |

对于 CONSTANT_Class_info（此类型的常量代表一个类或者接口的符号引用），它的结构如下：
```
CONSTANT_Class_info {
    u1 tag;
    u2 name_index;
}
```

tag 是标志位，用于区分常量类型；name_index 是一个索引值，它指向常量池中一个 CONSTANT_Utf8_info 类型常量，此常量代表这个类（或接口）的全限定名，这里 name_index 值若为 0x0002，也即是指向了常量池中的第二项常量。

CONSTANT_Utf8_info 型常量的结构如下：
```
CONSTANT_Utf8_info {
    u1 tag;
    u2 length;
    u1 bytes[length];
}
```
tag 是当前常量的类型；length 表示这个字符串的长度；bytes 是这个字符串的内容（采用缩略的 UTF8 编码）

### 访问标志

在常量池结束之后，紧接着的两个字节代表访问标志，这个标志用于识别一些类或者接口层次的访问信息，包括：这个 Class 是类还是接口；是否定义为 public 类型；是否被 abstract/final 修饰。  

| Flag Name | Value | Interpretation |
| --------- | ----- | -------------- |
| ACC_PUBLIC | 0x0001 | Declared public; may be accessed from outside its package. |
| ACC_FINAL | 0x0010 | Declared final; no subclasses allowed. |
| ACC_SUPER | 0x0020 | Treat superclass methods specially when invoked by the invokespecial instruction. |
| ACC_INTERFACE | 0x0200 | Is an interface, not a class. |
| ACC_ABSTRACT | 0x0400 | Declared abstract; must not be instantiated. |
| ACC_SYNTHETIC | 0x1000 | Declared synthetic; not present in the source code. |
| ACC_ANNOTATION | 0x2000 | Declared as an annotation type. |
| ACC_ENUM | 0x4000 | Declared as an enum type. |

### 类索引、父类索引、接口索引集合

类索引和父类索引都是一个 u2 类型的数据，而接口索引集合是一组 u2 类型的数据的集合，Class 文件中由这三项数据来确定类的继承关系。类索引用于确定这个类的全限定名，父类索引用于确定这个类的父类的全限定名。

由于 Java 不允许多重继承，所以父类索引只有一个，除了 java.lang.Object 之外，所有的 Java 类都有父类，因此除了 java.lang.Object 外，所有 Java 类的父类索引都不为 0。一个类可能实现了多个接口，因此用接口索引集合来描述。这个集合第一项为 u2 类型的数据，表示索引表的容量，接下来就是接口的名字索引。

类索引和父类索引用两个 u2 类型的索引值表示，它们各自指向一个类型为 CONSTANT_Class_info 的类描述符常量，通过该常量总的索引值可以找到定义在 CONSTANT_Utf8_info 类型的常量中的全限定名字符串。

### 字段表集合

字段表集合存储本类涉及到的成员变量，包括实例变量和类变量，但不包括方法中的局部变量。

每一个字段表只表示一个成员变量，本类中的所有成员变量构成了字段表集合。字段表结构如下：
```
field_info {
    u2             access_flags; // 字段的访问标志，与类稍有不同
    u2             name_index; // 字段名字的索引，指向常量池中类型为 CONSTANT_Utf8_info 的结构
    u2             descriptor_index; // 描述符，用于描述字段的数据类型，指向常量池中类型为 CONSTANT_Utf8_info 的结构。
    u2             attributes_count; // 属性表集合的长度
    attribute_info attributes[attributes_count]; //属性表集合，用于存放属性的额外信息，如属性的值。属性表的每个值都必须是attribute_info结构
}
```

访问标志：  

| Flag Name | Value | Interpretation |
| --------- | ----- | -------------- |
| ACC_PUBLIC | 0x0001 | Declared public; may be accessed from outside its package. |
| ACC_PRIVATE | 0x0002 | Declared private; usable only within the defining class. |
| ACC_PROTECTED | 0x0004 | Declared protected; may be accessed within subclasses. |
| ACC_STATIC | 0x0008 | Declared static. |
| ACC_FINAL | 0x0010 | Declared final; never directly assigned to after object construction (JLS 17.5). |
| ACC_VOLATILE | 0x0040 | Declared volatile; cannot be cached. |
| ACC_TRANSIENT | 0x0080 | Declared transient; not written or read by a persistent object manager. |
| ACC_SYNTHETIC | 0x1000 | Declared synthetic; not present in the source code. |
| ACC_ENUM | 0x4000 | Declared as an element of an enum. |

> 字段表集合中不会出现从父类（或接口）中继承而来的字段，但有可能出现原本 Java 代码中不存在的字段，譬如在内部类中为了保持对外部类的访问性，会自动添加指向外部类实例的字段。

### 方法表集合

方法表结构与属性表类似。
```
method_info {
    u2             access_flags;
    u2             name_index;
    u2             descriptor_index;
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```

访问标志：  

| Flag Name | Value | Interpretation |
| --------- | ----- | -------------- |
| ACC_PUBLIC |0x0001 | Declared public; may be accessed from outside its package. |
| ACC_PRIVATE |0x0002 | Declared private; accessible only within the defining class. |
| ACC_PROTECTED |0x0004 | Declared protected; may be accessed within subclasses. |
| ACC_STATIC | 0x0008 | Declared static. |
| ACC_FINAL |0x0010 | Declared final; must not be overridden. |
| ACC_SYNCHRONIZED | 0x0020 | Declared synchronized; invocation is wrapped by a monitor use. |
| ACC_BRIDGE | 0x0040 | A bridge method, generated by the compiler. |
| ACC_VARARGS | 0x0080 | Declared with variable number of arguments. |
| ACC_NATIVE | 0x0100 | Declared native; implemented in a language other than Java. |
| ACC_ABSTRACT | 0x0400 | Declared abstract; no implementation is provided. |
| ACC_STRICT | 0x0800 | Declared strictfp; floating-point mode is FP-strict. |
| ACC_SYNTHETIC | 0x1000 | Declared synthetic; not present in the source code. |

volatile 关键字 和 transient 关键字不能修饰方法，所以方法表的访问标志中没有 ACC_VOLATILE 和 ACC_TRANSIENT 标志。

方法表的属性表集合中有一张 Code 属性表，用于存储当前方法经编译器编译后的字节码指令。

### 属性表集合

每个属性对应一张属性表，属性表的结构如下：
```
attribute_info {
    u2 attribute_name_index;  // 指向常量池中类型为 CONSTANT_Utf8_info 的结构
    u4 attribute_length;
    u1 info[attribute_length];
}
```

