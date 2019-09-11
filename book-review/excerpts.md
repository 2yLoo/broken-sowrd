## 摘抄

> 摘录前辈们在书中传递的星火。

### 《阿里巴巴Java开发手册》

对程序员来说，关键是骨子里意识到规范也是生产力，个性化尽量表现在代码可维护性和算法效率的提升上。

很多编程方式客观上没有对错之分，一致性很重要，可读性很重要，团队沟通效率很重要。

### 《重构——改善既有代码的设计》

如果你发现自己需要为程序添加一个特性，而代码结构使你无法很方便地大成门目的，那就先重构那个程序，使特性的添加比较容易进行，然后再添加特性。

每当我要进行重构的时候，第一个步骤永远相同：我得为即将修改的代码建立一组可靠的测试环境。

任何不会被修改的变量都可以被我当成参数传入新的函数，至于会被修改的变量就需格外小心。

重构技术就是以微小的步伐修改程序。如果你犯下错误，很容易便可发现它。

任何一个傻瓜都能写出计算机可以理解的代码。唯有写出人类容易理解的代码，才是优秀的程序员。

绝大多数情况下，函数应该放在它所使用的数据的所属对象内。

最好不要在另一个对象的属性基础上运用switch语句。如果不得不使用，也应该在对象自己的数据上使用，而不是在别人的数据上使用。

重构（名词）：对软件内部结构的一种调整，目的是在不改变软件可观察行为的前提下，提高其可理解性，降低其修改成本。

重构不会改变软件可观察的行为——重构之后的软件功能一如既往。任何用户，不论最终用户或其他程序员，都不知道已经有东西发生了变化。

良好的设计是快速开发的根本。

第一次做某件事时只管去做；第二次做类似的事会产生反感，但无论如何还是可以去做；第三次再做类似的事，你就应该重构。

程序有两面价值：“今天可以为你做什么”和“明天可以为你做什么”。

难以阅读的程序，难以修改；逻辑重复的程序，难以修改；添加新行为时需要修改已有代码的程序，难以修改；带复杂条件逻辑的程序，难以修改。

你可以安全地修改对象内部实现而不影响他人，但对于接口要特别谨慎——如果接口被修改了，任何事情都有可能发生。

只有当需要修改的接口被哪些“找不到，即使找到也不能修改”的代码使用时，接口的修改才会成为问题。

让旧接口调用新接口。

不要过早发布接口。请修改你的代码所有权政策，使重构更顺畅。

如果项目已经非常接近最后期限，你不应该再分心于重构，因为已经没有时间了。

软件和机器有着很大的差异：软件的可塑性更强，而且完全是思想产品。

性能改善一旦被分散到程序各角落，每次改善都只不过是从对程序行为的一个狭隘视角出发而已。

让小函数容易理解的真正关键在于一个好名字。

我们遵循这样一条原则：每当感觉需要以注释来说明点什么的时候，我们就把需要说明的东西写进一个独立函数中，并以其用户（而非实践手法）命名。

关键不在于函数的长度，而在于函数“做什么”和“如何做”之间的语义距离。

如果代码前方有一行注释，就是在提醒你：可以将这段代码替换成一个函数，而且可以在注释的基础上给这个函数命名。

条件表达式和循环常常也是提炼的信号。

将常量独立成一个公共类来维护更加方便。

你所创建的每一个类，都得有人去理解它、维护它，这些工作都是要花钱的。

当你感觉需要撰写注释时，请先尝试重构，试着让所有注释都变得多余。

最好的注释就是没有注释。

把注意力集中于接口而非实现。

编写未臻完善的测试并实际运行，好过对完美测试的无尽期待。

对read()而言，边界条件应该是第一个字符、最后一个字符、倒数第二个字符。

在我看来，长度不是问题，关键在于函数名称和函数本体之间的语义距离。

类往往会因为承担过多责任而变得臃肿不堪。

面向对象编程的关键原则之一就是封装。

面向对象的首要原则之一就是封装，或者称为“数据隐藏”。

复杂的条件逻辑是最常导致复杂度上升的地点之一。

程序最后的成品往往将断言统统删除。

断言的价值在于：帮助程序员理解代码正确运行的必要条件。

代码的可理解性应该是我们虔诚的追求目标。

### 其他