# 类文件结构
Java能成为“Write Once, Run Anywhere”的语言，是基于 **统一的程序存储格式 + 适应不同平台的JVM** 所构成的基石完成。通过编译，Java代码被编译为字节码，也就是“.class 文件”，字节码适用于不同平台的虚拟机，具有统一的程序存储格式。

而JVM并非只用于Java语言，它与字节码交互，所有能按格式将代码转换为字节码的语言都可以在JVM上使用。

## Class类文件的结构
Class文件是一组以8字节为基础单位的二进制流，其中只有两种数据类型： **无符号数和表** 。

无符号数属于基本的数据类型，以u1、u2、u4、u8来分别代表1个字节、2个字节、4个字节和8个字节的无符号数，可以用于描述数字、索引引用、数量值或者按照UTF-8编码构成字符串值。

表是一种复合的数据类型，由多个无符号数或者其他表作为数据项构成，所有表都以“\_info”结尾。

整个Class文件的结构按如下顺序显示：

| 类型           | 名称                | 数量                   | 描述               |
| -------------- | ------------------- | ---------------------- | ------------------ |
| u4             | magic               | 1                      | 魔数，值为CAFEBABE |
| u2             | minor_version       | 1                      | 次版本号           |
| u2             | major_version       | 1                      | 主版本号           |
| u2             | constant_pool_count | 1                      | 常量池中元素数     |
| cp_info        | constant_pool       | constant_pool_count -1 | 常量池信息         |
| u2             | access_flags        | 1                      | 访问标志           |
| u2             | this_class          | 1                      | 类索引             |
| u2             | super_class         | 1                      | 父类索引           |
| u2             | interfaces_count    | 1                      | 接口数             |
| u2             | interfaces          | interfaces_count       | 接口信息           |
| u2             | fields_count        | 1                      | 字段数             |
| field_info     | fields              | fields_count           | 字段信息           |
| u2             | methods_count       | 1                      | 方法数             |
| method_info    | methods             | methods_count          | 方法信息           |
| u2             | attributes_count    | 1                      | 属性数             |
| attribute_info | attributes          | attributes_count       | 属性信息           |

![Java类文件结构](http://assets.processon.com/chart_image/5df83bcae4b0a9c790ef72bf.png?_=1576549850536)

### 魔数与版本号
开头即魔数与版本号，魔数并没有特殊实际意义，它的唯一作用就是确定此文件是否能被虚拟机接受的文件。魔数的后4位为版本号，次版本号在主版本号之前，用于标志Java版本。高版本的JDK可兼容低版本的Class文件，反之则不行，即使未改变文件格式与内容，虚拟机也会拒绝执行超过其版本的Class文件。

这两部分信息是虚拟机解析Class文件时首要校验的内容。

> Class文件是二进制文流，为何u4的魔数值可为“CAFEBABE”呢？
>
> 一个字节为8位，可将8位再继续拆为两个4位，通过4位表达的数字大小区间为 [0,15]，正好可用一个16进制的数表示。所以一个字节可表示为两位的16进制数，“CA FE BA BE”中的第一个字节“CA”对应2进制的1100 1010，其他字符以此类推。

### 常量池
接在版本号后的信息为常量池。常量池中存储着两大类常量：字面量和符号引用。

字面量比较接近于Java语言层面的常量概念，在Java语言里， `String s = "Eco"` 中的“Eco”即字面量。

符号引用属于编译原理方面的概念，包括以下3类常量：
- 类和接口的全限定名
- 字段的名称和描述符
- 方法的名称和描述符

常量池中的每一项都是一个表，在Java7种，常量池分为14种表结构，它们结构不同，但都有一个共同点，表的第一位是一个u1（两位16进制数）类型的标志位，用于标志该常量属于哪种表。

常量池的项目类型：

| 类型                             | 标志 | 描述                     |
| -------------------------------- | ---- | ------------------------ |
| CONSTANT_Utf8_info               | 1    | UTF-8编码的字符串        |
| CONSTANT_Integer_info            | 3    | 整形字面量               |
| CONSTANT_Float_info              | 4    | 浮点型字面量             |
| CONSTANT_Long_info               | 5    | 长整型字面量             |
| CONSTANT_Double_info             | 6    | 双精度浮点型字面量       |
| CONSTANT_Class_info              | 7    | 类或接口的符号引用       |
| CONSTANT_String_info             | 8    | 字符串类型字面量         |
| CONSTANT_Fieldref_info           | 9    | 字段的符号引用           |
| CONSTANT_Methodref_info          | 10   | 类中方法的符号引用       |
| CONSTANT_InterfaceMethodref_info | 11   | 接口中方法的符号引用     |
| CONSTANT_NameAndType_info        | 12   | 字段或方法的部分符号引用 |
| CONSTANT_MethodHandle_info       | 15   | 表示方法句柄             |
| CONSTANT_MethodType_info         | 16   | 标识方法类型             |
| CONSTANT_InvokeDynamic_info      | 18   | 表示一个动态方法调用点   |

> 我们可以通过简单的命令方便的查看一个类的常量池内容。
>
> 将Java文件编译成Class文件后，进入该Class文件目录，通过 javap -verbose [类名] 即可查看到其常量池中的信息（为输出内容中的“Constant pool”部分），不仅如此，Class文件中还有许多其他信息会展示出来，包括前面提到的版本号、以及后面要提到的访问标志、字节码指令等。
>
> PS：接口类也会被编译成Class文件。

### 访问标志
在常量池之后，是u2的访问标志，用于识别一些类或者接口层次的访问信息。通过访问标志，我们可以确定一个Class文件是类还是接口；是否为抽象类；是否被public修饰；是否被final修饰等。

| 标志名称       | 标志值 | 含义                                                                                                                                                                                                   |
| -------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| ACC_PUBLIC     | 0x0001 | 是否为public类型                                                                                                                                                                                       |
| ACC_FINAL      | 0x0010 | 是否被声明为final，只有类可设置                                                                                                                                                                        |
| ACC_SUPER      | 0x0020 | 是否允许使用invokespecial字节码指令的新语意，invokespecial是JVM支持的一种指令，在JDK1.0.2时该语意发生过改变，为了区分在调用invokespecial时该指令表达的语意，在JDK1.0.2后编译出来的类这个标志都必须为真 |
| ACC_INTERFACE  | 0x0200 | 标志这是一个接口                                                                                                                                                                                       |
| ACC_ABSTRACT   | 0x0400 | 是否为abstract类型， **接口和抽象类该标志都为真**                                                                                                                                                      |
| ACC_SYNTHETIC  | 0x1000 | 标识这个类并非由用户代码产生的                                                                                                                                                                         |
| ACC_ANNOTATION | 0x2000 | 标识这是一个注解                                                                                                                                                                                       |
| ACC_ENUM       | 0x4000 | 标识这是一个枚举                                                                                                                                                                                       |

> PS:以0x开头表示16进制，u2的内容需要通过4位16进制数表示。

### 类索引、父类索引与接口索引集合
类索引（this_class）和父类索引（super_class）都是一个u2类型的数据，而接口索引集合（interfaces）是一组u2类型的数据集合。通过这3项数据可确定类的继承关系。

类索引用于确定这个类的全限定名，例如常用的集合工具类ArrayList的全限定名为：java.util.ArrayList。不过它在编译为Class文件后会将“.”替换为“/”，变成“java/util/ArrayList”。

父类索引用于确定这个类的父类的全限定名。在Java中，java.lang.Object是所有类的祖类，即使一个类没有显式的继承任何类，它的父类会指定为java.lang.Object，除了java.lang.Objcet外，所有类的父索引都不为0。

接口索引集合用于描述这个类实现的接口。一个类可以实现多个接口，因此接口索引为一个集合。不仅如此，Java中的接口是可以多继承的，例如 `InterfaceA extends InterfaceB, InterfaceC`是合法的，该继承关系也通过接口索引集合表述，顺序从左到右排列在接口索引集合中。

### 字段表集合与方法表集合
字段表与方法表的结构以及细节都十分相似。

字段表用于描述接口或类中声明的变量，其中的字段包括类级变量（由static修饰）以及实例级变量（非static修饰）。Class文件仅描述类相关信息，不会包括方法内声明的局部变量。

方法表用于描述接口或类中声明的方法。方法中的代码经过编译后，会存放在方法的属性表集合中一个名为“Code”的属性里。

字段表和方法表具有相同的结构：

| 类型 | 名称             | 数量             | 描述                                                       |
| ---- | ---------------- | ---------------- | ---------------------------------------------------------- |
| u2   | access_flags     | 1                | 字段/方法访问标志，用于描述字段/方法的作用域、可变性、并发可见性等信息 |
| u2   | name_index       | 1                | 字段/方法的简单名称                                               |
| u2   | descriptor_index | 1                | 字段/方法的描述符                                                 |
| u2   | attributes_count | 1                | 额外属性数，可为0                                                 |
| u2   | attributes       | attributes_count | 额外属性信息                                                           |

其中字段的访问标志有如下几种：

| 标志名称      | 标志值 | 含义                       |
| ------------- | ------ | -------------------------- |
| ACC_PUBLIC    | 0x0001 | 字段是否public             |
| ACC_PRIVATE   | 0x0002 | 字段是否private            |
| ACC_PROTECTED | 0x0004 | 字段是否protected          |
| ACC_STATIC    | 0x0008 | 字段是否static             |
| ACC_FINAL     | 0x0010 | 字段是否final              |
| ACC_VOLATILE  | 0x0040 | 字段是否volatile           |
| ACC_TRANSIENT | 0x0080 | 字段是否transient          |
| ACC_SYNTHETIC | 0x1000 | 字段是否由编译器自动产生的 |
| ACC_ENUM      | 0x4000 | 字段是否enum               |

而方法的访问标志有如下几种：

| 标志名称         | 标志值 | 含义                           |
| ---------------- | ------ | ------------------------------ |
| ACC_PUBLIC       | 0x0001 | 方法是否public                 |
| ACC_PRIVATE      | 0x0002 | 方法是否private                |
| ACC_PROTECTED    | 0x0004 | 方法是否protected              |
| ACC_STATIC       | 0x0008 | 方法是否static                 |
| ACC_FINAL        | 0x0010 | 方法是否final                  |
| ACC_SYNCHRONIZED | 0x0020 | 方法是否synchronized           |
| ACC_BRIDGE       | 0x0040 | 方法是否由编译器产生的桥接方法 |
| ACC_VARARGS      | 0x0080 | 方法是否接受 不定参数          |
| ACC_NATIVE       | 0x0100 | 方法是否为native               |
| ACC_ABSTRACT     | 0x0400 | 方法是否为abstract             |
| ACC_STRICTFP     | 0x0800 | 方法是否为strictfp             |
| ACC_SYNTHETIC    | 0x1000 | 方法是否由编译器自动产生的 |


简单名称表示没有类型和参数修饰的方法或字段名称，在下面的样例类中：
```
public class Demo {
  public static int num = 123;

  public int getNum() {
    return num;
  }
}
```
字段 `num` 和方法 `getNum()` 的简单名称分别为“num”和“getNum”。

描述符用于描述字段/方法的数据类型，在方法表中还包括方法的参数列表（包括数量、类型以及顺序）和返回值，与字段名/方法名无关。

Java基本数据类型有8种，再加上无返回值的void类型，以及对象类型一共有10种描述符的标识字符：

| 描述字符 | 含义                           |
| -------- | ------------------------------ |
| B        | 基本类型byte                   |
| C        | 基本类型char                   |
| D        | 基本类型double                 |
| F        | 基本类型float                  |
| I        | 基本类型int                    |
| J        | 基本类型long                   |
| S        | 基本类型short                  |
| Z        | 基本类型boolean                |
| V        | 特殊类型void                   |
| L        | 对象类型，如 Ljava/lang/Object |

而对于数组类型，每一个维度使用一个前置“\[”字符标识，例如定义byte[][]，会被记录为“[[B”。

用描述符描述方法时，需按方法定义的字段类型以及顺序，将参数列表放到一组小括号“()”内，如果方法没有传参，例如ArrayList的方法 `void trimToSize()` ，会被记录为“()V”。

如果有传参数，例如 :
```
int indexOf(char[] source, int sourceOffset, int sourceCount,
            char[] target, int targetOffset, int targetCount,
            int fromIndex){...}
```

则会被记录为：“([CII[CIII)I”。

通过访问标志、简单名称以及描述符即可确定一个字段或方法的原代码定义：

![字段表集合举例](http://assets.processon.com/chart_image/5df8778ee4b06f5f145f92d0.png?_=1576565409903)

值得一提的是，如果父类方法在子类中没有被重写（Override），方法表集合中就不会出现父类的方法信息。并且可能出现编译器添加的自动方法，如类构造器“\<client>”方法以及实例构造器“\<init>”方法。

> 关于Java中的重载（Overload）
>
> 在Java中，重载方法要求具有相同的简单名称，以及不同的特征签名（在Java中包括参数类型以及参数顺序，不包括返回值，因此具有相同方法、相同参数类型、相同参数顺序、 **不相同返回值** 的方法不能称为重载，也无法通过编译）。但在Class文件格式中，特征签名的范围更大，意味着重载要求更低，只要描述符不完全一致即可共存。

### 属性表集合
属性表在Class中的扩展性十分高，在Class文件、字段表、方法表都可以携带自己的属性表集合。属性表内容丰富（光虚拟机规范预定义的属性就有20种），下面选择性的挑几个比较接近日常开发的内容进行说明。

#### Code
在前面提到的方法表集合中，通过访问标志、简单名称以及描述符可以确定一个方法的定义，但是方法的具体内容呢？方法的具体内容经过编译后会变成字节码存储在Code属性中，附加在方法表的attributes_count以及attributes中。

Code属性表的结构如下：

| 类型           | 名称                   | 数量                   | 描述                                     |
| -------------- | ---------------------- | ---------------------- | ---------------------------------------- |
| u2             | attribute_name_index   | 1                      | 指向常量池的索引，它代表该属性的属性名称 |
| u4             | attribute_length       | 1                      | 属性值的长度                             |
| u2             | max_stack              | 1                      | 操作数栈的最大深度                       |
| u2             | max_locals             | 1                      | 局部变量表所需的存储空间                 |
| u4             | code_length            | 1                      | 经过编译后字节码指令的长度                   |
| u1             | code                   | code_length            | 具体字节码指令                           |
| u2             | exception_table_length | 1                      | 异常表的长度                             |
| exception_info | exception_table        | exception_table_length | 异常信息                                 |
| u2             | attributes_count       | 1                      | 其他属性数                               |
| atrribute_info | attributes             | attributes_count       | 其他属性信息                             |

max_stack代表了操作数栈深度的最大值，在方法执行时，操作数栈都不会超过此深度。虚拟机运行的时候需根据此值分配栈帧中的操作栈深度。

> PS：在虚拟机内存模块划分的虚拟机栈中，每个方法的栈帧中包括局部变量表、操作数栈、动态链接、方法出口等

max_locals代表了局部变量表所需的存储空间。Slot是虚拟机为局部变量分配内存所使用的最小单位。数据类型不超过32位的局部变量会占用1个Slot，而double和long是64位的基本类型，需要使用两个Slot存放。局部变量表中，Slot可以重用，当代码执行到超出该局部变量的作用域时，该变量占用的Slot可以被其他局部变量使用。

code_length代表字节码指令的长度，单位为u4。理论上说字节码长度的上限应该是 2^32-1，但虚拟机规范限制一个方法不允许超过 2^16-1 条字节码指令。这里提到了字节码指令，字节码指令通过一个u1类型的单字节描述，取值范围为[0,255]，可以表达256条指令，而目前JVM规范已经定义了其中约200条编码之对应的指令含义，前面提到的 `invokespecial` 也是其中的一条指令。我们将Java代码（“.java”文件）编译为字节码（“.class”文件）后，可以通过命令 `javap -verbose Demo` 查看其字节码指令内容。

#### ConstantValue
ConstantValue属性的作用是通知虚拟机自动为静态变量赋值。只有类变量（被static修饰）才可使用这项属性。

代码 `int x = 1` 和 `static int x = 1`的区别不仅仅是表面的 static 修饰符。它们在虚拟机中的赋值方式以及时机都不相同。对于前者（实例变量），赋值操作是在实例构造器 `\<init>` 方法中进行的；而给后者（类变量）赋值的方式有两种：在类构造器 `\<clinit>` 方法中，或者使用 ConstantValue 属性。目前Sum Javac编译器的做法是： **同时** 满足以下几个条件的变量会生成ConstantValue属性来初始化，否则使用类构造器 `\<clinit>` 方法进行初始化：
  1. 使用final修饰
  2. 使用static修饰
  3. 数据类型为基本类型或 java.lang.String

## 字节码指令简介
字节码为二进制文件，直接阅读十分暴力生涩（全是二进制的001001...），而通过现成的工具（如上文提到的 `javap` ）可帮助我们将其转化为更易理解的表述方式。

无论我们创建的Java项目逻辑多复杂，内容多丰富，方法中的代码最终都会被转换成一个字节码指令集，其中每一个指令的长度为一个字节，因此指令的种类总数上限为256。

了解或掌握不同指令的大致含义对我们来说还是有很大帮助。200多条指令中，也有许多含义相近的指令，例如 `iload` 和 `fload` 指令，分别用于从局部变量表中加载 int 和 float类型的数据。

数据类型与字节码指令中特殊字符的对应关系如下：

| 数据类型  | 字节码指令中的特殊字符 |
| --------- | ---------------------- |
| int       | i                      |
| long      | l                      |
| short     | s                      |
| byte      | b                      |
| char      | c                      |
| float     | f                      |
| double    | d                      |
| reference | a                      |

类似命名的指令还有许多，不仅如此，仔细查阅会发现大部分指令没有针对某些数据类型的版本。因为整个字节码指令的内容只能用一个字节存储。如果针对每一个操作，都定义全量数据类型的版本，一个字节是不满足需求的。针对byte、char、short以及boolean类型，编译器会在编译器或者运行期将其扩展对相应的int类型数据，其中byte和short会带上原有符号（正负），而boolean和char类型则会使用零位扩展。因此大多数对于boolean、byte、short和char类型数据的操作，都是使用相应的int类型作为运算类型。

### 加载和存储指令
- 将一个局部变量加载到操作栈：ilaod、iload_\<n>、lload、lload_\<n>、fload、fload_\<n>、dload、dload_\<n>、aload、aload<n>。
- 将一个数值从操作数栈存储到局部变量表：istore、istore_\<n>、lstore、lstore_\<n>、fstore、fstore_\<n>、dstore、dstore_\<n>、astore、astore_\<n>。
- 将一个常量加载到操作数栈：bipush、sipush、ldc、ldc_w、lwd2_w、aconst_null、iconst_ml、iconst_\<i>、lconst_\<l>、fconst_\<f>、dconst_\<d>

其中类似 iload_\<n> 代表一组指令：iload_0、iload_1、iload_2、iload_3。

### 运算指令
- 加法指令：iadd、ladd、fadd、dadd。
- 减法指令：isub、lsub、fsub、dsub。
- 乘法指令：imul、lmul、fmul、dmul。
- 除法指令：idiv、ldiv、fdiv、ddiv。
- 求余指令：irem、lrem、frem、drem。
- 取反指令：ineg、lneg、fneg、dneg。
- 位移指令：ishl、ishr、iushr、lshl、lshr、lushr。
- 按位或指令：ior、lor。
- 按位与指令：iand、land。
- 按位异或指令：ixor、lxor。
- 局部变量自增指令：iinc。
- 比较指令：dcmpg、dcmpl、fcmpg、fcmpl、lcmp。

### 类型转换指令
JVM直接支持以下安全的宽化类型转换：
- int类型到long、float或者double类型。
- long类型到float、double类型。
- float类型到double类型。

而窄化类型转换需要显式的使用转换命令完成：i2b、i2c、i2s、l2i、f2i、f2l、d2i、d2l和d2f。

窄化类型转换是危险的，可能导致不同的正负号、不同的数量级的情况。例如将int类型窄化转换为short类型，会丢弃掉前16位的内容，可能导致转换结果与输入值有不同的正负号。

例如下面方法执行后，输出值为32767：
```
public void demo() {
  int i = -32769;
  short s = (short) i;
  System.out.println(s);
}
```

### 对象创建与访问指令
- 创建类实例的指令：new。
- 创建数组的指令：newarray、anewarray、multianewarray。
- 访问类字段和实例字段的指令：getfield、putfield、getstatic、putstatic。
- 把一个数组元素加载到操作数栈的指令：baload、caload、saload、iaload、laload、faload、daload、aaload。
- 将一个操作数栈的值存储到数组元素中的指令：bastore、castore、sasotre、iastore、fastore、dastore、aastore。
- 取数组长度的指令：arraylength。
- 检查类实例类型的指令：instanceof、checkcast。

### 操作数栈管理指令
- 将操作数栈的栈顶一个或两个元素出栈：pop、pop2。
- 复制栈顶一个或两个数值并将复制值或双份的复制值重新压入栈顶：dup、dup2、dup_x1、dup2_x1、dup_x1、dup2_x2。
- 将栈最顶端的两个数值互换：swap。

### 控制转移指令
- 条件分支：ifeq、iflt、ifle、ifne、ifgt、ifge、ifnull、ifnonnull、if_icmpeq、if_icmpne、if_icmplt、if_icmpgt、if_icmple、if_icmpge、if_acmpeq和if_acmpne。
- 复合条件分支：talbleswitch、lookupswitch。
- 无条件分支：goto、goto_w、jsr、jsr_w、ret。

### 方法调用指令
- invokevirtual：调用对象的实例方法。
- invokeinterface：调用接口方法。
- invokespecial：调用需要特殊处理的实例方法，包括实例初始化方法、私有方法和父类方法。
- invokestatic：调用类方法。
- invokedynamic：用于运行时动态解析出调用点限定符所引用的方法

### 方法返回指令：
- ireturn：返回值为boolean、byte、char、short和int类型时使用。
- lreturn：返回值为long类型时使用
- freturn：返回值为float类型时使用
- dreturn：返回值为double类型时使用
- areturn：返回值为引用时使用
- return：声明为void的方法、\<init>和\<clinit>方法使用

### 异常处理指令
在Java中显式抛出的异常都通过athrow指令实现。而运行时异常会自动抛出。

### 同步指令
通过指令monitorenter和monitorexit支持synchronized关键字的语义。

### 总结
Class文件格式平台中立、紧凑、稳定和可扩展的特点，是Java技术体系实现平台无关、语言无关两项特性的重要支柱。从知识点来说，Class文件格式是JVM内存模型与虚拟机执行引擎两部分内容的中间桥梁。文中讲述部分概念时也会提到方法区、栈帧等内容；而了解Class文件格式，可以为了解类的加载与执行预热。
