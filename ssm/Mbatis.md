# MyBatis -- 必知必会



# 一:MyBatis概述

​	MyBatis的前身是Apache的一个开源项目iBatis，2010年这个项目由apache software foundation 迁移到了google code，并且改名为MyBatis。2013年11月迁移到GitHub，因此目前MyBatis是由GitHub维护的。

​	同样作为持久层框架的Hibernate在前些年非常的火，它在配置了映射文件和数据库连接文件后就可以通过Session操作，它甚至提供了HQL去操作POJO进而操作数据库的数据，几乎可以使编程人员脱离sql语言。可是为什么MyBatis却越来越受欢迎呢？我们稍稍总结如下：

Hibernate: 1.不方便的全表映射，比如更新时需要发送所有的字段；

​		    2.无法根据不同的条件组装不同sql;

​		    3.对多表关联和复制sql查询支持较差；

 		    4.有HQL但性能较差，做不到sql优化；

​		    5.不能有效支持存储过程；

​	在当今的大型互联网中，灵活、sql优化，减少数据的传递是最基本的优化方法，但是Hibernate却无法满足我们的需求，而MyBatis提供了更灵活、更方便的方法。在MyBatis里，我们需要自己编写sql，虽然比Hibernate配置要多，但是是MyBatis可以配置动态sql，也可以优化sql,且支持存储过程，MyBatis几乎能做到 JDBC 所能做到的所有事情！凭借其高度灵活、可优化、易维护等特点，成为目前大型移动互联网项目的首选框架。

# 二：开发环境、流程及生命周期

### 1.开发环境

#### 1.1 导入依赖jar包

 - 导入mybatis的jar包，如果采用Maven进行部署，只需导入mybatis坐标。
- 导入数据库驱动jar包或坐标。
- 代码调试阶段可导入日志工具jar包或坐标，如log4j。
- 导入Junit单元测试坐标。

#### 1.2 创建mybatis的主配置文件

​	在resources目录下建立一个名字为mybatis-config.xml（名称随意）配置文件，编写所需配置信息。

#### 1.3 准备实体类和表结构

​	遵循开发规范：

- 类属性命名尽量和表字段保持一致。

- 实体类实现序列化接口。

- 实体类属性使用包装类型定义，如Integer。

  **Tips:** 有时可以考虑通过定义时初始化来避免可能的空指针异常！

  ​	如：private List<String> list = new ArrayList<>() ;

#### 1.4 创建Mapper接口（Dao接口）建立接口方法和sql映射文件

创建Mapper接口，在内定义CRUD方法。

​	**Tips:**  方法名唯一，需要在对应的mapper.xml文件中配置id。

在resources下创建sql映射文件。

​	**Tips:** 同对应的Mapper接口保持包结构及命名一致。

​	如：Mapper接口： cn.dintalk.dao.UserMapper

​		对应配置文件：cn.dintalk.dao.UserMapper.xml       

#### 1.5 将映射文件加入到mybatis主配置文件中

将映射文件通过引入的方式加入到mybatis的主配置文件中。

​	**Tips:** 所有映射文件会随主配置文件在程序运行时加入内存，任一映射文件出错都会导致整个环境报错！（初学者经常搞混resultType和resultMap)。

#### 1.6 编写代码进行CRUD操作

在映射文件中编写sql进行crud操作，在单元测试中，或service层中调用方法！

### 2.开发流程

环境搭建好后开发基本流程为：

- 接口定义方法 。
- Mapper.xml文件中编写sql。
- 单元测试或service调用。

###### Tips: 接口中方法名称和Mapper.xml文件中sql语句的id保持一致！

### 3.生命周期

MyBatis的核心组件：

- SqlSessionFactoryBuilder(构造器)：根据配置信息或代码生成SqlSessionFactory

- SqlSessionFactory(工厂接口):依靠工厂来生成SqlSession(会话)。

- SqlSession(会话): 既可以发生sql去执行并返回结果，也可以获取Mapper的接口

- SQL Mapper:它是MyBatis新设计的组件，它是由一个java接口和xml文件（或注解）构成的，需要给出对应的sql和映射规则。它负责发送sql去执行，并返回结果。

  ​	正确理解并掌握上述核心组件的生命周期可以让我们写出高效的程序，还可避免带来严重的并发问题。

#### 3.1 SqlSessionFactoryBuilder

​	其作用就是利用xml或java编码获得资源来构建SqlSessionFactory对象，构建成功就失去了存在的意义，将其回收。所以它的生命周期只存在于方法的局部。

#### 3.2 SqlSessionFactory

​	SqlSessionFactory的作用是创建SqlSession，而SqlSession就是一个会话，相当于JDBC中的Connection对象，每次访问数据库都需要通过SqlSessionFactory创建SqlSession，所以SqlSessionFactory应该在MyBatis应用的整个生命周期中。我们使每一个数据库只对应一个SqlSessionFactory（单例模式）。

#### 3.3 SqlSession

​	SqlSession是一个会话，相当于JDBC的一个Connection对象，它的生命周期应该是在请求数据库处理事务的过程中。是一个线程不安全的对象，涉及多线程时格外当心。此外，每次创建的SqlSession都必须及时关闭它。

#### 3.4 Mapper

​	Mapper是一个接口，没有任何实现类，其作用是发送sql,返回我们需要的结果，或者执行sql修改数据库数据，因此它也因该在一个SqlSession事务方法之内，是一个方法级别的东西。就如同 JDBC中的一条sql语句的执行，它的最大范围和SqlSession是相同的。

###### Tips: 根据核心组件封装工具、形成SqlSession使用模板

```java
public class MyBatisUtil {
    private static SqlSessionFactory build =null;
    static {
        try {  //使用MyBatis的Resources加载资源获得输入流，构建工厂
            InputStream inputStream = Resources.getResourceAsStream("mybatis-config.xml");
            build = new SqlSessionFactoryBuilder().build(inputStream);
        } catch (IOException e) {
            throw new RuntimeException("资源文件加载失败!");
        }
    }
    //使用工厂生产sqlSession 
    public static SqlSession openSession(){
        return build.openSession();
    }
}
```

###### SqlSession使用方法

```java
//1.定义sqlSession
SqlSession sqlSession = null;
try {
    sqlSession = openSession();
    //2.获取映射器
    UserMapper mapper = sqlSession.getMapper(UserMapper.class);
    //3.some code like 'User u = mapper.findById(id);'
    //4.sqlSession不提交默认回滚！增删改需格外注意！
    sqlSession.commit();
} catch (Exception e) {
    throw new RuntimeException(e);
} finally {
    if (sqlSession != null) {
        sqlSession.close();
    }
}
```

# 三：映射器

### 1.映射器元素简介	

我们可以在映射器中定义哪些元素，以及它们有何作用呢？

| 元素名称     | 作用                                   |
| ------------ | -------------------------------------- |
| insert       | 定义插入语句                           |
| delete       | 定义删除语句                           |
| update       | 定义修改语句                           |
| select       | 定义查询语句                           |
| parameterMap | 定义参数映射关系                       |
| resultMap    | 提供从数据库结果集到POJO映射规则       |
| cache        | 配置当前命名空间的缓存配置（二级缓存） |
| sql          | 定义部分sql，各个地方都可引用          |
| cache-ref    | 引用其他命名空间的缓存配置             |

在各个元素当中又有相当多的属性配置项，这里不多赘述，通过下一节掌握常用的内容即可。这里特别说明sql元素的使用：

```xml
<sql id="user_columns">  <!-- 此处定义后，处处使用，尤其字段多的时候 -->
	id, name, password   
</sql>
<select ....>
	select <include refid="user_columns"/> from users where uid=#{id}
</select>
```

### 2.简单CRUD操作

##### 2.1添加用户

```xml
<!-- 1.添加用户 -->
<insert id="add" parameterType="User">
    INSERT into users (name,password) VALUES (#{name},#{password});
</insert>
```

##### 2.2添加用户并返回数据库中的主键

```xml
<!-- 2.添加用户,获取数据库生成的主键值(需数据库具备主键自增功能) -->
<insert id="add" parameterType="User" useGeneratedKeys="true" keyProperty="uid">
    insert into users (name,password) values(#{name},#{password});
</insert>
<!-- 3.添加用户,获取数据库生成的主键:数据库有无自动增长能力都可以-->
<insert id="add" parameterType="User">
    <selectKey keyColumn="uid" keyProperty="uid" resultType="int" order="AFTER">
        select last_insert_id();
    </selectKey>
    insert into users (name,password) values(#{name},#{password});
</insert>
```

##### 2.3修改/删除用户

```xml
<!-- 4.修改/删除用户 -->
<update id="update" parameterType="User">
    update users set name=#{name},password=#{password} where uid=#{uid};
</update>
<delete id="delete" >
    delete from users where uid=#{uid};
</delete>
```

##### 2.4查询用户

```xml
<!-- 5.查询所有的用户 -->
<!-- 定义查询结果映射规则(结果集映射可以用别名) -->
<resultMap id="userMap" type="User">
    <id column="id" property="uid"></id>
    <result column="姓名" property="name"></result>
    <result column="password" property="password"></result>
</resultMap>
<!-- 查询语句 -->
<select id="findAll" resultMap="userMap">
    select uid as id,name as '姓名',password from users;
</select>
<!--6.根据用户id查询用户(单个参数)-->
<select id="findById" parameterType="int" resultType="User">
    select uid,name,password from users where uid=#{uid};
</select>
<!-- 7.根据用户名和密码查询用户(接口定义方法参数时@Param(" ")进行指定,否则多个参数时
默认使用 arg0,arg1   或param1,param2来映射) -->
<select id="findByNamePassword" resultType="User">
    select uid,name,password from users where name=#{name} and password=#{password};
</select>
<!-- 8.根据用户实体对象查询用户 -->
<select id="findByUser" parameterType="User" resultType="User">
    select uid,name,password from users where name=#{name};
</select>
<!-- 9.根据用户名进行模糊查询 -->
<select id="findUsersLikeName" resultType="User">
    select uid,name,password from users where name like concat("%",#{name},"%");
</select>
```

**Tips:**  sql中如果有特殊符号可使用 <![CDATA[    ]]> 将语句抱起来。

# 四：动态sql和高级查询

### 1.动态sql

​	MyBatis的动态sql包含这些元素：if  |  choose(when ohterwise) |trim(where set) | foreach ，通过这些元素来动态组装sql语句，主要是增改查操作。**(实际开发中，严禁使用select * 操作。这里为了简便使用select *演示）！**

##### 1.1使用 if,实现动态sql,完成查询操作

```xml
<!-- 1.使用if,实现动态sql,完成查询操作 -->
<select id="findByCondition" parameterType="user" resultType="user">
    select uid,name,password,email from users where 1 = 1
    <if test="name != null and name !=''">
        and name like concat("%",#{name},"%")
    </if>
    <if test="password != null and password != '' ">
        and password = #{password}
    </if>
</select>
```

##### 1.2使用if,实现动态sql,完成更新操作

```xml
<!-- 2.使用if,实现动态sql,完成更新操作 -->
<update id="updateUser" parameterType="user">
   update users set
   <if test="name != null and name != '' ">
       name=#{name},
   </if>
   <if test="password != null and password != '' ">
       password=#{password},
   </if>
   uid=#{uid} where uid=#{uid}; <!-- 保证任何情况下的sql完整 -->
</update>
```

##### 1.3使用if,实现动态sql,完成添加操作

```xml
<!-- 3.使用if,实现动态sql,完成添加操作 -->
<insert id="addUser" parameterType="user">
    insert into users (name,
    <if test="password != null and password != '' ">
        password,
    </if>email,phoneNumber,birthday) values(#{name},
    <if test="password != null and password != '' ">
        #{password},
    </if>#{email},#{phoneNumber},#{birthday});
</insert>
```

##### 1.4使用choose  when  otherwise,实现动态sql,完成查询操作

```xml
<!-- 4.使用choose when otherwise,实现动态sql,完成查询操作 -->
<select id="findByCondition" parameterType="user" resultType="user">
    select * from users where 1 = 1 <!-- 动态组装准备 -->
    <choose>
        <when test="name != null and name != '' ">
            and name like concat("%",#{name},"%");
        </when>
        <otherwise>
            and 1 = 2; <!-- 动态否决 -->
        </otherwise>
    </choose>
</select>
```

##### 1.5使用if和where,实现动态sql,完成查询操作  ★★★

```xml
<!-- 5.使用if和where,实现动态sql,完成查询操作 -->
<select id="findByCondition" resultType="user" parameterType="user">
    select * from users
    <where>  <!-- 至少有一个if执行时才会加上where关键字并去掉紧跟后面的and|or关键字 -->
        <if test="name != null and name != '' ">
            and name like concat("%",#{name},"%")
        </if>
        <if test="password != null and password != '' ">
            and password=#{password}
        </if>
    </where>
</select>
```

##### 1.6使用if和set,实现动态sql,完成更新操作 ★★

```xml
<!-- 6.使用if和set,实现动态sql,完成更新操作 -->
<update id="updateUser" parameterType="user">
    update users
    <set>   <!-- set元素会去掉最后一个,号 -->
        <if test="name != null and name != '' ">
            name=#{name},
        </if>
        <if test="password != null and password != '' ">
            password=#{password},
        </if>
        uid=#{uid},
    </set>
    where uid=#{uid};
</update>
<!-- trim元素的使用 -->
<trim prefix="where" prefixOverrides="and|or"></trim> <!-- 等同与where元素 -->
<trim prefix="set" suffixOverrides=","></trim> <!-- 等同与set元素 -->
```

#####  1.7使用foreach,实现动态sql,完成根据id集合、数组等的查询操作

```xml
<!-- 7.使用foreach,实现动态sql,完成根据id list列表的查询操作 -->
<select id="findByIds" resultType="user">
    select * from users where uid in
    <foreach collection="collection" open="(" close=")" separator="," item="uid">
        #{uid}
    </foreach>
</select>
<!-- foreach中的collection值取决于要遍历的对象类型(mybatis内部做判断后默认的)
 List : 可取 collection或list
 Set  : 取 collection
 Array: 取 array
 Map  : 取 _parameter   （用map无意义，遍历的依旧是value）
上述默认引用都可以在接口方法中通过@Param（“  ”）覆盖掉！
-->
```

##### 1.8bind元素

​	bind元素的作用是通过OGNL表达式自定义一个上下文变量，这样更方便我们使用。在进行模糊查询的时候，如果是MySQL数据库，我们常用concat函数用“%”和参数连接。然而在Oracle数据则是用连接符号“||”。这样sql就需要提供两种形式去实现。用bind元素，我们就不必使用数据库语言，只要使用MyBatis的语言即可与所需参数相连。

```xml
<select id="findByCondition" parameterType="user" resultType="user">
    <!-- 使用bind定义上下文变量 -->
    <bind name="pattern" value="'%' + name + '%' "/>
    select uid,name,password,email from users where 1 = 1
    <if test="name != null and name !=''">
       <!-- and name like concat("%",#{name},"%") -->
        and name like #{pattern}
    </if>
    <if test="password != null and password != '' ">
        and password = #{password}
    </if>
</select>
```

### 2.高级查询（多表查询）

##### 2.1一对多关联查询

```xml
<!-- 1.使用多表查询,完成一对多关联查询及映射 -->
    <!-- user结果集封装,可以通过继承重复使用 -->
<resultMap id="userMap" type="user">
    <id column="uid" property="uid" />
    <result column="name" property="name"/>
    <result column="password" property="password"/>
</resultMap>
    <!-- loginInfo结果集,继承user结果集完整数据 -->
<resultMap id="userLoginInfoMap" extends="userMap" type="user">
    <!-- 使用collection映射多的一方,ofType指定集合中的数据类型 -->
    <collection property="loginInfos" ofType="loginInfo">
        <id column="lid" property="lid"/>
        <result column="ip" property="ip"/>
        <result column="loginTime" property="loginTime"/>
    </collection>
</resultMap>
    <!-- 定义查询语句 -->
<select id="findAllUsers" resultMap="userLoginInfoMap">
    select * from users u left join login_infos li on u.uid = li.uid;
</select>
```

##### 2.2多对一关联查询(别名映射|resultMap映射|resultMap结合association映射)

```xml
 <!-- 2.多对一,采用别名映射关联关系 -->
<select id="findAllLoginInfos" resultType="loginInfo">
    select li.*,
    u.uid "user.uid",
    u.name "user.name",
    u.password "user.password"
    from login_infos li,users u
    where li.uid = u.uid;
</select>
<!-- 3.多对一,采用resultMap进行结关联关系的映射 -->
<resultMap id="userMap" type="loginInfo">
    <id column="uid" property="user.uid"/>
    <result column="name" property="user.name"/>
    <result column="password" property="user.password"/>
</resultMap>    
<!-- 这里的resultMap继承关系最好和POJO中的关联关系保持一致，便于理解
	 这里不一致，但也可以运行
 -->
<resultMap id="userLoginInfoMap" extends="userMap" type="loginInfo">
    <id column="lid" property="lid"/>
    <result column="ip" property="ip"/>
    <result column="loginTime" property="loginTime"/>
</resultMap>
<select id="findAllLoginInfos1" resultMap="userLoginInfoMap">
    select * from users u,login_infos li where u.uid=li.uid;
</select>
<!-- 4.多对一,使用resultMap结合association进行关联关系的映射 -->
<resultMap id="loginInfoMap" type="loginInfo">
    <id column="lid" property="lid"/>
    <result column="ip" property="ip"/>
    <result column="loginTime" property="loginTime"/>
</resultMap>
<!-- 这里的resultMap继承关系和POJO的关联关系保持了一致，即LoginInfo下有User属性 -->
<resultMap id="loginInfoUserMap" extends="loginInfoMap" type="loginInfo">
    <association property="user" javaType="user">
        <id column="uid" property="uid"/>
        <result column="name" property="name"/>
        <result column="password" property="password"/>
    </association>
</resultMap>
<select id="findAllLoginInfos2" resultMap="loginInfoUserMap">
    select * from users u,login_infos li where u.uid=li.uid;
</select>
```

##### 2.3多对多关联查询

```xml
<!-- 13.使用多表查询,完成多对多查询,查找所有的用户及其角色 -->
<!-- 注意：为了避免冗余，这里继承了2.2中的resultMap -->
 <resultMap id="userRoleMap" extends="userMap" type="user">
     <collection property="roles" ofType="role">
         <id column="rid" property="rid"/>
         <result column="name" property="name"/>
         <result column="description" property="description"/>
     </collection>
 </resultMap>
<!-- 多对多主要在sql编写上，需要借助中间表 -->
 <select id="findAllUsers" resultMap="userRoleMap">
     select * from users u left join users_roles ur 
     on u.uid=ur.uid inner join roles r on ur.rid=r.rid;
 </select>
```

# 五：嵌套查询和延迟加载

### 1.加载策略

​	在关联查询时，对于关联的一方是否查询出来，要根据业务需求而定。不能通过编码方式进行策略的改变，而应该通过修改配置文件改变加载策略。可以使用嵌套查询（分步查询）。

### 2.嵌套查询

##### 2.1根据多的一方，嵌套查询少的一方

```xml
<!-- 1.嵌套查询,使用reultMap结合association使用引用的mapper.xml进行查询 -->
<resultMap id="loginInfoMap" type="loginInfo">
    <id column="lid" property="lid"/>
    <result column="ip" property="ip"/>
    <result column="loginTime" property="loginTime"/>
</resultMap>
<resultMap id="loginInfoUserMap" extends="loginInfoMap" type="loginInfo">
    <!-- 解决user属性: property:属性
		column:查询依据，也是当前查询表的外键
        select:指向根据外键查询的xml唯一映射
    -->
    <association property="user" column="uid" 
                 select="cn.dintalk.UserMapper.findById"/>
</resultMap>
<select id="findAllLoginInfos3" resultMap="loginInfoUserMap">
    select * from login_infos;
</select>

<!-- 在association中，select指向的是另一个Mapper.xml文件中的映射（根据命名空间和id) -->
<!-- UserMapper.xml中  14.嵌套查询之 根据uid查询用户 -->
<select id="findById" resultType="user">
    select * from users where uid = #{uid};
</select>
```

##### 2.2根据少的一方，嵌套查询多的一方

```xml
<!-- 2.使用嵌套查询,可通过懒加载优化sql,查询所有的用户及其日志信息 -->
<!-- 为了简便，这里继承了上述的id为userMap的resultMap -->
<resultMap id="userLoginInfoMap" extends="userMap" type="user">
    <collection property="loginInfos"  column="uid" 
                select="cn.dintalk.dao.LoginInfoMapper.findAllLoginInfos"/>
</resultMap>
<select id="findAllUser" resultMap="userLoginInfoMap1">
    select * from users;
</select>
<!-- 同理，在collection中，select指向的是另一个Mapper.xml文件的映射 -->
<!--LoginInfoMapper.xml中  5.根据用户id查询所有的登录信息 -->
<select id="findAllLoginInfos" resultType="loginInfo">
    select * from login_infos where uid=#{uid};
</select>
```

### 3.配置延迟加载

##### 3.1全局配置，修改mybatis.xml主配置文件

```xml
<!-- 2.配置延迟加载,即sql优化 -->
<settings>
    <!-- 启用懒加载策略 -->
    <setting name="lazyLoadingEnabled" value="true"/>
    <!-- 覆盖掉延迟加载的触发方法 -->
    <setting name="lazyLoadTriggerMethods" value=""/>
</settings>
```

启用延迟加载后，mybatis默认有toString、equals等4个触发加载的方法。我们也可以将其覆盖掉。

延迟加载，即用到关联数据的时候再去查，不用就不查。（service层中会有很多方法调用dao方法，根据service层中的实际需求动态调整加载策略，高效利用资源！)

##### 3.2每个association和collection都有fetchType属性

###### 该属性的值或覆盖掉全局配置

fetchType="lazy"(默认） | eager

- lazy ： 支持延迟加载
- eager : 立即加载

# 六：事务控制及数据源

### 1.事务控制

###### 默认情况下：MySql的事务是自动提交的。

​	通过 JDBC 可以手动控制：

​		Connection.setAutoCommit(false);

​		Connection.commit();

​		Connection.rollback();// 开启事务后，未提交会自动回滚！

###### MyBatis中: mybatis-config.xml 作如下配置

```xml
<transactionManager type="JDBC"></transactionManager>
```

相当于使用 JDBC 进行事务控制：（增删改时不提交会自动回滚！）

获取SqlSession时：SqlSessionFactory.openSession();  // 手动控制事务  ★

​				   SqlSessionFactory.openSession(true); //自动提交事务

### 2.数据源

##### 2.1mybatis内置数据源

mybatis内置了三种数据源：

UNPOOLED :不带有池（连接池）|学习时用

POOLED : 带有池的 | 实际生产环境使用

JNDI : mybatis提供的JndiDataSourceFactory来获取数据源

##### 2.2内部原理

POOLED对应的是PooledDataSource数据源，PooledDataSourceFactory用来生产带有池的数据源。

UNPOOLED对应的是UnpooledDataSource数据源，UnpooledDataSourceFactory用来生产不带有池的数据源。

##### 2.3使用Druid等第三方数据源(以Druid为例)

###### 第一步：引入Druid的Jar包或数据源

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.1.14</version>
</dependency>
```

###### 第二步：编写工厂类，用来产生Druid的数据源，一般选择继承UnpooledDataSourceFactory

```java
public class DataSourceFactory extends UnpooledDataSourceFactory{
    public DataSourceFactory(){
        this.dataSource = new DruidDataSource();//创建druid数据源
    }
}
```

###### 第三步：在mybatis-config.xml中进行配置

```xml
<!--1.类别名的配置 -->
<typeAliases>
    <typeAlias type="cn.dintalk.dataSource.DataSourceFactory" alias="DRUID"/>
</typeAliases>
<environments default="mysql">
   <environment id="mysql">
       <transactionManager type="JDBC"></transactionManager>
       <!-- 2.配置druid数据源 -->
       <dataSource  type="DRUID">
           <!-- 3.配置数据库连接：name由数据源中的setXXX而定，value是外部配置的key -->
           <property name="driverClassName" value="${jdbc.driver}"></property>
           <property name="url" value="${jdbc.url}"></property>
           <property name="username" value="${jdbc.username}"></property>
           <property name="password" value="${jdbc.password}"></property>
       </dataSource>
   </environment>
</environments>
```

# 七：MyBatis的缓存

### 1.系统缓存

MyBatis对缓存提供支持，但是在没有配置的默认情况下，它只开启一级缓存。

##### 1.1一级缓存

​	一级缓存只是相对于同一个SqlSession而言的。使用SqlSession第一次查询后，MyBatis会将其放在缓存中，以后再查询时，如果没有声明需要刷新，且缓存未超时的情况下，SqlSession都只会取出当前缓存的数据，而不会再次发送Sql到数据库。

##### 1.2二级缓存

​	二级缓存是在SqlSessionFactory层面上的，可以将缓存提供给各个SqlSession对象共享。

###### 开启二级缓存配置

###### mybatis-confi.xml文件中(默认开启，可忽略)

```xml
<settings>  
  <!-- 二级缓存配置(默认开启,此行可省略) -->
  <setting name="cacheEnabled" value="true"/>
</settings>
```

###### Mapper.xml文件中

```xml
<mapper namespace="cn.dintalk.dao.UserMapper">
    <cache/>  <!-- 开启二级缓存，使用默认配置 -->
</mapper>
<!-- 使用默认配置意味着：
映射语句文件中的所有select语句将会被缓存。
映射语句文件中的所有insert、update和delete语句会刷新缓存。
缓存使用默认的Least Recently Used(LRU，最近最少使用的)回收策略。
缓存会存1024个列表集合或对象（无论查询方法返回什么）
缓存会被视为是read/write（可读可写）的缓存
-->
```

**Tips:** 

-  一级缓存中存放的是对象本身，是同一个对象！
- 二级缓存中存放的是对象的拷贝，对象所属类必须实现jav.io.Serializable接口!

###### 配置缓存

```xml
<cache  evicition="LRU" flushInterval="100000" size="1024" readOnly=true/>
```

- eviction: 代表的是缓存回收策略，目前MyBatis提供以下策略：

  ​	（1）LRU, 最近最少使用的，移除最长时间不用的对象。

  ​	（2）FIFO, 先进先出，按对象进入缓存的顺序来移除它们。

  ​	（3）SOFT,软引用，移除基于垃圾回收器状态和软引用规则的对象。

  ​	（4）WEAK,弱引用，更积极地移除基于垃圾收集器状态和弱引用规则的对象。

- flushInterval: 刷新间隔时间，单位为毫秒。

- size: 引用数目，代表缓存最多可以存储多少个对象。

- readOnly: 只读，意味着缓存数据只读。

### 2.自定义缓存

​	系统缓存是MyBatis应用机器上的本地缓存，我们也可以使用缓存服务器来定制缓存，如比较流行的Redis缓存。我们需要实现MyBatis为我们提供的接口org.apache.ibatis.cache.Cache,缓存接口简介：

```java
//获取缓存编号
String getId();
//保存key值缓存对象
void putObject(Object key,Object value);
//通过key获取缓存对象
Object getObject(Object key);
//通过key删除缓存对象
Object removeObject(Object key);
//清空缓存
void clear();
//获取缓存对象大小
int getSize();
//获取缓存的读写锁
ReadWriterLock getReadWriterLock();
```

由于每种缓存都有其不同的特点，上面的接口都需要我们去实现。假设我们已经有一个实现类：cn.dintalk.MyCache。则配置如下：

```xml
<cache type="cn.dintalk.MyCache"/>
```

完成上述配置，就能使用自定义的缓存了。MyBatis也支持在缓存中定义常用的属性，如：

```xml
<cache type="cn.dintalk.MyCache">
  <property name="host" value="localhost"/>
</cache>
```

如果我们在MyCache这个类中增加setHost(String host) 方法,那么它在初始化的时候就会被调用，这样我们可以对自定义的缓存设置一些外部参数。

**Tips:**  我们也可配置Sql层面的缓存规则，来决定它们是否需要刷新或使用缓存。

```xml
<insert ...flushCache="true"/>
<delete ...flushCache="true"/>
<update ...flushCache="true"/>
<select ...flushCache="false" useCache="true"/>
```

# 八：附录-MyBatis常用配置及开发Tips

#### 附录1：mybatis-config.xml常用配置

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <!-- 1.引入外部的配置文件 -->
    <properties resource="jdbc.properties"/>
    <!-- 2.配置延迟加载,即sql优化 -->
    <settings>
        <!-- 启用懒加载策略 -->
        <setting name="lazyLoadingEnabled" value="true"/>
        <!-- 覆盖掉延迟加载的触发方法 -->
        <setting name="lazyLoadTriggerMethods" value=""/>
        <!-- 二级缓存配置(默认开启,此行可省略) -->
        <!-- 使用二级缓存,在对应的mapper.xml中加入cache即可 -->
        <!--<setting name="cacheEnabled" value="true"/>-->
    </settings>
    <!-- 3.类别名的配置 -->
    <typeAliases>
        <!-- 单个类的配置 -->
        <!--<typeAlias type="cn.dintalk.domain.User" alias="user"/>-->
        <!-- 配置druid数据源工厂类别名 -->
        <typeAlias type="cn.dintalk.dataSource.DataSourceFactory" alias="DRUID"/>
        <!-- 给包中所有的类配置默认别名, 即类名首字母小写-->
        <package name="cn.dintalk.domain"/>
    </typeAliases>
    <!-- 4.使用默认的环境配置(可以是多个) -->
    <environments default="mysql">
        <environment id="mysql">
            <!-- 事务管理器,此处配置 为JDBC -->
            <transactionManager type="JDBC"></transactionManager>
            <!-- 数据源配置,此处配置为 POOLED-->
            <!--<dataSource  type="POOLED">-->
            <!-- 配置druid数据源 -->
            <dataSource  type="DRUID">
                <!-- 配置数据库连接：name由数据源中的setXXX而定，value是外部配置的key -->
                <property name="driverClassName" value="${jdbc.driver}"></property>
                <property name="url" value="${jdbc.url}"></property>
                <property name="username" value="${jdbc.username}"></property>
                <property name="password" value="${jdbc.password}"></property>
            </dataSource>
        </environment>
    </environments>
    <!-- 5.注册映射文件 -->
    <mappers>
        <!-- 指定资源文件路径 -->
        <!--<mapper resource="cn/dintalk/dao/UserMapper.xml"></mapper>-->
        <!--<mapper resource="cn/dintalk/dao/LoginInfoMapper.xml"></mapper>-->
        <!-- 基于Mapper接口的开发:指定类名-->
        <!--<mapper class="cn.dintalk.dao.UserMapper"/>-->
        <!-- 指定基于Mapper接口开发的包:(需类名和xml文件名一致,包名一致)-->
        <package name="cn.dintalk.dao"/>
    </mappers>
</configuration>
```

#### 附录2：Mapper.xml头约束

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="cn.dintalk.dao.UserMapper">
    <cache/> <!-- 开启二级缓存 -->
</mapper>
```

