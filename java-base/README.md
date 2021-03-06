<div align="center"><img src="https://ossweb-img.qq.com/images/lol/web201310/skin/big92000.jpg"/></div>

<div align="center"><img src="https://img.shields.io/badge/WeChat-yamolv-green.svg?logo=Wechat"/>&ensp;<img src="https://img.shields.io/badge/%E7%BD%97%E6%B4%8B%E6%BC%BE-yamolv%40qq.com-red.svg?logo=Tencent%20QQ"/>&ensp;<img src="https://img.shields.io/badge/java-base-yellow.svg"/></div>

# 简介
该目录下包含了部分我在阅读Java源码时的收获，包括经典的List/Set/Map，也有8大基础类型的包装类，甚至能拿出来吹一吹的双轴快速排序的等等内容。

# 解读
源码的阅读带给我跟深入的Java理解。

在看Collection和Map时，它们有许多很有共性的地方。通过最基本的接口来约束类的“造型”，通过扩展Abstract*来加强个性类的能力等等。

我对Java接口的理解为**协议**，它声明了许多方法，从而更准确的告诉实现类它的功能，或者说，约束了实现类的弹性。extends和implements的关系不再仅仅是is-a和has-a，也不是因为我们仅可继承一个类，而实现多个接口（此处表达欠妥，接口与接口之间是可以多继承的）。List/Set等都是我们常用的接口，他们对集合进一步的约束，List有序，元素可以重复，而Set的元素不应重复。我们也可以手写一个类来实现Set，并强行允许其元素重复，但那也许会给我们带来不必要的误解与麻烦。
