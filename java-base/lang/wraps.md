# Java 8大基础类型
java目前有8大基础类型：byte,short,int,long,float,double,char,boolean

分别对应装箱类：Byte,Short,Integer,Long,Float,Double,Character,Boolean

我们在使用这些包装类的时候，Java帮助我们自动装箱、拆箱。例如下面的操作，是不会报任何异常的：
```
int a = 1;
Integer b = a + 1;
```
而在使用过程中，它们还是有一点细微的差别。

### 差别1 —— 默认值
```
public class Demo {

    byte byteValue;

    Byte byteClassValue;

    short shortValue;

    Short shortClassValue;

    int intValue;

    Integer integerClassValue;

    long longValue;

    Long longClassValue;

    float floatValue;

    Float floatClassValue;

    double doubleValue;

    Double doubleClassValue;

    char charValue;

    Character characterClassValue;

    boolean booleanValue;

    Boolean booleanClassValue;

    @Override
    public String toString() {
        return "Demo{" +
                "\nbyteValue=" + byteValue +
                ", byteClassValue=" + byteClassValue +
                ", \nshortValue=" + shortValue +
                ", shortClassValue=" + shortClassValue +
                ", \nintValue=" + intValue +
                ", integerClassValue=" + integerClassValue +
                ", \nlongValue=" + longValue +
                ", longClassValue=" + longClassValue +
                ", \nfloatValue=" + floatValue +
                ", floatClassValue=" + floatClassValue +
                ", \ndoubleValue=" + doubleValue +
                ", doubleClassValue=" + doubleClassValue +
                ", \ncharValue=" + charValue +
                ", characterClassValue=" + characterClassValue +
                ", \nbooleanValue=" + booleanValue +
                ", booleanClassValue=" + booleanClassValue +
                "\n}";
    }

    public static void main(String[] args) {
        Demo demo = new Demo();
        System.out.println(demo.toString());
    }
}

// 控制台输出：
Demo{
byteValue=0, byteClassValue=null,
shortValue=0, shortClassValue=null,
intValue=0, integerClassValue=null,
longValue=0, longClassValue=null,
floatValue=0.0, floatClassValue=null,
doubleValue=0.0, doubleClassValue=null,
charValue= '\u0000' 0, characterClassValue=null,
booleanValue=false, booleanClassValue=null
}
```
从控制台可以看到，**8大基本数据类型都是有初始值的，而它们的包装类默认值均为null。这是他们的一个差别，也是各自的特性，更是我们选择基础类型/包装类时的一个参考点。**

例如我们在Mongo中有这样一个Document：
```
{
    "_id" : ObjectId("5b7cc85acef2ed536c124c2d"),
    "name" : "2y"
}
```

而我们定义的实体类如下：
```
public class Person {
  private String id;
  private String name;
  private int age; // 在无法获取该字段值时，默认值为0
  // Getter & Setter
}
```

则接收到的Person实例中age的值为0；如果将int替换为Integer，则值为null。

从而在调用getAge()时返回不同的值。这是我们需要注意的一个细节。

从这一点也暗示了我们看到、使用包装类时应把它们看做一个类，而不是简单的基础类型。

### 差别2——地址
它们的差别不仅如此，在判断两个值是否相等时稍不注意也许就会埋下炸弹，继续看例子：
```
public static void main(String[] args) {
    int a = 0;
    int b = 0;
    Integer c = 0;
    Integer d = 0;
    System.out.println("地址验证：");
    System.out.println("a == b? " + (a == b));
    System.out.println("a == c? " + (a == c));
    System.out.println("c == d? " + (c == d));

    System.out.println("equals方法验证：");
    System.out.println("c.equals(a)? " + c.equals(a));
    System.out.println("c.equals(d)? " + c.equals(d));
}

// 控制台输出：
地址验证：
a == b? true
a == c? true
c == b? true
equals方法验证：
c.equals(a)? true
c.equals(d)? true
```

看上去没什么问题，但是我们更改一下c与d的赋值语句：
```
public static void main(String[] args) {
    Integer c = new Integer(0);
    Integer d = new Integer(0);
    System.out.println("地址验证：");
    System.out.println("c == d? " + (c == d));

    System.out.println("equals方法验证：");
    System.out.println("c.equals(d)? " + c.equals(d));
}

// 控制台输出：
地址验证：
c == d? false
equals方法验证：
c.equals(d)? true
```
此处也好理解，我们创建了两个Integer对象，不同对象的地址是绝对不同的，而连等号```==```在java中即为简单的比较地址。

那我们再回到第一种赋值方式，对值做一下更改:
```
public static void main(String[] args) {
    Integer c = 128;
    Integer d = 128;
    System.out.println("地址验证：");
    System.out.println("c == d? " + (c == d));

    System.out.println("equals方法验证：");
    System.out.println("c.equals(d)? " + c.equals(d));
}

// 控制台输出：
地址验证：
c == d? false
equals方法验证：
c.equals(d)? true
```
有意思的事情来了，比较第一次的代码，我们仅将值0改为128，为什么c == d会为false呢？😮

由辣个看了源码的蓝人解释一下 😏😏😏

`java.lang.Integer.class` 中有这么一段代码：
```
private static class IntegerCache {
    static final int low = -128;
    static final int high;
    static final Integer cache[];

    static {
        // high value may be configured by property
        int h = 127;
        String integerCacheHighPropValue =
            sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
        if (integerCacheHighPropValue != null) {
            try {
                int i = parseInt(integerCacheHighPropValue);
                i = Math.max(i, 127);
                // Maximum array size is Integer.MAX_VALUE
                h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
            } catch( NumberFormatException nfe) {
                // If the property cannot be parsed into an int, ignore it.
            }
        }
        high = h;

        cache = new Integer[(high - low) + 1];
        int j = low;
        for(int k = 0; k < cache.length; k++)
            cache[k] = new Integer(j++);

        // range [-128, 127] must be interned (JLS7 5.1.7)
        assert IntegerCache.high >= 127;
    }

    private IntegerCache() {}
}
···
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

原来源码中[-128,127]之间的256个Integer对象都被缓存到一个数组中，而当我们通过 `Integer i = 233` 给i赋值时，会调用Integer类中的 `valueOf()` 方法（没有JVM知识的我是如何知道的？别问，问就是断点debug）。所以此处也就可以解释为什么上面的例子中c、d的值为0时 c==d 返回true，而在它们的值为128时结果为false。**因为当值在[-128,127]之间时valueOf()方法会返回同一个对象，其他情况才会创建新对象**。

与Integer类似，Byte,Short,Long也有[-128,127]的缓存（Boolean则有值为true和false的两个静态对象）。

回到我们真实使用的情景下，有时我们无法把控得到的值大小为多少， **如果我们需要的是判断两个值是否相等， `equals()` 方法比 `==` 更适合我们** ！

### 差别3——便捷性
包装类除了对基础类型的转换支持，还提供了许多便捷的方法，继续以Integer为例。
```
// 将i转换为3进制的String输出：
Integer.toString(13, 3) -> 111

// 将Integer对象输出为String：
Integer i = 12345;
i.toString() -> "12345"

// 指定进制将某Srring转换为int值
Integer.parseInt("157f", 16) -> 5503

// 将Integer对象转换成其他基础类型值
Integer i = 0;
System.out.println(i.longValue()); -> 0L
System.out.println(i.floatValue()); -> 0.0F

// 获取int值在二进制中最高位1的值
Integer.highestOneBit(9) -> 8 // 9的二进制为1001，最高位的1的值为8

// 生成哈希码
Integer i = 5503;
i.hashCode() -> 5503
```

不仅如此，当我们需要将集合的元素或者Map的键、值设置为基础类型时，是无法通过编译的，但是包装类可以帮我们解决这一难题：
```
// List<int> list = new ArrayList<>(); 编译失败
// Set<int> set = new HashSet<>(); 编译失败
// Map<int, int> map = new HashMap<>(); 编译失败
List<Integer> list = new ArrayList<>();
Set<Integer> set = new HashSet<>();
Map<Integer, Integer> map = new HashMap<>();
```
