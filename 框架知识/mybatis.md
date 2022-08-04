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



#### 4.Mapper的方法可以被重载吗

不可以