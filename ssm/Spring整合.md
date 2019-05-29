# SSM整合及聚合工程



- Spring整合MyBatis
- Spring整合SpringMVC
- 使用Maven搭建SSM工程
- 使用Maven搭建SSM聚合工程

------



##### 	所谓整合，即将配置汇总到一起统一管理。在整合之前要确保由其单独搭建的开发环境是没有任何错误的，这样利于排错。

## 一.Spring整合MyBatis

#### 1.搭建mybatis的开发环境并测试通过

##### mybatis-config.xml主要配置

```xml
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
    <!--<typeAlias type="User" alias="user"/>-->
    <!-- 配置druid数据源工厂类别名 -->
    <typeAlias type="DataSourceFactory" alias="DRUID"/>

    <!-- 给包中所有的类配置默认别名, 即类名首字母小写-->
    <package name="cn.dintalk.domain"/>
</typeAliases>
<!-- 4.使用默认的环境配置(可以是多个) -->
<environments default="mysql">
    <environment id="mysql">
        <!-- 事务管理器,此处配置 为JDBC -->
        <!--<transactionManager type="JDBC"></transactionManager>-->
        <transactionManager type="JDBC"></transactionManager>
        <!-- 数据源配置,此处配置为 POOLED-->
        <!--<dataSource  type="POOLED">-->
        <dataSource  type="POOLED">
            <!-- 配置数据库连接 -->
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
    <!--<mapper class="UserMapper"/>-->
    <!-- 指定基于Mapper接口开发的包:(需类名和xml文件名一致,包名一致)-->
    <package name="cn.dintalk.dao"/>
</mappers>
```

#### 2.搭建Spring的开发环境并测试通过

##### applicationContext.xml主要配置

```xml
<!-- 1.导入外部的数据源属性配置文件 -->
<context:property-placeholder location="classpath:jdbc.properties"/>
<!-- 2.配置注解扫描包路径 -->
<context:component-scan base-package="cn.dintalk"/>
<!-- 3.Druid数据源的配置 -->
<bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
    <property name="driverClassName" value="${jdbc.driverClassName}"/>
    <property name="url" value="${jdbc.url}"/>
    <property name="username" value="${jdbc.username}"/>
    <property name="password" value="${jdbc.password}"/>
</bean>

<!-- 基于xml的 声明式事务控制 -->
<!-- 1.将事务管理器交给Spring进行管理 -->
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
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

#### 3.进行整合

##### 第一步：导入整合包的坐标

```java
<dependency>
  <groupId>org.mybatis</groupId>
  <artifactId>mybatis-spring</artifactId>
  <version>1.3.0</version>
</dependency>
```

##### 第二步：Spring接管Mybatis的主要配置

##### applicationContext.xml接管mybatis的主要配置

```xml
<!-- Spring接管mybatis-config.xml的配置 -->
<!-- 1.接管SqlSessionFactory -->
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <!--类别名配置-->
    <property name="typeAliasesPackage" value="cn.dintalk.estore"/>
    <!--数据源配置-->
    <property name="dataSource" ref="dataSource"/>
</bean>
<!-- 2.基于接口的mybatis的Mapper交给Spring管理 -->
<bean id="mapperScan" class="org.mybatis.spring.mapper.MapperScannerConfigurer">
    <!--指定Mapper所在的包,Spring接管Mapper接口对应的代理对象存于容器-->
    <property name="basePackage" value="cn.dintalk.estore.dao"/>
    <!--指定sqlSessisonFactory的名字,若容器中仅有一个可忽略-->
    <!--<property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"/>-->
</bean>
```

## 二.Spring整合SpringMVC

#### 1.搭建Spring开发环境并测试通过

#### 2.搭建SpringMVC的开发环境并测试通过

##### 第一步：创建springmvc.xml并添加以下配置

```xml
<!-- 扫描web层的包 -->
<context:component-scan base-package="cn.dintalk.estore.web.controller"/>
<mvc:annotation-driven/>
<!-- 配置视图解析器 -->
<bean id="viewResolver" class="org.springframework.web.servlet.view.InternalResourceViewResolver">
    <property name="prefix" value="/WEB-INF/jsp/"/>
    <property name="suffix" value=".jsp"/>
</bean>
```

##### 第二步：修改web.xml头约束并配置如下

```xml
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
                      http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
         version="3.1">
    <!-- 配置前端控制器 -->
  <servlet>
    <servlet-name>dispatcherServlet</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
      <param-name>contextConfigLocation</param-name>
      <param-value>classpath:springmvc.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
  </servlet>
  <servlet-mapping>
    <servlet-name>dispatcherServlet</servlet-name>
    <url-pattern>/</url-pattern>
  </servlet-mapping>
    <!-- 配置post请求过滤器 -->
  <filter>
    <filter-name>characterEncodingFilter</filter-name>
    <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
    <init-param>
      <param-name>encoding</param-name>
      <param-value>utf-8</param-value>
    </init-param>
  </filter>
  <filter-mapping>
    <filter-name>characterEncodingFilter</filter-name>
    <url-pattern>/*</url-pattern>
  </filter-mapping>
</web-app>
```

#### 3.整合

##### 第一步：在web.xml中添加配置如下

```xml
<!-- 配置spring父容器的启动时机 -->
<context-param>
  <param-name>contextConfigLocation</param-name>
  <param-value>classpath:applicationContext.xml</param-value>
</context-param>
<listener>
  <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```

**Tips:** 这样配置的目的就是为了使springmvc的容器可以获得spring容器内的实例。因此配置一个监听器，在应用加载时就加载spring容器。

## 三.使用Maven搭建SSM工程

​	按照清晰的思路并进行阶段的测试，搭建SSM工程就是一个小case!(导入pom文件中的依赖坐标是基本功，这里不再赘述)。

#### 1.思路整理

- 第一步：保证mybatis独立运行
- 第二步：保证spring的Ioc可以独立运行
- 第三步：整合spring和mybatis（spring接管SqlSessionFactory的创建，以及dao接口的代理实现类创建）
- 第四步：保证spring的事务可以使用，测试整合结果
-  第五步：保证springmvc可以独立运行
-  第六步：整合spring和springMVC	 

#### 2.编写顺序：

- 第一：实体类（数据模型，三层都用）
- 第二：编写持久层接口和映射配置（..Dao.xml）
- 第三：编写业务层的接口和实现类
- 第四：编写applicationContext.xml文件并测试（service+dao）
- 第五：编写sprinmvc.xml、web.xml、控制器和页面并测试（springmvc)
-  第六：web+service+dao测试

#### 3.配置文件主要内容

##### applicationContext.xml

```xml
<!-- 1.导入数据源的外部配置 -->
<context:property-placeholder location="classpath:jdbc.properties"/>
<!-- 2.指定注解扫描的包 -->
<context:component-scan base-package="cn.dintalk.dao"/>
<context:component-scan base-package="cn.dintalk.service"/>

<!-- 3.配置数据源 -->
<bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
    <property name="driverClassName" value="${jdbc.driver}"/>
    <property name="url" value="${jdbc.url}"/>
    <property name="username" value="${jdbc.username}"/>
    <property name="password" value="${jdbc.password}"/>
</bean>

<!-- 4.配置sqlSessionFactory -->
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="dataSource"/>
    <!-- 配置别名 -->
    <property name="typeAliasesPackage" value="cn.dintalk.domain"/>
</bean>
<!-- 5.配置mapper扫描 -->
<bean id="mapperScan" class="org.mybatis.spring.mapper.MapperScannerConfigurer">
    <property name="basePackage" value="com.itheima.dao"/>
</bean>

<!-- 6.配置事务管理器 -->
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>

<!-- 7.配置事务通知 -->
<tx:advice id="txAdvice" transaction-manager="transactionManager">
    <tx:attributes>
        <tx:method name="*" read-only="false" propagation="REQUIRED"/>
        <tx:method name="find*" read-only="true" propagation="SUPPORTS"/>
    </tx:attributes>
</tx:advice>

<!-- 8.配置切面 -->
<aop:config>
    <aop:pointcut id="pt1" expression="execution(* cn.dintalk.service.*.*(..))"/>
    <aop:advisor advice-ref="txAdvice" pointcut-ref="pt1"/>
</aop:config>
```

##### springmvc.xml主要配置

```xml
<!-- 1.配置注解扫描的路径 -->
<context:component-scan base-package="cn.dintalk.web"/>
<!-- 2.开启注解支持 -->
<mvc:annotation-driven/>

<!-- 3.配置视图解析器 -->
<bean id="viewResolver" class="org.springframework.web.servlet.view.InternalResourceViewResolver">
    <property name="prefix" value="/WEB-INF/pages/"/>
    <property name="suffix" value=".jsp"/>
</bean>
<!-- 4.静态资源放行 -->
<mvc:default-servlet-handler />
```

##### web.xml主要配置

```xml
<!-- 1.配置监听器 -->
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:applicationContext.xml</param-value>
</context-param>

<!-- 2.配置前端控制器 -->
<servlet>
    <servlet-name>dispatcherServlet</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:spring-mvc.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>dispatcherServlet</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>

<!-- 3.配置字符过滤器 -->
<filter>
    <filter-name>characterEncodingFilter</filter-name>
    <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
    <init-param>
        <param-name>encoding</param-name>
        <param-value>utf-8</param-value>
    </init-param>
</filter>
<filter-mapping>
    <filter-name>characterEncodingFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

## 四.使用Maven搭建SSM聚合工程

#### 1.创建父工程

##### 第一步:创建父工程

​	创建父工程时不选择任何的maven骨架,使用其默认的(java项目)即可。

##### 第二步:配置父工程的pom文件

```xml
<!--1.父工程的打包方式：pom-->
<packaging>pom</packaging>
<!-- 2.集中定义依赖版本号 -->
<properties>
    <junit.version>4.12</junit.version>
    <spring.version>5.0.2.RELEASE</spring.version>
    <pagehelper.version>5.1.2</pagehelper.version>
    <servlet-api.version>2.5</servlet-api.version>
    <mybatis.version>3.2.8</mybatis.version>
    <mybatis.spring.version>1.2.2</mybatis.spring.version>
    <mysql.version>5.1.32</mysql.version>
    <druid.version>1.0.9</druid.version>
    <commons-fileupload.version>1.3.1</commons-fileupload.version>
    <activemq.version>5.11.2</activemq.version>
</properties>
<!-- 3.导入依赖 -->
<dependencies>
    <!-- Spring -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>${spring.version}</version>
    </dependency>
    ...
</dependencies>
```

#### 2.创建子模块

​	父工程上右击,创建子模块，（只将ssm_web模块建为web骨架即可，其他默认（java））。

-| ssm_parent

​	-| ssm_common

​	-| ssm_domain

​	-| ssm_dao

​	-| ssm_service

​	-| ssm_web

##### 第一步：创建各模块建的依赖关系

修改各个模块建的pom文件，添加依赖关系（利用依赖的传递性，简化结构）。

##### 第二步：在各模块下配置各模块的配置文件

|- ssm_dao

​		|-resources

​			|-cn....                 // ..Dao.xml 映射文件

​			|- jdbc.properties

​			|- spring/applicaitonContext-dao.xml  //只做关于dao层的相关配置

|- ssm_service

​		|- resources

​			|-spring/applicationContext-tx.xml  //只做关于service层的相关配置

|- ssm_web

​		|- resources

​			|- spring/spring-mvc.xml     //只做web层的相关配置

##### 第三步：修改web.xml文件中的路径

```xml
<!-- 配置监听器 -->
  <listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
  </listener>
  <context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath*:spring/applicationContext-*.xml</param-value>
  </context-param>

  <!-- 配置前端控制器 -->
  <servlet>
    <servlet-name>dispatcherServlet</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
      <param-name>contextConfigLocation</param-name>
      <param-value>classpath:spring/spring-mvc.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
  </servlet>
```

**Tips:** 父工程不写代码的，只在pom文件中对依赖做统一限定。配置文件也分模块后，在web.xml中配置监听器时需要使用通配符* 进行匹配，确保所有的配置文件可以加载。在service层配置文件中会需要用到dao层中配置文件的引用，在编译阶段会报错，但是运行阶段不会。



