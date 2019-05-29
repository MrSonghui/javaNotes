# Spring: (一) -- 春雨润物之 IOC 

​	作为一个Java人，想必都或多或少的了解过Spring。对于其优势也能道个一二，诸如方便解耦、支持AOP编程、支持声明式事务、方便测试等等。Spring也不仅仅局限于服务器端开发，它可以做非常多的事情，任何Java应用都可以在简单性、可测试性和松耦合等方面从Spring中受益。Spring丰富功能的底层都依赖于它的两个核心特性：

- 控制反转 IOC (Inversion Of Control)
- 面向切面编程 AOP (Aspect-Oriented Programming)

控制反转指的是应用中的对象依赖关系不在由自己维护，而交给Spring由它的容器帮我们维护，因此也叫做依赖注入DI (Dependency Injection)。

## 一.使用BeanFactory解耦

​	这里使用BeanFactory来降低我们熟知的MVC编程模式中service层与dao层之间的耦合关系。

##### 解耦前（service层关键代码）

```java
// Service层中需要Dao层的实例来与数据库交互完成业务逻辑
public class UserServiceImpl implements UserService {
    private UserDao userDao = new UserDaoImpl();
    public void registerUser(User user) {
        userDao.addUser(user);
    }
```

​	可以看出，如果我们不做解耦操作，那么Service层中强烈依赖UserDao的实现类UserDaoImpl（即如果不new UserDaoImpl()，Service层将寸步难行）。

##### 解耦后（service层关键代码）

```java
public class UserServiceImpl implements UserService {
    private UserDao userDao;// 提供set方法
    public void setUserDao(UserDao userDao) {
        this.userDao = userDao;
    }
    public void registerUser(User user) {
        userDao.addUser(user);
    }
```

##### BeanFactory

```java
public class BeanFactory {
    private static Map<String, Object> beans = new HashMap<String, Object>();
    //静态代码块加载资源
    static {
        try {
            ResourceBundle bundle = ResourceBundle.getBundle("objects");
            Enumeration<String> keys = bundle.getKeys();
            while (keys.hasMoreElements()) {
                String key = keys.nextElement();
                String className = bundle.getString(key);
                Object clazz = Class.forName(className).newInstance();
                beans.put(key, clazz);
            }
        } catch (Exception e) {
            e.printStackTrace();
            throw new RuntimeException("加载类配置文件出错!");
        }
    }
	//对外提供获取bean的方法
    public static <T> T getBean(String className, Class<T> T) {
        Object o = beans.get(className);
        if (o != null)
            return (T) o;
        else throw new RuntimeException("找不到类:" + className);
    }
}
```

##### objects.properties

```pro
userDao=com.dintalk.dao.impl.UserDaoImpl
```

##### 为UserServiceImpl实例注入依赖

```java
UserServiceImpl userServiceImpl = new UserServiceImpl();
UserDao userDao = BeanFactory.getBean("userDao",UserDao.class);
userServiceImpl.setUserDao(userDao);
```

##### 总结：

​	解耦前，service层中直接new出了其所依赖的实例对象userDaoImpl。而通过工厂解耦后，service中只声明了UserDao的接口引用，并提供了set方法，我们在使用servcie时，可以通过set方法传入从工厂中获得的实现了UserDao接口的任一实现类的实例。而实现类的配置又暴露在了配置文件当中，解耦的同时也增加了程序的动态性。

##### BeanFactory原理：

​	这里使用的是静态工厂，在工厂类中定义了一个Map用于存放工厂管理的Bean实例，静态代码块随类的加载执行一次，读取配置文件中的key-value信息。通过循环和反射，将配置文件中的key仍作为Map的key；将配置文件中key对应的类全限定名通过反射构造实例后作为其对应的value存于Map中。达到这样的效果：BeanFactory类加载完毕后，它便管理了一个Map集合，Map集合的key就是配置文件中的key，Map中的value就是配置文件中value对应的类的实例。如此，对外提供一个getBean方法，通过key返回其对应的实例，这便实现了通过BeanFactory来管理实例对象。

## 二.Spring使用步骤

以使用xml配置文件的方式示例：

##### 1.导入坐标或jar包

​	如果使用Maven构建，我们可以导入spring-context。因为上下文模块依赖其他模块，所有其他模块也会自动导入。

```xml
<dependency>
   <groupId>org.springframework</groupId>
   <artifactId>spring-context</artifactId>
   <version>5.0.2.RELEASE</version>
</dependency>
```

##### 2.创建applicationContext.xml文件并加入头信息

​	在resources下创建spring的主配置文件，添加头信息时需格外注意。最好保存模板或到官网复制粘贴，稍有差错将导致异常。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:p="http://www.springframework.org/schema/p"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans.xsd
           http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean>...</bean>
    <!-- 1.主配置文件中导入模块配置文件 -->
    <import resource="user.xml"/>
</beans>
```

##### 3.在配置文件中装配Bean并加载配置文件获取Bean

​	可以按分模块在配置文件中装配Bean，再在主配置文件中进行导入。但要注意，如果出现id相同的情况，后加载的配置会覆盖掉前面的配置！

###### 加载配置文件获取Bean

```java
ApplicationContext appContext = new 
    ClassPathXmlApplicationContext("applicationContext.xml");
UserDao userDao = appContext.getBean("userDao", UserDao.class);
```

**Tips:** Spring有自己的容器，需要读取配置文件装配好Bean后放入自己的容器，我们在用时直接找容器获取即可！如果分模块配置了但没有在主文件中导入其他文件也可以在加载配置文件时一块加载：

```java
ApplicationContext appContext = new 
    ClassPathXmlApplicationContext(new String[] {"applicationContext.xml","user.xml"});
```

## 三.IOC配置（Bean的装配方式）

​	Spring的一大特点是最小侵入性编程，它不会强迫我们去实现它的接口或实现类，POJO依旧是那个POJO。我们只是将依赖交由Spring管理，因此，IOC配置也就是Bean的装配便是很大一部分工作。Spring为我们提供了三种装配Bean的方式：

- 基于xml配置文件        ★★★★
- 基于注解（往往配合xml配置文件使用）      ★★★★
- 基于java类的配置(会用到注解）

**其实，无论使用哪一种方式，我们的目的只有一个，那就是我们要将程序中的依赖关系描述清楚，将Bean装配好交由Spring的容器！**

#### 1.基于xml文件的装配

##### bean的实例化

```xml
<!-- 0.通过默认构造方法生产bean -->
<bean id="userDao" class="cn.dintalk.dao.impl.UserDaoImpl"></bean>
<!-- 1.通过实例工厂生产bean -->
<bean id="myBeanFactory" class="cn.dintalk.factory.BeanFactory1"/>
<bean id="myBean" factory-bean="myBeanFactory" factory-method="getBean">
<!-- 2.通过静态工厂生产bean -->
<bean id="userDao1" class="cn.dintalk.factory.BeanFactory" factory-method="getBean"/>

<!-- bean的存活范围及生命周期方法 -->
<bean id="userDao2" scope="singleton" init-method="m1" destroy-method="m2" 
class="cn.dintalk.dao.impl.UserDaoImpl"></bean>  
```

###### **Tips:** scope可选值：

- singleton

- prototype

- request

- session

- globalsession

  生命周期方法在单例模式下才有意义，想想是为什么呢？

##### 数据的注入

```xml
<!-- 3.数据的注入 -->
<!-- 3.1构造方法注入 -->
<bean id="user" class="cn.dintalk.domain.User">
    <constructor-arg index="0" value="王舞"/>
    <constructor-arg index="1" value="wangwu"/>
</bean>
<!-- 3.2setter属性注入 -->
<bean id="user1" class="cn.dintalk.domain.User">
    <property name="name" value="赵思"/>
    <property name="password" value="zhaosi"/>
</bean>
<!-- 3.3p命名空间注入 -->
<bean id="user2" class="cn.dintalk.domain.User" p:name="张珊" p:password="zhangshan"/>

<!-- 4.常用数据类型的注入 -->
<bean id="user3" class="cn.dintalk.domain.User">
    <!-- 4.0数组的注入 -->
    <property name="myArr">
        <array>
            <value>str1</value>
            <value>str2</value>
        </array>
    </property>
    <!-- 4.1List的注入 -->
    <property name="myList">
        <list>
            <value>str1</value>
            <value>str2</value>
        </list>
    </property>
    <!-- 4.2Set的注入 -->
    <property name="mySet">
        <set>
            <value>str1</value>
            <value>str2</value>
        </set>
    </property>
    <!-- 4.3Map的注入-->
    <property name="myMap">
        <map>
            <entry key="s1" value="str1"/>
            <entry key="s2" value="str2"/>
        </map>
    </property>
    <!-- 4.4Properties的注入 -->
    <property name="myPro">
        <props>
            <prop key="s1">str1</prop>
            <prop key="s2">str2</prop>
        </props>
    </property>
</bean>

<!-- 5.依赖的注入-->
<bean id="userService" class="cn.dintalk.service.impl.UserServiceImpl">
    <property name="userDao" ref="userDao"></property>
</bean>
```



#### 2.基于注解的装配

​	使用注解来装配bean可以简化我们的步骤，提高效率。可以替代xml文件的装配方式，但是一般是和xml文件的方式打双打。使用第三方工具包时使用xml的方式要方便一些，章节末我们通过DButil的示例。由于xml的方式比较好理解，而注解又是xml文件方式的简化，因此我们对比着来学习。

##### bean的实例化 

```java
@Component("accountService")
public class AccountServiceImpl implements AccountService{
/*
- @Controller      用在表现层
- @Service		  用在业务层
- @Respository	  用在持久层	
这三个注解的作用和@Component完全一样，就是更加语义化（分层）
*/
// bean的存活范围和生命周期
@Component("accountService")
@Scope("singleton")
public class AccountServiceImpl implements AccountService {
// 初始化方法
@PostConstruct
private void init(){
// 销毁方法  
@PreDestroy
private void destroy(){
```

##### 数据的注入

```java
@Autowired
@Qualifier("accountDao")
private AccountDao accountDao;
/*
- @Autowired   自动装配，查找Spring容器按照类型自动赋予对象。★★★    
- @Qualifier("accountDao") 与@Autowired配合，指定具体名称的实现类对象。★★★ 
- @Resource(name="accountDao") Spring对JSR-250中定义的注解的支持。
*/
// - @Value 注入简单类型的数据
@Value("16")    // 值都是字符串类型，spring会自行解析
private Integer age;
@Value("张珊")
private String name;
```

基于注解的简单类型（基本类型+String）数据注入意义不是很大，都在源码里面，和直接赋值区别不大。

##### 基于注解的配置加载（获取容器对象）

方式一：依旧使用ClassPathXmlApplicationContext（需配置）★★★

```xml
<!-- 配置文件中，指定扫描注解的包 -->
<context:component-scan base-package="cn.dintalk"/>
```

方式二：使用AnnotationConfigApplicationContext加载配置，获取bean

```java
ApplicationContext context = new 
    AnnotationConfigApplicationContext(MyBean.class);
 MyBean myBean = context.getBean("myBean", MyBean.class);
```

**Tips:**  我一般使用方式一的配置，但是要注意不要引错了头约束。

#### 3.基于java类的装配

​	基于注解的通过组件扫描和自动装配实现Spring的自动化配置是更为推荐的方式，但有时候自动化配置的方案行不通，因此需要明确配置Spring。同样比如，我们想将第三方库中的组件装配到我们的应用中，这种情况下，没有办法在它的类上添加@Component和@Autowired注解的。因此我们必须采用显示装配的方式，显示装配有两种可选方案：上述的xml装配方式和我们即将阐述的Java类的装配方式。还是那一句话，无论是哪一种方式，目的只有一个，那就是将一些必要的信息告知我们的程序。

##### bean的实例化

```java
@Configuration   //Spring配置类，带有Configuratio注解就是配置类.加不加无所谓
@ComponentScan("cn.dintalk")  //<context:component-scan base-package="cn.dintalk"/>
@Import({JdbcConfig.class,MailConfig.class})  //聚合多个配置类<import resource=""/>
public class SpringConfig {
    
/*
@Configuration可不加：因为我们在加载配置时还会指定到该类
    - ApplicationContext applicationContext =
                new AnnotationConfigApplicationContext(SpringConfig.class);
*/
    
@PropertySource("jdbc.properties")//导入外部的properties文件
public class JdbcConfig {
    //读取properties文件中key对应的value值
    @Value("${jdbc.driverClassName}")
    private String driverClassName;
    @Value("${jdbc.url}")
    private String url;
    @Value("${jdbc.username}")
    private String username;
    @Value("${jdbc.password}")
    private String password;
    //创建数据源
    //告知spring容器，将该方法的返回值对象，以“druidDataSource”存放到容器中
    @Bean("druidDataSource")
    public DataSource createDataSource(){
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setDriverClassName(driverClassName);
        dataSource.setUrl(url);
        dataSource.setUsername(username);
        dataSource.setPassword(password);
        return dataSource;
    }
    //创建QueryRunner对象，并交给spring容器管理
    @Bean("queryRunner")
    //@Qualifier("druidDataSource") DataSource dataSource:
    //数据源对象对应spring容器中一个名字叫做druidDataSource的
    public QueryRunner createQueryRunner(@Qualifier("druidDataSource") 
                                         DataSource dataSource){
        QueryRunner queryRunner = new QueryRunner(dataSource);
        return queryRunner;
    }
```

##### 数据的注入

参考同基于注解的装配

##### 基于java类的配置加载

```java
//AnnotationConfigApplicationContext构造参数：指定配置类的类型
//可以指定多个
ApplicationContext applicationContext =
        new AnnotationConfigApplicationContext(SpringConfig.class);
UserService userService = applicationContext.getBean("userService", UserService.class);
```

## 四.DBUtils的使用

​	DBUtils是Apache提供的对JDBC封装了的公共组件。

#### 1.普通的使用

##### 第一步：导入jar包或Maven坐标

```xml
<dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <version>5.1.46</version>
  </dependency>
  <dependency>
      <groupId>commons-dbutils</groupId>
      <artifactId>commons-dbutils</artifactId>
      <version>1.7</version>
  </dependency>
  <dependency>
      <groupId>com.alibaba</groupId>
      <artifactId>druid</artifactId>
      <version>1.0.14</version>
  </dependency>
```

##### 第二步：创建工具类

```java
public class DruidUtil {
    private static DataSource dataSource;
    static {
        InputStream inputStream = null;
        try {
            inputStream = DruidUtil.class.getClassLoader()
                    .getResourceAsStream("jdbc.properties");
            Properties properties = new Properties();
            properties.load(inputStream);
            dataSource = DruidDataSourceFactory.createDataSource(properties);
        } catch (Exception e) {
            e.printStackTrace();
            throw new RuntimeException("加载数据库配置文件失败！");
        }finally {
            if (inputStream != null){
                try {
                    inputStream.close();
                } catch (IOException e) {
                    e.printStackTrace();
                    throw new RuntimeException("关闭文件资源失败!");
                }
            }
        }
    }
    public static DataSource getDataSource(){ // 获取数据源
        return dataSource;
    }
    public Connection getConnection(){ // 获取连接
        try {
            return dataSource.getConnection();
        } catch (SQLException e) {
            e.printStackTrace();
            throw new RuntimeException(e);
        }
    }
}
```

##### 第三步：CRUD操作(DAO层)

```java
private QueryRunner queryRunner = new QueryRunner(DruidUtil.getDataSource());
// 增删改： 使用update（sql,params）;
//update方法内部：先从给定的数据源获取一个连接，在方法即将执行完毕后，将连接归还(到连接池)
public void addAccount(Account account) {
        if (account == null)
            throw new RuntimeException("参数错误");
        try {
            queryRunner.update("insert into accounts values(null,?,?)",
                    account.getAccountName(), account.getBalance());
        } catch (SQLException e) {
            throw new RuntimeException(e);
        }
    }
//查询：使用 query(sql,Handler,params)
 public Account findById(Integer aid) {
        if (aid == null)
            throw new RuntimeException("参数异常");
        try {
            return queryRunner.query("select * from accounts where aid = ?", new
                    BeanHandler<Account>(Account.class), aid);
        } catch (SQLException e) {
            throw new RuntimeException(e);
        }
    }
```

#### 2.使用Spring基于xml进行解耦

##### 第一步：导入Spring的jar包或Maven坐标

```xml
<dependency>
   <groupId>org.springframework</groupId>
   <artifactId>spring-context</artifactId>
   <version>5.0.2.RELEASE</version>
</dependency>
```

##### 第二步：创建applicationContext.xml文件并进行配置

```xml
<!-- 1.配置druid数据源 -->
<bean id="druidDataSource" class="com.alibaba.druid.pool.DruidDataSource">
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    <property name="url" value="jdbc:mysql:///spring02"/>
    <property name="username" value="sh"/>
    <property name="password" value="sh123"/>
</bean>
<!-- 2.配置QueryRunner -->
<bean id="queryRunner" class="org.apache.commons.dbutils.QueryRunner">
    <constructor-arg index="0" ref="druidDataSource"/>
</bean>
<!-- 3.配置AccountDao -->
<bean id="accountDao" class="cn.dintalk.dao.impl.AccountDaoImpl">
    <property name="queryRunner" ref="queryRunner"/>
</bean>
<!-- 4.配置AccountService -->
<bean id="accountService" class="cn.dintalk.service.impl.AccountServiceImpl">
    <property name="accountDao" ref="accountDao"/>
</bean>
```

##### 第三步：CRUD操作

```java
//DAO层 提供set方法以注入
private QueryRunner queryRunner;
public void setQueryRunner(QueryRunner queryRunner) {
    this.queryRunner = queryRunner;
}

//service层 提供set方法以注入
private AccountDao accountDao;
public void setAccountDao(AccountDao accountDao) {
    this.accountDao = accountDao;
}

//CRUD操作同上
```

**Tips:**  配置加载方式

```java
ApplicationContext appContext = new 
    ClassPathXmlApplicationContext("applicationContext.xml");
```

#### 3.使用Spring基于注解进行解耦

由于使用到第三方包，所以无法全部使用注解，需要和xml的方式结合。

##### 第一步：配置applicationContext文件

```xml
<!-- 1.基于注解,声明扫描注解的包 -->
<context:component-scan base-package="cn.dintalk"/>
<!-- 2.配置druid数据源 -->
<bean id="druidDataSource" class="com.alibaba.druid.pool.DruidDataSource">
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    <property name="url" value="jdbc:mysql:///spring02"/>
    <property name="username" value="sh"/>
    <property name="password" value="sh123"/>
</bean>
<!-- 3.配置QueryRunner -->
<bean id="queryRunner" class="org.apache.commons.dbutils.QueryRunner">
    <constructor-arg index="0" ref="druidDataSource"/>
</bean>
```

##### 第二步：添加注解

```java
// DAO层中
@Repository("accountDao")
public class AccountDaoImpl implements AccountDao {
    @Autowired
    private QueryRunner queryRunner;
    
// Service层中
@Service("accountService")
public class AccountServiceImpl implements AccountService {
    @Autowired
    @Qualifier("accountDao")
    private AccountDao accountDao;
```

##### 第三步：CRUD操作参上

**Tips:**  配置加载方式

```java
ApplicationContext appContext = new 
    ClassPathXmlApplicationContext("applicationContext.xml");
```

#### 4.使用Spring基于Java类进行解耦

##### 第一步：创建配置类

```java
@PropertySource("jdbc.properties")//导入外部的properties文件
@ComponentScan("cn.dintalk")  // 添加注解扫描包
public class SpringConfig {
    //读取properties文件中key对应的value值
    @Value("${jdbc.driverClassName}")
    private String driverClassName;
    @Value("${jdbc.url}")
    private String url;
    @Value("${jdbc.username}")
    private String username;
    @Value("${jdbc.password}")
    private String password;
    //创建数据源
    @Bean("druidDataSource")
    public DataSource createDataSource(){
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setDriverClassName(driverClassName);
        dataSource.setUrl(url);
        dataSource.setUsername(username);
        dataSource.setPassword(password);
        return dataSource;
    }
    //创建QueryRunner对象，并交给spring容器管理
    @Bean("queryRunner")
    //@Qualifier("druidDataSource") DataSource dataSource:
    数据源对象对应spring容器中一个名字叫做druidDataSource的
    public QueryRunner createQueryRunner(@Qualifier("druidDataSource") 
                                         DataSource dataSource){
        QueryRunner queryRunner = new QueryRunner(dataSource);
        return queryRunner;
    }
```

##### 第二步：添加注解

```java
// DAO层中
@Repository("accountDao")
public class AccountDaoImpl implements AccountDao {
    @Autowired
    private QueryRunner queryRunner;
    
// Service层中
@Service("accountService")
public class AccountServiceImpl implements AccountService {
    @Autowired
    @Qualifier("accountDao")
    private AccountDao accountDao;
```

##### 第三步：CRUD参上

**Tips:**  配置加载方式

```java
 ApplicationContext context = new 
     AnnotationConfigApplicationContext(SpringConfig.class);
```

## V.附录：常用文件约束头

##### applicationContext.xml文件头约束

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">
</beans>
```

##### 带有p命名空间的约束头

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:p="http://www.springframework.org/schema/p"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans.xsd
           http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans.xsd">
</beans>
```

##### 带有context命名空间的约束头

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       http://www.springframework.org/schema/context/spring-context.xsd">
</beans>
```

