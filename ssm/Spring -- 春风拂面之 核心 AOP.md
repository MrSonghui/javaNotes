# Spring -- 春风拂面之 核心 AOP

​	”万物皆对象“是面向对象编程思想OOP（Object Oriented Programming） 的最高境界。在面向对象中，我一直将自己（开发者）放在一个至高无上的位置上，可以操纵万物（对象），犹如一军统帅。那现在有一个问题，我的士兵除了作战之外还要吃饭，显然不可能让每一个士兵自己去解决吃饭问题。因此，军中有后勤部门，专门解决士兵作战之外的衣食等问题。简单的说，这就是一个切面问题，也就是我们今天讨论的重点**面向切面编程**AOP(Aspect Oriented Programming )。

## 一.关于AOP的理解

​	面向切面编程是一直流行在学术领域的编程思想，近几年在应用领域流行起来。AOP是对OOP的一个有益的补充，它关注具有横切逻辑的代码。简而言之，横切关注点可以被描述为影响应用多处的功能。例如：事务管理、日志记录、安全等等，应用中的很多方法都会涉及到这些。

​	在使用面向切面编程时，我们在一个地方定义通用功能，但是可以通过声明的方式定义这个功能要以何种方式在何处应用，而无需修改受影响的类。横切关注点可以被模块化为特殊的类，这些类被称为切面（aspect）。而我把它理解为像“抽屉''一样的插片，哪里需要插到哪里。如此，服务模块只包含核心功能，更加简洁，而次要关注点的代码被转移到切面中了（还可复用）。

​	因此，AOP是一种思想，而并非Spring独有的功能。Spring只支持方法级别的连接点，因为Spring基于动态代理，通过在代理类中包裹切面，在运行期把切面织入到Spring管理的bean中。代理类封装了目标类，并拦截被通知方法的调用，再把调用转发给真正的目标bean。当代理拦截到方法调用时，在调用目标bean方法之前，会执行切面逻辑。我们以转账业务中的事务控制为例剖析。

## 二.转账--事务控制

##### 需求：A、B两个账户之间进行转账，基于MVC进行编程。DAO层只负责CRUD，Service层负责业务逻辑处理（避免出现属于DAO层的依赖）。未控制事务时代码如下：

##### Dao层关键代码

```java
private QueryRunner queryRunner = new QueryRunner(DruidUtil.getDataSource());
//1 查找账户信息
public Account findAccountByAccountName(String accountName) {
	return queryRunner.query("select * from accounts where accountName = ?",
         new BeanHandler<Account>(Account.class), accountName);     
    }
//2.更新账户信息
public void updateAccount(Account account) {
   queryRunner.update("update accounts set balance=? where accountName=? ",
      account.getBalance(),account.getAccountName());
}
```

##### Service层关键代码

```java
private AccountDao accountDao = new AccountDaoImpl();
public void transfer(String sourceAccount, String targetAccount, Float money) {
  //1.查询到相关账户并进行业务处理
  Account sourAccout = accountDao.findAccountByAccountName(sourceAccount);
  Account targAccount = accountDao.findAccountByAccountName(targetAccount);
  sourAccout.setBalance(sourAccout.getBalance()-money);
  targAccount.setBalance(targAccount.getBalance()+money);
  //2.更新账户信息
  accountDao.updateAccount(sourAccout);
  int i = 1/0;  // 若出现异常，则发生事务问题
  accountDao.updateAccount(targAccount);
}
```

### 1.使用TransactionManager优化

##### 第一步：编写事务管理器

**Tips:** 使用TreadLocal进行线程绑定，实现在业务处理中使用同一个连接（即在同一个事务下）

```java
public class TransactionMannager {
    private static ThreadLocal<Connection> local=new ThreadLocal<Connection>();
	//1. 获取连接的方法
    public static Connection getConnection() {
        Connection connection = local.get();
        if (connection == null) {
            connection = DruidUtil.getConnection();
            local.set(connection);
        }
        return connection;
    }
    //2. 给TreadLocal绑定连接,开启事务
    public static void startTransaction() {
        try {
            Connection connection = getConnection();
            connection.setAutoCommit(false);
        } catch (SQLException e) {
            throw new RuntimeException("开启事务失败!");
        }
    }
    //3. 提交
    public static void commit() {
        try {
            local.get().commit();
        } catch (SQLException e) {
            throw new RuntimeException(e);
        }
    }
    //4. 回滚
	...
    //5. 关闭连接,移除绑定（避免下次获取到已经关闭的连接）
    public static void close() {
        try {
            local.get().close();
            local.remove();
        } catch (SQLException e) {
            throw new RuntimeException(e);
        }
    }
}
```

##### 第二步：使用事务管理器优化

###### Dao层关键代码

```java
private QueryRunner queryRunner = new QueryRunner(DruidUtil.getDataSource());
//1 查找账户信息
public Account findAccountByAccountName(String accountName) {
	return queryRunner.query(TransactionMannager.getConnection(),
              "select * from accounts where accountName = ?",
         new BeanHandler<Account>(Account.class), accountName);     
    }
//2.更新账户信息
public void updateAccount(Account account) {
   queryRunner.update(TransactionMannager.getConnection(),
      "update accounts set balance=? where accountName=? ",
      account.getBalance(),account.getAccountName());
```

###### Service层关键代码

```java
public class AccountServiceImpl implements AccountService {
    private AccountDao accountDao = new AccountDaoImpl();
    public void transfer(String sourceAccount, String targetAccount, Float money) {
        //1.查询到账户进行业务处理
        try {
            TransactionMannager.startTransaction();//开启事务
            Account sourAccout = accountDao.findAccountByAccountName(sourceAccount);
            Account targAccount = accountDao.findAccountByAccountName(targetAccount);
            sourAccout.setBalance(sourAccout.getBalance()-money);
            targAccount.setBalance(targAccount.getBalance()+money);
            //2.更新账户
            accountDao.updateAccount(sourAccout);
            int i = 1/0;
            accountDao.updateAccount(targAccount);
            TransactionMannager.commit();//提交事务
        } catch (Exception e) {
            TransactionMannager.rollBack();//回滚事务
        }finally {
            TransactionMannager.close();//关闭连接
        }
    }
```

### 2.基于静态代理优化

##### 第一步：编写静态代理类

```java
public class AccountServiceImplProxy {
    private AccountService accountService;
    public AccountServiceImplProxy(AccountService accountService) {
        this.accountService = accountService;
    }
    // 1.方法增强
    public void transfer(String sourceAccountName, String targetAccountName,Float money){
        try {
            TransactionMannager.startTransaction();//开启事务
            accountService.transfer(sourceAccountName,targetAccountName,money);
            TransactionMannager.commit();//提交事务
        } catch (Exception e) {
            TransactionMannager.rollBack();//回滚事务
        } finally {
            TransactionMannager.close();//关闭连接
        }
    }
}
```

##### 总结：

​	与使用TransactionManager进行事务管理基本一致，只不过事务控制的代码不写在Service层中，而写在其代理类中，代理类的唯一构造方法需要传入accountService对象。在代理类中对accountService的方法进行增强（添加事务控制）。

### 3.基于动态代理优化

##### 第一步：编写动态代理

```java
// 使用 Jdk的Proxy 动态代理
public void testTransfer1(){
    //创建被代理对象
    AccountService accountService = new AccountServiceImpl();
    //创建代理
    AccountService proxy =(AccountService) Proxy.newProxyInstance(AccountTest.class.getClassLoader(),
    accountService.getClass().getInterfaces(),new MyInvocationHandler(accountService));
    proxy.transfer("a","b",1f);
}
// Proxy 动态代理类的处理类
class MyInvocationHandler implements InvocationHandler{
    private AccountService accountService; //构造方法需要传入被代理对象
    public MyInvocationHandler(AccountService accountService){
        this.accountService = accountService;
    }
    
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        try {
            TransactionMannager.startTransaction();
            Object rtValue= method.invoke(accountService,args);
            TransactionMannager.commit();
            System.out.println("proxy动态代理执行了");
            return rtValue;
        }catch (Exception e){
            TransactionMannager.rollBack();
            throw new RuntimeException(e);
        }finally {
            TransactionMannager.close();
        }
    }
}
//-------------------------------------
// 使用Cg-lib 的动态代理
public void testTransfer2(){
    AccountServiceImpl accountService = new AccountServiceImpl();
    Enhancer proxy = new Enhancer();
    proxy.setSuperclass(AccountServiceImpl.class);//指定父类的类型
    proxy.setCallback(new MyInvocationHandler1(accountService));//增强策略
    AccountServiceImpl pro = (AccountServiceImpl) proxy.create();
    pro.transfer("a","b",2f);
}
// cglib 的动态代理处理类
class MyInvocationHandler1 implements net.sf.cglib.proxy.InvocationHandler{
    private AccountServiceImpl accountService;//构造方法需要传入被代理对象
    public MyInvocationHandler1(AccountServiceImpl accountService) {
        this.accountService = accountService;
    }
    
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        try {
            TransactionMannager.startTransaction();
            Object rtValue = method.invoke(accountService,args);
            TransactionMannager.commit();
            System.out.println("cglib动态代理执行了");
            return rtValue;
        }catch (Exception e){
            TransactionMannager.rollBack();
            throw new RuntimeException(e);
        }finally {
            TransactionMannager.close();
        }
    }
}
```

##### 总结：

​	Proxy代理通过实现和被增强类同样的接口，如果目标没有实现任何的接口，Proxy将无法使用。Cglib是基于子类的动态代理，生成的代理类是被代理类的子类。在spring中，框架会根据目标类是否实现了接口来决定采用哪种动态代理的方式。

- Proxy 编译时间短，运行效率低。
- Cglib 编译时间长，运行效率高。

## 三.Spring AOP及使用配置

### 1.AOP相关术语

##### Joinpoint（连接点）：

​	指那些被拦截到的点，在spring中，这些点指的是方法，因为spring只支持方法类型的连接点。

##### Pointcut（切入点）：

​	指我们要对哪些 Joinpoint进行拦截的定义。

##### Advice（通知/增强）：

​	指拦截到 Joinpoint之后要做的事情就是通知。通知的类型有：前置通知，后置通知，异常通知

​	最终通知，环绕通知。

##### Introduction（引介）：

​	引介是一种特殊的通知，在不修改类代码的前提下，Introduction可以在运行期为类动态地添加

​	一些方法或Field。	

##### Target（目标对象）：

​	代理的目标对象。

##### Weaving（织入）：

​	指把增强应用到目标对象来创建新的代理对象的过程。（spring采用动态代理织入，而AspectJ

​	采用编译期织入和类装载期织入。

##### Proxy（代理）：

​	一个类被AOP织入增强后，就产生一个结果代理类。

##### Aspect（切面）：

​	是切入点和通知（引介）的结合。

### 2.准备必要代码

##### 编写需要加入切面的类

```java
public class UserServiceImpl implements UserService {
    public void login() {
        System.out.println("用户的登录方法执行了");
    }

    public void findUser() {
        System.out.println("用户的查找方法执行了");
    }
}
```

##### 编写advice通知类（公用的增强代码）

```java
//class Logger
public void printLogger(){
   System.out.println("logger中的printLogger方法执行了");
}
public void afterPrintLogger(){
   System.out.println("logger中的afgerPrintLogger方法执行了");
}
```



### 3.基于xml的Spring AOP  ★★★

##### 第一步：导入jar包或Maven坐标

```xml
<dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-context</artifactId>
      <version>5.0.2.RELEASE</version>
</dependency>
<dependency>
      <groupId>org.aspectj</groupId>
      <artifactId>aspectjweaver</artifactId>
      <version>1.8.13</version>
</dependency>
```

##### 第二步：配置applicationContext.xml文件

```xml
<!-- 基于xml文件的aop -->
<!--
aop 的配置步骤：
第一步：把通知类的创建也交给 spring 来管理
第二步：使用 aop:config 标签开始 aop 的配置
第三步：使用 aop:aspect 标签开始配置切面，写在 aop:config 标签内部
	id 属性：给切面提供一个唯一标识
	ref 属性：用于引用通知 bean 的 id。
第四步：使用对应的标签在 aop:aspect 标签内部配置通知的类型
使用 aop:befored 标签配置前置通知，写在 aop:aspect 标签内部
method 属性：用于指定通知类中哪个方法是前置通知
pointcut 属性：用于指定切入点表达式。
切入点表达式写法：
关键字：execution(表达式) 表达式内容：全匹配标准写法：
访问修饰符 返回值 包名.包名.包名...类名.方法名(参数列表)
例如：
public void cn.dintalk.service.impl.AccountServiceImpl.saveAccount()
-->
<!-- 1.配置bean 及logger通知类 --> 
<bean id="logger" class="cn.dintalk.Aop.Logger"/>
<bean id="userService" class="cn.dintalk.Aop.service.impl.UserServiceImpl"/>

<!-- 2.配置Aop -->
<aop:config>  <!-- 注意顺序,顺序不对会报错 -->
    <!-- 2.1配置切入点 -->
    <aop:pointcut id="userServiceM" expression="execution(* cn.dintalk.Aop.service.impl.UserServiceImpl.*(..))"/>
    <!-- 2.2配置切面(增强类) -->
    <aop:aspect id="loggerAdvice" ref="logger">
        <aop:before method="printLogger" pointcut-ref="userServiceM"/>
        <aop:after method="afterPrintLogger" pointcut-ref="userServiceM"/>
    </aop:aspect>
</aop:config>
```

##### 总结：

​	如此，调用userService的方法，在目标方法执行前后会执行相应的通知。

##### 切入点表达式说明：

- ​     \*   可表示任意返回值和任意包（单个包）或任意类及任意方法名
- ​     ..    可表示当前包及其子包  或参数

通常情况下，我们都是对业务层的代码进行增强，所以切入点表达式都是切到业务层实现类：

```xml
execution(* cn.dintalk.service.impl.*.*(..))
```

##### 通知的常用类型

- aop:before
- aop:after-returning     切入点方法正常执行后执行
- aop:after-throwing      切入点方法异常后执行
- aop:after

##### 环绕通知

配置方式

```xml
<!-- 配置环绕通知 -->
...
		<aop:around method="transactionAround" pointcut-ref="pt1"/>
	</aop:aspect>
</aop:config>
```

当配置完环绕通知之后，没有业务层方法执行（切入点方法执行）,

```java
/**
* spring 框架为我们提供了一个接口，该接口可以作为环绕通知的方法参数来使用
* ProceedingJoinPoint。当环绕通知执行时，spring 框架会为我们注入该接口的实现类。
* 它有一个方法 proceed()，就相当于 invoke，明确的业务层方法调用
*/
public void aroundPrintLog(ProceedingJoinPoint pjp) {
   try {
	System.out.println("前置 Logger 类中的 aroundPrintLog 方法开始记录日志了");
	pjp.proceed();//明确的方法调用
	System.out.println("后置 Logger 类中的 aroundPrintLog 方法开始记录日志了");
	} catch (Throwable e) {
		System.out.println("异常 Logger 类中的 aroundPrintLog 方法开始记录日志了");
	}finally {
		System.out.println("最终 Logger 类中的 aroundPrintLog 方法开始记录日志了");
	}
}
```

### 4.基于注解的Spring AOP

##### 第一步：配置applicationContext.xml文件

```xml
<!-- 基于注解的AOP：配置注解扫描包 -->
<context:component-scan base-package="cn.dintalk.Aop"/>
<!-- 1.开启AOP对注解的支持 -->
<aop:aspectj-autoproxy/>
```

##### 第二步：添加注解

```java
// userService实现类
@Service("userService")
public class UserServiceImpl implements UserService {
    
//切面类
@Component("logger")
@Aspect    // 声明为通知类
public class Logger {
    @Before("execution(* cn.dintalk.Aop..*.*(..))")
    public void printLogger(){
        System.out.println("logger中的printLogger方法执行了");
    }
    // 使用抽取出来的切入点表达式
    @AfterReturning("m1()")
    public void afterPrintLogger(){
        System.out.println("logger中的afgerPrintLogger方法执行了");
    }

    @Pointcut("execution(* cn.dintalk.Aop..*.*(..))")
    public void m1(){ } // 方法无意义，抽取切入点表达式而已 
```

### 5.基于类的Spring AOP

```java
@Configuration
@ComponentScan("cn.dintalk.Aop")
@EnableAspectJAutoProxy // 开启对注解的支持
public class SpringConfig {
}

//基于类的注解 获取容器
userService = new AnnotationConfigApplicationContext(SpringConfig.class)
	.getBean("userService",UserService.class);
userService.login();
```

## X附录：Spring整合Junit

在程序测试阶段，我们总是需要将加载Spring的配置获取容器，即诸如以下代码：

```java
ApplicationContext applicationContext1 = new
           ClassPathXmlApplicationContext("applicationContext.xml");
UserServiceImpl userService = 
    applicationContext1.getBean("userService", UserServiceImpl.class);
```

我们可以借助spring提供的运行器来读取配置文件或注解来创建容器。使用如下：

##### 第一步：导入spring整合Junit的坐标

```xml
<dependency>
     <groupId>org.springframework</groupId>
     <artifactId>spring-test</artifactId>
     <version>5.0.2.RELEASE</version>
     <scope>test</scope>
</dependency>
```

##### 第二步：使用@RunWith注解替换原有运行器

##### 第三步：使用@ContextConfiguration指定spring配置文件的位置

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration("classpath:applicationContext.xml")
public class UserAopTest {
```

##### 第四步：使用@AutoWired注入数据

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration("classpath:applicationContext.xml")
public class UserAopTest {
    @Autowired
    private UserService userService;
```

##### **Tips:** 在测试方法上必须显示的添加@Test

使用@RunWith注解替换原有运行器，测试类中的测试方法即使不添加@Test注解也可以运行（有运行按钮）。但是会报如下错误：java.lang.Exception: No runnable methods。此时只要在测试方法上显示的添加@Test注解即可！

