# 虚拟机中类的加载与执行
对类文件结构有基础的理解后（了解[类文件结构](https://github.com/2yLoo/broken-sowrd/blob/master/book-review/inside-in-jvm/MemoryAllocationAndRecovery.md)），可以更方便的了解我们的Java代码是如何加载到虚拟机中并执行的。

## 类的加载
Class文件描述了一个类的信息——类名、字段、方法等。它们被编译后以二进制数据的形式存在于磁盘中，需要通过一道工序，才能成为平时我们在内存中访问、使用的类。这道工序就是类加载。

虚拟机把描述类的数据从Class文件加载到内存，并对数据进行校验、转换解析和初始化，最终形成可以被虚拟机直接使用的Java类型，这就是类的加载机制。

类的生命周期可分为7个阶段：加载、验证、准备、解析、初始化、使用和卸载。其中验证、准备、解析3部分统称为连接，经过初始化后，类加载步骤就算完成了。

![类的生命周期](http://assets.processon.com/chart_image/5dfb59dae4b004cc9a3a2259.png?_=1576753983890)

其中的加载、验证、准备、初始化和卸载这5个步骤是顺序开始（开始时间具有先后顺序）的，但它们可以交叉进行，并非要求串行执行（不要求前一步执行完再开始下一步的工作）。

需要立即对类初始化的情况有5种，在这5种情况下，都需要能够从内存中获取类信息：
1. 遇到new、getstatic、putstatic或者invokestatic这4条指令，如果类还没被初始化，需先触发其初始化。因为这4条指令会具体使用到一个已初始化的类信息，指令具体意义可参考[类文件结构](https://github.com/2yLoo/broken-sowrd/blob/master/book-review/inside-in-jvm/MemoryAllocationAndRecovery.md)中字节码指令相关内容。

2. 使用java.lang.reflect包的方法对类进行反射调用的时候。我们必须知道一个类的信息才可进行反射操作。

3. 当初始化一个类时，如果其父类还没有进行过初始化，需先触发父类初始化。

4. 启动虚拟机时，用户需要指定一个执行的主类（包含main()方法），虚拟机会先初始化这个类。

5. 当使用JDK1.7的动态语言支持时，如果一个java.lang.invoke.MethodHandle实例最后的解析结果REF_getStatic、REF_putStatic、REF_invokeStatic的方法句柄，并且这个方法句柄所对应的类没有进行过初始化，则需要先触发其初始化。

这5种情况也可以概括为为对一个类对 **主动引用** 。所有其他情况下，引用类的方式都不会触发类初始化，被称为 **被动引用** 。

下面是几个被动引用的例子（面试题）：

- 通过子类引用父类的静态字段
  ```
  public class SuperClass {
        static {
            System.out.println("父类初始化")
        }

        public static int value = 1;
    }

    public class SubClass {
        static {
            System.out.println("子类初始化")
        }
    }

    public class Main {
        public static void main(String[] args) {
            System.out.println(SubClass.value);
            // 输出结果：
            //    "父类初始化"
            //    1
        }
    }
  ```

  即使 **通过子类引用父类的静态字段，并不会使子类被初始化** 。

- 通过数组定义来引用一个类
  ```
  public class Main {
    public static void main(String[] args) {
      SuperClass[] arr = new SuperClass[10];
      // 输出结果：无
    }
  }
  ```

  **在通过数组定义的方式来引用一个类时，并不会使被引用的类初始化** ，声明数组会初始化另一个类：“[Lcom.packagename.your.SuperClass”。它不是用户定义的类，而是由虚拟机自动生成、继承于java.lang.Object的子类。

- 使用类的常量
  ```
  public class ConstantClass {
    static {
      System.out.println("常量类初始化")
    }
    public static final String VALUE = "HELLOWORLD";
  }

  public class Main {
    public static void main(String[] args) {
      System.out.println(ConstantClass.VALUE);
      // 输出结果：
      //    "HELLOWORLD"
    }
  }
  ```

  在编译阶段， **通过常量传播优化，常量值已经被存储到了“Main”类的常量池中，在“Main”类中访问`ConstantClass.VALUE`时，并不会真正引用到ConstantClass类** 。


接口与类不同的一点是，初始化接口时，并不会要求其父接口都全部完成了初始化。如果了解类文件结构中继承的表达方式，可得知，在类A继承类B时，会将该继承关系保存到类A的父类索引（super_class）中，而接口A继承接口B、接口C时，该继承关系会被保存到接口索引集合（interfaces）中。从字节码的层面上看，接口之间的继承（extends）关系，实则为实现（implements）。

所以第3点中的“父类”应该是针对类文件结构中的父类索引信息“super_class”，而不是我们平时理解的“父类”。

### 类加载步骤——1. 加载
加载是类加载的第一步。在该阶段虚拟机需完成以下3件事情：
1. 通过一个类的全限定名获取定义此类的二进制字节流。
2. 将该字节流所代表的静态存储结构转化为方法区的运行时数据结构。
3. 在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据的访问入口。

其中第一步获取该类的二进制字节流可以有很多种实现方式，最常见的是从ZIP包中获取，例如JAR、EAR、WAR等格式的ZIP包；也可从网络中获取，最典型的应用就是Applet；在动态代理技术中，则是在运行时计算生成；还可由其他文件生成，如JSP应用……

对数组而言，数组类本身并不通过类加载器创建，而是由Java虚拟机直接创建。但数组类的元素类型（指数组去掉所有维度后的类型。在数组类String[]中，元素类型即String。）最终要靠类加载器创建。

在HotSpot中， **加载一个类时产生的实例化java.lang.Class对象被存储在方法区里** ，其他对象一般存储在Java堆中。

### 类加载步骤——2. 验证
验证是连接阶段的第一步， **验证的目的是为了保证Class文件的字节流中包含的信息符合当前虚拟机的要求，以及不危害虚拟机自身的安全** 。验证是类加载过程中比较占资源的一步，验证的过程很复杂，大致可分为4个阶段：文件格式验证、元数据验证、字节码验证、符号引用验证。

#### 文件格式验证
文件格式验证阶段主要验证字节流是否符合Class文件格式的规范。例如是否以魔数开头、主次版本号是否为当前虚拟机所接受、常量的索引值是否指向不存在或不符合类型的常量……

文件格式验证是基于二进制字节流进行的，通过此阶段验证后，字节流才会进入内存的 **方法区** 中进行存储，后面的3个验证阶段全部都是基于方法区的存储结构进行的，不会再直接操作字节流。

#### 元数据验证
在元数据验证阶段，会对字节码描述的语义进行分析，以保证其描述的信息符合Java语言规范的要求。如该类是否有父类、是否继承了被final修饰的类（被final修饰的类不允许被继承）、非抽象类是否实现了父类或接口中要求实现的所有方法……

#### 字节码验证
字节码验证阶段的主要目的是通过数据流和控制流分析，确定程序语义是合法的、符合逻辑的。此阶段确保被校验类的方法在运行时不会危害虚拟机安全。如保证任意时刻操作数栈的数据类型与指令代码序列都能配合工作；保证跳转指令不会跳转到方法体以外的字节码指令上；保证方法体中的类型转换有效性……

#### 符号引用验证
符号引用验证的目的是验证符号引用，它发生在 **虚拟机将符号引用转化为直接引用的时候** 。这个转化动作发生在类加载的第四步——解析阶段中。

如符号引用中通过字符串描述的全限定名是否能匹配对应类；指定类中是否存在符合方法的字段描述符以及简单名称所描述的方法和字段；符号引用中的类、字段、方法是否能被当前类访问……

其实有许多验证步骤在编译器即可被过滤，但编译与加载并不是一个整体过程，我们无法保证被加载的类是安全可靠的，所以在类加载过程中，做这些验证并不是多此一举。

### 类加载步骤——3. 准备
准备阶段会将方法区中的内存分配给 **类变量（被static修饰的变量）** ，并设置类变量的初始值。

初始值并不是我们平时理解的编码时赋给变量的值，而是“零值”。

如在 `public static int value = 1;` 中，通过准备阶段，类变量value的初始值为0，而不是1。在准备阶段中，并不会执行任何方法。将value赋值为1的动作发生在类构造器方法\<clinit>中，此方法会在类加载的第五步——初始化阶段执行。

Java中各种类型的零值如下：

| 数据类型  | 零值      |
| --------- | --------- |
| char      | '\u0000'  |
| byte      | (byte)0   |
| short     | (short) 0 |
| int       | 0         |
| long      | 0L        |
| float     | 0.0f      |
| double    | 0.0d      |
| boolean   | false     |
| reference | null      |

如果字段为包装类型，如Integer，则其零值为null，因为它属于reference。

而在一个特殊情况下，类变量不会设置零值：被final修饰的类变量（我们更习惯称为类常量）。在这种情况下，类字段对字段属性表中会存在ConstantValue属性，在准备阶段，value会被初始化为ConstantValue的指定值，而不是零值，如 `public static final value = 1;`

关于零值，除了在类加载时期，新建对象时也会有体现：
```
public class Demo {

    private boolean value;

    public boolean getValue() {
        return value;
    }

}

public class Main {

    public static void main(String[] args) {
        // 此时新的Demo()对象中 value 字段并未被显式赋值
        // 返回的结果为boolean的零值 false
        // 输出 false
        System.out.println(new Demo().getValue());
    }

}
```

### 类加载步骤——4. 解析
解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程。符号引用通过验证后，会被虚拟机加载到内存中，使之后的访问直接从内存中获取，而不是通过字节码读取其符号引用。

- 符号引用：符号引用以一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时能无歧义地定位到目标即可，符号引用与虚拟机实现的内存布局无关。

- 直接引用：直接引用可以是直接指向目标的指针、相对偏移量或是一个能间接定位到目标的句柄，直接引用是和虚拟机实现的内存布局相关的。

解析的对象主要是 **类或接口、字段、类方法、接口方法** 、方法类型、方法句柄和调用点限定符这7类符号引用。

| 类型         | 常量池中对应项目                 |
| ------------ | -------------------------------- |
| 类或接口     | CONSTANT_Class_info              |
| 字段         | CONSTANT_Fieldref_info           |
| 类方法       | CONSTANT_Methodref_info          |
| 接口方法     | CONSTANT_InterfaceMethodref_info |
| 方法类型     | CONSTANT_MethodType_info         |
| 方法句柄     | CONSTANT_MethodHandle_info       |
| 调用点限定符 | CONSTANT_InvokeDynamic_info      |

其中前4类与Java关系密切，而后3类是关于动态语言支持的。

#### 类或接口的解析
在Java中，一个类引用到另一个类是很常见的事。 它们之间的关系在Class中会通过符号引用的方式关联起来。当需要把 **类A** 中 **符号引用B** 解析为 **类或接口C** 的直接引用时，会经历3个步骤：

1. 如果C不是数组类型，虚拟机会把代表B的全限定名传给A的 **类加载器** ，并加载C。
2. 如果C是数组类型，并且其元素类型为对象（这个限定条件并不是多余的，在文章开头提到声明数组时会初始化一个由虚拟机生成的数组类，而此处讨论的是类似Integer[]的数组类型，需排除int[]这种元素类型并非对象的数组类型。），则按第1点的规则加载数组元素类型。
3. 如果上述步骤可将C加载成功，还需要确保A是否具备对C的访问权限，如果不具备，则抛出 `java.lang.IllegalAccessError` 异常。

#### 字段解析
在解析字段符号引用时，需要先解析该字段所属的 **类或接口C** 的符号引用，也就是上面提到的解析步骤。C的符号引用解析成功后，查找字段时经过以下流程：

1. 如果C本身就办好了简单名称和字段描述都与目标相匹配的字段，则返回这个字段的直接引用，查找结束。
2. 否则，如果C中实现了接口，则按照继承关系 **从下往上** 递归搜索各个接口和它的父接口，如果查找到匹配字段，则返回并结束。
3. 否则，如果C不是祖类java.lang.Object，按继承关系从 **从下往上** 递归搜索其父类，如果查找到匹配字段，则返回并结束。
4. 否则，查找失败，抛出java.lang.NoSuchFieldError异常。

围绕这段步骤，可以出几个简单的面试题。

当C并无该字段，而C实现的两个接口A，B都定义了该字段，即使A、B中定义的字段类型不同，并且在调用时要求了int类型，编译还是无法通过：
```
public class MainClass {

    interface Interface1 {
        String VALUE = "Interface1";
    }

    interface Interface2 {
        int VALUE = 1;
    }

    static class Sub implements Interface1, Interface2 {
    }

    public static void main(String[] args) {
        test(Sub.VALUE); // IDEA 提示Reference of 'VALUE' is ambiguous
    }

    public static void test(int value) {
        System.out.println(value);
    }
}
```

当A、B其中一个为类，另一个为接口时，同样适用以上逻辑，代码就不贴上来了。

以上讨论的是A、B之间并无实现关系的情况，或者说A（或B）的类文件中接口索引集合中不存在B（或A）的接口索引（在类文件结构一章中有提到， **类对接口的实现关系以及接口之间的继承关系都是通过类文件中的接口索引集合维护的** ）。如果A、B存在实现关系（或者接口间的继承关系），则结果会有所变化：
```
public class MainClass {

    interface Interface1 {
        String VALUE = "Interface1";
    }

    interface Interface2 extends Interface1 {
        String VALUE = "Interface2";
    }

    // 注意此处先实现接口1，再实现接口2
    static class Sub implements Interface1, Interface2 {
    }

    public static void main(String[] args) {
        System.out.println(Sub.VALUE); // IDEA 提示Reference of 'VALUE' is ambiguous
    }

}
```

但如果改变一下类C实现的接口顺序，则可行：
```
public class MainClass {

    interface Interface1 {
        String VALUE = "Interface1";
    }

    interface Interface2 extends Interface1 {
        String VALUE = "Interface2";
    }

    // 此处先实现接口2，再实现接口1
    static class Sub implements Interface2, Interface1 {
    }

    public static void main(String[] args) {
        System.out.println(Sub.VALUE); // 输出接口Interface2的值：Interface2
    }

}
```

如果类A实现了接口B，C继承了A实现了B，则不会有问题：
```
public class MainClass {

    static class Parent1 implements Interface1 {
        public static int VALUE = 1;
    }

    interface Interface1 {
        String VALUE = "Interface1";
    }

    static class Sub extends Parent1 implements Interface1 {
    }

    public static void main(String[] args) {
        System.out.println(Sub.VALUE); // 输出父类Parent1的值：1
    }

}
```

#### 类方法解析
> 多嘴一句，类方法是类中由static关键字修饰的方法，而不是指类中的所有方法（类方法+对象方法）。

类方法解析前也需要先解析其所属类或接口的符号引用。类方法解析步骤与字段解析步骤相似，以C表示这个类，步骤如下：
1. 如果C是接口，则抛出java.lang.IncompatibleClassChangeError异常。
2. 如果通过第一步校验，则在C中查找是否有简单名称（方法名）和描述符（参数信息与返回类型）与目标相匹配的方法，如果查到匹配方法，则返回并结束。
3. 否则，在类C的父类中递归查找，如果查到匹配方法，则返回并结束。
4. 否则，在类C实现的接口列表以及它们的父接口中递归查找，如果查到匹配方法，说明C是一个抽象类，查找结束，抛出java.lang.AbstractMethodError异常。
5. 否则，宣告方法查找失败，抛出java.lang.NoSuchMethodError。

#### 接口方法解析
接口方法解析前同样需要先解析出接口方法表的class_index项中索引的方法所属的类或接口的符号引用，如果解析成功，以C表示该接口，虚拟机按一下步骤进行方法搜索：
1. 如果C是类而不是接口，则抛出java.lang.IncompatibleClassChangeError异常。
2. 否则，在接口C中查找是否有简单名称和描述符与目标相匹配的方法，如果查到匹配方法，则返回并结束。
3. 否则，在接口C的父接口中递归查找，直到查到java.lang.Object类为止，如果查到匹配方法，则返回并结束。
4. 否则，查找失败，抛出java.lang.NoSuchMethodError异常。

### 类加载步骤——5. 初始化
类初始化结束后，类加载的动作就完成了。初始化阶段会执行类构造器\<clinit>()方法。
#### \<clinit>()方法
\<clinit>()方法由编译器自动收集类中所有类变量的赋值动作和静态语句块中的语句合并产生，编译器收集的顺序是由语句在源文件中出现的顺序所决定的，静态语句块中只能访问到定义在静态语句块之前的变量，定义在之后的变量只能在这个语句快中被赋值，无法被访问。
```
static class Demo {

    static {
        i = 0; // 编译可通过
        System.out.println(i); // 编译不通过，提示“非法向前引用”
    }

    static int i = 1;

}
```
```
static class Demo {

    static {
        i = 0;
    }

    static int i = 1;

    static {
        i++;
        System.out.println(i); // 输出结果：2
    }
}
```

当类之间存在继承关系时， **父类的\<clinit>()方法优先执行，父类中的静态语句块也会优先于子类的变量赋值操作** 。

在接口中，虽然不能定义静态语句块，但是接口中的初始化赋值操作也是通过\<clinit>()方法完成的。与类不同的是，执行接口的\<clinit>()方法时，不需要先执行父类的\<clinit>()方法，接口的实现类在初始化时，也不会执行接口的\<clinit>()方法。从类文件结构的角度上来看，初始化 **类或接口C** 时，C的类文件结构里接口索引集合中的接口都不会执行\<clinit>()方法。

虚拟机也会保证一个类的\<clinit>()方法在多线程环境中被正确地加锁、同步，保证同一时间只有一个线程执行这个类的\<clinit>()方法。

### 类加载器
类加载器的作用是通过一个类的全限定名来获取描述此类的二进制字节流。它用于实现类的加载动作。使用不同的类加载器加载同一个Class文件时，两个类 **必定不相等** 。

类加载器可分为三类：
- 启动类加载器：它是虚拟机自带的类加载器，由C++实现，用于加载目录\<JAVA_HOME>\\lib下的类，也可通过参数-Xbootclasspath指定其他路径。启动类加载器无法被Java程序直接引用。
- 扩展类加载器：它负责加载\<JAVA_HOME>\\lib\\ext目录下的类，可被开发者使用。
- 应用程序类加载器：一般情况下，这是程序中默认的类加载器，它负责加载用户类路径上所指定的类库，也可被开发者直接使用。

我们也可自定义加载器，一般加载器之间的关系如图：
![类加载器双亲委派模型](http://assets.processon.com/chart_image/5e00a078e4b0aef94ca6ce73.png?_=1577099605524)

图中的类加载器合作关系称为 **双亲委派模型** 。这种模型下的父子关系一般不通过继承实现，而是通过组合关系复用父加载器。

在双亲委派模型下，父加载器的优先级更高。在子加载器收到一个类时，会优先委派给父加载器处理，只有父类的搜索范围里没有该类时，子加载器才会尝试加载。

而这种约束并不是强制性的，虽然它被Java设计者们所推荐，但人们可以不按这套规定自定义实现方式。只要有足够的意义和理由，突破已有的原则就可认为是一种创新。

## 类的执行

### 运行时栈帧结构
在虚拟机运行时数据区中，虚拟机栈是线程隔离的数据区，其中的栈元素称为栈帧，栈帧是用于支持虚拟机进行方法调用和方法执行的数据结构，它存储了方法的局部变量表、操作数栈、动态连接和方法返回地址等信息。

栈帧的结构如图所示：
![栈帧的概念结构](http://assets.processon.com/chart_image/5e00a6f2e4b0125e2916bbfe.png?_=1577101390199)

#### 局部变量表
局部变量表用于存放方法参数和方法内部定义的局部变量。在字节码的Code属性中，max_locals代表了局部变量表所需的存储空间。

局部变量表的容量基本单位为变量槽（Variable Slot，简称为Slot）。一般Slot占用32位长度的内存空间。可用于存储boolean、byte、char、short、int、float、reference和returnAddress8种类型。其中reference类型表示一个对象实例的引用。returnAddress已很少使用，非重点关注类型。至于64位的long和double，在虚拟机中使用两个Slot维护，其读写被分割为两次32位读写。 **在执行实例方法（非static修饰的方法）时，局部变量表的第0位索引为“this”，用于传递方法所属对象实例的引用** 。

之前也提到Slot是可以重用的，在某个变量已经超出其作用域时，它所占用的Slot就可给其他变量使用。但其回收时机并不是变量超出其作用域的时候，而是下一次要对其Slot填充的时候。

例如在下面代码中，即使触发了系统的垃圾回收，存储v所用的Slot内存也不会被回收掉。
```
public static void main(String[] args) {
    int[] v = new int[1024];
    System.gc();
}
```

但如果后续变量需要使用该Slot，则会被回收：
```
public static void main(String[] args) {
    int[] v = new int[1024];
    int m = 0;
    System.gc();
}
```

也可通过将变量值置为null，使其Slot为空：
```
public static void main(String[] args) {
    int[] v = new int[1024];
    v = null;
    System.gc();
}
```

但开发人员不应该依赖这种方式来节省空间，从编码角度讲，以恰当的变量作用域来控制变量回收的时间才是最优雅的解决方法；从执行角度讲，使用赋null值的操作来优化内存回收是建立在对字节码执行引擎概念模型的理解之上。

#### 操作数栈
通过使用各种字节码指令对操作数栈执行出栈/入栈操作，可以对代码中的各项Java数据类型进行读取、写入，在字节码的Code属性中，max_stacks代表了操作数栈所需的最大深度。

#### 动态连接
之前提到，在类的加载阶段，或者第一次使用的时候，会把符号引用转化为直接引用，这是一种静态解析的行为。与之相对的另一部分在每次运行期间转化为直接引用，称为动态连接。

#### 方法返回地址
退出方法的方式有两种：通过return或正常执行完的方式称为正常完成出口，这样退出后会将返回值传递给上层的方法调用者；通过抛出异常的方式退出方法称为异常完成出口，可能是虚拟机内部产生的异常，或者使用字节码指令athrow抛出的异常，只要本方法的异常表中没有搜索到匹配的异常处理器，就会导致方法退出，并且不会产生返回值。

### 方法调用
方法调用阶段唯一的任务就是确定被调用方法的版本。

#### 解析
有一类方法，调用目标在程序代码写好、编译器编译时就必须确定下来，这类方法的调用称为解析。

虚拟机中方法调用的字节码指令有5条：
1. invokestatic：调用静态方法。
2. invokespecial：调用实例构造器\<init>方法、私有方法和父类方法。
3. invokevirtual：调用所有的虚方法。
4. invokeinterface：调用接口方法，会在运行时再确定一个实现此接口的对象。
5. invokedynamic：在运行时动态解析出调用点限定符所引用的方法。

其中通过invokestatic和invokespecial调用的方法，在解析阶段就可确定其唯一的调用版本，有静态方法、私有方法、实例构造器、父类方法这4种。


解析调用是静态的过程，在编译器间就会完全确定，在类装载的解析阶段就会把涉及的符号引用全部转变为可确定的直接引用。

#### 分派
分派调用可分为静态分派和动态分派。在了解它们之前，需要先了解静态类型与实际类型。

在代码 `List<String> list = new ArrayList();` 中， `List<String>` 称为静态类型，而 `ArrayList()` 称为实际类型。


##### 静态分派
依赖静态类型来定位方法执行版本的分派动作称为静态分派。静态分派常用在方法重载（Overload）上。

浅显地说，对于多个重载方法 `method(??? arg)` ，实际执行时会分派调用到哪个版本，与arg的静态类型相关。

```
public class Overload {

    public static void print(char arg) {
        System.out.println("char: " + arg);
    }

    public static void print(int arg) {
        System.out.println("int: " + arg);
    }

    public static void print(long arg) {
        System.out.println("long: " + arg);
    }

    public static void print(float arg) {
        System.out.println("float: " + arg);
    }

    public static void print(double arg) {
        System.out.println("double: " + arg);
    }

    public static void print(Character arg) {
        System.out.println("Character: " + arg);
    }

    public static void print(Serializable arg) {
        System.out.println("Serializable: " + arg);
    }

    public static void print(Comparable arg) {
        System.out.println("Comparable: " + arg);
    }

    public static void print(Object arg) {
        System.out.println("Object: " + arg);
    }

    public static void print(char... arg) {
        System.out.println("chars: " + arg);
    }

    public static void main(String[] args) {
        char c = 1;
        print(c);
    }
}
```

在上例代码中，最匹配的重载方法版本为 `print(char arg)` ，但如果把这个方法进行注释，则会通过类型转换调用 `print(int arg)`，这种自动转换能发生多次，顺序为char、int、long、float、double，由于char到byte或short的转型是不安全的，因此自动转换并不包括这两类。

当自动转换后仍然没有匹配的重载版本，会通过 **自动装箱** 调用 `print(Character arg)` 。

由于Character实现了Serializable和Comparable两个接口，因此当把Character版本注释掉时，则会调用其接口版本 `print(Serializable arg)` 或 `print(Comparable arg)` ，但是两者优先级相同，因此无法通过编译。出现这种情况时，可以在调用处显式指定参数的静态类型：
```
public static void main(String[] args) {
    char c = 1;
    print((Serializable) c);
}
```

继续注释后，会调用其父类 `print(Object arg)` 的版本。在有多个父类的情况下，会优先匹配子类。

继续注释，则调用变长参数的版本 `print(char... arg)`，其实这并不是结尾，同样地，也可通过重载版本 `print(Character... arg)` 或者 `print(Serializable... arg)` 执行main中的方法，这种极端情况很少见，但是对我们理解重载方法匹配优先级是有帮助的。

重载方法的匹配顺序总结如图：

![重载方法匹配优先级](http://assets.processon.com/chart_image/5e0185cfe4b0bb7c58c7fdfb.png?_=1577158304477)

##### 动态分派
动态分派依赖实际类型来定位执行版本，与重写（Override）密切相关，也是多态性的一种体现。

```
public static void main(String[] args) {
    Collection<String> list = new ArrayList<>();
    Collection<String> set = new HashSet<>();
    list.add("A");
    list.add("A");
    list.add("A");
    set.add("A");
    set.add("A");
    set.add("A");
    System.out.println("List: " + list);
    System.out.println("Set: " + set);
}
```

在上例中，list和set的静态类型都是Collection<String>，但是对他们调用add方法后，却得到不同的结果。这是因为它们的实际类型不同，在字节码指令中，会通过它们的实例对象（ArrayList、HashSet）调用相关方法（add）。
