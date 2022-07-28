# Java八股文整理版

**<u>要用自己的语言去诠释问题，不仅仅是背，最好有些自己的理解</u>**

## Java基础

### 1.Java8新特性

- **Lambda 表达式** − Lambda 允许把函数作为一个方法的参数（函数作为参数传递到方法中）。
- **方法引用** − 方法引用提供了非常有用的语法，可以直接引用已有Java类或对象（实例）的方法或构造器。与lambda联合使用，方法引用可以使语言的构造更紧凑简洁，减少冗余代码。
- **默认方法** − 默认方法就是一个在接口里面有了一个实现的方法。
- **新工具** − 新的编译工具，如：Nashorn引擎 jjs、 类依赖分析器jdeps。
- **Stream API** −新添加的Stream API（java.util.stream） 把真正的函数式编程风格引入到Java中。
- **Date Time API** − 加强对日期与时间的处理。
- **Optional 类** − Optional 类已经成为 Java 8 类库的一部分，用来解决空指针异常。
- **Nashorn, JavaScript 引擎** − Java 8提供了一个新的Nashorn javascript引擎，它允许我们在JVM上运行特定的javascript应用。



### 2.反射相关问题

**反射就是在运行时才知道要操作的类是什么，并且可以在运行时获取类的完整构造，并调用对应的方法。**

```Java
public class Apple {

    private int price;

    public int getPrice() {
        return price;
    }

    public void setPrice(int price) {
        this.price = price;
    }

    public static void main(String[] args) throws Exception{
        //正常的调用
        Apple apple = new Apple();
        apple.setPrice(5);
        System.out.println("Apple Price:" + apple.getPrice());
        //使用反射调用
        Class clz = Class.forName("com.chenshuyi.api.Apple");
        Method setPriceMethod = clz.getMethod("setPrice", int.class);
        Constructor appleConstructor = clz.getConstructor();
        Object appleObj = appleConstructor.newInstance();
        setPriceMethod.invoke(appleObj, 14);
        Method getPriceMethod = clz.getMethod("getPrice");
        System.out.println("Apple Price:" + getPriceMethod.invoke(appleObj));
    }
}
```



#### 2.1Class.forName和classloader.loadClass的区别

（1）Class.forName除了将类的.class文件加载到jvm中之外，还会对类进行解释，执行类中的static块。

（2）而classloader只干一件事情，就是将.class文件加载到jvm中，不会执行static中的内容，只有在newInstance才会去执行static块。



#### 2.2.如何创建对象

1. 使用new关键字
2. Class对象的newInstance()方法
3. 构造函数对象的newInstance()方法
4. 对象反序列化
5. Object对象的clone()方法



#### 2.3使用反射获取一个对象的步骤：

- 获取类的 Class 对象实例

```
Class clz = Class.forName("com.zhenai.api.Apple");
```

- 根据 Class 对象实例获取 Constructor 对象

```
Constructor appleConstructor = clz.getConstructor();
```

- 使用 Constructor 对象的 newInstance 方法获取反射类对象

```
Object appleObj = appleConstructor.newInstance();
```

而如果要调用某一个方法，则需要经过下面的步骤：

- 获取方法的 Method 对象

```
Method setPriceMethod = clz.getMethod("setPrice", int.class);
```

- 利用 invoke 方法调用方法

```Java
setPriceMethod.invoke(appleObj, 14);
```





#### 2.4反射源码相关

JDK 的 invoke 方法到底做了些什么。

进入 Method 的 invoke 方法我们可以看到，一开始是进行了一些权限的检查，最后是调用了 MethodAccessor 类的 invoke 方法进行进一步处理。

其实 MethodAccessor 是一个接口，定义了方法调用的具体操作，所以要找他的实现类

而要看 ma.invoke() 到底调用的是哪个类的 invoke 方法，则需要看看 MethodAccessor 对象返回的到底是哪个类对象，所以我们需要进入 acquireMethodAccessor() 方法中看看。

从 acquireMethodAccessor() 方法我们可以看到，代码先判断是否存在对应的 MethodAccessor 对象，如果存在那么就复用之前的 **MethodAccessor** 对象，否则调用 ReflectionFactory 对象的 newMethodAccessor 方法生成一个 MethodAccessor 对象。

在 ReflectionFactory 类的 newMethodAccessor 方法里，我们可以看到首先是生成了一个 NativeMethodAccessorImpl 对象，再这个对象作为参数调用 DelegatingMethodAccessorImpl 类的构造方法。

这里的实现是使用了代理模式，将 NativeMethodAccessorImpl 对象交给 DelegatingMethodAccessorImpl 对象代理。我们查看 DelegatingMethodAccessorImpl 类的构造方法可以知道，其实是将 **NativeMethodAccessorImpl** 对象赋值给 DelegatingMethodAccessorImpl 类的 delegate 属性。

所以说ReflectionFactory 类的 newMethodAccessor 方法最终返回 **DelegatingMethodAccessorImpl** 类对象。所以我们在前面的 ma.invoke() 里，其将会进入 DelegatingMethodAccessorImpl 类的 invoke 方法中。

而在 NativeMethodAccessorImpl 的 invoke 方法里，其会判断调用次数是否超过阀值（numInvocations）。如果超过该阀值，那么就会生成另一个MethodAccessor 对象，并将原来 DelegatingMethodAccessorImpl 对象中的 delegate 属性指向最新的 MethodAccessor 对象。

到这里，其实我们可以知道 MethodAccessor 对象其实就是具体去生成反射类的入口。通过查看源码上的注释，我们可以了解到 MethodAccessor 对象的一些设计信息。

Native 版本一开始启动快，但是随着运行时间边长，速度变慢。Java 版本一开始加载慢，但是随着运行时间边长，速度变快。正是因为两种存在这些问题，所以第一次加载的时候我们会发现使用的是 NativeMethodAccessorImpl 的实现，而当反射调用次数超过 15 次之后，则使用 MethodAccessorGenerator 生成的 MethodAccessorImpl 对象去实现反射。







### 3.深拷贝与浅拷贝

浅拷贝是按位拷贝对象，它会创建一个新对象，这个对象有着原始对象属性值的一份精确拷贝。

- 如果属性是基本类型，拷贝的就是基本类型的值；如果属性是内存地址（引用类型），拷贝的就是内存地址 ，因此如果其中一个对象改变了这个地址，就会影响到另一个对象。

深拷贝会拷贝所有的属性,并拷贝属性指向的动态分配的内存。当对象和它所引用的对象一起拷贝时即发生深拷贝。深拷贝相比于浅拷贝速度较慢并且花销较大。

序列化属于深拷贝





### 4.关键字相关

#### 4.1final关键字

**1、修饰类**

当用final修饰一个类时，表明这个类不能被继承。也就是说，如果一个类你永远不会让他被继承，就可以用final进行修饰。final类中的成员变量可以根据需要设为final，但是要注意final类中的所有成员方法都会被隐式地指定为final方法。

在使用final修饰类的时候，要注意谨慎选择，除非这个类真的在以后不会用来继承或者出于安全的考虑，尽量不要将类设计为final类。

**2、修饰方法**

“使用final方法的原因有两个。第一个原因是把方法锁定，以防任何继承类修改它的含义；第二个原因是效率。在早期的Java实现版本中，会将final方法转为内嵌调用。但是如果方法过于庞大，可能看不到内嵌调用带来的任何性能提升。在最近的Java版本中，不需要使用final方法进行这些优化了。“

**3、修饰变量**

final成员变量表示常量，只能被赋值一次，赋值后值不再改变。

当final修饰一个基本数据类型时，表示该基本数据类型的值一旦在初始化后便不能发生变化；如果final修饰一个引用类型时，则在对其初始化之后便不能再让其指向其他对象了，但该引用所指向的对象的内容是可以发生变化的。本质上是一回事，因为引用的值是一个地址，final要求值，即地址的值不发生变化。

final修饰一个成员变量（属性），必须要显示初始化。这里有两种初始化方式，一种是在变量声明的时候初始化；第二种方法是在声明变量的时候不赋初值，但是要在这个变量所在的类的所有的构造函数中对这个变量赋初值。

当函数的参数类型声明为final时，说明该参数是只读型的。即你可以读取使用该参数，但是无法改变该参数的值。



#### 4.2static关键字

“static方法就是没有this的方法。在static方法内部不能调用非静态方法，反过来是可以的。而且可以在没有创建任何对象的前提下，仅仅通过类本身来调用static方法。这实际上正是static方法的主要用途。”

1）static方法

　　static方法一般称作静态方法，由于静态方法不依赖于任何对象就可以进行访问，因此对于静态方法来说，是没有this的，因为它不依附于任何对象，既然都没有对象，就谈不上this了。并且由于这个特性，在静态方法中不能访问类的非静态成员变量和非静态成员方法，因为非静态成员方法/变量都是必须依赖具体的对象才能够被调用。

2）static变量

　　static变量也称作静态变量，静态变量和非静态变量的区别是：静态变量被所有的对象所共享，在内存中只有一个副本，它当且仅当在类初次加载时会被初始化。而非静态变量是对象所拥有的，在创建对象的时候被初始化，存在多个副本，各个对象拥有的副本互不影响。

　　static成员变量的初始化顺序按照定义的顺序进行初始化。

3）static代码块

　　static关键字还有一个比较关键的作用就是 用来形成静态代码块以优化程序性能。static块可以置于类中的任何地方，类中可以有多个static块。在类初次被加载的时候，会按照static块的顺序来执行每个static块，并且只会执行一次。





### 5.String底层相关

#### 5.1**String为什么是final**

​	**1.为了实现字符串池**

​    **2.为了线程安全**

​    **3.为了实现String可以创建HashCode不可变性**



#### 5.2主要变量与实现接口

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    
/** The value is used for character storage. */
    private final char value[];

    /** Cache the hash code for the string */
    private int hash; // Default to 0
　　public static final Comparator<String> CASE_INSENSITIVE_ORDER        = new CaseInsensitiveComparator();
```

length()，isEmpty()，charAt()这些方法其实就是在内部调用数组的方法。

equals直接比较value数组一个个比较



#### 5.3如何输出一个某种编码的字符串？

```Java
public String translate (String str) {
        
        String tempStr = "";
        
        try {
            tempStr = new String(str.getBytes("ISO-8859-1"), "GBK");
            tempStr = tempStr.trim();
        }catch (Exception e) {
            System.err.println(e.getMessage());
        }
        return tempStr;
        }
```





### 6.为什么重写equals方法要重写hashcode方法

 如果一个只重写了**equals(比较所有属性是否相等)**的类 new 出了两个**属性相同的对象**。这时可以得到的信息是这个属性相同的对象地址肯定不同，但是equals是true，hashCode返回的是不相等的(一般不会出现hash碰撞)。

也就是说这个类对象违背了Java对于两个对象相等的约定。违背约定的原因是 **可靠的equals判断两个对象是相等的，但是他们两个的散列码确是不相等的。**





### 7.Java定时任务

JDK 自带的定时器实现

- Timer类
  这个类允许你调度一个java.util.TimerTask任务。主要有以下几个方法：

1. schedule(TimerTask task, long delay) 延迟 delay 毫秒 执行
2. schedule(TimerTask task, Date time) 特定時間執行
3. schedule(TimerTask task, long delay, long period) 延迟 delay 执行并每隔period 执行一次

Quartz 定时器实现

简单地创建一个org.quarz.Job接口的Java类，Job接口包含唯一的方法：

```java
public void execute(JobExecutionContext context) throws JobExecutionException;
```

在Job接口实现类里面，添加需要的逻辑到execute()方法中。配置好Job实现类并设定好调度时间表（Trigger），Quartz就会自动在设定的时间调度作业执行execute()。

整合了Quartz的应用程序可以重用不同事件的作业，还可以为一个事件组合多个作业。Quartz通过属性文件来配置JDBC事务的数据源、全局作业、触发器侦听器、插件、线程池等等。





### 8.AIO BIO NIO

**1.BIO (同步阻塞I/O模式)**

数据的读取写入必须阻塞在一个线程内等待其完成。

这里使用那个经典的烧开水例子，这里假设一个烧开水的场景，有一排水壶在烧开水，BIO的工作模式就是， 叫一个线程停留在一个水壶那，直到这个水壶烧开，才去处理下一个水壶。但是实际上线程在等待水壶烧开的时间段什么都没有做。

**2.NIO（同步非阻塞）**

同时支持阻塞与非阻塞模式，但这里我们以其同步非阻塞I/O模式来说明，那么什么叫做同步非阻塞？如果还拿烧开水来说，NIO的做法是叫一个线程不断的轮询每个水壶的状态，看看是否有水壶的状态发生了改变，从而进行下一步的操作。

**3.AIO （异步非阻塞I/O模型）**

异步非阻塞与同步非阻塞的区别在哪里？异步非阻塞无需一个线程去轮询所有IO操作的状态改变，在相应的状态改变后，系统会通知对应的线程来处理。对应到烧开水中就是，为每个水壶上面装了一个开关，水烧开之后，水壶会自动通知我水烧开了。

**4.同步与异步的区别**

- 同步

发送一个请求，等待返回，再发送下一个请求，同步可以避免出现死锁，脏读的发生。

- 异步

发送一个请求，不等待返回，随时可以再发送下一个请求，可以提高效率，保证并发。

**5.阻塞和非阻塞**

- **阻塞**

传统的IO流都是阻塞式的。也就是说，当一个线程调用read()或者write()方法时，该线程将被阻塞，直到有一些数据读读取或者被写入，在此期间，该线程不能执行其他任何任务。在完成网络通信进行IO操作时，由于线程会阻塞，所以服务器端必须为每个客户端都提供一个独立的线程进行处理，当服务器端需要处理大量的客户端时，性能急剧下降。

- **非阻塞**

JavaNIO是非阻塞式的。当线程从某通道进行读写数据时，若没有数据可用时，该线程会去执行其他任务。线程通常将非阻塞IO的空闲时间用于在其他通道上执行IO操作，所以单独的线程可以管理多个输入和输出通道。因此NIO可以让服务器端使用一个或有限几个线程来同时处理连接到服务器端的所有客户端。

**6.BIO、NIO、AIO适用场景**

- BIO方式适用于连接数目比较小且固定的架构，这种方式对服务器资源要求比较高，并发局限于应用中，JDK1.4以前的唯一选择。
- NIO方式适用于连接数目多且连接比较短（轻操作）的架构，比如聊天服务器，并发局限于应用中，编程比较复杂。
- AIO方式使用于连接数目多且连接比较长（重操作）的架构，比如相册服务器，充分调用OS参与并发操作，编程比较复杂，JDK7开始支持。

**NIO的3个核心概念**

NIO重点是把Channel（通道），Buffer（缓冲区），Selector（选择器）三个类之间的关系弄清楚。

**1.缓冲区Buffer**

Buffer是一个对象。它包含一些要写入或者读出的数据。在面向流的I/O中，可以将数据写入或者将数据直接读到Stream对象中。

在NIO中，所有的数据都是用缓冲区处理。这也就本文上面谈到的IO是面向流的，NIO是面向缓冲区的。

缓冲区实质是一个数组，通常它是一个字节数组（ByteBuffer），也可以使用其他类的数组。但是一个缓冲区不仅仅是一个数组，缓冲区提供了对数据的结构化访问以及维护读写位置（limit）等信息。

最常用的缓冲区是ByteBuffer，一个ByteBuffer提供了一组功能于操作byte数组。除了ByteBuffer，还有其他的一些缓冲区，事实上，每一种Java基本类型（除了Boolean）都对应一种缓冲区，具体如下：

- ByteBuffer：字节缓冲区
- CharBuffer:字符缓冲区
- ShortBuffer：短整型缓冲区
- IntBuffer：整型缓冲区
- LongBuffer:长整型缓冲区
- FloatBuffer：浮点型缓冲区
- DoubleBuffer：双精度浮点型缓冲区

**2.通道Channel**

Channel是一个通道，可以通过它读取和写入数据，他就像自来水管一样，网络数据通过Channel读取和写入。

通道和流不同之处在于通道是双向的，流只是在一个方向移动，而且通道可以用于读，写或者同时用于读写。

因为Channel是全双工的，所以它比流更好地映射底层操作系统的API，特别是在UNIX网络编程中，底层操作系统的通道都是全双工的，同时支持读和写。

Channel有四种实现：

- FileChannel:是从文件中读取数据。
- DatagramChannel:从UDP网络中读取或者写入数据。
- SocketChannel:从TCP网络中读取或者写入数据。
- ServerSocketChannel:允许你监听来自TCP的连接，就像服务器一样。每一个连接都会有一个SocketChannel产生。

**3.多路复用器Selector**

Selector选择器可以监听多个Channel通道感兴趣的事情(read、write、accept(服务端接收)、connect，实现一个线程管理多个Channel，节省线程切换上下文的资源消耗。Selector只能管理非阻塞的通道，FileChannel是阻塞的，无法管理。

**关键对象**

- Selector：选择器对象，通道注册、通道监听对象和Selector相关。
- SelectorKey：通道监听关键字，通过它来监听通道状态。

**监听注册**

监听注册在Selector

> socketChannel.register(selector, SelectionKey.OP_READ);

**监听的事件有**

- OP_ACCEPT: 接收就绪，serviceSocketChannel使用的
- OP_READ: 读取就绪，socketChannel使用
- OP_WRITE: 写入就绪，socketChannel使用
- OP_CONNECT: 连接就绪，socketChannel使用





### 9.Java基础类型以及其包装类空指针问题

| 关键字  | 类型   | 位数 (8位一字节) | 取值范围(表示范围)            |
| ------- | ------ | ---------------- | ----------------------------- |
| byte    | 整型   | 8                | -2^7 ~ 2^7-1                  |
| short   | 整型   | 16               | -2^15 ~ 2^15-1                |
| int     | 整型   | 32               | -2^31 ~ 2^31-1                |
| long    | 整型   | 64               | -2^63 ~ 2^63-1                |
| float   | 浮点数 | 32               | 3.402823e+38 ~ 1.401298e-45   |
| double  | 浮点数 | 64               | 1.797693e+308~ 4.9000000e-324 |
| char    | 文本型 | 16               | 0 ~ 2^16-1                    |
| boolean | 布尔值 | 32/8             | true/false                    |

- 对于万物皆对象的java,为什么会存在基本类型?因为java产生对象，一般是需在堆创建维护，再通过栈的引用来使用，但是对于简单的小的变量，需要在堆创建再使用太麻烦了
- 为什么会有包装类
  - 包装类将基本类型包装起来，使其具有对象的性质，可以添加属性和方法，丰富基本类型的操作
  - 对于泛型编程，或使用collection集合，需要包装类。因为ArrayList，HashMap的泛型无法指定基本类型
  - 区别，基本类型可以直接声明使用，包装类需要在堆创建，再通过引用使用；基本类型默认初始值，int为0，boolean则是true/false，且无法赋值为null；而包装类默认初始值是null
- 需要注意的点：Byte、Int、Short、Long直接赋值（或使用valueOf）如Integer x = value(value 在-128 ~ 127)是直接引用常量池里的对象，此时对象比较 == 和 equals 都为true ；Character声明值则在0~127 是引用常量池对象



### 10.Java元注解

- @Target
- @Retention
- @Documented
- @Inherited

Target注解的作用是：描述注解的使用范围(即被修饰的注解可以用在什么地方).

Reteniton注解的作用是：描述注解保留的时间范围(即：被描述的注解在它所修饰的类中可以被保留到何时).

Documented注解的作用是：描述在使用 javadoc 工具为类生成帮助文档时是否要保留其注解信息。

Inherited注解的作用是：使被它修饰的注解具有继承性（如果某个类使用了被@Inherited修饰的注解，则其子类将自动具有该注解）。



### 11.正则表达式

```java
public static void main(String[] args) {
    // 要验证的字符串
    String str = "service@xsoftlab.net";
    // 邮箱验证规则
    String regEx = "[a-zA-Z_]{1,}[0-9]{0,}@(([a-zA-z0-9]-*){1,}\\.){1,3}[a-zA-z\\-]{1,}";
    // 编译正则表达式
    Pattern pattern = Pattern.compile(regEx);
    // 忽略大小写的写法
    // Pattern pat = Pattern.compile(regEx, Pattern.CASE_INSENSITIVE);
    Matcher matcher = pattern.matcher(str);
    // 字符串是否与正则表达式相匹配
    boolean rs = matcher.matches();
    System.out.println(rs);
}
```



### 12.Java 重写(Override)与重载(Overload)

重写(Override)

重写是子类对父类的允许访问的方法的实现过程进行重新编写, 返回值和形参都不能改变。**即外壳不变，核心重写！**

重写的好处在于子类可以根据需要，定义特定于自己的行为。 也就是说子类能够根据需要实现父类的方法。

方法的重写规则

- 参数列表与被重写方法的参数列表必须完全相同。
- 返回类型与被重写方法的返回类型可以不相同，但是必须是父类返回值的派生类（java5 及更早版本返回类型要一样，java7 及更高版本可以不同）。
- 访问权限不能比父类中被重写的方法的访问权限更低。例如：如果父类的一个方法被声明为 public，那么在子类中重写该方法就不能声明为 protected。
- 父类的成员方法只能被它的子类重写。
- 声明为 final 的方法不能被重写。
- 声明为 static 的方法不能被重写，但是能够被再次声明。
- 子类和父类在同一个包中，那么子类可以重写父类所有方法，除了声明为 private 和 final 的方法。
- 子类和父类不在同一个包中，那么子类只能够重写父类的声明为 public 和 protected 的非 final 方法。
- 重写的方法能够抛出任何非强制异常，无论被重写的方法是否抛出异常。但是，重写的方法不能抛出新的强制性异常，或者比被重写方法声明的更广泛的强制性异常，反之则可以。
- 构造方法不能被重写。
- 如果不能继承一个类，则不能重写该类的方法。

重载(Overload)

重载(overloading) 是在一个类里面，方法名字相同，而参数不同。返回类型可以相同也可以不同。

每个重载的方法（或者构造函数）都必须有一个独一无二的参数类型列表。

最常用的地方就是构造器的重载。

**重载规则:**

- 被重载的方法必须改变参数列表(参数个数或类型不一样)；
- 被重载的方法可以改变返回类型；
- 被重载的方法可以改变访问修饰符；
- 被重载的方法可以声明新的或更广的检查异常；
- 方法能够在同一个类中或者在一个子类中被重载。
- 无法以返回值类型作为重载函数的区分标准。



### 13.Java异常相关

![img](https://images0.cnblogs.com/i/288799/201406/051650515056805.jpg)

throws和thow关键字

　　1）throws出现在方法的声明中，表示该方法可能会抛出的异常，然后交给上层调用它的方法程序处理，允许throws后面跟着多个异常类型；

　　2）一般会用于程序出现某种逻辑时程序员主动抛出某种特定类型的异常。throw只会出现在方法体中，当方法在执行过程中遇到异常情况时，将异常信息封装为异常对象，然后throw出去。throw关键字的一个非常重要的作用就是 异常类型的转换（会在后面阐述道）。

　　throws表示出现异常的一种可能性，并不一定会发生这些异常；throw则是抛出了异常，执行throw则一定抛出了某种异常对象。两者都是消极处理异常的方式（这里的消极并不是说这种方式不好），只是抛出或者可能抛出异常，但是不会由方法去处理异常，真正的处理异常由此方法的上层调用处理。



### 14.接口和抽象类的区别

相同点：
（1）都不能被实例化
（2）接口的实现类或抽象类的子类都只有实现了接口或抽象类中的方法后才能实例化。

不同点：
（1）接口只有定义，不能有方法的实现，java 1.8中可以定义default方法体，而抽象类可以有定义与实现，方法可在抽象类中实现。
（2）实现接口的关键字为implements，继承抽象类的关键字为extends。一个类可以实现多个接口，但一个类只能继承一个抽象类。所以，使用接口可以间接地实现多重继承。
（3）接口强调特定功能的实现，而抽象类强调所属关系。
（4）接口成员变量默认为public static final，必须赋初值，不能被修改；其所有的成员方法都是public、abstract的。抽象类中成员变量默认default，可在子类中被重新定义，也可被重新赋值；抽象方法被abstract修饰，不能被private、static、synchronized和native等修饰，必须以分号结尾，不带花括号。
（5）接口被用于常用的功能，便于日后维护和添加删除，而抽象类更倾向于充当公共类的角色，不适用于日后重新对立面的代码修改。功能需要累积时用抽象类，不需要累积时用接口。



### 15.泛型相关问题

#### 1.extends与super

- extends上限通配符，用来限制类型的上限，只能传入本类和子类，add方法受阻，可以从一个数据类型里获取数据；
- super下限通配符，用来限制类型的下限，只能传入本类和父类，get方法受阻，可以把对象写入一个数据结构里；



### 16.Comparable和Comparator接口是干什么的？列出它们的区别。.

java提供了只包含一个compareTo()方法的Comparable接口。这个方法可以个给两个对象排序。具体来说，它返回负数，0，正数来表明已经存在的对象小于，等于，大于输入对象。
Java提供了包含compare()和equals()两个方法的Comparator接口。compare()方法用来给两个输入参数排序，返回负数，0，正数表明第一个参数是小于，等于，大于第二个参数。equals()方法需要一个对象作为参数，它用来决定输入参数是否和comparator相等。只有当输入参数也是一个comparator并且输入参数和当前comparator的排序结果是相同的时候，这个方法才返回true。

Comparable & Comparator 都是用来实现集合中元素的比较、排序的，只是 Comparable 是在集合内部定义的方法实现的排序，Comparator 是在集合外部实现的排序

### 17.内部类问题

- 成员内部类
- 静态内部类
- 局部内部类
- 匿名内部类

成员内部类：

定义在另一个类(外部类)的内部，而且与成员方法和属性平级叫成员内部类，......相当于外部类的非静态方法，如果被static修饰，就变成静态内部类了。

1. 成员内部类中不能存在static关键字，即，不能声明静态属性、静态方法、静态代码块等。【**非静态内部类也可以定义静态成员但需要同时有final关键词修饰，静态方法鉴于无法用final修饰，仍必须是在静态内部类 或者非内部类中定义。**】
2. 创建成员内部类的实例使用：外部类名.内部类名 实例名 = 外部类实例名.new 内部类构造方法(参数),可以理解为隐式地保存了一个引用，指向创建它的外部类对象。
3. 在成员内部类中访问外部类的成员方法和属性时使用：外部类名.this.成员方法/属性。
4. 内部类在编译之后生成一个单独的class文件，里面包含该类的定义，所以内部类中定义的方法和变量可以跟父类的方法和变量相同。例如上面定义的三个类的class文件分别是：MyTest.class、Outer.class和Outer$Inner.class三个文件。
5. 外部类无法直接访问成员内部类的方法和属性，需要通过内部类的一个实例来访问。
6. 与外部类平级的类继承内部类时，其构造方法中需要传入父类的实例对象。且在构造方法的第一句调用“外部类实例名.super(内部类参数)”。

静态内部类

使用static修饰的成员内部类叫静态内部类。

可以这样理解：与外部类同级的类，或者叫做外部类的静态成员。与成员内部类的对比如下：

| 说明                        | 成员内部类                                                   | 静态内部类                                                   |
| --------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 静态成员                    | 静态成员需同时有final关键词修饰                              | 可以                                                         |
| 静态方法                    | 不可定义                                                     | 可以                                                         |
| 访问外部类非static属性/方法 | 外部类名.this.成员方法/属性                                  | 不可以                                                       |
| 外部类访问内部类            | 需要通过内部类的一个实例来访问                               | 需要通过内部类的一个实例来访问                               |
| 创建实例                    | 外部类名.内部类名 实例名 = 外部类实例名.new 内部类构造方法(参数) | 外部类名.内部类名 实例名 = new 外部类名.内部类名(参数)       |
| 编译后的class文件           | 单独的class文件(so内部类中的方法和变量可以跟父类的方法和变量同名)，外部类$内部类.class | 单独的class文件(so内部类中的方法和变量可以跟父类的方法和变量同名)，外部类$内部类.class |
| 其他                        | 与外部类平级的类继承内部类时，其构造方法中需要传入父类的实例对象。且在构造方法的第一句调用“外部类实例名.super(内部类参数)” | 无                                                           |

局部内部类

定义在代码块、方法体内、作用域（使用花括号“{}”括起来的一段代码）内的类叫局部内部类。

1. 局部内部类只能在代码代码块、方法体内和作用域中使用（如创建对象和使用类对象等）
2. 局部内部类访问作用域内的局部变量，该局部变量需要使用final修饰。
3. 可以使用abstract修饰，声明为抽象类。

匿名内部类

1. 只能使用一次，创建实例之后，类定义会立即消失（想要多次使用就要用到反射的知识了）
2. 必须继承一个类（抽象的、非抽象的都可以）或者实现一个接口。如果父类（或者父接口）是抽象类，则匿名内部类必须实现其所有抽象方法。
3. 不能是抽象类，因为匿名内部类在定义之后，会立即创建一个实例。
4. 不能定义构造方法，匿名内部类没有类名，无法定义构造方法，但是，匿名内部类拥有与父类相同的所有构造方法。
5. 可以定义代码块，用于实例的初始化，但是不能定义静态代码块。
6. 可以定义新的方法和属性(不能使用static修饰)，但是无法显式的通过“实例名.方法名(参数)”的形式调用，因为使用new创建的是“上转型对象”（即父类声明指向子类对象）。
7. 是局部内部类，所以要符合局部内部类的要求。

| **说明**                    | **成员内部类**                                               | **匿名内部类**                     |
| --------------------------- | ------------------------------------------------------------ | ---------------------------------- |
| 静态成员                    | 静态成员需同时有final关键词修饰                              | 不可定义                           |
| 静态方法                    | 不可定义                                                     | 不可定义                           |
| 访问外部类非static属性/方法 | 外部类名.this.成员方法/属性                                  | 外部类名.this.成员方法/属性        |
| 外部类访问内部类            | 需要通过内部类的一个实例来访问                               | 需要通过内部类的一个实例来访问     |
| 创建实例                    | 外部类名.内部类名 实例名 = 外部类实例名.new 内部类构造方法(参数) | 如上：父类 实例名 = new 父类（）{} |
| 编译后的class文件           | 单独的class文件(so内部类中的方法和变量可以跟父类的方法和变量同名)，外部类$内部类.class | 单独的class文件，使用类$数字.class |
| 其他                        | 与外部类平级的类继承内部类时，其构造方法中需要传入父类的实例对象。且在构造方法的第一句调用“外部类实例名.super(内部类参数)” | 无                                 |







## Java集合类

### 1.hashmap相关问题

#### 1.1hashmap底层数据结构

JDK 1.8，底层是由“数组+链表+红黑树”组成，如下图，而在 JDK 1.8 之前是由“数组+链表”组成。主要是为了提升在 hash 冲突严重时（链表过长）的查找性能，使用链表的查找性能是 O(n)，而使用红黑树是 O(logn)。

对于插入，默认情况下是使用链表节点。当同一个索引位置的节点在新增后达到9个（阈值8）：如果此时数组长度大于等于 64，则会触发链表节点转红黑树节点（treeifyBin）；而如果数组长度小于64，则不会触发链表转红黑树，而是会进行扩容，因为此时的数据量还比较小。

对于移除，当同一个索引位置的节点在移除后达到 6 个，并且该索引位置的节点为红黑树节点，会触发红黑树节点转链表节点（untreeify）。

我们平时在进行方案设计时，必须考虑的两个很重要的因素是：时间和空间。对于 HashMap 也是同样的道理，简单来说，阈值为8是在时间和空间上权衡的结果（这 B 我装定了）。

红黑树节点大小约为链表节点的2倍，在节点太少时，红黑树的查找性能优势并不明显，付出2倍空间的代价作者觉得不值得。

理想情况下，使用随机的哈希码，节点分布在 hash 桶中的频率遵循泊松分布，按照泊松分布的公式计算，链表中节点个数为8时的概率为 0.00000006（跟大乐透一等奖差不多，中大乐透？不存在的），这个概率足够低了，并且到8个节点时，红黑树的性能优势也会开始展现出来，因此8是一个较合理的数字。

如果我们设置节点多于8个转红黑树，少于8个就马上转链表，当节点个数在8徘徊时，就会频繁进行红黑树和链表的转换，造成性能的损耗。





#### 1.2hashmap底层属性

除了用来存储我们的节点 table 数组外，HashMap 还有以下几个重要属性：1）size：HashMap 已经存储的节点个数；2）threshold：扩容阈值，当 HashMap 的个数达到该值，触发扩容。3）loadFactor：负载因子，扩容阈值 = 容量 * 负载因子。

在我们新建 HashMap 对象时， threshold 还会被用来存初始化时的容量。HashMap 直到我们第一次插入节点时，才会对 table 进行初始化，避免不必要的空间浪费。

默认初始容量是16。HashMap 的容量必须是2的N次方，HashMap 会根据我们传入的容量计算一个大于等于该容量的最小的2的N次方，例如传 9，容量为16。

```java
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

这5个公式会通过最高位的1，拿到2个1、4个1、8个1、16个1、32个1。当然，有多少个1，取决于我们的入参有多大，但我们肯定的是经过这5个计算，得到的值是一个低位全是1的值，最后返回的时候 +1，则会得到1个比n 大的 2 的N次方。开头的 cap - 1 就很简单了，这是为了处理 cap 本身就是 2 的N次方的情况。

计算索引位置的公式为：(n - 1) & hash，当 n 为 2 的 N 次方时，n - 1 为低位全是 1 的值，此时任何值跟 n - 1 进行 & 运算会等于其本身，达到了和取模同样的效果，实现了均匀分布。实际上，这个设计就是基于公式：x mod 2^n = x & (2^n - 1)，因为 & 运算比 mod 具有更高的效率。

16的原因主要是：16是2的N次方，并且是一个较合理的大小。如果用8或32，我觉得也是OK的。实际上，我们在新建 HashMap 时，最好是根据自己使用情况设置初始容量，这才是最合理的方案。

这个也是在时间和空间上权衡的结果。如果值较高，例如1，此时会减少空间开销，但是 hash 冲突的概率会增大，增加查找成本；而如果值较低，例如 0.5 ，此时 hash 冲突会降低，但是有一半的空间会被浪费，所以折衷考虑 0.75 似乎是一个合理的值。





#### 1.3hashmap插入流程

<img src="https://pic1.zhimg.com/80/v2-3c9aa56b20ea776bd0ccf1e4584bfff8_1440w.jpg" alt="img" style="zoom:50%;" />

拿到 key 的 hashCode，并将 hashCode 的高16位和 hashCode 进行异或（XOR）运算，得到最终的 hash 值。主要是为了在 table 的长度较小的时候，让高位也参与运算，并且不会有太大的开销。





#### 1.4hashmap扩容

<img src="https://pic1.zhimg.com/80/v2-b7e2b2cef64b2729cd767c8d4fd5da28_1440w.jpg" alt="img" style="zoom:50%;" />

计算新表的索引位置时，只取决于新表在高位多出来的这一位，而这一位的值刚好等于 oldCap。

因为只取决于这一位，所以只会存在两种情况：1）  (e.hash & oldCap) == 0 ，则新表索引位置为“原索引位置” ；2）(e.hash & oldCap) == 1，则新表索引位置为“原索引 + oldCap 位置”。





#### 1.5hashmap线程安全与死循环问题

HashMap 在并发下存在数据覆盖、遍历的同时进行修改会抛出 ConcurrentModificationException 异常等问题，JDK 1.8 之前还存在死循环问题。

导致死循环的根本原因是 JDK 1.7 扩容采用的是“头插法”，会导致同一索引位置的节点在扩容后顺序反掉。而 JDK 1.8 之后采用的是“尾插法”，扩容后节点顺序不会反掉，不存在死循环问题。





#### 1.6hashmap与其他集合类对比

<img src="https://pic3.zhimg.com/80/v2-c94466eaeb78f31a73b936da04bb45f6_1440w.jpg" alt="img" style="zoom:80%;" />



#### 1.7hashmap和hashtable的区别

1.HashMap和Hashtable不仅作者不同，而且连父类也是不一样的。HashMap是继承自AbstractMap类，而HashTable是继承自Dictionary类。不过它们都实现了同时实现了map、Cloneable（可复制）、Serializable（可序列化）这三个接口
2.Hashtable既不支持Null key也不支持Null value。Hashtable的put()方法的注释中有说明。

3.Hashtable是线程安全的，它的每个方法中都加入了Synchronize方法。在多线程并发的环境下，可以直接使用Hashtable，不需要自己为它的方法实现同步

4.Hashtable默认的初始大小为11，之后每次扩充，容量变为原来的2n+1。HashMap默认的初始化大小为16。之后每次扩充，容量变为原来的2倍。



#### 1.8HashMap的遍历方式

1.通过HashMap.entrySet()得到键值对集合；

2.通过HashMap.keySet()获得键的Set集合；

3.通过HashMap.values()得到“值”的集合





### 2.基础数据结构相关

#### 2.1.数组

数组（Array）是一种线性表数据结构。它用一组连续的内存空间，来存储一组具有相同类型的数据。

为什么数组要从 0 开始编号，而不是从 1 开始呢？

数组的下标是一种offset的概念, 而不是第几个的意思. 方便与机器对于地址的解析, 符合计算机的寻址方式.

从 1 开始编号，每次随机访问数组元素都多了一次减法运算，对于 CPU 来说，就是多了一次减法指令。

通过下面的寻址公式，计算出该元素存储的内存地址：

a[i]_address = base_address + i * data_type_size

优化删除与插入

如果数组中的数据是有序的，我们在某个位置插入一个新的元素时，就必须按照刚才的方法搬移 k 之后的数据。但是，如果数组中存储的数据并没有任何规律，数组只是被当作一个存储数据的集合。在这种情况下，如果要将某个数据插入到第 k 个位置，为了避免大规模的数据搬移，我们还有一个简单的办法就是，直接将第 k 位的数据搬移到数组元素的最后，把新的元素直接放入第 k 个位置。

我们可以先记录下已经删除的数据。每次的删除操作并不是真正地搬移数据，只是记录数据已经被删除。当数组没有更多空间存储数据时，我们再触发执行一次真正的删除操作，这样就大大减少了删除操作导致的数据搬移。

ArrayList 最大的优势就是可以将很多数组操作的细节封装起来。比如前面提到的数组插入、删除数据时需要搬移其他数据等。另外，它还有一个优势，就是支持动态扩容。



#### 2.2.链表

链表不需要连续空间存储，单链表、双向链表和循环链表。

链表中的数据并非连续存储的，所以无法像数组那样，根据首地址和下标，通过寻址公式就能直接计算出对应的内存地址，而是需要根据指针一个结点一个结点地依次遍历，直到找到相应的结点。

删除值等于给定值的结点对应的链表操作的总时间复杂度为 O(n)。

删除给定指针指向的结点。针对这种情况，单链表删除操作需要 O(n) 的时间复杂度，而双向链表只需要在 O(1) 的时间复杂度内就搞定了！

用链表实现LRU

\1. 如果此数据之前已经被缓存在链表中了，我们遍历得到这个数据对应的结点，并将其从原来的位置删除，然后再插入到链表的头部。2. 如果此数据没有在缓存链表中，又可以分为两种情况：如果此时缓存未满，则将此结点直接插入到链表的头部；如果此时缓存已满，则链表尾结点删除，将新的数据结点插入链表的头部。

用数组实现

因为是最简单的LRU，没有散列表，也没有新老生代，用数组来模拟LRU的思想我们可以借鉴MySQL的redo log环，用两个指针来表示当前数组的可用空间，分别是write pos来标记当前写入的位置，check point来标记清除数据的检查点。假设数组的长度为n，初始write pos和check point都是0，数组为空。当写入的时候write pos不断递增，这里有一个特殊操作，我们要让write pos和check point对n取模以实现环状的效果，当write pos追上check point的时候（第一次表现为write pos % n = 0，check point此时也为0），check point就向前推进某个step，这里是1，当然也可以设置更大的值，比如说 0.1 * n，在这个环上，write pos 到 check point的区间就表征了可用空间，check point到 write pos就代表写入的数据。



#### 2.3.栈

栈在函数调用中的应用；

栈在表达式求值中的应用；

栈在括号匹配中的应用；

我们在讲栈的应用时，讲到用函数调用栈来保存临时变量，为什么函数调用要用“栈”来保存临时变量呢？用其他数据结构不行吗？

其实，我们不一定非要用栈来保存临时变量，只不过如果这个函数调用符合后进先出的特性，用栈这种数据结构来实现，是最顺理成章的选择。 从调用函数进入被调用函数，对于数据来说，变化的是什么呢？是作用域。所以根本上，只要能保证每进入一个新的函数，都是一个新的作用域就可以。而要实现这个，用栈就非常方便。在进入被调用函数的时候，分配一段栈空间给这个函数的变量，在函数结束的时候，将栈顶复位，正好回到调用函数的作用域内。 ------引自评论区。 这里解释一下，最重要的是因为只能操作"栈顶元素"，在作用域的角度看，操作完一个作用域的再回到之前的作用域下，用栈保存临时变量则是最好的选择，其他的数组，链表等都能"违规地"进行随机访问。

那 JVM 里面的“栈”跟我们这里说的“栈”是不是一回事呢？如果不是，那它为什么又叫作“栈”呢？

内存中的堆栈和数据结构堆栈不是一个概念，可以说内存中的堆栈是真实存在的物理区，数据结构中的堆栈是抽象的数据存储结构。 内存空间在逻辑上分为三部分：代码区、静态数据区和动态数据区，动态数据区又分为栈区和堆区。 代码区：存储方法体的二进制代码。高级调度（作业调度）、中级调度（内存调度）、低级调度（进程调度）控制代码区执行代码的切换。 静态数据区：存储全局变量、静态变量、常量，常量包括final修饰的常量和String常量。系统自动分配和回收。 栈区：存储运行方法的形参、局部变量、返回值。由系统自动分配和回收。 堆区：new一个对象的引用或地址存储在栈区，指向该对象存储在堆区中的真实数据。



#### 2.4.队列

```java
// 用数组实现的队列
public class ArrayQueue {
  // 数组：items，数组大小：n
  private String[] items;
  private int n = 0;
  // head表示队头下标，tail表示队尾下标
  private int head = 0;
  private int tail = 0;

  // 申请一个大小为capacity的数组
  public ArrayQueue(int capacity) {
    items = new String[capacity];
    n = capacity;
  }

  // 入队
  public boolean enqueue(String item) {
    // 如果tail == n 表示队列已经满了
    if (tail == n) return false;
    items[tail] = item;
    ++tail;
    return true;
  }

  // 出队
  public String dequeue() {
    // 如果head == tail 表示队列为空
    if (head == tail) return null;
    // 为了让其他语言的同学看的更加明确，把--操作放到单独一行来写了
    String ret = items[head];
    ++head;
    return ret;
  }
}

   // 入队操作，将item放入队尾
  public boolean enqueue(String item) {
    // tail == n表示队列末尾没有空间了
    if (tail == n) {
      // tail ==n && head==0，表示整个队列都占满了
      if (head == 0) return false;
      // 数据搬移
      for (int i = head; i < tail; ++i) {
        items[i-head] = items[i];
      }
      // 搬移完之后重新更新head和tail
      tail -= head;
      head = 0;
    }
    
    items[tail] = item;
    ++tail;
    return true;
  }
```



#### 2.5Arrays.sort

查看了下Arrays.sort的源码，主要采用TimSort算法, 大致思路是这样的： 1 元素个数 < 32, 采用二分查找插入排序(Binary Sort) 2 元素个数 >= 32, 采用归并排序，归并的核心是分区(Run) 3 找连续升或降的序列作为分区，分区最终被调整为升序后压入栈 4 如果分区长度太小，通过二分插入排序扩充分区长度到分区最小阙值 5 每次压入栈，都要检查栈内已存在的分区是否满足合并条件，满足则进行合并 6 最终栈内的分区被全部合并，得到一个排序好的数组 Timsort的合并算法非常巧妙： 1 找出左分区最后一个元素(最大)及在右分区的位置 2 找出右分区第一个元素(最小)及在左分区的位置 3 仅对这两个位置之间的元素进行合并，之外的元素本身就是有序的





### 3.ConcurrentHashMap相关问题

#### 3.1ConcurrentHashMap锁加在了哪些地方？

Java7是由Segment数组实现的，一个Segment锁住几个HashEntry元素； 

  Java8是由synchronized锁住transient volatile **Node**<K,V>[] table 数组的一个元素(即头结点，锁粒度比Java7低)，还通过CAS(Unsafe对象)操作数据更新、插入等。



#### 3.2concurrenthashmap有啥优势，1.7，1.8区别？

ConcurrentHashMap是线程安全的，在jdk1.7是采用Segment+HashEntry的方式实现的，lock加在segment上， 1.7的size计算是先采用不加锁的方式，连续计算元素的个数，最多计算3次：

（1）如果前后两次计算结果相同，则说明计算出来的元素个数是准确的。

（2）如果前后两次计算结果不同，则给每个Segment进行加锁，再计算一次元素的个数。

1.8中放弃了Segment臃肿的设计，取而代之的是采用Node+CAS+synchronized来保证并发安全进行实现，1.8中使用一个volatile类型的变量baseCount记录元素的个数，当插入或删除数据时，会通过addCount()方法更新baseCount，通过累加baseCount和CounterCell数组中的数量，即可得到元素的总个数。



#### 3.3成员变量

table：默认为null，初始化发生在第一次插入操作，默认大小为16的数组，用来存储Node节点数据，扩容时大小总是2的幂次方。
nextTable：默认为null，扩容时新生成的数组，其大小为原数组的两倍。
sizeCtl ：默认为0，用来控制table的初始化和扩容操作，具体应用在后续会体现出来。
-1 代表table正在初始化
-N 表示有N-1个线程正在进行扩容操作
其余情况：
1、如果table未初始化，表示table需要初始化的大小。
2、如果table初始化完成，表示table的容量，默认是table大小的0.75倍，居然用这个公式算0.75（n - (n >>> 2)）。
Node：保存key，value及key的hash值的数据结构。
其中value和next都用volatile修饰，保证并发的可见性。

- ForwardingNode：一个特殊的Node节点，hash值为-1，其中存储nextTable的引用。
  只有table发生扩容的时候，ForwardingNode才会发挥作用，作为一个占位符放在table中表示当前节点为null或则已经被移动。



#### 3.4实例初始化

实例化ConcurrentHashMap时倘若声明了table的容量，在初始化时会根据参数调整table大小，==确保table的大小总是2的幂次方==。默认的table大小为16.

table的初始化操作回延缓到第一put操作再进行，并且初始化只会执行一次。



#### 3.5put过程

假设table已经初始化完成，put操作采用==CAS+synchronized==实现并发插入或更新操作：
- 当前bucket为空时，使用CAS操作，将Node放入对应的bucket中。
- 出现hash冲突，则采用synchronized关键字。倘若当前hash对应的节点是链表的头节点，遍历链表，若找到对应的node节点，则修改node节点的val，否则在链表末尾添加node节点；倘若当前节点是红黑树的根节点，在树结构上遍历元素，更新或增加节点。
- 倘若当前map正在扩容f.hash == MOVED， 则跟其他线程一起进行扩容

hash算法与定位索引与hashmap相同，获取table对应的索引元素f时，采用Unsafe.getObjectVolatie()来获取，而不是直接用table[index]的原因跟ConcurrentHashMap的弱一致性有关。在java内存模型中，我们已经知道每个线程都有一个工作内存，里面存储着table的副本，虽然table是volatile修饰的，但不能保证线程每次都拿到table中的最新元素，Unsafe.getObjectVolatile可以直接获取指定内存的数据，保证了每次拿到数据都是最新的。



#### 3.6table 扩容

当table的元素数量达到容量阈值sizeCtl，需要对table进行扩容：
\- 构建一个nextTable，大小为table两倍
\- 把table的数据复制到nextTable中。
在扩容过程中，依然支持并发更新操作；也支持并发插入。

如何在扩容时，并发地复制与插入？
1. 遍历整个table，当前节点为空，则采用CAS的方式在当前位置放入fwd
2. 当前节点已经为fwd(with hash field “MOVED”)，则已经有有线程处理完了了，直接跳过 ，这里是控制并发扩容的核心
3. 当前节点为链表节点或红黑树，重新计算链表节点的hash值，移动到nextTable相应的位置（构建了一个反序链表和顺序链表，分别放置在i和i+n的位置上）。移动完成后，用Unsafe.putObjectVolatile在tab的原位置赋为为fwd, 表示当前节点已经完成扩容。



### 4.ArrayList相关问题

#### 4.1快速失败(fail-fast)和安全失败(fail-safe)的区别是什么？

Iterator的安全失败是基于对底层集合做拷贝，因此，它不受源集合上修改的影响。java.util包下面的所有的集合类都是快速失败的，而java.util.concurrent包下面的所有的类都是安全失败的。快速失败的迭代器会抛出ConcurrentModificationException异常，而安全失败的迭代器永远不会抛出这样的异常。

当多个线程对同一个集合进行操作的时候，某线程访问集合的过程中，该集合的内容被其他线程所改变(即其它线程通过add、remove、clear等方法，改变了modCount的值)；这时，就会抛出ConcurrentModificationException异常，产生fail-fast事件。

采用安全失败机制的集合容器，在遍历时不是直接在集合内容上访问的，而是先复制原有集合内容，在拷贝的集合上进行遍历。

原理：由于迭代时是对原集合的拷贝进行遍历，所以在遍历过程中对原集合所作的修改并不能被迭代器检测到，所以不会触发Concurrent Modification Exception。

缺点：基于拷贝内容的优点是避免了Concurrent Modification Exception，**但同样地，**迭代器并不能访问到修改后的内容，即：迭代器遍历的是开始遍历那一刻拿到的集合拷贝，在遍历期间原集合发生的修改迭代器是不知道的。

场景：java.util.concurrent包下的容器都是安全失败，可以在多线程下并发使用，并发修改。





#### 4.2ArrayList和LinkedList有什么区别？

**1、数据结构不同**

ArrayList是Array(动态数组)的数据结构，LinkedList是Link(链表)的数据结构。

**2、效率不同**

当随机访问List（get和set操作）时，ArrayList比LinkedList的效率更高，因为LinkedList是线性的数据存储方式，所以需要移动指针从前往后依次查找。

当对数据进行增加和删除的操作(add和remove操作)时，LinkedList比ArrayList的效率更高，因为ArrayList是数组，所以在其中进行增删操作时，会对操作点之后所有数据的下标索引造成影响，需要进行数据的移动。

**3、自由性不同**

ArrayList自由性较低，因为它需要手动的设置固定大小的容量，但是它的使用比较方便，只需要创建，然后添加数据，通过调用下标进行使用；而LinkedList自由性较高，能够动态的随数据量的变化而变化，但是它不便于使用。

**4、主要控件开销不同**

ArrayList主要控件开销在于需要在lList列表预留一定空间；而LinkList主要控件开销在于需要存储结点信息以及结点指针信息。





#### 4.3Copy-On-Write

Copy-On-Write简称COW。通俗的理解是当我们往一个容器添加元素的时候，不直接往当前容器添加，而是先将当前容器进行Copy，复制出一个新的容器，然后新的容器里添加元素，添加完元素之后，再将原容器的引用指向新的容器。这样做的好处是我们可以对CopyOnWrite容器进行并发的读，而不需要加锁，因为当前容器不会添加任何元素。所以CopyOnWrite容器也是一种读写分离的思想，读和写不同的容器。从JDK1.5开始Java并发包里提供了两个使用CopyOnWrite机制实现的并发容器,它们是CopyOnWriteArrayList和CopyOnWriteArraySet。
CopyOnWrite并发容器用于读多写少的并发场景







## 并发编程

### 1.线程池相关问题

#### 1.1线程池概念

线程池（Thread Pool）是一种基于池化思想管理线程的工具，经常出现在多线程服务器中，如MySQL。

线程过多会带来额外的开销，其中包括创建销毁线程的开销、调度线程的开销等等，同时也降低了计算机的整体性能。线程池维护多个线程，等待监督管理者分配可并发执行的任务。这种做法，一方面避免了处理任务时创建销毁线程开销的代价，另一方面避免了线程数量膨胀导致的过分调度问题，保证了对内核的充分利用。

而本文描述线程池是JDK中提供的ThreadPoolExecutor类。

当然，使用线程池可以带来一系列好处：

- **降低资源消耗**：通过池化技术重复利用已创建的线程，降低线程创建和销毁造成的损耗。
- **提高响应速度**：任务到达时，无需等待线程创建即可立即执行。
- **提高线程的可管理性**：线程是稀缺资源，如果无限制创建，不仅会消耗系统资源，还会因为线程的不合理分布导致资源调度失衡，降低系统的稳定性。使用线程池可以进行统一的分配、调优和监控。
- **提供更多更强大的功能**：线程池具备可拓展性，允许开发人员向其中增加更多的功能。比如延时定时线程池ScheduledThreadPoolExecutor，就允许任务延期执行或定期执行。





#### 1.2线程池核心设计与实现

![图2 ThreadPoolExecutor运行流程](https://p0.meituan.net/travelcube/77441586f6b312a54264e3fcf5eebe2663494.png)

线程池内部使用一个变量维护两个值：运行状态(runState)和线程数量 (workerCount)。在具体实现中，线程池将运行状态(runState)、线程数量 (workerCount)两个关键参数的维护放在了一起`ctl`这个AtomicInteger类型，是对线程池的运行状态和线程池中有效线程的数量进行控制的一个字段， 它同时包含两部分的信息：线程池的运行状态 (runState) 和线程池内有效线程的数量 (workerCount)，高3位保存runState，低29位保存workerCount，两个变量之间互不干扰。用一个变量去存储两个值，可避免在做相关决策时，出现不一致的情况，不必为了维护两者的一致，而占用锁资源。通过阅读线程池源代码也可以发现，经常出现要同时判断线程池运行状态和线程数量的情况。线程池也提供了若干方法去供用户获得线程池当前的运行状态、线程个数。这里都使用的是位运算的方式，相比于基本运算，速度也会快很多。

![图3 线程池生命周期](https://p0.meituan.net/travelcube/582d1606d57ff99aa0e5f8fc59c7819329028.png)

1. 首先检测线程池运行状态，如果不是RUNNING，则直接拒绝，线程池要保证在RUNNING的状态下执行任务。
2. 如果workerCount < corePoolSize，则创建并启动一个线程来执行新提交的任务。
3. 如果workerCount >= corePoolSize，且线程池内的阻塞队列未满，则将任务添加到该阻塞队列中。
4. 如果workerCount >= corePoolSize && workerCount < maximumPoolSize，且线程池内的阻塞队列已满，则创建并启动一个线程来执行新提交的任务。
5. 如果workerCount >= maximumPoolSize，并且线程池内的阻塞队列已满, 则根据拒绝策略来处理该任务, 默认的处理方式是直接抛异常。

![img](https://p0.meituan.net/travelcube/9ffb64cc4c64c0cb8d38dac01c89c905178456.png)



#### 1.3什么是阻塞队列？阻塞队列的实现原理是什么？如何使用阻塞队列来实现生产者-消费者模型？

阻塞队列（BlockingQueue）是一个支持两个附加操作的队列。

这两个附加的操作是：在队列为空时，获取元素的线程会等待队列变为非空。当队列满时，存储元素的线程会等待队列可用。

阻塞队列常用于生产者和消费者的场景，生产者是往队列里添加元素的线程，消费者是从队列里拿元素的线程。阻塞队列就是生产者存放元素的容器，而消费者也只从容器里拿元素。

JDK7提供了7个阻塞队列。分别是： 

ArrayBlockingQueue ：一个由数组结构组成的有界阻塞队列。 

LinkedBlockingQueue ：一个由链表结构组成的有界阻塞队列。 

PriorityBlockingQueue ：一个支持优先级排序的无界阻塞队列。 

DelayQueue：一个使用优先级队列实现的无界阻塞队列。 

SynchronousQueue：一个不存储元素的阻塞队列。 

LinkedTransferQueue：一个由链表结构组成的无界阻塞队列。 

LinkedBlockingDeque：一个由链表结构组成的双向阻塞队列。

```JAVA
public class BlockingQueueTests {

    public static void main(String[] args) {
        BlockingQueue queue = new ArrayBlockingQueue(10);
        new Thread(new Producer(queue)).start();
        new Thread(new Consumer(queue)).start();
        new Thread(new Consumer(queue)).start();
        new Thread(new Consumer(queue)).start();
    }

}
//生产者
 class Producer implements Runnable {


    private BlockingQueue<Integer> queue;

    public Producer(BlockingQueue<Integer> queue) {
        this.queue = queue;
    }

    @Override
    public void run() {
        try {
            for (int i = 0; i < 100; i++) {
                Thread.sleep(20);
                queue.put(i);
                System.out.println(Thread.currentThread().getName() + "生产:" + queue.size());
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}

//消费者
class Consumer implements Runnable {

    private BlockingQueue<Integer> queue;

    public Consumer(BlockingQueue<Integer> queue) {
        this.queue = queue;
    }

    @Override
    public void run() {
        try {
            while (true) {
                Thread.sleep(new Random().nextInt(1000));
                queue.take();
                System.out.println(Thread.currentThread().getName() + "消费:" + queue.size());
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```



#### 1.4线程池类型

1. newSingleThreadExecutor

- 这个线程池只有一个线程在工作，也就是相当于单线程串行执行所有任务。
  - 保证所有任务按照指定顺序(FIFO, LIFO, 优先级)执行。
  - 应用场景：**GUI单线程队列**

2.newFixedThreadPool

① 线程数少于核心线程数，也就是设置的线程数时，新建线程执行任务 ② 线程数等于核心线程数后，**将任务加入阻塞队列**，于队列容量非常大，可以一直加加加 ③ **执行完任务的线程反复去队列中取任务执行**

3. newCachedThreadPool

① 没有核心线程，直接向 SynchronousQueue 中提交任务 ② 如果有空闲线程，就去取出任务执行；如果没有空闲线程，就新建一个 ③ **执行完任务的线程有 60 秒生存时间**，如果在这个时间内可以接到新任务，就可以继续活下去，否则会被销毁

4.newScheduledThreadPool

- 创建一个定长线程池，支持定时和周期性任务执行。







### 2.线程相关

#### 2.1.Java线程状态

<img src="https://images2018.cnblogs.com/blog/930824/201807/930824-20180715222029724-1669695888.jpg" alt="img" style="zoom:50%;" />



#### 2.2线程上下文切换

引起线程上下文切换的原因

当前正在执行的任务完成，系统的cpu正常调度下一个任务

当前正在执行的任务遇到i/o等阻塞操作，调度器挂起此任务，继续调度下一个任务。

多个任务并发抢占资源，当前任务么有抢到锁资源，被调度器挂起，继续调度下一个任务，

用户的代码挂起当前任务，比如线程执行sleep方法，让出CPU。

硬件中断。



#### 2.3线程创建方式

1. 继承Thread类

通过继承 Thread 类来创建线程的一般步骤如下：

1. 定义一个 Thread 类的子类，重写 run() 方法，将相关逻辑实现，run() 方法就是线程要执行的业务逻辑方法；
2. 创建自定义的线程子类对象；
3. 调用子类实例的 start() 方法来启动线程。

```javascript
public class MyThread extends Thread{ @Override public void run() { System.out.println(Thread.currentThread().getName()); }
}
public class MyThreadTest { public static void main(String[] args) { // 创建线程 MyThread thread = new MyThread(); // 启动线程 thread.start(); }
}
```

2. 实现 Runnable 接口

通过实现 Runnable 接口创建线程一般步骤如下：

1. 定义 Runnable 接口实现类 MyRunnable，并重写 run() 方法；
2. 创建 MyRunnable 实例 runnable，以 runnable 作为 target 创建 Thead 对象，该 Thread 对象才是真正的线程对象；
3. 调用线程对象的 start() 方法。

```javascript
public class MyRunnable implements Runnable{ @Override public void run() { System.out.println(Thread.currentThread().getName()); }
}
public class MyRunnableTest { public static void main(String[] args) { MyRunnable myRunnable = new MyRunnable(); // 创建线程 Thread thread = new Thread(myRunnable); // 启动线程 thread.start(); }
}
```

3. **使用 Callable 和 Future 创建线程**

与 Runnable 接口不一样，Callable 接口提供了一个 call() 方法作为线程执行体，call() 方法比 run() 方法功能要强大，比如：call() 方法可以有返回值、call() 方法可以声明抛出异常。

Java5 提供了 Future 接口来代表 Callable 接口里 call() 方法的返回值，并且为 Future 接口提供了一个实现类 FutureTask，这个实现类既实现了 Future 接口，还实现了 Runnable 接口，因此可以作为 Thread 类的 target。在 Future 接口里定义了几个公共方法来控制它关联的 Callable 任务。

使用 Callable 和 Future 创建线程的一般步骤如下：

1. 创建实现 Callable 接口的类 myCallable；
2. 以 myCallable 为参数创建 FutureTask 对象；
3. 将 FutureTask 作为参数创建 Thread 对象；
4. 调用线程对象的 start() 方法。

```javascript
import java.util.concurrent.Callable;

public class MyCallable implements Callable { @Override public Integer call() throws Exception { System.out.println(Thread.currentThread().getName()); return 99; }
}
public class MyCallableTest { public static void main(String[] args) { FutureTask futureTask = new FutureTask<>(new MyCallable()); // 创建线程 Thread thread = new Thread(futureTask); // 启动线程 thread.start(); // 结果返回 try { Thread.sleep(1000); System.out.println("返回的结果是：" + futureTask.get()); } catch (Exception e) { e.printStackTrace(); } }
}
```

4. **使用线程池创建线程**

Executors 提供了一系列工厂方法用于创先线程池，返回的线程池都实现了ExecutorService 接口。

主要有四种：

1. newFixedThreadPool
2. newCachedThreadPool
3. newSingleThreadExecutor
4. newScheduledThreadPool

```javascript
public class MyRunnable implements Runnable{ @Override public void run() { System.out.println(Thread.currentThread().getName()); }
}
public class SingleThreadExecutorTest { public static void main(String[] args) { ExecutorService executorService = Executors.newSingleThreadExecutor(); MyRunnable myRunnable = new MyRunnable(); for(int i = 0; i < 10; i++){ executorService.execute(myRunnable); } System.out.println("=======任务开始======="); executorService.shutdown(); }
}
```





#### 2.4.在java中守护线程和本地线程区别

任何线程都可以设置为守护线程和用户线程，通过方法 Thread.setDaemon(boolon)；true 则把该线程设置为守护线程，反之则为用户线程。Thread.setDaemon()必须在 Thread.start()之前调用，否则运行时会抛出异常。

两者的区别：

唯一的区别是判断虚拟机(JVM)何时离开，Daemon 是为其他线程提供服务，如果全部的 User Thread 已经撤离，Daemon 没有可服务的线程，JVM 撤离。也可以理解为守护线程是 JVM 自动创建的线程（但不一定），用户线程是程序创建的线程；比如 JVM 的垃圾回收线程是一个守护线程，当所有线程已经撤离，不再产生垃圾，守护线程自然就没事可干了，当垃圾回收线程是 Java 虚拟机上仅剩的线程时，Java 虚拟机会自动离开。

扩展：Thread Dump 打印出来的线程信息，含有 daemon 字样的线程即为守护进程，可能会有：服务守护进程、编译守护进程、windows 下的监听 Ctrl+break的守护进程、Finalizer 守护进程、引用处理守护进程、GC 守护进程。









#### 2.5.Java中用到的线程调度算法是什么？

**有两种调度模型：分时调度模型和抢占式调度模型。**

分时调度模型是指让所有的线程轮流获得 cpu 的使用权,并且平均分配每个线程占用的 CPU 的时间片这个也比较好理解。

java 虚拟机采用抢占式调度模型，是指优先让可运行池中优先级高的线程占用 CPU，如果可运行池中的线程优先级相同，那么就随机选择一个线程，使其占用 CPU。处于运行状态的线程会一直运行，直至它不得不放弃 CPU。





#### 2.6.executor和executors的区别

Executors 工具类的不同方法按照我们的需求创建了不同的线程池，来满足业务的需求。

Executor 接口对象能执行我们的线程任务。

ExecutorService 接口继承了 Executor 接口并进行了扩展，提供了更多的方法我们能获得任务执行的状态并且可以获取任务的返回值。

使用 ThreadPoolExecutor 可以创建自定义线程池。





#### 2.7wait()/notify()/sleep()/yield()/join()几个方法的意义

在Object.java中，定义了wait(), notify()和notifyAll()等方法。

wait()的作用是让当前线程进入等待状态，同时，wait()也会让当前线程释放它所持有的锁。而notify()和notifyAll()的作用，则是唤醒当前对象上的等待线程；notify()是唤醒单个线程，而notifyAll()是唤醒所有的线程。

notify()唤醒在此对象锁上等待的单个线程。
notifyAll()唤醒在此对象锁上等待的所有线程。
wait()让当前线程处于“等待(阻塞)状态”，“直到其他线程调用此对象的 notify() 方法或 notifyAll() 方法”，当前线程被唤醒(进入“就绪状态”)。
wait(long timeout) 让当前线程处于“等待(阻塞)状态”，“直到其他线程调用此对象的 notify() 方法或 notifyAll() 方法，或者超过指定的时间量”，当前线程被唤醒(进入“就绪状态”)。
wait(long timeout, int nanos) 让当前线程处于“等待(阻塞)状态”，“直到其他线程调用此对象的 notify() 方法或 notifyAll() 方法，或者其他某个线程中断当前线程，或者已超过某个实际时间量”，当前线程被唤醒(进入“就绪状态”)。

在Thread.java中，定义了join(),sleep(),yield()等方法。

join方法把指定的线程添加到当前线程中，可以不给参数直接thread.join（），也可以给一个时间参数，单位为毫秒thread.join（100）。事实上join方法是通过wait方法来实现的。比如线程A中加入了线程B.join方法，则线程A默认执行wait()方法，释放资源进入等待状态，此时线程B获得资源，执行结束后释放资源，线程A重新获取自CPU，继续执行，由此实现线程的顺序执行。

sleep()方法导致了程序暂停执行指定的时间，让出cpu给其他线程，但是它的监控状态依然保持者，当指定的时间到了又会自动苏醒，并返回到可运行状态，不是运行状态。sleep()中指定的时间是线程不会运行的最短时间。因此，sleep()方法不能保证该线程睡眠到期后就开始执行。在调用sleep()方法的过程中，线程不会释放对象锁。

yield意味着放手，放弃，投降。一个调用yield()方法的线程告诉虚拟机它乐意让其他线程占用自己的位置。这表明该线程没有在做一些紧急的事情。注意，这仅是一个暗示，并不能保证不会产生任何影响。

让我们列举一下关于以上定义重要的几点：

Yield是一个静态的原生(native)方法
Yield告诉当前正在执行的线程把运行机会交给线程池中拥有相同优先级的线程
Yield不能保证使得当前正在运行的线程迅速转换到可运行的状态
它仅能使一个线程从运行状态转到可运行状态，而不是等待或阻塞状态









### 2.java原子类实现原理

incrementAndGet的逻辑：

1.先获取当前的value值

　　2.对value加一

　　3.第三步是关键步骤，调用compareAndSet方法来来进行原子更新操作，这个方法的语义是：

　　　　**先检查当前value是否等于current，如果相等，则意味着value没被其他线程修改过，更新并返回true。如果不相等，compareAndSet则会返回false，然后循环继续尝试更新。**

　　compareAndSet调用了Unsafe类的compareAndSwapInt方法，Unsafe的compareAndSwapInt是个native方法，也就是平台相关的。它是基于CPU的CAS指令来完成的。





### 3.Java当中的锁

#### 3.1分类

1.公平锁 / 非公平锁

2.可重入锁 / 不可重入锁

3.独享锁 / 共享锁

4.互斥锁 / 读写锁

5.乐观锁 / 悲观锁

6.分段锁

7.偏向锁 / 轻量级锁 / 重量级锁





#### 3.2.轻量级锁、重量级锁说一下原理

 通俗的讲，偏向锁就是在运行过程中，对象的锁偏向某个线程。即在开启偏向锁机制的情况下，某个线程获得锁，当该线程下次再想要获得锁时，不需要再获得锁（即忽略synchronized关键词），直接就可以执行同步代码，比较适合竞争较少的情况。

> 偏向锁的获取流程：

 （1）查看Mark Word中偏向锁的标识以及锁标志位，若是否偏向锁为1且锁标志位为01，则该锁为可偏向状态。

 （2）若为可偏向状态，则测试Mark Word中的线程ID是否与当前线程相同，若相同，则直接执行同步代码，否则进入下一步。

 （3）当前线程通过CAS操作竞争锁，若竞争成功，则将Mark Word中线程ID设置为当前线程ID，然后执行同步代码，若竞争失败，进入下一步。

 （4）当前线程通过CAS竞争锁失败的情况下，说明有竞争。当到达全局安全点时之前获得偏向锁的线程被挂起，偏向锁升级为轻量级锁，然后被阻塞在安全点的线程继续往下执行同步代码。

>  偏向锁只有遇到其他线程尝试竞争偏向锁时，持有偏向锁状态的线程才会释放锁，线程不会主动去释放偏向锁。偏向锁的撤销需要等待全局安全点（即没有字节码正在执行），它会暂停拥有偏向锁的线程，撤销后偏向锁恢复到未锁定状态或轻量级锁状态。

轻量级锁不是用来替代传统的重量级锁的，而是在没有多线程竞争的情况下，使用轻量级锁能够减少性能消耗，但是当多个线程同时竞争锁时，轻量级锁会膨胀为重量级锁。

（1）当线程执行代码进入同步块时，若Mark Word为无锁状态，虚拟机先在当前线程的栈帧中建立一个名为Lock Record的空间，用于存储当前对象的Mark Word的拷贝，官方称之为“Dispalced Mark Word”

（2）复制对象头中的Mark Word到锁记录中。

（3）复制成功后，虚拟机将用CAS操作将对象的Mark Word更新为执行Lock  Record的指针，并将Lock Record里的owner指针指向对象的Mark Word。如果更新成功，则执行4，否则执行5。；

（4）如果更新成功，则这个线程拥有了这个锁，并将锁标志设为00，表示处于轻量级锁状态，此时状态图：

（5）如果更新失败，虚拟机会检查对象的Mark Word是否指向当前线程的栈帧，如果是则说明当前线程已经拥有这个锁，可进入执行同步代码。否则说明多个线程竞争，轻量级锁就会膨胀为重量级锁，Mark Word中存储重量级锁（互斥锁）的指针，后面等待锁的线程也要进入阻塞状态。

五、重量级锁

即当有其他线程占用锁时，当前线程会进入阻塞状态。

六、自旋锁

 在自旋状态下，当一个线程A尝试进入同步代码块，但是当前的锁已经被线程B占有时，线程A不进入阻塞状态，而是不停的空转，等待线程B释放锁。**如果锁的线程能在很短时间内释放资源，那么等待竞争锁的线程就不需要做内核态和用户态之间的切换进入阻塞状态，只需自旋，等持有锁的线程释放后即可立即获取锁，避免了用户线程和内核的切换消耗。**

**自旋等待最大时间**：线程自旋会消耗cpu，若自旋太久，则会让cpu做太多无用功，因此要设置自旋等待最大时间。

**优点**：开启自旋锁后能减少线程的阻塞，在对于锁的竞争不激烈且占用锁时间很短的代码块来说，能提升很大的性能，在这种情况下自旋的消耗小于线程阻塞挂起的消耗。

**缺点**：在线程竞争锁激烈，或持有锁的线程需要长时间执行同步代码块的情况下，使用自旋会使得cpu做的无用功太多。





#### 3.3.synchronize关键字

简单来说，synchronized关键字用于多线程工作时，保证同一时刻最多只有一个线程执行该段代码。

它有两种形态，一种作为对象锁，一种作为类锁。

1. 给普通方法加锁
2. 给同步代码块加锁
3. 给静态方法加锁
4. 给代码块加锁

synchronized加在代码块上，JVM是通过**monitorenter**和**monitorexit**来控制锁的获取的释放的；

synchronized加在方法上，JVM是通过**ACC_SYNCHRONIZED**标记来控制的，但本质上也是通过monitorenter和monitorexit指令控制的。

- 执行同步代码块后首先要先执行**monitorenter**指令，退出的时候**monitorexit**指令。通过分析之后可以看出，使用Synchronized进行同步，其关键就是必须要对对象的监视器monitor进行获取，当线程获取monitor后才能继续往下执行，否则就只能等待。而这个获取的过程是**互斥**的，即同一时刻只有一个线程能够获取到monitor。上面的demo中在执行完同步代码块之后紧接着再会去执行一个静态同步方法，而这个方法锁的对象依然就这个类对象，那么这个正在执行的线程还需要获取该锁吗？答案是不必的，从上图中就可以看出来，执行静态同步方法的时候就只有一条monitorexit指令，并没有monitorenter获取锁的指令。这就是**锁的重入性**，即在同一锁程中，线程不需要再次获取同一把锁。Synchronized先天具有重入性。**每个对象拥有一个计数器，当线程获取该对象锁后，计数器就会加一，释放锁后就会将计数器减一**。

jdk1.6之前synchronized是很重的，所以并不被开发者偏爱，随着后续版本jdk对synchronized的优化使其越来越轻量，它还是很好用的，甚至ConcurrentHashMap在jdk的put方法都在jdk1.8时从ReetrantLock.tryLock()改为用synchronized来实现同步。 并且还引入了偏向锁，轻量级锁等概念，下面是偏向锁和轻量级锁的获取流程

| 区别               | synchronized                                                 | ReentrantLock                                                |
| ------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 灵活性             | 代码简单，自动获取、释放锁                                   | 相对繁琐，需要手动获取、释放锁                               |
| 是否可重入         | 是                                                           | 是                                                           |
| 作用位置           | 可作用在方法和代码块                                         | 只能用在代码块                                               |
| 获取、释放锁的方式 | monitorenter、monitorexit、ACC_SYNCHRONIZED                  | 尝试非阻塞获取锁tryLock()、超时获取锁tryLock(long timeout,TimeUnit unit)、unlock() |
| 获取锁的结果       | 不知道                                                       | 可知，tryLock()返回boolean                                   |
| 使用注意事项       | 1、锁对象不能为空（锁保存在对象头中，null没有对象头） 2、作用域不宜过大 | 1、切记要在finally中unlock()，否则会形成死锁  2、不要将获取锁的过程写在try块内，因为如果在获取锁时发生了异常，异常抛出的同时，也会导致锁无故被释放。 |







#### 3.4.volatile作用，底层原理

如果对声明了volatile变量进行写操作时，**JVM会向处理器发送一条Lock前缀的指令，将这个变量所在缓存行的数据写会到系统内存**。这一步确保了如果有其他线程对声明了`volatile`变量进行修改，则立即更新主内存中数据。但这时候其他处理器的缓存还是旧的，所以在多处理器环境下，为了保证各个处理器缓存一致，每个处理会通过**嗅探在总线上传播的数据**来检查自己的缓存是否过期，当处理器发现自己缓存行对应的内存地址被修改了，就会将当前处理器的缓存行**设置成无效状态**，当处理器要对这个数据进行修改操作时，会强制重新从系统内存把数据读到处理器缓存里。 这一步确保了其他线程获得的声明了`volatile`变量都是从主内存中获取最新的。

Lock前缀指令实际上相当于一个**内存屏障（也成内存栅栏）**，它确保指令重排序时不会把其后面的指令排到内存屏障之前的位置，也不会把前面的指令排到内存屏障的后面；即在**执行到内存屏障这句指令时，在它前面的操作已经全部完成**。

`volatile`的底层实现是通过**插入内存屏障**，但是对于编译器来说，发现一个最优布置来最小化插入内存屏障的总数几乎是不可能的，所以，JMM采用了保守策略。如下：

- **在每一个volatile写操作前面插入一个StoreStore屏障：保证在volatile写之前，其前面的所有普通写操作都已经刷新到主内存中**
- **在每一个volatile写操作后面插入一个StoreLoad屏障：避免volatile写与后面可能有的volatile读/写操作重排序**
- **在每一个volatile读操作后面插入一个LoadLoad屏障：禁止处理器把上面的volatile读与下面的普通读重排序**
- **在每一个volatile读操作后面插入一个LoadStore屏障：禁止处理器把上面的volatile读与下面的普通写重排序**



#### 3.5.JMM

JMM 最重要的的三点内容：**重排序、原子性、内存可见性**。那么 JMM 又是如何解决这些问题的呢？

JMM 抽象出主存储器（Main Memory）和工作存储器（Working Memory）两种。

- 主存储器是实例位置所在的区域，所有的实例都存在于主存储器内。比如，实例所拥有的字段即位于主存储器内，主存储器是所有的线程所共享的。
- 工作存储器是线程所拥有的作业区，每个线程都有其专用的工作存储器。工作存储器存有主存储器中必要部分的拷贝，称之为工作拷贝（Working Copy）。

线程是无法直接对主内存进行操作的，如下图所示，线程 A 想要和线程 B 通信，只能通过主存进行交换。

经历下面 2 个步骤：

1）线程 A 把本地内存 A 中更新过的共享变量刷新到主内存中去。

2）线程 B 到主内存中去读取线程 A 之前已更新过的共享变量。

为了支持 JMM，Java 定义了 8 种原子操作（Action），用来控制主存与工作内存之间的交互：

1. **read** 读取：作用于主内存，将共享变量从主内存传动到线程的工作内存中，供后面的 load 动作使用。
2. **load** 载入：作用于工作内存，把 read 读取的值放到工作内存中的副本变量中。
3. **store** 存储：作用于工作内存，把工作内存中的变量传送到主内存中，为随后的 write 操作使用。
4. **write** 写入：作用于主内存，把 store 传送值写到主内存的变量中。
5. **use** 使用：作用于工作内存，把工作内存的值传递给执行引擎，当虚拟机遇到一个需要使用这个变量的指令，就会执行这个动作。
6. **assign** 赋值：作用于工作内存，把执行引擎获取到的值赋值给工作内存中的变量，当虚拟机栈遇到给变量赋值的指令，执行该操作。比如 `int i = 1;`
7. **lock（锁定）** 作用于主内存，把变量标记为线程独占状态。
8. **unlock（解锁）** 作用于主内存，它将释放独占状态。



#### 3.6.**Happens-Before**

- 程序顺序原则：如果程序操作 A 在操作 B 之前，那么多线程中的操作依然是 A 在 B 之前执行。
- 监视器锁原则：在监视器锁上的解锁操作必须在同一个监视器上的加锁操作之前执行。
- volatile 变量原则：对 volatile 修饰的变量写入操作必须在该变量的毒操作之前执行。
- 线程启动原则：在线程对 Tread.start 调用必须在该线程执行任何操作之前执行。
- 线程结束原则：线程的任何操作必须在其他线程检测到该线程结束前执行，或者从 Thread.join 中成功返回，或者在调用 Thread.isAlive 返回 false。
- 中断原则：当一个线程在另一个线程上调用 interrupt 时，必须在被中断线程检测到 interrupt 调用之前执行。
- 终结器规则：对象的构造方法必须在启动对象的终结器之前完成。
- 传递性：如果操作 A 在操作 B 之前执行，并且操作 B 在操作 C 之前执行，那么操作 A 必须在操作 C 之前执行。



#### 3.7CAS乐观锁

CAS机制当中使用了3个基本操作数：内存地址V，旧的预期值A，要修改的新值B。

更新一个变量的时候，只有当变量的预期值A和内存地址V当中的实际值相同时，才会将内存地址V对应的值修改为B。

CAS的缺点：

1.CPU开销较大
 在并发量比较高的情况下，如果许多线程反复尝试更新某一个变量，却又一直更新不成功，循环往复，会给CPU带来很大的压力。

2.不能保证代码块的原子性
 CAS机制所保证的只是一个变量的原子性操作，而不能保证整个代码块的原子性。比如需要保证3个变量共同进行原子性的更新，就不得不使用Synchronized了。



#### 3.8COW

CopyOnWrite容器即写时复制的容器。通俗的理解是当我们往一个容器添加元素的时候，不直接往当前容器添加，而是先**将当前容器进行Copy，复制出一个新的容器，然后新的容器里添加元素，添加完元素之后，再将原容器的引用指向新的容器。**这样做的好处是我们可以对CopyOnWrite容器进行并发的读，而不需要加锁，因为当前容器不会添加任何元素。所以CopyOnWrite容器也是一种读写分离的思想，读和写不同的容器。



### 4.ThreadLocal

ThreadLocal 是 Java 里一种特殊变量，它是一个线程级别变量，每个线程都有一个 ThreadLocal 就是每个线程都拥有了自己独立的一个变量，竞态条件被彻底消除了，在并发模式下是绝对安全的变量。

可以通过 `ThreadLocal<T> value = new ThreadLocal<T>();` 来使用。

会自动在每一个线程上创建一个 T 的副本，副本之间彼此独立，互不影响，可以用 ThreadLocal 存储一些参数，以便在线程中多个方法中使用，用以代替方法传参的做法。

```java
public class ThreadLocalDemo {
    /**
     * ThreadLocal变量，每个线程都有一个副本，互不干扰
     */
    public static final ThreadLocal<String> THREAD_LOCAL = new ThreadLocal<>();

    public static void main(String[] args) throws Exception {
        new ThreadLocalDemo().threadLocalTest();
    }

    public void threadLocalTest() throws Exception {
        // 主线程设置值
        THREAD_LOCAL.set("wupx");
        String v = THREAD_LOCAL.get();
        System.out.println("Thread-0线程执行之前，" + Thread.currentThread().getName() + "线程取到的值：" + v);

        new Thread(new Runnable() {
            @Override
            public void run() {
                String v = THREAD_LOCAL.get();
                System.out.println(Thread.currentThread().getName() + "线程取到的值：" + v);
                // 设置 threadLocal
                THREAD_LOCAL.set("huxy");
                v = THREAD_LOCAL.get();
                System.out.println("重新设置之后，" + Thread.currentThread().getName() + "线程取到的值为：" + v);
                System.out.println(Thread.currentThread().getName() + "线程执行结束");
            }
        }).start();
        // 等待所有线程执行结束
        Thread.sleep(3000L);
        v = THREAD_LOCAL.get();
        System.out.println("Thread-0线程执行之后，" + Thread.currentThread().getName() + "线程取到的值：" + v);
    }
}
```

```
Thread-0线程执行之前，main线程取到的值：wupx
Thread-0线程取到的值：null
重新设置之后Thread-0线程取到的值为：huxy
Thread-0线程执行结束
Thread-0线程执行之后，main线程取到的值：wupx
```

首先在 `Thread-0` 线程执行之前，先给 `THREAD_LOCAL` 设置为 `wupx`，然后可以取到这个值，然后通过创建一个新的线程以后去取这个值，发现新线程取到的为 null，意外着这个变量在不同线程中取到的值是不同的，不同线程之间对于 ThreadLocal 会有对应的副本，接着在线程 `Thread-0` 中执行对 `THREAD_LOCAL` 的修改，将值改为 `huxy`，可以发现线程 `Thread-0` 获取的值变为了 `huxy`，主线程依然会读取到属于它的副本数据 `wupx`，这就是线程的封闭。

首先看下 ThreadLocal 都有哪些重要属性：

```java
// 当前 ThreadLocal 的 hashCode，由 nextHashCode() 计算而来，用于计算当前 ThreadLocal 在 ThreadLocalMap 中的索引位置
private final int threadLocalHashCode = nextHashCode();
// 哈希魔数，主要与斐波那契散列法以及黄金分割有关
private static final int HASH_INCREMENT = 0x61c88647;
// 返回计算出的下一个哈希值，其值为 i * HASH_INCREMENT，其中 i 代表调用次数
private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
// 保证了在一台机器中每个 ThreadLocal 的 threadLocalHashCode 是唯一的
private static AtomicInteger nextHashCode = new AtomicInteger();
```

其中的 `HASH_INCREMENT` 也不是随便取的，它转化为十进制是 `1640531527`，`2654435769` 转换成 int 类型就是 `-1640531527`，`2654435769` 等于 `(√5-1)/2` 乘以 2 的 32 次方。`(√5-1)/2` 就是黄金分割数，近似为 `0.618`，也就是说 `0x61c88647` 理解为一个黄金分割数乘以 2 的 32 次方，它可以保证 nextHashCode 生成的哈希值，均匀的分布在 2 的幂次方上，且小于 2 的 32 次方。

ThreadLocalMap 是 ThreadLocal 的静态内部类，当一个线程有多个 ThreadLocal 时，需要一个容器来管理多个 ThreadLocal，ThreadLocalMap 的作用就是管理线程中多个 ThreadLocal，ThreadLocalMap 其实就是一个简单的 Map 结构，底层是数组，有初始化大小，也有扩容阈值大小，数组的元素是 Entry，**Entry 的 key 就是 ThreadLocal 的引用，value 是 ThreadLocal 的值**。ThreadLocalMap 解决 hash 冲突的方式采用的是**线性探测法**，如果发生冲突会继续寻找下一个空的位置。

ThreadLocal 在没有外部强引用时，发生 GC 时会被回收，那么 ThreadLocalMap 中保存的 key 值就变成了 null，而 Entry 又被 threadLocalMap 对象引用，threadLocalMap 对象又被 Thread 对象所引用，那么当 Thread 一直不终结的话，value 对象就会一直存在于内存中，也就导致了内存泄漏，直至 Thread 被销毁后，才会被回收。

那么如何避免内存泄漏呢？

在使用完 ThreadLocal 变量后，需要我们手动 remove 掉，防止 ThreadLocalMap 中 Entry 一直保持对 value 的强引用，导致 value 不能被回收。

```java
/**
 * 为当前 ThreadLocal 对象关联 value 值
 *
 * @param value 要存储在此线程的线程副本的值
 */
public void set(T value) {
	// 返回当前ThreadLocal所在的线程
	Thread t = Thread.currentThread();
	// 返回当前线程持有的map
	ThreadLocalMap map = getMap(t);
	if (map != null) {
		// 如果 ThreadLocalMap 不为空，则直接存储<ThreadLocal, T>键值对
		map.set(this, value);
	} else {
		// 否则，需要为当前线程初始化 ThreadLocalMap，并存储键值对 <this, firstValue>
		createMap(t, value);
	}
}
```

set 方法的作用是把我们想要存储的 value 给保存进去。set 方法的流程主要是：

- 先获取到当前线程的引用
- 利用这个引用来获取到 ThreadLocalMap
- 如果 map 为空，则去创建一个 ThreadLocalMap
- 如果 map 不为空，就利用 ThreadLocalMap 的 set 方法将 value 添加到 map 中

```java
/**
 * 返回当前 ThreadLocal 对象关联的值
 *
 * @return
 */
public T get() {
	// 返回当前 ThreadLocal 所在的线程
	Thread t = Thread.currentThread();
	// 从线程中拿到 ThreadLocalMap
	ThreadLocalMap map = getMap(t);
	if (map != null) {
		// 从 map 中拿到 entry
		ThreadLocalMap.Entry e = map.getEntry(this);
		// 如果不为空，读取当前 ThreadLocal 中保存的值
		if (e != null) {
			@SuppressWarnings("unchecked")
			T result = (T) e.value;
			return result;
		}
	}
	// 若 map 为空，则对当前线程的 ThreadLocal 进行初始化，最后返回当前的 ThreadLocal 对象关联的初值，即 value
	return setInitialValue();
}
```

get 方法的主要流程为：

- 先获取到当前线程的引用
- 获取当前线程内部的 ThreadLocalMap
- 如果 map 存在，则获取当前 ThreadLocal 对应的 value 值
- 如果 map 不存在或者找不到 value 值，则调用 setInitialValue() 进行初始化

ThreadLocal 的特性也导致了应用场景比较广泛，主要的应用场景如下：

- 线程间数据隔离，各线程的 ThreadLocal 互不影响
- 方便同一个线程使用某一对象，避免不必要的参数传递
- 全链路追踪中的 traceId 或者流程引擎中上下文的传递一般采用 ThreadLocal
- Spring 事务管理器采用了 ThreadLocal
- Spring MVC 的 RequestContextHolder 的实现使用了 ThreadLocal





### 5.AQS相关

#### 5.1原理

  它内部有一个int类型的state变量，被volatile关键字修饰，保证线程之间的可见。还会维护一个Node内部类（用于生成同步对列和等待队列），并继承过来一个加锁线程。state变量的访问方式有三种：getState()，setState(int)，compareAndSetState(int,int)三个方法。AQS定义了两种资源共享的方式，独占模式和共享方式。
   为了将此类用作同步器的基础，需要适当的重新定义以下方法，这是通过使用getState()，setState(int)，compareAndSetState(int,int)三个方法来检查或修改同步状态来实现的。

        tryAcquire(int)       试图在独占模式下获取对象状态，由acquire自动调用，至少调用一次
    
        tryRelease(int)      试图设置状态来反映独占模式下的一个释放，由release自动调用，至少调用一次
    
        tryAcquireShared(int)         试图在共享模式下获取对象状态，由acquireShared自动调用，至少调用一次
    
         tryReleaseShared(int)      试图设置状态来反映共享模式下的一个释放，由releaseShared自动调用，至少调用一次
  ReentrantLock就是使用AQS而实现的一把锁，它实现了可重入锁，公平锁和非公平锁。它有一个内部类用作同步器是Sync，Sync是继承了AQS的一个子类，并且公平锁和非公平锁是继承了Sync的两个子类。ReentrantLock的原理是：假设有一个线程A来尝试获取锁，它会先CAS修改state的值，从0修改到1，如果修改成功，那就说明获取锁成功，设置加锁线程为当前线程。如果此时又有一个线程B来尝试获取锁，那么它也会CAS修改state的值，从0修改到1，因为线程A已经修改了state的值，那么线程B就会修改失败，然后他会判断一下加锁线程是否为自己本身线程，如果是自己本身线程的话它就会将state的值直接加1，这是为了实现锁的可重入。如果加锁线程不是当前线程的话，那么就会将它生成一个Node节点，加入到等待队列。

同步队列的作用是：当线程获取资源失败之后，就进入同步队列的尾部保持自旋等待，不断判断自己是否是链表的头节点，如果是头节点，就不断参试获取资源，获取成功后则退出同步队列。
条件队列是为Lock实现的一个基础同步器，并且一个线程可能会有多个条件队列，只有在使用了Condition才会存在条件队列。

同步队列和条件队列都是由一个个Node组成的。AQS内部有一个静态内部类Node。



ReentrantLock主要利用CAS+AQS队列来实现。它支持公平锁和非公平锁，两者的实现类似。

CAS：Compare and Swap，比较并交换。CAS有3个操作数：内存值V、预期值A、要修改的新值B。当且仅当预期值A和内存值V相同时，将内存值V修改为B，否则什么都不做。该操作是一个原子操作，被广泛的应用在Java的底层实现中。在Java中，CAS主要是由sun.misc.Unsafe这个类通过JNI调用CPU底层指令实现

synchronized和ReentrantLock都是可重入的，后者使用更加灵活，也提供了更多的高级特性，但其本质的实现原理是差不多的（synchronized 在jdk1.6以后优化很多都是借鉴了ReentrantLock的实现原理，就是努力在用户态把加锁问题解决，避免进入内核态的线程阻塞）。

**NonfairSync :**

非公平锁首先用一个CAS操作，判断state是否是0（表示当前锁未被占用），如果是0则把它置为1，直接将exclusiveOwnerThread设置为当前线程，设置当前线程为该锁的独占线程，表示获取锁成功。当多个线程同时尝试占用同一个锁时，CAS操作只能保证一个线程操作成功，剩下的只能乖乖的去排队啦。

非公平”即体现在这里，如果占用锁的线程刚释放锁，state置为0，而排队等待锁的线程还未唤醒时，新来的线程就直接抢占了该锁，那么就“插队”了。

区别： 这里就是 公平锁和非公平锁的第一个不同，非公平锁首先会调用CAS将state从0改为1，如果能改成功则表示获取到锁，直接将exclusiveOwnerThread设置为当前线程，不用再进行后续操作；否则则同公平锁一样调用acquire方法获取锁。

公平锁和非公平锁的代码基本是一样的，**区别**在于首次加锁需要判断是否已经有队列存在，没有才去加锁，有则直接返回false。

addWaiter入队

2.第二步，入队。如果已经有线程A已经占用了锁，所以B和C执行tryAcquire失败，并且入等待队列。如果线程A拿着锁死死不放，那么B和C就会被挂起。

先看下入队的过程。先看addWaiter(Node.EXCLUSIVE)方法，当尝试加锁失败时，首先就会调用该方法创建一个Node节点并添加到队列中去。






#### 5.2AQS组件：

ReentrantReadWriteLock、CountDownLatch、CyclicBarrier、Semaphore原理掌握

**ReentrantLock**是一把独占锁，只支持重入，不支持共享，所以JUC包下还提供了**读写锁**，这把锁**支持读读并发，但读写、写写都是互斥的**。将state字段分为了高二字节和低二字节，即高16位用来表示读锁状态，低16位则用来表示写锁

```java
import org.junit.Test;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

/*
 * 学习常用的几个锁的类
 * 1.ReentrantLock  可重用锁
 *
 * 2.CountDownLatch  场景：任务需要在几个线程后才能执行，类似于一个计数器
 *
 * 3.CyclicBarrier  循环屏障   任务会在几个线程都到达时，才会一起执行，还有一个高级构造方法actionBarrier来进行优先处理
 *
 * 4.Semaphore  信号量     用于限制并发线程的数量
 *
 * 5.reentrantReadWriteLock  实现一个缓存器
 * */
public class LockStudy {
    @Test
    public void ReentrantLockTest() {
        final ReentrantLock rl = new ReentrantLock();
        new Thread(new Runnable() {
            public void run() {
                try {
                    rl.lock();
                    Thread.sleep(3000);
                    System.out.println("子线程" + rl.getHoldCount());
                } catch (Exception e) {
                    System.out.println("线程上锁异常");
                } finally {
                    rl.unlock();
                }

            }
        }).start();

        try {
            Thread.sleep(1000);
            rl.lock();
            System.out.println("主线程的阻塞长度" + rl.getQueueLength());
        } catch (Exception e) {
            System.out.println("主线程上锁失败");
        } finally {
            rl.unlock();
        }
    }

    @Test
    public void CountDownLatch() {
        final CountDownLatch cdl = new CountDownLatch(4);

        //3个线程后，才会执行下面的业务
        for (int i = 0; i < 3; i++) {
            new Thread(new Runnable() {
                public void run() {
                    System.out.println("子线程");
                    cdl.countDown();
                }
            }).start();
        }
        try {
            cdl.await(3, TimeUnit.SECONDS);
        } catch (Exception e) {
            System.out.println("超时自动中断");
        } finally {
            System.out.println("执行业务代码");
        }

    }

    public volatile int i = 0;

    @Test
    public void CyclicBarrierTest() {
        final CyclicBarrier cyclicBarrier = new CyclicBarrier(3);

        new Thread(new Runnable() {
            public void run() {
                try {
                    System.out.println("第一个子线程准备好了");
                    cyclicBarrier.await();
                } catch (Exception e) {
                    System.out.println("第一个++++子线程+++阻塞失败");
                } finally {
                    System.out.println("子线程执行了");
                    i++;
                    System.out.println("子线程的运行结果" + i);
                }

            }
        }).start();
        new Thread(new Runnable() {
            public void run() {
                try {
                    System.out.println("第二个子线程准备好了");

                    cyclicBarrier.await();
                } catch (Exception e) {
                    System.out.println("子线程的子线程++++阻塞失败");
                } finally {
                    System.out.println("第二个子线程执行了");
                    i++;
                    System.out.println("子线程的运行结果" + i);
                }
            }
        }).start();
        try {
            try {
                System.out.println("主线程线程准备好了");
                Thread.sleep(3000);


            } catch (Exception e) {
                System.out.println("线程睡眠失败");
            }
            cyclicBarrier.await();
        } catch (Exception e) {
            System.out.println("第二个+++主线程++++阻塞失败");
        } finally {
            System.out.println("主线程执行了");
            i++;
            System.out.println("主线程结果" + i);
        }


    }

    @Test
    public void resTest() {
        try {
            CyclicBarrierTest();
            Thread.sleep(3000);
        } catch (Exception e) {

        } finally {
            System.out.println("最终结果" + i);
        }


    }

    @Test
    public void semaphoreTest() {
        final Semaphore semaphore = new Semaphore(2);

        for (int i = 0; i < 8; i++) {
            new Thread(new Runnable() {
                public void run() {
                    try {
                        System.out.println("获取资源");
                        semaphore.acquire();

                        Thread.sleep(2000);
                        System.out.println("使用资源");
                        //semaphore.release();

                        System.out.println("释放资源");
                    } catch (Exception e) {
                        System.out.println("异常");
                    }

                }
            }).start();
        }
    }

    @Test
    public void driverCarTest() {
        for (int i = 0; i < 3; i++) {
            Driver driver = new Driver();
            new car(driver).start();
        }
    }

    @Test
    public void ReentrantReadWriteLockTest() {
        for (int i=0;i<6;i++){
            new Thread(new Runnable() {
                public void run() {
                    ReentrantReadWriteLockDome demo=new ReentrantReadWriteLockDome();
                    demo.get("1");
                    System.out.println(demo.get("1")+"读数据");
                }
            }).start();

        }
        for (int i=0;i<4;i++){
            new Thread(new Runnable() {
                public void run() {
                    ReentrantReadWriteLockDome demo=new ReentrantReadWriteLockDome();
                    demo.put("1","hehe");
                }
            }).start();
        }
    }

    //实例使用semaphore
    class Driver {
        final Semaphore semaphore = new Semaphore(1);

        public void driverCar() {
            try {
                semaphore.acquire();
                System.out.println("司机开始开车" + Thread.currentThread().getName() + "时间" + System.currentTimeMillis());

                semaphore.release();
                System.out.println("司机下车" + Thread.currentThread().getName() + "时间" + System.currentTimeMillis());

            } catch (Exception e) {
                System.out.println("线程异常");
            }
        }
    }

    class car extends Thread {
        private Driver driver;

        car(Driver driver) {
            super();
            this.driver = driver;
        }

        @Override
        public void run() {
            driver.driverCar();
        }
    }

    //读写锁的使用
    class ReentrantReadWriteLockDome{

        private Map<String, Object> map = new HashMap<String, Object>();
        private final ReentrantReadWriteLock rrwl = new ReentrantReadWriteLock();
        public Object get(String id) {
            Object data=map.get(id);
            try {
                rrwl.readLock().lock();
                if (map.get(id) != null) {
                    System.out.println("map有数据" + Thread.currentThread().getName() + "数据：" + data);

                } else {
                    System.out.println("没有数据"+Thread.currentThread().getName());
                    data=null;
                }
            } catch (Exception e){
                System.out.println("中断异常");
            }finally {
                rrwl.readLock().unlock();
            }
            return data;
        }

        public void put(String id,String data){
            rrwl.writeLock().lock();
            map.put(id,data);
            System.out.println(Thread.currentThread().getName()+"写数据"+data);
            rrwl.writeLock().unlock();
        }
    }
}

```





#### 5.3AQS源码详解

 这里我们说下Node。Node结点是对每一个等待获取资源的线程的封装，其包含了需要同步的线程本身及其等待状态，如是否被阻塞、是否等待唤醒、是否已经被取消等。变量waitStatus则表示当前Node结点的等待状态，共有5种取值CANCELLED、SIGNAL、CONDITION、PROPAGATE、0。

- **CANCELLED**(1)：表示当前结点已取消调度。当timeout或被中断（响应中断的情况下），会触发变更为此状态，进入该状态后的结点将不会再变化。
- **SIGNAL**(-1)：表示后继结点在等待当前结点唤醒。后继结点入队时，会将前继结点的状态更新为SIGNAL。
- **CONDITION**(-2)：表示结点等待在Condition上，当其他线程调用了Condition的signal()方法后，CONDITION状态的结点将**从等待队列转移到同步队列中**，等待获取同步锁。
- **PROPAGATE**(-3)：共享模式下，前继结点不仅会唤醒其后继结点，同时也可能会唤醒后继的后继结点。
- **0**：新结点入队时的默认状态。

注意，**负值表示结点处于有效等待状态，而正值表示结点已被取消。所以源码中很多地方用>0、<0来判断结点的状态是否正常**。

##### 3.1 acquire(int)

　　此方法是独占模式下线程获取共享资源的顶层入口。如果获取到资源，线程直接返回，否则进入等待队列，直到获取到资源为止，且整个过程忽略中断的影响。这也正是lock()的语义，当然不仅仅只限于lock()。获取到资源后，线程就可以去执行其临界区代码了。下面是acquire()的源码：

```Java
1 public final void acquire(int arg) {
2     if (!tryAcquire(arg) &&
3         acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
4         selfInterrupt();
5 }
```

　　函数流程如下：

1. 调用自定义同步器的tryAcquire()尝试直接去获取资源，如果成功则直接返回；
2. 没成功，则addWaiter()将该线程加入等待队列的尾部，并标记为独占模式；
3. acquireQueued()使线程在等待队列中休息，有机会时（轮到自己，会被unpark()）会去尝试获取资源。获取到资源后才返回。如果在整个等待过程中被中断过，则返回true，否则返回false。
4. 如果线程在等待过程中被中断过，它是不响应的。只是获取资源后才再进行自我中断selfInterrupt()，将中断补上。

##### 3.1.2 addWaiter(Node)

通过tryAcquire()和addWaiter()，该线程获取资源失败，已经被放入等待队列尾部了。聪明的你立刻应该能想到该线程下一部该干什么了吧：**进入等待状态休息，直到其他线程彻底释放资源后唤醒自己，自己再拿到资源，然后就可以去干自己想干的事了**。没错，就是这样！是不是跟医院排队拿号有点相似~~acquireQueued()就是干这件事：**在等待队列中排队拿号（中间没其它事干可以休息），直到拿到号后再返回**。

1. 结点进入队尾后，检查状态，找到安全休息点；
2. 调用park()进入waiting状态，等待unpark()或interrupt()唤醒自己；
3. 被唤醒后，看自己是不是有资格能拿到号。如果拿到，head指向当前结点，并返回从入队到拿到号的整个过程中是否被中断过；如果没拿到，继续流程1。

##### 3.2 release(int)

逻辑并不复杂。它调用tryRelease()来释放资源。有一点需要注意的是，**它是根据tryRelease()的返回值来判断该线程是否已经完成释放掉资源了！所以自定义同步器在设计tryRelease()的时候要明确这一点！！**

　跟tryAcquire()一样，这个方法是需要独占模式的自定义同步器去实现的。正常来说，tryRelease()都会成功的，因为这是独占模式，该线程来释放资源，那么它肯定已经拿到独占资源了，直接减掉相应量的资源即可(state-=arg)，也不需要考虑线程安全的问题。但要注意它的返回值，上面已经提到了，**release()是根据tryRelease()的返回值来判断该线程是否已经完成释放掉资源了！**所以自义定同步器在实现时，如果已经彻底释放资源(state=0)，要返回true，否则返回false。

这个函数并不复杂。一句话概括：**用unpark()唤醒等待队列中最前边的那个未放弃线程**，这里我们也用s来表示吧。此时，再和acquireQueued()联系起来，s被唤醒后，进入if (p == head && tryAcquire(arg))的判断（即使p!=head也没关系，它会再进入shouldParkAfterFailedAcquire()寻找一个安全点。这里既然s已经是等待队列中最前边的那个未放弃线程了，那么通过shouldParkAfterFailedAcquire()的调整，s也必然会跑到head的next结点，下一次自旋p==head就成立啦），然后s把自己设置成head标杆结点，表示自己已经获取到资源了，acquire()也返回了！！And then, DO what you WANT!





### 6.集合框架的多线程实现类：

CopyOnWriteArrayList、CopyOnWriteArraySet、ConcurrentHashMap、ConcurrentSkipListMap、ConcurrentSkipListSet、ArrayBlockingQueue、LinkedBlockingQueue、ConcurrentLinkedQueue、ConcurrentLinkedDeque



### 7.常见的编程题

#### 1.循环输出ABC

```Java

import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;
public class ABC_Lock {
    private static Lock lock = new ReentrantLock();// 通过JDK5中的Lock锁来保证线程的访问的互斥
    private static int state = 0;//通过state的值来确定是否打印
    static class ThreadA extends Thread {
        @Override
        public void run() {
            for (int i = 0; i < 10;) {
                try {
                    lock.lock();
                    while (state % 3 == 0) {// 多线程并发，不能用if，必须用循环测试等待条件，避免虚假唤醒
                        System.out.print("A");
                        state++;
                        i++;
                    }
                } finally {
                    lock.unlock();// unlock()操作必须放在finally块中
                }
            }
        }
    }
    static class ThreadB extends Thread {
        @Override
        public void run() {
            for (int i = 0; i < 10;) {
                try {
                    lock.lock();
                    while (state % 3 == 1) {
                        System.out.print("B");
                        state++;
                        i++;
                    }
                } finally {
                    lock.unlock();// unlock()操作必须放在finally块中
                }
            }
        }
    }
    static class ThreadC extends Thread {
        @Override
        public void run() {
            for (int i = 0; i < 10;) {
                try {
                    lock.lock();
                    while (state % 3 == 2) {
                        System.out.print("C");
                        state++;
                        i++;
                    }
                } finally {
                    lock.unlock();// unlock()操作必须放在finally块中
                }
            }
        }
    }
    public static void main(String[] args) {
        new ThreadA().start();
        new ThreadB().start();
        new ThreadC().start();
    }
}
```



#### 2.写一个死锁

![img](https://img-blog.csdnimg.cn/2021032320341937.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMjYyMzcy,size_16,color_FFFFFF,t_70)

  1.破坏互斥条件：这个条件无法破坏，因为我们用锁本来就是让他们单独拥有

  2.破坏请求与保持条件：一次性把所有进程需要的资源全部拿走。这样就不会在运行的途中进行再去申请资源了。

3.破坏不剥夺条件：以退为进。当某个线程申请不到资源的时候，把自己拥有的资源都释放

4.破坏循环等待条件：靠按序申请资源来预防。按某一顺序申请资源，释放资源则反序释放。
![img](https://img-blog.csdnimg.cn/20210323203455375.png)

![img](https://img-blog.csdnimg.cn/20210323203506999.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMjYyMzcy,size_16,color_FFFFFF,t_70)



## JVM

### 1.什么是OOM？常见有哪些OOM

1 StackOverFlowError

示例：递归调用后方法特别多，将栈空间撑爆

2 java.lang.OutOfMemoryError: Java heap space

*循环创建字符串对象*

3 java.lang.OutOfMemoryError: GC overhead limit exceeded

简单理解：GC回收时间过长，时间过多耗费在GC中，但是回收效果不佳

4 java.lang.OutOfMemoryError: Direct buffer memory

直接内存溢出

原因分析：

直接内存崩溃，此处元空间并不在虚拟机中，而是使用本地内存，与GC无关

5 java.lang.OutOfMemoryError: unable to create new native thread

高并发情况下会出现该异常，该异常与对应的平台有关

原因分析

一个应用进程创建太多的线程，超过系统承载极限。如Linux默认允许单个进程可以创建的线程数是1024个。

6 java.lang.OutOfMemoryError: Metaspace

metaspace存放数据：（永久代 JAVA8之后被metaspace取代了）

- 虚拟机加载的类信息
- 常量池
- 静态变量
- 即时编译后的代码



### 2.触发Full GC的场景及应对策略

年轻代空间（包括 Eden 和 Survivor 区域）回收内存被称为 Minor GC，对老年代GC称为MajorGC，而Full GC是对整个堆来说的，在最近几个版本的JDK里默认包括了对永生带即方法区的回收（JDK8中无永生带了），出现Full GC的时候经常伴随至少一次的Minor GC,但非绝对的。MajorGC的速度一般会比Minor GC慢10倍以上。

触发Full GC的场景及应对策略： 

   1.System.gc()方法的调用，应对策略：通过-XX:+DisableExplicitGC来禁止调用System.gc ;

   2.老年代代空间不足，应对策略：让对象在Minor GC阶段被回收，让对象在新生代多存活一段时间，不要创建过大的对象及数组;

   3.永生区空间不足，应对策略：增大PermGen空间

   4.GC时出现promotionfailed和concurrent mode failure，应对策略：增大survivor space

  5.Minor GC后晋升到旧生代的对象大小大于老年代的剩余空间，应对策略：增大Tenured space 或下调CMSInitiatingOccupancyFraction=60

   \6.  内存持续增涨达到上限导致Full GC  ，应对策略：通过dumpheap 分析是否存在内存泄漏



### 3.内存泄露与内存溢出

**1. 内存泄漏（memory leak ）**

申请了内存用完了不释放，比如一共有 1024M 的内存，分配了 521M 的内存一直不回收，那么可以用的内存只有 521M 了，仿佛泄露掉了一部分；

通俗一点讲的话，内存泄漏就是【占着茅坑不拉shi】。

**2. 内存溢出（out of memory）**

申请内存时，没有足够的内存可以使用；

通俗一点儿讲，一个厕所就三个坑，有两个站着茅坑不走的（内存泄漏），剩下最后一个坑，厕所表示接待压力很大，这时候一下子来了两个人，坑位（内存）就不够了，内存泄漏变成内存溢出了。

可见，内存泄漏和内存溢出的关系：内存泄露的增多，最终会导致内存溢出。



### 4.jvm内存模型

JVM的内存模型主要包括：堆内存、方法区（包括运行时常量池）、栈内存（包括虚拟机栈和本地方法栈）、程序计数器。

从共享范围上来划分可分为两类：线程共享区域（堆和方法区）；线程私有区域（虚拟机栈、本地方法栈和程序计数器）。

从堆概念来划分可分为两类：堆内存和非堆内存。

<img src="https://res-static.hc-cdn.cn/fms/img/920a064b98226df12fda18e8298d7b121614891585695.png" alt="【JVM】JVM内存模型（JVMMM）2" style="zoom:60%;" />







### 5.堆相关

#### 5.1新生代和老年代，为什么是8:1:1

新生代中的对象98%都是“朝生夕死”的（即：将被回收的对象：存活的对象 > 9：1），所以如果根据复制算法完成按照1：1的比例划分新生代的内存空间，将会造成相当大的浪费。

只存在少量存活的对象，只需复制少量存活的对象，远远比标记和标记整理高效多。

复制算法所需要的担保内存 = 9：1，这样即使所有的对象都不会存活，那么也只会“浪费”10%的内存空间。不过我们也无法保证存活的对象一定<2%或10%，当新生代中Survivor to区内存不够用时，就会触发老年代的**担保机制**进行分配担保。

为什么一个Survivor区不行？第一部分中，我们知道了必须设置Survivor区。假设现在只有一个survivor区，我们来模拟一下流程： 刚刚新建的对象在Eden中，一旦Eden满了，触发一次Minor GC，Eden中的存活对象就会被移动到Survivor区。这样继续循环下去，下一次Eden满了的时候，问题来了，此时进行Minor GC，Eden和Survivor各有一些存活对象，如果此时把Eden区的存活对象硬放到Survivor区，很明显这两部分对象所占有的内存是不连续的，也就导致了内存**碎片化**。

那么，顺理成章的，应该建立两块Survivor区，刚刚新建的对象在Eden中，经历一次Minor GC，Eden中的存活对象就会被移动到第一块survivor space S0，Eden被清空；等Eden区再满了，就再触发一次Minor GC，Eden和S0中的存活对象又会被复制送入第二块survivor space S1（**这个过程非常重要，因为这种复制算法保证了S1中来自S0和Eden两部分的存活对象占用连续的内存空间，避免了碎片化的发生**）。S0和Eden被清空，然后下一轮S0与S1交换角色，如此循环往复。如果对象的复制次数达到16次，该对象就会被送到老年代中。





#### 5.2是用什么垃圾收集器

- jdk1.7 默认垃圾收集器Parallel Scavenge（新生代）+Parallel Old（老年代）
- jdk1.8 默认垃圾收集器Parallel Scavenge（新生代）+Parallel Old（老年代）
- jdk1.9 默认垃圾收集器G1



#### 5.3.CMS垃圾收集器

CMS是老年代垃圾收集器，在收集过程中可以与用户线程并发操作。它可以与Serial收集器和Parallel New收集器搭配使用。CMS牺牲了系统的吞吐量来追求收集速度，适合追求垃圾收集速度的服务器上。可以通过JVM启动参数：`-XX:+UseConcMarkSweepGC`来开启CMS。

初始标记(CMS-initial-mark) ,会导致stw;

并发标记(CMS-concurrent-mark)，与用户线程同时运行；

预清理（CMS-concurrent-preclean），与用户线程同时运行；

可被终止的预清理（CMS-concurrent-abortable-preclean） 与用户线程同时运行；

重新标记(CMS-remark) ，会导致swt；

并发清除(CMS-concurrent-sweep)，与用户线程同时运行；

并发重置状态等待下次CMS的触发(CMS-concurrent-reset)，与用户线程同时运行

初始标记

这是CMS中两次stop-the-world事件中的一次。这一步的作用是标记存活的对象，有两部分：

1. 标记老年代中所有的GC Roots对象，如下图节点1；
2. 标记年轻代中活着的对象引用到的老年代的对象（指的是年轻带中还存活的引用类型对象，引用指向老年代中的对象）

在Java语言里，可作为GC Roots对象的包括如下几种：  1. 虚拟机栈(栈桢中的本地变量表)中的引用的对象 ；  2. 方法区中的类静态属性引用的对象 ；  3. 方法区中的常量引用的对象 ；  4. 本地方法栈中JNI的引用的对象；  ps：为了加快此阶段处理速度，减少停顿时间，可以开启初始标记并行化，XX:+CMSParallelInitialMarkEnabled，同时调大并行标记的线程数，线程数不要超过cpu的核数。

并发标记

从“初始标记”阶段标记的对象开始找出所有存活的对象; 因为是并发运行的，在运行期间会发生新生代的对象晋升到老年代、或者是直接在老年代分配对象、或者更新老年代对象的引用关系等等，对于这些对象，都是需要进行重新标记的，否则有些对象就会被遗漏，发生漏标的情况。为了提高重新标记的效率，该阶段会把上述对象所在的Card标识为Dirty，后续只需扫描这些Dirty Card的对象，避免扫描整个老年代； 并发标记阶段只负责将引用发生改变的Card标记为Dirty状态，不负责处理

重新标记

这个阶段会导致第二次stop the word，该阶段的任务是完成标记整个年老代的所有的存活对象。 这个阶段，重新标记的内存范围是整个堆，包含_young_gen和_old_gen。为什么要扫描新生代呢，因为对于老年代中的对象，如果被新生代中的对象引用，那么就会被视为存活对象，即使新生代的对象已经不可达了，也会使用这些不可达的对象当做cms的“gc root”，来扫描老年代； 因此对于老年代来说，引用了老年代中对象的新生代的对象，也会被老年代视作“GC ROOTS”:当此阶段耗时较长的时候，可以加入参数-XX:+CMSScavengeBeforeRemark，在重新标记之前，先执行一次ygc，回收掉年轻带的对象无用的对象，并将对象放入幸存带或晋升到老年代，这样再进行年轻带扫描时，只需要扫描幸存区的对象即可，一般幸存带非常小，这大大减少了扫描时间。 由于之前的预处理阶段是与用户线程并发执行的，这时候可能年轻带的对象对老年代的引用已经发生了很多改变，这个时候，remark阶段要花很多时间处理这些改变，会导致很长stop the word，所以通常CMS尽量运行Final Remark阶段在年轻代是足够干净的时候。 另外，还可以开启并行收集：-XX:+CMSParallelRemarkEnabled。

并发清理

通过以上5个阶段的标记，老年代所有存活的对象已经被标记并且现在要通过Garbage Collector采用清扫的方式回收那些不能用的对象了。 这个阶段主要是清除那些没有标记的对象并且回收空间； 由于CMS并发清理阶段用户线程还在运行着，伴随程序运行自然就还会有新的垃圾不断产生，这一部分垃圾出现在标记过程之后，CMS无法在当次收集中处理掉它们，只好留待下一次GC时再清理掉。这一部分垃圾就称为“浮动垃圾”。



**优点：** CMS是一款优秀的收集器，它的主要优点在名字上已经体现出来了：并发收集、低停顿。

**缺点：** **CMS收集器对CPU资源非常敏感** 其实，面向并发设计的程序都对CPU资源比较敏感。在并发阶段，它虽然不会导致用户线程停顿，但是会因为占用了一部分线程（或者说CPU资源）而导致应用程序变慢，总吞吐量会降低。 CMS默认启动的回收线程数是（CPU数量+3）/ 4，也就是当CPU在4个以上时，并发回收时垃圾收集线程不少于25%的CPU资源，并且随着CPU数量的增加而下降。但是当CPU不足4个（譬如2个）时，CMS对用户程序的影响就可能变得很大。

**CMS收集器无法处理浮动垃圾** CMS收集器无法处理浮动垃圾，可能出现“Concurrent Mode Failure”失败而导致另一次Full GC的产生。

由于CMS并发清理阶段用户线程还在运行着，伴随程序运行自然就还会有新的垃圾不断产生，这一部分垃圾出现在标记过程之后，CMS无法在当次收集中处理掉它们，只好留待下一次GC时再清理掉。这一部分垃圾就称为“浮动垃圾”。 也是由于在垃圾收集阶段用户线程还需要运行，那也就还需要预留有足够的内存空间给用户线程使用，因此CMS收集器不能像其他收集器那样等到老年代几乎完全被填满了再进行收集，需要预留一部分空间提供并发收集时的程序运作使用。要是CMS运行期间预留的内存无法满足程序需要，就会出现一次“Concurrent Mode Failure”失败，这时虚拟机将启动后备预案：临时启用Serial Old收集器来重新进行老年代的垃圾收集，这样停顿时间就很长了。

**CMS收集器会产生大量空间碎片** **CMS是一款基于“标记—清除”算法实现的收集器，这意味着收集结束时会有大量空间碎片产生。**

空间碎片过多时，将会给大对象分配带来很大麻烦，往往会出现老年代还有很大空间剩余，但是无法找到足够大的连续空间来分配当前对象，不得不提前触发一次Full GC。





#### 5.4G1垃圾收集器

G1的各代存储地址是不连续的，每一代都使用了n个不连续的大小相同的Region，每个Region占有一块连续的虚拟内存地址

- G1最大的特点是引入分区的思路，弱化了分代的概念。
- 合理利用垃圾收集各个周期的资源，解决了其他收集器甚至CMS的众多缺陷。

- 算法： G1基于标记-整理算法, 不会产生空间碎片，分配大对象时不会无法得到连续的空间而提前触发一次FULL GC。
- 停顿时间可控： G1可以通过设置预期停顿时间（Pause Time）来控制垃圾收集时间避免应用雪崩现象。
- 并行与并发：G1能更充分的利用CPU，多核环境下的硬件优势来缩短stop the world的停顿时间。

- 服务端多核CPU、JVM内存占用较大的应用（至少大于4G）
- 应用在运行过程中会产生大量内存碎片、需要经常压缩空间
- 想要更可控、可预期的GC停顿周期，防止高并发下应用雪崩现象

**特性：** G1（Garbage-First）是一款面向服务端应用的垃圾收集器。HotSpot开发团队赋予它的使命是未来可以**替换掉JDK 1.5中发布的CMS收集器。**与其他GC收集器相比，G1具备如下特点。

**并行与并发** G1能充分利用多CPU、多核环境下的硬件优势，使用多个CPU来缩短Stop-The-World停顿的时间，部分其他收集器原本需要停顿Java线程执行的GC动作，G1收集器仍然可以通过并发的方式让Java程序继续执行。

**分代收集** 与其他收集器一样，分代概念在G1中依然得以保留。虽然G1可以不需要其他收集器配合就能**独立管理整个GC堆**，但它能够采用不同的方式去处理新创建的对象和已经存活了一段时间、熬过多次GC的旧对象以获取更好的收集效果。

**空间整合** 与CMS的“标记—清理”算法不同，**G1从整体来看是基于“标记—整理”算法实现的收集器，从局部（两个Region之间）上来看是基于“复制”算法实现的**，但无论如何，这两种算法都意味着G1运作期间**不会产生内存空间碎片**，收集后能提供规整的可用内存。这种特性有利于程序长时间运行，分配大对象时不会因为无法找到连续内存空间而提前触发下一次GC。

**可预测的停顿** 这是G1相对于CMS的另一大优势，降低停顿时间是G1和CMS共同的关注点，但G1除了追求低停顿外，还能建立可预测的停顿时间模型，能让使用者明确指定在一个长度为M毫秒的时间片段内，消耗在垃圾收集上的时间不得超过N毫秒。

在G1之前的其他收集器进行收集的范围都是整个新生代或者老年代，而G1不再是这样。使用G1收集器时，Java堆的内存布局就与其他收集器有很大差别，它将整个Java堆划分为多个大小相等的独立区域（Region），虽然还保留有新生代和老年代的概念，但新生代和老年代不再是物理隔离的了，它们都是一部分Region（不需要连续）的集合。

G1收集器之所以能建立可预测的停顿时间模型，是因为它可以有计划地避免在整个Java堆中进行全区域的垃圾收集。G1跟踪各个Region里面的垃圾堆积的价值大小（回收所获得的空间大小以及回收所需时间的经验值），在后台维护一个优先列表，每次根据允许的收集时间，优先回收价值最大的Region（这也就是Garbage-First名称的来由）。这种使用Region划分内存空间以及有优先级的区域回收方式，保证了G1收集器在有限的时间内可以获取尽可能高的收集效率。

**执行过程：** G1收集器的运作大致可划分为以下几个步骤：

**初始标记（Initial Marking）** 初始标记阶段仅仅只是标记一下GC Roots能直接关联到的对象，并且修改TAMS（Next Top at Mark Start）的值，让下一阶段用户程序并发运行时，能在正确可用的Region中创建新对象，这阶段需要停顿线程，但耗时很短。

**并发标记（Concurrent Marking）** 并发标记阶段是从GC Root开始对堆中对象进行可达性分析，找出存活的对象，这阶段耗时较长，但可与用户程序并发执行。

**最终标记（Final Marking）** 最终标记阶段是为了修正在并发标记期间因用户程序继续运作而导致标记产生变动的那一部分标记记录，虚拟机将这段时间对象变化记录在线程Remembered Set Logs里面，最终标记阶段需要把Remembered Set Logs的数据合并到Remembered Set中，这阶段需要停顿线程，但是可并行执行。

**筛选回收（Live Data Counting and Evacuation）** 筛选回收阶段首先对各个Region的回收价值和成本进行排序，根据用户所期望的GC停顿时间来制定回收计划，这个阶段其实也可以做到与用户程序一起并发执行，但是因为只回收一部分Region，时间是用户可控制的，而且停顿用户线程将大幅提高收集效率。





#### 5.5垃圾回收算法

标记-清除算法、复制算法、标记-整理算法、分代收集算法



#### 5.6可达性分析GC Roots

1. 虚拟机栈(栈帧中的本地变量表)中引用的对象。(可以理解为:引用栈帧中的本地变量表的所有对象)
2. 方法区中静态属性引用的对象(可以理解为:引用方法区该静态属性的所有对象)
3. 方法区中常量引用的对象(可以理解为:引用方法区中常量的所有对象)
4. 本地方法栈中(Native方法)引用的对象(可以理解为:引用Native方法的所有对象)



####  5.7ZGC

与CMS中的ParNew和G1类似，ZGC也采用标记-复制算法，不过ZGC对该算法做了重大改进：ZGC在标记、转移和重定位阶段几乎都是并发的，这是ZGC实现停顿时间小于10ms目标的最关键原因。

ZGC只有三个STW阶段：**初始标记**，**再标记**，**初始转移**。其中，初始标记和初始转移分别都只需要扫描所有GC Roots，其处理时间和GC Roots的数量成正比，一般情况耗时非常短；再标记阶段STW时间很短，最多1ms，超过1ms则再次进入并发标记阶段。即，ZGC几乎所有暂停都只依赖于GC Roots集合大小，停顿时间不会随着堆的大小或者活跃对象的大小而增加。与ZGC对比，G1的转移阶段完全STW的，且停顿时间随存活对象的大小增加而增加。

ZGC通过着色指针和读屏障技术，解决了转移过程中准确访问对象的问题，实现了并发转移。大致原理描述如下：并发转移中“并发”意味着GC线程在转移对象的过程中，应用线程也在不停地访问对象。假设对象发生转移，但对象地址未及时更新，那么应用线程可能访问到旧地址，从而造成错误。而在ZGC中，应用线程访问对象将触发“读屏障”，如果发现对象被移动了，那么“读屏障”会把读出来的指针更新到对象的新地址上，这样应用线程始终访问的都是对象的新地址。那么，JVM是如何判断对象被移动过呢？就是利用对象引用的地址，即着色指针。

着色指针是一种将信息存储在指针中的技术。ZGC仅支持64位系统，它把64位虚拟地址空间划分为多个子空间，当应用程序创建对象时，首先在堆空间申请一个虚拟地址，但该虚拟地址并不会映射到真正的物理地址。ZGC同时会为该对象在M0、M1和Remapped地址空间分别申请一个虚拟地址，且这三个虚拟地址对应同一个物理地址，但这三个空间在同一时间有且只有一个空间有效。ZGC之所以设置三个虚拟地址空间，是因为它使用“空间换时间”思想，去降低GC停顿时间。“空间换时间”中的空间是虚拟空间，而不是真正的物理空间。后续章节将详细介绍这三个空间的切换过程。与上述地址空间划分相对应，ZGC实际仅使用64位地址空间的第0~41位，而第42~45位存储元数据，第47~63位固定为0。ZGC将对象存活信息存储在42~45位中，这与传统的垃圾回收并将对象存活信息放在对象头中完全不同。

读屏障是JVM向应用代码插入一小段代码的技术。当应用线程从堆中读取对象引用时，就会执行这段代码。需要注意的是，仅“从堆中读取对象引用”才会触发这段代码。



#### 5.8逃逸分析

逃逸分析：当一个对象在方法中被定义后，它可能被外部方法引用，例如作为调用传入传入参数传递到其他方法中，成为方法逃逸；有些可能被其他线程访问到，譬如赋值给类变量    或者在其他线程中访问的实例变量，称为线程逃逸。

逃逸分析能分析出一个对象会不会逃逸到方法或线程之外，如果不会发生逃逸，那么就会进行一些高效的优化：

栈上分配(针对方法逃逸)：将不会逃逸的对象分配到栈上，对象随着方法的结束而结束，减少 GC 系统的压力
同步消除(针对线程逃逸)：如果其他线程不访问该对象，那么我们可以把同步措施取消掉
标量替换：将对象的一个聚合产物分解成多个基本成员变量。这个方法不是创建对象了，直接创建对象里面的成员变量到栈上分配与读写。

#### 5.9G1和CMS区别

- CMS
  - 首先是支持并发清理，线程运行的时候依旧可以进行垃圾回收
    - 所以不能清理浮动垃圾【缺点】
    - 追求低停顿时间
  - 垃圾回收算法是Mark-Sweep，会产生大量的空间碎片
- G1
  - 使用Mark-Compact算法，不会产生内存碎片
  - 可以预测停顿时间





### 6.强、软、弱、虚引用以及区别

1. 强引用(StrongReference)：**强引用**是使用最普遍的引用。如果一个对象具有强引用，那**垃圾回收器**绝不会回收它，当**内存空间不足**时，`Java`虚拟机宁愿抛出`OutOfMemoryError`错误，使程序**异常终止**，也不会靠随意**回收**具有**强引用**的**对象**来解决内存不足的问题。

2. 软引用(SoftReference)：如果一个对象只具有**软引用**，则**内存空间充足**时，**垃圾回收器**就**不会**回收它；如果**内存空间不足**了，就会**回收**这些对象的内存。只要垃圾回收器没有回收它，该对象就可以被程序使用。软引用可用来实现内存敏感的高速缓存。**软引用**可以和一个**引用队列**(`ReferenceQueue`)联合使用。如果**软引用**所引用对象被**垃圾回收**，`JAVA`虚拟机就会把这个**软引用**加入到与之关联的**引用队列**中。
3. 弱引用(WeakReference)：**弱引用**与**软引用**的区别在于：只具有**弱引用**的对象拥有**更短暂**的**生命周期**。在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有**弱引用**的对象，不管当前**内存空间足够与否**，都会**回收**它的内存。不过，由于垃圾回收器是一个**优先级很低的线程**，因此**不一定**会**很快**发现那些只具有**弱引用**的对象。
4. 虚引用(PhantomReference)：**虚引用**顾名思义，就是**形同虚设**。与其他几种引用都不同，**虚引用**并**不会**决定对象的**生命周期**。如果一个对象**仅持有虚引用**，那么它就和**没有任何引用**一样，在任何时候都可能被垃圾回收器回收。**虚引用**主要用来**跟踪对象**被垃圾回收器**回收**的活动。虚引用必须和引用队列(ReferenceQueue)联合使用。当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之关联的引用队列中。

| 引用类型 | 被垃圾回收时间 | 用途               | 生存时间          |
| -------- | -------------- | ------------------ | ----------------- |
| 强引用   | 从来不会       | 对象的一般状态     | JVM停止运行时终止 |
| 软引用   | 当内存不足时   | 对象缓存           | 内存不足时终止    |
| 弱引用   | 正常垃圾回收时 | 对象缓存           | 垃圾回收后终止    |
| 虚引用   | 正常垃圾回收时 | 跟踪对象的垃圾回收 | 垃圾回收后终止    |



### 7.对象创建存储相关

#### 7.1Java对象的创建过程

1：检查类是否已经被加载；

虚拟机遇到一条 new 指令的时候，首先去常量池中检查该对象的符号引用，并检查该引用是否被加载过、初始化过、解析过。如果没有，就要去执行类加载过程。

2：为对象分配内存空间；

在类加载检查通过后，接下来虚拟机将为新生对象分配内存。对象所需的内存大小在类加载完成后便可确定。***\*分配\****方式有***\*两种\****：”***\*指针碰撞\****”和“***\*空闲列表\****”两种，选择那种分配方式由Java ***\*堆是否规整\****决定，而 Java 堆***\*是否规整\****又由所采用的***\*垃圾收集\****器***\*是否带有压缩整理\****功能所决定。

指针碰撞：
适用场合：堆内存规整(没有内存碎片)的情况(复制算法，标记压缩算法)

原理：用过的内存全部整合到一边，其中用一个指针来分隔，来了一个新对象，指针往没有用过内存的地方移动。

GC 收集器:serial(标记压缩),parallel(serial 的多线程版本)

空闲列表
使用场合：堆内存不规整,有内存碎片(标记清楚算法)

原理：虚拟机会维护一个列表，该列表中会记录那些内存块是可用的，在分配的是偶，找一块足够大的内存块来创建对象实例，然后更新列表。

内存分配并发问题
在创建对象中，我们肯定不能允许另外的线程来干扰，就比如你女票被男的骚扰了，你爽吗？所以我们虚拟机在创建对象的时候要保证线程安全。通常也有两种方式来保证创建对象    是线程安全的：

CAS+失败重试：

CAS 是乐观锁的一种实现。乐观锁是指，它每次都假设没有其他线程来干扰的，如果有线程干扰，那就重新创建，直到创建成功。这样可以保证更新操作的原子性。

TLAB:

为每一个线程预先在 Eden 区域分配一块内存，首先 TLAB 分配，对象的需要的内存大于了 TLAB 提供的，再采用 CAS 进行内存分配。

3：为分配的内存空间初始化零值（为对象字段设置零值）；

当内存分配完成后，虚拟机需要将分配到的内存空间都初始化为零值(不包括对象头)，这    一步操作保证了对象的实例字段在 Java 代码中可以不赋初值就直接使用，程序能访问到这些字段的数据类型所对应的零值。就跟有些成员变量你赋值了，有些没有赋值，那么那    些没有赋值的就是 Null 的道理是一样的。

4：对对象进行其他设置（设置对象头）；

初始化完成后，我们需要一个东西去辨认我们这个新创建对象的一些信息。很多事物的基本信息都存在什么头，比如 http，它的大概属性都会存在信息头中，比如请求方式之类的。当然我们这个新创建对象也是一样的，我们就用对象头来存储对象是那个类的实例、类的元数据信息、对象的哈希码、对象的 GC 分代年龄等信息。

5：执行构造方法。



#### 7.2.**对象在内存的布局：**

对象创建完成后在内存中保存了保存的信息包括对象头、实例数据及对齐填充三类信息。

**对象头：**
对象头里主要包括几类信息，分别是锁状态标志、持有锁的线程ID、GC分代年龄、对象HashCode，类元信息地址、数组长度，这里并没有对对象头里的每个信息都列出而是进行大致的分类，下面是对其中几类信息进行说明。
**锁状态标志：**对象的加锁状态分为无锁、偏向锁、轻量级锁、重量级锁几种标记。
**持有锁的线程**： 持有当前对象锁定的线程ID。
**GC分代年龄：** 对象每经过一次GC还存活下来了，GC年龄就加1。
**类元信息地址：** 可通过对象找到类元信息，用于定位对象类型。
**数组长度：** 当对象是数组类型的时候会记录数组的长度。

![Java 锁与对象头2](https://res-static.hc-cdn.cn/fms/img/bfc0197b088782c58f52555c83d00fb61603856918315.jpg)

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy95aWJiT056ZHRGZjBybFdRZmNUWFFHWTdYaWJTZ1JxWW9vM2sxWmljQWg5dEpvclczU2JtTlIzbG9aQ0llUDdGdlNiZUlJMzEyZEY5WmVBTFNpY2tUYnl1cFEvNjQw?x-oss-process=image/format,png)



**实例数据**
对象实例数据才是对象的自身真正的数据，主要包括自身的成员变量信息，同时还包括实现的接口、父类的成员变量信息。

**对齐填充**
根据JVM规范对象申请的内存地址必须是8的倍数，换句话说对象在申请内存大小时候8字节的倍数，如果对象自身的信息大小没有达到申请的内存大小，那么这部分是对剩余部分进行填充。





#### 7.3.类加载过程以及类加载器

加载：类加载过程的一个阶段：通过一个类的完全限定查找此类字节码文件，并利用字节码文件创建一个Class对象

验证：目的在于确保Class文件的字节流中包含信息符合当前虚拟机要求，不会危害虚拟机自身安全。主要包括四种验证，文件格式验证，元数据验证，字节码验证，符号引用验证。

准备：为类变量(即static修饰的字段变量)分配内存并且设置该类变量的初始值即0(如static int i=5;这里只将i初始化为0，至于5的值将在初始化时赋值)，这里不包含用final修饰的static，因为final在编译的时候就会分配了，注意这里不会为实例变量分配初始化，类变量会分配在方法区中，而实例变量是会随着对象一起分配到Java堆中。

解析：主要将常量池中的符号引用替换为直接引用的过程。符号引用就是一组符号来描述目标，可以是任何字面量，而直接引用就是直接指向目标的指针、相对偏移量或一个间接定位到目标的句柄。有类或接口的解析，字段解析，类方法解析，接口方法解析(这里涉及到字节码变量的引用，如需更详细了解，可参考《深入Java虚拟机》)。

初始化：类加载最后阶段，若该类具有超类，则对其进行初始化，执行静态初始化器和静态初始化成员变量(如前面只初始化了默认值的static变量将会在这个阶段赋值，成员变量也将被初始化)。
启动类加载器主要加载的是JVM自身需要的类，这个类加载使用C++语言实现的，是虚拟机自身的一部分，它负责将 <JAVA_HOME>/lib路径下的核心类库或-Xbootclasspath参数指定的路径下的jar包加载到内存中，注意必由于虚拟机是按照文件名识别加载jar包的，如rt.jar，如果文件名不被虚拟机识别，即使把jar包丢到lib目录下也是没有作用的(出于安全考虑，Bootstrap启动类加载器只加载包名为java、javax、sun等开头的类)。
扩展类加载器是指Sun公司(已被Oracle收购)实现的sun.misc.Launcher$ExtClassLoader类，由Java语言实现的，是Launcher的静态内部类，它负责加载<JAVA_HOME>/lib/ext目录下或者由系统变量-Djava.ext.dir指定位路径中的类库，开发者可以直接使用标准扩展类加载器。
系统（System）类加载器
也称应用程序加载器是指 Sun公司实现的sun.misc.Launcher$AppClassLoader。它负责加载系统类路径java -classpath或-D java.class.path 指定路径下的类库，也就是我们经常用到的classpath路径，开发者可以直接使用系统类加载器，一般情况下该类加载是程序中默认的类加载器，通过ClassLoader#getSystemClassLoader()方法可以获取到该类加载器。





#### 7.4双亲委派模型

采用双亲委派模式的是好处是Java类随着它的类加载器一起具备了一种带有优先级的层次关系，通过这种层级关可以避免类的重复加载，当父亲已经加载了该类时，就没有必要子ClassLoader再加载一次。其次是考虑到安全因素，java核心api中定义类型不会被随意替换，假设通过网络传递一个名为java.lang.Integer的类，通过双亲委托模式传递到启动类加载器，而启动类加载器在核心Java API发现这个名字的类，发现该类已被加载，并不会重新加载网络传递的过来的java.lang.Integer，而直接返回已加载过的Integer.class，这样便可以防止核心API库被随意篡改。可能你会想，如果我们在classpath路径下自定义一个名为java.lang.SingleInterge类(该类是胡编的)呢？该类并不存在java.lang中，经过双亲委托模式，传递到启动类加载器中，由于父类加载器路径下并没有该类，所以不会加载，将反向委托给子类加载器加载，最终会通过系统类加载器加载该类。但是这样做是不允许，因为java.lang是核心API包，需要访问权限，强制加载将会报出如下异常
双亲委派模型的破坏者-线程上下文类加载器

破坏双亲委派模型

线程上下文加载器（Thread Context ClassLoader）。这个类加载器可以通过 java.lang.Thread 类的 setContextClassLoader()（原文在这里写成 setContextLoaser） 方法进行设置，如果创建线程时还未设置，它将从父线程中继承一个，如果在应用程序的全局范围内没有设置过的话，那么这个类加载器默认就是应用程序类加载器。

以 JDBC 源码来看看是如何破坏双亲委派模型的。 JDBC 也属于 SPI，其 Driver 接口是 Java 核心类库的一部分，但是 Driver 的实现类却是由第三方实现，是需要使用系统类加载器进行加载的，符合上述说的情况。

Tomcat 中的类加载器



#### 7.5对象的访问定位有哪两种方式？

如果使用句柄的话，那么 Java 堆中将会划分出一块内存来作为句柄池，reference 中存储的就是对象的句柄地址，而句柄中包含了对象实例数据与类型数据各自的具体地址信息;

Reference 中直接存的就是 Java 堆中的对象实例数据，然后堆中空间里有指向对象类型数据的地址。

使用句柄的好处是 reference 中存储的是稳定的句柄地址，在对象被移动时指挥改变句柄中的实例数据指针，而 reference 本身不需要修改。使用直接指针访问方式最大的好处就是速度快，节省了一次指针定位的时间开销。Hotspot 默认的时直接指针。



### 8.gc分析问题常用指令

评判 GC 的两个核心指标：

- **延迟（Latency）：**也可以理解为最大停顿时间，即垃圾收集过程中一次 STW 的最长时间，越短越好，一定程度上可以接受频次的增大，GC 技术的主要发展方向。
- **吞吐量（Throughput）：**应用系统的生命周期内，由于 GC 线程会占用 Mutator 当前可用的 CPU 时钟周期，吞吐量即为 Mutator 有效花费的时间占系统总运行时间的百分比，例如系统运行了 100 min，GC 耗时 1 min，则系统吞吐量为 99%，吞吐量优先的收集器可以接受较长的停顿。

简而言之即为**一次停顿的时间不超过应用服务的 TP9999，GC 的吞吐量不小于 99.99%。**

重点需要关注的几个GC Cause：

- **System.gc()：**手动触发GC操作。
- **CMS：**CMS GC 在执行过程中的一些动作，重点关注 CMS Initial Mark 和 CMS Final Remark 两个 STW 阶段。
- **Promotion Failure：**Old 区没有足够的空间分配给 Young 区晋升的对象（即使总可用内存足够大）。
- **Concurrent Mode Failure：**CMS GC 运行期间，Old 区预留的空间不足以分配给新的对象，此时收集器会发生退化，严重影响 GC 性能，下面的一个案例即为这种场景。

- **GCLocker Initiated GC：**如果线程执行在 JNI 临界区时，刚好需要进行 GC，此时 GC Locker 将会阻止 GC 的发生，同时阻止其他线程进入 JNI 临界区，直到最后一个线程退出临界区时触发一次 GC。

- 标准终端类：jps、jinfo、jstat、jstack、jmap
- 功能整合类：jcmd、vjtools、**arthas、**greys
- 简易：JConsole、JVisualvm、HA、GCHisto、GCViewer
- 进阶：MAT、JProfiler





### 9.jvm调优过程

**对JVM内存的系统级的调优主要的目的是减少GC的频率和Full GC的次数。**

**JVM调优的一般步骤为：**

   第1步：分析GC日志及dump文件，判断是否需要优化，确定瓶颈问题点；

   第2步：确定JVM调优量化目标；

   第3步：确定JVM调优参数（根据历史JVM参数来调整）；

   第4步：调优一台服务器，对比观察调优前后的差异；

   第5步：不断的分析和调整，直到找到合适的JVM参数配置；

   第6步：找到最合适的参数，将这些参数应用到所有服务器，并进行后续跟踪。

**重要参数（可调优）解析：**

-Xms12g：初始化堆内存大小为12GB。

-Xmx12g：堆内存最大值为12GB 。

-Xmn2400m：新生代大小为2400MB，包括 Eden区与2个Survivor区。

-XX:SurvivorRatio=1：Eden区与一个Survivor区比值为1:1。

-XX:MaxDirectMemorySize=1G：直接内存。报java.lang.OutOfMemoryError: Direct buffer memory 异常可以上调这个值。

-XX:+DisableExplicitGC：禁止运行期显式地调用 System.gc() 来触发fulll GC。

注意: Java RMI的定时GC触发机制可通过配置-Dsun.rmi.dgc.server.gcInterval=86400来控制触发的时间。

-XX:CMSInitiatingOccupancyFraction=60：老年代内存回收阈值，默认值为68。

-XX:ConcGCThreads=4：CMS垃圾回收器并行线程线，推荐值为CPU核心数。

-XX:ParallelGCThreads=8：新生代并行收集器的线程数。

-XX:MaxTenuringThreshold=10：设置垃圾最大年龄。如果设置为0的话，则年轻代对象不经过Survivor区，直接进入年老代。对于年老代比较多的应用，可以提高效率。如果将此值设置为一个较大值，则年轻代对象会在Survivor区进行多次复制，这样可以增加对象再年轻代的存活时间，增加在年轻代即被回收的概论。

-XX:CMSFullGCsBeforeCompaction=4：指定进行多少次fullGC之后，进行tenured区 内存空间压缩。

-XX:CMSMaxAbortablePrecleanTime=500：当abortable-preclean预清理阶段执行达到这个时间时就会结束。



**1.监控GC的状态**

使用各种JVM工具，查看当前日志，分析当前JVM参数设置，并且分析当前堆内存快照和gc日志，根据实际的各区域内存划分和GC执行时间，觉得是否进行优化。

**举一个例子： 系统崩溃前的一些现象：**

- 每次垃圾回收的时间越来越长，由之前的10ms延长到50ms左右，FullGC的时间也有之前的0.5s延长到4、5s
- FullGC的次数越来越多，最频繁时隔不到1分钟就进行一次FullGC
- 年老代的内存越来越大并且每次FullGC后年老代没有内存被释放

之后系统会无法响应新的请求，逐渐到达OutOfMemoryError的临界值，这个时候就需要分析JVM内存快照dump。



**2.生成堆的dump文件、分析dump文件**

通过JMX的MBean生成当前的Heap信息，大小为一个3G（整个堆的大小）的hprof文件，如果没有启动JMX可以通过Java的jmap命令来生成该文件。

**3.分析dump文件**

打开这个3G的堆信息文件，显然一般的Window系统没有这么大的内存，必须借助高配置的Linux，几种工具打开该文件：

- Visual VM
- IBM HeapAnalyzer
- JDK 自带的Hprof工具
- **Mat(Eclipse专门的静态内存分析工具)推荐使用**

备注：文件太大，建议使用Eclipse专门的静态内存分析工具Mat打开分析。

**4.分析结果，判断是否需要优化**

如果各项参数设置合理，系统没有超时日志出现，GC频率不高，GC耗时不高，那么没有必要进行GC优化，如果GC时间超过1-3秒，或者频繁GC，则必须优化。

**注：如果满足下面的指标，则一般不需要进行GC：**

- Minor GC执行时间不到50ms；
- Minor GC执行不频繁，约10秒一次；
- Full GC执行时间不到1s；
- Full GC执行频率不算频繁，不低于10分钟1次；



**5.调整GC类型和内存分配**

如果内存分配过大或过小，或者采用的GC收集器比较慢，则应该优先调整这些参数，并且先找1台或几台机器进行beta，然后比较优化过的机器和没有优化的机器的性能对比，并有针对性的做出最后选择。

**6.不断的分析和调整**

通过不断的试验和试错，分析并找到最合适的参数，如果找到了最合适的参数，则将这些**参数应用到所有服务器。**

**JVM调优量化目标（示例）：**

   1. Heap 内存使用率 <= 70%;

   2. Old generation内存使用率<= 70%;

   3. avgpause <= 1秒; 

4. Full gc 次数0 或 avg pause interval >= 24小时 ;

   注意：不同应用，其JVM调优量化目标是不一样的







### 10.具体场景可以用来举例

到底是结果（现象）还是原因，在一次 GC 问题处理的过程中，如何判断是 GC 导致的故障，还是系统本身引发 GC 问题。这里继续拿在本文开头提到的一个 Case：“GC 耗时增大、线程 Block 增多、慢查询增多、CPU 负载高等四个表象，如何判断哪个是根因？”，笔者这里根据自己的经验大致整理了四种判断方法供参考：

- **时序分析：**先发生的事件是根因的概率更大，通过监控手段分析各个指标的异常时间点，还原事件时间线，如先观察到 CPU 负载高（要有足够的时间 Gap），那么整个问题影响链就可能是：CPU 负载高 -> 慢查询增多 -> GC 耗时增大 -> 线程 Block 增多 -> RT 上涨。
- **概率分析：**使用统计概率学，结合历史问题的经验进行推断，由近到远按类型分析，如过往慢查的问题比较多，那么整个问题影响链就可能是：慢查询增多 -> GC 耗时增大 ->  CPU 负载高   -> 线程 Block 增多 -> RT上涨。
- **实验分析：**通过故障演练等方式对问题现场进行模拟，触发其中部分条件（一个或多个），观察是否会发生问题，如只触发线程 Block 就会发生问题，那么整个问题影响链就可能是：线程Block增多  -> CPU 负载高  -> 慢查询增多  -> GC 耗时增大 ->  RT 上涨。
- **反证分析：**对其中某一表象进行反证分析，即判断表象的发不发生跟结果是否有相关性，例如我们从整个集群的角度观察到某些节点慢查和 CPU 都正常，但也出了问题，那么整个问题影响链就可能是：GC 耗时增大 -> 线程 Block 增多 ->  RT 上涨。

不同的根因，后续的分析方法是完全不同的。如果是 CPU 负载高那可能需要用火焰图看下热点、如果是慢查询增多那可能需要看下 DB 情况、如果是线程 Block 引起那可能需要看下锁竞争的情况，最后如果各个表象证明都没有问题，那可能 GC 确实存在问题，可以继续分析 GC 问题了。

场景一：动态扩容引起的空间震荡-服务**刚刚启动时** GC 次数较多，最大空间剩余很多但是依然发生 GC

在 JVM 的参数中 -Xms 和 -Xmx 设置的不一致，在初始化时只会初始 -Xms 大小的空间存储信息，每当空间不够用时再向操作系统申请，这样的话必然要进行一次 GC。

场景二：显式 GC 的去与留-System.gc 会引发一次 STW 的 Full GC，对整个堆做收集。

此外 JVM 还提供了 **-XX:+ExplicitGCInvokesConcurrent  和 -XX:+ExplicitGCInvokesConcurrentAndUnloadsClasses**  参数来将 System.gc 的触发类型从 Foreground 改为 Background，同时 Background 也会做 Reference Processing，这样的话就能大幅降低了 STW 开销，同时也不会发生 NIO Direct Memory OOM。

MetaSpace 区 OOM-JVM 在启动后或者某个时间点开始**，MetaSpace 的已使用大小在持续增长，同时每次 GC 也无法释放，调大 MetaSpace 空间也无法彻底解决**。

可以 dump 快照之后通过 JProfiler 或 MAT 观察 Classes 的 Histogram（直方图） 即可，或者直接通过命令即可定位， jcmd 打几次 Histogram 的图，看一下具体是哪个包下的 Class 增加较多就可以定位了。

场景四：过早晋升-对象晋升年龄较小/full gc 频繁且一次gc之后变化很大

如果是 **Young/Eden 区过小**，我们可以在总的 Heap 内存不变的情况下适当增大 Young 区，具体怎么增加？一般情况下 Old 的大小应当为活跃对象的 2~3 倍左右，考虑到浮动垃圾问题最好在 3 倍左右，剩下的都可以分给 Young 区。

如果是分配速率过大：

- 偶发较大：通过内存分析工具找到问题代码，从业务逻辑上做一些优化。
- 一直较大：当前的 Collector 已经不满足 Mutator 的期望了，这种情况要么扩容 Mutator 的 VM，要么调整 GC 收集器类型或加大空间。

场景五：CMS Old GC 频繁*Old 区频繁的 CMS GC，但是每次耗时不是特别长，整体最大 STW 也在可接受范围内，但由于 GC 太频繁导致吞吐下降比较多。

基本都是一次 Young GC 完成后，负责处理 CMS GC 的一个后台线程 concurrentMarkSweepThread 会不断地轮询，使用 shouldConcurrentCollect() 方法做一次检测，判断是否达到了回收条件。如果达到条件，使用 collect_in_background() 启动一次 Background 模式 GC。轮询的判断是使用 sleepBeforeNextCycle() 方法，间隔周期为 -XX:CMSWaitDuration 决定，默认为2s。

 场景六：单次 CMS Old GC 耗时长*

- 【方向】观察详细 GC 日志，找到出问题时 Final Remark 日志，分析下 Reference 处理和元数据处理 real 耗时是否正常，详细信息需要通过 -XX:+PrintReferenceGC 参数开启。**基本在日志里面就能定位到大概是哪个方向出了问题，耗时超过 10% 的就需要关注。**

- 对 FinalReference 的分析主要观察 java.lang.ref.Finalizer 对象的 dominator tree，找到泄漏的来源。经常会出现问题的几个点有 Socket 的 SocksSocketImpl 、Jersey 的 ClientRuntime、Mysql 的 ConnectionImpl 等等。
- scrub symbol table 表示清理元数据符号引用耗时，符号引用是 Java 代码被编译成字节码时，方法在 JVM 中的表现形式，生命周期一般与 Class 一致，当 _should_unload_classes 被设置为 true 时在 CMSCollector::refProcessingWork() 中与 Class Unload、String Table 一起被处理。

- **Trade Off：**与 CAP 注定要缺一角一样，GC 优化要在延迟（Latency）、吞吐量（Throughput）、容量（Capacity）三者之间进行权衡。
- **最终手段：**GC 发生问题不是一定要对 JVM 的 GC 参数进行调优，大部分情况下是通过 GC 的情况找出一些业务问题，切记上来就对 GC 参数进行调整，当然有明确配置错误的场景除外。
- **控制变量：**控制变量法是在蒙特卡洛（Monte Carlo）方法中用于减少方差的一种技术方法，我们调优的时候尽量也要使用，每次调优过程尽可能只调整一个变量。
- **善用搜索：**理论上 99.99% 的 GC 问题基本都被遇到了，我们要学会使用搜索引擎的高级技巧，重点关注 StackOverFlow、Github 上的 Issue、以及各种论坛博客，先看看其他人是怎么解决的，会让解决问题事半功倍。能看到这篇文章，你的搜索能力基本过关了~
- **调优重点：**总体上来讲，我们开发的过程中遇到的问题类型也基本都符合正态分布，太简单或太复杂的基本遇到的概率很低，笔者这里将中间最重要的三个场景添加了“*”标识，希望阅读完本文之后可以观察下自己负责的系统，是否存在上述问题。

- **GC 参数：**如果堆、栈确实无法第一时间保留，一定要保留 GC 日志，这样我们最起码可以看到 GC Cause，有一个大概的排查方向。关于 GC 日志相关参数，最基本的 -XX:+HeapDumpOnOutOfMemoryError 等一些参数就不再提了，笔者建议添加以下参数，可以提高我们分析问题的效率。

| 分类     | 参数                                                         | 作用                                                 |
| :------- | :----------------------------------------------------------- | :--------------------------------------------------- |
| 基本参数 | -XX:+PrintGCDetails、-XX:+PrintGCDateStamps、-XX:+PrintGCTimeStamps | GC 日志的基本参数                                    |
| 时间相关 | -XX:+PrintGCApplicationConcurrentTime、-XX:+PrintGCApplicationStoppedTime | 详细步骤的并行时间，STW 时间等等                     |
| 年龄相关 | -XX:+PrintTenuringDistribution                               | 可以观察 GC 前后的对象年龄分布，方便发现过早晋升问题 |
| 空间变化 | -XX:+PrintHeapAtGC                                           | 各个空间在 GC 前后的回收情况，非常详细               |
| 引用相关 | -XX:+PrintReferenceGC                                        | 观察系统的软引用，弱引用，虚引用等回收情况           |



### 11.字节码的结构

![img](https://img-blog.csdnimg.cn/20191028230444892.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE0NDI3NjA=,size_16,color_FFFFFF,t_70)





## 设计模式

### 1.单例模式

懒汉式

```java
public class Singleton {  
    private static Singleton instance;  
    private Singleton (){}  
  
    public static Singleton getInstance() {  
    if (instance == null) {  
        instance = new Singleton();  
    }  
    return instance;  
    }  
}
```

饿汉式

```java
public class Singleton {  
    private static Singleton instance = new Singleton();  
    private Singleton (){}  
    public static Singleton getInstance() {  
    return instance;  
    }  
}
```

双重锁

```java
public class Singleton {  
    private volatile static Singleton singleton;  
    private Singleton (){}  
    public static Singleton getSingleton() {  
    if (singleton == null) {  
        synchronized (Singleton.class) {  
        if (singleton == null) {  
            singleton = new Singleton();  
        }  
        }  
    }  
    return singleton;  
    }  
}
```

静态内部类

```java
public class Singleton {  
    private static class SingletonHolder {  
    private static final Singleton INSTANCE = new Singleton();  
    }  
    private Singleton (){}  
    public static final Singleton getInstance() {  
    return SingletonHolder.INSTANCE;  
    }  
}
```





### 2.设计模式七大原则

| 标记 | 设计模式原则名称  | 简单定义                                         |
| :--- | :---------------- | :----------------------------------------------- |
| OCP  | 开闭原则          | 对扩展开放，对修改关闭                           |
| SRP  | 单一职责原则      | 一个类只负责一个功能领域中的相应职责             |
| LSP  | 里氏代换原则      | 所有引用基类的地方必须能透明地使用其子类的对象   |
| DIP  | 依赖倒转原则      | 依赖于抽象，不能依赖于具体实现                   |
| ISP  | 接口隔离原则      | 类之间的依赖关系应该建立在最小的接口上           |
| CARP | 合成/聚合复用原则 | 尽量使用合成/聚合，而不是通过继承达到复用的目的  |
| LOD  | 迪米特法则        | 一个软件实体应当尽可能少的与其他实体发生相互作用 |



### 3.代理模式

静态代理：

创建一个接口，然后创建被代理的类实现该接口并且实现该接口中的抽象方法。之后再创建一个代理类，同时使其也实现这个接口。在代理类中持有一个被代理对象的引用，而后在代理类方法中调用该对象的方法。

使用JDK静态代理很容易就完成了对一个类的代理操作。但是`JDK`静态代理的缺点也暴露了出来：由于代理只能为一个类服务，如果需要代理的类很多，那么就需要编写大量的代理类，比较繁琐

JDK动态代理：

**使用JDK动态代理的五大步骤：**

1. 通过实现InvocationHandler接口来自定义自己的InvocationHandler；
2. 通过`Proxy.getProxyClass`获得动态代理类；
3. 通过反射机制获得代理类的构造方法，方法签名为`getConstructor(InvocationHandler.class)`；
4. 通过构造函数获得代理对象并将自定义的`InvocationHandler`实例对象传为参数传入；
5. 通过代理对象调用目标方法；

JDK静态代理与JDK动态代理之间有些许相似，比如说都要创建代理类，以及代理类都要实现接口等。

**不同之处：** 在静态代理中我们需要对哪个接口和哪个被代理类创建代理类，所以我们在编译前就需要代理类实现与被代理类相同的接口，并且直接在实现的方法中调用被代理类相应的方法；但是动态代理则不同，我们不知道要针对哪个接口、哪个被代理类创建代理类，因为它是在运行时被创建的。

**一句话来总结一下JDK静态代理和JDK动态代理的区别：**

JDK静态代理是通过直接编码创建的，而`JDK`动态代理是利用反射机制在运行时创建代理类的。

CGLib：

CGLIB包的底层是通过使用一个小而快的字节码处理框架`ASM`，来转换字节码并生成新的类

**CGLIB代理实现如下：**

1. 首先实现一个MethodInterceptor，方法调用会被转发到该类的intercept()方法。 
2. 然后在需要使用的时候，通过CGLIB动态代理获取代理对象。

JDK代理要求被代理的类必须实现接口，有很强的局限性。

而CGLIB动态代理则没有此类强制性要求。简单的说，`CGLIB`会让生成的代理类继承被代理类，并在代理类中对代理方法进行强化处理(前置处理、后置处理等)。





### 4.观察者模式/监听器模式

观察者（Observer）模式的定义：指多个对象间存在一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并被自动更新。这种模式有时又称作发布-订阅模式、模型-视图模式，它是对象行为型模式。

观察者模式是一种对象行为型模式，其主要优点如下。

1. 降低了目标与观察者之间的耦合关系，两者之间是抽象耦合关系。符合依赖倒置原则。
2. 目标与观察者之间建立了一套触发机制。

观察者模式的主要角色如下。

1. 抽象主题（Subject）角色：也叫抽象目标类，它提供了一个用于保存观察者对象的聚集类和增加、删除观察者对象的方法，以及通知所有观察者的抽象方法。
2. 具体主题（Concrete Subject）角色：也叫具体目标类，它实现抽象目标中的通知方法，当具体主题的内部状态发生改变时，通知所有注册过的观察者对象。
3. 抽象观察者（Observer）角色：它是一个抽象类或接口，它包含了一个更新自己的抽象方法，当接到具体主题的更改通知时被调用。
4. 具体观察者（Concrete Observer）角色：实现抽象观察者中定义的抽象方法，以便在得到目标的更改通知时更新自身的状态。

刚开始接触监听器的时候，很是不理解为什么我点击按钮(触发事件)监听器会自动运行，而且每当我应用监听器处理事件的时候，就会困惑不已。学完观察者模式之后，渐渐的对这个问题有了一个清晰地认识。
首先，创建监听器对象(具体观察者对象),然后将监听器添加到事件源(具体主题角色也可以叫被观察者对象)上，最后事件源变化触发事件(具体主题角色状态改变，通知观察者)！其实就是观察者模式的实现。

监听器模型涉及以下三个对象，模型图如下：

（1）事件：用户对组件的一个操作，或者说程序执行某个方法，称之为一个事件，如机器人程序执行工作。
（2）事件源：发生事件的组件就是事件源，也就是被监听的对象，如机器人可以工作，可以跳舞，那么就可以把机器人看做是一个事件源。
（3）事件监听器（处理器）：监听并负责处理事件的方法，如监听机器人工作情况，在机器人工作前后做出相应的动作，或者获取机器人的状态信息。

执行顺序如下：

1、给事件源注册监听器。
2、组件接受外部作用，也就是事件被触发。
3、组件产生一个相应的事件对象，并把此对象传递给与之关联的事件处理器。
4、事件处理器启动，并执行相关的代码来处理该事件。

监听器模式：事件源注册监听器之后，当事件源触发事件，监听器就可以回调事件对象的方法；更形象地说，监听者模式是基于：注册-回调的事件/消息通知处理模式，就是被监控者将消息通知给所有监控者。 

1、注册监听器：事件源.setListener。
2、回调：事件源实现onListener。





### 5.工厂模式

##### 1. 创建统一的接口

```
public interface IceCream {

    public void taste();
}
```

##### 2. 不同的实现类

```
public class AppleIceCream implements IceCream {

    public void taste(){
        System.out.println("这是苹果口味的冰激凌");
    }
}

public class BananaIceCream implements IceCream {

    public void taste() {
        System.out.println("这是香蕉口味的冰激凌");
    }
}

public class OrangeIceCream implements IceCream{

    public void taste(){
        System.out.println("这是橘子口味的冰激凌");
    }
}
```

##### 3. 创建工厂

```
public class IceCreamFactory {

    public static IceCream creamIceCream(String taste){

        IceCream iceCream = null; //这里是关键-为什么要继承自同一接口

        // 这里我们通过switch来判断，具体制作哪一种口味的冰激凌
        switch(taste){

            case "Apple":
                iceCream = new AppleIceCream();
                break;

            case "Orange":
                iceCream = new OrangeIceCream();
                break;

            case "Banana":
                iceCream = new BananaIceCream();
                break;

            default:
                break;
        }

        return iceCream;
    }
}
```

##### 4. 客户端调用

```
// 通过统一的工厂，传入不同参数调用生产冰激凌的方法去生产不同口味的冰激凌
public class Client {

    public static void main(String[] args) {

        IceCream appleIceCream = IceCreamFactory.creamIceCream("Apple");
        appleIceCream.taste();

        IceCream bananaIceCream = IceCreamFactory.creamIceCream("Banana");
        bananaIceCream.taste();

        IceCream orangeIceCream = IceCreamFactory.creamIceCream("Orange");
        orangeIceCream.taste(); 
    }
}
```





## Spring框架知识

### spring

#### 1.springIOC相关

##### 1.**Spring IoC 的容器构建流程**

1.构造函数
2.注册组件
包括自定义加入容器的组件，各种后置通知，监听器，增强器，beanFactory等等
3.刷新容器
创建容器和AOP切面增强

![img](https://filescdn.proginn.com/e58214bf96c75cd435847ededc272c61/a59f9c06cbad43f4106852ba0bc2b18b.webp)

- Spring是怎么将这些bean初始化到IOC容器的呢？

初始化bean工厂 > 通过BeanDefinitionReader从不同的来源读取配置文件 > 执行针对bean工厂本身的BeanFactoryPostProcessor > 通过反射递归实例化bean(过程中调用BeanPostProcessor) > 建立整个应用上下文 > 一些后续工作(清除缓存，发布事件之类)

- IOC容器具体是什么呢？

所谓IOC容器，其实就是一个Map<String，Object>的数据结构



##### 2.spring IOC 定义

**IOC是指在程序开发过程中，对象实例的创建不再由调用者管理，而是由Spring容器创建，Spring容器会负责控制程序之间的关系，而不是由代码直接控制，因此，控制权由程序代码转移到了Spring容器，控制权发生了反转，即控制反转。** **Spring IOC提供了两种IOC容器，分别是BeanFactory和ApplicationContext。**

**ApplicationContext是BeanFactory的字接口，也被称为应用上下文，不仅提供了BeanFactory的所有功能，还添加了对国际化、资源访问、事件传播等方面的支持。**

 Spring容器在创建被调用实例时，会自动将调用者需要的对象实例注入为调用者，这样，通过 Spring容器获得被调用者实例，成为依赖注入。





##### 3.依赖注入的方式

- 属性setter注入

- 构造构造器注入

- 静态工厂方式实例化

- 实例工厂实例化





##### 4.Bean的会话状态

状态会话的Bean 每个用户有自己特有的一个实例，在用户的生存期内，bean保持可用户的信息，即"有状态"，一旦用户灭亡(调用结束或实例结束)，bean的生命周期也会结束。

无状态会话的Bean bean一旦被实例化就被加到会话池中，各个用户可以公用，即使用户死亡，bean的生命周期也不一定结束，他可能依然存在于会话池中，供其他用户使用，由于没有特定的用户，也就没办法保存用户的状态，所以叫无状态Bean，但无状态Bean并非没有状态，如果它有自己的属性，那么这些属性就会受到所有调用它的用户的影响。





##### 5.Servlet的线程安全问题

Servlet体系结构是建立在Java多线程机制上的，它的生命周期是由web容器负责的，一个Servlet类Application中只有一个实例存在，也就是有多个线程在使用这个实例，这是单例模式的使用。Servlet本身是无状态的，无状态的单例是线程安全的，但是如果在Servlet中使用了实例变量，那么就会变成有状态了，是非线程安全的。

- 避免使用实例变量
- 避免使用非线程安全集合
- 在多个Servlet中对某个外部对象的修改进行加锁操作





##### 6.Spring中Bean的作用域

在Spring配置文件中，使用的scope属性设置Bean的作用域

singleton 单例模式，使用singleton定义的Bean在Spring容器中只有一个实例，这也是Bean的默认作用域，所有的Bean请求，只要id与该Bean定义相匹配，就只会返回Bean的同一个实例。适用于无回话状态的Bean，例如(DAO层、Service层)。

prototype 原型模式，每次通过Spring容器获取prototype定义的Bean时，容器都会创建一个新的Bean实例，适用于需要需要保持会话状态的Bean(比如Struts2的Action类)。

request 在一次HTTP请求中，容器会返回该Bean的同一个实例，而对于不同的HTTP请求，会返回不同的实例，该作用域仅在当前HttpRequest内有效

session 在一次HttpSession中，容器会返回该Bean的同一个实例，而对于不同的HTTP请求，会返回不同的实例，该作用域仅在当前HttpSession内有效

global Session 在一个全局的session中，容器会返回该Bean的同一个实例，该作用域仅在使用portlet context时有效。





##### 7.Spring Bean的生命周期

![img](https://user-gold-cdn.xitu.io/2019/12/24/16f37c264fc41968?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

- 根据配置文件的配置调用Bean的构造方法或者工厂方法实例化Bean
- 利用依赖注入完成Bean中所有属性值的配置注入
- 如果Bean实现了BeanNameAware接口，则Spring调用Bean的setBeanName方法传入当前Bean的id值
- 如果Bean实现了BeanFactoryAware接口，则Spring调用setBeanFactory方法传入当前工厂实例的引用
- 如果Bean实现了ApplicationContextAware接口，则Spring调用setApplicationContext方法传入当前ApplicationContext实例的引用
- 如果BeanPostProcessor和Bean关联，则Spring将调用该接口的预初始化方法postProcessBeforeInitialization对Bean进行加工，Spring AOP就是利用它实现的
- 如果Bean实现了InitializingBean接口，则Spring调用afterPropertiesSet方法
- 如果在配置文件中通过init-method属性指定了初始化方法，则调用该初始化方法
- 如果BeanPostProcessor 和Bean关联，则Spring将调用该接口的初始化方法postPrecessAfterInitialization，此时，Bean已经可以使用
- 如果在中指定了该Bean的作用范围为scope="singleton"，则将该Bean放入Spring IOC的缓存池中，将触发Spring对该Bean的生命周期管理，若在中指定了该Bean的作用范围为scope="prototype"，则将该Bean交给调用者，调用者管理该Bean的生命周期，Spring不再管理该Bean
- 如果Bean实现了DisposableBean接口，则Spring会调用destroy方法将Spring中的Bean销毁，如果在配置文件中通过destroy-method属性指定了Bean的销毁方法，则Spring将调用该方法对Bean进行销毁





##### 8.常用注解

@Component：可以使用此注解描述Spring中的Bean，但它是一个泛化的概念，仅仅表示一个组件(Bean)，并且可以用在任何层次

@Repository：用于将数据访问层(DAO层)的类标示为Spring中的Bean

@Service：通常用作在业务层(Service层)，用于将业务层的类标示为Spring中的Bean

@Controlle：通常作用在控制层,用于将控制层的类标示为Spring中的Bean

@Autowired：用于Bean的属性变量、属性的Set方法及构造函数进行标注，配合对应的注解处理器完成Bean的自动配置工作，默认按照Bean的类型进行装配

@Resource：作用与Autowired一样，区别是@Autowired默认按照Bean的类型装配，@而Resource默认按照Bean的实例类型进行装配，@Resource有name、type属性，Spring将name属性解析为Bean实例名称，将type属性解析为Bean的梳理类型，如果指定name属性，则按照实例名称进行装配，如果指定type属性，按照Bean类型进行装配，如果都不指定，则先按照Bean实例名称装配，如果不能装配，则按照Bean的类型进行装配，如果都不能匹配，抛出NoSuchBeanDefinitionException异常

@Qualifier：与@Autowired配合使用，会将默认的按照Bean配型装配修改为按Bean的实例名称装配，Bean的实例名称由@qualifier注解的参数指定





##### 9.@Resource和@Autowired的区别

@Resource和@Autowired都是做bean的注入时使用

@Resource不是Spring中的注解，但是Spring支持该注解，而@Autowired是Spring的注解

@Autowired是按照类型(byType)来装配Bean的，不回去匹配name，默认情况下他要求依赖对象必须存在，如果需允许null，可以设置它的required属性为false，如果想让@Autowired按照名称（byName）来装配，则需要配合@Qualifier一起使用





##### 10.@Resource注解装配步骤

如果同时指定了name和type属性，则从Spring上下文中找到唯一匹配的Bean进行装配，找不到抛出异常

如果指定了name，则从上下文中查找名称(id)匹配的Bean进行装配，找不到抛出异常

如果指定了type，则从上下文查找类型匹配的唯一Bean进行装载，找不到或者是找到了多个，抛出异常

如果即没有指定name，也没有指定type，则默认按照byName进行查找并装载，如果没有匹配，则按照byType进行匹配并装载



##### 11.spring的循环依赖以及解决方案

Spring 解决循环依赖的核心就是提前暴露对象，而提前暴露的对象就是放置于第二级缓存中。下表是三级缓存的说明：

| 名称                  | 描述                                                         |
| --------------------- | ------------------------------------------------------------ |
| singletonObjects      | 一级缓存，存放完整的 Bean。                                  |
| earlySingletonObjects | 二级缓存，存放提前暴露的Bean，Bean 是不完整的，未完成属性注入和执行 init 方法。 |
| singletonFactories    | 三级缓存，存放的是 Bean 工厂，主要是生产 Bean，存放到二级缓存中。 |

实例化 A，此时 A 还未完成属性填充和初始化方法（@PostConstruct）的执行，A 只是一个半成品。

为 A 创建一个 Bean 工厂，并放入到  singletonFactories 中。

发现 A 需要注入 B 对象，但是一级、二级、三级缓存均为发现对象 B。

实例化 B，此时 B 还未完成属性填充和初始化方法（@PostConstruct）的执行，B 只是一个半成品。

为 B 创建一个 Bean 工厂，并放入到  singletonFactories 中。

发现 B 需要注入 A 对象，此时在一级、二级未发现对象 A，但是在三级缓存中发现了对象 A，从三级缓存中得到对象 A，并将对象 A 放入二级缓存中，同时删除三级缓存中的对象 A。（注意，此时的 A 还是一个半成品，并没有完成属性填充和执行初始化方法）

将对象 A 注入到对象 B 中。

对象 B 完成属性填充，执行初始化方法，并放入到一级缓存中，同时删除二级缓存中的对象 B。（此时对象 B 已经是一个成品）

对象 A 得到对象 B，将对象 B 注入到对象 A 中。（对象 A 得到的是一个完整的对象 B）

对象 A 完成属性填充，执行初始化方法，并放入到一级缓存中，同时删除二级缓存中的对象 A。





##### 12.为什么三级缓存

如果 Spring 选择二级缓存来解决循环依赖的话，那么就意味着所有 Bean 都需要在实例化完成之后就立马为其创建代理，而 Spring 的设计原则是在 Bean 初始化完成之后才为其创建代理。所以，Spring 选择了三级缓存。但是因为循环依赖的出现，导致了 Spring 不得不提前去创建代理，因为如果不提前创建代理对象，那么注入的就是原始对象，这样就会产生错误。





##### 13.BeanFactory 和 FactoryBean 的区别

BeanFactory是个Factory，也就是IOC容器或对象工厂，FactoryBean是个Bean。在Spring中，**所有的Bean都是由BeanFactory(也就是IOC容器)来进行管理的**。但对FactoryBean而言，**这个Bean不是简单的Bean，而是一个能生产或者修饰对象生成的工厂Bean,它的实现与设计模式中的工厂模式和修饰器模式类似**，它是实现了FactoryBean<T>接口的Bean，根据该Bean的ID从BeanFactory中获取的实际上是FactoryBean的getObject()返回的对象，而不是FactoryBean本身，如果要获取FactoryBean对象，请在id前面加一个&符号来获取。



##### 14.**Spring 能解决构造函数循环依赖吗**

答案是不行的，对于使用构造函数注入产生的循环依赖，Spring 会直接抛异常。

为什么无法解决构造函数循环依赖？

上面解决逻辑的第一句话：“首先使用构造函数创建一个 “不完整” 的 bean 实例”，从这句话可以看出，构造函数循环依赖是无法解决的，因为当构造函数出现循环依赖，我们连 “不完整” 的 bean 实例都构建不出来。





##### 15**@PostConstruct 修饰的方法里用到了其他 bean 实例，会有问题吗**

1、PostConstruct 注解被封装在 CommonAnnotationBeanPostProcessor中，具体触发时间是在 postProcessBeforeInitialization 方法，从 doCreateBean 维度看，则是在 initializeBean 方法里，属于初始化 bean 阶段。

2、属性的依赖注入是在 populateBean 方法里，属于属性填充阶段。

3、属性填充阶段位于初始化之前，所以本题答案为没有问题。





##### 16.**要在 Spring IoC 容器构建完毕之后执行一些逻辑，怎么实现**

1、比较常见的方法是使用事件监听器，实现 ApplicationListener 接口，监听 ContextRefreshedEvent 事件。

2、还有一种比较少见的方法是实现 SmartLifecycle 接口，并且 isAutoStartup 方法返回 true，则会在 finishRefresh() 方法中被触发。

两种方式都是在 finishRefresh 中被触发，SmartLifecycle在ApplicationListener之前。





##### 17.**Spring 中的常见扩展点有哪些**

1、ApplicationContextInitializer

initialize 方法，在 Spring 容器刷新前触发，也就是 refresh 方法前被触发。

2、BeanFactoryPostProcessor

postProcessBeanFactory 方法，在加载完 Bean 定义之后，创建 Bean 实例之前被触发，通常使用该扩展点来加载一些自己的 bean 定义。

3、BeanPostProcessor

postProcessBeforeInitialization 方法，执行 bean 的初始化方法前被触发；postProcessAfterInitialization 方法，执行 bean 的初始化方法后被触发。

4、@PostConstruct

该注解被封装在 CommonAnnotationBeanPostProcessor 中，具体触发时间是在 postProcessBeforeInitialization 方法。

5、InitializingBean

afterPropertiesSet 方法，在 bean 的属性填充之后，初始化方法（init-method）之前被触发，该方法的作用基本等同于 init-method，主要用于执行初始化相关操作。

6、ApplicationListener，事件监听器

onApplicationEvent 方法，根据事件类型触发时间不同，通常使用的 ContextRefreshedEvent 触发时间为上下文刷新完毕，通常用于 IoC 容器构建结束后处理一些自定义逻辑。

7、@PreDestroy

该注解被封装在 DestructionAwareBeanPostProcessor 中，具体触发时间是在 postProcessBeforeDestruction 方法，也就是在销毁对象之前触发。

8、DisposableBean

destroy 方法，在 bean 的销毁阶段被触发，该方法的作用基本等同于destroy-method，主用用于执行销毁相关操作。





##### 18.Spring处理id相同的bean

1、在spring同一个配置文件中，不能存在id相同的两个bean，否则会报错。

2、在两个不同的spring配置文件中，可以存在id相同的两个bean，启动时，不会报错。这是因为spring ioc容器在加载bean的过程中，类DefaultListableBeanFactory会对id相同的bean进行处理：后加载的配置文件的bean，覆盖先加载的配置文件的bean。DefaultListableBeanFactory类中，有个属性allowBeanDefinitionOverriding，默认值为true，该值就是用来指定出现两个bean的id相同的情况下，如何进行处理。如果该值为false，则不会进行覆盖，而是抛出异常。



##### 19.如何从一个配置文件到一个对象

bean定义对象：定义bean的一些属性，比如作用域，是否懒加载等。

从xml文件到最后的bean拿来应用放在哪里，bean放在IOC容器(是一个概念)中，实际上用数据结构存储，用的Map存储。

既然是一个容器，那么肯定有一个入口，所有Spring容器有一个根接口。**BeanFactory**

选择入口和容器都有了，那么就该放对象了，放对象的操作。

因此在容器中就会有一个beanDefination，给容器添加元素。

有不同类型进行定义Bean，因此其中会有一个统一接口BeanDefinitionReader去解析成统一规格给BeanDefinition。

因此从BeanDefinition无论是new还是反射，都不会到实例化的。

为什么要使用反射？反射性能不是低吗？ 反射只是成十万对象的时候才会出现性能瓶颈，几十个、几百个不印象的是不会成为性能问题。因为反射足够的灵活！！我们想要new一个实例出来，必须要知道这个类。我们在配置文件中写好了类吗？我们在配置文件中是写好了类的完全限定名，我们知道这个限定名，是无法直接获取class对象和字节码文件的。
在进行反射之前，肯定要换成真实变量，所以就要开始慢慢引入把占位符的东西换成我们真实需要的东西。

 这个PostProcessor就是将占位符、空配符换成我们真实的属性。这两个是**不同阶段的操作**，因此要分成BeanFactory和Bean。

BeanDefinition读过来后，我们要进行实例化了，要修改工厂中的某些对象，将那些占位符的修改成真实值。因此经过BeanFactoryPostProcessor将BeanDefinition中一些具有占位符的属性换成我们真实的值。

BeanFactoryPostProcessor可以修改bean的definition参数，上图是自己建立的一个类，然后去实现BeanFactoryPostProcessor接口。

Spring一开始出来的时候，是写xml,后来注解可以用了，我们把注解集成到Spring生态中去。

通过BeanFactoryPostProcessor修改完后，我们就可以通过反射对类进行实例化。
  通过反射获得Class（三种方式：1.Class.forName(类的路径)；2.类名.class；3.对象.forClass()），然后通过Class获得类构造器（.getDelcareConstructor），然后通过类构造器获得实例对象(.getIntance)

  这样创建的对象是不完整的，创建对象是真正包括两块的：
那么回到IOC容器中，我们在实例化后，还有一系列的初始化操作，调用aware接口的方法。Bean的生命周期操作。













#### 2.SprinAOP相关

##### 1.定义

AOP和OOP类似，也是一种编程模式，Spring AOP是基于AOP编程模式的一个框架，它的使用有效减少了系统间的重复代码，达到了模块间解耦的作用。

AOP的全程是"Aspect Oriented Programming"，即面向切面编程，它将业务逻辑的各部分进行隔离，使开发人员在编写业务逻辑时可以专心核心业务，从而提高开发基础。

AOP采取横向抽取机制，取代了传统的纵向继承体系，其应用主要体现在事务处理、日志管理、权限控制、异常处理等方面。

目前最流行的AOP有Spring AOP和AspectJ

Spring AOP使用纯Java实现，不需要专门的编译过程和类加载器，在运行期间通过代理方式向目标类植入增强的代码。

AspectJ是一个基于Java语言的AOP框架，从Spring2.0开始，Spring AOP引入了对AspectJ的支持。





##### 2.AOP相关术语

| 名称              | 说明                                                         |
| ----------------- | ------------------------------------------------------------ |
| Joinpoint(连接点) | 指那些被拦截到的点，在Spring中，可以被动态代理拦截目标类的方法 |
| Pointcut(切入点)  | 指要对哪些Joinpoint进行拦截，即被拦截的连接点                |
| Advice(通知)      | 指拦截到Joinpoint之后要做的事情，即对切入点增强的内容        |
| Target(目标)      | 指代理的目标对象                                             |
| Weaving(植入)     | 指把增强代码应用到目标上，生成代理对象的过程                 |
| Proxy(代理)       | 指生成的代理对象                                             |
| Aspect(切面)      | 切入点和通知的结合                                           |





##### 3.Spring AOP的实现原理

本质是通过动态代理来实现的，主要有以下几个步骤。

1、获取增强器，例如被 Aspect 注解修饰的类。

2、在创建每一个 bean 时，会检查是否有增强器能应用于这个 bean，简单理解就是该 bean 是否在该增强器指定的 execution 表达式中。如果是，则将增强器作为拦截器参数，使用动态代理创建 bean 的代理对象实例。

3、当我们调用被增强过的 bean 时，就会走到代理类中，从而可以触发增强器，本质跟拦截器类似。

`Spring`的`AOP`实现原理其实很简单，就是通过**动态代理**实现的。如果我们为`Spring`的某个`bean`配置了切面，那么`Spring`在创建这个`bean`的时候，实际上创建的是这个`bean`的一个代理对象，我们后续对`bean`中方法的调用，实际上调用的是代理类重写的代理方法。而`Spring`的`AOP`使用了两种动态代理，分别是**JDK的动态代理**，以及**CGLib的动态代理**。

`JDK`实现动态代理需要两个组件，首先第一个就是`InvocationHandler`接口。我们在使用`JDK`的动态代理时，需要编写一个类，去实现这个接口，然后重写`invoke`方法，这个方法其实就是我们提供的代理方法。然后`JDK`动态代理需要使用的第二个组件就是`Proxy`这个类，我们可以通过这个类的`newProxyInstance`方法，返回一个代理对象。生成的代理类实现了原来那个类的所有接口，并对接口的方法进行了代理，我们通过代理对象调用这些方法时，底层将通过反射，调用我们实现的`invoke`方法。

`JDK`的动态代理存在限制，那就是被代理的类必须是一个实现了接口的类，代理类需要实现相同的接口，代理接口中声明的方法。若需要代理的类没有实现接口，此时`JDK`的动态代理将没有办法使用，于是`Spring`会使用`CGLib`的动态代理来生成代理对象。`CGLib`直接操作字节码，生成类的子类，重写类的方法完成代理。









##### 3.说说Spring中的事务传播行为

事务的传播行为，默认值为 Propagation.REQUIRED。可以手动指定其他的事务传播行为，如下：
**（1）Propagation.REQUIRED**

如果当前存在事务，则加入该事务，如果当前不存在事务，则创建一个新的事务。
**（2）Propagation.SUPPORTS**

如果当前存在事务，则加入该事务；如果当前不存在事务，则以非事务的方式继续运行。
**（3）Propagation.MANDATORY**

如果当前存在事务，则加入该事务；如果当前不存在事务，则抛出异常。
**（4）Propagation.REQUIRES_NEW**

重新创建一个新的事务，如果当前存在事务，延缓当前的事务。
**（5）Propagation.NOT_SUPPORTED**

以非事务的方式运行，如果当前存在事务，暂停当前的事务。
**（6）Propagation.NEVER**

以非事务的方式运行，如果当前存在事务，则抛出异常。
**（7）Propagation.NESTED**

如果没有，就新建一个事务；如果有，就在当前事务中嵌套其他事务。





##### 4.JDK动态代理为什么不能代理类

类Proxy的作用简单来说就是保存下用于自定义的InvocationHandler，便于在方法代理时执行自定义InvocationHandler的逻辑。由于$Proxy0已经继承了类Proxy，所以不能再extends一个类了，所以只能implements一个接口了。





##### 5.**Spring 的事务隔离级别**

Spring 的事务隔离级别底层其实是基于数据库的，Spring 并没有自己的一套隔离级别。

DEFAULT：使用数据库的默认隔离级别。

READ_UNCOMMITTED：读未提交，最低的隔离级别，会读取到其他事务还未提交的内容，存在脏读。

READ_COMMITTED：读已提交，读取到的内容都是已经提交的，可以解决脏读，但是存在不可重复读。

REPEATABLE_READ：可重复读，在一个事务中多次读取时看到相同的内容，可以解决不可重复读，但是存在幻读。

SERIALIZABLE：串行化，最高的隔离级别，对于同一行记录，写会加写锁，读会加读锁。在这种情况下，只有读读能并发执行，其他并行的读写、写读、写写操作都是冲突的，需要串行执行。可以防止脏读、不可重复度、幻读，没有并发事务问题。





##### 6.JDK动态代理与Cglib代理的区别

1、JDK 动态代理本质上是实现了被代理对象的接口，而 Cglib 本质上是继承了被代理对象，覆盖其中的方法。

2、JDK 动态代理只能对实现了接口的类生成代理，Cglib 则没有这个限制。但是 Cglib 因为使用继承实现，所以 Cglib 无法代理被 final 修饰的方法或类。

3、在调用代理方法上，JDK 是通过反射机制调用，Cglib是通过FastClass 机制直接调用。FastClass 简单的理解，就是使用 index 作为入参，可以直接定位到要调用的方法直接进行调用。

4、在性能上，JDK1.7 之前，由于使用了 FastClass 机制，Cglib 在执行效率上比 JDK 快，但是随着 JDK 动态代理的不断优化，从 JDK 1.7 开始，JDK 动态代理已经明显比 Cglib 更快了。





##### 7.**Spring 事务的实现原理**

Spring 事务的底层实现主要使用的技术：AOP（动态代理） + ThreadLocal + try/catch。

动态代理：基本所有要进行逻辑增强的地方都会用到动态代理，AOP 底层也是通过动态代理实现。

ThreadLocal：主要用于线程间的资源隔离，以此实现不同线程可以使用不同的数据源、隔离级别等等。

try/catch：最终是执行 commit 还是 rollback，是根据业务逻辑处理是否抛出异常来决定。





### springSecurity原理

![640?wx_fmt=png](https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_png/GvtDGKK4uYngjiaicZ6UqibkEHRhzUg8JYLz2G6ILGaaibJe3fOl7LSDyBmHFJy3wdJqmVKdYKUYxQALGibSef9QJRA/640?wx_fmt=png)

客户端发起一个请求，进入 Security 过滤器链。

当到 LogoutFilter 的时候判断是否是登出路径，如果是登出路径则到 logoutHandler ，如果登出成功则到 logoutSuccessHandler 登出成功处理，如果登出失败则由 ExceptionTranslationFilter ；如果不是登出路径则直接进入下一个过滤器。

当到 UsernamePasswordAuthenticationFilter 的时候判断是否为登录路径，如果是，则进入该过滤器进行登录操作，如果登录失败则到 AuthenticationFailureHandler 登录失败处理器处理，如果登录成功则到 AuthenticationSuccessHandler 登录成功处理器处理，如果不是登录请求则不进入该过滤器。

当到 FilterSecurityInterceptor 的时候会拿到 uri ，根据 uri 去找对应的鉴权管理器，鉴权管理器做鉴权工作，鉴权成功则到 Controller 层否则到 AccessDeniedHandler 鉴权失败处理器处理。

### springMVC

<img src="https://img-blog.csdnimg.cn/20210426130950153.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMjYyMzcy,size_16,color_FFFFFF,t_70" alt="img" style="zoom:80%;" />

#### 1.**MVC流程，@RequestMapping的注解具体怎么实现的？**

  1.Spring扫描所有的Bean

  2.遍历这些bean，依次判断是否是处理器，并检测其HandlerMethod

  3.遍历Handler中的所有方法，找出其中被@RequestMapping注解标记的方法。

  4.获取方法method上的@RequestMapping实例

  5.检查方法所属的类有没有@RequestMapping注解

  6.将类和方法的RequestMapping结合

  7.当请求到达时，去UrlMap中找匹配的Url，以及获取对应mapping实例，然后去handerMethods中获取匹配HandlerMethod实例。

  8.将RequestMappingInfo实例以及处理方法注册到缓存中。








### Mybatis

#### 1.Mybatis 中 # 和 $ 的区别

**#{}** 是预编译处理，像传进来的数据会加个" "（#将传入的数据都当成一个字符串，会对自动传入的数据加一个双引号）

**${}** 就是字符串替换。直接替换掉占位符。$方式一般用于传入数据库对象，例如传入表名.





#### 2.防止sql注入的几种方式

(1)、jdbc使用 PreparedStatement代替Statement， PreparedStatement 不仅提高了代码的可读性和可维护性.而且也提高了安全性，有效防止sql注入；
(2)、在程序代码中使用正则表达式过滤参数。使用正则表达式过滤可能造成注入的符号，如' --等
(3)、在页面输入参数时也进行字符串检测和提交时进行参数检查，同样可以使用正则表达式，不允许特殊符号出现。





#### 3.使用 Mybatis 时，调用 DAO（Mapper）接口时是怎么调用到 SQL 的

1、DAO接口会被加载到 Spring 容器中，通过动态代理来创建

2、XML中的SQL会被解析并保存到本地缓存中，key是SQL 的 namespace + id，value 是SQL的封装

3、当我们调用DAO接口时，会走到代理类中，通过接口的全路径名，从步骤2的缓存中找到对应的SQL，然后执行并返回结果

1）扫描注册 basePackage 包下的所有 bean，将 basePackage 包下的所有 bean 进行一些特殊处理：beanClass 设置为 MapperFactoryBean、bean 的真正接口类作为构造函数参数传入 MapperFactoryBean、为 MapperFactoryBean 添加 sqlSessionFactory 和 sqlSessionTemplate属性。

2）解析 mapperLocations 属性的 mapper 文件，将 mapper 文件中的每个 SQL 封装成 MappedStatement，放到 mappedStatements 缓存中，key 为 id，例如：com.joonwhee.open.mapper.UserPOMapper.queryByPrimaryKey，value 为 MappedStatement。并且将解析过的 mapper 文件的 namespace 放到 knownMappers 缓存中，key 为 namespace 对应的 class，value 为 MapperProxyFactory。

3）创建 DAO 的 bean 时，通过 mapperInterface 从 knownMappers 缓存中获取到 MapperProxyFactory 对象，通过 JDK 动态代理创建 MapperProxyFactory 实例对象，InvocationHandler 为 MapperProxy。

4）DAO 中的接口被调用时，通过动态代理，调用 MapperProxy 的 invoke 方法，最终通过 mapperInterface 从 mappedStatements 缓存中拿到对应的 MappedStatement，执行相应的操作。

### springboot

#### 1.springboot启动流程

@SpringBootApplication注解

启动类上标注了`@SpringBootApplication`注解，然后在`main`函数中调用`SpringApplication.run(MainApplication.class, args);`这句代码就完成了SpringBoot的启动流程，非常简单。这行代码的源码中连续调用两次run方法，构建了一个`SpringApplication`对象，然后再调用其`run`方法来启动SpringBoot项目。

`@SpringBootApplication`注解是一个组合注解，主要由`@SpringBootConfiguration`,`@EnableAutoConfiguration`和`@ComponentScan`这三个注解组合而成。

1、@SpringBootConfiguration继承自@Configuration，二者功能也一致，标注当前类是配置类，并会将当前类内声明的一个或多个以@Bean注解标记的方法的实例纳入到spring容器中，并且实例名就是方法名。

说明：用于bean加载spring容器

2、@EnableAutoConfiguration的作用启动自动的配置，@EnableAutoConfiguration注解的意思就是Springboot根据你添加的jar包来配置你项目的默认配置，比如根据spring-boot-starter-web ，来判断你的项目是否需要添加了webmvc和tomcat，就会自动的帮你配置web项目中所需要的默认配置。借助@Import的帮助，将所有符合自动配置条件的bean定义加载到IoC容器

借助于Spring框架原有的一个工具类：SpringFactoriesLoader的支持，@EnableAutoConfiguration可以智能的自动配置功效才得以大功告成！

说明：用于配置添加jar包默认配置项

3、@ComponentScan，扫描当前包及其子包下被@Component，@Controller，@Service，@Repository注解标记的类并纳入到spring容器中进行管理。是以前的（以前使用在xml中使用的标签，用来扫描包配置的平行支持）。

因此`@SpringBootApplication`注解主要作为一个配置类，能够触发包扫描和自动配置的逻辑，从而使得SpringBoot的相关`bean`被注册进Spring容器。

1. 从`spring.factories`配置文件中**加载`EventPublishingRunListener`对象**，该对象拥有`SimpleApplicationEventMulticaster`属性，即在SpringBoot启动过程的不同阶段用来发射内置的生命周期事件;
2. **准备环境变量**，包括系统变量，环境变量，命令行参数，默认变量，`servlet`相关配置变量，随机值以及配置文件（比如`application.properties`）等;
3. 控制台**打印SpringBoot的`bannner`标志**；
4. **根据不同类型环境创建不同类型的`applicationcontext`容器**，因为这里是`servlet`环境，所以创建的是`AnnotationConfigServletWebServerApplicationContext`容器对象；
5. 从`spring.factories`配置文件中**加载`FailureAnalyzers`对象**,用来报告SpringBoot启动过程中的异常；
6. **为刚创建的容器对象做一些初始化工作**，准备一些容器属性值等，对`ApplicationContext`应用一些相关的后置处理和调用各个`ApplicationContextInitializer`的初始化方法来执行一些初始化逻辑等；
7. **刷新容器**，这一步至关重要。比如调用`bean factory`的后置处理器，注册`BeanPostProcessor`后置处理器，初始化事件广播器且广播事件，初始化剩下的单例`bean`和SpringBoot创建内嵌的`Tomcat`服务器等等重要且复杂的逻辑都在这里实现，主要步骤可见代码的注释，关于这里的逻辑会在以后的spring源码分析专题详细分析；
8. **执行刷新容器后的后置处理逻辑**，注意这里为空方法；
9. **调用`ApplicationRunner`和`CommandLineRunner`的run方法**，我们实现这两个接口可以在spring容器启动后需要的一些东西比如加载一些业务数据等;
10. **报告启动异常**，即若启动过程中抛出异常，此时用`FailureAnalyzers`来报告异常;
11. 最终**返回容器对象**，这里调用方法没有声明对象来接收。

当然在SpringBoot启动过程中，每个不同的启动阶段会分别发射不同的内置生命周期事件，比如在准备`environment`前会发射`ApplicationStartingEvent`事件，在`environment`准备好后会发射`ApplicationEnvironmentPreparedEvent`事件，在刷新容器前会发射`ApplicationPreparedEvent`事件等，总之SpringBoot总共内置了7个生命周期事件，除了标志SpringBoot的不同启动阶段外，同时一些监听器也会监听相应的生命周期事件从而执行一些启动初始化逻辑。



## MySQL

### 1.索引相关问题

B+树，也有哈希表

- 为什么使用 B+ 树，与其他索引相比有什么优点，一个整数字段索引大概1200叉，树高为4，就可以17亿值

高度低，磁盘读取次数少

- 各种索引之间的区别

非聚集/聚集 

- B+ 树在进行范围查找时怎么处理

顺序访问

- MySQL 索引叶子节点存放的是什么

数据

- 联合索引（复合索引）的底层实现

先按左侧的索引排序

#### 1.1索引分类

按数据结构分类可分为：**B+tree索引、Hash索引、Full-text索引**。
按物理存储分类可分为：**聚簇索引、二级索引（辅助索引）**。
按字段特性分类可分为：**主键索引、普通索引、前缀索引**。
按字段个数分类可分为：**单列索引、联合索引（复合索引、组合索引）**。

#### 1.2B+树与B树对比

- B+tree 非叶子节点只存储键值信息， 数据记录都存放在叶子节点中。而B-tree的非叶子节点也存储数据。所以B+tree单个节点的数据量更小，在相同的磁盘I/O次数下，能查询更多的节点。

- B+tree 所有叶子节点之间都采用单链表连接。适合MySQL中常见的基于范围的顺序检索场景，而B-tree无法做到这一点。

- B树大量应用在数据库和文件系统当中。

  它的设计思想是，将相关数据尽量集中在一起，以便一次读取多个数据，减少硬盘操作次数。B树算法减少定位记录时所经历的中间过程，从而加快存取速度。

  假定一个节点可以容纳100个值，那么3层的B树可以容纳100万个数据，如果换成二叉查找树，则需要20层！假定操作系统一次读取一个节点，并且根节点保留在内存中，那么B树在100万个数据中查找目标值，只需要读取两次硬盘。

  如mongoDB数据库使用，单次查询平均快于Mysql（但侧面来看Mysql至少平均查询耗时差不多）

#### 1.3B+树与hash索引比较

Hash 索引结构的特殊性，其检索效率非常高，索引的检索可以一次定位，不像B-Tree 索引需要从根节点到枝节点，最后才能访问到页节点这样多次的IO访问，所以 Hash 索引的查询效率要远高于 B-Tree 索引。虽然 Hash 索引效率高，但是 Hash 索引本身由于其特殊性也带来了很多限制和弊端，主要有以下这些。

Hash 索引仅仅能满足 `=` , `IN` 和 `<=>`(表示NULL安全的等价) 查询，不能使用范围查询。

由于 Hash 索引比较的是进行 Hash 运算之后的 Hash值，所以它只能用于等值的过滤，不能用于基于范围的过滤，因为经过相应的 Hash算法处理之后的 Hash 值的大小关系，并不能保证和Hash运算前完全一样。

Hash 索引无法适用数据的排序操作。

由于 Hash 索引中存放的是经过 Hash 计算之后的 Hash值，而且Hash值的大小关系并不一定和 Hash运算前的键值完全一样，所以数据库无法利用索引的数据来避免任何排序运算；

Hash 索引不能利用部分索引键查询。

对于组合索引，Hash 索引在计算 Hash 值的时候是组合索引键合并后再一起计算 Hash 值，而不是单独计算 Hash值，所以通过组合索引的前面一个或几个索引键进行查询的时候，Hash 索引也无法被利用。

Hash 索引依然需要回表扫描。

Hash 索引是将索引键通过 Hash 运算之后，将 Hash运算结果的 Hash值和所对应的行指针信息存放于一个 Hash 表中，由于不同索引键可能存在相同 Hash 值，所以即使取满足某个 Hash 键值的数据的记录条数，也无法从 Hash索引中直接完成查询，还是要通过访问表中的实际数据进行相应的比较，并得到相应的结果。

Hash索引遇到大量Hash值相等的情况后性能并不一定就会比B-Tree索引高。

选择性比较低的索引键，如果创建 Hash 索引，那么将会存在大量记录指针信息存于同一个Hash值相关联。这样要定位某一条记录时就会非常麻烦，会浪费多次表数据的访问，而造成整体性能低下

**由于范围查询是MySQL数据库查询中常见的场景，Hash表不适合做范围查询，它更适合做等值查询。另外Hash表还存在Hash函数选择和Hash值冲突等问题。因此，B+tree索引要比Hash表索引有更广的适用场景。**

#### 1.4创建索引原则

1.索引最左匹配原则

2.为经常需要排序、分组操作的字段建立索引

3.为常作为查询条件的字段建立索引

4.限制索引的数目

5.尽量选择区分度高的列作为索引

6.索引列不能参与计算

7.扩展索引

8.条件带like 注意事项

9.尽量使用数据量少的索引

10.尽量使用前缀来索引

11.删除不再使用或者很少使用的索引

12.=和in可以乱序。

#### 1.5最左前缀原则

- 索引可以简单如一个列(a)，也可以复杂如多个列(a, b, c, d)，即联合索引。
- 如果是联合索引，那么key也由多个列组成，同时，索引只能用于查找key是否存在（相等），遇到范围查询(>、<、between、like左匹配)等就不能进一步匹配了，后续退化为线性查找。因此，列的排列顺序决定了可命中索引的列数。

例子：

如有索引(a, b, c, d)，查询条件a = 1 and b = 2 and c > 3 and d = 4，则会在每个节点依次命中a、b、c，无法命中d。(很简单：索引命中只能是相等的情况，不能是范围匹配)

#### 1.6创建索引的方式

在执行CREATE TABLE语句时可以创建索引，也可以单独用CREATE INDEX或ALTER TABLE来为表增加索引。



#### 1.7选择普通索引or唯一索引

查询过程区别微乎其微-更新过程：当需要更新一个数据页时，如果数据页在内存中就直接更新，而如果这个数据页还没有在内存中的话，在不影响数据一致性的前提下，InnoDB会将这些更新操作缓存在change buffer中，这样就不需要从磁盘中读入这个数据页了。在下次查询需要访问这个数据页的时候，将数据页读入内存，然后执行change buffer中与这个页有关的操作。通过这种方式就能保证这个数据逻辑的正确性。

对于唯一索引来说，所有的更新操作都要先判断这个操作是否违反唯一性约束。比如，要插入(4,400)这个记录，就要先判断现在表中是否已经存在k=4的记录，而这必须要将数据页读入内存才能判断。如果都已经读入到内存了，那直接更新内存会更快，就没必要使用change buffer了。

因此，唯一索引的更新就不能使用change buffer，实际上也只有普通索引可以使用。

因为merge的时候是真正进行数据更新的时刻，而change buffer的主要目的就是将记录的变更动作缓存下来，所以在一个数据页做merge之前，change buffer记录的变更越多（也就是这个页面上要更新的次数越多），收益就越大。

因此，对于写多读少的业务来说，页面在写完以后马上被访问到的概率比较小，此时change buffer的使用效果最好。这种业务模型常见的就是账单类、日志类的系统。

**redo log 主要节省的是随机写磁盘的IO消耗（转成顺序写），而change buffer主要节省的则是随机读磁盘的IO消耗。**

```
set long_query_time=0;
```



#### 1.8选错索引

一个索引上不同的值越多，这个索引的区分度就越好。而一个索引上不同的值的个数，我们称之为“基数”

采样统计的时候，InnoDB默认会选择N个数据页，统计这些页面上的不同值，得到一个平均值，然后乘以这个索引的页面数，就得到了这个索引的基数。

**一种方法是，像我们第一个例子一样，采用force index强行选择一个索引**

**第二种方法就是，我们可以考虑修改语句，引导MySQL使用我们期望的索引**

**第三种方法是，在有些场景下，我们可以新建一个更合适的索引，来提供给优化器做选择，或删掉误用的索引。**





#### 1.9怎么给字符串加索引：

**使用前缀索引，定义好长度，就可以做到既节省空间，又不用额外增加太多的查询成本。**

使用前缀索引就用不上覆盖索引对查询性能的优化

**第一种方式是使用倒序存储**

**第二种方式是使用hash字段**



#### 1.10一棵B+树能存多少行数据

以页大小为 16k 为例，若单行数据占 1k ，则每个叶子节点能存 16 行数据。中间节点假设主键大小为 8k， 指针节点为 6k， 则每个中间节点能存 16384/14=1170 个索引。双层可存 1170*16=18720 条数据；三层可存 1170*1170*16=21902400 条数据。



### 2.mysql分库分表相关问题

#### 2.1分库分表实现

目前市面上的分库分表中间件相对较多，其中基于代理方式的有MySQL Proxy和Amoeba， 基于Hibernate框架的是Hibernate Shards，基于jdbc的有当当sharding-jdbc， 基于mybatis的类似maven插件式的有蘑菇街的蘑菇街TSharding， 通过重写spring的ibatis template类的Cobar Client。

垂直拆分

1. 垂直分表

   也就是“大表拆小表”，基于列字段进行的。一般是表中的字段较多，将不常用的， 数据较大，长度较长（比如text类型字段）的拆分到“扩展表“。 一般是针对那种几百列的大表，也避免查询时，数据量太大造成的“跨页”问题。

2. 垂直分库

   垂直分库针对的是一个系统中的不同业务进行拆分，比如用户User一个库，商品Producet一个库，订单Order一个库。 切分后，要放在多个服务器上，而不是一个服务器上。为什么？ 我们想象一下，一个购物网站对外提供服务，会有用户，商品，订单等的CRUD。没拆分之前， 全部都是落到单一的库上的，这会让数据库的单库处理能力成为瓶颈。按垂直分库后，如果还是放在一个数据库服务器上， 随着用户量增大，这会让单个数据库的处理能力成为瓶颈，还有单个服务器的磁盘空间，内存，tps等非常吃紧。                所以我们要拆分到多个服务器上，这样上面的问题都解决了，以后也不会面对单机资源问题。

   数据库业务层面的拆分，和服务的“治理”，“降级”机制类似，也能对不同业务的数据分别的进行管理，维护，监控，扩展等。 数据库往往最容易成为应用系统的瓶颈，而数据库本身属于“有状态”的，相对于Web和应用服务器来讲，是比较难实现“横向扩展”的。 数据库的连接资源比较宝贵且单机处理能力也有限，在高并发场景下，垂直分库一定程度上能够突破IO、连接数及单机硬件资源的瓶颈。

水平拆分

1. 水平分表

   针对数据量巨大的单张表（比如订单表），按照某种规则（RANGE,HASH取模等），切分到多张表里面去。 但是这些表还是在同一个库中，所以库级别的数据库操作还是有IO瓶颈。不建议采用。

2. 水平分库分表

   将单张表的数据切分到多个服务器上去，每个服务器具有相应的库与表，只是表中数据集合不同。 水平分库分表能够有效的缓解单机和单库的性能瓶颈和压力，突破IO、连接数、硬件资源等的瓶颈。

3. 水平分库分表切分规则

4. 1. RANGE

      从0到10000一个表，10001到20000一个表；

   2. HASH取模

      一个商场系统，一般都是将用户，订单作为主表，然后将和它们相关的作为附表，这样不会造成跨库事务之类的问题。 取用户id，然后hash取模，分配到不同的数据库上。

   3. 地理区域

      比如按照华东，华南，华北这样来区分业务，七牛云应该就是如此。

   4. 时间

      按照时间切分，就是将6个月前，甚至一年前的数据切出去放到另外的一张表，因为随着时间流逝，这些表的数据 被查询的概率变小，所以没必要和“热数据”放在一起，这个也是“冷热数据分离”。

#### 2.2怎么分，垂直/水平

##### 1、IO瓶颈

第一种：磁盘读IO瓶颈，热点数据太多，数据库缓存放不下，每次查询时会产生大量的IO，降低查询速度 -> 分库和垂直分表。

第二种：网络IO瓶颈，请求的数据太多，网络带宽不够 -> 分库。

##### 2、CPU瓶颈

第一种：SQL问题，如SQL中包含join，group by，order by，非索引字段条件查询等，增加CPU运算的操作 -> SQL优化，建立合适的索引，在业务Service层进行业务计算。

第二种：单表数据量太大，查询时扫描的行太多，SQL效率低，CPU率先出现瓶颈 -> 水平分表。

根据容量（当前容量和增长量）评估分库或分表个数 -> 选key（均匀）-> 分表规则（hash或range等）-> 执行（一般双写）-> 扩容问题（尽量减少数据的移动）。





#### 2.3.分库分表后面临的问题

事务支持

分库分表后，就成了分布式事务了。如果依赖数据库本身的分布式事务管理功能去执行事务，将付出高昂的性能代价； 如果由应用程序去协助控制，形成程序逻辑上的事务，又会造成编程方面的负担。

跨库join

TODO 分库分表后表之间的关联操作将受到限制，我们无法join位于不同分库的表，也无法join分表粒度不同的表， 结果原本一次查询能够完成的业务，可能需要多次查询才能完成。 粗略的解决方法： 全局表：基础数据，所有库都拷贝一份。 字段冗余：这样有些字段就不用join去查询了。 系统层组装：分别查询出所有，然后组装起来，较复杂。











### 3.主从复制问题

#### 3.1主从同步原理与过程

在实际的生产中，为了解决Mysql的单点故障已经提高MySQL的整体服务性能，一般都会采用**「主从复制」**。

比如：在复杂的业务系统中，有一句sql执行后导致锁表，并且这条sql的的执行时间有比较长，那么此sql执行的期间导致服务不可用，这样就会严重影响用户的体验度。

主从复制中分为**「主服务器（master）\**「和」\**从服务器（slave）」**，**「主服务器负责写，而从服务器负责读」**，Mysql的主从复制的过程是一个**「异步的过程」**。

Mysql的主从复制中主要有三个线程：`master（binlog dump thread）、slave（I/O thread 、SQL thread）`，Master一条线程和Slave中的两条线程。

`master（binlog dump thread）`主要负责Master库中有数据更新的时候，会按照`binlog`格式，将更新的事件类型写入到主库的`binlog`文件中。

并且，Master会创建`log dump`线程通知Slave主库中存在数据更新，这就是为什么主库的binlog日志一定要开启的原因。

`I/O thread`线程在Slave中创建，该线程用于请求Master，Master会返回binlog的名称以及当前数据更新的位置、binlog文件位置的副本。

然后，将`binlog`保存在 **「relay log（中继日志）」** 中，中继日志也是记录数据更新的信息。

SQL线程也是在Slave中创建的，当Slave检测到中继日志有更新，就会将更新的内容同步到Slave数据库中，这样就保证了主从的数据的同步。

以上就是主从复制的过程，当然，主从复制的过程有不同的策略方式进行数据的同步，主要包含以下几种：

1. **「同步策略」**：Master会等待所有的Slave都回应后才会提交，这个主从的同步的性能会严重的影响。
2. **「半同步策略」**：Master至少会等待一个Slave回应后提交。
3. **「异步策略」**：Master不用等待Slave回应就可以提交。
4. **「延迟策略」**：Slave要落后于Master指定的时间。

对于不同的业务需求，有不同的策略方案，但是一般都会采用最终一致性，不会要求强一致性，毕竟强一致性会严重影响性能。

- MySQL 的主从同步原理

在master机器上，主从同步事件会被写到特殊的log文件中(binary-log);
在slave机器上，slave读取主从同步事件，并根据读取的事件变化，在slave库上做相应的更改。

- statement：会将对数据库操作的sql语句写入到binlog中。
- row：会将每一条数据的变化写入到binlog中。
- mixed：statement与row的混合。Mysql决定什么时候写statement格式的，什么时候写row格式的binlog。
- 当slave连接到master的时候，master机器会为slave开启binlog dump线程。当master 的 binlog发生变化的时候，binlog dump线程会通知slave，并将相应的binlog内容发送给slave。

当主从同步开启的时候，slave上会创建2个线程。

- I/O线程。该线程连接到master机器，master机器上的**binlog dump线程**会将binlog的内容发送给该**I/O线程**。该**I/O线程**接收到binlog内容后，再将内容写入到本地的relay log。
- SQL线程。该线程读取I/O线程写入的relay log。并且根据relay log的内容对slave数据库做相应的操作。

- 分库分表的实现方案





### 4.MySQL锁相关

#### 4.1MySQL 如何锁住一行数据

Update delete insert 会自动加锁，select。。for update，select * from users where id =1 lock in share mode共享锁

#### 4.2SELECT 语句能加互斥锁吗

select * from users where id =1 for update

#### 4.3多个事务同时对一行数据进行 SELECT FOR UPDATE 会阻塞还是异常

阻塞

#### 4.4.MySQL 的 gap 锁

说白了gap就是索引树中插入新记录的空隙。相应的gap lock就是加在gap上的锁，还有一个next-key锁，是记录+记录前面的gap的组合的锁。

#### 4.5.行锁

行锁：**在InnoDB事务中，行锁是在需要的时候才加上的，但并不是不需要了就立刻释放，而是要等到事务结束时才释放。这个就是两阶段锁协议。**

如果你的事务中需要锁多个行，要把最可能造成锁冲突、最可能影响并发度的锁尽量往后放。

死锁检测要耗费大量的CPU资源：**一种头痛医头的方法，就是如果你能确保这个业务一定不会出现死锁，可以临时把死锁检测关掉。****另一个思路是控制并发度**



#### 4.6nextkey

**我总结的加锁规则里面，包含了两个“原则”、两个“优化”和一个“bug”。**

1. 原则1：加锁的基本单位是next-key lock。希望你还记得，next-key lock是前开后闭区间。
2. 原则2：查找过程中访问到的对象才会加锁。
3. 优化1：索引上的等值查询，给唯一索引加锁的时候，next-key lock退化为行锁。
4. 优化2：索引上的等值查询，向右遍历时且最后一个值不满足等值条件的时候，next-key lock退化为间隙锁。
5. 一个bug：唯一索引上的范围查询会访问到不满足条件的第一个值为止。

lock in share mode只锁覆盖索引，但是如果是for update就不一样了。 执行 for update时，系统会认为你接下来要更新数据，因此会顺便给主键索引上满足条件的行加上行锁。

InnoDB会往前扫描到第一个不满足条件的行为止，也就是id=20。而且由于这是个范围扫描，因此索引id上的(15,20]这个next-key lock也会被锁上。

**在删除数据的时候尽量加limit**。这样不仅可以控制删除数据的条数，让操作更安全，还可以减小加锁的范围。



### 5.MySQL 不同执行引擎的区别

不支持事务，叶子结点存数据地址，innodb支持外键，不保存行数，锁的粒度区别，





### 6.MySQL 的事务相关问题

#### 6.1隔离级别

读未提交、读已提交、可重复读以及可串行化

#### 6.2.MySQL 的可重复读是怎么实现的

每条记录在更新的时候都会同时记录一条回滚操作（回滚操作日志undo log）。同一条记录在系统中可以存在多个版本，这就是数据库的多版本并发控制（MVCC）。即通过回滚（rollback操作），可以回到前一个状态的值。

#### 6.3.MySQL 是否会出现幻读

select 某记录是否存在，不存在，准备插入此记录，但执行 insert 时发现此记录已存在，无法插入，此时就发生了幻读。加gap锁

#### 6.4事务启动方式

1. 显式启动事务语句， begin 或 start transaction。配套的提交语句是commit，回滚语句是rollback。
2. set autocommit=0，这个命令会将这个线程的自动提交关掉。意味着如果你只执行一个select语句，这个事务就启动了，而且并不会自动提交。这个事务持续存在直到你主动执行commit 或 rollback 语句，或者断开连接。

#### 6.5MVCC

MVCC每条数据都有多版本快照

InnoDB里面每个事务有一个唯一的事务ID，叫作transaction id。它是在事务开始的时候向InnoDB的事务系统申请的，是按申请顺序严格递增的。

**InnoDB利用了“所有数据都有多个版本”的这个特性，实现了“秒级创建快照”的能力。**

**更新数据都是先读后写的，而这个读，只能读当前的值，称为“当前读”除了update语句外，select语句如果加锁，也是当前读。**

可重复读的核心就是一致性读（consistent read）；而事务更新数据的时候，只能用当前读。如果当前的记录的行锁被其他事务占用的话，就需要进入锁等待。

- 在可重复读隔离级别下，只需要在事务开始的时候创建一致性视图，之后事务里的其他查询都共用这个一致性视图；
- 在读提交隔离级别下，每一个语句执行前都会重新算出一个新的视图。



#### 6.6.redolog与binlog

写入redo log 处于prepare阶段之后、写binlog之前，发生了崩溃（crash），由于此时binlog还没写，redo log也还没提交，所以崩溃恢复的时候，这个事务会回滚。这时候，binlog还没写，所以也不会传到备库。

1. 如果redo log里面的事务是完整的，也就是已经有了commit标识，则直接提交；
2. 如果redo log里面的事务只有完整的prepare，则判断对应的事务binlog是否存在并完整：
   a. 如果是，则提交事务；
   b. 否则，回滚事务。

一个事务的binlog是有完整格式的：

- statement格式的binlog，最后会有COMMIT；
- row格式的binlog，最后会有一个XID event。

对于InnoDB引擎来说，如果redo log提交完成了，事务就不能回滚（如果这还允许回滚，就可能覆盖掉别的事务的更新）。而如果redo log直接提交，然后binlog写入的时候失败，InnoDB又回滚不了，数据和binlog日志又不一致了。

实际上，redo log并没有记录数据页的完整数据，所以它并没有能力自己去更新磁盘数据页，也就不存在“数据最终落盘，是由redo log更新过去”的情况。

1. 如果是正常运行的实例的话，数据页被修改以后，跟磁盘的数据页不一致，称为脏页。最终数据落盘，就是把内存中的数据页写盘。这个过程，甚至与redo log毫无关系。
2. 在崩溃恢复场景中，InnoDB如果判断到一个数据页可能在崩溃恢复的时候丢失了更新，就会将它读到内存，然后让redo log更新内存内容。更新完成后，内存页变成脏页，就回到了第一种情况的状态。



#### 6.7binlog写入机制,redolog写入机制

1. sync_binlog=0的时候，表示每次提交事务都只write，不fsync；
2. sync_binlog=1的时候，表示每次提交事务都会执行fsync；
3. sync_binlog=N(N>1)的时候，表示每次提交事务都write，但累积N个事务后才fsync。

因此，在出现IO瓶颈的场景里，将sync_binlog设置成一个比较大的值，可以提升性能。在实际的业务场景中，考虑到丢失日志量的可控性，一般不建议将这个参数设成0，比较常见的是将其设置为100~1000中的某个数值。

但是，将sync_binlog设置为N，对应的风险是：如果主机发生异常重启，会丢失最近N个事务的binlog日志

redolog写入机制

为了控制redo log的写入策略，InnoDB提供了innodb_flush_log_at_trx_commit参数，它有三种可能取值：

1. 设置为0的时候，表示每次事务提交时都只是把redo log留在redo log buffer中;
2. 设置为1的时候，表示每次事务提交时都将redo log直接持久化到磁盘；
3. 设置为2的时候，表示每次事务提交时都只是把redo log写到page cache。

InnoDB有一个后台线程，每隔1秒，就会把redo log buffer中的日志，调用write写到文件系统的page cache，然后调用fsync持久化到磁盘。





### 7.分布式唯一 ID 方案

数据库自增ID、UUID生成、snowflake雪花算法（把64-bit分别划分成多段，分开来标示机器、时间、某一并发序列等，从而使每台机器及同一机器生成的ID都是互不相同。）





### 8.慢查询相关问题

#### 8.1explain相关问题

优化SQL

- explain 中每个字段的意思

- id: SELECT 查询的标识符. 每个 SELECT 都会自动分配一个唯一的标识符.
- select_type: SELECT 查询的类型.
- table: 查询的是哪个表
- partitions: 匹配的分区
- type: join 类型
- possible_keys: 此次查询中可能选用的索引
- key: 此次查询中确切使用到的索引.
- ref: 哪个字段或常数与 key 一起被使用
- rows: 显示此查询一共扫描了多少行. 这个是一个估计值.
- filtered: 表示此查询条件所过滤的数据的百分比
- extra: 额外的信息

type（重要）

这是**最重要的字段之一**，显示查询使用了何种类型。从最好到最差的连接类型依次为：

> **system，const，eq_ref，ref，fulltext，ref_or_null，index_merge，unique_subquery，index_subquery，range，index，ALL**

rows（重要）

**rows 也是一个重要的字段**。 这是mysql估算的需要扫描的行数（不是精确值）。
这个值非常直观显示 SQL 的效率好坏, 原则上 rows 越少越好.

extra（重要）

**Using index**
 "覆盖索引扫描", 表示查询在索引树中就可查找所需数据, 不用扫描表数据文件, 往往说明**性能不错**

**Using temporary**
 查询有使用临时表, 一般出现于排序, 分组和多表 join 的情况, 查询效率不高, 建议优化.

- explain 中的 type 字段有哪些常见的值
- explain 中你通常关注哪些字段，为什么





#### 8.2SQL语句偶尔变慢

掌柜记账的账本是数据文件，记账用的粉板是日志文件（redo log），掌柜的记忆就是内存。

**当内存数据页跟磁盘数据页内容不一致的时候，我们称这个内存页为“脏页”。内存数据写入到磁盘后，内存和磁盘上的数据页的内容就一致了，称为“干净页”**。

**InnoDB用缓冲池（buffer pool）管理内存，缓冲池中的内存页有三种状态**

InnoDB的刷盘速度就是要参考这两个因素：一个是脏页比例，一个是redo log写盘速度。



#### 8.3只查一行为什么会慢

第一类：查询长时间不返回

出现**这个状态表示的是，现在有一个线程正在表t上请求或者持有MDL写锁，把select语句堵住了。**等flush等行锁

查询慢：可能是当前读的原因

幻读问题：产生幻读的原因是，行锁只能锁住行，但是新插入记录这个动作，要更新的是记录之间的“间隙”。因此，为了解决幻读问题，InnoDB只好引入新的锁，也就是间隙锁(Gap Lock)。

**间隙锁的引入，可能会导致同样的语句锁住更大的范围，这其实是影响了并发度的**



#### 8.4慢查询

1. 索引没有设计好；

   这种场景一般就是通过紧急创建索引来解决。MySQL 5.6版本以后，创建索引都支持Online DDL了，对于那种高峰期数据库已经被这个语句打挂了的情况，最高效的做法就是直接执行alter table 语句。

   比较理想的是能够在备库先执行。假设你现在的服务是一主一备，主库A、备库B，这个方案的大致流程是这样的：

   1. 在备库B上执行 set sql_log_bin=off，也就是不写binlog，然后执行alter table 语句加上索引；
   2. 执行主备切换；
   3. 这时候主库是B，备库是A。在A上执行 set sql_log_bin=off，然后执行alter table 语句加上索引。

2. SQL语句没写好；

```
mysql> insert into query_rewrite.rewrite_rules(pattern, replacement, pattern_database) values ("select * from t where id + 1 = ?", "select * from t where id = ? - 1", "db1");

call query_rewrite.flush_rewrite_rules();
```

这里，call query_rewrite.flush_rewrite_rules()这个存储过程，是让插入的新规则生效，也就是我们说的“查询重写”。你可以用图4中的方法来确认改写规则是否生效。

1. MySQL选错了索引。

这时候，应急方案就是给这个语句加上force index。

同样地，使用查询重写功能，给原来的语句加上force index，也可以解决这个问题。







### 9.一条SQL语句如何执行的

连接器-查询缓存-分析器-分析器-优化器-执行器

redolog 先写日志在写磁盘（酒店老板的小黑板）有了redo log，InnoDB就可以保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称为**crash-safe**。物理日志，在某个数据页做了什么修改，循环写

binlog逻辑日志，记录这个语句的原始逻辑，追加写





- 















### 10.重建表的流程:

1. 建立一个临时文件，扫描表A主键的所有数据页；
2. 用数据页中表A的记录生成B+树，存储到临时文件中；
3. 生成临时文件的过程中，将所有对A的操作记录在一个日志文件（row log）中，对应的是图中state2的状态；
4. 临时文件生成后，将日志文件中的操作应用到临时文件，得到一个逻辑数据上与表A相同的数据文件，对应的就是图中state3的状态；
5. 用临时文件替换表A的数据文件。

- MyISAM引擎把一个表的总行数存在了磁盘上，因此执行count(*)的时候会直接返回这个数，效率很高；
- 而InnoDB引擎就麻烦了，它执行count(*)的时候，需要把数据一行一行地从引擎里面读出来，然后累积计数。















### 11.**如果你的MySQL现在出现了性能瓶颈，而且瓶颈在IO上，可以通过哪些方法来提升性能呢？**

1. 设置 binlog_group_commit_sync_delay 和 binlog_group_commit_sync_no_delay_count参数，减少binlog的写盘次数。这个方法是基于“额外的故意等待”来实现的，因此可能会增加语句的响应时间，但没有丢失数据的风险。
2. 将sync_binlog 设置为大于1的值（比较常见是100~1000）。这样做的风险是，主机掉电时会丢binlog日志。
3. 将innodb_flush_log_at_trx_commit设置为2。这样做的风险是，主机掉电的时候会丢数据。

主库接收到客户端的更新请求后，执行内部事务的更新逻辑，同时写binlog。

备库B跟主库A之间维持了一个长连接。主库A内部有一个线程，专门用于服务备库B的这个长连接。一个事务日志同步的完整过程是这样的：

1. 在备库B上通过change master命令，设置主库A的IP、端口、用户名、密码，以及要从哪个位置开始请求binlog，这个位置包含文件名和日志偏移量。
2. 在备库B上执行start slave命令，这时候备库会启动两个线程，就是图中的io_thread和sql_thread。其中io_thread负责与主库建立连接。
3. 主库A校验完用户名、密码后，开始按照备库B传过来的位置，从本地读取binlog，发给B。
4. 备库B拿到binlog后，写到本地文件，称为中转日志（relay log）。
5. sql_thread读取中转日志，解析出日志里的命令，并执行。

一种是statement，一种是row。可能你在其他资料上还会看到有第三种格式，叫作mixed，其实它就是前两种格式的混合。

当binlog_format使用row格式的时候，binlog里面记录了真实删除行的主键id，这样binlog传到备库去的时候，就肯定会删除id=4的行，不会有主备删除不同行的问题。

- 因为有些statement格式的binlog可能会导致主备不一致，所以要使用row格式。
- 但row格式的缺点是，很占空间。比如你用一个delete语句删掉10万行数据，用statement的话就是一个SQL语句被记录到binlog中，占用几十个字节的空间。但如果用row格式的binlog，就要把这10万条记录都写到binlog中。这样做，不仅会占用更大的空间，同时写binlog也要耗费IO资源，影响执行速度。
- 所以，MySQL就取了个折中方案，也就是有了mixed格式的binlog。mixed格式的意思是，MySQL自己会判断这条SQL语句是否可能引起主备不一致，如果有可能，就用row格式，否则就用statement格式。

循环复制的问题

1. 规定两个库的server id必须不同，如果相同，则它们之间不能设定为主备关系；
2. 一个备库接到binlog并在重放的过程中，生成与原binlog的server id相同的新的binlog；
3. 每个库在收到从自己的主库发过来的日志后，先判断server id，如果跟自己的相同，表示这个日志是自己生成的，就直接丢弃这个日志。

高可用

主备延迟最直接的表现是，备库消费中转日志（relay log）的速度，比主库生产binlog的速度要慢。

1. 一主多从。除了备库外，可以多接几个从库，让这些从库来分担读的压力。
2. 通过binlog输出到外部系统，比如Hadoop这类系统，让外部系统提供统计类查询的能力。

大事务

在满足数据可靠性的前提下，MySQL高可用系统的可用性，是依赖于主备延迟的。延迟的时间越小，在主库故障的时候，服务恢复需要的时间就越短，可用性就越高。

并行复制策略

按表分发策略/按行分发策略

Global Transaction Identifier，也就是全局事务ID



### 12.常见指令

a. show tables或show tables from database_name; -- 显示当前数据库中所有表的名称
b. show databases; -- 显示mysql中所有数据库的名称
c. show columns from table_name from database_name; 或show columns from database_name.table_name; -- 显示表中列名称
d. show grants for user_name; -- 显示一个用户的权限,显示结果类似于grant 命令
e. show index from table_name; -- 显示表的索引
f. show status; -- 显示一些系统特定资源的信息,例如,正在运行的线程数量
g. show variables; -- 显示系统变量的名称和值
h. show processlist; -- 显示系统中正在运行的所有进程,也就是当前正在执行的查询。大多数用户可以查看他们自己的进程,但是如果他们拥有process权限,就可以查 看所有人的进程,包括密码。
i. show table status; -- 显示当前使用或者指定的database中的每个表的信息。信息包括表类型和表的最新更新时间
j. show privileges; -- 显示服务器所支持的不同权限
k. show create database database_name; -- 显示create database 语句是否能够创建指定的数据库
l. show create table table_name; -- 显示create database 语句是否能够创建指定的数据库
m. show engies; -- 显示安装以后可用的存储引擎和默认引擎。
n. show innodb status; -- 显示innoDB存储引擎的状态
o. show logs; -- 显示BDB存储引擎的日志
p. show warnings; -- 显示最后一个执行的语句所产生的错误、警告和通知
q. show errors; -- 只显示最后一个执行语句所产生的错误
r. show [storage] engines; --显示安装后的可用存储引擎和默认引擎
s. show procedure status --显示数据库中所有存储的存储过程基本信息,包括所属数据库,存储过 程名称,创建时间等

t. show create procedure sp_name --显示某一个存储过程的详细信息



### 13.两张表join数据库底层如何执行

t1驱动表，t2被驱动表

1. 对驱动表t1做了全表扫描，这个过程需要扫描100行；
2. 而对于每一行R，根据a字段去表t2查找，走的是树搜索过程。由于我们构造的数据都是一一对应的，因此每次的搜索过程都只扫描一行，也是总共扫描100行；
3. 所以，整个执行流程，总扫描行数是200。

1. 使用join语句，性能比强行拆成多个单表执行SQL语句的性能要好；
2. 如果使用join语句的话，需要让小表做驱动表。

被驱动表上没有可用的索引，算法的流程是这样的：

1. 把表t1的数据读入线程内存join_buffer中，由于我们这个语句中写的是select *，因此是把整个表t1放入了内存；
2. 扫描表t2，把表t2中的每一行取出来，跟join_buffer中的数据做对比，满足join条件的，作为结果集的一部分返回。

如果你的join语句很慢，就把join_buffer_size改大。

1. 如果可以使用Index Nested-Loop Join算法，也就是说可以用上被驱动表上的索引，其实是没问题的；
2. 如果使用Block Nested-Loop Join算法，扫描行数就会过多。尤其是在大表上的join操作，这样可能要扫描被驱动表很多次，会占用大量的系统资源。所以这种join尽量不要用。



### 14.join语句怎么优化

Multi-Range Read优化(MRR)。这个优化的主要目的是尽量使用顺序读盘。

1. 根据索引a，定位到满足条件的记录，将id值放入read_rnd_buffer中;
2. 将read_rnd_buffer中的id进行递增排序；
3. 排序后的id数组，依次到主键id索引中查记录，并作为结果返回。

这里，read_rnd_buffer的大小是由read_rnd_buffer_size参数控制的。如果步骤1中，read_rnd_buffer放满了，就会先执行完步骤2和3，然后清空read_rnd_buffer。之后继续找索引a的下个记录，并继续循环。

**MRR能够提升性能的核心**在于，这条查询语句在索引a上做的是一个范围查询（也就是说，这是一个多值查询），可以得到足够多的主键id。这样通过排序以后，再去主键索引查数据，才能体现出“顺序性”的优势。

那怎么才能一次性地多传些值给表t2呢？方法就是，从表t1里一次性地多拿些行出来，一起传给表t2。

既然如此，我们就把表t1的数据取出来一部分，先放到一个临时内存。这个临时内存不是别人，就是join_buffer。

通过上一篇文章，我们知道join_buffer 在BNL算法里的作用，是暂存驱动表的数据。但是在NLJ算法里并没有用。那么，我们刚好就可以复用join_buffer到BKA算法中。

如图5所示，是上面的NLJ算法优化后的BKA算法的流程。

BNL转BKA



一些情况下，我们可以直接在被驱动表上建索引，这时就可以直接转成BKA算法了。

1. 把表t2中满足条件的数据放在临时表tmp_t中；
2. 为了让join使用BKA算法，给临时表tmp_t的字段b加上索引；
3. 让表t1和tmp_t做join操作。



### 15.MySQL大表优化

#### 1.单表优化

##### 字段

- 尽量使用`TINYINT`、`SMALLINT`、`MEDIUM_INT`作为整数类型而非`INT`，如果非负则加上`UNSIGNED`
- `VARCHAR`的长度只分配真正需要的空间
- 使用枚举或整数代替字符串类型
- 尽量使用`TIMESTAMP`而非`DATETIME`，
- 单表不要有太多字段，建议在20以内
- 避免使用NULL字段，很难查询优化且占用额外索引空间
- 用整型来存IP

##### 索引

- 索引并不是越多越好，要根据查询有针对性的创建，考虑在`WHERE`和`ORDER BY`命令上涉及的列建立索引，可根据`EXPLAIN`来查看是否用了索引还是全表扫描
- 应尽量避免在`WHERE`子句中对字段进行`NULL`值判断，否则将导致引擎放弃使用索引而进行全表扫描
- 值分布很稀少的字段不适合建索引，例如"性别"这种只有两三个值的字段
- 字符字段只建前缀索引
- 字符字段最好不要做主键
- 不用外键，由程序保证约束
- 尽量不用`UNIQUE`，由程序保证约束
- 使用多列索引时主意顺序和查询条件保持一致，同时删除不必要的单列索引

##### 查询SQL

- 可通过开启慢查询日志来找出较慢的SQL
- 不做列运算：`SELECT id WHERE age + 1 = 10`，任何对列的操作都将导致表扫描，它包括数据库教程函数、计算表达式等等，查询时要尽可能将操作移至等号右边
- sql语句尽可能简单：一条sql只能在一个cpu运算；大语句拆小语句，减少锁时间；一条大sql可以堵死整个库
- 不用`SELECT *`
- `OR`改写成`IN`：`OR`的效率是n级别，`IN`的效率是log(n)级别，in的个数建议控制在200以内
- 不用函数和触发器，在应用程序实现
- 避免`%xxx`式查询
- 少用`JOIN`
- 使用同类型进行比较，比如用`'123'`和`'123'`比，`123`和`123`比
- 尽量避免在`WHERE`子句中使用!=或<>操作符，否则将引擎放弃使用索引而进行全表扫描
- 对于连续数值，使用`BETWEEN`不用`IN`：`SELECT id FROM t WHERE num BETWEEN 1 AND 5`
- 列表数据不要拿全表，要使用`LIMIT`来分页，每页数量也不要太大

#### 2.读写分离

也是目前常用的优化，从库读主库写，一般不要采用双主或多主引入很多复杂性，尽量采用文中的其他方案来提高性能。同时目前很多拆分的解决方案同时也兼顾考虑了读写分离

#### 3.缓存

缓存可以发生在这些层次：

- MySQL内部：在系统调优参数介绍了相关设置
- 数据访问层：比如MyBatis针对SQL语句做缓存，而Hibernate可以精确到单个记录，这里缓存的对象主要是持久化对象`Persistence Object`
- 应用服务层：这里可以通过编程手段对缓存做到更精准的控制和更多的实现策略，这里缓存的对象是数据传输对象`Data Transfer Object`
- Web层：针对web页面做缓存
- 浏览器客户端：用户端的缓存

可以根据实际情况在一个层次或多个层次结合加入缓存。这里重点介绍下服务层的缓存实现，目前主要有两种方式：

- 直写式（Write Through）：在数据写入数据库后，同时更新缓存，维持数据库与缓存的一致性。这也是当前大多数应用缓存框架如Spring Cache的工作方式。这种实现非常简单，同步好，但效率一般。
- 回写式（Write Back）：当有数据要写入数据库时，只会更新缓存，然后异步批量的将缓存数据同步到数据库上。这种实现比较复杂，需要较多的应用逻辑，同时可能会产生数据库与缓存的不同步，但效率非常高。



### 16.数据库三范式

第一范式（**1NF**）是指数据库表的每一列都是不可分割的基本数据项，同一列中不能有多个值。

第二范式（**1NF**）是在第一范式的基础上建立起来的，即满足第二范式必须先满足第一范式。
第二范式要求数据库的每个实例或行必须可以被唯一的区分，即表中要有一列属性可以将实体完全区分，该属性即**主键**。

第三范式（**2NF**）要求一个数据库表中不包含已在其他表中已包含的非主关键字信息，即表中属性不依赖与其他非主属性。

















## Redis

### 1.底层数据结构

#### 1.SDS与C字符串区别

C字符串不记录字符串长度，获取长度必须遍历整个字符串，复杂度为O(N)；而SDS结构中本身就有记录字符串长度的`len`属性，所有复杂度为O(1)。Redis将获取字符串长度所需的复杂度从O(N)降到了O(1)，确保获取字符串长度的工作不会成为Redis的性能瓶颈

SDS实现了**空间预分配**和**惰性空间释放**两种优化的空间分配策略，解决了字符串拼接和截取的空间问题

二进制安全，兼容部分C字符串函数

#### 2.底层数据结构

可以使用 `type` 命令查看键的数据结构，包括：string、hash、list、set、zset，这些是 Redis 对外的数据结构。实际上每种数据结构都有底层的内部编码，Redis 根据场景选择合适的内部编码，可以使用 `object encoding`。

| 数据类型 | 内部编码                                                     |
| -------- | ------------------------------------------------------------ |
| string   | int：8B 整形。                                               |
|          | embstr：value <= 39B                                         |
|          | raw：value > 39B                                             |
| hash     | ziplist：field <= 512 且 value <= 64B                        |
|          | hashtable：field > 512 或 value > 64B，此时 ziplist 的读写效率下降，而 hashtable 的读写时间复杂度都为 O(1)。 |
| list     | ziplist：key <= 512 且 value <= 64B                          |
|          | linkedlist：key > 512 或 value > 64B                         |
|          | Redis 3.2 提供了 quicklist，是以一个 ziplist 作为节点的 linkedlist，结合了两者的优势。 |
| set      | intset：key <= 512 且 element 是整数                         |
|          | hashtable：key > 512 或 element 不是整数                     |
| zset     | ziplist：key <= 128 且 member <= 64B                         |
|          | skiplist：key > 128 或 member > 64B                          |

#### 3.跳表

Redis 的跳跃表由 `redis.h/zskiplistNode` 和 `redis.h/zskiplist` 两个结构定义， 其中 `zskiplistNode` 结构用于表示跳跃表节点， 而 `zskiplist` 结构则用于保存跳跃表节点的相关信息， 比如节点的数量， 以及指向表头节点和表尾节点的指针， 等等。

每次创建一个新跳跃表节点的时候， 程序都根据幂次定律 （[power law](https://link.juejin.cn/?target=http%3A%2F%2Fen.wikipedia.org%2Fwiki%2FPower_law)，越大的数出现的概率越小） 随机生成一个介于 `1` 和 `32` 之间的值作为 `level` 数组的大小， 这个大小就是层的“高度”。

skiplist的复杂度和红黑树一样，而且实现起来更简单。

在并发环境下skiplist有另外一个优势，红黑树在插入和删除的时候可能需要做一些rebalance的操作，这样的操作可能会涉及到整个树的其他部分，而skiplist的操作显然更加局部性一些，锁需要盯住的节点更少，因此在这样的情况下性能好一些。

Skip list(跳表）是一种可以代替平衡树的数据结构，默认是按照Key值升序的。Skip list让已排序的数据分布在多层链表中，以0-1随机数决定一个数据的向上攀升与否，通过“空间来换取时间”的一个算法，**在每个节点中增加了向前的指针**，在插入、删除、查找时可以忽略一些不可能涉及到的结点，从而提高了效率。

在Java的API中已经有了实现：分别是ConcurrentSkipListMap(在功能上对应HashTable、HashMap、TreeMap) ；ConcurrentSkipListSet(在功能上对应HashSet)

**跳跃表以有序的方式在层次化的链表中保存元素， 效率和AVL树媲美 —— 查找、删除、添加等操作都可以在O(LogN)时间下完成**， 并且比起二叉搜索树来说， 跳跃表的实现要简单直观得多。

```
跳表的构造过程是：
1、给定一个有序的链表。
2、选择连表中最大和最小的元素，然后从其他元素中按照一定算法随即选出一些元素，将这些元素组成有序链表。这个新的链表称为一层，原链表称为其下一层。
3、为刚选出的每个元素添加一个指针域，这个指针指向下一层中值同自己相等的元素。Top指针指向该层首元素
4、重复2、3步，直到不再能选择出除最大最小元素以外的元素。
```

#### 1. 查找

从最上层往下查找，平均时间复杂度O(logN)

#### 2. 插入

- 为什么要随机节点出现的次数
  - 新插入一个节点之后，就会打乱上下相邻两层链表上节点个数严格的2:1的对应关系。如果要维持这种对应关系，就必须把新插入的节点后面的所有节点（也包括新插入的节点）重新进行调整，这会让时间复杂度重新蜕化成O(n)。删除数据也有同样的问题。
  - skiplist为了避免这一问题，它不要求上下相邻两层链表之间的节点个数有严格的对应关系，而是为每个节点随机出一个层数(level)。比如，一个节点随机出的层数是3，那么就把它链入到第1层到第3层这三层链表中。







### 3.Redis 怎么保证高可用

redis的高可用架构，叫做故障转移 failover，也可以叫做主备切换。

在master node故障时，自动检测，并且将某个slave node自动切换为master node的过程，叫做主备切换。这个过程，实现了redis的主从架构下的高可用性。

一旦master故障，在很短的时间内，就会切换到另一个master上去，可能就几分钟、几秒钟redis是不可用的。

这都依赖于sentinal node，即哨兵。

edis高并发：主从架构，一主多从，一般来说，很多项目其实就足够了，单主用来写入数据，单机几万QPS，多从用来查询数据，多个从实例可以提供每秒10万的QPS。

redis高并发的同时，还需要容纳大量的数据：一主多从，每个实例都容纳了完整的数据，比如redis主就10G的内存量，其实你最多只能容纳10g的数据量。如果你的缓存要容纳的数据量很大，达到了几十g，甚至几百g，或者是几t，那你就需要redis集群，而且用redis集群之后，可以提供可能每秒几十万的读写并发。

redis高可用：如果你做主从架构部署，其实就是加上哨兵就可以了，就可以实现，任何一个实例宕机，自动会进行主备切换。





### 4.Redis 的选举流程

**Sentinel是Redis实现高可用的保证**。Sentinel系统作用就是监视Redis服务器集群，它可以不停的获得redis集群状态，当一个主节点挂了，故障转移操作会在从节点中选出一个新的主节点，这里故障转移就是由Sentinel来主导完成的。

不要把Sentinel想的太复杂，**它其实就是一个特殊工作模式的Redis服务器**而已，Redis是集群部署的，这里的Sentinel也是要集群部署的，要是非单点部署，你的Sentinel挂了，此时的Redis集群就GG了。

接着上边说，当主服务器节点挂了，Sentinel系统就会选出一个**领头的Sentinel**来完成故障转移工作。选举规则如下: - 监视这个挂了的主节点的所有Sentinel都有被选举为领头的资格

- **每进行一次选举，不论是否成功，配置纪元+1**，配置纪元就是个计数器
- 每个Sentinel在**每个配置纪元中有且仅有一次选举机会**，一旦选好了该节点认为的主节点，在这个纪元内，不可以再更改
- 每个发现服务器挂了的Sentinel都会配置纪元+1并投自己一票，接着发消息要求其他Sentinel设置自己为领头人1，**每个Sentinel都想成为领头的**
- 每个Sentinel会将最先发来请求领头的节点设为自己的领头节点并发送回复，**谁先来我选谁**
- 当源Sentinel收到回复，并且**回复中的配置纪元和自己的一致且领头Id是自己的Sentinel** Id时，表明目标Sentinel已经将自己设为领头
- 在一个配置纪元内，当某个Sentinel收到半数以上的同意回复时，它就是领头的了
- 如果在给定时间内，没有被成功选举的Sentinel，那么过段时间发起新的选举

选举领头Sentinel的过程和规则大概就如上所述，需要注意的是只有集群出现节点挂了才需要选举出领头Sentinel，平时每个Sentinel还是平等身份~





### 6.Redis 的集群模式

- 主从复制模式
- Sentinel（哨兵）模式
- Cluster 模式



### 7.Redis 删除过期键的策略

1. 定时删除

   在设置键的过期时间的同时，创建一个定时器，让定时器在键的过期时间来临时，立即执行对键的删除操作。

2. 惰性删除

   放任过期键不管，每次从键空间中获取键时，检查该键是否过期，如果过期，就删除该键，如果没有过期，就返回该键。

3. 定期删除

   每隔一段时间，程序对数据库进行一次检查，删除里面的过期键，至于要删除哪些数据库的哪些过期键，则由算法决定。



### 8.内存淘汰策略

- noeviction(默认策略)：对于写请求不再提供服务，直接返回错误（DEL请求和部分特殊请求除外）
- allkeys-lru：从所有key中使用LRU算法进行淘汰
- volatile-lru：从设置了过期时间的key中使用LRU算法进行淘汰
- allkeys-random：从所有key中随机淘汰数据
- volatile-random：从设置了过期时间的key中随机淘汰
- volatile-ttl：在设置了过期时间的key中，根据key的过期时间进行淘汰，越早过期的越优先被淘汰





### 10.Redis 中 Hash 对象的扩容流程

渐进式rehash

### 11.Redis 的 Hash 对象的扩容流程在数据量大的时候会有什么问题吗

1、扩容期开始时，会先给 ht[1] 申请空间，所以在整个扩容期间，会同时存在 ht[0] 和 ht[1]，会占用额外的空间。

2、扩容期间同时存在 ht[0] 和 ht[1]，查找、删除、更新等操作有概率需要操作两张表，耗时会增加。

3、redis 在内存使用接近 maxmemory 并且有设置驱逐策略的情况下，出现 rehash 会使得内存占用超过 maxmemory，触发驱逐淘汰操作，导致 master/slave 均有有大量的 key 被驱逐淘汰，从而出现 saster/slave 主从不一致。



### 12.Redis 的持久化机制有哪几种

AOF/RDB

#### 13.RDB 和 AOF 的实现原理、优缺点

rdb持久化就是周期性的将redis中的数据进行持久化(保存到磁盘上), 周期可能是几分钟, 几小时, 或者几天

aof持久化就是实时的将redis中的数据进行持久化, 通常是redis将每一条数据保存到linux系统的os cache中(系统缓存), 每隔一秒调用一次操作系统的数据同步操作, 将数据刷到磁盘里面, 因此aof虽然具有更好的实时性, 实际上在使用时也会有一定的数据丢失, 只是相对于rdb来说丢失的数据量会更小; 另外, aof还有一个rewrite功能, 用来重写aof文件. 我们都知道, 系统的内存是一定的, 不会无限的增长, 但aof保存的是操作数据的指令, 因此会不断变大, 当aof文件达到一定程度之后, redis会先使用LRU数据清除算法, 删除不常用的数据, 然后aof会触发rewrite操作, redis会根据当前保存的数据重新写一个容量更小的aof文件;

当redis在挂掉又重新启动时, 如果只使用了rdb和aof其中的一种, 那么redis会根据使用的持久化方法恢复数据, 如果两种持久化都使用了, 那么redis会优先使用aof, 因为aof中保存的数据更全;

- rdb优缺点

  - 优点
    - rdb生成的数据文件可以用来做数据的冷备(因为每一个rdb文件都是redis某一时刻的完整数据, aof也可以用来做冷备, 但aof文件只有一个, 容量会比较大, 而且rdb数据恢复比aof更快);
    - rdb对redis的读写服务性能影响较小, 因为redis可以启动一个fork子进程来进行数据的持久化;
    - rdb数据恢复更快, 因为rdb是一个数据文件, 恢复时直接放到内存里即可, 而aof则是一个指令的集合, 恢复数据时需要逐条执行指令, 效率比较慢;
  - 缺点
    - rdb因为保存数据的时间间隔比较大, 因此会丢失更多的数据;
    - 如果数据量过大, 由于需要进行数据保存, 可能服务会暂停较长时间(因此, 一般不要将rdb的保存时间间隔定的过长);

- aof优缺点

  - 优点
    - 保存数据时间间隔为1秒, 数据丢失少;
    - aof日志文件以append-only模式写入, 所以没有任何寻址开销, 写入性能很快, 即使文件尾部被破坏也可以很容易修复;
    - aof有rewrite功能, 可以将容量大的aof文件进行压缩;
  - 缺点
    - aof文件过大, 恢复的话时间长, 效率低;
    - aof对每条指令进行日志存储, 对redis的性能影响较大, 会降低QPS;

  RDB：快照形式，触发条件手动save/bgsave/多少秒有多少个key被修改

  AOF：Redis 每执行一个修改数据的命令，都会把它添加到 AOF 文件中，当 Redis 重启时，将会读取 AOF 文件进行“重放”以恢复到 Redis 关闭前的最后时刻。



#### 14.AOF 重写的过程

从主进程中fork出子进程，并拿到fork时的AOF文件数据写到一个临时AOF文件中

在重写过程中，redis收到的命令会同时写到AOF缓冲区和重写缓冲区中，这样保证重写不丢失重写过程中的命令

重写完成后通知主进程，主进程会将AOF缓冲区中的数据追加到子进程生成的文件中

redis会原子的将旧文件替换为新文件，并开始将数据写入到新的aof文件上



### 15.哨兵模式的原理

　　哨兵集群中的每个节点都会启动三个定时任务

- 第一个定时任务： 每个sentinel节点每隔1s向所有的master、slaver、别的sentinel节点发送一个PING命令，作用：心跳检测
- 第二个定时任务： 每个sentinel每隔2s都会向master的__sentinel__:hello这个channel中发送自己掌握的集群信息和自己的一些信息（比如host,ip,run id），这个是利用redis的pub/sub功能，每个sentinel节点都会订阅这个channel，也就是说，每个sentinel节点都可以知道别的sentinel节点掌握的集群信息，作用：信息交换，了解别的sentinel的信息和他们对于主节点的判断
- 第三个定时任务： 每个sentinel节点每隔10s都会向master和slaver发送INFO命令，作用：发现最新的集群拓扑结构

主观下线

　　这个就是上面介绍的第一个定时任务做的事情，当sentinel节点向master发送一个PING命令，如果超过own-after-milliseconds（默认是30s，这个在sentinel的配置文件中可以自己配置）时间都没有收到有效回复，不好意思，我就认为你挂了，就是说为的主观下线（SDOWN），修改其flags状态为SRI_S_DOWN

客观下线

　　要了解什么是客观下线要先了解几个重要参数：

- quorum：如果要认为master客观下线，最少需要主观下线的sentinel节点个数，举例：如果5个sentinel节点，quorum = 2,那只要2个sentinel主观下线，就可以判断master客观下线
- majority：如果确定了master客观下线了，就要把其中一个slaver切换成master，做这个事情的并不是整个sentinel集群，而是sentinel集群会选出来一个sentinel节点来做，那怎么选出来的呢，下面会讲，但是有一个原则就是需要大多数节点都同意这个sentinel来做故障转移才可以，这个大多数节点就是这个参数。注意：如果sentinel节点个数5，quorum=2，majority=3，那就是3个节点同意就可以，如果quorum=5，majority=3，这时候majority=3就不管用了，需要5个节点都同意才可以。
- configuration epoch：这个其实就是version，类似于中国每个皇帝都要有一个年号一样，每个新的master都要生成一个自己的configuration epoch，就是一个编号

客观下线处理过程

1. 每个主观下线的sentinel节点都会向其他sentinel节点发送 SENTINEL is-master-down-by-addr ip port current_epoch runid，（ip：主观下线的服务id，port：主观下线的服务端口，current_epoch：sentinel的纪元，runid：*表示检测服务下线状态，如果是sentinel 运行id，表示用来选举领头sentinel（下面会讲选举领头sentinel））来询问其它sentinel是否同意服务下线。
2. 每个sentinel收到命令之后，会根据发送过来的ip和端口检查自己判断的结果，如果自己也认为下线了，就会回复，回复包含三个参数：down_state（1表示已下线，0表示未下线），leader_runid（领头sentinal id），leader_epoch（领头sentinel纪元）。由于上面发送的runid参数是*，这里后两个参数先忽略。
3. sentinel收到回复之后，根据quorum的值，判断达到这个值，如果大于或等于，就认为这个master客观下线

选择领头sentinel的过程

　　到现在为止，已经知道了master客观下线，那就需要一个sentinel来负责故障转移，那到底是哪个sentinel节点来做这件事呢？需要通过选举实现，具体的选举过程如下：

1. 判断客观下线的sentinel节点向其他节点发送SENTINEL is-master-down-by-addr ip port current_epoch runid（注意：这时的runid是自己的run id，每个sentinel节点都有一个自己运行时id）
2. 目标sentinel回复，由于这个选择领头sentinel的过程符合先到先得的原则，举例：sentinel1判断了客观下线，向sentinel2发送了第一步中的命令，sentinel2回复了sentinel1，说选你为领头，这时候sentinel3也向sentinel2发送第一步的命令，sentinel2会直接拒绝回复
3. 当sentinel发现选自己的节点个数超过majority（注意上面写的一种特殊情况quorum>majority）的个数的时候，自己就是领头节点
4. 如果没有一个sentinel达到了majority的数量，等一段时间，重新选举

选主过程

1. 选择优先级最高的节点，通过sentinel配置文件中的replica-priority配置项，这个参数越小，表示优先级越高
2. 如果第一步中的优先级相同，选择offset最大的，offset表示主节点向从节点同步数据的偏移量，越大表示同步的数据越多
3. 如果第二步offset也相同，选择run id较小的

后续事项

　　新的主节点已经选择出来了，并不是到这里就完事了，后续还需要做一些事情，如下

1. 领头sentinel向别的slaver发送slaveof命令，告诉他们新的master是谁谁谁，你们向这个master复制数据
2. 如果之前的master重新上线时，领头sentinel同样会给起发送slaveof命令，将其变成从节点



### 15.使用缓存时，先操作数据库还是先操作缓存

1）线程A发起一个写操作，第一步write DB

2）线程A第二步del cache

3）线程B发起一个读操作，cache miss

4）线程B从DB获取最新数据

5）线程B同时set cache

这种方案**没有明显的并发问题**，但是有可能**步骤二删除缓存失败**，虽然概率比较小，**优于方案一和方案二**，平时工作中也是使用方案三。

通过数据库的**binlog**来**异步淘汰key**，以mysql为例 可以**使用阿里的canal将binlog日志采集发送到MQ队列**里面，然后**通过ACK机制 确认处理** 这条更新消息，删除缓存，保证数据缓存一致性。

一、 延时双删策略

在写库前后都进行redis.del(key)操作，并且设定合理的超时时间。具体步骤是：

1）先删除缓存

2）再写数据库

3）休眠500毫秒（根据具体的业务时间来定）

4）再次删除缓存。

**那么，这个500毫秒怎么确定的，具体该休眠多久呢？**

需要评估自己的项目的读数据业务逻辑的耗时。这么做的目的，就是确保读请求结束，写请求可以删除读请求造成的缓存脏数据。

当然，这种策略还要考虑 redis 和数据库主从同步的耗时。最后的写数据的休眠时间：则在读数据业务逻辑的耗时的基础上，加上几百ms即可。比如：休眠1秒。

二、设置缓存的过期时间

从理论上来说，给缓存设置过期时间，是保证最终一致性的解决方案。所有的写操作以数据库为准，只要到达缓存过期时间，则后面的读请求自然会从数据库中读取新值然后回填缓存

结合双删策略+缓存超时设置，这样最差的情况就是在超时时间内数据存在不一致，而且又增加了写请求的耗时。

三、如何写完数据库后，再次删除缓存成功？

上述的方案有一个缺点，那就是操作完数据库后，由于种种原因删除缓存失败，这时，可能就会出现数据不一致的情况。这里，我们需要提供一个保障重试的方案。

1、方案一具体流程

（1）更新数据库数据；

（2）缓存因为种种问题删除失败；

（3）将需要删除的key发送至消息队列；

（4）自己消费消息，获得需要删除的key；

（5）继续重试删除操作，直到成功。

然而，该方案有一个缺点，对业务线代码造成大量的侵入。于是有了方案二，在方案二中，启动一个订阅程序去订阅数据库的binlog，获得需要操作的数据。在应用程序中，另起一段程序，获得这个订阅程序传来的信息，进行删除缓存操作。

2、方案二具体流程

（1）更新数据库数据；

（2）数据库会将操作信息写入binlog日志当中；

（3）订阅程序提取出所需要的数据以及key；

（4）另起一段非业务代码，获得该信息；

（5）尝试删除缓存操作，发现删除失败；

（6）将这些信息发送至消息队列；

（7）重新从消息队列中获得该数据，重试操作。



### 16.为什么是让缓存失效，而不是更新缓存

其实删除缓存，而不是更新缓存，就是一个 lazy 计算的思想，不要每次都重新做复杂的计算，不管它会不会用到，而是让它到需要被使用的时候再重新计算。像 mybatis，hibernate，都有懒加载思想。





### 17.缓存击穿，缓存穿透，缓存雪崩

一、缓存雪崩：
1、什么是缓存雪崩：

如果缓在某一个时刻出现大规模的key失效，那么就会导致大量的请求打在了数据库上面，导致数据库压力巨大，如果在高并发的情况下，可能瞬间就会导致数据库宕机。这时候如果运维马上又重启数据库，马上又会有新的流量把数据库打死。这就是缓存雪崩。

2、问题分析：

造成缓存雪崩的关键在于同一时间的大规模的key失效，为什么会出现这个问题，主要有两种可能：第一种是Redis宕机，第二种可能就是采用了相同的过期时间。搞清楚原因之后，那么有什么解决方案呢？

3、解决方案：

（1）事前：

① 均匀过期：设置不同的过期时间，让缓存失效的时间尽量均匀，避免相同的过期时间导致缓存雪崩，造成大量数据库的访问。

② 分级缓存：第一级缓存失效的基础上，访问二级缓存，每一级缓存的失效时间都不同。

③ 热点数据缓存永远不过期。

永不过期实际包含两层意思：

物理不过期，针对热点key不设置过期时间

逻辑过期，把过期时间存在key对应的value里，如果发现要过期了，通过一个后台的异步线程进行缓存的构建

④ 保证Redis缓存的高可用，防止Redis宕机导致缓存雪崩的问题。可以使用 主从+ 哨兵，Redis集群来避免 Redis 全盘崩溃的情况。

（2）事中：

① 互斥锁：在缓存失效后，通过互斥锁或者队列来控制读数据写缓存的线程数量，比如某个key只允许一个线程查询数据和写缓存，其他线程等待。这种方式会阻塞其他的线程，此时系统的吞吐量会下降

② 使用熔断机制，限流降级。当流量达到一定的阈值，直接返回“系统拥挤”之类的提示，防止过多的请求打在数据库上将数据库击垮，至少能保证一部分用户是可以正常使用，其他用户多刷新几次也能得到结果。

（3）事后：

① 开启Redis持久化机制，尽快恢复缓存数据，一旦重启，就能从磁盘上自动加载数据恢复内存中的数据。

1、什么是缓存击穿：

缓存击穿跟缓存雪崩有点类似，缓存雪崩是大规模的key失效，而缓存击穿是某个热点的key失效，大并发集中对其进行请求，就会造成大量请求读缓存没读到数据，从而导致高并发访问数据库，引起数据库压力剧增。这种现象就叫做缓存击穿。

2、问题分析：

关键在于某个热点的key失效了，导致大并发集中打在数据库上。所以要从两个方面解决，第一是否可以考虑热点key不设置过期时间，第二是否可以考虑降低打在数据库上的请求数量。

3、解决方案：

（1）在缓存失效后，通过互斥锁或者队列来控制读数据写缓存的线程数量，比如某个key只允许一个线程查询数据和写缓存，其他线程等待。这种方式会阻塞其他的线程，此时系统的吞吐量会下降

（2）热点数据缓存永远不过期。

永不过期实际包含两层意思：

物理不过期，针对热点key不设置过期时间

逻辑过期，把过期时间存在key对应的value里，如果发现要过期了，通过一个后台的异步线程进行缓存的构建

 缓存穿透：

1、什么是缓存穿透：

缓存穿透是指用户请求的数据在缓存中不存在即没有命中，同时在数据库中也不存在，导致用户每次请求该数据都要去数据库中查询一遍。如果有恶意攻击者不断请求系统中不存在的数据，会导致短时间大量请求落在数据库上，造成数据库压力过大，甚至导致数据库承受不住而宕机崩溃。

2、问题分析：

缓存穿透的关键在于在Redis中查不到key值，它和缓存击穿的根本区别在于传进来的key在Redis中是不存在的。假如有黑客传进大量的不存在的key，那么大量的请求打在数据库上是很致命的问题，所以在日常开发中要对参数做好校验，一些非法的参数，不可能存在的key就直接返回错误提示。
3、解决方法：

（1）将无效的key存放进Redis中：

当出现Redis查不到数据，数据库也查不到数据的情况，我们就把这个key保存到Redis中，设置value="null"，并设置其过期时间极短，后面再出现查询这个key的请求的时候，直接返回null，就不需要再查询数据库了。但这种处理方式是有问题的，假如传进来的这个不存在的Key值每次都是随机的，那存进Redis也没有意义。

（2）使用布隆过滤器：

如果布隆过滤器判定某个 key 不存在布隆过滤器中，那么就一定不存在，如果判定某个 key 存在，那么很大可能是存在(存在一定的误判率)。于是我们可以在缓存之前再加一个布隆过滤器，将数据库中的所有key都存储在布隆过滤器中，在查询Redis前先去布隆过滤器查询 key 是否存在，如果不存在就直接返回，不让其访问数据库，从而避免了对底层存储系统的查询压力。



### 18.redis单线程多线程问题以及线程模型

Redis基于Reactor模式开发了网络事件处理器，这个处理器被称为文件事件处理器。它的组成结构为4部分：多个套接字、IO多路复用程序、文件事件分派器、事件处理器。因为文件事件分派器队列的消费是单线程的，所以Redis才叫单线程模型。

Redis 4.0 开始就有多线程的概念了，比如 Redis 通过多线程方式在后台删除对象、以及通过 Redis 模块实现的阻塞命令等。

Redis 的瓶颈并不在 CPU，而在内存和网络。内存不够的话，可以加内存或者做数据结构优化和其他优化等，但网络的性能优化才是大头，网络 IO 的读写在 Redis 整个执行期间占用了大部分的 CPU 时间，如果把网络处理这部分做成多线程处理方式，那对整个 Redis 的性能会有很大的提升。

<img src="https://img-blog.csdnimg.cn/20210204224921839.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMjYyMzcy,size_16,color_FFFFFF,t_70" alt="img" style="zoom: 67%;" />

多个 socket 可能会并发产生不同的操作，每个操作对应不同的文件事件，但是 IO 多路复用程序会监听多个 socket，会将 socket 产生的事件都放入队列中，事件分派器每次从队列中取出一个事件，把该事件交给对应的事件处理器进行处理。







### 20.Redis主从复制

**全量同步**
Redis全量复制一般发生在Slave初始化阶段，这时Slave需要将Master上的所有数据都复制一份。具体步骤如下： 
\- 从服务器连接主服务器，发送SYNC命令； 
\- 主服务器接收到SYNC命名后，开始执行BGSAVE命令生成RDB文件并使用缓冲区记录此后执行的所有写命令； 
\- 主服务器BGSAVE执行完后，向所有从服务器发送快照文件，并在发送期间继续记录被执行的写命令； 
\- 从服务器收到快照文件后丢弃所有旧数据，载入收到的快照； 
\- 主服务器快照发送完毕后开始向从服务器发送缓冲区中的写命令； 
\- 从服务器完成对快照的载入，开始接收命令请求，并执行来自主服务器缓冲区的写命令；

**增量同步**
Redis增量复制是指Slave初始化后开始正常工作时主服务器发生的写操作同步到从服务器的过程。 
增量复制的过程主要是主服务器每执行一个写命令就会向从服务器发送相同的写命令，从服务器接收并执行收到的写命令。

**Redis主从同步策略**
主从刚刚连接的时候，进行全量同步；全同步结束后，进行增量同步。当然，如果有需要，slave 在任何时候都可以发起全量同步。redis 策略是，无论如何，首先会尝试进行增量同步，如不成功，要求从机进行全量同步。



## 计算机网络

### 1.http/https

#### 1.1http长连接和短连接

长连接：长连接，指在一个连接上可以连续发送多个数据包，在连接保持期间，如果没有数据包发送，需要双方发链路检测包。

短连接：短连接（short connnection）是相对于长连接而言的概念，指的是在数据传送过程中，只在需要发送数据时，才去建立一个连接，数据发送完成后，则断开此连接，即每次连接只完成一项业务的发送。

长连接：长连接多用于操作频繁，点对点的通讯，而且连接数不能太多情况。每个TCP连接都需要三步握手，这需要时间，如果每个操作都是先连接，再操作的话那么处理速度会降低很多，所以每个操作完后都不断开，次处理时直接发送数据包就OK了，不用建立TCP连接。

例如：数据库的连接用长连接， 如果用短连接频繁的通信会造成socket错误，而且频繁的socket 创建也是对资源的浪费。

短连接：而像WEB网站的http服务一般都用短链接，因为长连接对于服务端来说会耗费一定的资源，而像WEB网站这么频繁的成千上万甚至上亿客户端的连接用短连接会更省一些资源，如果用长连接，而且同时有成千上万的用户，如果每个用户都占用一个连接的话，那可想而知吧。

所以并发量大，但每个用户无需频繁操作情况下需用短连好。

#### 1.2HTTP1.0/1.1/2.0

HTTP/1.1相较于HTTP/1.0协议的区别主要体现在：
缓存处理
带宽优化及网络连接的使用
错误通知的管理
消息在网络中的发送
互联网地址的维护
安全性及完整性

帧、消息、流和TCP连接

有别于HTTP/1.1在连接中的明文请求，HTTP/2与SPDY一样，将一个TCP连接分为若干个流（Stream），每个流中可以传输若干消息（Message），每个消息由若干最小的二进制帧（Frame）组成。[12]这也是HTTP/1.1与HTTP/2最大的区别所在。 HTTP/2中，每个用户的操作行为被分配了一个流编号(stream ID)，这意味着用户与服务端之间创建了一个TCP通道；协议将每个请求分割为二进制的控制帧与数据帧部分，以便解析。这个举措在SPDY中的实践表明，相比HTTP/1.1，新页面加载可以加快11.81% 到 47.7%[17]

HPACK 算法

HPACK算法是新引入HTTP/2的一个算法，用于对HTTP头部做压缩。其原理在于：

客户端与服务端根据 RFC 7541 的附录A，维护一份共同的静态字典（Static Table），其中包含了常见头部名及常见头部名称与值的组合的代码；
客户端和服务端根据先入先出的原则，维护一份可动态添加内容的共同动态字典（Dynamic Table）；
客户端和服务端根据 RFC 7541 的附录B，支持基于该静态哈夫曼码表的哈夫曼编码（Huffman Coding）。

服务器推送

网站为了使请求数减少，通常采用对页面上的图片、脚本进行极简化处理。但是，这一举措十分不方便，也不高效，依然需要诸多HTTP链接来加载页面和页面资源。

HTTP/2引入了服务器推送，即服务端向客户端发送比客户端请求更多的数据。这允许服务器直接提供浏览器渲染页面所需资源，而无须浏览器在收到、解析页面后再提起一轮请求，节约了加载时间。



#### 1.3http**常用的请求方式**

GET     请求获取Request-URI所标识的资源

POST    在Request-URI所标识的资源后附加新的数据

HEAD    请求获取由Request-URI所标识的资源的响应消息报头

PUT     请求服务器存储一个资源，并用Request-URI作为其标识

DELETE  请求服务器删除Request-URI所标识的资源

TRACE   请求服务器回送收到的请求信息，主要用于测试或诊断

CONNECT 保留将来使用

OPTIONS 请求查询服务器的性能，或者查询与资源相关的选项和需求

GET 用于获取资源，而 POST 用于传输实体主体。GET 和 POST 的请求都能使用额外的参数，但是 GET 的参数是以查询字符串出现在 URL 中，而 POST 的参数存储在实体主体中。不能因为 POST 参数存储在实体主体中就认为它的安全性更高，因为照样可以通过一些抓包工具（Fiddler）查看因为 URL 只支持 ASCII 码，因此 GET 的参数中如果存在中文等字符就需要先进行编码。例如中文会转换为%E4%B8%AD%E6%96%87 ,而空格会转换为%20。POST参数支持标准字符集。GET请求在URL中传送的参数是有长度限制的，而POST么有HEAD获取报文首部，和 GET 方法类似，但是不返回报文实体主体部分







#### 1.4.HTTP请求体/响应体

HTTP的请求报文包括：请求行(request line)、请求头部(header)、空行 和 请求数据(request data) 四个部分组成

请求行包括： 请求方法，URL(包括参数信息)，协议版本这些信息（GET /admin_ui/rdx/core/images/close.png HTTP/1.1）

请求头部(Header)是一个个的key-value值，比如Accept-Encoding: gzip, deflateUser-Agent: Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 6.1; WOW64; Trident/7.0; SLCC2; .NET CLR 2.0.50727; .NET CLR 3.5.30729; .NET CLR 3.0.30729; .NET4.0C; .NET4.0E)

空行(CR+LF)：请求报文用空行表示header和请求数据的分隔请求数据：GET方法没有携带数据， POST方法会携带一个body

HTTP的响应报文包括：状态行，响应头，空行，数据(响应体)

状态行包括：HTTP版本号，状态码和状态值组成。

响应头类似请求头，是一系列key-value值Cache-Control: privateContent-Encoding: gzipServer: BWS/1.1Set-Cookie: delPer=0; path=/; domain=.baidu.com

空白行：同上，响应报文也用空白行来分隔header和数据响应体：响应的data，本例中是一段HTML



#### 1.5.HTTP状态码

| 1**  | 信息，服务器收到请求，需要请求者继续执行操作   |
| ---- | ---------------------------------------------- |
| 2**  | 成功，操作被成功接收并处理                     |
| 3**  | 重定向，需要进一步的操作以完成请求             |
| 4**  | 客户端错误，请求包含语法错误或无法完成请求     |
| 5**  | 服务器错误，服务器在处理请求的过程中发生了错误 |

| 状态码 | 状态码英文名称                  | 中文描述                                                     |
| :----- | :------------------------------ | :----------------------------------------------------------- |
| 100    | Continue                        | 继续。[客户端](http://www.dreamdu.com/webbuild/client_vs_server/)应继续其请求 |
| 101    | Switching Protocols             | 切换协议。服务器根据客户端的请求切换协议。只能切换到更高级的协议，例如，切换到HTTP的新版本协议 |
|        |                                 |                                                              |
| 200    | OK                              | 请求成功。一般用于GET与POST请求                              |
| 201    | Created                         | 已创建。成功请求并创建了新的资源                             |
| 202    | Accepted                        | 已接受。已经接受请求，但未处理完成                           |
| 203    | Non-Authoritative Information   | 非授权信息。请求成功。但返回的meta信息不在原始的服务器，而是一个副本 |
| 204    | No Content                      | 无内容。服务器成功处理，但未返回内容。在未更新网页的情况下，可确保浏览器继续显示当前文档 |
| 205    | Reset Content                   | 重置内容。服务器处理成功，用户终端（例如：浏览器）应重置文档视图。可通过此返回码清除浏览器的表单域 |
| 206    | Partial Content                 | 部分内容。服务器成功处理了部分GET请求                        |
|        |                                 |                                                              |
| 300    | Multiple Choices                | 多种选择。请求的资源可包括多个位置，相应可返回一个资源特征与地址的列表用于用户终端（例如：浏览器）选择 |
| 301    | Moved Permanently               | 永久移动。请求的资源已被永久的移动到新URI，返回信息会包括新的URI，浏览器会自动定向到新URI。今后任何新的请求都应使用新的URI代替 |
| 302    | Found                           | 临时移动。与301类似。但资源只是临时被移动。客户端应继续使用原有URI |
| 303    | See Other                       | 查看其它地址。与301类似。使用GET和POST请求查看               |
| 304    | Not Modified                    | 未修改。所请求的资源未修改，服务器返回此状态码时，不会返回任何资源。客户端通常会缓存访问过的资源，通过提供一个头信息指出客户端希望只返回在指定日期之后修改的资源 |
| 305    | Use Proxy                       | 使用代理。所请求的资源必须通过代理访问                       |
| 306    | Unused                          | 已经被废弃的HTTP状态码                                       |
| 307    | Temporary Redirect              | 临时重定向。与302类似。使用GET请求重定向                     |
|        |                                 |                                                              |
| 400    | Bad Request                     | 客户端请求的语法错误，服务器无法理解                         |
| 401    | Unauthorized                    | 请求要求用户的身份认证                                       |
| 402    | Payment Required                | 保留，将来使用                                               |
| 403    | Forbidden                       | 服务器理解请求客户端的请求，但是拒绝执行此请求               |
| 404    | Not Found                       | 服务器无法根据客户端的请求找到资源（网页）。通过此代码，网站设计人员可设置"您所请求的资源无法找到"的个性页面 |
| 405    | Method Not Allowed              | 客户端请求中的方法被禁止                                     |
| 406    | Not Acceptable                  | 服务器无法根据客户端请求的内容特性完成请求                   |
| 407    | Proxy Authentication Required   | 请求要求代理的身份认证，与401类似，但请求者应当使用代理进行授权 |
| 408    | Request Time-out                | 服务器等待客户端发送的请求时间过长，超时                     |
| 409    | Conflict                        | 服务器完成客户端的 PUT 请求时可能返回此代码，服务器处理请求时发生了冲突 |
| 410    | Gone                            | 客户端请求的资源已经不存在。410不同于404，如果资源以前有现在被永久删除了可使用410代码，网站设计人员可通过301代码指定资源的新位置 |
| 411    | Length Required                 | 服务器无法处理客户端发送的不带Content-Length的请求信息       |
| 412    | Precondition Failed             | 客户端请求信息的先决条件错误                                 |
| 413    | Request Entity Too Large        | 由于请求的实体过大，服务器无法处理，因此拒绝请求。为防止客户端的连续请求，服务器可能会关闭连接。如果只是服务器暂时无法处理，则会包含一个Retry-After的响应信息 |
| 414    | Request-URI Too Large           | 请求的URI过长（URI通常为网址），服务器无法处理               |
| 415    | Unsupported Media Type          | 服务器无法处理请求附带的媒体格式                             |
| 416    | Requested range not satisfiable | 客户端请求的范围无效                                         |
| 417    | Expectation Failed              | 服务器无法满足Expect的请求头信息                             |
|        |                                 |                                                              |
| 500    | Internal Server Error           | 服务器内部错误，无法完成请求                                 |
| 501    | Not Implemented                 | 服务器不支持请求的功能，无法完成请求                         |
| 502    | Bad Gateway                     | 作为网关或者代理工作的服务器尝试执行请求时，从远程服务器接收到了一个无效的响应 |
| 503    | Service Unavailable             | 由于超载或系统维护，服务器暂时的无法处理客户端的请求。延时的长度可包含在服务器的Retry-After头信息中 |
| 504    | Gateway Time-out                | 充当网关或代理的服务器，未及时从远端服务器获取请求           |
| 505    | HTTP Version not supported      | 服务器不支持请求的HTTP协议的版本，无法完成处理               |



#### 1.6.http与HTTPS

> 超文本协议http协议被用于在web浏览器和网站服务器之间传递信息，HTTP协议以明文的方式发送内容，不提供任何方式的数据加密，如果攻击者截取浏览器和服务器之间传递的报文，就可以直接读懂其中的信息，因此，http不适合传输一些敏感信息。比如：信用卡号，密码支付等信息。

> 为了解决HTTP协议这个缺陷，需要使用另外一个协议：安全套接字层超文本传输协议HTTPS，为了数据传输的安全，https在http的基础上加入了SSL协议，SSL依靠证书来验证服务器的身份，并为浏览器和服务器之间的通信加密。
>
> ![img](https://static001.geekbang.org/resource/image/10/7e/10315ffa19492462cadfbdfb3113987e.jpg)



#### 1.7.访问一个网站的具体过程

(1) 浏览器通过域名判断使用哪种协议访问服务器

(2) 浏览器向DNS请求解析域名对应的IP地址（浏览器缓存>本地host>本地DNS服务器>DNS服务器缓存>根DNS服务器或转发模式下的上一级DNS服务器）

(3) 域名系统DNS解析出IP地址

(4) 浏览器与该服务器建立TCP连接

(5) 浏览器发出HTTP请求，请求百度首页

(6) 服务器通过HTTP响应把首页文件发送给浏览器

(7) 浏览器将首页文件进行解析，并将Web页显示给用户

涉及协议：HTTP/HTTPS，DNS,UDP,TCP,IP,ARP(将本机默认网关 ip 地址映射成物理 MAC 地址,这样数据包便可以在以太网上传输)

从大致上来讲经历了客户端获取URL - > DNS解析 - > TCP连接 - >发送HTTP请求 - >服务器处理请求 - >返回报文 - >浏览器解析渲染页面 - > TCP断开连接详细文字讲解！

客户端：（应用层开始）获取URL，通过负责域名解析的DNS服务获取网址的IP地址，根据HTT协议生成HTTP请求报文（应用层结束）（传输层开始）根据TCP协议连接从客户端到服务端（通过三次握手）客户端给服务端发送一个带SYN（同步）标志的数据包给客户端，然后客户端接收到信息再给客户端回传一个带有SYN / ACK（确认）标志的数据包以示传达确认信息，客户求最后端的再传送一个带ACK标志的数据包，代表“握手”结束，连接成功.TCP协议在把请求报文按序号分割成多个报文段（传输层结束）

（网络层开始）根据IP协议（传输数据），ARP协议（获取MAC地址），OSPF协议（选择最优路径），搜索服务器地址，一边中转一边传输数据（网络层结束）

（数据链路层开始）到达后通过数据链路层，物理层负责0,1比特流与物理设备电压高低，光的闪灭之间的互换。数据链路层负责将0,1序列划分为数据帧从一个节点传输到临近的另一个节点，这些节点是通过MAC来唯一标识的（MAC，物理地址，一个中主机会有一个MAC地址）。 （数据链路层结束）

服务端通过数据链路层 - >通过网络层 - >再通过传输层（根据TCP协议接收请求报文并重组报文段） - >再通过应用层（通过HTTP协议对请求的内容进行处理） - >再通过应用层 - >传输层 - >网络层 - >数据链路层 - >到达客户端客户端通过数据链路层 - >网络层 - >传输层（根据TCP协议接收响应报文并重组） - >应用层（HTTP协议对响应进行处理） - >浏览器渲染页面 - >断开连接协议四次挥手）四次挥手主动方发送标志位：（ACK + FIN）+（发送序号= 200 +确认序号= 500）第一次挥手被动方接收后发送标志位：ACK +（发送序号=主动方确认序号500 +确认序号=主动方发送序号+1201）第二次挥手             标志位：（ACK + FIN）+（发送序号=主动方确认序号+1 501）第三次挥手主动方接收后发送标志位：（ACK）+（发送序号=被动方的确认序号201 +确认序号=被动方的发生序号+1502）









### 2.TCP/UDP相关

#### 2.1.普通 Hash 和一致性 Hash 原理

定义
Hash函数：一般翻译做散列、杂凑，或音译为哈希，是把任意长度的输入（又叫做预映射pre-image）通过散列算法变换成固定长度的输出，该输出就是散列值。
碰撞（冲突）：如果两个关键字通过hash函数得到的值是一样的，就是碰撞或冲突。
Hash表（散列表）：根据散列函数和冲突处理将一组关键字分配在连续的地址空间内，并以散列值记录在表中的存储位置，这种表成为散列表。

常用算法
直接寻址法：即取关键字或关键字的线性函数为散列地址：H(key)=key或H(key)=a*key+b;
数字分析法：即分析一组数据后采用的方法：如人的出生年月为92-09-03则前三位重复的几率比较大，容易产生碰撞，所以应该采用后三位作为hash值好点
平方取中法：取关键字平方的后几位。
折叠法：把关键字分割成位数相同的几部分，最后一部分可以位数不同，然后取这几部分的叠加值
随机数法：以关键值作为生成随机数的种子生成散列地址，通常适用于关键字长度不同的场合。
除留余数法：取关键字被某个不大于散列表长度m的数p除后所得的余数为散列地址:H(key)=key%p(p<=m);不仅可以对关键字直接取模，也可在折叠、平方取中等运算之后取模。对p的选择很重要，一般取素数或m，若p选的不好，容易产生碰撞。

一致性哈希主要就是解决当机器减少或增加的时候，大面积的数据重新哈希的问题，主要从下面 2 个方向去考虑的，当节点宕机时，数据记录会被定位到下一个节点上，当新增节点的时候 ，相关区间内的数据记录就需要重新哈希。

为了避免出现数据倾斜问题，一致性 Hash 算法引入了虚拟节点的机制，也就是每个机器节点会进行多次哈希，最终每个机器节点在哈希环上会有多个虚拟节点存在，使用这种方式来大大削弱甚至避免数据倾斜问题。同时数据定位算法不变，只是多了一步虚拟节点到实际节点的映射，例如定位到“D1#1”、“D1#2”、“D1#3”三个虚拟节点的数据均定位到 D1 上。这样就解决了服务节点少时数据倾斜的问题。在实际应用中，通常将虚拟节点数设置为32甚至更大，因此即使很少的服务节点也能做到相对均匀的数据分布。这也是 Dubbo 负载均衡中有一种一致性哈希负载均衡的实现思想。

一致性哈希用到的地方很多，特别是中间件里面，比如 Dubbo 的负载均衡也有一种策略是一致性哈希策略，使用的就是虚拟节点实现的。Redis 集群中也用到了相关思想但是没有用它而是根据实际情况改进了一下。而对于存储数据的节点水平切分的时候它的作用就更不可代替了。and so on···



#### 2.2.tcp三次握手

![TCP三次握手，四次挥手1](https://res-static.hc-cdn.cn/fms/img/e3e446f4b41ddd04a180196e2fdc9add1603447450544)

- 第一次握手(SYN=1, seq=x):

  客户端发送一个 TCP 的 SYN 标志位置1的包，指明客户端打算连接的服务器的端口，以及初始序号 X,保存在包头的序列号(Sequence Number)字段里。

  发送完毕后，客户端进入 `SYN_SEND` 状态。

- 第二次握手(SYN=1, ACK=1, seq=y, ACKnum=x+1):

  服务器发回确认包(ACK)应答。即 SYN 标志位和 ACK 标志位均为1。服务器端选择自己 ISN 序列号，放到 Seq 域里，同时将确认序号(Acknowledgement Number)设置为客户的 ISN 加1，即X+1。 发送完毕后，服务器端进入 `SYN_RCVD` 状态。

- 第三次握手(ACK=1，ACKnum=y+1)

  客户端再次发送确认包(ACK)，SYN 标志位为0，ACK 标志位为1，并且把服务器发来 ACK 的序号字段+1，放在确定字段中发送给对方，并且在数据段放写seq+1

  发送完毕后，客户端进入 `ESTABLISHED` 状态，当服务器端接收到这个包时，也进入 `ESTABLISHED` 状态，TCP 握手结束。

试想如果是用两次握手，则会出现下面这种情况：

> 如客户端发出连接请求，但因连接请求报文丢失而未收到确认，于是客户端再重传一次连接请求。后来收到了确认，建立了连接。数据传输完毕后，就释放了连接，客户端共发出了两个连接请求报文段，其中第一个丢失，第二个到达了服务端，但是第一个丢失的报文段只是在某些网络结点长时间滞留了，延误到连接释放以后的某个时间才到达服务端，此时服务端误认为客户端又发出一次新的连接请求，于是就向客户端发出确认报文段，同意建立连接，不采用三次握手，只要服务端发出确认，就建立新的连接了，此时客户端忽略服务端发来的确认，也不发送数据，则服务端一致等待客户端发送数据，浪费资源。



#### 2.3.四次挥手

![TCP三次握手，四次挥手2](https://res-static.hc-cdn.cn/fms/img/f191f236957a72720ca98da89c2a60ca1603447450545)

1）客户端进程发出连接释放报文，并且停止发送数据。释放数据报文首部，FIN=1，其序列号为seq=u（等于前面已经传送过来的数据的最后一个字节的序号加1），此时，客户端进入FIN-WAIT-1（终止等待1）状态。 TCP规定，FIN报文段即使不携带数据，也要消耗一个序号。

2）服务器收到连接释放报文，发出确认报文，ACK=1，ack=u+1，并且带上自己的序列号seq=v，此时，服务端就进入了CLOSE-WAIT（关闭等待）状态。TCP服务器通知高层的应用进程，客户端向服务器的方向就释放了，这时候处于半关闭状态，即客户端已经没有数据要发送了，但是服务器若发送数据，客户端依然要接受。这个状态还要持续一段时间，也就是整个CLOSE-WAIT状态持续的时间。

3）客户端收到服务器的确认请求后，此时，客户端就进入FIN-WAIT-2（终止等待2）状态，等待服务器发送连接释放报文（在这之前还需要接受服务器发送的最后的数据）。

4）服务器将最后的数据发送完毕后，就向客户端发送连接释放报文，FIN=1，ack=u+1，由于在半关闭状态，服务器很可能又发送了一些数据，假定此时的序列号为seq=w，此时，服务器就进入了LAST-ACK（最后确认）状态，等待客户端的确认。

5）客户端收到服务器的连接释放报文后，必须发出确认，ACK=1，ack=w+1，而自己的序列号是seq=u+1，此时，客户端就进入了TIME-WAIT（时间等待）状态。注意此时TCP连接还没有释放，必须经过2∗∗MSL（最长报文段寿命）的时间后，当客户端撤销相应的TCB后，才进入CLOSED状态。

6）服务器只要收到了客户端发出的确认，立即进入CLOSED状态。同样，撤销TCB后，就结束了这次的TCP连接。可以看到，服务器结束TCP连接的时间要比客户端早一些。

因为当服务端收到客户端的SYN连接请求报文后，可以直接发送SYN+ACK报文。其中ACK报文是用来应答的，SYN报文是用来同步的。但是关闭连接时，当服务端收到FIN报文时，很可能并不会立即关闭SOCKET，所以只能先回复一个ACK报文，告诉客户端，“你发的FIN报文我收到了”。只有等到我服务端所有的报文都发送完了，我才能发送FIN报文，因此不能一起发送。故需要四次挥手。



#### 2.5.**2MSL等待状态**以及为什么time_wait的时间是2MSL，不是MSL

MSL是Maximum Segment Lifetime的英文缩写，可译为“最长报文段寿命”，它是任何报文在网络上存在的最长时间，超过这个时间报文将被丢弃。

为了保证客户端发送的最后一个ACK报文段能够到达服务器。因为这个ACK有可能丢失，从而导致处在LAST-ACK状态的服务器收不到对FIN-ACK的确认报文。服务器会超时重传这个FIN-ACK，接着客户端再重传一次确认，重新启动时间等待计时器。最后客户端和服务器都能正常的关闭。假设客户端不等待2MSL，而是在发送完ACK之后直接释放关闭，一但这个ACK丢失的话，服务器就无法正常的进入关闭连接状态。

两个理由：

- 保证客户端发送的最后一个ACK报文段能够到达服务端。

这个ACK报文段有可能丢失，使得处于LAST-ACK状态的B收不到对已发送的FIN+ACK报文段的确认，服务端超时重传FIN+ACK报文段，而客户端能在2MSL时间内收到这个重传的FIN+ACK报文段，接着客户端重传一次确认，重新启动2MSL计时器，最后客户端和服务端都进入到CLOSED状态，若客户端在TIME-WAIT状态不等待一段时间，而是发送完ACK报文段后立即释放连接，则无法收到服务端重传的FIN+ACK报文段，所以不会再发送一次确认报文段，则服务端无法正常进入到CLOSED状态。

- 防止“已失效的连接请求报文段”出现在本连接中。

客户端在发送完最后一个ACK报文段后，再经过2MSL，就可以使本连接持续的时间内所产生的所有报文段都从网络中消失，使下一个新的连接中不会出现这种旧的连接请求报文段。

首先，存在这样的情况，某个路由器崩溃或者两个路由器之间的某个链接断开时，路由协议需要花费数秒到数分钟的时间才能稳定找出另一条通路。在这段时间内，可能发生路由循环（路由器A把分组发送给B，B又发送回给A），这种情况我们称之为迷途。假设迷途的分组是一个TCP分节，在迷途期间，发送端TCP超时并重传该分组，重传分组通过某路径到达目的地，而后不久（最多MSL秒）路由循环修复，早先迷失在这个循环中的分组最终也被送到目的地。这种分组被称之为重复分组或者漫游的重复分组，TCP必须要正确处理这些重复的分组。

我们假设ip1:port1和ip2:port2 之间有一个TCP连接。我们关闭了这个链接，过一段时间后在相同IP和端口之间建立了另一个连接。**TCP必须防止来自之前那个连接的老的重复分组在新连接上出现。为了做到这一点，TCP将不复用处于TIME_WAIT状态的连接。2MSL的时间足以让某个方向上的分组存活MSL秒后被丢弃，另一个方向上的应答也最多存活MSL秒后被丢弃。**

Client会在发送出ACK之后进入到TIME_WAIT状态。Client会设置一个计时器，等待2MSL的时间。如果在该时间内再次收到FIN，那么Client会重发ACK并再次等待2MSL。





#### 2.6.**TIME_WAIT数量太多以及解决方案**

一般情况服务器端不会出现TIME_WAIT状态，因为大多数情况都是客户端主动发起连接并主动关闭连接。但是某些服务如pop/smtp、ftp却是服务端收到客户端的QUIT命令后主动关闭连接，这就造成这类服务器上容易出现大量的TIME_WAIT状态的连接，而且并发量越大处于此种状态的连接越多。另外，对于被动关闭连接的服务在主动关闭客户端非法请求或清理长时间不活动的连接时（这种情况很可能是客户端程序忘记关闭连接）也会出现TIME_WAIT的状态。

有 TIME_WAIT 不一定不好，也不是因为它多，而是占用了资源致使不能创建更多的socket。
TIME_WAIT 对于web服务器来说，占用了一个socket 60秒，socket的数量是有限的，最多65535。

打开系统的TIMEWAIT重用和快速回收。

①解决发起端的IP地址,添加更多的IP(time_wait多的服务器)
②改用长链接方式





#### 2.7TCP如何保证可靠传输

**校验和：**

发送的数据包的二进制相加然后取反，**目的是检测数据在传输过程中的任何变化**。如果收到段的检验和有差错，TCP将丢弃这个报文段和不确认收到此报文段。 

**确认应答+序列号（累计确认+seq）：**

接收方收到报文就会确认（累积确认：对所有按序接收的数据的确认）

TCP给发送的**每一个包进行编号**，接收方**对数据包进行排序**，把有序数据传送给应用层。 

**超时重传：**

当TCP**发出一个段后，它启动一个定时器**，**等待目的端确认收到这个报文段**。**如果不能及时收到一个确认，将重发这个报文段**。 

**流量控制：**

**TCP连接的每一方都有固定大小的缓冲空间**，TCP的**接收端只允许发送端发送接收端缓冲区能接纳的数据**。当接收方来不及处理发送方的数据，能提示发送方降低发送的速率，防止包丢失。TCP使用的流量控制协议是可变大小的滑动窗口协议。

**接收方有即时窗口（滑动窗口），随ACK报文发送**

**拥塞控制：**

当网络拥塞时，减少数据的发送。

**发送方有拥塞窗口，发送数据前比对接收方发过来的即使窗口，取小**

**慢启动、拥塞避免、拥塞发送、快速恢复**



#### 2.8TCP滑动窗口与流量控制

  TCP采用可变滑动窗口来实现流量控制。TCP连接的两端交互作用，互相提供数据流的相关信息，包括报文段序列号、ACK号和窗口大小（即接收端的可用空间）。发送端根据这些信息动态调节窗口大小来控制发送，以达到流量控制的目的。每个TCP头部的窗口大小字段表明接收端可用缓存空间的大小，以字节为单位。该字段长度为16位，但窗口缩放选项可用大于65535的值。报文段发送方在相反方向上可接受的最大序列号值为TCP头部中ACK号和窗口大小字段之和（单位保持一致）。

###### 1. 滑动窗口作用

1. Reliability ，提供TCP的可靠性，TCP的传输要保证数据能够准确到达目的地，如果不能，需要能检测出来并且重新发送数据。
2. Data Flow Control，提供TCP的**流控特性**，管理发送数据的速率，不要超过设备的承载能力

###### 2. 滑动窗口组成

1. Sent and Acknowledged：这些数据表示已经发送成功并已经被确认的数据，比如图中的前31个bytes，这些数据其实的位置是在窗口之外了，因为窗口内顺序最低的被确认之后，要移除窗口，实际上是窗口进行合拢，同时打开接收新的带发送的数据
2. Send But Not Yet Acknowledged：这部分数据称为发送但没有被确认，数据被发送出去，没有收到接收端的ACK，认为并没有完成发送，这个属于窗口内的数据。
3. Not Sent，Recipient Ready to Receive：这部分是尽快发送的数据，这部分数据已经被加载到缓存中，也就是窗口中了，等待发送，其实这个窗口是完全有接收方告知的，接收方告知还是能够接受这些包，所以发送方需要尽快的发送这些包
4. Not Sent，Recipient Not Ready to Receive： 这些数据属于未发送，同时接收端也不允许发送的，因为这些数据已经超出了发送端所接收的范围

###### 3. 滑动窗口案例

1. Send Window ： 20个bytes 这部分值是有接收方在三次握手的时候进行通告的，同时在接收过程中也不断的通告可以发送的窗口大小，来进行适应
2. Window Already Sent: 已经发送的数据，但是并没有收到ACK。

TCP并不是每一个报文段都会回复ACK的，可能会对两个报文段发送一个ACK，也可能会对多个报文段发送1个ACK【累计ACK】，比如说发送方有1/2/3 3个报文段，先发送了2,3 两个报文段，但是接收方期望收到1报文段，这个时候2,3报文段就只能放在缓存中等待报文1的空洞被填上，如果报文1，一直不来，报文2/3也将被丢弃，如果报文1来了，那么会发送一个ACK对这3个报文进行一次确认。

举一个例子来说明一下滑动窗口的原理：

1. 假设32~45 这些数据，是上层Application发送给TCP的，TCP将其分成四个Segment来发往interne
2. seg1 32~~34 seg3 35~~36 seg3 37~~41 seg4 42~~45 这四个片段，依次发送出去，此时假设接收端之接收到了seg1 seg2 seg4
3. 此时接收端的行为是回复一个ACK包说明已经接收到了32~36的数据，并将seg4进行缓存（保证顺序，产生一个保存seg3 的hole）
4. 发送端收到ACK之后，就会将32~36的数据包从发送并没有确认切到发送已经确认，提出窗口，这个时候窗口向右移动
5. 假设接收端通告的Window Size仍然不变，此时窗口右移，产生一些新的空位，这些是接收端允许发送的范畴
6. **对于丢失的seg3，如果超过一定时间，TCP就会重新传送（重传机制），重传成功会seg3 seg4一块被确认，不成功，seg4也将被丢弃**

就是不断重复着上述的过程，随着窗口不断滑动，将真个数据流发送到接收端，实际上接收端的Window Size通告也是会变化的，接收端根据这个值来确定何时及发送多少数据，从对数据流进行流控。



#### 2.9.拥塞控制

TCP协议通过滑动窗口来进行流量控制，它是控制发送方的发送速度从而使接受者来得及接收并处理。而拥塞控制是作用于网络，它是防止过多的包被发送到网络中，避免出现网络负载过大，网络拥塞的情况。

1.慢开始
2.拥塞控制
3.快重传
4.快恢复

<img src="https://res-static.hc-cdn.cn/fms/img/26b4e606203b6c15ac22e28040fa7e4a1603441882875.png" alt="TCP的拥塞控制（详解）3" style="zoom:67%;" />

![img](https://pic3.zhimg.com/80/v2-de79bf2c38bddb0c1caf5768577648e2_1440w.jpg)

![TCP的拥塞控制（详解）6](https://res-static.hc-cdn.cn/fms/img/e25a4a79b538197e69c7356766d688861603441882876.png)

![TCP的拥塞控制（详解）7](https://res-static.hc-cdn.cn/fms/img/a7741d3223791e04828b06cf566a6bf71603441882876.png)





#### 2.10.TCP 如何解决流控、乱序、丢包问题

**1 ACK 携带信息**

- 1）期待要收到下一个数据包的编号；
- 2）接收方的接收窗口的剩余容量。

每一个数据包都带有下一个数据包的编号。如果下一个数据包没有收到，那么 ACK 的编号就不会发生变化。

接收端给发送端的Ack确认只会确认最后一个连续的包，比如，发送端发了1,2,3,4,5一共五份数据，接收端收到了1，2，于是回ack 3，然后收到了4（注意此时3没收到），此时的TCP会怎么办？我们要知道，因为正如前面所说的，SeqNum和Ack是以字节数为单位，所以ack的时候，不能跳着确认，只能确认最大的连续收到的包，不然，发送端就以为之前的都收到了。如果发送方发现收到连续的重复 ACK，或者超时了还没有收到任何 ACK，就会确认丢包，从而再次发送这个包。通过重传这种机制，TCP 保证了不会有数据包丢失

一种是不回ack，死等3，当发送方发现收不到3的ack超时后，会重传3。一旦接收方收到3后，会ack 回 4——意味着3和4都收到了。

但是，这种方式会有比较严重的问题，那就是因为要死等3，所以会导致4和5即便已经收到了，而发送方也完全不知道发生了什么事，因为没有收到Ack，所以，发送方可能会悲观地认为也丢了，所以有可能也会导致4和5的重传。

接收端告诉发送端自己还有多少缓冲区可以接收数据。于是发送端就可以根据这个接收端的处理能力来发送数据，而不会导致接收端处理不过来。 接收端在给发送端回ACK中会汇报自己的可用窗口大小，而发送方会根据这个窗口来控制发送数据的大小，以保证接收方可以处理。





#### 2.11.为什么会出现粘包和拆包，如何解决

1、TCP是基于字节流的，虽然应用层和传输层之间的数据交互是大小不等的数据块，但是TCP把这些数据块仅仅看成一连串无结构的字节流，没有边界；
2、在TCP的首部没有表示数据长度的字段，基于上面两点，在使用TCP传输数据时，才有粘包或者拆包现象发生的可能。

- 发送端将每个包都封装成固定的长度，比如100字节大小。如果不足100字节可通过补0或空等进行填充到指定长度；
- 发送端在每个包的末尾使用固定的分隔符，例如\r\n。如果发生拆包需等待多个包发送过来之后再找到其中的\r\n进行合并；例如，FTP协议；
- 将消息分为头部和消息体，头部中保存整个消息的长度，只有读取到足够长度的消息之后才算是读到了一个完整的消息；
- 通过自定义协议进行粘包和拆包的处理。





#### 2.12.**SYN攻击是什么？**

服务器端的资源分配是在二次握手时分配的，而客户端的资源是在完成三次握手时分配的，所以服务器容易受到SYN洪泛攻击。SYN攻击就是Client在短时间内伪造大量不存在的IP地址，并向Server不断地发送SYN包，Server则回复确认包，并等待Client确认，由于源地址不存在，因此Server需要不断重发直至超时，这些伪造的SYN包将长时间占用未连接队列，导致正常的SYN请求因为队列满而被丢弃，从而引起网络拥塞甚至系统瘫痪。SYN 攻击是一种典型的 DoS/DDoS 攻击。

检测 SYN 攻击非常的方便，当你在服务器上看到大量的半连接状态时，特别是源IP地址是随机的，基本上可以断定这是一次SYN攻击。在 Linux/Unix 上可以使用系统自带的 netstats 命令来检测 SYN 攻击。

```javascript
netstat -n -p TCP | grep SYN_RECV
```

常见的防御 SYN 攻击的方法有如下几种：

- 缩短超时（SYN Timeout）时间
- 增加最大半连接数
- 过滤网关防护
- SYN cookies技术



#### 2.13.**什么是半连接队列？**

服务器第一次收到客户端的 SYN 之后，就会处于 SYN_RCVD 状态，此时双方还没有完全建立其连接，服务器会把此种状态下请求连接放在一个队列里，我们把这种队列称之为半连接队列。

当然还有一个全连接队列，就是已经完成三次握手，建立起连接的就会放在全连接队列中。如果队列满了就有可能会出现丢包现象。

这里在补充一点关于SYN-ACK 重传次数的问题：

服务器发送完SYN-ACK包，如果未收到客户确认包，服务器进行首次重传，等待一段时间仍未收到客户确认包，进行第二次重传。如果重传次数超过系统规定的最大重传次数，系统将该连接信息从半连接队列中删除。

注意，每次重传等待的时间不一定相同，一般会是指数增长，例如间隔时间为 1s，2s，4s，8s…



#### 2.14TCP与UDP得区别

1、基于连接与无连接；

2、对系统资源的要求（TCP较多，UDP少）；

3、UDP程序结构较简单；

4、流模式与数据报模式 ；

5、TCP保证数据正确性，UDP可能丢包；

6、TCP保证数据顺序，UDP不保证。

- - #### 1. TCP和UDP的应用场景

    **UDP协议比TCP协议的效率更高**

    - 什么时候使用TCP
      - 当对网络通讯质量有要求的时候，比如：整个数据要准确无误的传递给对方，这往往用于一些要求可靠的应用，比如**HTTP、HTTPS**、FTP等传输文件的协议，POP、SMTP等邮件传输的协议。
    - 什么时候使用UDP
      - 对通讯质量要求不严的场景：QQ**视频**、语音、DNS协议

    #### 2. 怎么理解TCP的面向连接和UDP的无连接（不面向连接）

    - 实际上就是在客户端和服务器端都维护一个变量，这个变量维护现在数据传输的状态，例如传输了哪些数据，下一次需要传输哪些数据，等等，并不是真的我们想象中的真的有什么东西连接着这两端，因为无论对于有连接还是无连接，都有网线连着呢(不包括无线网)，所以连接根本就不是是否真的有什么东西把他们连接起来，真实的含义就是我上面说的，两边维护一个状态变量。
    - **UDP通讯有四个参数：源IP、源端口、目的IP和目的端口。而TCP通讯至少有有六个参数：源IP、源端口、目的IP和目的端口，以及序列号和应答号。** 序列号和应答号是TCP通讯特有的参数，TCP通讯利用序列号和应答号来保持和确认数据的关联与正确性，是在三次握手中确定的，不正确的序列号和应答号会导致无法正常通讯。因此对TCP连接的连接概念可以简单理解成为同UDP通讯相比，用序列号和应答号确定了相互之间的连接特征，来保证数据传输的正确性。





#### 2.15socket

TCP协议简化一下，就只有三个核心功能：建立连接、发送数据以及接收数据。我们再来看一下Java中提供的Socket类中的核心功能：

![img](https://pic3.zhimg.com/80/v2-c0f04274bf5557c2aa4940359d84593a_1440w.jpg)

可以把Socket编程理解为对TCP协议的具体实现。Socket编程基本就是listen，accept以及send，write等几个基本的操作。

套接字（socket）是通信的基石，是支持TCP/IP协议的网络通信的基本操作单元。它是网络通信过程中端点的抽象表示，包含进行网络通信必须的五种信息：连接使用的协议，本地主机的IP地址，本地进程的协议端口，远地主机的IP地址，远地进程的协议端口。





#### 2.16TCP头

<img src="https://img-blog.csdn.net/20170501223823511" alt="img" style="zoom:67%;" />




#### 2.17http的keep-alive和tcp的keepalive区别

**1、HTTP Keep-Alive**
在http早期，每个http请求都要求打开一个tpc socket连接，并且使用一次之后就断开这个tcp连接。
使用keep-alive可以改善这种状态，即在一次TCP连接中可以持续发送多份数据而不会断开连接。通过使用keep-alive机制，可以减少tcp连接建立次数，也意味着可以减少TIME_WAIT状态连接，以此提高性能和提高httpd服务器的吞吐率(更少的tcp连接意味着更少的系统内核调用,socket的accept()和close()调用)。
但是，keep-alive并不是免费的午餐,长时间的tcp连接容易导致系统资源无效占用。配置不当的keep-alive，有时比重复利用连接带来的损失还更大。所以，正确地设置keep-alive timeout时间非常重要。
keepalvie timeout
Httpd守护进程，一般都提供了keep-alive timeout时间设置参数。比如nginx的keepalive_timeout，和Apache的KeepAliveTimeout。这个keepalive_timout时间值意味着：一个http产生的tcp连接在传送完最后一个响应后，还需要hold住keepalive_timeout秒后，才开始关闭这个连接。
当httpd守护进程发送完一个响应后，理应马上主动关闭相应的tcp连接，设置 keepalive_timeout后，httpd守护进程会想说：”再等等吧，看看浏览器还有没有请求过来”，这一等，便是keepalive_timeout时间。如果守护进程在这个等待的时间里，一直没有收到浏览发过来http请求，则关闭这个http连接。
**2、TCP KEEPALIVE**
链接建立之后，如果应用程序或者上层协议一直不发送数据，或者隔很长时间才发送一次数据，当链接很久没有数据报文传输时如何去确定对方还在线，到底是掉线了还是确实没有数据传输，链接还需不需要保持，这种情况在TCP协议设计中是需要考虑到的。
TCP协议通过一种巧妙的方式去解决这个问题，当超过一段时间之后，TCP自动发送一个数据为空的报文给对方，如果对方回应了这个报文，说明对方还在线，链接可以继续保持，如果对方没有报文返回，并且重试了多次之后则认为链接丢失，没有必要保持链接。

http keep-alive是为了让tcp活得更久一点，以便在同一个连接上传送多个http，提高socket的效率。而tcp keep-alive是TCP的一种检测TCP连接状况的保鲜机制。



#### 2.18QUIC协议

①多路复用：但是基于UDP的，就算其中有一个资源丢包，也不需要把全部资源重传，只需要单独传被丢失的那个包。

  ②加密认证的报文：因为TCP协议头部没有任何加密和认证，所以在传输过程中容易被中间网络设备篡改，注入和窃听。QUIC将所有报文头、报文Body都经过加密，只要对QUIC报文有任何修改，接收端都能及时发现。

  ③向前纠错：传输的每个数据包都包含了其他数据包的数据，如果其中一个丢包了，可以从其他包的冗余数据直接组装而无需重传。但是出现丢了多个包，也只有重传。




#### 2.19超时重传

**就是发送端死等接收端的ack，直到发送端超时之后，在发送一个包，直到收到接收端的ack为止。**

例如：接收端给发送端的Ack确认只会确认最后一个连续的包，比如，发送端发了1,2,3,4,5一共五份数据，接收端收到了1，2，于是回ack 3，然后收到了4（注意此时3没收到），此时的TCP会怎么办？等待发送端的ACK 3，直到超时后，就会再发送3。

当发送端发送数据，发生丢包时，则丢掉的包的ACK一直不会返回。此时发送端就一直等那个ACK返回，若超时，则重传该数据包。对于**超时时间RTO（Retransmission TimeOut）**，有很多复杂的算法。RTO的选择很重要，选短了，可能只是返回时间长但并未丢包，却当做丢包。选长了，迟迟不发丢的包也是个问题。





### 3.session相关

HTTP无状态协议，是指协议对于事务处理没有记忆能力。缺少状态意味着如果后续处理需要前面的信息，则它必须重传，这样可能导致每次连接传送的数据量增大。另一方面，在服务器不需要先前信息时它的应答就较快。

TCP协议是一种有状态协议,因为它是什么,而不是因为它是通过IP使用的,或者因为HTTP构建在它之上. TCP以窗口大小的形式维护状态(端点告知彼此准备好接收多少数据)和数据包顺序(端点必须在从另一个接收数据包时彼此确认).这个状态(另一个人可以接收多少字节,以及他是否接收到最后一个数据包)允许TCP甚至在固有的非可靠协议上是可靠的.因此,TCP是一种有状态协议,因为它需要状态才有用

cookie是本地客户端用来存储少量数据信息的，保存在客户端，用户能够很容易的获取，安全性不高，存储的数据量小
session是服务器用来存储部分数据信息，保存在服务器，用户不容易获取，安全性高，储存的数据量相对大，存储在服务器，会占用一些服务器资源，但是对于它的优点来说，这个缺点可以忽略了

在一次客户端和服务器为之间的会话中，客户端(浏览器)向服务器发送请求，首先cookie会自动携带上次请求存储的数据(JSESSIONID)到服务器，服务器根据请求参数中的JSESSIONID到服务器中的session库中查询是否存在此JSESSIONID的信息，如果存在，那么服务器就知道此用户是谁，如果不存在，就会创建一个JSESSIONID，并在本次请求结束后将JSESSIONID返回给客户端，同时将此JSESSIONID在客户端cookie中进行保存

客户端和服务器之间是通过http协议进行通信，但是http协议是无状态的，不同次请求会话是没有任何关联的，但是优点是处理速度快

session是一次浏览器和服务器的交互的会话，当浏览器关闭的时候，会话就结束了，但是会话session还在，默认session是还保留30分钟的

分布式session一致性

方案一：客户端存储

方案二：session复制

方案三：session绑定

方案四：基于redis存储session方案





### 4.OSI模型 7/4/5

1、应用层：最靠近用户的OSI层，这一层为用户的应用程序如：（电子邮件、文件传输）提供网络服务。

2、表示层：提供用于应用层数据编码和转换功能，可以确保一个系统的应用层发送的信息被另一个系统的应用层识别读取。数据的价码和压缩也是表示层可提供的转换功能之一。例如：PC程序与另一台程序计算机进行通信，其中一台计算机使用扩展二十一进制交换码（EBDIC），二另一台则使用美国信息交换码（ASCII)来表示相同的字符，表示层会通过使用一种通用格式来实现多种数据格式之间的转换。

3、会话层：通过传输层（端口号：传输端口与接收端口）简历数据传输通路。主要在系统之间发起会话或者接受会话请求。（设备之间需要相互认识可以是IP地址也可以是MAC地址或者主机名）

4、传输层：定义了一些传输数据的协议和端口号（WWW端口80等），TCP、UDP。建立主机端对端的连接，为上层协议提供端对端的可靠透明的数据传输服务，将从下层的接受的数据进行分段和传输，到达目的地后再进行传输。

5、网络层：通过IP寻址，建立两个节点之间的连接，选择正确的路由和交换节点，正确无误的按照地址传送给目的端的传输层。也就是说网络层是在位于不同地理位置的网络的两个主机之间提供连接和路径选择。

6、数据链路层：将比特组成字节，再将字节组成帧，使用链路层地址来访问（以太网使用MAC地址）介质，并进行差错检测。

7、物理层：实现最终信号的传输是通过物理层实现的，通过物理介质传输比特流。网线等就是物理层的传输介质。

物理层：RJ45、CLOCK、IEEE802.3（中继器、集线器）

数据链路层：PPP、FR、HDLC、VLAN、MAC（网桥、交换机）

网络层：IP、ARP、 RARP、ICMP、OSPF、IPX、RIP、IGRP（交换机）

传输层：TCP、UDP、 SPX

会话层：NFS、SQL、NETBIOS、RPC

表示层：JPEG、MPEG、ASII

应用层：FTP、DNS、Telnet、SMTP、HTTP、HTTPS、WWW、NFS



### 5.跨域问题

#### 1. 浏览器同源策略

因为[浏览器的同源策略](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy)规定某域下的客户端在没明确授权的情况下，不能读写另一个域的资源。而在实际开发中，前后端常常是相互分离的，并且前后端的项目部署也常常不在一个服务器内或者在一个服务器的不同端口下。前端想要获取后端的数据，就必须发起请求，如果不做一些处理，就会受到浏览器同源策略的约束。后端可以收到请求并返回数据，但是前端无法收到数据。

#### 2. 假设没有同源策略

有一个小小的东西叫cookie大家应该知道，一般用来处理登录等场景，目的是让服务端知道谁发出的这次请求。如果你请求了接口进行登录，服务端验证通过后会在响应头加入Set-Cookie字段，然后下次再发请求的时候，浏览器会自动将cookie附加在HTTP请求的头字段Cookie中，服务端就能知道这个用户已经登录过了。知道这个之后，我们来看场景：

1.你准备去清空你的购物车，于是打开了买买买网站www.maimaimai.com，然后登录成功，一看，购物车东西这么少，不行，还得买多点。

2.你在看有什么东西买的过程中，你的好基友发给你一个链接www.nidongde.com，一脸yin笑地跟你说：“你懂的”，你毫不犹豫打开了。

3.你饶有兴致地浏览着www.nidongde.com，谁知这个网站暗地里做了些不可描述的事情！由于没有同源策略的限制，它向www.maimaimai.com发起了请求！聪明的你一定想到上面的话“服务端验证通过后会在响应头加入Set-Cookie字段，然后下次再发请求的时候，浏览器会自动将cookie附加在HTTP请求的头字段Cookie中”，这样一来，这个不法网站就相当于登录了你的账号，可以为所欲为了！如果这不是一个买买买账号，而是你的银行账号，那……

#### 3. 解决方案

##### 1. Cross Origin Resource Share (CORS)

CORS是一个**跨域资源共享**方案，为了解决跨域问题，通过增加一系列请求头和响应头，规范安全地进行跨站数据传输。

**请求头主要包括**

| 请求头                             | 解释                                                         |
| ---------------------------------- | ------------------------------------------------------------ |
| **Origin**                         | Origin头在跨域请求或预先请求中，标明发起跨域请求的源域名。   |
| **Access-Control-Request-Method**  | Access-Control-Request-Method头用于表明跨域请求使用的实际HTTP方法 |
| **Access-Control-Request-Headers** | Access-Control-Request-Headers用于在预先请求时，告知服务器要发起的跨域请求中会携带的请求头信息 |
| **with-credentials**               | **跨域请求携带cookie**                                       |

**响应头主要包括**

| 响应头                            | 解释                                                         |
| --------------------------------- | ------------------------------------------------------------ |
| **Access-Control-Allow-Origin**   | Access-Control-Allow-Origin头中携带了服务器端验证后的允许的跨域请求域名，可以是一个具体的域名或是一个*（表示任意域名）。 |
| **Access-Control-Expose-Headers** | Access-Control-Expose-Headers头用于允许返回给跨域请求的响应头列表，在列表中的响应头的内容，才可以被浏览器访问。 |
| **Access-Control-Max-Age**        | Access-Control-Max-Age用于告知浏览器可以将预先检查请求返回结果缓存的时间，在缓存有效期内，浏览器会使用缓存的预先检查结果判断是否发送跨域请求。 |
| **Access-Control-Allow-Methods**  | Access-Control-Allow-Methods用于告知浏览器可以在实际发送跨域请求时，可以支持的请求方法，可以是一个具体的方法列表或是一个*（表示任意方法）。 |

SpringBoot解决方案

```
HttpServletResponse httpServletResponse = (HttpServletResponse) response;
        String temp = request.getHeader("Origin");
        httpServletResponse.setHeader("Access-Control-Allow-Origin", temp);
        // 允许的访问方法
        httpServletResponse.setHeader("Access-Control-Allow-Methods", "POST, GET, PUT, OPTIONS, DELETE, PATCH");
        //Access-Control-Max-Age 用于 CORS 相关配置的缓存
        httpServletResponse.setHeader("Access-Control-Max-Age", "3600");
        httpServletResponse.setHeader("Access-Control-Allow-Headers",
                "Origin, X-Requested-With, Content-Type, Accept,token");
        httpServletResponse.setHeader("Access-Control-Allow-Credentials", "true");
```



### 6.IP地址

IP公网地址和私网地址?

**公网IP地址** 公有地址分配和管理由Inter NIC（Internet Network Information Center 因特网信息中心）负责。各级ISP使用的公网地址都需要向Inter NIC提出申请，有Inter NIC统一发放，这样就能确保地址块不冲突。

**私网IP地址** 创建IP寻址方案的人也创建了私网IP地址。这些地址可以被用于私有网络，在Internet没有这些IP地址，Internet上的路由器也没有到私有网络的路由表。

- A类：10.0.0.0 255.0.0.0，保留了1个A类网络。
- B类：172.16.0.0 255.255.0.0～172.31.0.0 255.255.0.0，保留了16个B类网络。
- C类：192.168.0.0 255.255.255.0～192.168.255.0 255.255.255.0，保留了256个C类网络。

PS：私网地址访问Internet需要做NAT或PAT网络地址转换

**备注：NAT是什么？**

NAT是 Network Address Translation 网络地址转换的缩写。 NAT是将私有IP地址通过边界路由转换成外网IP地址，在边界路由的NAT地址转换表记录下这个转换映射记录，当外部数据返回时，路由使用NAT技术查询NAT转换表，再将目标地址替换成内网用户IP地址。

**PAT(port address Translation，端口地址转换，也叫端口地址复用)**

这是最常用的NAT技术，也是IPv4能够维持到今天的最重要的原因之一，它提供了一种多对一的方式，对多个内网IP地址，边界路由可以给他们分配一个外网IP，利用这个外网IP的不同端口和外部进行通信。

**附加问题：为什么需要使用子网掩码**

虽然我们说子网掩码可以分离出 *IP* 地址中的网络部分与主机部分，可大家还是会有疑问，比如为什么要区分网络地址与主机地址？区分以后又怎样呢？那么好，让我们再详细的讲一下吧！

在使用 *TCP/IP* 协议的两台计算机之间进行通信时，我们通过将本机的子网掩码与接受方主机的 *IP* 地址进行 *'* 与 *'* 运算，即可得到目标主机所在的网络号，又由于每台主机在配置 *TCP/IP* 协议时都设置了一个本机 *IP* 地址与子网掩码，所以可以知道本机所在的网络号。

通过比较这两个网络号，就可以知道接受方主机是否在本网络上。如果网络号相同，表明接受方在本网络上，那么可以通过相关的协议把数据包直接发送到目标主机；如果网络号不同，表明目标主机在远程网络上，那么数据包将会发送给本网络上的路由器，由路由器将数据包发送到其他网络，直至到达目的地。在这个过程中你可以看到，子网掩码是不可或缺的！

**计算方式**

*1.* 将 *IP* 地址与子网掩码转换成二进制；

*2.* 将二进制形式的 *IP* 地址与子网掩码做 *'* 与 *'* 运算，将答案化为十进制便得到网络地址；

*3.* 将二进制形式的子网掩码取 *'* 反 *'* ；

*4.* 将取 *'* 反 *'* 后的子网掩码与 *IP* 地址做 *'* 与 *'* 运算，将答案化为十进制便得到主机地址。

下面我们用一个例子给大家演示：

假设有一个 *I P* 地址： *192.168.0.1*

子网掩码为： *255.255.255.0*

化为二进制为： *I P* 地址 *11000000.10101000.00000000.00000001*

子网掩码 *11111111.11111111.11111111.00000000*

将两者做 *'* 与 *'* 运算得： *11000000.10101000.00000000.00000000*

将其化为十进制得： *192.168.0.0*

这便是上面 *IP* 的网络地址，主机地址以此类推。

**小技巧：由于观察到上面的子网掩码为 C** **类地址的默认子网掩码（至于为什么，可看后面的子网掩码分类就明白了），便可直接看出网络地址为 IP** **地址的前三部分，即前三个字节，主机地址为最后一部分。**



### 7.网络层协议

#### 1. PING原理

- 作用：检测两台计算机是否联通
- 核心是ICMP协议
- 过程
  - 首先，Ping命令会构建一个固定格式的ICMP请求数据包，然后由ICMP协议将这个数据包连同地址“192.168.1.2”一起交给IP层协议（和ICMP一样，实际上是一组后台运行的进程），IP层协议将以地址“192.168.1.2”作为目的地址，本机IP地址作为源地址，加上一些其他的控制信息，构建一个IP数据包，并在一个映射表中查找出IP地址192.168.1.2所对应的物理地址（也叫MAC地址，熟悉网卡配置的朋友不会陌生，这是数据链路层协议构建数据链路层的传输单元——帧所必需的），一并交给数据链路层。后者构建一个数据帧，目的地址是IP层传过来的物理地址，源地址则是本机的物理地址，还要附加上一些控制信息，依据以太网的介质访问规则，将它们传送出去。 其中映射表由ARP实现。ARP(Address Resolution Protocol)是地址解析协议,是一种将IP地址转化成物理地址的协议。**ARP具体说来就是将网络层（IP层，也就是相当于OSI的第三层）地址解析为数据连接层（MAC层，也就是相当于OSI的第二层）的MAC地址。**
  - **主机B收到这个数据帧后，先检查它的目的地址，并和本机的物理地址对比，如符合，则接收；否则丢弃。接收后检查该数据帧，将IP数据包从帧中提取出来，交给本机的IP层协议。同样，IP层检查后，将有用的信息提取后交给ICMP协议，后者处理后，马上构建一个ICMP应答包，发送给主机A，其过程和主机A发送ICMP请求包到主机B一模一样。**

#### 2.ICMP协议

前面讲到了，IP协议并不是一个可靠的协议，它不保证数据被送达，那么，自然的，保证数据送达的工作应该由其他的模块来完成。其中一个重要的模块就是ICMP(网络控制报文)协议。

**当传送IP数据包发生错误－－比如主机不可达，路由不可达等等，ICMP协议将会把错误信息封包，然后传送回给主机。**给主机一个处理错误的机会，这 也就是为什么说建立在IP层以上的协议是可能做到安全的原因。







## 操作系统

### 1.虚拟内存

**虚拟内存**是计算机系统内存管理的一种技术。它使得应用程序认为它拥有连续可用的内存（一个连续完整的地址空间），而实际上[物理内存](https://zh.wikipedia.org/wiki/物理内存)通常被分隔成多个[内存碎片](https://zh.wikipedia.org/wiki/碎片化)，还有部分暂时存储在外部磁盘存储器上，在需要时进行数据交换。与没有使用虚拟内存技术的系统相比，使用这种技术使得大型程序的编写变得更容易，对真正的物理内存（例如[RAM](https://zh.wikipedia.org/wiki/隨機存取記憶體)）的使用也更有效率。此外，虚拟内存技术可以使多个[进程](https://zh.wikipedia.org/wiki/行程)共享同一个[运行库](https://zh.wikipedia.org/wiki/函式庫)，并通过分割不同进程的内存空间来提高系统的安全性。

虚拟内存的实现需要建立在离散分配的内存管理方式的基础上，虚拟内存的实现有以下三种方式：

- 请求分页存储管理
- 请求分段存储管理
- 请求段页式存储管理

不管哪种方式，都需要有一定的硬件支持。一般需要的支持有以下几个方面：

- 一定容量的内存和外存
- 页表机制（或段表机制），作为主要的数据结构
- 中断机构，当用户程序要访问的部分尚未调入内存，则产生中断
- 地址变换机构，逻辑地址到物理地址的变换



### 2.进程线程相关

#### 1.概念

进程:

进程是程序的一次执行过程，是程序在执行过程中的分配和管理资源的基本单位，每个进程都有自己的地址空间，线程至少有 5 种状态：初始态、执行态、等待态、就绪态、终止态。

线程：

线程是CPU调度和分派的基本单位，它可以和同一进程下的其他线程共享全部资源

联系：

线程是进程中的一部分，一个进程可以有多个线程，但线程只能存在于一个进程中。

区别：

1. 根本区别：进程是操作系统资源调度的基本单位，线程是任务的调度执行的基本单位
2. 开销方面：进程都有自己的独立数据空间，程序之间的切换开销大；线程也有自己的运行栈和程序计数器，线程间的切换开销较小。
3. 共享空间：进程拥有各自独立的地址空间、资源，所以共享复杂，需要用IPC（Inter-Process Communication，进程间通信），但是同步简单。而线程共享所属进程的资源，因此共享简单，但是同步复杂，需要用加锁等措施。

设计线程的原因：

操作系统模型中，进程有两个功能：

1、任务的调度执行基本单位

2、资源的所有权

线程的出现就是将这两个功能分离开来了：thread 执行任务的调度和执行 ； process 资源所有权

这样的好处是：

操作系统中有两个重要概念：并发和隔离

并发：提高硬件利用率，进程的上下文切换比线程的上下文切换效率低，所以线程可以提高并发的效率

隔离：计算机的资源是共享的，当程序发生奔溃时，需要保证这些资源要被回收，进程的资源是独立的，奔溃时不会影响其他程 序的进行，线程资源是共享的，奔溃时整个进程也会奔溃

线程和并发有关系，进程和隔离有关系





#### 2.进程通信

进程之间要交换数据必须通过内核，在内核中开辟一块缓冲区，进程1把数据从用户空间拷到内核缓冲区，进程2再从内核缓冲区把数据读走，内核提供的这种机制称为**进程间通信**

**1. 管道/匿名管道(pipe)**

- 管道是半双工的，数据只能向一个方向流动；需要双方通信时，需要建立起两个管道。
- 只能用于父子进程或者兄弟进程之间(具有亲缘关系的进程);
- 单独构成一种独立的文件系统：管道对于管道两端的进程而言，就是一个文件，但它不是普通的文件，它不属于某种文件系统，而是自立门户，单独构成一种文件系统，并且只存在与内存中。
- 数据的读出和写入：一个进程向管道中写的内容被管道另一端的进程读出。写入的内容每次都添加在管道缓冲区的末尾，并且每次都是从缓冲区的头部读出数据。

**管道的实质：**
 管道的实质是一个内核缓冲区，进程以先进先出的方式从缓冲区存取数据，管道一端的进程顺序的将数据写入缓冲区，另一端的进程则顺序的读出数据。
 该缓冲区可以看做是一个循环队列，读和写的位置都是自动增长的，不能随意改变，一个数据只能被读一次，读出来以后在缓冲区就不复存在了。
 当缓冲区读空或者写满时，有一定的规则控制相应的读进程或者写进程进入等待队列，当空的缓冲区有新数据写入或者满的缓冲区有数据读出来时，就唤醒等待队列中的进程继续读写。

**2. 有名管道(FIFO)**
 匿名管道，由于没有名字，只能用于亲缘关系的进程间通信。为了克服这个缺点，提出了有名管道(FIFO)。
 有名管道不同于匿名管道之处在于它提供了一个路径名与之关联，**以有名管道的文件形式存在于文件系统中**，这样，**即使与有名管道的创建进程不存在亲缘关系的进程，只要可以访问该路径，就能够彼此通过有名管道相互通信**，因此，通过有名管道不相关的进程也能交换数据。值的注意的是，有名管道严格遵循**先进先出(first in first out)**,对匿名管道及有名管道的读总是从开始处返回数据，对它们的写则把数据添加到末尾。它们不支持诸如lseek()等文件定位操作。**有名管道的名字存在于文件系统中，内容存放在内存中。**

**3. 信号(Signal)**

- 信号是Linux系统中用于进程间互相通信或者操作的一种机制，信号可以在任何时候发给某一进程，而无需知道该进程的状态。
- 如果该进程当前并未处于执行状态，则该信号就有内核保存起来，知道该进程回复执行并传递给它为止。
- 如果一个信号被进程设置为阻塞，则该信号的传递被延迟，直到其阻塞被取消是才被传递给进程。

**4. 消息(Message)队列**

- 消息队列是存放在内核中的消息链表，每个消息队列由消息队列标识符表示。
- 与管道（无名管道：只存在于内存中的文件；命名管道：存在于实际的磁盘介质或者文件系统）不同的是消息队列存放在内核中，只有在内核重启(即，操作系统重启)或者显示地删除一个消息队列时，该消息队列才会被真正的删除。
- 另外与管道不同的是，消息队列在某个进程往一个队列写入消息之前，并不需要另外某个进程在该队列上等待消息的到达。

**5. 共享内存(share memory)**

- 使得多个进程可以可以直接读写同一块内存空间，是最快的可用IPC形式。是针对其他通信机制运行效率较低而设计的。
- 为了在多个进程间交换信息，内核专门留出了一块内存区，可以由需要访问的进程将其映射到自己的私有地址空间。进程就可以直接读写这一块内存而不需要进行数据的拷贝，从而大大提高效率。
- 由于多个进程共享一段内存，因此需要依靠某种同步机制（如信号量）来达到进程间的同步及互斥。

**6. 信号量(semaphore)**
 信号量是一个计数器，用于多进程对共享数据的访问，信号量的意图在于进程间同步。
 为了获得共享资源，进程需要执行下列操作：
 （1）**创建一个信号量**：这要求调用者指定初始值，对于二值信号量来说，它通常是1，也可是0。
 （2）**等待一个信号量**：该操作会测试这个信号量的值，如果小于0，就阻塞。也称为P操作。
 （3）**挂出一个信号量**：该操作将信号量的值加1，也称为V操作。

**7. 套接字(socket)**
 套接字是一种通信机制，凭借这种机制，客户/服务器（即要进行通信的进程）系统的开发工作既可以在本地单机上进行，也可以跨网络进行。也就是说它可以让不在同一台计算机但通过网络连接计算机上的进程进行通信。



#### 3.线程通信

\# 锁机制：包括互斥锁、条件变量、读写锁
*互斥锁提供了以排他方式防止数据结构被并发修改的方法。
*读写锁允许多个线程同时读共享数据，而对写操作是互斥的。
*条件变量可以以原子的方式阻塞进程，直到某个特定条件为真为止。对条件的测试是在互斥锁的保护下进行的。条件变量始终与互斥锁一起使用。
\# 信号量机制(Semaphore)：包括无名线程信号量和命名线程信号量
\# 信号机制(Signal)：类似进程间的信号处理
线程间的通信目的主要是用于线程同步，所以线程没有像进程通信中的用于数据交换的通信机制。

采用信号SIGCHLD通知处理，并在信号处理程序中调用wait函数



#### 4.僵尸进程孤儿进程

运行程序的时候，一个父进程可能会有多个子进程跑

子进程**执行完毕后**会发送一个exit()信号，父进程没有去处理，导致这个子进程一直在进程表中。

解决方法：

1.重启服务器电脑

2.找到僵尸进程的父进程杀掉 kill -9，而不是去杀僵尸线程。

  父进程退出后，而它的子进程**还在运行**，那么这些子进程就是孤儿进程。

**解决方法：**

成了孤儿进程后，有init进程对它进行操作，最后孤儿进程在init进程下结束生命周期。

避免方法

一、让僵尸进程的父进程来回收，父进程每隔一段时间来查询子进程是否结束并回收，调用wait()或者waitpid(),通知内核释放僵尸进程



#### 5.协程是什么

**协程，英文Coroutines，是一种比线程更加轻量级的存在。**正如一个进程可以拥有多个线程一样，一个线程也可以拥有多个协程。

最重要的是，**协程不是被操作系统内核所管理，而完全是由程序所控制（也就是在用户态执行）**。

这样带来的好处就是性能得到了很大的提升，不会像线程切换那样消耗资源。

由于Java的原生语法中并没有实现协程。



#### 6.进程调度方式

##### 先来先服务(FCFS)

先来先服务(FCFS)调度算法是一种最简单的调度算法，该算法既可用于作业调度，也可用于进程调度。当在作业调度中采用该算法时，每次调度都是从后备作业队列中选择一个或多个最先进入该队列的作业，将它们调入内存，为它们分配资源、创建进程，然后放入就绪队列。在进程调度中采用FCFS算法时，则每次调度是从就绪队列中选择一个最先进入该队列的进程，为之分配处理机，使之投入运行。该进程一直运行到完成或发生某事件而阻塞后才放弃处理机，即具有不可剥夺性。FCFS算法简单，但是效率较低；对长作业比较有利，对短作业不利；有利于CPU密集型作业，不利于IO密集型作业。

##### 短作业优先(SJF)

短作业(进程)优先调度算法，是指对短作业或短进程优先调度的算法。它们可以分别用于作业调度和进程调度。短作业优先(SJF)的调度算法是从后备队列中选择一个或若干个估计运行时间最短的作业，将它们调入内存运行。而短进程优先(SPF)调度算法则是从就绪队列中选出一个估计运行时间最短的进程，将处理机分配给它，使它立即执行并一直执行到完成，或发生某事件而被阻塞放弃处理机时再重新调度，即具有不可剥夺性。短作业算法对于长作业不利，会产生饥饿现象；未考虑作业的紧急程度；难以实现真正意义上的短作业优先。

##### 优先权调度

为了照顾紧迫型作业，使之在进入系统后便获得优先处理，引入了最高优先权优先(FPF)调度算法。此算法常被用于批处理系统中，作为作业调度算法，也作为多种操作系统中的进程调度算法，还可用于实时系统中。当把该算法用于作业调度时，系统将从后备队列中选择若干个优先权最高的作业装入内存。当用于进程调度时，该算法是把处理机分配给就绪队列中优先权最高的进程，这时，又可进一步把该算法分成如下两种。

##### 非抢占式优先权算法

在这种方式下，系统一旦把处理机分配给就绪队列中优先权最高的进程后，该进程便一直执行下去，直至完成；或因发生某事件使该进程放弃处理机时，系统方可再将处理机重新分配给另一优先权最高的进程。这种调度算法主要用于批处理系统中；也可用于某些对实时性要求不严的实时系统中。

##### 抢占式优先权算法

在这种方式下，系统同样是把处理机分配给优先权最高的进程，使之执行。但在其执行期间，只要又出现了另一个其优先权更高的进程，进程调度程序就立即停止当前进程(原优先权最高的进程)的执行，重新将处理机分配给新到的优先权最高的进程。因此，在采用这种调度算法时，是每当系统中出现一个新的就绪进程i时，就将其优先权Pi与正在执行的进程j的优先权Pj进行比较。如果Pi≤Pj，原进程Pj便继续执行；但如果是Pi>Pj，则立即停止Pj的执行，做进程切换，使i进程投入执行。显然，这种抢占式的优先权调度算法能更好地满足紧迫。进程优先级可分为静态优先级和动态优先级。静态优先级在进程创建时就已经确定，且整个运行期间不发生改变。动态优先级在运行期间可变，主要依据为进程占用CPU时间的长短、就绪进程等待CPU时间的长短。

##### 时间片轮转法

在早期的时间片轮转法中，系统将所有的就绪进程按先来先服务的原则排成一个队列，每次调度时，把CPU分配给队首进程，并令其执行一个时间片。时间片的大小从几ms到几百ms。当执行的时间片用完时，由一个计时器发出时钟中断请求，调度程序便据此信号来停止该进程的执行，并将它送往就绪队列的末尾；然后，再把处理机分配给就绪队列中新的队首进程，同时也让它执行一个时间片。这样就可以保证就绪队列中的所有进程在一给定的时间内均能获得一时间片的处理机执行时间。换言之，系统能在给定的时间内响应所有用户的请求。

##### 多级反馈队列调度

前面介绍的各种用作进程调度的算法都有一定的局限性。如短进程优先的调度算法，仅照顾了短进程而忽略了长进程，而且如果并未指明进程的长度，则短进程优先和基于进程长度的抢占式调度算法都将无法使用。而多级反馈队列调度算法则不必事先知道各种进程所需的执行时间，而且还可以满足各种类型进程的需要，因而它是目前被公认的一种较好的进程调度算法。在采用多级反馈队列调度算法的系统中，调度算法的实施过程如下所述：

- 应设置多个就绪队列，并为各个队列赋予不同的优先级。第一个队列的优先级最高，第二个队列次之，其余各队列的优先权逐个降低。该算法赋予各个队列中进程执行时间片的大小也各不相同，在优先权愈高的队列中，为每个进程所规定的执行时间片就愈小。例如，第二个队列的时间片要比第一个队列的时间片长一倍，第i+1个队列的时间片要比第i个队列的时间片长一倍
- 当一个新进程进入内存后，首先将它放入第一队列的末尾，按FCFS原则排队等待调度。当轮到该进程执行时，如它能在该时间片内完成，便可准备撤离系统；如果它在一个时间片结束时尚未完成，调度程序便将该进程转入第二队列的末尾，再同样地按FCFS原则等待调度执行；如果它在第二队列中运行一个时间片后仍未完成，再依次将它放入第三队列，……，如此下去，当一个长作业(进程)从第一队列依次降到第n队列后，在第n队列便采取按时间片轮转的方式运行
- 仅当第一队列空闲时，调度程序才调度第二队列中的进程运行；仅当第1～(i-1)队列均空时，才会调度第i队列中的进程运行。如果处理机正在第i队列中为某进程服务时，又有新进程进入优先权较高的队列(第1～(i-1)中的任何一个队列)，则此时新进程将抢占正在运行进程的处理机，即第i队列中某个正在运行的进程的时间片用完后，由调度程序选择优先权较高的队列中的那一个进程，把处理机分配给它

#### 7.进程切换与线程切换

当操作系统决定要把控制权从当前进程转移到某个新进程时，就会进行上下文切换，即保存当前进程的上下文，恢复新进程的上下文，然后将控制权传递到新进程，新进程就会从上次停止的地方开始

内核为每一个进程维持一个上下文。**上下文就是内核重新启动一个被抢占的进程所需的状态。**包括一下内容：

- 通用目的寄存器
- 浮点寄存器
- 程序计数器
- 用户栈
- 状态寄存器
- 内核栈
- 各种内核数据结构：比如描绘地址空间的页表，包含有关当前进程信息的进程表，以及包含进程已打开文件的信息的文件表。

上下文是由程序正确运行所需的状态组成的，这个状态包括存放在存储器中的程序的代码和数据，他的栈，通用目的寄存器的内容，程序计数器，环境变量以及打开文件描述符的集合。

**每个进程都有自己的虚拟地址空间，进程内的所有线程共享进程的虚拟地址空间**。

进程切换和线程切换的区别

最主要的一个区别在于**进程切换涉及虚拟地址空间的切换而线程不会**。因为每个进程都有自己的虚拟地址空间，而线程是共享所在进程的虚拟地址空间的，因此同一个进程中的线程进行线程切换时不涉及虚拟地址空间的转换。

有的同学可能还是不太明白，为什么虚拟地址空间切换会比较耗时呢？

现在我们已经知道了进程都有自己的虚拟地址空间，把虚拟地址转换为物理地址需要查找页表，页表查找是一个很慢的过程，因此通常使用Cache来缓存常用的地址映射，这样可以加速页表查找，这个cache就是TLB（translation Lookaside Buffer，我们不需要关心这个名字只需要知道TLB本质上就是一个cache，是用来加速页表查找的）。由于每个进程都有自己的虚拟地址空间，那么显然每个进程都有自己的页表，那么**当进程切换后页表也要进行切换，页表切换后TLB就失效了**，cache失效导致命中率降低，那么虚拟地址转换为物理地址就会变慢，表现出来的就是程序运行会变慢，而线程切换则不会导致TLB失效，因为线程线程无需切换地址空间，因此我们通常说线程切换要比较进程切换块，原因就在这里。



### 3.用户态和内核态相关

#### 1.系统调用和程序调用

系统调用是操作系统内核和用户态运行程序之间的接口，它把用户程序的请求传送至内核，调用相应的内核函数完成所需的处理，将处理结果返回给用户程序。

系统中各种共享资源都由操作系统统一管理，因此在操作系统的外层软件或用户程序中，凡是与资源有关的操作（如存储分配、I/O等）都必须通过系统调用的方式向操作系统提出服务请求，并由操作系统代为完成，所以系统调用是用户程序获得操作系统服务的唯一途径。

---- 操作系统提供的系统调用很多，按功能分为：

进程和作业管理、***\*文件操作\****、***\*设备管理\****、***\*内存管理\****、***\*信息维护\****和***\*进程通信\****系统调用六大类。



#### 2.定义

在计算机系统中，通常运行着两类程序：系统程序和应用程序，为了保证系统程序不被应用程序有意或无意地破坏，为计算机设置了两种状态：

系统态(也称为管态或核心态)，操作系统在系统态运行——运行操作系统程序
用户态(也称为目态)，应用程序只能在用户态运行——运行用户程序
在实际运行过程中，处理机会在系统态和用户态间切换。相应地，现代多数操作系统将 CPU 的指令集分为特权指令和非特权指令两类。

1) 特权指令——在系统态时运行的指令
对内存空间的访问范围基本不受限制，不仅能访问用户存储空间，也能访问系统存储空间，
特权指令只允许操作系统使用，不允许应用程序使用，否则会引起系统混乱。

2) 非特权指令——在用户态时运行的指令
一般应用程序所使用的都是非特权指令，它只能完成一般性的操作和任务，不能对系统中的硬件和软件直接进行访问，其对内存的访问范围也局限于用户空间。

- **用户态**切换到**内核态**的**唯一**途径——>**中断/异常/陷入**
- **内核态**切换到**用户态**的途径——>设置程序状态字

**注意一条特殊的指令——陷入指令（又称为访管指令**，因为内核态也被称为管理态，访管就是访问管理态）该指令给用户提供接口，用于调用操作系统的服务。

#### 3.中断与缺页中断

为什么在操作系统中设计了中断的概念？为了提高并发执行的效率。

CPU中断正在运行的程序，转到处理中断事件程序

在请求分页系统中，可以通过查询页表中的状态位来确定所要访问的页面是否存在于内存中。每当所要访问的页面不在内存时，会产生一次缺页中断，此时操作系统会根据页表中的外存地址在外存中找到所缺的一页，将其调入内存。 
　　缺页本身是一种中断，与一般的中断一样，需要经过4个处理步骤： 
　　1. 保护CPU现场 
　　2. 分析中断原因 
　　3. 转入缺页中断处理程序进行处理 
　　4. 恢复CPU现场，继续执行 





### 4.Linux命令相关

#### 6.1查看隐藏文件

lsattr

#### 6.2查找文件里面的内容

grep

#### 6.3查看CPU使用情况

top

#### 6.4Linux环境下如何查找哪个线程使用CPU最长

（1）获取项目的pid，jps或者ps -ef | grep java，这个前面有讲过

（2）top -H -p pid，顺序不能改变

这样就可以打印出当前的项目，每条线程占用CPU时间的百分比。注意这里打出的是LWP，也就是操作系统原生线程的线程号，我笔记本山没有部署Linux环境下的Java工程，因此没有办法截图演示，网友朋友们如果公司是使用Linux环境部署项目的话，可以尝试一下。

使用”top -H -p pid”+”jps pid”可以很容易地找到某条占用CPU高的线程的线程堆栈，从而定位占用CPU高的原因，一般是因为不当的代码操作导致了死循环。

最后提一点，”top -H -p pid”打出来的LWP是十进制的，”jps pid”打出来的本地线程号是十六进制的，转换一下，就能定位到占用CPU高的线程的当前线程堆栈了。



#### 6.5linux终端按下Ctrl+C发生了什么

ctrl-c: ( kill foreground process ) 发送 SIGINT 信号给前台进程组中的所有进程，强制终止程序的执行；
ctrl-z: ( suspend foreground process ) 发送 SIGTSTP 信号给前台进程组中的所有进程，常用于挂起一个进程，而并 非结束进程，用户可以使用使用fg/bg操作恢复执行前台或后台的进程。

根据 `setpgrp` manual page 的说法，按下 `Ctrl-c` 后（主要是操作系统进程通信-信号机制）:

- 终端产生 `SIGINT` 信号
- 前台进程组中的所有进程都会接收到 `SIGINT` 信号然后退出(默认动作)
- shell通过调用 `waitpid` 清理进程表中子进程信息



#### 6.6. 显示最近登录的5个用户名

`BEGIN`和`END`可以确定开始执行的命令和结束执行的命令

```
awk 'BEGIN{print "aaa"} {print $1} file.txt'
awk '{print $1} END{print "bbb"} file.txt'
```

#### 6.7. 查询文件/文件夹下特定的字符串

```
grep 'string' fileName
grep 'string' dirPath/*
```

#### 6.8. 输出文本的1~5行

```
sed -n '1,5 p' test.txt
```

- -p：print打印
- -n：取消默认输出

#### 6.9Linux文件系统

- /home：每个账户对应的文件夹，用于管理自己的数据
- /usr：主要包括系统的主要程序、安装的软件、图形接口所需的文档、额外的函数库、共享目录与文件
- /bin：存放客户自行文件
- /boot：存放Linux开机会用的文件
- /dev：存放Linux的任何装置和接口设备文档
- /etc：存放系统设定文档，如账户密码
- /lib：系统使用的函数库放置的目录
- /mnt：软盘和光盘预设挂载点的位置
- /opt：主机额外安装软件所在的目录
- /root：系统管理员的home
- /sbin：只有系统管理员能使用的执行指令
- /tmp：临时存放文件的路径

#### 6.10查看线程/进程状态

- ps -T -p <pid> 可查看由进程号为<pid>的进程创建的所有线程</pid></pid>
- top -H 可实时显示各个线程情况
- pstree 以树状图显示进程

#### 6.11查看系统负载

- top

- uptime

  `# uptime``02``:``03``:``50` `up ``126` `days, ``12``:``57``, ``2` `users, load average: ``0.08``, ``0.03``, ``0.05``10``:``19``:``04` `up ``257` `days, ``18``:``56``, ``12` `users, load average: ``2.10``, ``2.10``,``2.09`

系统平均负载是指在特定时间间隔内运行队列中的平均进程数，如果你的 linux 主机是 1 个双核 CPU 的话，当 Load Average 为 6 的时候说明机器已经被充分使用了。1 可以被认为是最优的负载值。

#### 6.12统计文件行数的命令

wc命令 - c 统计字节数 - l 统计行数 - w 统计字数

- 统计某文件中，单词出现的行数：find ./linux.txt | xargs cat | grep "hello" | wc -l

#### 6.13统计文件数目的命令

- 6.统计文件夹下文件的个数：ls -l | grep “^-” | wc -l
- 统计文件夹下文件夹的个数：ls -l | grep “^d” | wc -l
- 统计文件夹下文件个数，包括子文件：ls -lR | grep "^-"| wc -l
- 统计文件夹下目录个数，包括子目录：ls -lR | grep "^d"| wc -l

#### 6.14查看资源的命令

- 显示所有文件及目录 (ls内定将文件名或目录名称开头为"."的视为隐藏档，不会列出)：ls -a
- 除文件名称外，亦将文件型态、权限、拥有者、文件大小等资讯详细列出：ls -l
- 将文件以相反次序显示(原定依英文字母次序)；ls -r
- 将文件依建立时间之先后次序列出：ls -t
- 递归列出文件夹中的文件：ls -R

#### 6.15杀掉某个后台进程

- ps -ef | grep firefox; kill -s 9 1827; 先查出Firefox的进程号，然后给进程发送信号
- ps -ef | grep firefox | grep -v grep | cut -c 9-15 | xargs kill -s 9; “grep firefox”的输出结果是，所有含有关键字“firefox”的进程。“grep -v grep”是在列出的进程中去除含有关键字“grep”的进程。“cut -c 9-15”是截取输入行的第9个字符到第15个字符，而这正好是进程号PID。“xargs kill -s 9”中的xargs命令是用来把前面命令的输出结果（PID）作为“kill -s 9”命令的参数，并执行该命令。“kill -s 9”会强行杀掉指定进程
- pkill -9 firefox
- killall -9 firefox
- kill -9 pid // 向指定pid的进程发送信号，终止进程

#### 6.16查看网络状况

- 查看linux的端口使用情况：netstat -tln
- 查看所有的服务端口：netstat -a
- 只显示监听端口：natstat -l

#### 6.17在/usr目录下找出大小超过10M的文件

```
find /usr -size +10M
```

#### 6.18在/home目录下找出120天前被修改过得文件

```
find /home -mtime +``120
```

#### 6.19在/var目录下找出90天内未访问过的文件

```
find /var \! -atime -``90
```

#### 6.20检查Java进程是否存在

```
ps -ef | grep java
```

#### 6.21常见的通配符

- ?代替任意单个字符
- *代替任意多个字符

#### 6.22查看系统支持的所有信号

```
kill -l
```

#### 6.23top时，进程对应的状态及表示

- D 不可中断 Uninterruptible（usually IO）
- R 正在运行，或在队列中的进程
- S 处于休眠状态
- T 停止或被追踪
- Z 僵尸进程
- W 进入内存交换（从内核 2.6 开始无效）
- X 死掉的进程

#### 6.24在后台运行命令

一般都是使用 & 在命令结尾来让程序自动运行。(命令后可以不追加空格)

#### 6.25grep命令

Linux 系统中 grep 命令是一种强大的文本搜索工具，它能使用正则表达式搜索文本，并把匹配的行打印出来。grep 全称是 Global Regular Expression Print，表示全局正则表达式版本，它的使用权限是所有用户。

#### 6.26文件系统

- 查看挂载的文件系统：cat /proc/filesystems
- 查看目录的使用情况：df -h //-h表示以G为单位查看
- 查看文件或目录大小：df -sh
- 查看文件夹的磁盘使用情况：du -a // -a表示全部目录下每个文件所占的磁盘空间
- 目录切换的命令：cd ~
- 打印当前路径：pwd
- 创建目录：mkdir
- 删除目录：rmdir
- 显示所有文件及目录 (ls内定将文件名或目录名称开头为"."的视为隐藏档，不会列出)：ls -a
- 除文件名称外，亦将文件型态、权限、拥有者、文件大小等资讯详细列出：ls -l
- 将文件以相反次序显示(原定依英文字母次序)；ls -r
- 将文件依建立时间之先后次序列出：ls -t
- 递归列出文件夹中的文件：ls -R
- 文件复制：cp from to
- 文件夹复制：cp from to -r
- 删除文件：rm -rf file
- 文件移动：mv file desdir
- 获取路径名：dirname
- ????显示文件内容： cat file -n // -n表示行号
- 显示文件的前n行或后n行：head -n; tail -n;
- 修改文件的时间(修改、访问、状态修改)：touch -d "10/13/2013" file
- 查找文件位置：whereis file / locate file / find file
- 比较两个文件的差异：diff -c file1 file2
- 以并列的方式显示两个文件的差异：diff -y file1 file2
- 建立文件的软连接：ln file1 file2
- 建立文件的硬链接：ln -s file1 file2
- 修改文件拥有者：chown -R path
- 修改权限：chmod {u|g|o|a}{+|-|=}{r|w|x} filename // {用户、同组用户、其他用户、所有用户}{增加权限、去除权限、增加所有权限}{读、写、执行}
- 统计文件字数：wc -c file // -l表示行数; -w表示单词数; -c表示字符数

#### 6.27系统

- 查看用户标识：id
- 查看当前登录的用户：users
- 查看都有哪些用户登录到机器上：who
- 显示当前终端的用户名：whoami
- !!!!显示进程： ps -ef // -a显示所有进程的信息、-e显示当前运行的每一个进程信息、-f显示一个完成列表、-x显示包括没有终端控制的进程状况
- 杀掉进程：kill -9 pid // 向指定pid的进程发送信号，终止进程
- 查看IP地址：ifconfig
- 查看路由表：netstat
- 查看历史输入的指令：history
- 查看后台任务 job -l

#### 6.28日期

- 显示日期：date
- 显示日历：cal 2019







### 5.虚拟地址物理地址

1、每个进程的4G内存空间只是虚拟内存空间，每次访问内存空间的某个地址，都需要把地址翻译成实际物理内存地址

2、所有进程共享同一物理内存，每个进程只需要把自己目前需要的虚拟内存映射并存储到物理内存上。

3、进程要知道哪些内存地址的数据在物理内存上，哪些不在，还有在物理内存上哪里，需要用页表记录

4、页表的每一项分两部分，第一部分记录此页是否在物理内存上，第二部分记录物理内存页的地址（如果在的话）

5、当进程访问每个虚拟地址，去看页表，如果发现对应的数据不在物理内存中，则缺页异常。

6、缺页异常的处理过程，就是把进程需要的数据从磁盘上拷贝到物理内存中，如果内存已经满了，没有空地方了，那就找一个页覆盖（页面置换算法）



### 6.段页式存储管理

分页

##### 3. 页式内存管理

将程序的逻辑地址空间划分为固定大小的页(page)，而物理内存划分为同样大小的页框(page frame)。程序加载时，可将任意一页放人内存中任意一个页框，这些页框不必连续，从而实现了离散分配。该方法需要CPU的硬件支持，来实现逻辑地址和物理地址之间的映射。

CPU中的内存管理单元(MMU)按逻辑页号通过查进程页表得到物理页框号，将物理页框号与页内地址相加形成物理地址。

上述过程通常由处理器的硬件直接完成，不需要软件参与。通常，操作系统只需在进程切换时，把进程页表的首地址装入处理器特定的寄存器中即可。一般来说，页表存储在主存之中。这样处理器每访问一个在内存中的操作数，就要访问两次内存：

- 第一次用来查找页表将操作数的 逻辑地址变换为物理地址；

- 第二次完成真正的读写操作。

  这样做时间上耗费严重。为缩短查找时间，可以将页表从内存装入CPU内部的关联存储器(例如，快表) 中，实现按内容查找。此时的地址变换过程是：在CPU给出有效地址后，由地址变换机构自动将页号送人快表，并将此页号与快表中的所有页号进行比较，而且这 种比较是同时进行的。若其中有与此相匹配的页号，表示要访问的页的页表项在快表中。于是可直接读出该页所对应的物理页号，这样就无需访问内存中的页表。由于关联存储器的访问速度比内存的访问速度快得多。

分段

分段地址中的逻辑地址为：`| 段号 | 段内地址 |`
每个段都是从0开始编址，并采用一段连续的地址空间，段长由相应的逻辑信息组的长度决定（**各段段长不同**）。每个段即包含了一部分地址空间，又标识了段与段之间的逻辑关系。

段表
分段存储管理系统为每个段分配一个连续的分区，进程的各个段可离散地装入内存的不同位置，用一张段映射表（段表）记录每段在内存的起始地址（基址）和段的长度。在配置了段表后，执行中的进程可通过逻辑地址中的段号来查询段表，找到段的对应内存区。
段表表项的结构为：| 段长 | 基址 |

分页与分段的主要区别:
➢段是信息的逻辑单位，它是根据用户的需要划分的，因此段对用户是可见的;页是信息的物理单位，是为了管理主存的方便而划分的，对用户是透明的。
➢页的大小固定不变，由系统决定。段的大小是不固定的，它由其完成的功能决定。
➢段式向用户提供的是二维地址空间，页式向用户提供的是一~维地址空间，其页号和页内偏移是机器硬件的功能。
➢由于段是信息的逻辑单位，因此便于存贮保护和信息的共享，页的保护和共享受到限制。
结合分段和分贝思想，先将用户程序分成若干段并分别赋予段名，再将这些段分为若干页
地址结构:由段号、段内页号和页内地址三项共同构成地址

<img src="https://img-blog.csdnimg.cn/20200512151658681.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2xvdzUyNTI=,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:30%;" />



### 7.5种网络IO模型

linux的五种IO模型，分别是：阻塞IO、非阻塞IO、多路复用IO、信号驱动IO以及异步IO。其中阻塞IO、非阻塞IO、多路复用IO、信号驱动IO都属于同步IO。

同步IO：应用程序主动向内核查询是否有可用数据，如果有自己负责把数据从内核copy到用户空间。

异步IO：应用程序向内核发起读数据请求需要：（1）告诉内核数据存放位置（2）注册回调函数，当内核完成数据copy后调用回调通知应用程序取数据。

同步IO/异步IO最大区别：同步IO数据从内核空间到用户空间的copy动作是由应用程序自己完成。而异步IO则是注册回调函数并告知内核用户空间缓冲区存放地址，数据copy由内核完成。

**阻塞IO**, 给女神发一条短信, 说我来找你了, 然后就默默的一直等着女神下楼, 这个期间除了等待你不会做其他事情, 属于备胎做法.
**非阻塞IO**, 给女神发短信, 如果不回, 接着再发, 一直发到女神下楼, 这个期间你除了发短信等待不会做其他事情, 属于专一做法.
**IO多路复用**, 是找一个宿管大妈来帮你监视下楼的女生, 这个期间你可以些其他的事情. 例如可以顺便看看其他妹子,玩玩王者荣耀, 上个厕所等等. IO复用又包括 select, poll, epoll 模式. 那么它们的区别是什么?

- 1） **select大妈** 每一个女生下楼, select大妈都不知道这个是不是你的女神, 她需要一个一个询问, 并且select大妈能力还有限, 最多一次帮你监视1024个妹子

- 2） **poll大妈**不限制盯着女生的数量, 只要是经过宿舍楼门口的女生, 都会帮你去问是不是你女神

- 3） **epoll大妈**不限制盯着女生的数量, 并且也不需要一个一个去问. 那么如何做呢? epoll大妈会为每个进宿舍楼的女生脸上贴上一个大字条,上面写上女生自己的名字, 只要女生下楼了, epoll大妈就知道这个是不是你女神了, 然后大妈再通知你。
  上面这些同步IO有一个共同点就是, 当女神走出宿舍门口的时候, 你已经站在宿舍门口等着女神的, 此时你属于**阻塞状态**

  接下来是**异步IO**的情况：
  你告诉女神我来了, 然后你就去打游戏了, 一直到女神下楼了, 发现找不见你了, 女神再给你打电话通知你, 说我下楼了, 你在哪呢? 这时候你才来到宿舍门口。 此时属于逆袭做法





### 8.select，poll，epoll

- select、poll，epoll本质上都是**同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的**。
- select时间复杂度O(n)，它仅仅知道了，有I/O事件发生了，却并不知道是哪几个流（可能有一个，多个，甚至全部），我们只能无差别轮询所有流，找出能读出数据，或者写入数据的流，对他们进行操作。所以**select具有O(n)的无差别轮询复杂度**，同时处理的流越多，无差别轮询时间就越长。
- poll（翻译：轮询）时间复杂度O(n)，poll本质上和select没有区别，它将用户传入的数组拷贝到内核空间，然后查询每个fd对应的设备状态， **但是它没有最大连接数的限制**，原因是它是基于链表来存储的.
- epoll时间复杂度O(1)，**epoll可以理解为event poll**，不同于忙轮询和无差别轮询，epoll会把哪个流发生了怎样的I/O事件通知我们。所以我们说epoll实际上是**事件驱动（每个事件关联上fd）**的，此时我们对这些流的操作都是有意义的。**（复杂度降低到了O(1)）。**  

#### select

首先创建事件的描述符集合。对于一个描述符，可以关注其上的读事件、写事件以及异常事件，所以要创建三类事件的描述符集合，分别用来收集读事件描述符、写事件描述符以及异常事件描述符。select 调用时，首先将时间描述符集合 fd_set 从用户空间拷贝到内核空间；注册回调函数并遍历所有 fd ，调用其 poll 方法， poll 方法返回时会返回一个描述读写操作是否就绪的 mask 掩码，根据这个掩码给 fd 赋值，如果遍历完所有 fd 后依旧没有一个可以读写就绪的 mask 掩码，则会使进程睡眠；如果已过超时时间还是未被唤醒，则调用 select 的进程会被唤醒并获得 CPU ，重新遍历 fd 判断是否有就绪的fd；最后将 fd_set从内核空间拷贝回用户空间。

select缺点：

- 每次调用 select ，都需要把 fd 集合从用户态拷贝到内核态，这个开销在 fd 很多时会很大
- 同时每次调用 select 都需要在内核遍历传递进来的所有 fd ，这个开销在 fd 很多时也很大
- select支持的文件描述符数量较小，默认是1024

#### poll

poll 是 select 的优化版。poll 使用 pollfd 结构而不是 select 的 fd_set 结构。select 需要为**读事件**、**写事件**和**异常事件**分别创建一个描述符集合，轮询时需要分别轮询这三个集合。而 poll 库只需要创建一个集合，在**每个描述符**对应的结构上分别设置读事件、写事件或者异常事件，最后轮询时可同时检查这三类事件是否发生。

#### epoll

select 与 poll 中，都创建一个待处理事件列表。然后把这个列表发送给内核，返回的时候再去轮询这个列表，以判断事件是否发生。在描述符比较多的时候，效率极低。epoll 将文件描述符列表的管理交给内核负责，每次注册新的事件时，将 fd 拷贝仅内核，epoll 保证 fd 在整个过程中仅被拷贝一次，避免了反复拷贝重复 fd 的巨大开销。此外，一旦某个事件发生时，内核就把发生事件的描述符列表通知进程，避免对所有描述符列表进行轮询。最后， epoll 没有文件描述符的限制，fd 上限是系统可以打开的最大文件数量，通常远远大于2048 。

#### select/poll/epoll对比

select，poll实现需要自己不断轮询所有 fd 集合，直到设备就绪，期间可能要睡眠和唤醒多次交替。而 epoll 其实也需要调用 epoll_wait 不断轮询**就绪链表**，期间也可能多次睡眠和唤醒交替。但是它在设备就绪时，调用回调函数，把就绪 fd 放入**就绪链表**中，并唤醒在 epoll_wait 中进入睡眠的进程。虽然都要睡眠和交替，但是 select 和 poll 在醒着的时候要遍历整个 fd 集合，而 epoll 在醒着的时候只要判断一下就绪链表是否为空就行了，这节省了大量的 CPU 时间。

#### select/poll/epoll各自的应用场景

- select：timeout 参数精度为 1ns，而 poll 和 epoll 为 1ms，因此 select 更加适用于实时性要求比较高的场景，比如核反应堆的控制。select 可移植性更好，几乎被所有主流平台所支持。
- poll：poll 没有最大描述符数量的限制，如果平台支持并且对实时性要求不高，应该使用 poll 而不是 select
- epoll：只运行在 Linux 平台上，有大量的描述符需要同时轮询，并且这些连接最好是长连接；需要同时监控小于 1000 个描述符，就没有必要使用 epoll，因为这个应用场景下并不能体现 epoll 的优势；需要监控的描述符状态变化多，而且都是非常短暂的，也没有必要使用 epoll。因为 epoll 中的所有描述符都存储在内核中，造成每次需要对描述符的状态改变都需要通过 epoll_ctl() 进行系统调用，频繁系统调用降低效率。并且 epoll 的描述符存储在内核，不容易调试



### 9.CPU利用率和负载

CPU 利用率显示的是程序在运行期间实时占用的 CPU 百分比；CPU 使用率反映的是当前 CPU 的繁忙程度，忽高忽低的原因在于占用 CPU 处理时间的进程可能处于 IO 等待状态但却还未释放进入 wait 。

CPU 负载是指某段时间内占用 CPU 时间的进程和等待 CPU 时间的进程数，这里等待 CPU 时间的进程是指等待被唤醒的进程，不包括处于 wait 状态进程。负载越小越好。

CPU 利用率高，并不意味着 CPU 的负载大。两者之间没有必然的关系。无论 CPU 的利用率是高是低，跟后面有多少任务在排队没有必然关系。负载表示的是“等待进程的平均数”。只有进程处于运行态（running）和不可中断状态（interruptible）才会被加入到负载等待进程中，也就是下面这两种情况的进程才会表现为负载的值：

- 即便需要立即使用 CPU，也还需等待其他进程用完 CPU
- 即便需要继续处理，也必须等待磁盘输入输出完成才能进行

CPU使用率低负载高

产生的原因就是：等待磁盘 I/O 完成的进程过多，导致进程队列长度过大，但是 CPU 运行的进程却很少，这样就体现到负载过大了，CPU 使用率低。常见场景如下：

- 磁盘读写请求过多就会导致大量 I/O 等待
- MySQL 中存在没有索引的语句或存在死锁等情况
- 外接硬盘故障，常见有挂了 NFS ，但是 NFS server 故障



## Kafka

### 1.为什么用kafka

首先他可以实现消息通知的功能，生产者消费者模式消息队列的优点：解耦，可恢复性，缓冲，峰值处理能力，异步通信机制其本身的优点：高吞吐量、消息持久化（存到硬盘，顺序读写保证性能）、高可靠性、高扩展性。



### 2.消息队列选型

1)在架构模型方面，

RabbitMQ遵循AMQP协议，RabbitMQ的broker由Exchange,Binding,queue组成，其中exchange和binding组成了消息的路由键；客户端Producer通过连接channel和server进行通信，Consumer从queue获取消息进行消费（长连接，queue有消息会推送到consumer端，consumer循环从输入流读取数据）。rabbitMQ以broker为中心；有消息的确认机制。

kafka遵从一般的MQ结构，producer，broker，consumer，以consumer为中心，消息的消费信息保存的客户端consumer上，consumer根据消费的点，从broker上批量pull数据；无消息确认机制。

2)在吞吐量，

rabbitMQ在吞吐量方面稍逊于kafka，他们的出发点不一样，rabbitMQ支持对消息的可靠的传递，支持事务，不支持批量的操作；基于存储的可靠性的要求存储可以采用内存或者硬盘。

kafka具有高的吞吐量，内部采用消息的批量处理，zero-copy机制，数据的存储和获取是本地磁盘顺序批量操作，具有O(1)的复杂度，消息处理的效率很高。

3)在可用性方面，

rabbitMQ支持miror的queue，主queue失效，miror queue接管。

kafka的broker支持主备模式。

4)在集群负载均衡方面，

rabbitMQ的负载均衡需要单独的loadbalancer进行支持。

kafka采用zookeeper对集群中的broker、consumer进行管理，可以注册topic到zookeeper上；通过zookeeper的协调机制，producer保存对应topic的broker信息，可以随机或者轮询发送到broker上；并且producer可以基于语义指定分片，消息发送到broker的某分片上。



### 3.Kafka怎么保证有序消费

在Kafka中Partition(分区)是真正保存消息的地方，发送的消息都存放在这里。Partition(分区)又存在于Topic（主题）中，并且一个Topic（主题）可以指定多个Partition(分区)。

在Kafka中，只保证Partition(分区)内有序，不保证Topic所有分区都是有序的。

**一、1个Topic（主题）只创建1个Partition(分区)，这样生产者的所有数据都发送到了一个Partition(分区)，保证了消息的消费顺序。**

**二、生产者在发送消息的时候指定要发送到哪个Partition(分区)。**

但是绝大多数用户都可以通过message key来定义，因为同一个key的message可以保证只发送到同一个partition，比如说key是user id，table row id等等，所以同一个user或者同一个record的消息永远只会发送到同一个partition上，保证了同一个user或record的顺序。当然，如果你有key skewness 就有些麻烦，需要特殊处理



### 4.介绍下 Kafka 的各个组件

1）Producer ：消息生产者，就是向 kafka broker 发消息的客户端；

2）Consumer ：消息消费者，向 kafka broker 取消息的客户端；

3）Consumer Group （CG）：消费者组，由多个 consumer 组成。消费者组内每个消费者负责消费不同分区的数据，一个分区只能由一个组内消费者消费；消费者组之间互不影响。所有的消费者都属于某个消费者组，即消费者组是逻辑上的一个订阅者。

4）Broker ：一台 kafka 服务器就是一个 broker。一个集群由多个 broker 组成。一个 broker可以容纳多个 topic。

5）Topic ：可以理解为一个队列，生产者和消费者面向的都是一个 topic；

6）Partition：为了实现扩展性，一个非常大的 topic 可以分布到多个 broker（即服务器）上，一个 topic 可以分为多个 partition，每个 partition 是一个有序的队列；

7）Replica：副本，为保证集群中的某个节点发生故障时，该节点上的 partition 数据不丢失，且 kafka 仍然能够继续工作，kafka 提供了副本机制，一个 topic 的每个分区都有若干个副本，一个 leader 和若干个 follower。

8）leader：每个分区多个副本的“主”，生产者发送数据的对象，以及消费者消费数据的对象都是 leader。

9）follower：每个分区多个副本中的“从”，实时从 leader 中同步数据，保持和 leader 数据的同步。leader 发生故障时，某个 follower 会成为新的 follower。



### 5.如何保证写入 Kafka 的数据不丢失

为保证 producer 发送的数据，能可靠的发送到指定的 topic，topic 的每个 partition 收到producer 发送的数据后，都需要向 producer 发送 ack（acknowledgement 确认收到），如果producer 收到 ack，就会进行下一轮的发送，否则重新发送数据。全部完成同步，才发送ack，选举新的 leader 时，容忍 n 台节点的故障，需要 n+1 个副本



### 6.什么是 ISR，为什么需要引入 ISR

Leader 维护了一个动态的 in-sync replica set (ISR)，意为和 leader 保持同步的 follower 集合。当 ISR 中的 follower 完成数据的同步之后，leader 就会给 follower 发送 ack。如果 follower长时间 未 向 leader 同 步 数 据 ， 则 该 follower 将 被 踢 出 ISR ， 该 时 间 阈 值 由**replica.lag.time.max.ms** 参数设定。Leader 发生故障之后，就会从 ISR 中选举新的 leader。

### 7.副本之间数据一致性利用HW与LEO实现

### 8.消费者怎么保证数据不丢失

在consumer消费阶段，对offset的处理，关系到是否丢失数据，是否重复消费数据，因此，我们把处理好offset就可以做到exactly-once && at-least-once(只消费一次)数据。

当[enable.auto.commit=true](http://enable.auto.commit%3dtrue/)时　　　　表示由kafka的consumer端自动提交offset,当你在pull(拉取)30条数据，在处理到第20条时自动提交了offset,但是在处理21条的时候出现了异常，当你再次pull数据时，由于之前是自动提交的offset，所以是从30条之后开始拉取数据，这也就意味着21-30条的数据发生了丢失。

当[enable.auto.commit=false](http://enable.auto.commit%3dfalse/)时　　　　由于上面的情况可知自动提交offset时，如果处理数据失败就会发生数据丢失的情况。那我们设置成手动提交。　　　　

当设置成false时，由于是手动提交的，可以处理一条提交一条，也可以处理一批，提交一批，由于consumer在消费数据时是按一个batch来的，当pull了30条数据时，如果我们处理一条，提交一个offset，这样会严重影响消费的能力，那就需要我们来按一批来处理，或者设置一个累加器，处理一条加1，如果在处理数据时发生了异常，那就把当前处理失败的offset进行提交(放在finally代码块中)注意一定要确保offset的正确性，当下次再次消费的时候就可以从提交的offset处进行再次消费由于 consumer 在消费过程中可能会出现断电宕机等故障，consumer 恢复后，需要从故障前的位置的继续消费，所以 consumer 需要实时记录自己消费到了哪个 offset，以便故障恢复后继续消费。

### 9.Kafka 为什么性能这么高

1）顺序写磁盘Kafka 的 producer 生产数据，要写入到 log 文件中，写的过程是一直追加到文件末端，为顺序写。官网有数据表明，同样的磁盘，顺序写能到 600M/s，而随机写只有 100K/s。这与磁盘的机械机构有关，顺序写之所以快，是因为其省去了大量磁头寻址的时间。2）零复制技术



### 10.零拷贝

零拷贝技术实现的方式通常有 2 种：mmap + writesendfilemmap + writemmap() 系统调用函数会直接把内核缓冲区里的数据「映射」到用户空间，这样，操作系统内核与用户空间就不需要再进行任何的数据拷贝操作。应用进程调用了 mmap() 后，DMA 会把磁盘的数据拷贝到内核的缓冲区里。接着，应用进程跟操作系统内核「共享」这个缓冲区；应用进程再调用 write()，操作系统直接将内核缓冲区的数据拷贝到 socket 缓冲区中，这一切都发生在内核态，由 CPU来搬运数据；最后，把内核的 socket 缓冲区里的数据，拷贝到网卡的缓冲区里，这个过程是由 DMA 搬运的。 我们可以得知，通过使用mmap() 来代替 read()， 可以减少一次数据拷贝的过程。但这还不是最理想的零拷贝，因为仍然需要通过 CPU 把内核缓冲区的数据拷贝到 socket 缓冲区里，而且仍然需要 4 次上下文切换，因为系统调用还是 2 次。



### 11.kafka事务

为了实现跨分区跨会话的事务，需要引入一个全局唯一的 Transaction ID，并将 Producer获得的PID 和Transaction ID 绑定。这样当Producer 重启后就可以通过正在进行的 TransactionID 获得原来的 PID。对于 Consumer 而言，事务的保证就会相对较弱，尤其时无法保证 Commit 的信息被精确消费。这是由于 Consumer 可以通过 offset 访问任意信息，而且不同的 Segment File 生命周期不同，同一事务的消息可能会出现重启后被删除的情况



### 12.消息发送过程

**两个线程——****main** **线程和** **Sender** **线程**，以及**一个线程共享变量——****RecordAccumulator**Consumer 消费数据时的可靠性是很容易保证的，因为数据在 Kafka 中是持久化的，故不用担心数据丢失问题



### 13.重复消费

开启幂等性的 Producer 在初始化的时候会被分配一个 PID，发往同一 Partition 的消息会附带 Sequence Number。而Broker 端会对<PID, Partition, SeqNumber>做缓存，当具有相同主键的消息提交时，Broker 只会持久化一条。



### 14.如何处理消息丢失

- 消息可靠性
  - 没有重复数据
  - 数据不会丢失

- 消息丢失的处理情况

  - 生产者将数据发送到 RabbitMQ 的时候，可能数据就在半路给搞丢了，因为网络问题啥的，都有可能。

    此时可以选择用 RabbitMQ 提供的事务功能，就是生产者**发送数据之前**开启 RabbitMQ 事务`channel.txSelect`，然后发送消息，如果消息没有成功被 RabbitMQ 接收到，那么生产者会收到异常报错，此时就可以回滚事务`channel.txRollback`，然后重试发送消息；如果收到了消息，那么可以提交事务`channel.txCommit`。

  - 就是 RabbitMQ 自己弄丢了数据，这个你必须**开启 RabbitMQ 的持久化**，就是消息写入之后会持久化到磁盘，哪怕是 RabbitMQ 自己挂了，**恢复之后会自动读取之前存储的数据**，一般数据不会丢。除非极其罕见的是，RabbitMQ 还没持久化，自己就挂了，**可能导致少量数据丢失**，但是这个概率较小。

  - RabbitMQ 如果丢失了数据，主要是因为你消费的时候，**刚消费到，还没处理，结果进程挂了**，比如重启了，那么就尴尬了，RabbitMQ 认为你都消费了，这数据就丢了。这个时候得用 RabbitMQ 提供的`ack`机制，简单来说，就是你关闭 RabbitMQ 的自动`ack`，可以通过一个 api 来调用就行，然后每次你自己代码里确保处理完的时候，再在程序里`ack`一把。这样的话，如果你还没处理完，不就没有`ack`？那 RabbitMQ 就认为你还没处理完，这个时候 RabbitMQ 会把这个消费分配给别的 consumer 去处理，消息是不会丢的。








## Elasticsearch

### 1.存储结构

ES 里的 Index 可以看做一个库，而 Types 相当于表，Documents 则相当于表的行。

### 2.分片&副本

1）允许你水平分割 / 扩展你的内容容量。 2）允许你在分片之上进行分布式的、并行的操作，进而提高性能/吞吐量

在分片/节点失败的情况下，提供了高可用性。因为这个原因，注意到复制分片从不与 原/主要（original/primary）分片置于同一节点上是非常重要的。 扩展你的搜索量/吞吐量，因为搜索可以在所有的副本上并行运行。

所以当你拥有越多的副本分片 时，也将拥有越高的吞吐量。

### 3.路由计算

routing 通过 hash 函数生成一个数字，然后这个数字再除以 number_of_primary_shards （主分片的数量）后得到余数 。这个分布在 0 到 number_of_primary_shards-1 之间的余数，就是我们所寻求 的文档所在分片的位置。

### 4.写流程

\1. 客户端向 Node 1 发送新建、索引或者删除请求。 2. 节点使用文档的 _id 确定文档属于分片 0 。请求会被转发到 Node 3，因为分片 0 的 主分片目前被分配在 Node 3 上。 3. Node 3 在主分片上面执行请求。如果成功了，它将请求并行转发到 Node 1 和 Node 2 的副本分片上。一旦所有的副本分片都报告成功, Node 3 将向协调节点报告成功，协调 节点向客户端报告成功。 在客户端收到成功响应时，文档变更已经在主分片和所有副本分片执行完成，变更是安全的。

### 5.读流程

\1. 客户端向 Node 1 发送获取请求。 2. 节点使用文档的 _id 来确定文档属于分片 0 。分片 0 的副本分片存在于所有的三个 节点上。 在这种情况下，它将请求转发到 Node 2 。 3. Node 2 将文档返回给 Node 1 ，然后将文档返回给客户端。 在处理读取请求时，协调结点在每次请求的时候都会通过轮询所有的副本分片来达到负载均 衡。在文档被检索时，已经被索引的文档可能已经存在于主分片上但是还没有复制到副本分 片。 在这种情况下，副本分片可能会报告文档不存在，但是主分片可能成功返回文档。 一 旦索引请求成功返回给用户，文档在主分片和副本分片都是可用的。

### 6.倒排索引

把文件 ID对应到关键词的映射转换为关键词到文件ID的映射，每个关键词都对应着一系列的文件， 这些文件中都出现这个关键词。分词和标准化的过程称为分析

### 7.存储

在 Elasticsearch 中，写入和打开一个新段的轻量的过程叫做 refresh 。 默认情况下每个分 片会每秒自动刷新一次。这就是为什么我们说 Elasticsearch 是 近 实时搜索: 文档的变化 并不是立即对搜索可见，但会在一秒之内变为可见。

Elasticsearch 增加了一个 translog ，或者叫事务日志，在每一次对 Elasticsearch 进行 操作时均进行了日志记录。

\1. 一个文档被索引之后，就会被添加到内存缓冲区，并且追加到了 translog

\2. 刷新（refresh）使分片每秒被刷新（refresh）一次：

  这些在内存缓冲区的文档被写入到一个新的段中，且没有进行 fsync 操作。

  这个段被打开，使其可被搜索

  内存缓冲区被清空

\3. 这个进程继续工作，更多的文档被添加到内存缓冲区和追加到事务日志

\4. 每隔一段时间—例如 translog 变得越来越大—索引被刷新（flush）；

一个新的 translog 被创建，并且一个全量提交被执行  所有在内存缓冲区的文档都被写入一个新的段。  缓冲区被清空。  一个提交点被写入硬盘。  文件系统缓存通过 fsync 被刷新（flush）。  老的 translog 被删除。

执行一个提交并且截断 translog 的行为在 Elasticsearch 被称作一次 flush 分片每 30 分钟被自动刷新（flush），或者在 translog 太大的时候也会刷新

### 8.为什么要使用 Elasticsearch? 

系统中的数据，随着业务的发展，时间的推移，将会非常多，而业务中往往采用模糊查询进行数据的 搜索，而模糊查询会导致查询引擎放弃索引，导致系统查询数据时都是全表扫描，在百万级别的数据库中， 查询效率是非常低下的，而我们使用 ES 做一个全文索引，将经常查询的系统功能的某些字段，比如说电 商系统的商品表中商品名，描述、价格还有 id 这些字段我们放入 ES 索引库里，可以提高查询速度。

### 9.Elasticsearch 的 master 选举流程？

  Elasticsearch 的选主是 ZenDiscovery 模块负责的，主要包含 Ping（节点之间通过这个 RPC 来发现彼此） 和 Unicast（单播模块包含一个主机列表以控制哪些节点需要 ping 通）这两部分  对所有可以成为 master 的节点（node.master: true）根据 nodeId 字典排序，每次选举每个节点都把自 己所知道节点排一次序，然后选出第一个（第 0 位）节点，暂且认为它是 master 节点。  如果对某个节点的投票数达到一定的值（可以成为 master 节点数 n/2+1）并且该节点自己也选举自己， 那这个节点就是 master。否则重新选举一直到满足上述条件。  master 节点的职责主要包括集群、节点和索引的管理，不负责文档级别的管理；data 节点可以关闭 http 功能。

### 10.Elasticsearch 集群脑裂问题？ 

“脑裂”问题可能的成因:  网络问题：集群间的网络延迟导致一些节点访问不到 master，认为 master 挂掉了从而选举出新的 master，并对 master 上的分片和副本标红，分配新的主分片  节点负载：主节点的角色既为 master 又为 data，访问量较大时可能会导致 ES 停止响应造成大面积延 迟，此时其他节点得不到主节点的响应认为主节点挂掉了，会重新选取主节点。  内存回收：data 节点上的 ES 进程占用的内存较大，引发 JVM 的大规模内存回收，造成 ES 进程失去 响应。 脑裂问题解决方案：  减少误判：discovery.zen.ping_timeout 节点状态的响应时间，默认为 3s，可以适当调大，如果 master 在该响应时间的范围内没有做出响应应答，判断该节点已经挂掉了。调大参数（如 6s， discovery.zen.ping_timeout:6），可适当减少误判。  选举触发: discovery.zen.minimum_master_nodes:1

该参数是用于控制选举行为发生的最小集群主节点数量。当备选主节点的个数大于等于该参数的值， 且备选主节点中有该参数个节点认为主节点挂了，进行选举。官方建议为（n/2）+1，n 为主节点个数 （即有资格成为主节点的节点个数）  角色分离：即 master 节点与 data 节点分离，限制角色 主节点配置为：node.master: true node.data: false 从节点配置为：node.master: false node.data: true

### 11.Elasticsearch 索引文档的流程？

协调节点默认使用文档 ID 参与计算（也支持通过 routing），以便为路由提供合适的分片： shard = hash(document_id) % (num_of_primary_shards)  当分片所在的节点接收到来自协调节点的请求后，会将请求写入到 Memory Buffer，然后定时（默认 是每隔 1 秒）写入到 Filesystem Cache，这个从 Memory Buffer 到 Filesystem Cache 的过程就叫做 refresh；  当然在某些情况下，存在 Momery Buffer 和 Filesystem Cache 的数据可能会丢失，ES 是通过 translog 的机制来保证数据的可靠性的。其实现机制是接收到请求后，同时也会写入到 translog 中，当 Filesystem cache 中的数据写入到磁盘中时，才会清除掉，这个过程叫做 flush；  在 flush 过程中，内存中的缓冲将被清除，内容被写入一个新段，段的 fsync 将创建一个新的提交点， 并将内容刷新到磁盘，旧的 translog 将被删除并开始一个新的 translog。  flush 触发的时机是定时触发（默认 30 分钟）或者 translog 变得太大（默认为 512M）时；

### 13.Elasticsearch 更新和删除文档的流程？

删除和更新也都是写操作，但是 Elasticsearch 中的文档是不可变的，因此不能被删除或者改动以展示 其变更；  磁盘上的每个段都有一个相应的.del 文件。当删除请求发送后，文档并没有真的被删除，而是在.del 文件中被标记为删除。该文档依然能匹配查询，但是会在结果中被过滤掉。当段合并时，在.del 文件中被标记为删除的文档将不会被写入新段。  在新的文档被创建时，Elasticsearch 会为该文档指定一个版本号，当执行更新时，旧版本的文档在.del 文件中被标记为删除，新版本的文档被索引到一个新段。旧版本的文档依然能匹配查询，但是会在结 果中被过滤掉。

### 14.Elasticsearch 搜索的流程？

搜索被执行成一个两阶段过程，我们称之为 Query Then Fetch；  在初始查询阶段时，查询会广播到索引中每一个分片拷贝（主分片或者副本分片）。 每个分片在本 地执行搜索并构建一个匹配文档的大小为 from + size 的优先队列。PS：在搜索的时候是会查询 Filesystem Cache 的，但是有部分数据还在 Memory Buffer，所以搜索是近实时的。  每个分片返回各自优先队列中 所有文档的 ID 和排序值 给协调节点，它合并这些值到自己的优先队 列中来产生一个全局排序后的结果列表。  接下来就是取回阶段，协调节点辨别出哪些文档需要被取回并向相关的分片提交多个 GET 请求。每 个分片加载并丰富文档，如果有需要的话，接着返回文档给协调节点。一旦所有的文档都被取回了， 协调节点返回结果给客户端。  Query Then Fetch 的搜索类型在文档相关性打分的时候参考的是本分片的数据，这样在文档数量较少 的时候可能不够准确，DFS Query Then Fetch 增加了一个预查询的处理，询问 Term 和 Document frequency，这个评分更准确，但是性能会变差。

## 项目/实习相关问题

### 1.抓取文档数据存入数据库表

学城开放API，传入ID，username，token即可获得（可涉及单点登录问题）

```
ContentResultDTO content = getContentFromKM(ssoConfig.getLoginName(), contentId, ssoId);
```

解析三张表

content.getData.getBody

Documenr doc = JSoup.parse(body)

XML解析Elements contents = doc.select(".ct-collapse-content");

定时刷新/点按钮刷新

### 2.单点登录

单点登录（Single Sign On），简称为 SSO，是比较流行的企业业务整合的解决方案之一。SSO的定义是在多个[应用](https://baike.baidu.com/item/应用/3994271)系统中，用户只需要登录一次就可以访问所有相互信任的应用系统。

sso登录以后，可以将Cookie的域设置为顶域，即.a.com，这样所有子域的系统都可以访问到顶域的Cookie。**我们在设置Cookie时，只能设置顶域和自己的域，不能设置其他的域。比如：我们不能在自己的系统中给baidu.com的域设置Cookie。**

Cookie的问题解决了，我们再来看看session的问题。我们在sso系统登录了，这时再访问app1，Cookie也带到了app1的服务端（Server），app1的服务端怎么找到这个Cookie对应的Session呢？这里就要把3个系统的Session共享，如图所示。共享Session的解决方案有很多，例如：Spring-Session。这样第2个问题也解决了。

1. 用户访问app系统，app系统是需要登录的，但用户现在没有登录。
2. 跳转到CAS server，即SSO登录系统，**以后图中的CAS Server我们统一叫做SSO系统。** SSO系统也没有登录，弹出用户登录页。
3. 用户填写用户名、密码，SSO系统进行认证后，将登录状态写入SSO的session，浏览器（Browser）中写入SSO域下的Cookie。
4. SSO系统登录完成后会生成一个ST（Service Ticket），然后跳转到app系统，同时将ST作为参数传递给app系统。
5. app系统拿到ST后，从后台向SSO发送请求，验证ST是否有效。
6. 验证通过后，app系统将登录状态写入session并设置app域下的Cookie。

至此，跨域单点登录就完成了。以后我们再访问app系统时，app就是登录的。接下来，我们再看看访问app2系统时的流程。

1. 用户访问app2系统，app2系统没有登录，跳转到SSO。
2. 由于SSO已经登录了，不需要重新登录认证。
3. SSO生成ST，浏览器跳转到app2系统，并将ST作为参数传递给app2。
4. app2拿到ST，后台访问SSO，验证ST是否有效。
5. 验证成功后，app2将登录状态写入session，并在app2域下写入Cookie。

使用基于 Token 的身份验证方法，在服务端不需要存储用户的登录记录。大概的流程是这样的：

　　　　a）、客户端使用用户名、密码请求登录。
　　　　b）、服务端收到请求，去验证用户名、密码。
　　　　c）、验证成功后，服务端会签发一个 Token（令牌），再把这个 Token 发送给客户端。
　　　　d）、客户端收到 Token 以后可以把它存储起来，比如放在 Cookie 里或者 Local Storage 、Session Storage里。
　　　　e）、客户端每次向服务端请求资源的时候需要带着服务端签发的 Token。
　　　　f）、服务端收到请求，然后去验证客户端请求里面带着的 Token，如果验证成功，就向客户端返回请求的数据。

　　a、JWT是一种紧凑（小而少，只要包含了用户信息即可）且自包含（json数据中包含本次访问简单记录，记录不敏感数据）的，用于在多方传递JSON对象的技术。传递的数据可以使用数字签名增加其安全行。可以使用HMAC加密算法或RSA公钥/私钥加密方式。
　　b、紧凑：数据小，可以通过URL，POST参数，请求头发送。且数据小代表传输速度快。
　　c、自包含：使用payload数据块记录用户必要且不隐私的数据，可以有效的减少数据库访问次数，提高代码性能。
　　d、JWT一般用于处理用户身份验证或数据信息交换。
　　e、用户身份验证：一旦用户登录，每个后续请求都将包含JWT，允许用户访问该令牌允许的路由，服务和资源。单点登录是当今广泛使用JWT的一项功能，因为它的开销
  很小，并且能够轻松地跨不同域使用。
　　f、数据信息交换：JWT是一种非常方便的多方传递数据的载体，因为其可以使用数据签名来保证数据的有效性和安全性。

A：header 头信息。

　　a）、数据结构： {"alg": "加密算法名称", "typ" : "JWT"}。
　　b）、alg是加密算法定义内容，如：HMAC SHA256 或 RSA。
　　c）、typ是token类型，这里固定为JWT。

B：payload （有效荷载，可以考虑不传递任何信息）。

　　a）、在payload数据块中一般用于记录实体（通常为用户信息）或其他数据的。主要分为三个部分，分别是：已注册信息（registered claims），公开数据（public claims），私有数据（private claims）。
　　b）、payload中常用信息有：iss（发行者），exp（到期时间），sub（主题），aud（受众）等。前面列举的都是已注册信息。
　　c）、公开数据部分一般都会在JWT注册表中增加定义。避免和已注册信息冲突。
　　d）、公开数据和私有数据可以由程序员任意定义。
　　e）、注意：即使JWT有签名加密机制，但是payload内容都是明文记录，除非记录的是加密数据，否则不排除泄露隐私数据的可能。不推荐在payload中记录任何敏感数据。

C：Signature 签名。

　　签名信息。这是一个由开发者提供的信息。是服务器验证的传递的数据是否有效安全的标准。在生成JWT最终数据的之前。先使用header中定义的加密算法，将header和payload进行加密，并使用点进行连接。如：加密后的head.加密后的payload。再使用相同的加密算法，对加密后的数据和签名信息进行加密。得到最终结果。

<img src="https://img2018.cnblogs.com/blog/1002211/201908/1002211-20190820221922876-1531253020.png" alt="img" style="zoom:50%;" />



### 3.用户user表怎么设计的

(#{username}, #{password}, #{salt}, #{email}, #{type}, #{status}, # 

{activationCode}, #{headerUrl}, #{createTime}) 



### 4.社区首页怎么显示帖子

\- 开发社区首页，显示前10个帖子

\- 开发分页组件，分页显示所有的帖子

先写好DiscussPost类进一步写Mapper接口->实现mapper即写SQL语句

id, user_id, title, content, type, status, create_time, comment_count, score

开发业务层：写两个service 完成mapper功能

开发视图层Controller

public String getIndexPage(Model model, Page page) { 

// 方法调用前,SpringMVC会自动实例化Model和Page,并将Page注入Model. 

// 所以,在thymeleaf中可以直接访问Page对象中的数据. 

page类需要两个参数，总行数与查询路径

```Java
//page的属性
// 当前页码
private int current = 1;
// 显示上限
private int limit = 10;
// 数据总数(用于计算总页数)
private int rows;
// 查询路径(用于复用分页链接)
private String path;
 /**
     * 获取当前页的起始行
     *
     * @return
     */
    public int getOffset() {
        // current * limit - limit
        return (current - 1) * limit;
    }

    /**
     * 获取总页数
     *
     * @return
     */
    public int getTotal() {
        // rows / limit [+1]
        if (rows % limit == 0) {
            return rows / limit;
        } else {
            return rows / limit + 1;
        }
    }

    /**
     * 获取起始页码
     *
     * @return
     */
    public int getFrom() {
        int from = current - 2;
        return from < 1 ? 1 : from;
    }

    /**
     * 获取结束页码
     *
     * @return
     */
    public int getTo() {
        int to = current + 2;
        int total = getTotal();
        return to > total ? total : to;
    }
```

返回model

修改下现有的index.html->//静态网址修改//动态值修改

### 5.项目记录日志

private static final Logger logger = LoggerFactory.getLogger(LoggerTests.class); 

将日志存入文件 写一个logback-spring.xml 



### 6.怎么发送邮件

• 邮箱设置

\- 启用客户端SMTP服务

• Spring Email- 导入 jar 包 

\- 邮箱参数配置

\- 使用 JavaMailSender 发送邮件 ：写一个工具类 mailclient类 



### 7.注册功能

• 提交注册数据

导工具包，commons lang

追加配置文件，域名

新建工具类：生成随机字符串，MD5加密

写UserService 注入邮件相关注解，注入域名及项目名

写注册方法，//返回值是注册相关的信息 ，// 注册用户，添加到库里 

在service中设计激活功能，和上文中测试邮件代码相同

\- 通过表单提交数据。

\- 服务端验证账号是否已存在、邮箱是否已注册。

\- 服务端发送激活邮件。

进一步开发controller，注入UserService

设计中间页面/site/operate-result

• 激活注册账号

\- 点击邮件中的链接，访问服务端的激活服务。

在service中加激活方法：先写一个常量接口包括激活的各种状态

在controller中处理激活请求



### 8.怎么生成验证码

• Kaptcha

\- 导入 jar 包

com.github.penggle

kaptcha

2.3.2

\- 编写 Kaptcha 配置类 ，kaptchaconfig

\- 生成随机字符、生成图片，在logincontroller中写一个方法



### 9.登录退出功能

• 登录

写实体类封装属性LoginTicket，(#{userId},#{ticket},#{status},#{expired})

int insertLoginTicket(LoginTicket loginTicket);

LoginTicket selectByTicket(String ticket); 

int updateStatus(String ticket, int status); 

\- 验证账号、密码、验证码。

在userservice里写功能，错误信息返回一个map

\- 成功时，生成登录凭证，发放给客户端。- 失败时，跳转回登录页。 

service中实现功能，这个表实现了session功能

在表现层中处理，在logincontroller中写

public String login(String username, String password, String code, boolean rememberme, Model model, HttpSession session, HttpServletResponse response) {

// 检查验证码 -// 检查账号,密码 -//表明登录成功了，重定向到首页 

• 退出

\- 将登录凭证修改为失效状态。

\- 跳转至网站首页。先写业务层service，在写表现层controller



### 10.登录与非登录显示不同页面

核心思想登录时拦截器拦截到将信息user存入hostholder，之后存入模板引擎，首页index可调用model中数据进而改变状态栏显示

• 拦截器示例

\- 定义拦截器，实现HandlerInterceptor

\- 配置拦截器，为它指定拦截、排除的路径

登录后浏览器持有cookie（ticket）访问服务器时，服务器将其ticket与自己数据库中数据比对可得到user，进而存放至model中，模板就可以得到相关信息

• 拦截器应用

\- 在请求开始时查询登录用户

从request中取cookie，进而得到ticket，封装成工具类cookieutil

拦截器中查ticket

\- 在本次请求中持有用户数据 ，就是保存user，考虑多线程情况，线程隔离，编写工具类hostholder将user存在threadlocal中，每个线程存入一个map，以线程为key

拦截器中持有用户

\- 在模板视图上显示用户数据

先将用户存入模板引擎

### 11.用户设置头像，修改密码功能

• 上传文件

\- 请求：必须是POST请求

\- 表单：enctype=“multipart/form-data”

\- Spring MVC：通过 MultipartFile 处理上传文件

\- 上传头像

修改配置文件，图片存放位置 community.path.upload=d:/work/data/upload

业务层代码：更新用户的headurl，userservice添加功能

表现层代码usercontroller实现上传功能



### 12.检查登录

未登录时不能访问设置页面等

• 使用拦截器

\- 在方法前标注自定义注解

\- 拦截所有请求，只处理带有该注解的方法

• 自定义注解

\- 常用的元注解：

@Target（作用在什么位置）、@Retention（保留时间）、@Document、@Inherited

\- 如何读取注解：

Method.getDeclaredAnnotations()

Method.getAnnotation(Class annotationClass)

写注解接口LoginRequired

在usercontroller中 setting方法以及upload方法上加注解

写注解拦截器LoginRequiredInterceptor

### 13.发帖过滤敏感词

• 前缀树

\- 名称：Trie、字典树、查找树

\- 特点：查找效率高，消耗内存大

\- 应用：字符串检索、词频统计、字符串排序等

• 敏感词过滤器

\- 定义前缀树

\- 根据敏感词，初始化前缀树

\- 编写过滤敏感词的方法



### 14.怎么发帖子

数据访问层：首先先在discusspostmapper与其相应的xml文件中增加 新增帖子的语句

业务层：在discusspostservice中增加方法

视图层：新建DiscussPostController

视图层：处理首页HTML与index.js



### 15.点击查看帖子详情

数据访问层• DiscussPostMapper-DiscussPost selectDiscussPostById(int id); 

业务层• DiscussPostService-public DiscussPost findDiscussPostById(int id) { 

视图层• DiscussPostController-public String getDiscussPost(@PathVariable("discussPostId") int discussPostId, Model model) {

• index.html-在帖子标题上增加访问详情页面的链接

• discuss-detail.htm\- 处理静态资源的访问路径



### 16.怎么显示评论

• 数据层-根据实体查询一页评论数据。-根据实体查询评论的数量。实体类：comment.java

List<Comment> selectCommentsByEntity(int entityType, int entityId, int offset, int limit); 

int selectCountByEntity(int entityType, int entityId);

• 业务层，处理查询评论的业务，处理查询评论数量的业务。CommentService

public List<Comment> findCommentsByEntity(int entityType, int entityId, int offset, int limit) { 

public int findCommentCount(int entityType, int entityId) { 

• 表现层\- 显示帖子详情数据时，同时显示该帖子所有的评论数据。修改DiscussPostController的getDiscussPost方法

public String getDiscussPost(@PathVariable("discussPostId") int discussPostId, Model model, Page page) { 

// 评论: 给帖子的评论 

// 回复: 给评论的评论 



### 17.怎么添加评论

• 数据层

\- 增加评论数据。

\- 修改帖子的评论数量。commentmapper中增加方法int insertComment(Comment comment);

在discusspostmapper中增加方法int updateCommentCount(int id,int commentCount);

• 业务层

\- 处理添加评论的业务：

先增加评论、再更新帖子的评论数量。DiscussPostService中增加更新数量功能，public int updateCommentCount(int id, int commentCount) { 

CommentService中增加添加评论功能public int addComment(Comment comment) {

• 表现层

\- 处理添加评论数据的请求。

- 设置添加评论的表单。CommentController
- public String addComment(@PathVariable("discussPostId") int discussPostId, Comment comment) {



### 18.怎么实现查看私信功能

• 私信列表

\- 查询当前用户的会话列表，

每个会话只显示一条最新的私信。

\- 支持分页显示。

数据访问层：实体类Message

业务层MessageService

public List<Message> findConversations(int userId, int offset, int limit) { 

public int findConversationCount(int userId) { 

public List<Message> findLetters(String conversationId, int offset, int limit) { 

public int findLetterCount(String conversationId) { 

public int findLetterUnreadCount(int userId, String conversationId) { 

表现层MessageController

public String getLetterList(Model model, Page page) { 

• 私信详情

\- 查询某个会话所包含的私信。

\- 支持分页显示。

MessageController中增加详情功能

public String getLetterDetail(@PathVariable("conversationId") String conversationId, Page page, Model model) { 



### 19.怎么实现发送私信功能

• 发送私信

\- 采用异步的方式发送私信。

\- 发送成功后刷新私信列表。

数据访问层mapper及SQL，int insertMessage(Message message); int updateStatus(List<Integer>ids, int status); 

业务层MessageService

表现层MessageController，public String sendLetter(String toName, String content) { 

• 设置已读

\- 访问私信详情时，

将显示的私信设置为已读状态。

表现层MessageController，private List<Integer> getLetterIds(List<Message> letterList) { 



### 20.项目中的异常怎么处理的

主要针对表现层，springboot中可以把错误页面在特定位置放好

将error放在templates文件夹下

修改404/500页面的静态资源

• @ControllerAdvice

\- 用于修饰类，表示该类是Controller的全局配置类。

\- 在此类中，可以对Controller进行如下三种全局配置：

异常处理方案、绑定数据方案、绑定参数方案。

• @ExceptionHandler

\- 用于修饰方法，该方法会在Controller出现异常后被调用，用于处理捕获到的异常。

• @ModelAttribute

\- 用于修饰方法，该方法会在Controller方法执行前被调用，用于为Model对象绑定参数。

• @DataBinder

\- 用于修饰方法，该方法会在Controller方法执行前被调用，用于绑定参数的转换器。

新建ExceptionAdvice类，可以囊括一切controller



### 21.关于记录日志

AOP，@Before("pointcut()") 

public void before(JoinPoint joinPoint) { 

// 用户[1.2.3.4],在[xxx],访问了[com.nowcoder.community.service.xxx()]. 

//通过request获取ip，先获得request对象 



### 22.点赞的实现

• 点赞

\- 支持对帖子、评论点赞。

\- 第1次点赞，第2次取消点赞。

先写key的工具类RedisKeyUtil，// 某个实体的赞 

// like:entity:entityType:entityId -> set(userId) 

public static String getEntityLikeKey(int entityType, int entityId) { 

业务层：LikeService

public void like(int userId, int entityType, int entityId) { 

public long findEntityLikeCount(int entityType, int entityId) { 

public int findEntityLikeStatus(int userId, int entityType, int entityId) { 

表现层：LikeController，public String like(int entityType, int entityId) { 

• 首页点赞数量

\- 统计帖子的点赞数量。 HomeController

 详情页点赞数量

\- 统计点赞数量。

\- 显示点赞状态。DiscussPostController 分为帖子/评论/回复



### 23.怎么在个人主页看到我收到的赞

• 重构点赞功能

\- 以用户为key，记录点赞数量

\- increment(key)，decrement(key)

在Rediskey工具类中增加方法

业务层：重构likeservice以保持事务，同时实现记录某人获得的赞

表现层：在homecontroller点赞方法中增加参数entityUserId

• 开发个人主页

\- 以用户为key，查询点赞数量

在Usercontroller中增加方法



### 24.关注与取消关注功能实现

数据访问层：还是用Redis存储，那就还要设置key

// 某个用户关注的实体 

// followee:userId:entityType -> zset(entityId,now) 

public static String getFolloweeKey(int userId, int entityType) { 

// 某个实体拥有的粉丝 

// follower:entityType:entityId -> zset(userId,now) 

public static String getFollowerKey(int entityType, int entityId) { 

业务层followservice

表现层FollowController 异步请求

\- 统计用户的关注数、粉丝数。

业务层：FollowService

// 查询关注的实体的数量 zCard

// 查询当前用户是否已关注该实体 score



### 25.查看自己的关注列表与粉丝列表

• 业务层FollowService 逻辑类似，key不同而已

\- 查询某个用户关注的人，支持分页。

Set<Integer> targetIds = redisTemplate.opsForZSet().reverseRange(followeeKey, offset, offset + limit - 1); 

\- 查询某个用户的粉丝，支持分页。

Set<Integer> targetIds = redisTemplate.opsForZSet().reverseRange(followerKey, offset, offset + limit - 1); 

• 表现层FollowController

\- 处理“查询关注的人”、“查询粉丝”请求。

\- 编写“查询关注的人”、“查询粉丝”模板。



### 26.利用Redis对项目中部分功能做优化

**• 使用Redis存储验证码**

\- 验证码需要频繁的访问与刷新，对性能要求较高。

\- 验证码不需永久保存，通常在很短的时间后就会失效。

\- 分布式部署时，存在Session共享的问题。

还是定义key，// 登录验证码，传一个临时凭证owner 

String kaptchaOwner = CommunityUtil.generateUUID(); 

Cookie cookie = new Cookie("kaptchaOwner", kaptchaOwner); 

String redisKey = RedisKeyUtil.getKaptchaKey(kaptchaOwner); 

redisTemplate.opsForValue().set(redisKey, text, 60, TimeUnit.SECONDS);

//得到验证码从cookie中取kaptchaOwner进而从Redis中取kaptcha

**• 使用Redis存储登录凭证** 

\- 处理每次请求时，都要查询用户的登录凭证，访问的频率非常高。

还是定义key，public static String getTicketKey(String ticket) { 

修改UserService中的登录功能中的存凭证

redisTemplate.opsForValue().set(redisKey, loginTicket);

**• 使用Redis缓存用户信息**

\- 处理每次请求时，都要根据凭证查询用户信息，访问的频率非常高。

还是定义key，public static String getUserKey(int userId) { 

// 1.优先从缓存中取值 ，// 2.取不到时初始化缓存数据，// 3.数据变更时清除缓存数据 



### 27.怎么实现发送系统通知

• 触发事件

\- 评论后，发布通知

\- 点赞后，发布通知

\- 关注后，发布通知

• 处理事件

\- 封装事件对象

\- 开发事件的生产者

\- 开发事件的消费者

数据访问层：新建event实体类，不过修改下set方法，这样可以set（）.get（）

创建生产者消费者类：

EventProducer，

```Java
// 处理事件 

public void fireEvent(Event event) { 

// 将事件发布到指定的主题 

kafkaTemplate.send(event.getTopic(), JSONObject.toJSONString(event)); 

} 
```

EventConsumer，//主题令为三个事件 

@KafkaListener(topics = {TOPIC_COMMENT, TOPIC_LIKE, TOPIC_FOLLOW}) 

public void handleCommentMessage(ConsumerRecord record) { 

修改已有的controller

CommentController：，LikeController，FollowController



### 28.显示消息通知

• 通知列表

\- 显示评论、点赞、关注三种类型的通知

• 未读消息

\- 在页面头部显示所有的未读消息数量

首先修改messagemapper，

// 查询某个主题下最新的通知 

Message selectLatestNotice(int userId, String topic); 

// 查询某个主题所包含的通知数量 

int selectNoticeCount(int userId, String topic); 

// 查询未读的通知的数量 

int selectNoticeUnreadCount(int userId, String topic);

业务层：MessageService

表现层：MessageController

最后处理头部未处理消息数量

拦截器MessageInterceptor



### 29.实现搜索功能

• 搜索服务

\- 将帖子保存至Elasticsearch服务器。

\- 从Elasticsearch服务器删除帖子。

\- 从Elasticsearch服务器搜索帖子。

业务层：ElasticsearchService ，public Page<DiscussPost> searchDiscussPost(String keyword, int current, int limit) {

• 发布事件

\- 发布帖子时，将帖子异步的提交到Elasticsearch服务器。

\- 增加评论时，将帖子异步的提交到Elasticsearch服务器。

\- 在消费组件中增加一个方法，消费帖子发布事件。

表现层：DiscussPostController增加触发发帖事件，CommentController中触发发帖事件

消费事件：EventConsumer增加消费发帖事件

• 显示结果

\- 在控制器中处理搜索请求，在HTML上显示搜索结果。

表现层SearchController： 

### 30.权限管理功能

授权配置：对当前系统内包含的所有请求，分配访问权限

写SecurityConfig配置类，protected void configure(HttpSecurity http) throws Exception { 



### 31.置顶加精删除等功能

在Dispostmapper和相应的SQL中增加方法，业务层增加方法，表现层，三种功能逻辑类似，在eventconsumer中增加消化删帖功能

权限管理：版主可以执行置顶、加精操作，管理员可以删除，修改配置类



### 32.统计网站数据，如日活

HyperLogLog：采用一种技术算法，用于完成独立总数的统计，占用空间好，无论统计多少个数据，只占12K内存空间，不精确的统计算法，标准误差为0.81%

Bitmap：不是一种独立的数据结构，实际上就是字符串，支持按位存取数据，可以将其看成byte数组，适合存储大量的连续的布尔值

UV：独立访客。须通过用户IP去重统计数据，每次访问都要进行统计，HyperLogLog，性能好，且存储空间小

DAU：日活跃用户，需通过用户ID去重统计数据，访问过一次则认为其活跃，Bitmap，性能好，且可以统计精确地结果

key用时间/时间范围

业务层DataService，uv存ip dau存userid

拦截器DataInterceptor 来统计UV/DAU

表现层：DataController，权限管理



### 33.热帖排行的实现

![image-20210807105202071](C:\Users\TianYu\AppData\Roaming\Typora\typora-user-images\image-20210807105202071.png)

想存到Redis 先定义key，修改各种controller，DiscussPostController 在发帖和加精时触发 CommentController评论时 触发，LikeController点暂时触发

redisTemplate.opsForSet().add(redisKey, post.getId());

定时更新需要quartz 新建PostScoreRefreshJob类

定时任务通过判断redis中有无元素来执行刷新任务，刷新任务需要计算分数，更新分数，同步搜索数据

网页模板想显示按热度/按时间，先重构帖子展示方法增加按分数排序，List<DiscussPost> selectDiscussPosts(int userId, int offset, int limit, int orderMode); 



### 34.性能优化相关问题

本地缓存：将数据存在应用服务器上，性能最好，常用的缓存工具：Ehacache、Guava、Caffenine等

分布式缓存：将数据缓存在Nosql数据库上，跨服务器，工具：MemCache、Redis等

多级缓存：一级缓存（本地缓存空间小）>二级缓存（分布式缓存）>DB

避免缓存雪崩（缓存失效，大量请求直达DB），提高系统可用性

DiscussPostService增加功能，// Caffeine核心接口: Cache, LoadingCache, AsyncLoadingCache

// 帖子列表缓存 

private LoadingCache<String, List<DiscussPost>> postListCache; 

// 帖子总数缓存 

private LoadingCache<Integer, Integer> postRowsCache;







## 智力题

### 1.假设现在我们有一个包含 10 亿个搜索关键词的日志文件，如何能快速获取到热门榜 Top 10 的搜索关键词呢？

如果我们将处理的场景限定为单机，可以使用的内存为 1GB。那这个问题该如何解决呢？

因为用户搜索的关键词，有很多可能都是重复的，所以我们首先要统计每个搜索关键词出现的频率。我们可以通过散列表、平衡二叉查找树或者其他一些支持快速查找、插入的数据结构，来记录关键词及其出现的次数。

然后，我们再根据前面讲的用堆求 Top K 的方法，建立一个大小为 10 的小顶堆，遍历散列表，依次取出每个搜索关键词及对应出现的次数，然后与堆顶的搜索关键词对比。如果出现次数比堆顶搜索关键词的次数多，那就删除堆顶的关键词，将这个出现次数更多的关键词加入到堆中。

可以根据哈希算法的这个特点，将 10 亿条搜索关键词先通过哈希算法分片到 10 个文件中。

具体可以这样做：我们创建 10 个空文件 00，01，02，……，09。我们遍历这 10 亿个关键词，并且通过某个哈希算法对其求哈希值，然后哈希值同 10 取模，得到的结果就是这个搜索关键词应该被分到的文件编号。对这 10 亿个关键词分片之后，每个文件都只有 1 亿的关键词，去除掉重复的，可能就只有 1000 万个，每个关键词平均 50 个字节，所以总的大小就是 500MB。1GB 的内存完全可以放得下。

我们针对每个包含 1 亿条搜索关键词的文件，利用散列表和堆，分别求出 Top 10，然后把这个 10 个 Top 10 放在一块，然后取这 100 个关键词中，出现次数最多的 10 个关键词，这就是这 10 亿数据中的 Top 10 最频繁的搜索关键词了。

### 2.假设我们有 100 个小文件，每个文件的大小是 100MB，每个文件中存储的都是有序的字符串。我们希望将这些 100 个小文件合并成一个有序的大文件。

我们将从小文件中取出来的字符串放入到小顶堆中，那堆顶的元素，也就是优先级队列队首的元素，就是最小的字符串。我们将这个字符串放入到大文件中，并将其从堆中删除。然后再从小文件中取出下一个字符串，放入到堆中。循环这个过程，就可以将 100 个小文件中的数据依次放入到大文件中。



### 3.top K问题，eg：有1亿个浮点数，如果找出期中最大的10000个？

最容易想到的方法是将数据全部排序，然后在排序后的集合中进行查找，最快的排序算法的时间复杂度一般为O（nlogn），如快速排序。但是在32位的机器上，每个float类型占4个字节，1亿个浮点数就要占用400MB的存储空间，对于一些可用内存小于400M的计算机而言，很显然是不能一次将全部数据读入内存进行排序的。其实即使内存能够满足要求（我机器内存都是8GB），该方法也并不高效，因为题目的目的是寻找出最大的10000个数即可，而排序却是将所有的元素都排序了，做了很多的无用功。

​    第二种方法为局部淘汰法，该方法与排序方法类似，用一个容器保存前10000个数，然后将剩余的所有数字——与容器内的最小数字相比，如果所有后续的元素都比容器内的10000个数还小，那么容器内这个10000个数就是最大10000个数。如果某一后续元素比容器内最小数字大，则删掉容器内最小元素，并将该元素插入容器，最后遍历完这1亿个数，得到的结果容器中保存的数即为最终结果了。此时的时间复杂度为O（n+m^2），其中m为容器的大小，即10000。

​    第三种方法是分治法，将1亿个数据分成100份，每份100万个数据，找到每份数据中最大的10000个，最后在剩下的100*10000个数据里面找出最大的10000个。如果100万数据选择足够理想，那么可以过滤掉1亿数据里面99%的数据。100万个数据里面查找最大的10000个数据的方法如下：用快速排序的方法，将数据分为2堆，如果大的那堆个数N大于10000个，继续对大堆快速排序一次分成2堆，如果大的那堆个数N大于10000个，继续对大堆快速排序一次分成2堆，如果大堆个数N小于10000个，就在小的那堆里面快速排序一次，找第10000-n大的数字；递归以上过程，就可以找到第1w大的数。参考上面的找出第1w大数字，就可以类似的方法找到前10000大数字了。此种方法需要每次的内存空间为10^6*4=4MB，一共需要101次这样的比较。

​    第四种方法是Hash法。如果这1亿个书里面有很多重复的数，先通过Hash法，把这1亿个数字去重复，这样如果重复率很高的话，会减少很大的内存用量，从而缩小运算空间，然后通过分治法或最小堆法查找最大的10000个数。

​    第五种方法采用最小堆。首先读入前10000个数来创建大小为10000的最小堆，建堆的时间复杂度为O（mlogm）（m为数组的大小即为10000），然后遍历后续的数字，并于堆顶（最小）数字进行比较。如果比最小的数小，则继续读取后续数字；如果比堆顶数字大，则替换堆顶元素并重新调整堆为最小堆。整个过程直至1亿个数全部遍历完为止。然后按照中序遍历的方式输出当前堆中的所有10000个数字。该算法的时间复杂度为O（nmlogm），空间复杂度是10000（常数）。

（1）单机+单核+足够大内存
        如果需要查找10亿个查询次（每个占8B）中出现频率最高的10个，考虑到每个查询词占8B，则10亿个查询次所需的内存大约是10^9 * 8B=8GB内存。如果有这么大内存，直接在内存中对查询次进行排序，顺序遍历找出10个出现频率最大的即可。这种方法简单快速，使用。然后，也可以先用HashMap求出每个词出现的频率，然后求出频率最大的10个词。

（2）单机+多核+足够大内存
        这时可以直接在内存总使用Hash方法将数据划分成n个partition，每个partition交给一个线程处理，线程的处理逻辑同（1）类似，最后一个线程将结果归并。

​    该方法存在一个瓶颈会明显影响效率，即数据倾斜。每个线程的处理速度可能不同，快的线程需要等待慢的线程，最终的处理速度取决于慢的线程。而针对此问题，解决的方法是，将数据划分成c×n个partition（c>1），每个线程处理完当前partition后主动取下一个partition继续处理，知道所有数据处理完毕，最后由一个线程进行归并。

（3）单机+单核+受限内存
        这种情况下，需要将原数据文件切割成一个一个小文件，如次啊用hash(x)%M，将原文件中的数据切割成M小文件，如果小文件仍大于内存大小，继续采用Hash的方法对数据文件进行分割，知道每个小文件小于内存大小，这样每个文件可放到内存中处理。采用（1）的方法依次处理每个小文件。

（4）多机+受限内存
        这种情况，为了合理利用多台机器的资源，可将数据分发到多台机器上，每台机器采用（3）中的策略解决本地的数据。可采用hash+socket方法进行数据分发。



### 4.**海量数据的解决方案**：

1. 使用缓存。
2. 页面静态化技术。
3. 数据库优化。
4. 分类数据库中活跃的数据。
5. 批量读取和延迟修改。
6. 读写分离。
7. 适应nosql和Hadoop技术。
8. 分布式部署数据库。
9. 应用服务和数据服务分离。
10. 使用搜索引擎搜索数据库中的数据。
11. 进行业务拆分。

### 5.1TB的数据需要排序，限定使用32GB的内存如何处理？

例如，考虑一个 1G 文件，只可用内存 100M 的排序方法。首先将文件分成 10 个 100M ，并依次载入内存中进行排序，最后结果存入硬盘。得到的是 10 个分别排序的文件。接着从每个文件载入 9M 的数据到输入缓存区，输出缓存区大小为 10M 。对输入缓存区的数据进行归并排序，输出缓存区写满之后写在硬盘上，缓存区清空继续写接下来的数据。对于输入缓存区，当一个块的 9M 数据全部使用完，载入该块接下来的 9M 数据，一直到所有的 9 个块的所有数据都已经被载入到内存中被处理过。最后我们得到的是一个 1G 的排序好的存在硬盘上的文件。

解法：

- 把磁盘上的 1TB 数据分割为 40 块，每份 25GB。注意，要留一些系统空间！
- 顺序将每份 25GB 数据读入内存，使用 quick sort 算法排序。
- 把排序好的数据存放回磁盘。
- 循环 40 次，现在，所有的 40 个块都已经各自排序了。
- 从 40 个块中分别读取 25G/40 = 0.625G入内存
- 执行 40 路合并，并将合并结果临时存储于2GB 基于内存的输出缓冲区中。当缓冲区写满 2GB 时，写入硬盘上最终文件，并清空输出缓冲区当
- 40 个输入缓冲区中任何一个处理完毕时，写入该缓冲区所对应的块中的下一个 0.625GB ，直到全部处理完成。

继续优化：

- 使用并发：如多磁盘（并发I/O提高）、多线程、使用异步 I/O 、使用多台主机集群计算
- 提升硬件性能：如更大内存、更高 RPM 的磁盘、升级为 SSD 、 Flash 、使用更多核的 CPU
- 提高软件性能：比如采用 radix sort 、压缩文件（提高I/O效率）等



### 6.海量日志数据，提取出某日访问百度次数最多的那个IP

解法：
首先过滤日志文件，将某日的 ip 过滤出来，保存在一个大文件内。然后对这些 ip 求 Hash 值，得到 Hash 值后再对 1000 取模，然后将计算后的值作为该 ip 写入的目标文件下标。上述操作中，每个 IP 仅会保存在 1000 个小文件中的某一个文件中。每个小文件中的数据以键值对的形式保存。最后可从这 1000 个小文件中找到一个或几个出现概率最大的 IP 。



### 7.海量日志数据，提取出搜索量前十的查询串，每个查询串的长度为1-255字节，要求内存不能超过1G(Top-K)

解法：
虽然有一千万 个Query，但是由于重复度比较高，因此事实上只有 300 万的 Query ，每个 Query 最大 255 Byte，因此我们可以考虑把他们都放进内存中去，而现在只是需要一个合适的数据结构，在这里， Hash 表是优先的选择。所以摒弃分而治之或 hash 映射的方法，直接上 hash 统计，然后排序。



### 8.一个1G大小的一个文件，里面每一行是一个词，词的大小不超过16字节，内存限制大小是1M。返回频数最高的100个词。

解法：
顺序读取文件，对于每个词 x，取 Hash(x)%5000 ，存放到对应的 5000 个小文件中。这样，每个小文件大概为 200k 左右。然后加载每一个小文件并做Hash统计。最后使用堆排序取出前 100 个高频词，使用归并排序进行总排序。

### 9.给定a、b两个文件，各存放50亿个url，每个url各占64字节，内存限制是4G，让你找出a、b文件共同的url？

解法：
顺序读取文件 a ，对每个 url 做如下操作：求 Hash(url)%1000 ，然后根据该值将 url 存放到 1000 个小文件中的某一个小文件并记录出现次数。按照这样划分，每个小文件的大小大约 300M 。同理，对文件 b 做上述操作。此时，如果存在相同的 url ，那么 url 一定会出现在同一个下标内的文件中。接下来只需要遍历这 1000 个文件即可。

### 9.5亿个整数中找出不重复的整数的个数，内存空间不足以容纳这2.5亿个整数

采用 2-Bitmap（每个数分配2bit，00表示不存在，01表示出现一次，10表示多次，11无意义）进行，共需内存 `2^32*2bit=1GB` 内存，还可以接受。然后扫描这 2.5 亿个整数，查看 Bitmap 中相对应位，如果是 00 变 01 ， 01 变 10 ， 10 保持不变。扫描后，查看 bitmap ，把对应位是 01 的整数输出即可。

### 10.5亿个int找它们的中位数

首先我们将 int 划分为 2^16 个区域，然后读取数据统计落到各个区域里的数的个数，之后我们根据统计结果就可以判断中位数落到那个区域，同时知道这个区域中的第几大数刚好是中位数。然后第二次扫描我们只统计落在这个区域中的那些数就可以了。
实际上，如果不是 int 是 int64 ，我们可以经过 3 次这样的划分即可降低到可以接受的程度。即可以先将 int64 分成 2^24 个区域，然后确定区域的第几大数，在将该区域分成 2^20 个子区域，然后确定是子区域的第几大数，然后子区域里的数的个数只有 2^20 ，就可以直接利用direct addr table 进行统计了。 　　

### 11.已知某个文件内包含一些电话号码，每个号码为8位数字，统计不同号码的个数

8 位最多99 999 999，大概需要 99m 个bit，大概 10m 字节的内存即可。 （可以理解为从0-99 999 999的数字，每个数字对应一个Bit位，所以只需要 99m 个Bit==1.2MBytes，这样，就用了小小的 1.2M 左右的内存表示了所有的8位数的电话）

### 12.给40亿个不重复的unsigned int的整数，没排过序的，然后再给一个数，如何快速判断这个数是否在那40亿个数当中？

申请 512M 的内存，一个 bit 位代表一个 unsigned int 值。读入 40 亿个数，设置相应的 bit 位，读入要查询的数，查看相应 bit 位是否为 1 ，为 1 表示存在，为 0 表示不存在。





## 数据结构与算法

### 1.图BFS,DFS

![img](https://upload-images.jianshu.io/upload_images/272719-a3bcab502f925cb5?imageMogr2/auto-orient/strip|imageView2/2/w/218/format/webp)

初始状态，从顶点1开始，队列={1}

访问1的邻接顶点，1出队变黑，2,3入队，队列={2,3,}

访问2的邻接结点，2出队，4入队，队列={3,4}

访问3的邻接结点，3出队，队列={4}

访问4的邻接结点，4出队，队列={ 空}
 结点5对于1来说不可达。

![img](https://upload-images.jianshu.io/upload_images/272719-49a1bce1bdbb51c9.PNG?imageMogr2/auto-orient/strip|imageView2/2/w/214/format/webp)

初始状态，从顶点1开始

依次访问过顶点1,2,3后，终止于顶点3

从顶点3回溯到顶点2，继续访问顶点5，并且终止于顶点5

从顶点5回溯到顶点2，并且终止于顶点2

从顶点2回溯到顶点1，并终止于顶点1

从顶点4开始访问，并终止于顶点4

### 2.快排（partition过程挺重要的）与堆排

```java
class Solution {
    public int[] sortArray(int[] nums) {
        if (nums == null || nums.length < 2) {
            return nums;
        }
        quickSort(nums, 0, nums.length - 1);
        return nums;
    }
    public static void quickSort(int[] arr, int l, int r) {
        if (l < r) {
            swap(arr, l + (int) (Math.random() * (r - l + 1)), r);   //随机选择一个数作为比较对象
            int[] p = partition(arr, l, r);
            quickSort(arr, l, p[0] - 1);
            quickSort(arr, p[1] + 1, r);
        }
    }
    public static int[] partition(int[] arr, int l, int r) {
        int less = l - 1;
        int more = r;
        while (l < more) {
            if (arr[l] < arr[r]) {
                swap(arr, ++less, l++);
            } else if (arr[l] > arr[r]) {
                swap(arr, --more, l);
            } else {
                l++;
            }
        }
        swap(arr, more, r);
        return new int[] { less + 1, more };
    }


    public static void swap(int[] arr, int i, int j) {
        int tmp = arr[i];
        arr[i] = arr[j];
        arr[j] = tmp;
    }
}
```



```Java
import java.util.Arrays;

/**
 * 大顶堆-->升序排列
 *
 * http://www.cnblogs.com/chengxiao/p/6129630.html
 * （1）首先要定义大顶堆：arr[0...n-1] 则 arr[i] > arr[2*i+1] 并且 arr[i] > arr[2*i+2]
 * （2）对于最开始的数组生成一个大顶堆：从最后一个非叶子节点开始调整
 * （3）对于生成的大顶堆，根节点元素是整个数组中最大的元素
 *      （i）首先将根节点元素和数组最后一个元素置换，则最后一个元素是最大的元素【有一点冒泡排序的处理味道】
 *      （ii）上面的操作会打乱大顶堆，所以需要对于arr[0...n-2]重新生成大顶堆
 * （4）堆排序中涉及到的树的概念，但是并没有显示的存储树，而是根据数组元素位置和树节点之间的对应关系调整假想中的树，有点意思
 *
 * 堆排序可以用来解决topK问题：从100W个数中找到最大的K个数
 * （1）首先对于数组前k个数建立一个小顶堆
 * （2）然后从第k个数到最后一个数，如果数比堆顶元素大，那么交换
 * （3）调整小顶堆的顶部元素
 * （4）最后得到的小顶堆就是从小到大的元素
 */
public class HeapSort {

    public static void heapSort(int[] arr){
        //0. 首先是建立一个大顶堆
        for(int i=arr.length/2-1; i >=0 ; i--){ //1.从第一个非叶子节点开始，直到顶端root节点
            adjustHeap(arr,i,arr.length);
        }

        //1. 然后不断交换元素,缩小大顶堆
        for(int j=arr.length-1;j > 0; j--){ //2. 将大顶堆第一个元素和最后一个元素替换，替换之后重新调整堆
            swap(arr,0,j);
            adjustHeap(arr,0,j); //这里只需要调整被打乱的根节点即可
        }
    }

    //3. 参数：数组本身，调整节点的下标，数组的长度
    public static void adjustHeap(int[] arr, int i, int length){ //i的含义：调整树中第i个节点【和左右子树对比】
        int temp = arr[i];
        for(int k = 2*i+1; k < length; k = 2*k+1){
            if(k+1 < length && arr[k] < arr[k+1]) //找到左右子树中最大的【反证：如果左子树<右子树，那么交换之后根节点还是小于右子树】
                k += 1;

            if(arr[k] > temp){ //对比子树和父节点
                arr[i] = arr[k];
                i = k; //注意为什么每次要改变i的值
            }else{
                break; //不需要调整，完事
            }
        }
        arr[i] = temp; //归位
    }

    public static void swap(int[] arr, int a, int b){
        int temp = arr[a];
        arr[a] = arr[b];
        arr[b] = temp;
    }

    public static void main(String[] args){
        int[] arr = {9,8,7,6,5,4,3,2,1};
        heapSort(arr);
        System.out.println(Arrays.toString(arr));
    }
}
```

### 3.常见排序的时间复杂度空间复杂度是否稳定

| 排序法 | 平均时间     | 最差情形    | 稳定度     | 额外空间 | 备注                          |
| ------ | ------------ | ----------- | ---------- | -------- | ----------------------------- |
| 冒泡   | **O(n2)**    | O(n2)       | **稳定**   | O(1)     | n小时较好                     |
| 交换   | O(n2)        | O(n2)       | 不稳定     | O(1)     | n小时较好                     |
| 选择   | O(n2)        | O(n2)       | 不稳定     | O(1)     | n小时较好                     |
| 插入   | O(n2)        | O(n2)       | 稳定       | O(1)     | 大部分已排序时较好            |
| 基数   | O(logRB)     | O(logRB)    | 稳定       | O(n)     | B是真数(0-9)，R是基数(个十百) |
| Shell  | O(nlogn)     | O(ns) 1<s<2 | 不稳定     | O(1)     | s是所选分组                   |
| 快速   | O(nlogn)     | O(n2)       | **不稳定** | O(nlogn) | n大时较好                     |
| 归并   | O(nlogn)     | O(nlogn)    | 稳定       | O(1)     | n大时较好                     |
| 堆     | **O(nlogn)** | O(nlogn)    | 不稳定     | O(1)     | n大时较好                     |

**关于排序稳定性的定义**

通俗地讲就是能保证排序前两个相等的数其在序列的前后位置顺序和排序后它们两个的前后位置顺序相同。在简单形式化一下，如果Ai = Aj，Ai原来在位置前，排序后Ai还是要在Aj位置前。

1.快速排序是否是稳定排序

不稳定。比如序列为5 3 3 4 3 8 9 10 11，现在中枢元素5和3（第5个元素，下标从1开始计）交换就会把元素3的稳定性打乱，所以快速排序是一个不稳定的排序算法，不稳定发生在中枢元素和a[j] 交换的时刻。

2.哪个常见排序是稳定的，为什么

冒泡排序，两个元素如果相同，可以控制不用发生交换。

桶排序比较适合用在外部排序中。所谓的外部排序就是数据存储在外部磁盘中，数据量比较大，内存有限，无法将数据全部加载到内存中。

我们可以先扫描一遍文件，看订单金额所处的数据范围。假设经过扫描之后我们得到，订单金额最小是 1 元，最大是 10 万元。我们将所有订单根据金额划分到 100 个桶里，第一个桶我们存储金额在 1 元到 1000 元之内的订单，第二桶存储金额在 1001 元到 2000 元之内的订单，以此类推。每一个桶对应一个文件，并且按照金额范围的大小顺序编号命名（00，01，02...99）。

理想的情况下，如果订单金额在 1 到 10 万之间均匀分布，那订单会被均匀划分到 100 个文件中，每个小文件中存储大约 100MB 的订单数据，我们就可以将这 100 个小文件依次放到内存中，用快排来排序。等所有文件都排好序之后，我们只需要按照文件编号，从小到大依次读取每个小文件中的订单数据，并将其写入到一个文件中，那这个文件中存储的就是按照金额从小到大排序的订单数据了。



### 4.图相关代码

```java

public class Graph { // 无向图
  private int v; // 顶点的个数
  private LinkedList<Integer> adj[]; // 邻接表

  public Graph(int v) {
    this.v = v;
    adj = new LinkedList[v];
    for (int i=0; i<v; ++i) {
      adj[i] = new LinkedList<>();
    }
  }

  public void addEdge(int s, int t) { // 无向图一条边存两次
    adj[s].add(t);
    adj[t].add(s);
  }
}
```

BFS

```java

public void bfs(int s, int t) {
  if (s == t) return;
  boolean[] visited = new boolean[v];
  visited[s]=true;
  Queue<Integer> queue = new LinkedList<>();
  queue.add(s);
  int[] prev = new int[v];
  for (int i = 0; i < v; ++i) {
    prev[i] = -1;
  }
  while (queue.size() != 0) {
    int w = queue.poll();
   for (int i = 0; i < adj[w].size(); ++i) {
      int q = adj[w].get(i);
      if (!visited[q]) {
        prev[q] = w;
        if (q == t) {
          print(prev, s, t);
          return;
        }
        visited[q] = true;
        queue.add(q);
      }
    }
  }
}

private void print(int[] prev, int s, int t) { // 递归打印s->t的路径
  if (prev[t] != -1 && t != s) {
    print(prev, s, prev[t]);
  }
  System.out.print(t + " ");
}
```

DFS

```java

boolean found = false; // 全局变量或者类成员变量

public void dfs(int s, int t) {
  found = false;
  boolean[] visited = new boolean[v];
  int[] prev = new int[v];
  for (int i = 0; i < v; ++i) {
    prev[i] = -1;
  }
  recurDfs(s, t, visited, prev);
  print(prev, s, t);
}

private void recurDfs(int w, int t, boolean[] visited, int[] prev) {
  if (found == true) return;
  visited[w] = true;
  if (w == t) {
    found = true;
    return;
  }
  for (int i = 0; i < adj[w].size(); ++i) {
    int q = adj[w].get(i);
    if (!visited[q]) {
      prev[q] = w;
      recurDfs(q, t, visited, prev);
    }
  }
}
```

### 5.字符串匹配

```java
package NowCoder.AdvancedClass;
/*应用判断树2是否是树1子树，先序列化然后KMP
怎么确定字符串不是某一子串重复得到的
或者将字符串增加得到一个新字符串是原串的重复abcabc->abcabcabc*/
public class KMP {
    public static int kmp(String str1,String str2){
        if (str1 == null || str2 == null || str1.length()<str2.length() || str2.length() < 1){
            return -1;
        }
        char [] ch1 = str1.toCharArray();
        char [] ch2 = str2.toCharArray();
        int c1 = 0;
        int c2 = 0;
        int [] next = getNextArray(ch2);
        while (c1<ch1.length && c2<ch2.length){
            if (ch1[c1] == ch2[c2]){
                c1++;
                c2++;
            }else if (next[c2] == -1){
                c1++;
            }else {
                c2 = next[c2];
            }
        }
        return c2 == ch2.length ? c1-c2 : -1;
    }

    public static int [] getNextArray(char [] str2){
        if (str2.length == 1){
            return new int [-1];
        }
        int [] next = new int [str2.length];
        next[0] = -1;
        next[1] = 0;
        int pos = 2;//走到那个位置
        int cn = 0;//当前位置的最长子串个数
        while (pos<next.length){
            if (str2[pos-1] == str2[cn]){
                next[pos++] = ++cn;
            }else if (cn>0){
                cn = next[cn];
            }else {
                next[pos++] = 0;
            }
        }
        return next;
    }

    public static void main(String[] args) {
        String str = "abdabc";
        String str2 = "abc";
        System.out.println(kmp(str, str2));
    }

}
```



### 6.字典树

```Java

public class Trie {
  private TrieNode root = new TrieNode('/'); // 存储无意义字符

  // 往Trie树中插入一个字符串
  public void insert(char[] text) {
    TrieNode p = root;
    for (int i = 0; i < text.length; ++i) {
      int index = text[i] - 'a';
      if (p.children[index] == null) {
        TrieNode newNode = new TrieNode(text[i]);
        p.children[index] = newNode;
      }
      p = p.children[index];
    }
    p.isEndingChar = true;
  }

  // 在Trie树中查找一个字符串
  public boolean find(char[] pattern) {
    TrieNode p = root;
    for (int i = 0; i < pattern.length; ++i) {
      int index = pattern[i] - 'a';
      if (p.children[index] == null) {
        return false; // 不存在pattern
      }
      p = p.children[index];
    }
    if (p.isEndingChar == false) return false; // 不能完全匹配，只是前缀
    else return true; // 找到pattern
  }

  public class TrieNode {
    public char data;
    public TrieNode[] children = new TrieNode[26];
    public boolean isEndingChar = false;
    public TrieNode(char data) {
      this.data = data;
    }
  }
}
//Trie 树比较适合的是查找前缀匹配的字符串
```



23.AC自动机

```java
//AC 自动机实际上就是在 Trie 树之上，加了类似 KMP 的 next 数组，只不过此处的 next 数组是构建在树上罢了

public class AcNode {
  public char data; 
  public AcNode[] children = new AcNode[26]; // 字符集只包含a~z这26个字符
  public boolean isEndingChar = false; // 结尾字符为true
  public int length = -1; // 当isEndingChar=true时，记录模式串长度
  public AcNode fail; // 失败指针
  public AcNode(char data) {
    this.data = data;
  }
}

public void buildFailurePointer() {
  Queue<AcNode> queue = new LinkedList<>();
  root.fail = null;
  queue.add(root);
  while (!queue.isEmpty()) {
    AcNode p = queue.remove();
    for (int i = 0; i < 26; ++i) {
      AcNode pc = p.children[i];
      if (pc == null) continue;
      if (p == root) {
        pc.fail = root;
      } else {
        AcNode q = p.fail;
        while (q != null) {
          AcNode qc = q.children[pc.data - 'a'];
          if (qc != null) {
            pc.fail = qc;
            break;
          }
          q = q.fail;
        }
        if (q == null) {
          pc.fail = root;
        }
      }
      queue.add(pc);
    }
  }
}

public void match(char[] text) { // text是主串
  int n = text.length;
  AcNode p = root;
  for (int i = 0; i < n; ++i) {
    int idx = text[i] - 'a';
    while (p.children[idx] == null && p != root) {
      p = p.fail; // 失败指针发挥作用的地方
    }
    p = p.children[idx];
    if (p == null) p = root; // 如果没有匹配的，从root开始重新匹配
    AcNode tmp = p;
    while (tmp != root) { // 打印出可以匹配的模式串
      if (tmp.isEndingChar == true) {
        int pos = i-tmp.length+1;
        System.out.println("匹配起始下标" + pos + "; 长度" + tmp.length);
      }
      tmp = tmp.fail;
    }
  }
}
```

23.拓扑排序

```java
//拓扑排序本身就是基于有向无环图的一个算法
public class Graph {
  private int v; // 顶点的个数
  private LinkedList<Integer> adj[]; // 邻接表

  public Graph(int v) {
    this.v = v;
    adj = new LinkedList[v];
    for (int i=0; i<v; ++i) {
      adj[i] = new LinkedList<>();
    }
  }

  public void addEdge(int s, int t) { // s先于t，边s->t
    adj[s].add(t);
  }
}


public void topoSortByKahn() {
  int[] inDegree = new int[v]; // 统计每个顶点的入度
  for (int i = 0; i < v; ++i) {
    for (int j = 0; j < adj[i].size(); ++j) {
      int w = adj[i].get(j); // i->w
      inDegree[w]++;
    }
  }
  LinkedList<Integer> queue = new LinkedList<>();
  for (int i = 0; i < v; ++i) {
    if (inDegree[i] == 0) queue.add(i);
  }
  while (!queue.isEmpty()) {
    int i = queue.remove();
    System.out.print("->" + i);
    for (int j = 0; j < adj[i].size(); ++j) {
      int k = adj[i].get(j);
      inDegree[k]--;
      if (inDegree[k] == 0) queue.add(k);
    }
  }
}
```

### 7.最短路径问题

```java
//在一个有向有权图中，求两个顶点间的最短路径。

public class Graph { // 有向有权图的邻接表表示
  private LinkedList<Edge> adj[]; // 邻接表
  private int v; // 顶点个数

  public Graph(int v) {
    this.v = v;
    this.adj = new LinkedList[v];
    for (int i = 0; i < v; ++i) {
      this.adj[i] = new LinkedList<>();
    }
  }

  public void addEdge(int s, int t, int w) { // 添加一条边
    this.adj[s].add(new Edge(s, t, w));
  }

  private class Edge {
    public int sid; // 边的起始顶点编号
    public int tid; // 边的终止顶点编号
    public int w; // 权重
    public Edge(int sid, int tid, int w) {
      this.sid = sid;
      this.tid = tid;
      this.w = w;
    }
  }
  // 下面这个类是为了dijkstra实现用的
  private class Vertex {
    public int id; // 顶点编号ID
    public int dist; // 从起始顶点到这个顶点的距离
    public Vertex(int id, int dist) {
      this.id = id;
      this.dist = dist;
    }
  }
}


// 因为Java提供的优先级队列，没有暴露更新数据的接口，所以我们需要重新实现一个
private class PriorityQueue { // 根据vertex.dist构建小顶堆
  private Vertex[] nodes;
  private int count;
  public PriorityQueue(int v) {
    this.nodes = new Vertex[v+1];
    this.count = v;
  }
  public Vertex poll() { // TODO: 留给读者实现... }
  public void add(Vertex vertex) { // TODO: 留给读者实现...}
  // 更新结点的值，并且从下往上堆化，重新符合堆的定义。时间复杂度O(logn)。
  public void update(Vertex vertex) { // TODO: 留给读者实现...} 
  public boolean isEmpty() { // TODO: 留给读者实现...}
}

public void dijkstra(int s, int t) { // 从顶点s到顶点t的最短路径
  int[] predecessor = new int[this.v]; // 用来还原最短路径
  Vertex[] vertexes = new Vertex[this.v];
  for (int i = 0; i < this.v; ++i) {
    vertexes[i] = new Vertex(i, Integer.MAX_VALUE);
  }
  PriorityQueue queue = new PriorityQueue(this.v);// 小顶堆
  boolean[] inqueue = new boolean[this.v]; // 标记是否进入过队列
  vertexes[s].dist = 0;
  queue.add(vertexes[s]);
  inqueue[s] = true;
  while (!queue.isEmpty()) {
    Vertex minVertex= queue.poll(); // 取堆顶元素并删除
    if (minVertex.id == t) break; // 最短路径产生了
    for (int i = 0; i < adj[minVertex.id].size(); ++i) {
      Edge e = adj[minVertex.id].get(i); // 取出一条minVetex相连的边
      Vertex nextVertex = vertexes[e.tid]; // minVertex-->nextVertex
      if (minVertex.dist + e.w < nextVertex.dist) { // 更新next的dist
        nextVertex.dist = minVertex.dist + e.w;
        predecessor[nextVertex.id] = minVertex.id;
        if (inqueue[nextVertex.id] == true) {
          queue.update(nextVertex); // 更新队列中的dist值
        } else {
          queue.add(nextVertex);
          inqueue[nextVertex.id] = true;
        }
      }
    }
  }
  // 输出最短路径
  System.out.print(s);
  print(s, t, predecessor);
}

private void print(int s, int t, int[] predecessor) {
  if (s == t) return;
  print(s, predecessor[t], predecessor);
  System.out.print("->" + t);
}
```

### 8.位图

我们有 1 千万个整数，整数的范围在 1 到 1 亿之间。如何快速查找某个整数是否在这 1 千万个整数中呢？

```java

public class BitMap { // Java中char类型占16bit，也即是2个字节
  private char[] bytes;
  private int nbits;
  
  public BitMap(int nbits) {
    this.nbits = nbits;
    this.bytes = new char[nbits/16+1];
  }

  public void set(int k) {
    if (k > nbits) return;
    int byteIndex = k / 16;
    int bitIndex = k % 16;
    bytes[byteIndex] |= (1 << bitIndex);
  }

  public boolean get(int k) {
    if (k > nbits) return false;
    int byteIndex = k / 16;
    int bitIndex = k % 16;
    return (bytes[byteIndex] & (1 << bitIndex)) != 0;
  }
}
```

如果用散列表存储这 1 千万的数据，数据是 32 位的整型数，也就是需要 4 个字节的存储空间，那总共至少需要 40MB 的存储空间。如果我们通过位图的话，数字范围在 1 到 1 亿之间，只需要 1 亿个二进制位，也就是 12MB 左右的存储空间就够了。

我们使用 K 个哈希函数，对同一个数字进行求哈希值，那会得到 K 个不同的哈希值，我们分别记作 X1，X2，X3，…，XK。我们把这 K 个数字作为位图中的下标，将对应的 BitMap[X1]，BitMap[X2]，BitMap[X3]，…，BitMap[XK]都设置成 true，也就是说，我们用 K 个二进制位，来表示一个数字的存在。

当我们要查询某个数字是否存在的时候，我们用同样的 K 个哈希函数，对这个数字求哈希值，分别得到 Y1，Y2，Y3，…，YK。我们看这 K 个哈希值，对应位图中的数值是否都为 true，如果都是 true，则说明，这个数字存在，如果有其中任意一个不为 true，那就说明这个数字不存在。



### 9.概率统计

如果你是一名手机应用开发工程师，让你实现一个简单的垃圾短信过滤功能以及骚扰电话拦截功能，该用什么样的数据结构和算法实现呢？

基于概率统计的过滤器，是基于短信内容来判定是否是垃圾短信。而计算机没办法像人一样理解短信的含义。所以，我们需要把短信抽象成一组计算机可以理解并且方便计算的特征项，用这一组特征项代替短信本身，来做垃圾短信过滤。

我们可以通过分词算法，把一个短信分割成 n 个单词。这 n 个单词就是一组特征项，全权代表这个短信。因此，判定一个短信是否是垃圾短信这样一个问题，就变成了，判定同时包含这几个单词的短信是否是垃圾短信

<img src="https://static001.geekbang.org/resource/image/0f/2a/0f0369a955ee8d15bd7d7958829d5b2a.jpg" alt="img" style="zoom:50%;" />

### 10.向量空间

\1. 基于相似用户做推荐

\2. 基于相似歌曲做推荐



### 11.搜索引擎背后的数据结构

搜索引擎大致可以分为四个部分：搜集、分析、索引、查询。其中，搜集，就是我们常说的利用爬虫爬取网页。分析，主要负责网页内容抽取、分词，构建临时索引，计算 PageRank 值这几部分工作。索引，主要负责通过分析阶段得到的临时索引，构建倒排索引。查询，主要负责响应用户的请求，根据倒排索引获取相关网页，计算网页排名，返回查询结果给用户。

搜索引擎采用的是广度优先搜索策略。具体点讲的话，那就是，我们先找一些比较知名的网页（专业的叫法是权重比较高）的链接（比如新浪主页网址、腾讯主页网址等），作为种子网页链接，放入到队列中。爬虫按照广度优先的策略，不停地从队列中取出链接，然后去爬取对应的网页，解析出网页里包含的其他网页链接，再将解析出来的链接添加到队列中。

1. 待爬取网页链接文件：links.bin在广度优先搜索爬取页面的过程中，爬虫会不停地解析页面链接，将其放到队列中。于是，队列中的链接就会越来越多，可能会多到内存放不下。所以，我们用一个存储在磁盘中的文件（links.bin）来作为广度优先搜索中的队列。爬虫从 links.bin 文件中，取出链接去爬取对应的页面。等爬取到网页之后，将解析出来的链接，直接存储到 links.bin 文件中。这样用文件来存储网页链接的方式，还有其他好处。比如，支持断点续爬。也就是说，当机器断电之后，网页链接不会丢失；当机器重启之后，还可以从之前爬取到的位置继续爬取。关于如何解析页面获取链接，我额外多说几句。我们可以把整个页面看作一个大的字符串，然后利用字符串匹配算法，在这个大字符串中，搜索这样一个网页标签，然后顺序读取之间的字符串。这其实就是网页链接。

2. 网页判重文件：bloom_filter.bin如何避免重复爬取相同的网页呢？这个问题我们在位图那一节已经讲过了。使用布隆过滤器，我们就可以快速并且非常节省内存地实现网页的判重。不过，还是刚刚那个问题，如果我们把布隆过滤器存储在内存中，那机器宕机重启之后，布隆过滤器就被清空了。这样就可能导致大量已经爬取的网页会被重复爬取。这个问题该怎么解决呢？我们可以定期地（比如每隔半小时）将布隆过滤器持久化到磁盘中，存储在 bloom_filter.bin 文件中。这样，即便出现机器宕机，也只会丢失布隆过滤器中的部分数据。当机器重启之后，我们就可以重新读取磁盘中的 bloom_filter.bin 文件，将其恢复到内存中。

3. 原始网页存储文件：doc_raw.bin爬取到网页之后，我们需要将其存储下来，以备后面离线分析、索引之用。那如何存储海量的原始网页数据呢？如果我们把每个网页都存储为一个独立的文件，那磁盘中的文件就会非常多，数量可能会有几千万，甚至上亿。常用的文件系统显然不适合存储如此多的文件。所以，我们可以把多个网页存储在一个文件中。每个网页之间，通过一定的标识进行分隔，方便后续读取。具体的存储格式，如下图所示。其中，doc_id 这个字段是网页的编号，我们待会儿再解释。

   当然，这样的一个文件也不能太大，因为文件系统对文件的大小也有一定的限制。所以，我们可以设置每个文件的大小不能超过一定的值（比如 1GB）。随着越来越多的网页被添加到文件中，文件的大小就会越来越大，当超过 1GB 的时候，我们就创建一个新的文件，用来存储新爬取的网页。假设一台机器的硬盘大小是 100GB 左右，一个网页的平均大小是 64KB。那在一台机器上，我们可以存储 100 万到 200 万左右的网页。假设我们的机器的带宽是 10MB，那下载 100GB 的网页，大约需要 10000 秒。也就是说，爬取 100 多万的网页，也就是只需要花费几小时的时间。

4. 网页链接及其编号的对应文件：doc_id.bin刚刚我们提到了网页编号这个概念，我现在解释一下。网页编号实际上就是给每个网页分配一个唯一的 ID，方便我们后续对网页进行分析、索引。那如何给网页编号呢？我们可以按照网页被爬取的先后顺序，从小到大依次编号。具体是这样做的：我们维护一个中心的计数器，每爬取到一个网页之后，就从计数器中拿一个号码，分配给这个网页，然后计数器加一。在存储网页的同时，我们将网页链接跟编号之间的对应关系，存储在另一个 doc_id.bin 文件中。爬虫在爬取网页的过程中，涉及的四个重要的文件，我就介绍完了。其中，links.bin 和 bloom_filter.bin 这两个文件是爬虫自身所用的。另外的两个（doc_raw.bin、doc_id.bin）是作为搜集阶段的成果，供后面的分析、索引、查询用的。

分析

网页爬取下来之后，我们需要对网页进行离线分析。分析阶段主要包括两个步骤，第一个是抽取网页文本信息，第二个是分词并创建临时索引。我们逐一来讲解。

\1. 抽取网页文本信息网页是半结构化数据，里面夹杂着各种标签、JavaScript 代码、CSS 样式。对于搜索引擎来说，它只关心网页中的文本信息，也就是，网页显示在浏览器中时，能被用户肉眼看到的那部分信息。我们如何从半结构化的网页中，抽取出搜索引擎关系的文本信息呢？我们之所以把网页叫作半结构化数据，是因为它本身是按照一定的规则来书写的。这个规则就是 HTML 语法规范。我们依靠 HTML 标签来抽取网页中的文本信息。这个抽取的过程，大体可以分为两步。第一步是去掉 JavaScript 代码、CSS 格式以及下拉框中的内容（因为下拉框在用户不操作的情况下，也是看不到的）。第二步是去掉所有 HTML 标签。这一步也是通过字符串匹配算法来实现的。过程跟第一步类似，我就不重复讲了。

\2. 分词并创建临时索引经过上面的处理之后，我们就从网页中抽取出了我们关心的文本信息。接下来，我们要对文本信息进行分词，并且创建临时索引。对于英文网页来说，分词非常简单。我们只需要通过空格、标点符号等分隔符，将每个单词分割开来就可以了。但是，对于中文来说，分词就复杂太多了。我这里介绍一种比较简单的思路，基于字典和规则的分词方法。其中，字典也叫词库，里面包含大量常用的词语（我们可以直接从网上下载别人整理好的）。我们借助词库并采用最长匹配规则，来对文本进行分词。所谓最长匹配，也就是匹配尽可能长的词语。我举个例子解释一下。比如要分词的文本是“中国人民解放了”，我们词库中有“中国”“中国人”“中国人民”“中国人民解放军”这几个词，那我们就取最长匹配，也就是“中国人民”划为一个词，而不是把“中国”、“中国人”划为一个词。具体到实现层面，我们可以将词库中的单词，构建成 Trie 树结构，然后拿网页文本在 Trie 树中匹配。每个网页的文本信息在分词完成之后，我们都得到一组单词列表。我们把单词与网页之间的对应关系，写入到一个临时索引文件中（tmp_Index.bin），这个临时索引文件用来构建倒排索引文件。临时索引文件的格式如下：

在临时索引文件中，我们存储的是单词编号，也就是图中的 term_id，而非单词本身。这样做的目的主要是为了节省存储的空间。那这些单词的编号是怎么来的呢？给单词编号的方式，跟给网页编号类似。我们维护一个计数器，每当从网页文本信息中分割出一个新的单词的时候，我们就从计数器中取一个编号，分配给它，然后计数器加一。在这个过程中，我们还需要使用散列表，记录已经编过号的单词。在对网页文本信息分词的过程中，我们拿分割出来的单词，先到散列表中查找，如果找到，那就直接使用已有的编号；如果没有找到，我们再去计数器中拿号码，并且将这个新单词以及编号添加到散列表中。当所有的网页处理（分词及写入临时索引）完成之后，我们再将这个单词跟编号之间的对应关系，写入到磁盘文件中，并命名为 term_id.bin。经过分析阶段，我们得到了两个重要的文件。它们分别是临时索引文件（tmp_index.bin）和单词编号文件（term_id.bin）。

索引

索引阶段主要负责将分析阶段产生的临时索引，构建成倒排索引。倒排索引（ Inverted index）中记录了每个单词以及包含它的网页列表。文字描述比较难理解，我画了一张倒排索引的结构图，你一看就明白。

我们刚刚讲到，在临时索引文件中，记录的是单词跟每个包含它的文档之间的对应关系。那如何通过临时索引文件，构建出倒排索引文件呢？这是一个非常典型的算法问题，你可以先自己思考一下，再看我下面的讲解。解决这个问题的方法有很多。考虑到临时索引文件很大，无法一次性加载到内存中，搜索引擎一般会选择使用多路归并排序的方法来实现。我们先对临时索引文件，按照单词编号的大小进行排序。因为临时索引很大，所以一般基于内存的排序算法就没法处理这个问题了。我们可以用之前讲到的归并排序的处理思想，将其分割成多个小文件，先对每个小文件独立排序，最后再合并在一起。当然，实际的软件开发中，我们其实可以直接利用 MapReduce 来处理。临时索引文件排序完成之后，相同的单词就被排列到了一起。我们只需要顺序地遍历排好序的临时索引文件，就能将每个单词对应的网页编号列表找出来，然后把它们存储在倒排索引文件中。除了倒排文件之外，我们还需要一个文件，来记录每个单词编号在倒排索引文件中的偏移位置。我们把这个文件命名为 term_offset.bin。这个文件的作用是，帮助我们快速地查找某个单词编号在倒排索引中存储的位置，进而快速地从倒排索引中读取单词编号对应的网页编号列表。

查询前面三个阶段的处理，只是为了最后的查询做铺垫。因此，现在我们就要利用之前产生的几个文件，来实现最终的用户搜索功能。doc_id.bin：记录网页链接和编号之间的对应关系。term_id.bin：记录单词和编号之间的对应关系。index.bin：倒排索引文件，记录每个单词编号以及对应包含它的网页编号列表。term_offsert.bin：记录每个单词编号在倒排索引文件中的偏移位置。这四个文件中，除了倒排索引文件（index.bin）比较大之外，其他的都比较小。为了方便快速查找数据，我们将其他三个文件都加载到内存中，并且组织成散列表这种数据结构。当用户在搜索框中，输入某个查询文本的时候，我们先对用户输入的文本进行分词处理。假设分词之后，我们得到 k 个单词。我们拿这 k 个单词，去 term_id.bin 对应的散列表中，查找对应的单词编号。经过这个查询之后，我们得到了这 k 个单词对应的单词编号。我们拿这 k 个单词编号，去 term_offset.bin 对应的散列表中，查找每个单词编号在倒排索引文件中的偏移位置。经过这个查询之后，我们得到了 k 个偏移位置。我们拿这 k 个偏移位置，去倒排索引（index.bin）中，查找 k 个单词对应的包含它的网页编号列表。经过这一步查询之后，我们得到了 k 个网页编号列表。我们针对这 k 个网页编号列表，统计每个网页编号出现的次数。具体到实现层面，我们可以借助散列表来进行统计。统计得到的结果，我们按照出现次数的多少，从小到大排序。出现次数越多，说明包含越多的用户查询单词（用户输入的搜索文本，经过分词之后的单词）。经过这一系列查询，我们就得到了一组排好序的网页编号。我们拿着网页编号，去 doc_id.bin 文件中查找对应的网页链接，分页显示给用户就可以了。



### 12.布隆过滤器

- **能够判定元素一定不存在**
- 能够判定元素可能存在

(1)添加元素

布隆过滤器的原理是，**当一个元素被加入集合时，通过K个hash函数将这个元素映射成一个位数组中的K个点，把它们置为1。**检索时，我们只要看看这些点是不是都是1就（大约）知道集合中有没有它了：如果这些点有任何一个0，则被检元素一定不在；如果都是1，则被检元素很可能在。这就是布隆过滤器的基本思想。

<因为不同的key，可能经过K个hash函数之后产生的多个位下标是相同的[或者多个key构成了一个新的映射集合]，**导致误判**>

(2)判断元素是否存在

当我们要判断一个元素是否在布隆过滤器中时，我们把这个值传入k个hash函数中获得映射的k个点。这一次我们确认下是否所有的点都被置为1了，如果有某一位没有置为1则这个元素**肯定不在集合中**。如果都在那这个元素就**有可能在集合中**。











## 分布式

### 1.CAP理论

分区容错性：指的分布式系统中的某个节点或者网络分区出现了故障的时候，整个系统仍然能对外提供满足一致性和可用性的服务。也就是说部分故障不影响整体使用。事实上我们在设计分布式系统是都会考虑到bug,硬件，网络等各种原因造成的故障，所以即使部分节点或者网络出现故障，我们要求整个系统还是要继续使用的(不继续使用,相当于只有一个分区,那么也就没有后续的一致性和可用性了)

可用性： 一直可以正常的做读写操作。简单而言就是客户端一直可以正常访问并得到系统的正常响应。用户角度来看就是不会出现系统操作失败或者访问超时等问题。

一致性：在分布式系统完成某写操作后任何读操作，都应该获取到该写操作写入的那个最新的值。相当于要求分布式系统中的各节点时时刻刻保持数据的一致性。

### 2.base理论

向参与者基本可用（Basically Available）软状态（Soft State）最终一致性（Eventually Consistent）

什么是基本可用呢？

假设系统，出现了不可预知的故障，但还是能用，相比较正常的系统而言：响应时间上的损失：正常情况下的搜索引擎 0.5 秒即返回给用户结果，而基本可用的搜索引擎可以在 1 秒作用返回结果。

功能上的损失：在一个电商网站上，正常情况下，用户可以顺利完成每一笔订单，但是到了大促期间，为了保护购物系统的稳定性，部分消费者可能会被引导到一个降级页面。

什么是软状态呢？

相对于原子性而言，要求多个节点的数据副本都是一致的，这是一种 “硬状态”。软状态指的是：允许系统中的数据存在中间状态，并认为该状态不影响系统的整体可用性，即允许系统在多个不同节点的数据副本存在数据延时。系统能够保证在没有其他新的更新操作的情况下，数据最终一定能够达到一致的状态，因此所有客户端对系统的数据访问最终都能够获取到最新的值。

### 3.分布式事务 2PC 和 TCC 的原理

分布式事务是为了解决微服务架构（形式都是分布式系统）中不同节点之间的数据一致性问题。这个一致性问题本质上解决的也是传统事务需要解决的问题，即一个请求在多个微服务调用链中，所有服务的数据处理要么全部成功，要么全部回滚。当然分布式事务问题的形式可能与传统事务会有比较大的差异，但是问题本质是一致的，都是要求解决数据的一致性问题。

两阶段提交（2PC）分布式事务的发起方在向分布式事务协调者（Coordinator）发送请求时，Coordinator首先会分别（Partcipant）节点A、参与这节点（Partcipant）节点B分别发送事务预处理请求，称之为Prepare，有些资料也叫"Vote Request"。说的直白点就是问一下这些参与节点"这件事你们能不能处理成功了"，此时这些参与者节点一般来说就会打开本地数据库事务，然后开始执行数据库本地事务，但在执行完成后并不会立马提交数据库本地事务，而是先向Coordinator报告说：“我这边可以处理了/我这边不能处理”。如果所有参与者节点都向协调者报告说“我这边可以处理”，那么此时协调者就会向所有参与者节点发送“全局提交确认通知（global_commit）”，即你们都可以进行本地事务提交了，此时参与者节点就会完成自身本地数据库事务的提交，并最终将提交结果回复“ack”消息给Coordinator，然后Coordinator就会向调用方返回分布式事务处理完成的结果。在第二阶段除了所有的参与者节点都反馈“我这边可以处理了”的情况外，也会有节点反馈说“我这边不能处理”的情况发生，此时参与者节点就会向协调者节点反馈“Vote_Abort”的消息。此时分布式事务协调者节点就会向所有的参与者节点发起事务回滚的消息（“global_rollback”），此时各个参与者节点就会回滚本地事务，释放资源，并且向协调者节点发送“ack”确认消息，协调者节点就会向调用方返回分布式事务处理失败的结果。

三阶段提交（3PC）三阶段提交又称3PC，其在两阶段提交的基础上增加了CanCommit阶段，并引入了超时机制。一旦事务参与者迟迟没有收到协调者的Commit请求，就会自动进行本地commit，这样相对有效地解决了协调者单点故障的问题。事务的协调者向所有参与者询问“你们是否可以完成本次事务？”，如果参与者节点认为自身可以完成事务就返回“YES”，否则“NO”。而在实际的场景中参与者节点会对自身逻辑进行事务尝试，其实说白了就是检查下自身状态的健康性，看有没有能力进行事务操作。在阶段一中，如果所有的参与者都返回Yes的话，那么就会进入PreCommit阶段进行事务预提交。此时分布式事务协调者会向所有的参与者节点发送PreCommit请求，参与者收到后开始执行事务操作，并将Undo和Redo信息记录到事务日志中。参与者执行完事务操作后（此时属于未提交事务的状态），就会向协调者反馈“Ack”表示我已经准备好提交了，并等待协调者的下一步指令。否则，如果阶段一中有任何一个参与者节点返回的结果是No响应，或者协调者在等待参与者节点反馈的过程中超时（2PC中只有协调者可以超时，参与者没有超时机制）。整个分布式事务就会中断，协调者就会向所有的参与者发送“abort”请求。在阶段二中如果所有的参与者节点都可以进行PreCommit提交，那么协调者就会从“预提交状态”-》“提交状态”。然后向所有的参与者节点发送"doCommit"请求，参与者节点在收到提交请求后就会各自执行事务提交操作，并向协调者节点反馈“Ack”消息，协调者收到所有参与者的Ack消息后完成事务。相反，如果有一个参与者节点未完成PreCommit的反馈或者反馈超时，那么协调者都会向所有的参与者节点发送abort请求，从而中断事务。这个优化点，主要是避免了参与者在长时间无法与协调者节点通讯（协调者挂掉了）的情况下，无法释放资源的问题，因为参与者自身拥有超时机制会在超时后，自动进行本地commit从而进行释放资源。而这种机制也侧面降低了整个事务的阻塞时间和范围。

补偿事务（TCC）TCC事务的处理流程与2PC两阶段提交类似，不过2PC通常都是在跨库的DB层面，而TCC本质上就是一个应用层面的2PC，需要通过业务逻辑来实现。这种分布式事务的实现方式的优势在于，可以让应用自己定义数据库操作的粒度，使得降低锁冲突、提高吞吐量成为可能。而不足之处则在于对应用的侵入性非常强，业务逻辑的每个分支都需要实现try、confirm、cancel三个操作。此外，其实现难度也比较大，需要按照网络状态、系统故障等不同的失败原因实现不同的回滚策略。为了满足一致性的要求，confirm和cancel接口还必须实现幂等。





### 4.分布式锁Redis

- **加锁**

在沙滩上踩一脚，留下自己的脚印，就对应了加锁操作。其他进程或者线程，看到沙滩上已经有脚印，证明锁已被别人持有，则等待。

- **解锁**

把脚印从沙滩上抹去，就是解锁的过程。

- **锁超时**

为了避免死锁，我们可以设置一阵风，在单位时间后刮起，将脚印自动抹去。

我们先来看如何通过单节点Redis实现一个简单的分布式锁。

**实现分布式锁要满足3点：多进程可见，互斥，可重入。**

![image-20210802193343607](C:\Users\TianYu\AppData\Roaming\Typora\typora-user-images\image-20210802193343607.png)

![image-20210802193400746](C:\Users\TianYu\AppData\Roaming\Typora\typora-user-images\image-20210802193400746.png)

![image-20210802193410053](C:\Users\TianYu\AppData\Roaming\Typora\typora-user-images\image-20210802193410053.png)

![image-20210802193453672](C:\Users\TianYu\AppData\Roaming\Typora\typora-user-images\image-20210802193453672.png)





### 5.RPC相关问题

#### 1.rpc工作过程

1. 服务消费端（client）以本地调用的方式调用远程服务；
2. 客户端 Stub（client stub） 接收到调用后负责将方法、参数等组装成能够进行网络传输的消息体（序列化）：`RpcRequest`；
3. 客户端 Stub（client stub） 找到远程服务的地址，并将消息发送到服务提供端；
4. 服务端 Stub（桩）收到消息将消息反序列化为 Java 对象: `RpcRequest`；
5. 服务端 Stub（桩）根据`RpcRequest`中的类、方法、方法参数等信息调用本地的方法；
6. 服务端 Stub（桩）得到方法执行结果并将组装成能够进行网络传输的消息体：`RpcResponse`（序列化）发送至消费方；
7. 客户端 Stub（client stub）接收到消息并将消息反序列化为 Java 对象:`RpcResponse` ，这样也就得到了最终结果。over!

#### 2.为什么用 RPC，不用 HTTP

首先需要指正，这两个并不是并行概念。RPC 是一种**设计**，就是为了解决**不同服务之间的调用问题**，完整的 RPC 实现一般会包含有 **传输协议** 和 **序列化协议** 这两个。

而 HTTP 是一种传输协议，RPC 框架完全可以使用 HTTP 作为传输协议，也可以直接使用 TCP，使用不同的协议一般也是为了适应不同的场景。

使用 TCP 和使用 HTTP 各有优势：

**传输效率**：

- TCP，通常自定义上层协议，可以让请求报文体积更小 
- HTTP：如果是基于HTTP 1.1 的协议，请求中会包含很多无用的内容 

**性能消耗**，主要在于序列化和反序列化的耗时

- TCP，可以基于各种序列化框架进行，效率比较高 
- HTTP，大部分是通过 json 来实现的，字节大小和序列化耗时都要更消耗性能 

**跨平台**：

- TCP：通常要求[客户端]()和服务器为统一平台 
- HTTP：可以在各种异构系统上运行 

**总结**：
  RPC 的 TCP 方式主要用于公司内部的服务调用，性能消耗低，传输效率高。HTTP主要用于对外的异构环境，浏览器接口调用，APP接口调用，第三方接口调用等。



#### 3.序列化方式

JSON 是一种轻量级的数据交换语言，该语言以易于让人阅读的文字为基础，用来传输由属性值或者序列性的值组成的数据对象，类似 xml，Json 比 xml更小、更快更容易解析。JSON 由于采用字符方式存储，占用相对于字节方式较大，并且序列化后类的信息会丢失，可能导致反序列化失败。

剩下的都是基于字节的序列化。

Kryo 是一个快速高效的 Java 序列化框架，旨在提供快速、高效和易用的 API。无论文件、数据库或网络数据 Kryo 都可以随时完成序列化。 Kryo 还可以执行自动深拷贝、浅拷贝。这是对象到对象的直接拷贝，而不是对象->字节->对象的拷贝。kryo 速度较快，序列化后体积较小，但是跨语言支持较复杂。

Hessian 是一个基于二进制的协议，Hessian 支持很多种语言，例如 Java、python、[c++](),、net/c#、D、Erlang、PHP、Ruby、object-c等，它的序列化和反序列化也是非常高效。速度较慢，序列化后的体积较大。

protobuf（[Proto]()col Buffers）是由 Google 发布的数据交换格式，提供跨语言、跨平台的序列化和反序列化实现，底层由 C++ 实现，其他平台使用时必须使用 protocol compiler 进行预编译生成 protoc 二进制文件。性能主要消耗在文件的预编译上。序列化反序列化性能较高，平台无关。



#### 4.netty线程模型

[Netty]() 通过 Reactor 模型基于多路复用器接收并处理用户请求，内部实现了两个线程池， boss 线程池和 worker 线程池，其中 boss 线程池的线程负责处理请求的 accept 事件，当接收到 accept 事件的请求时，把对应的 socket 封装到一个 NioSocketChannel 中，并交给 worker 线程池，其中 worker 线程池负责请求的 read 和 write 事件，由对应的Handler 处理。

单线程模型：所有I/O操作都由一个线程完成，即多路复用、事件分发和处理都是在一个Reactor 线程上完成的。既要接收[客户端]()的连接请求,向服务端发起连接，又要发送/读取请求或应答/响应消息。一个NIO 线程同时处理成百上千的链路，性能上无法支撑，速度慢，若线程进入死循环，整个程序不可用，对于高负载、大并发的应用场景不合适。

多线程模型：有一个NIO 线程（Acceptor） 只负责监听服务端，接收[客户端]()的TCP 连接请求；NIO 线程池负责网络IO 的操作，即消息的读取、解码、编码和发送；1 个NIO 线程可以同时处理N 条链路，但是1 个链路只对应1 个NIO 线程，这是为了防止发生并发操作问题。但在并发百万[客户端]()连接或需要安全认证时，一个Acceptor 线程可能会存在性能不足问题。

主从多线程模型：Acceptor 线程用于绑定监听端口，接收[客户端]()连接，将SocketChannel 从主线程池的 Reactor 线程的多路复用器上移除，重新注册到Sub 线程池的线程上，用于处理I/O 的读写等操作，从而保证 mainReactor 只负责接入认证、握手等操作；



#### 5.说下 [Netty](https://www.nowcoder.com/jump/super-jump/word?word=Netty) 零拷贝

[Netty]() 的零拷贝主要包含三个方面：

- [Netty]() 的接收和发送 ByteBuffer 采用 DIRECT BUFFERS，使用堆外直接内存进行 Socket 读写，不需要进行字节缓冲区的二次拷贝。如果使用传统的堆内存（HEAP BUFFERS）进行 Socket 读写，JVM 会将堆内存 Buffer 拷贝一份到直接内存中，然后才写入 Socket 中。相比于堆外直接内存，消息在发送过程中多了一次缓冲区的内存拷贝。 
- [Netty]() 提供了组合 Buffer 对象，可以聚合多个 ByteBuffer 对象，用户可以像操作一个 Buffer 那样方便的对组合 Buffer 进行操作，避免了传统通过内存拷贝的方式将几个小 Buffer 合并成一个大的 Buffer。 
- [Netty]() 的文件传输采用了 transferTo 方法，它可以直接将文件缓冲区的数据发送到目标 Channel，避免了传统通过循环 write 方式导致的内存拷贝问题。



#### 6.netty组件

Channel：[Netty]() 网络操作抽象类，它除了包括基本的 I/O 操作，如 bind、connect、read、write 等。 

EventLoop：主要是配合 Channel 处理 I/O 操作，用来处理连接的生命周期中所发生的事情。 

ChannelFuture：[Netty]() 框架中所有的 I/O 操作都为异步的，因此我们需要 ChannelFuture 的 addListener()注册一个 ChannelFutureListener 监听事件，当操作执行成功或者失败时，监听就会自动触发返回结果。 

ChannelHandler：充当了所有处理入站和出站数据的逻辑容器。ChannelHandler 主要用来处理各种事件，这里的事件很广泛，比如可以是连接、数据接收、异常、数据转换等。 

ChannelPipeline：为 ChannelHandler 链提供了容器，当 channel 创建时，就会被自动分配到它专属的 ChannelPipeline，这个关联是永久性的。



#### 7.Netty]() 是如何保持长连接的（心跳）

首先 TCP 协议的实现中也提供了 [keep]()alive 报文用来探测对端是否可用。TCP 层将在定时时间到后发送相应的 KeepAlive 探针以确定连接可用性。

`ChannelOption.SO_KEEPALIVE, true` 表示打开 TCP 的 [keep]()Alive 设置。

TCP 心跳的问题：

考虑一种情况，某台服务器因为某些原因导致负载超高，CPU 100%，无法响应任何业务请求，但是使用 TCP 探针则仍旧能够确定连接状态，这就是典型的连接活着但业务提供方已死的状态，对[客户端]()而言，这时的最好选择就是断线后重新连接其他服务器，而不是一直认为当前服务器是可用状态一直向当前服务器发送些必然会失败的请求。

[Netty]() 中提供了 `IdleStateHandler` 类专门用于处理心跳。

`IdleStateHandler` 的构造函数如下：

[复制代码](#)

```
public` `IdleStateHandler(``long` `readerIdleTime, ``long` `writerIdleTime, ``            ``long` `allIdleTime,TimeUnit unit){``}
```

第一个参数是隔多久检查一下读事件是否发生，如果 `channelRead()` 方法超过 readerIdleTime 时间未被调用则会触发超时事件调用 `userEventTrigger()` 方法；

第二个参数是隔多久检查一下写事件是否发生，writerIdleTime 写空闲超时时间设定，如果 `write()` 方法超过 writerIdleTime 时间未被调用则会触发超时事件调用 `userEventTrigger()` 方法；

第三个参数是全能型参数，隔多久检查读写事件；

第四个参数表示当前的时间单位。

所以这里可以分别控制读，写，读写超时的时间，单位为秒，如果是0表示不检测，所以如果全是0，则相当于没添加这个 IdleStateHandler，连接是个普通的短连接。



#### 8.Netty]() 中责任链

首先说明责任链模式：

适用场景:

- 对于一个请求来说,如果有个对象都有机会处理它,而且不明确到底是哪个对象会处理请求时,我们可以考虑使用责任链模式实现它,让请求从链的头部往后移动,直到链上的一个节点成功处理了它为止 

优点:

- 发送者不需要知道自己发送的这个请求到底会被哪个对象处理掉,实现了发送者和接受者的解耦 
- 简化了发送者对象的设计 
- 可以动态的添加节点和删除节点 

缺点:

- 所有的请求都从链的头部开始遍历,对性能有损耗 
- 极差的情况,不保证请求一定会被处理 

[Netty]()的责任链：

netty 的 pipeline 设计,就采用了责任链设计模式, 底层采用双向[链表]()的数据结构, 将链上的各个处理器串联起来

[客户端]()每一个请求的到来，netty 都认为，pipeline 中的所有的处理器都有机会处理它，因此，对于入栈的请求，全部从头节点开始往后传播，一直传播到尾节点（来到尾节点的 msg 会被释放掉）。

责任终止机制

- 在pipeline中的任意一个节点，只要我们不手动的往下传播下去，这个事件就会终止传播在当前节点 
- 对于入站数据，默认会传递到尾节点，进行回收，如果我们不进行下一步传播，事件就会终止在当前节点





### 6.分布式ID

- 全局唯一：必须保证ID是全局性唯一的，基本要求
- 高性能：高可用低延时，ID生成响应要块，否则反倒会成为业务瓶颈
- 高可用：100%的可用性是骗人的，但是也要无限接近于100%的可用性
- 好接入：要秉着拿来即用的设计原则，在系统设计和实现上要尽可能的简单
- 趋势递增：最好趋势递增，这个要求就得看具体业务场景了，一般不严格要求

##### 1、基于UUID

**优点：**

- 生成足够简单，本地生成无网络消耗，具有唯一性

**缺点：**

- 无序的字符串，不具备趋势自增特性
- 没有具体的业务含义
- 长度过长16 字节128位，36位长度的字符串，存储以及查询对MySQL的性能消耗较大，MySQL官方明确建议主键要尽量越短越好，作为数据库主键 `UUID` 的无序性会导致数据位置频繁变动，严重影响性能。

##### 2、基于数据库自增ID

基于数据库的`auto_increment`自增ID完全可以充当`分布式ID`，具体实现：需要一个单独的MySQL实例用来生成ID

**优点：**

- 实现简单，ID单调自增，数值类型查询速度快

**缺点：**

- DB单点存在宕机风险，无法扛住高并发场景

##### 3、基于数据库集群模式

前边说了单点数据库方式不可取，那对上边的方式做一些高可用优化，换成主从模式集群。害怕一个主节点挂掉没法用，那就做双主模式集群，也就是两个Mysql实例都能单独的生产自增ID。

那这样还会有个问题，两个MySQL实例的自增ID都从1开始，**会生成重复的ID怎么办？**

**解决方案**：设置`起始值`和`自增步长`

**优点：**

- 解决DB单点问题

**缺点：**

- 不利于后续扩容，而且实际上单个数据库自身压力还是大，依旧无法满足高并发场景。

##### 5、基于Redis模式

`Redis`也同样可以实现，原理就是利用`redis`的 `incr`命令实现ID的原子性自增。

用`redis`实现需要注意一点，要考虑到redis持久化的问题。`redis`有两种持久化方式`RDB`和`AOF`

- `RDB`会定时打一个快照进行持久化，假如连续自增但`redis`没及时持久化，而这会Redis挂掉了，重启Redis后会出现ID重复的情况。
  - `AOF`会对每条写命令进行持久化，即使`Redis`挂掉了也不会出现ID重复的情况，但由于incr命令的特殊性，会导致`Redis`重启恢复的数据时间过长。

##### 6、基于雪花算法（Snowflake）模式

Snowflake ID组成结构：`正数位`（占1比特）+ `时间戳`（占41比特）+ `机器ID`（占5比特）+ `数据中心`（占5比特）+ `自增值`（占12比特），总共64比特组成的一个Long类型。

- 第一个bit位（1bit）：Java中long的最高位是符号位代表正负，正数是0，负数是1，一般生成ID都为正数，所以默认为0。
- 时间戳部分（41bit）：毫秒级的时间，不建议存当前时间戳，而是用（当前时间戳 - 固定开始时间戳）的差值，可以使产生的ID从更小的值开始；41位的时间戳可以使用69年，(1L << 41) / (1000L *60* 60 *24* 365) = 69年
- 工作机器id（10bit）：也被叫做`workId`，这个可以灵活配置，机房或者机器号组合都可以。
- 序列号部分（12bit），自增值支持同一毫秒内同一个节点可以生成4096个ID

##### 8、美团（Leaf）

`Leaf`的snowflake模式依赖于`ZooKeeper`，不同于`原始snowflake`算法也主要是在`workId`的生成上，`Leaf`中`workId`是基于`ZooKeeper`的顺序Id来生成的，每个应用在使用`Leaf-snowflake`时，启动时都会都在`Zookeeper`中生成一个顺序Id，相当于一台机器对应一个顺序节点，也就是一个`workId`。



### 7.负载均衡算法

1.轮询法

轮询很容易理解，将请求按顺序轮流分配到后端服务器上，它均衡地对待后端每一台服务器，而不关心服务器的实际连接数和当前的系统负载。使用轮询策略的目的在于做到请求转移的绝对均衡，但付出的性能代价是相当大的。

2.随机法

通过系统随机函数，根据后端服务器列表大小的值随机选取其中的一台进行访问。有概率统计理论可以得知，随着调用量的增大，其实际效果越来越接近于平均分配流量到每一台后端服务器，也就是轮询的结果。

3.源地址哈希法

源地址哈希的思想是获取客户端访问的IP地址值，通过哈希函数计算得到一个数值，用该数值对服务器列表大小进行取模运算，得到的结果便是要 访问的服务器序号。采用哈希法进行负载均衡，同一IP地址的客户端，当后端服务器列表大小不变时，它每次都会被映射到同一台后端服务器进行访问。根据此特性可以在服务消费者和服务提供者之间建立有状态的session 会话。

4.加权轮询法

不同的后端服务器的配置和负载并不相同，因此它们的抗压能力也不尽相同。给配置高、负载低的机器设置更高的权重，让其处理更多的请求，而低配置、负载高的机器，则给其配置较低的权重，降低其系统负载，加权轮 询能很好地处理这一问题，并将请求顺序按权重分配给后端。

5.加权随机法

与加权轮询法类似，加权随机法也根据后端服务器不同的配置和负载情况，配置不同的权重。不同的是，它是按照权重来随机选取服务器的，而非顺序。

总结：前面我们费尽心思来实现服务消费者请求次数分配的均衡，我们知道这样做是没错的，可以为后端的多台服务器平均分配工作量，最大程度地提高服务器的利用率，但是，实际情况真的如此吗?在实际情况中，请求次数的均衡真的能代表系统的负载吗？我们必须认真地思考这一问题，从算法实施的角度来看，以后端服务器的视角来观察系统负载，而非请求发起方来观察。因此，我们得有其它的算法来实现可供选择，最小连接数法便属于此类算法。

 6.最小连接数法

最小连接数法比较灵活和智能，由于后端服务器的配置不尽相同，对于请求的处理有快有慢，它正是根据后端服务器当前的连接情况，动态地选取其中积压连接数最少的一台服务器来处理当前请求，尽可能提高后端服务器 的利用效率，将负载合理地分流到每一台机器。



### 8.分布式系统中的流量控制

#### 1.令牌桶

令牌桶算法最初来源于计算机网络。在网络传输数据时，为了防止网络拥塞，需限制流出网络的流量，使流量以比较均匀的速度向外发送。令牌桶算法就实现了这个功能，可控制发送到网络上数据的数目，并允许突发数据的发送。

令牌桶算法是网络流量整形（Traffic Shaping）和速率限制（Rate Limiting）中最常使用的一种算法。典型情况下，令牌桶算法用来控制发送到网络上的数据的数目，并允许突发数据的发送。

**大小固定的令牌桶可自行以恒定的速率源源不断地产生令牌。如果令牌不被消耗，或者被消耗的速度小于产生的速度，令牌就会不断地增多，直到把桶填满。后面再产生的令牌就会从桶中溢出。最后桶中可以保存的最大令牌数永远不会超过桶的大小。**

传送到令牌桶的数据包需要消耗令牌。不同大小的数据包，消耗的令牌数量不一样。

**令牌桶这种控制机制基于令牌桶中是否存在令牌来指示什么时候可以发送流量。令牌桶中的每一个令牌都代表一个字节。如果令牌桶中存在令牌，则允许发送流量；而如果令牌桶中不存在令牌，则不允许发送流量。因此，如果突发门限被合理地配置并且令牌桶中有足够的令牌，那么流量就可以以峰值速率发送。**

- 假如用户配置的平均发送速率为r，则每隔1/r秒一个令牌被加入到桶中（每秒会有r个令牌放入桶中）；
- 假设桶中最多可以存放b个令牌。如果令牌到达时令牌桶已经满了，那么这个令牌会被丢弃；
- 当一个n个字节的数据包到达时，就从令牌桶中删除n个令牌（不同大小的数据包，消耗的令牌数量不一样），并且数据包被发送到网络；
- 如果令牌桶中少于n个令牌，那么不会删除令牌，并且认为这个数据包在流量限制之外（n个字节，需要n个令牌。该数据包将被缓存或丢弃）；
- 算法允许最长b个字节的突发，但从长期运行结果看，数据包的速率被限制成常量r。对于在流量限制外的数据包可以以不同的方式处理：（1）它们可以被丢弃；（2）它们可以排放在队列中以便当令牌桶中累积了足够多的令牌时再传输；（3）它们可以继续发送，但需要做特殊标记，网络过载的时候将这些特殊标记的包丢弃。




## Zookeeper

### 1.Zookeeper 的使用场景

发布与订阅模型，即所谓的配置中心，顾名思义就是发布者将数据发布到ZK节点上，供订阅者动态获取数据，实现配置信息的集中式管理和动态更新。例如全局的配置信息，服务式服务框架的服务地址列表等就非常适合使用。

这里说的负载均衡是指软负载均衡。在分布式环境中，为了保证高可用性，通常同一个应用或同一个服务的提供方都会部署多份，达到对等服务。而消费者就须要在这些对等的服务器中选择一个来执行相关的业务逻辑，其中比较典型的是消息中间件中的生产者，消费者负载均衡。命名服务也是分布式系统中比较常见的一类场景。

在分布式系统中，通过使用命名服务，客户端应用能够根据指定名字来获取资源或服务的地址，提供者等信息。被命名的实体通常可以是集群中的机器，提供的服务地址，远程对象等等——这些我们都可以统称他们为名字（Name）。其中较为常见的就是一些分布式服务框架中的服务地址列表。通过调用ZK提供的创建节点的API，能够很容易创建一个全局唯一的path，这个path就可以作为一个名称。 ZooKeeper中特有watcher注册与异步通知机制，能够很好的实现分布式环境下不同系统之间的通知与协调，实现对数据变更的实时处理。使用方法通常是不同系统都对ZK上同一个znode进行注册，监听znode的变化（包括znode本身内容及子节点的），其中一个系统update了znode，那么另一个系统能够收到通知，并作出相应处理集群管理与Master选举

### 2.Zookeeper 怎么实现分布式锁

大家都是上来直接创建一个锁节点下的一个接一个的临时顺序节点如果自己不是第一个节点，就对自己上一个节点加监听器只要上一个节点释放锁，自己就排到前面去了，相当于是一个排队机制。而且用临时顺序节点的另外一个用意就是，如果某个客户端创建临时顺序节点之后，不小心自己宕机了也没关系，zk感知到那个客户端宕机，会自动删除对应的临时顺序节点，相当于自动释放锁，或者是自动取消自己的排队。

### 3.Zookeeper 怎么保证数据的一致性

消息广播阶段Leader 节点接受事务提交，并且将新的 Proposal 请求广播给 Follower 节点，收集各个节点的反馈，决定是否进行 Commit，在这个过程中，也会使用上一课时提到的 Quorum 选举机制。崩溃恢复阶段如果在同步过程中出现 Leader 节点宕机，会进入崩溃恢复阶段，重新进行 Leader 选举，崩溃恢复阶段还包含数据同步操作，同步集群中最新的数据，保持集群的数据一致性。整个 ZooKeeper 集群的一致性保证就是在上面两个状态之前切换，当 Leader 服务正常时，就是正常的消息广播模式；当 Leader 不可用时，则进入崩溃恢复模式，崩溃恢复阶段会进行数据同步，完成以后，重新进入消息广播阶段。

### 4.ZAB 协议的原理

zab协议包含两种基本模式：

1，崩溃恢复2，原子广播

1，崩溃恢复：假如说现在zookeeper及zk的集群中有3台机器，一台leader，两台flower。如果说leader挂了，这时候集群内要重新选举出leader，原来的leader变成了flower。新选出来的leader是根据什么来的呢，那就是zxid，集群内zxid最大的就是新的leader。zk集群还要有连个原则：

1，已经被处理的消息不能丢：当leader 收到合法数量 follower 的 ACKs 后，就向各个 follower 广播 COMMIT 命令，同时也会在本地执行COMMIT 并向连接的客户端返回「成功」。但是如果在各个 follower 在收到 COMMIT 命令前 leader 就挂了，导致剩下的服务器并没有执行都这条消息。这时候zab协议就要确保这条消息请求在集群内的所有机器都能提交commit成功了。

2，被丢弃的消息不能再次出现: 当leader收到了客户端的请求就挂了，只是leader机器本机做了commit，还没有发给其他机器做提交动作。这时候根据zxid重新选举出来了leader，原来的leader变成了flower，新的leader会通知这个flower把刚才那个请求删掉(数据同步时会做的。)。

2：原子广播 ：消息广播的过程实际上是一个简化版本的二阶段提交过程(2pc),leader 接收到消息请求后，将消息赋予一个全局唯一的zxid。leader 为每个 follower 准备了一个 FIFO 队列（通过 TCP协议来实现，以实现了全局有序这一个特点）将带有 zxid的消息作为一个提案（ proposal ）分发给所有的 follower。当 follower 接收到 proposal ，先把 proposal 写到磁盘，写入成功以后再向 leader 回复一个 ack。当 leader 接收到合法数量（超过半数节点）的 ACK 后，leader 就会向这些 follower 发送 commit 命令，同时会在本地执行该消息。当 follower 收到消息的 commit 命令以后，会提交该消息。过半的flower发回了ack，这时候leader就会commit。

### Zookeeper 遵循 CAP 中的哪些

在此ZooKeeper保证的是CP分析：可用性（A:Available）不能保证每次服务请求的可用性。任何时刻对ZooKeeper的访问请求能得到一致的数据结果，同时系统对网络分割具备容错性；但是它不能保证每次服务请求的可用性（注：也就是在极端环境下，ZooKeeper可能会丢弃一些请求，消费者程序需要重新请求才能获得结果）。所以说，ZooKeeper不能保证服务可用性。进行leader选举时集群都是不可用。在使用ZooKeeper获取服务列表时，当master节点因为网络故障与其他节点失去联系时，剩余节点会重新进行leader选举。问题在于，选举leader的时间太长，30 ~ 120s, 且选举期间整个zk集群都是不可用的，这就导致在选举期间注册服务瘫痪，虽然服务能够最终恢复，但是漫长的选举时间导致的注册长期不可用是不能容忍的。所以说，ZooKeeper不能保证服务可用性。

### 6.Zookeeper 和 Eureka 的区别

eureka优先保证可用性。在Eureka平台中，如果某台服务器宕机，Eureka不会有类似于ZooKeeper的选举leader的过程；客户端请求会自动切换 到新的Eureka节点；当宕机的服务器重新恢复后，Eureka会再次将其纳入到服务器集群管理之中；而对于它来说，所有要做的无非是同步一些新的服务 注册信息而已。所以，再也不用担心有“掉队”的服务器恢复以后，会从Eureka服务器集群中剔除出去的风险了。 Eureka可以很好的应对因网络故障导致部分节点失去联系的情况，而不会像zookeeper那样使整个注册服务瘫痪。Eureka作为单纯的服务注册中心来说要比zookeeper更加“专业”，因为注册服务更重要的是可用性，我们可以接受短期内达不到一致性的状况。

### 7.Zookeeper 的 Leader 选举

ZooKeeper状态的每次变化都接收一个ZXID（ZooKeeper事务id）形式的标记。ZXID是一个64位的数字，由Leader统一分配，全局唯一，不断递增。ZXID展示了所有的ZooKeeper的变更顺序。每次变更会有一个唯一的zxid，如果zxid1小于zxid2说明zxid1在zxid2之前发生。在集群初始化阶段，当有一台服务器ZK1启动时，其单独无法进行和完成Leader选举，当第二台服务器ZK2启动时，此时两台机器可以相互通信，每台机器都试图找到Leader，于是进入Leader选举过程。选举过程开始，过程如下：　　

(1) 每个Server发出一个投票。由于是初始情况，ZK1和ZK2都会将自己作为Leader服务器来进行投票，每次投票会包含所推举的服务器的myid和ZXID，使用(myid, ZXID)来表示，此时ZK1的投票为(1, 0)，ZK2的投票为(2, 0)，然后各自将这个投票发给集群中其他机器。　　

(2) 接受来自各个服务器的投票。集群的每个服务器收到投票后，首先判断该投票的有效性，如检查是否是本轮投票、是否来自LOOKING状态的服务器。　　

(3) 处理投票。针对每一个投票，服务器都需要将别人的投票和自己的投票进行比较，规则如下　　　　· 优先检查ZXID。ZXID比较大的服务器优先作为Leader。　　　　· 如果ZXID相同，那么就比较myid。myid较大的服务器作为Leader服务器。　　对于ZK1而言，它的投票是(1, 0)，接收ZK2的投票为(2, 0)，首先会比较两者的ZXID，均为0，再比较myid，此时ZK2的myid最大，于是ZK2胜。ZK1更新自己的投票为(2, 0)，并将投票重新发送给ZK2。　　

(4) 统计投票。每次投票后，服务器都会统计投票信息，判断是否已经有过半机器接受到相同的投票信息，对于ZK1、ZK2而言，都统计出集群中已经有两台机器接受了(2, 0)的投票信息，此时便认为已经选出ZK2作为Leader。　　

(5) 改变服务器状态。一旦确定了Leader，每个服务器就会更新自己的状态，如果是Follower，那么就变更为FOLLOWING，如果是Leader，就变更为LEADING。当新的Zookeeper节点ZK3启动时，发现已经有Leader了，不再选举，直接将直接的状态从LOOKING改为FOLLOWING。(1) 变更状态。Leader挂后，余下的非Observer服务器都会讲自己的服务器状态变更为LOOKING，然后开始进入Leader选举过程。　　(2) 每个Server会发出一个投票。在运行期间，每个服务器上的ZXID可能不同，此时假定ZK1的ZXID为124，ZK3的ZXID为123；在第一轮投票中，ZK1和ZK3都会投自己，产生投票(1, 124)，(3, 123)，然后各自将投票发送给集群中所有机器。　　(3) 接收来自各个服务器的投票。与启动时过程相同。　　(4) 处理投票。与启动时过程相同，由于ZK1事务ID大，ZK1将会成为Leader。　　(5) 统计投票。与启动时过程相同。　　(6) 改变服务器的状态。与启动时过程相同。
