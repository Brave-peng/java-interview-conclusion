### spring

#### 1.springIOC相关

##### 1.**Spring IoC 的容器构建流程**

1. prepareRefresh()，在刷新前有一些预处理。

2. 获取一个beanFactory.（一般为ApplicationContext，beanFactory比较少，因为前者是后者的增强版）,这里在获取beanFactory的时候，一般会有前置的预处理和后置的一些处理。
3. 实例化和调用BeanFactoryPostProcessor方法，这个方法是个比较重要的扩展点，一些扩展会在这里增强。
4. 初始化MessageSource组件，以此做一些消息绑定，消息解析的工作。
5. 实例化组件
   包括自定义加入容器的组件，各种后置通知，监听器，增强器，beanFactory等等
6. 刷新容器
   完成容器刷新，推送上下文刷新完毕事件到监听器。

![img](https://filescdn.proginn.com/e58214bf96c75cd435847ededc272c61/a59f9c06cbad43f4106852ba0bc2b18b.webp)

- Spring是怎么将这些bean初始化到IOC容器的呢？

初始化bean工厂 > 通过BeanDefinitionReader从不同的来源读取配置文件 > 执行针对bean工厂本身的BeanFactoryPostProcessor > 通过反射递归实例化bean(过程中调用BeanPostProcessor) > 建立整个应用上下文 > 一些后续工作(清除缓存，发布事件之类)

- IOC容器具体是什么呢？

**所谓IOC容器，其实就是一个Map<String，Object>的数据结构**



##### 2.spring IOC 定义

IOC是指在程序开发过程中，对象实例的创建不再由调用者管理，而是由Spring容器创建，Spring容器会负责控制程序之间的关系，而不是由代码直接控制，因此，控制权由程序代码转移到了Spring容器，控制权发生了反转，即控制反转。 Spring IOC提供了两种IOC容器，分别是BeanFactory和ApplicationContext。

**ApplicationContext是BeanFactory的子接口，也被称为应用上下文，不仅提供了BeanFactory的所有功能，还添加了对国际化、资源访问、事件传播等方面的支持。**

 Spring容器在创建被调用实例时，会自动将调用者需要的对象实例注入为调用者，这样，通过 Spring容器获得被调用者实例，成为依赖注入。





##### 3.依赖注入的方式

- 属性setter注入

- 构造器注入

- 通过接口实现依赖注入
  - 相对而言，接口的使用比较少




##### 4.Bean的会话状态

有状态会话的Bean：每个用户有自己特有的一个实例，在用户的生存期内，bean保持用户的信息，即"有状态"，一旦用户灭亡(调用结束或实例结束)，bean的生命周期也会结束。

无状态会话的Bean：bean一旦被实例化就被加到会话池中，各个用户可以公用，即使用户死亡，bean的生命周期也不一定结束，他可能依然存在于会话池中，供其他用户使用，由于没有特定的用户，也就没办法保存用户的状态，所以叫无状态Bean，但无状态Bean并非没有状态，如果它有自己的属性，那么这些属性就会受到所有调用它的用户的影响。



##### 5.Servlet的线程安全问题

Servlet体系结构是建立在Java多线程机制上的，它的生命周期是由web容器负责的，一个Servlet类在Application中只有一个实例存在，也就是有多个线程在同时使用这个实例，这是单例模式的使用。Servlet本身是无状态的，无状态的单例是线程安全的，但是如果在Servlet中使用了成员变量，那么就会变成有状态了，是非线程安全的。

所以，保证servlet线程安全的方法，关键在于成员变量和外部变量的使用，应尽量不使用成员变量，尽量不对外部的引用对象进行修改，如果在servlet中定义了成员变量，对其访问应加锁，或者选择线程安全的实现类；同理，对外部引用对象也是如此。



##### 6.Spring中Bean的作用域

注：在Spring配置文件中，使用的scope属性设置Bean的作用域。

Bean的作用域一共有五种：

1、singleton：单例，默认的作用域

2、prototype：原型，每次使用（注入或调用getBean()方法都会new一个新的对象，一旦被使用或者注入，spring就不再只有Bean的引用，清楚bean并释放资源是调用者的职责；

3、request：请求作用域，针对每一次HTTP请求都会产生一个新的bean，仅适用于WebApplicationContext环境

4、session：会话作用域，每次一次新的会话都会产生一个新的对象

5、globalSession：global session为整个HTTP请求中，在作用域方面就是application；

globalSession仅仅在基于portlet的web应用中才有意义。因为只有Portlet规范定义了全局Session的概念。





##### 7.Spring Bean的生命周期



简单来说，一个Bean的生命周期分为四个阶段：

1. 创建前准备阶段
   - 这个阶段主要是在开始Bean加载之前，从Spring上下文和相关配置中解析并查找Bean有关的配置内容。

2. 创建实例阶段
   - 这个阶段主要是通过反射来创建Bean的实例对象，并且扫描和解析Bean声明的一些属性。

3. 依赖注入阶段
   - 在这个阶段，会检测被实例化的Bean是否存在其他依赖，如果存在其他依赖，就需要对这些被依赖Bean进行注入。

4. 容器缓存阶段

   - 容器缓存阶段主要是把Bean保存到IoC容器中缓存起来，到了这个阶段，Bean就可以被开发者使用了。

   - 这个阶段涉及到的操作，常见的有，`init-method`这个属性配置的方法，会在这个阶段调用。

     在比如BeanPostProcessors方法中的后置处理器方法如：postProcessAfterInitialization，也是在这个阶段触发的。

5. 销毁实例阶段
   - 这个阶段，是完成Spring应用上下文关闭时，将销毁Spring上下文中所有的Bean。



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



##### 11.*spring的循环依赖以及解决方案

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





##### 12.*为什么三级缓存

如果 Spring 选择二级缓存来解决循环依赖的话，那么就意味着所有 Bean 都需要在实例化完成之后就立马为其创建代理，而 Spring 的设计原则是在 Bean 初始化完成之后才为其创建代理。所以，Spring 选择了三级缓存。但是因为循环依赖的出现，导致了 Spring 不得不提前去创建代理，因为如果不提前创建代理对象，那么注入的就是原始对象，这样就会产生错误。





##### 13.BeanFactory 和 FactoryBean 的区别

BeanFactory是个Factory，也就是IOC容器或对象工厂，FactoryBean是个Bean。在Spring中，**所有的Bean都是由BeanFactory(也就是IOC容器)来进行管理的**。

关于FactoryBean: 当我们使用第三方框架或者库时，有时候是无法去new一个对象的，比如静态工厂，对象是不可见的，只能通过getInstance（）之类方法获取，此时就需要用到FactoryBean,通过实现FactoryBean接口的bean重写getObject（）方法，返回我们所需要的bean对象。



##### 14.**Spring 能解决构造函数循环依赖吗**

答案是不行的，对于使用构造函数注入产生的循环依赖，Spring 会直接抛异常。

为什么无法解决构造函数循环依赖？

上面解决逻辑的第一句话：“首先使用构造函数创建一个 “不完整” 的 bean 实例”，从这句话可以看出，构造函数循环依赖是无法解决的，因为当构造函数出现循环依赖，我们连 “不完整” 的 bean 实例都构建不出来。





##### 15**@PostConstruct 修饰的方法里用到了其他 bean 实例，会有问题吗**

1、PostConstruct 注解被封装在 CommonAnnotationBeanPostProcessor中，具体触发时间是在 postProcessBeforeInitialization 方法，从 doCreateBean 维度看，则是在 initializeBean 方法里，**属于初始化 bean 阶段。**

**2、**属性的依赖注入是在 populateBean 方法里，**属于属性填充阶段。**

3、**属性填充阶段位于初始化之前，所以本题答案为没有问题。**





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

2、在两个不同的spring配置文件中，可以存在id相同的两个bean，启动时，不会报错。这是因为spring ioc容器在加载bean的过程中，类DefaultListableBeanFactory会对id相同的bean进行处理：**后加载的配置文件的bean，覆盖先加载的配置文件的bean。**DefaultListableBeanFactory类中，有个属性allowBeanDefinitionOverriding，默认值为true，该值就是用来指定出现两个bean的id相同的情况下，如何进行处理。如果该值为false，则不会进行覆盖，而是抛出异常。



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

因为Java中不支持多继承,而JDK的动态代理在创建代理对象时,默认让代理对象继承了Proxy类,所以JDK只能通过接口去实现动态代理。





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



##### 8.（等待编写）谈谈Spring的事务



#### 3. springSecurity原理

![640?wx_fmt=png](https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_png/GvtDGKK4uYngjiaicZ6UqibkEHRhzUg8JYLz2G6ILGaaibJe3fOl7LSDyBmHFJy3wdJqmVKdYKUYxQALGibSef9QJRA/640?wx_fmt=png)

客户端发起一个请求，进入 Security 过滤器链。

当到 LogoutFilter 的时候判断是否是登出路径，如果是登出路径则到 logoutHandler ，如果登出成功则到 logoutSuccessHandler 登出成功处理，如果登出失败则由 ExceptionTranslationFilter ；如果不是登出路径则直接进入下一个过滤器。

当到 UsernamePasswordAuthenticationFilter 的时候判断是否为登录路径，如果是，则进入该过滤器进行登录操作，如果登录失败则到 AuthenticationFailureHandler 登录失败处理器处理，如果登录成功则到 AuthenticationSuccessHandler 登录成功处理器处理，如果不是登录请求则不进入该过滤器。

当到 FilterSecurityInterceptor 的时候会拿到 uri ，根据 uri 去找对应的鉴权管理器，鉴权管理器做鉴权工作，鉴权成功则到 Controller 层否则到 AccessDeniedHandler 鉴权失败处理器处理。