## Java的程序编译和代码优化

Java语言的“编译期”有不同的解释，例如它可能是指把*.java文件转变成*.class文件的过程，也可能是指把*.class文件转变成机器码的过程。
第一个过程的编译器被称为“前端编译器”，例如javac
第二个过程的编译器被称为“JIT编译器，Just In Time Compiler”，例如HotSpot的C1、C2编译器。

![](http://wx1.sinaimg.cn/mw1024/9c96fc9cly1g6lh158ygoj20pc0fygn4.jpg)

### 前期(编译期)优化

#### Javac编译器
 Javac编译器是JDK自带的，由Java实现的前端编译器，源码在 JDK_SRC_HOME/langtools/src/share/classes/com/sun/*

Javac编译过程大概分为3个步骤
1. 插入和填充符号表
2. 插入式注解处理器的注解处理
3. 分析与字节码生成
其中1、2部分会循环进行，直到语法树不再变化。

![](http://wx4.sinaimg.cn/mw1024/9c96fc9cly1g6lh15wdunj20xy098acn.jpg)

#### Javac编译过程的主体方法

![](http://wx3.sinaimg.cn/mw1024/9c96fc9cly1g6lh156qo6j20fm08ujrm.jpg)

![](http://wx3.sinaimg.cn/mw1024/9c96fc9cly1g6lh1565fej20d906pglm.jpg)

##### 1.1 词法、语法分析
词法分析是将源代码的字符流转变为标记集合，单个字符是程序编写过程的最小元素，而标记是编译过程的最小元素，关键词、变量名、字面量、运算符都可以成为标记，如`int a = b + 2`这行代码包含了6个标记，分别是int、a、=、b、+、2，虽然关键词int是三个字符，但它只是一个Token，不能再拆分。

语法分析是根据Token序列构造抽象语法树的过程，**抽象语法树(Abstract Syntax Tree,AST)**是一种用来描述程序代码语法结构的树形表示方式，语法树上的每一个节点都代表源代码中的一个语法结构，例如包、类型、修饰符、运算符、接口、返回值、甚至是注释都可以是一个语法结构。

##### 抽象语法树

![](http://wx2.sinaimg.cn/mw1024/9c96fc9cly1g6lh15apkfj20pb0b4js2.jpg)

上图来自IDEA插件**JDT ASTView**

##### 1.2 填充符号表
符号表是由一组符号地址和符号信息构成的表格，可以把它想象成哈希表中的K-V形式。
符号表中登记的信息在编译的不用阶段都要用到。在语义分析（后面的步骤）中，符号表所登记的内容将用于语义检查和产生中间代码，在目标代码生成阶段，当对符号名进行地址分配时，符号表是地址分配的依据。

##### 2 执行注解处理
JDK1.5之后提供了对注解的支持，运行期间发挥作用。
JDK1.6之后提供了插入式注解处理器标准API，在编译期间对注解进行处理，我们可以把它看做是一组编译器的插件，在这些插件里面，可以读取、修改、添加抽象语法树中的任意元素。

>注：如插入式注解处理器在处理注解期间对语法树进行了修改，编译器将重新填充符号表过程

直到所有的插入式注解处理器都没有再对语法树进行修改为止

>初始化过程在initProcessAnnotations()方法中完成
执行过程在processAnnotation()方法中完成

判断是否还有新的注解处理器需要执行
如果有，通过com.sun.tools.javac.processing.JacacProcessingEnvironment类的doProcessing()方法生成一个新JavaCompiler对象对编译的后续步骤进行处理

##### 3 语义分析与字节码生成
语义分析的任务：
对结构上正确的源程序进行上下文审查，例如类型审查；

![](http://wx4.sinaimg.cn/mw1024/9c96fc9cly1g6lh157pajj20vp04xdg2.jpg)

如上右图，三个运算都可以构成结构正确的语法树，但是显然只有第一种可以通过编译。

##### 3.1 标注检查
标注检查步骤检查的内容包括诸如变量使用前是否已被声明、变量与赋值之间的数据类型是否能够匹配等。 
优化：常量折叠，下面两种写法并不会增加程序运行期哪怕一个CPU指令
int a = 1 + 2
int a = 3

##### 3.2 数据及控制流分析
数据及控制流分析是对程序上下文逻辑更进一步的验证，它可以检查出诸如程序局部变量在使用前是否有赋值、方法的每条路径是否都有返回值、是否所有的受查异常都被正确处理了等问题。

##### 3.3 解语法糖
语法糖是编程语言提供的一种语法，他并不会对语言的功能有实质性改进，但是他们可以提高效率，或者提高严谨性，或者减少出错的机会，在javac编译时，会用desuger方法来解语法糖。但是，程序员也要看清语法糖背后程序代码的真实面目，通过反编译class文件，可以看到在剥离了语法糖衣背后的代码。

- 泛型与类型擦除
- 自动拆装箱与遍历循环
- 条件编译

##### 4. 生成字节码
字节码生成是Javac编译过程的最后一个阶段。字节码生成阶段不仅仅是把前面各个步骤所生成的信息（语法树、符号表）转化成字节码写到磁盘中，编译器还进行了少量的代码添加和转换工作。
例如生产实例构造器<init>()和类构造器<clinit>()就是在这个阶段添加到语法树中的。

### 晚期（运行期）优化

#### 解释器和编译器
大部分的Java虚拟机都是解释器和编译器共存的，两者各有优势：
- 当程序需要迅速启动和执行的时候，解释器可以首先发挥作用，省去编译的时间立即执行。
- 在程序运行后，随着时间的推移，编译器逐渐发挥作用，把越来越多的代码编译成本地代码之后。获取更高的执行效率。

逆优化：当编译器使用了激进的优化策略，在运行中发现不适用时，可以退回到解释状态继续使用解释器执行。

因此，在整个虚拟机执行过程中，解释器和编译器经常配合工作。

#### JIT编译器(即时编译器)
在HotSpot虚拟机当中，Java程序最初是通过解释器进行解释执行的，当虚拟机发现某个方法或代码块运行特别频繁时，会把它定义为“热点代码”。并且把这些代码编译成与本地平台相关的机器码，并进行各种层次的优化，以提高执行效率。完成这个过程的编译器被称为JIT编译器。

##### HotSpot中的JIT编译器
HotSpot虚拟机中内置了多个即时编译器，分别称为Client Compiler和Server Compiler，或者简称为C1编译器和C2编译器。不同在于C1只进行简单可靠的优化，C2会进行尽可能高效的优化。
java -version命令可以查看编译模式，有解释/编译/混合三种模式。编译模式下解释器也会在无法编译时介入执行。
解释器也会帮编译器在运行时搜集性能监控信息。
JDK1.6加入了分层编译策略，根据编译器编译优化的规模和耗时，划分处不同的编译层次。
- 第0层，程序解释执行，解释器不开启性能监控功能（Profiling），可触发第1层编译。
- 第1层，也称为C1编译，将字节码编译为本地代码，进行简单、可靠的优化，如有必要将加入性能监控的逻辑。
- 第2层（或2层以上），也称为C2编译，也是将字节码编译为本地代码，但是会启用一些编译耗时较长的优化，甚至会根据性能监控信息进行一些不可靠的激进优化。 

分层编译后，Client Compiler和Server Compiler将会同时工作，许多代码都可能会被多次编译，用Client Compiler获取更高的编译速度，

#### 编译对象和触发条件
上文中提到，运行中即时编译器会对“热点代码”进行编译，“热点代码”有两类：
- 被多次调用的方法
- 被多次执行的循环体

对于第一种情况，编译器理所当然会以整个方法作为编译对象，这种编译也是虚拟机中标准的JIT编译方式，而对于后一种情况，尽管编译动作是由循环体所触发的，但编译器依然会以整个方法（而不是单独的循环体）作为编译对象。这种编译方式因为编译发生在方法执行过程中，因此形象地称为**栈上替换（On Stack Replacement**，简称为**OSR编译**，即方法栈帧还在栈上，方法就被替换了）。

#### 热点探测计数器
在HotSpot虚拟机中使用的是—基于计数器的热点探测方法，因此它为每个方法准备了两类计数器：**方法调用计数器**（Invocation Counter）和 **回边计数器**（Back Edge Counter）
方法调用计数器在一定时间内达不到阈值，会对计数器的值进行减半，称为**半衰周期**，回边计数器没有衰减。

#### 编译优化技术

##### 公共子表达式消除
如果一个表达式E已经计算过了，而且从先前的计算到现在所有表达式中的值都没有变化，那么这个E就成为了公共子表达式。对于这种表达式，没有比较再次计算，直接使用前结果代替就行。

int d = (c * b) * 12 + a + (a + b * c)

编译器检测到“b*c”和“c*b”式一样的的表达式，所以可能被优化为

int d = **E** * 12 + a + (a + **E**)

编译器还可能会进行代数化简操作

int d = **E** * 13 + a * 2

##### 数组范围检查消除
数组边界检查消除（Array Bounds Checking Elimination）是即时编译器中的一项语言相关的经典优化技术。Java访问数组的时候系统将会自动进行上下界的范围检查，但对于虚拟机的执行子系统来说，每次数组元素的读写都带有一次隐含的条件判定操作，对于拥有大量数组访问的程序代码，这无疑也是一种性能负担。
数组边界检查是必须做的，但数组边界检查在某些情况下可以简化。例如数组下标是一个常量，如foo[3]，只要在编译期根据数据流分析来确定foo.length的值，并判断下标“3”没有越界，执行的时候就无须判断了。再例如数组访问发生在循环之中，并且使用循环变量来进行数组访问，如果编译器只要通过数据流分析就可以判定循环变量的取值范围永远在区间[0, foo.length)之内，那在整个循环中就可以把数组的上下界检查消除掉，这可以节省很多次的条件判断操作。
与语言相关的其他消除操作还有自动装箱消除（Autobox Elimination）、安全点消除（Safepoint Elimination）、消除反射（Dereflection）等。

##### 方法内联
方法内联是编译器最重要的优化手段之一，除了消除方法调用成本，也可以为其他优化手段建立良好的基础，如

![](http://wx2.sinaimg.cn/mw1024/9c96fc9cly1g6lhi7xeabj20db05b0ss.jpg)

图中两个方法事实上都是无用代码，但单独看都发现不了任何“DeadCode”，但是做了方法内联之后就很容易检测出来。

##### 逃逸分析
逃逸分析的基本行为就是分析对象动态作用域：当一个对象在方法中被定义后，它可能被外部方法所引用，称为方法逃逸。甚至还有可能被外部线程访问到，譬如赋值给类变量或可以在其他线程中访问的实例变量，称为线程逃逸。
如果证明一个对象不会逃逸到方法或线程外，那么就能进行一些非常高效的优化策略：
- 栈上分配
栈上分配就是把方法中的变量和对象分配到栈上，方法执行完后自动销毁，而不需要垃圾回收的介入，从而提高系统性能。



- 同步消除
线程同步本身比较耗，如果确定一个对象不会逃逸出线程，无法被其它线程访问到，那该对象的读写就不会存在竞争，对这个变量的同步措施就可以消除掉。单线程中是没有锁竞争。（锁和锁块内的对象不会逃逸出线程就可以把这个同步块取消）


- 标量替换
Java虚拟机中的原始数据类型（int，long等数值类型以及reference类型等）都不能再进一步分解，它们就可以称为标量。相对的，如果一个数据可以继续分解，那它称为聚合量，Java中最典型的聚合量是对象。如果逃逸分析证明一个对象不会被外部访问，并且这个对象是可分解的，那程序真正执行的时候将可能不创建这个对象，而改为直接创建它的若干个被这个方法使用到的成员变量来代替。拆散后的变量便可以被单独分析与优化，
可以各自分别在栈帧或寄存器上分配空间，原本的对象就无需整体分配空间了。

