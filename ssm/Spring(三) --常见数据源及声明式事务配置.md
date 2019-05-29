# Spring:(三) --常见数据源及声明式事务配置

​	Spring自带了一组数据访问框架,集成了多种数据访问技术。无论我们是直接通过 JDBC 还是像Hibernate或Mybatis那样的框架实现数据持久化，Spring都可以为我们消除持久化代码中那些单调枯燥的数据访问逻辑。Spring对大多数的持久化方式提供支持。

​	Spring在数据访问中使用模板的模式，将访问过程中固定的和可变的部分明确划分为两个不同的类：模板（template）和回调（callback）。模板处理数据访问中固定的部分——事务控制、管理资源及处理异常，而回调处理应用程序相关的的数据访问——语句、绑定参数及整理结果集。基于此，我们只需关心自己的数据访问逻辑即可。

​	针对不同的持久化平台，Spring提供了多个可选的模板，如果直接使用JDBC，那么我们可以选择JdbcTemplate。

## 一.常见数据源的配置

​	无论我们选择哪一种数据访问方式，都需要配置一个数据源的引用。我们总结了几种常见的数据源配置方式。

#### 1.导入外部的数据源属性配置文件

##### jdbc.properties:

```xml
jdbc.driverClassName=com.mysql.jdbc.Driver
jdbc.url=jdbc:mysql:///spring_02
jdbc.username=sh
jdbc.password=sh123
```

##### applicationContext.xml中进行导入

```xml
<!-- 导入外部的数据源属性配置文件 -->
<!-- 第一种方法 -->
<bean class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
    <property name="location" value="classpath:jdbc.properties"/>
</bean>
<!-- 第二种方法 -->
<context:property-placeholder location="classpath:jdbc.properties"/>
```

### 1.Spring内置数据源

##### 第一步：导入jar包或Maven坐标

```xml
<dependency>
     <groupId>org.springframework</groupId>
     <artifactId>spring-jdbc</artifactId>
     <version>5.0.2.RELEASE</version>
</dependency>
```

##### 第二步：在applicationContext.xml中配置

```xml
<!-- 1.Spring内置的数据源 -->
<bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
   <property name="driverClassName" value="${jdbc.driverClassName}"/>
   <property name="url" value="${jdbc.url}"/>
   <property name="username" value="${jdbc.username}"/>
   <property name="password" value="${jdbc.password}"/>
</bean>
```

**Tips:** DriverManagerDataSource是一个标准的数据源（实现了接口javax.sql.DataSource）。没有连接池技术，没次都会获取新的数据库连接。用于学习、练习及小应用。

### 2.C3P0数据源

##### 第一步：导入jar包或Maven坐标

```java
<dependency>
    <groupId>com.mchange</groupId>
    <artifactId>c3p0</artifactId>
    <version>0.9.5.2</version>
</dependency>
```

##### 第二步：在applicationContext.xml中配置

```xml
<!-- 2.C3P0数据源的配置 -->
<bean id="dataSource1" class="com.mchange.v2.c3p0.ComboPooledDataSource">
   <property name="driverClass" value="${jdbc.driverClassName}"/>
   <property name="jdbcUrl" value="${jdbc.url}"/>
   <property name="user" value="${jdbc.username}"/>
   <property name="password" value="${jdbc.password}"/>
</bean>
```

**Tips:** C3P0是一个标准的数据源，带有池技术，适合生产环境。

### 3.Druid数据源

##### 第一步：导入jar包或Maven坐标

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.0.14</version>
</dependency>
```

##### 第二步：在applicationContext.xml中配置

```xml
<!-- 3.Druid数据源的配置 -->
<bean id="dataSource2" class="com.alibaba.druid.pool.DruidDataSource">
    <property name="driverClassName" value="${jdbc.driverClassName}"/>
    <property name="url" value="${jdbc.url}"/>
    <property name="username" value="${jdbc.username}"/>
    <property name="password" value="${jdbc.password}"/>
</bean>
```

**Tips:** Druid是一个标准数据源，带有池技术，适合生产环境

### 4.DBCP数据源

##### 第一步：导入jar包或Maven坐标

```xml
<dependency>
     <groupId>commons-dbcp</groupId>
     <artifactId>commons-dbcp</artifactId>
     <version>1.4</version>
</dependency>
```

##### 第二步：在applicationContext.xml中配置

```xml
<!-- 4.DBCP数据源的配置 -->
<bean id="dataSource3" class="org.apache.commons.dbcp.BasicDataSource">
     <property name="driverClassName" value="${jdbc.driverClassName}"/>
     <property name="url" value="${jdbc.url}"/>
     <property name="username" value="${jdbc.username}"/>
     <property name="password" value="${jdbc.password}"/>
</bean>
```

**Tips:** tomcat内置的就是dbcp数据源，也是一个标准的数据源，带有池技术，适合生		

​	产环境。

## 二、JdbcDaoSupport

### 1.使用jdbcTemplate

##### dao实现类关键代码

```java
public class AccountDaoImpl implements AccountDao {
   private JdbcTemplate jdbcTemplate;
   public void setJdbcTemplate(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }
    public void add(Account account) {
        jdbcTemplate.update("insert into accounts VALUES (null,?,?)"
        ,account.getAccountName(),account.getBalance());
    }
```

##### applicationContext.xml关键配置

```xml
...
<!-- JdbcTemplate配置bean -->
<bean id="jdbcTem" class="org.springframework.jdbc.core.JdbcTemplate">
     <property name="dataSource" ref="dataSource"/>
</bean>-->
<bean id="accountDao" class="cn.dintalk.dao.impl.AccountDaoImpl">
     <property name="jdbcTemplate" ref="jdbcTem"/>
</bean>
```

### 2.继承JdbcDaoSupport

JdbcDaoSupport是Spring框架为我们提供的一个类，该类中定义了一个JdbcTemplate对象，我们可以直接获取使用，但是要创建该对象，需要为其提供一个数据源：

##### JdbcDaoSupport关键源代码

```java
public abstract class JdbcDaoSupport extends DaoSupport {
    @Nullable
    private JdbcTemplate jdbcTemplate;

    public JdbcDaoSupport() {
    }
    public final void setDataSource(DataSource dataSource) {
        if (this.jdbcTemplate == null || dataSource != this.jdbcTemplate.getDataSource()) {
            this.jdbcTemplate = this.createJdbcTemplate(dataSource);
            this.initTemplateConfig();
        }
    }
    protected JdbcTemplate createJdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
```

##### dao实现类关键代码

```java
// 此时，实现类需要继承JdbcDaoSupport
public class AccountDaoImpl1 extends JdbcDaoSupport implements AccountDao {
    public void add(Account account) {//父类的 getJdbcTemplate（）方法。
        getJdbcTemplate().update("insert into accounts VALUES (null,?,?)"
        ,account.getAccountName(),account.getBalance());
        System.out.println("是我,没错");
    }
```

##### applicationContext.xml关键配置

```xml
...
<!-- JdbcDataSupport配置 bean 需要为父类注入数据源依赖 -->
<bean id="accountDao" class="cn.dintalk.dao.impl.AccountDaoImpl1">
        <property name="dataSource" ref="dataSource"/>
</bean>
```

**Tips:** 因为AccountDaoImpl1继承了JdbcDataSupport，所以我们这里配置子类（间接给父类）就

​	可以。当然，我们也可以不配置数据源，直接配置（间接给父类）一个jdbcTemplate也是可

​	以的。

**总结：**

- 第一种在Dao类中定义JdbcTemplate的方式，适用于所有配置方式（xml和注解都可以）。
- 第二种让Dao继承JdbcDaoSupport的方式，只能用于基于xml的方式，注解用不了。

## 三、声明式事务的配置

### 1.Spring事务控制的API简介

#### 1.1PlatformTransactionManager

​	此接口是Spring的事务管理器，提供了常用的操作事务的方法，包含3个具体的操作

- 获取事务状态信息：Transaction   getTransaction(TransactionDefinition definition)
- 提交事务： void  commit(TransactionStatus status)
- 回滚事务： void  rollback(TansactionStatus status)

我们在开发时都是使用它的实现类，在使用Spring JDBC或Mybatis进行持久化数据时，真正管理事务的对象是：

org.springframework.jdbc.datasource.DataSourceTransactionManager

#### 1.2TransactionDefinition

​	它是事务的定义信息对象，内有如下方法：

- 获取事务对象名称     ：String  getName()
- 获取事务隔离级别    ： int   getIsolationLevel()
- 获取事务传播行为    ： int   getPropagationBehavior()
- 获取事务超时时间      :  int   getTimeOut()
- 获取事务是否是只读 ： boolean  isReadOnly()

**Tips:** 并不是所有的数据库都支持事务支持的，默认为-1,即没有超时限制。

​	读写型事务：增删改时开启事务。

​	只读型事务：执行查询时也开启事务。

##### 事务的传播行为

m1方法有自己的事务，m2方法也有自己的事务，若m2方法中调用了m1的方法，那m1该采用什么用的事务？

​	这里只介绍常用的两种：

- REQUIRED:若m2有事务，则m1加入到m2的事务；若m2没有事务，m1用自己的事务。
- SUPPORTS:若m2有事务，则m1加入到m2的事务；若m2没有事务，m1放弃事务。

#### 1.3TransactionStatus

此接口提供事务具体的运行状态：

- 刷新事务      ：  void  flush()
- 是否有存储点：boolean  hasSavepoint()
- 事务是否完成：boolean  isCompleted()
- 是否为新的事务： isNew  Transaction()
- 事务是否回滚：  boolean  isRollbackOnly()
- 设置事务回滚：void  setRollbackOnly()

### 2.事务管理器 DataSourceTransactionManager

​	事务控制是横切面问题，采用AOP编程，通知代码即事务管理器是一个实现了

PlatformTransactionManager的类，Spring已经为我们提供好了。这里我们使用

DataSourceTransactionManager。

**Tips： ** 此事务管理器仅用于实现了标准数据源的事务控制。

### 3.基于xml的事务控制配置

##### 第一步：编写核心业务代码

```java
public class AccountServiceImpl implements AccountService {
    private AccountDao accountDao;
	public void setAccountDao(AccountDao accountDao) {
        this.accountDao = accountDao;
    }
    public void transfer(String sourceAccount, String targetAccount, Float money) {
        Account source = accountDao.findByName(sourceAccount);
        Account target = accountDao.findByName(targetAccount);
        source.setBalance(source.getBalance()-money);
        target.setBalance(target.getBalance()+money);
        accountDao.update(source);
//        int i=1/0;
        accountDao.update(target);
```

##### 第二步：配置applicationContext.xml

```xml
<!-- 1.导入外部的数据源属性配置文件 -->
<context:property-placeholder location="classpath:jdbc.properties"/>
<!-- 2.配置Spring内置的数据源 -->
<bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
    <property name="driverClassName" value="${jdbc.driverClassName}"/>
    <property name="url" value="${jdbc.url}"/>
    <property name="username" value="${jdbc.username}"/>
    <property name="password" value="${jdbc.password}"/>
</bean>
<!-- 3.IOC配置 Bean和JdbcTemplate配置 -->
<bean id="jdbcTem" class="org.springframework.jdbc.core.JdbcTemplate">
        <property name="dataSource" ref="dataSource"/>
</bean>-->
<bean id="accountDao" class="cn.dintalk.dao.impl.AccountDaoImpl">
        <property name="jdbcTemplate" ref="jdbcTem"/>
</bean>
<bean id="accountService" class="cn.dintalk.service.impl.AccountServiceImpl">
    <property name="accountDao" ref="accountDao"/>
</bean>
<!--=============================== -->

<!-- 基于xml的 声明式事务控制 -->
<!-- 1.将事务管理器交给Spring进行管理 -->
<bean id="transactionManager" 
      class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>
<!-- 2.配置事务通知的属性 -->
<tx:advice id="txAdvice" transaction-manager="transactionManager" >
    <tx:attributes>
        <tx:method name="transfer" propagation="REQUIRED" read-only="false"/>
        <tx:method name="add*" propagation="REQUIRED" read-only="false"/>
    </tx:attributes>
</tx:advice>
<!-- 3.配置切面 -->
<aop:config>
    <aop:advisor advice-ref="txAdvice" pointcut="execution(* cn.dintalk..*.*(..))"/>
</aop:config>
```

**Tips:**  事务通知属性中的method配置精确控制到方法名，而切点的配置则是模糊匹配。两者并不冲突，而是相互

​	配合。

- 如果事务管理器的id为“transactionManager" 则事务通知中的：transaction-manager=”...“可以省略。

### 4.基于注解的事务控制配置

##### 第一步：添加IOC注解

```java
// dao实现类关键代码
@Repository
public class AccountDaoImpl implements AccountDao {
    @Autowired
    private JdbcTemplate jdbcTemplate;
// service实现类关键代码   
@Service("accountService")
public class AccountServiceImpl implements AccountService {
    @Autowired
    private AccountDao accountDao;
    //配置事务管理器
    //要求事务管理器id必须是transactionManager方可省略
    @Transactional(readOnly = false,propagation = Propagation.REQUIRED)
    public void transfer(String sourceAccount, String targetAccount, Float money) {
        Account source = accountDao.findByName(sourceAccount);
        Account target = accountDao.findByName(targetAccount);
        source.setBalance(source.getBalance()-money);
        target.setBalance(target.getBalance()+money);
        accountDao.update(source);
//        int i=1/0;
        accountDao.update(target);
    }
```

##### 第二步：添加applicationContext.xml关键配置

```xml
...
<context:component-scan base-package="cn.dintalk"/>
<!-- 1.将事务管理器交给Spring进行管理 -->
<bean id="transactionManager" 
      class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
   <property name="dataSource" ref="dataSource"/>
</bean>
<!-- 2.开启注解事务的支持 -->
<tx:annotation-driven transaction-manager="transactionManager"/>

```

##### 第三步：测试关键代码(xml和注解方式通用)

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration("classpath:applicationContext.xml")
public class AccountServiceTest {
    @Autowired
    private AccountService accountService;
    @Test
    public void testTransfer(){
        accountService.transfer("a","b",1f);
    }
```

### 5.基于类的事务控制配置

##### 第一步：编写配置类

```java
@ComponentScan("cn.dintalk")
@PropertySource("classpath:jdbc.properties")// 加载外部属性配置文件
@EnableTransactionManagement  //开启注解事务的支持
public class SpringConfig {
    @Value("${jdbc.driverClassName}")
    private String driverClassName;
    @Value("${jdbc.url}")
    private String url;
    @Value("${jdbc.username}")
    private String username;
    @Value("${jdbc.password}")
    private String password;
	// 将数据源交予Spring容器
    @Bean("dataSource")
    public DataSource createDataSource(){
        DriverManagerDataSource dataSource = new DriverManagerDataSource();
        dataSource.setDriverClass(driverClassName);
        dataSource.setJdbcUrl(url);
        dataSource.setUser(username);
        dataSource.setPassword(password);
        return dataSource;
    }
	//将jdbcTemplate交予Spring容器
    @Bean("jdbcTemplate")
    public JdbcTemplate createJdbcTem(@Qualifier("dataSource") DataSource dataSource){
        JdbcTemplate jdbcTemplate = new JdbcTemplate(dataSource);
        return jdbcTemplate;
    }
	//将事务管理器交予Spring容器
    @Bean("transactionManager")
    public DataSourceTransactionManager createTransManager(@Qualifier("dataSource") 
                                                           DataSource dataSource){
        DataSourceTransactionManager dataSourceTransactionManager = 
            new DataSourceTransactionManager(dataSource);
        return dataSourceTransactionManager;
    }
}
```

##### 第二步：添加IOC注解

```java
// dao实现类关键代码
@Repository
public class AccountDaoImpl implements AccountDao {
    @Autowired
    private JdbcTemplate jdbcTemplate;
// service实现类关键代码   
@Service("accountService")
public class AccountServiceImpl implements AccountService {
    @Autowired
    private AccountDao accountDao;
    //配置事务管理器
    //要求事务管理器id必须是transactionManager方可省略
    @Transactional(readOnly = false,propagation = Propagation.REQUIRED)
    public void transfer(String sourceAccount, String targetAccount, Float money) {
        Account source = accountDao.findByName(sourceAccount);
        Account target = accountDao.findByName(targetAccount);
        source.setBalance(source.getBalance()-money);
        target.setBalance(target.getBalance()+money);
        accountDao.update(source);
//        int i=1/0;
        accountDao.update(target);
    }
```

##### 第三步：测试关键代码

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = {SpringConfig.class})
public class AccountServiceClassTest {
    @Autowired
    private AccountService accountService;
    @Test
    public void testTransfer(){
        accountService.transfer("a","b",3f);
    }
```

