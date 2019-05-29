# SpringMVC -- 必知必会

​	SpringMVC基于模型--视图--控制器（Model-View-Controller，MVC）模式实现，属于SpringFrameWork的后续产品，已经融合在SpringWebFlow里面。它通过一套注解，让一个简单的Java类成为处理请求的控制器，而无需实现任何接口。同时它还支持RESTful编程风格的请求。SpringMVC是基于方法设计的，相比基于类设计的Struts2要稍微快一些。

## 一.使用步骤

##### 第一步：导入jar包或Maven坐标

```xml
<!-- 这里导入spring-webmvc即可，会自动导入它依赖的其他jar包 -->
<dependency>
   <groupId>org.springframework</groupId>
   <artifactId>spring-webmvc</artifactId>
   <version>5.0.2.RELEASE</version>
</dependency>
```

##### 第二步：修改并配置web.xml文件

```xml
<!-- 修改配置文件头约束为如下：（可到tomcat的web.xml中复制 -->
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
                      http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
         version="3.1">
    
    <!-- 1.配置核心控制器 -->
    <servlet>
        <servlet-name>dispathcerServlet</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <!-- 配置核心控制器的初始化参数,指定spring的配置文件 -->
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:webApplicationContext.xml</param-value>
        </init-param>
        <!-- servlet初始化时机:服务器启动第一个加载 -->
        <load-on-startup>1</load-on-startup>
    </servlet>
    
    <!-- 2.配置映射路径 -->
    <servlet-mapping>
        <servlet-name>dispathcerServlet</servlet-name>
        <!-- / 代表默认,用其可使用spring的REST风格的url -->
        <url-pattern>/</url-pattern>
    </servlet-mapping>
    
    <!-- 3.配置编码过滤器：解决post中文乱码问题 -->
    <filter>
        <filter-name>characterEncodingFilter</filter-name>
        <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
        <!-- 设置编码方式 -->
        <init-param>
            <param-name>encoding</param-name>
            <param-value>UTF-8</param-value>
        </init-param>
    </filter>
    <filter-mapping>
        <filter-name>characterEncodingFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>

</web-app>
```

##### 第三步：创建SpringMVC的配置文件

###### webApplicationContext.xml

```xml
<!-- 1.指定要扫描的包 -->
<context:component-scan base-package="cn.dintalk"/>
<!-- 2.开启mvc的注解驱动-->
<mvc:annotation-driven/>

<!-- 3.配置解析器 -->
<bean id="viewResolver" class="org.springframework.web.servlet.view.InternalResourceViewResolver">
    <property name="prefix" value="/WEB-INF/jsp/"/>
    <property name="suffix" value=".jsp"/>
</bean>
```

##### 第四步：编写控制器（POJO)   ★★★

```java
@Controller
@RequestMapping("/user") //适合模块化开发(不加也可以)
public class HelloController {
   //1. 根据url地址进行url映射(handlerMapping)
    @RequestMapping("/hello")
    public String hello(){
        System.out.println("我的第一个处理器执行了..");
        return "hello";
    }
```

##### 第五步：编写页面  ★★★

我们在页面中的链接如 .../user/hello便会访问到该方法

##### 总结：完成一次请求的过程

![QQ拼音截图未命名](C:\Users\Administrator\Desktop\QQ拼音截图未命名.png)





## 二.URL映射

​	url映射的规则，主要在添加@RequestMapping注解时指定

```java
//1. 指定单个url地址映射(handlerMapping)
@RequestMapping("/hello")
public String hello(){
    
//2. 指定多个url地址映射，但需保证不能和其他方法有重复的。
//思考一下为什么。
@RequestMapping({"/hello2","hello1"})
public String hello1(){
    
//3. 根据请求方式进行映射
@RequestMapping(path = "/hello3",method = RequestMethod.POST)
public String hello3(){
//若当前处理器添加了模块映射：如/user，那么该方法会拦截到所有以POST方式
//请求到该处理器（请求路径以/user结尾，即未定位到指定方法）的请求
@RequestMapping(method = RequestMethod.POST)
    public String hello4(){
        
//4. 根据请求参数进行映射
//params指定后，若不传参则会报错
@RequestMapping(path = "/hello5",params = {"name","password"})
    public String hello5(String name,String password){
        
//5. 根据请求消息头进行映射
//不携带cookie消息头就会报错
@RequestMapping(path = "/hello6",headers = {"cookie"})
    public String hello6(){
        
//6. 根据请求的正文mime类型进行映射
//只有post请求才会有正文
@RequestMapping(path = "/hello7",
    method = RequestMethod.POST,consumes = "multipart/form-data")
    public String hello7(){
```

## 三.请求参数的封装

#### 1.简单类型的封装

```java
//处理器的方法,springmvc会自动进行数据类型转换，转换失败则报错
@RequestMapping(path = "/param",params ={"name","age"} )
public String param(String name, Integer age){
//jsp页面
<a href="${pageContext.request.contextPath}/param?name=song&age=21">简单参数的封装</a>        
```

#### 2.数组的封装

```java
//处理器的方法
@RequestMapping(path = "/param1")
public String param1(String[] myAr){
//jsp页面
<form  action="${pageContext.request.contextPath}/param1" method="post">
    arr:<input name="myArr" value="song">
    arr:<input name="myArr" value="hui">
    <%-- hui1会覆盖掉hui --%>
    arr:<input name="myArr[1]" value="hui1">
    arr:<input name="myArr" value="慧">
    <input type="submit" value="封装数组myArr">
</form>
//页面传参数时，后边指定角标的会将前面的覆盖掉
```

#### 3.POJO的封装

```java
//处理器的方法
@RequestMapping(path = "/param2",params ={"name","age"} )
public String param2(User user){
//jsp页面
<a href="${pageContext.request.contextPath}/param2?name=song&age=21">POJO的封装</a>
```

#### 4.封装POJO关联的POJO

```java
//处理器的方法
@RequestMapping(path = "/param3")
public String param3(User user){
//jsp页面
<form  action="${pageContext.request.contextPath}/param3" method="post">
    name:<input name="name" value="song"/>
    name:<input name="password" value="hui"/>
    address:<input name="address.province" value="bj">
    address:<input name="address.city" value="zjk">
    <input type="submit" value="封装pojo关联的pojo">
</form>
//封装user中关联的address
```

#### 5.封装POJO关联的数组

```java
//处理器的方法 
@RequestMapping(path = "/param4")
public String param4(User user){
//jsp页面
<form  action="${pageContext.request.contextPath}/param4" method="post">
    name:<input name="name" value="song"/>
    arr:<input name="myArr" value="s">
    arr:<input name="myArr" value="hui">
    <%-- hui1会覆盖掉hui --%>
    arr:<input name="myArr[1]" value="hui1">
    <input type="submit" value="封装pojo关联的arr">
</form>
//页面传参数时，后边指定角标的会将前面的覆盖掉
```

#### 6.封装POJO关联的List

```java
//处理器的方法
@RequestMapping(path = "/param5")
public String param5(User user){
//jsp页面
<form  action="${pageContext.request.contextPath}/param5" method="post">
    name:<input name="name" value="song"/>
    list:<input name="myList" value="1">
    list:<input name="myList" value="hui">
    <input type="submit" value="封装pojo关联的list">
</form>
```

#### 7.封装POJO关联的Map

```java
//处理器的方法
 @RequestMapping(path = "/param6")
public String param6(User user){
//jsp页面
<form  action="${pageContext.request.contextPath}/param6" method="post">
    name:<input name="name" value="song"/>
    map:<input name="myMap['s']" value="song">
    map:<input name="myMap[s1]" value="hui">
    <input type="submit" value="封装pojo关联的map">
</form>
//页面传参数指定map的key可用单引号也可不用
```

## 四.请求的转发和重定向

```java
//请求转发
@RequestMapping("/hello")
public String hello(){
    System.out.println("我的第一个处理器执行了..");
    return "hello";  // 服务器默认用的是请求转发
}

@RequestMapping("/hello1")
public String hello1(){
    System.out.println("我的第一个处理器执行了..");
    // 自己转发的话，必须用实际视图地址
    return "forward:/WEB-INF/pages/hello.jsp";  
}
//请求重定向
@RequestMapping("/hello2")
public String hello2(){
    System.out.println("我的第一个处理器执行了..");
    // 重定向
    return "redirect:http://www.baidu.com";  
}
```

## 五.SpringMVC下静态资源的访问

​	我们在Spring MVC框架中为了是URL更符合RESTful风格，通常在web.xml中会配置Spring框架servlet 的 url 拦截为  "/"  ,也就是拦截所有资源的url请求，这样一来，所有的资源包括， js  | css |  图片  |  所有静态资源都将经过框架的servlet拦截 。而又没有对应的处理器，因此会找不到资源（404）。因此我们要对静态资源放行：

##### 第一种方法：激活web应用服务器（如Tomcat）的defaultServlet来处理静态文件

###### web.xml中加入以下配置

```xml
<servlet-mapping>  
    <servlet-name>default</servlet-name>  
    <url-pattern>*.jpg</url-pattern>  
</servlet-mapping>  
<servlet-mapping>  
    <servlet-name>default</servlet-name>  
    <url-pattern>*.js</url-pattern>  
</servlet-mapping>  
<servlet-mapping>  
    <servlet-name>default</servlet-name>  
    <url-pattern>*.css</url-pattern>  
</servlet-mapping>  
```

**Tips:** 要配置多个，每种文件配置一个,要写在DispatcherServlet的前面,让defaultServlet先拦截，这样就不会进入Spring了，可能性能是最好的。

##### 第二种方法：使用Spring3.0.4以后版本提供的  <mvc:resources />

###### webApplicationContext.xml中加入以下配置

```xml
<mvc:resources mapping="/static/**" location="/static/" />
<!-- 不加下面这句话可能会报错 -->
<mvc:annotation-driven />
```

**Tips:** location的值要配到最后一个/ ,如果这里只写到 /static 还是放行不了的。

##### 第三种方法：使用 <mvc:default-servlet-handler/ >

###### webApplicationContext.xml中加入以下配置

```java
<mvc:default-servlet-handler default-servlet-name="default"/>
```

##### **Tips：如果用后两种方法的话** 加载一个静态资源是却要经过框架servlet的层层pattern，会有不必要的性能开销。 但对于一些比较重要的静态文件，我们可以将其放在WEB-INF目录下保护起来（该目录下不可直接访问 )，但我们可以在服务端应用(请求转发)。

## 六.常用注解、异步交互和Restful风格的url

#### 1.常用注解

```java
//1. @RequestParam:可处理请求参数名和处理器方法参数名不一致的情况
@RequestMapping("demo1")
//RequestParam默认required=true,指必须提供,否则报错!将username参数赋值给name
public String demo1(@RequestParam(value = "username",required = false)String name){
  
//2. @RequestHeader:用指定消息头为处理器参数赋值
@RequestMapping("demo2")
public String demo2(@RequestHeader("accept-encoding")String encode){
    
//3. @CookieValue:获取cookie的值给处理器参数赋值
@RequestMapping("demo3")
public String demo3(@CookieValue("JSESSIONID")String JsessionId){

//4. @RequestBody:获取请求消息正文,注意只有post请求才有正文
@RequestMapping("demo4")
public String demo4(@RequestBody String Content){

//5. @ModelAttribute:用在参数上，获取指定数据给参数赋值（无对应数据会报错）
@RequestMapping("demo5")
public String demo5(@ModelAttribute("user1") User user1){
    
//6. @ModelAttribute:用在方法上，会在控制器任意方法执行前执行
	//会把方法的返回值以指定名称存到域对象中。无返回值可通过隐式对象存数据
@ModelAttribute("user")
public User createUser(){

//7. @SessionAttributes:用在类定义上,将存放在model中对应的数据暂存到HttpSession 中。
    //会将model中所有类型为 User的属性添加到会话中。
@SessionAttributes(types=User.class)  
    //会将model中属性名为user1和user2的属性添加到会话中。
@SessionAttributes(value={“user1”, “user2”}) 
    //会将model中所有类型为 User和Dept的属性添加到会话中。
@SessionAttributes(types={User.class, Address.class}) 
    //会将model中属性名为user1和user2以及类型为Dept的属性添加到会话中。
@SessionAttributes(value={“user1”,“user2”},types={Address.class})
//需要注意的是,value和types之间是取并集的关系

//8. @ResponseBody:将方法返回的对象，通过 HttpMessageConverter接口转换为指定格式的
	//数据如：json,xml 等，通过 Response 响应给客户端
@RequestMapping("/demo8")
@ResponseBody  // 也可加到返回值前面，即User前面：public @ResponseBody User ...
public User demo8(){
 
//9. @PathVariable:绑定url中占位符如请求url中/delete/{id}，这个{id}就是url占位符。
    //将其赋给处理器参数，结合method=...,可实现rest风格的url
@RequestMapping(value = "/user/{id}",method = RequestMethod.GET)
public String find(@PathVariable("id") Integer uid){
```

**Tips:**    @ModelAttribute用在参数上是从请求域中获取指定数据赋给参数，用在方法上是每一次访问该处理器的任意方法前都会执行该方法。可将所修饰的方法返回值以指定名称存入请求域中。

#### 2.异步交互

​	使用SpringMVC如何完成ajax的异步交互呢？很简单，我们主要使用它的两个注解来实现:

##### 导入jackson的jar包或Maven坐标

```xml
<!-- 该包依赖的其他包会自动导入 -->
<dependency>
  <groupId>com.fasterxml.jackson.core</groupId>
  <artifactId>jackson-databind</artifactId>
  <version>2.9.0</version>
</dependency>
```

##### 2.1@RequestBody接受异步请求

###### 处理器方法

```java
@RequestMapping("/demo1")
public String demo1(@RequestBody String content) throws IOException {
    ObjectMapper objectMapper = new ObjectMapper();
    //使用jackson解析json数据封装到pojo中
    User user = objectMapper.readValue(content, User.class);
    System.out.println(user);
    System.out.println("异步请求到了");
    return "success";
}
```

###### 异步请求页面

```js
$(function () {
   $("#b1").click(function () {
       $.ajax({
           type: "POST",
           url: "${pageContext.request.contextPath}/json/demo1",
           dataType: "json",
           contentType: "application/json",
           data: '{"name":"song","password":"hui"}'
       });
   });
})
```

**Tips**: 如果不加入contentType:"application/json"会解析失败，因为加入它才会发送json格式的数据。

##### 2.2@ResponseBody返回Json数据

###### 处理器方法

```java
@RequestMapping("/demo2")
@ResponseBody
public User demo2() {
    User user = new User();
    user.setName("da");
    user.setUid(22);
    return user;
}
```

###### 异步请求页面

```js
$(function () {
    $("#b2").click(function () {
        $.ajax({
            type: "POST",
            url: "${pageContext.request.contextPath}/json/demo2",
            data: '{"name":"song","password":"hui"}',
            dataType: "json",
            contentType: "application/json",
            success: function (data) {
                alert(data.uid);
                alert(data.name);
            }
        });
    });
})
```

#### 3.Restful风格的url

##### 3.1restful的状态转化特性

​	HTTP 协议，是一个无状态协议，即所有的状态都保存在服务器端。因此，如果客户端想要操作服务器，必须通过某种手段，让服务器端发生 “状态转化 ”（ State Transfer）。而这种转化是建立在表现层之上的，所以就是  “表现层状态转化 ”。具体说，就是  HTTP 协议里面，四个表示操作方式的动词： GET、 POST、 PUT、DELETE。它们分别对应四种基本操作： GET 用来获取资源， POST 用来新建资源， PUT 用来更新资源， DELETE 用来删除资源。

**restful  的示例：**
/account/1 HTTP  GET 	：  	 得到 id = 1 的 account
/account/1 HTTP  DELETE	：	 删除 id = 1 的 account
/account/1 HTTP  PUT		： 	 更新 id = 1 的 account
/account HTTP  POST		：  	 新增 account

##### 3.2基于HiddenHttpMethodFilter使用@PathVariable注解构建rest风格的url

​	由于浏览器 form 表单只支持 GET 与 POST 请求，而 DELETE、PUT 等 method 并不支持，Spring3.0 添加了一个过滤器，可以将浏览器请求改为指定的请求方式，发送给我们的控制器方法，使得支持 GET、POST、PUT与 DELETE 请求。

###### 第一步：web.xml中配置过滤器

```xml
<!-- 配置过滤器将表单不支持的请求方式转化(支持restful风格) -->
<filter>
    <filter-name>hiddenHttpMethodFilter</filter-name>
    <filter-class>org.springframework.web.filter.HiddenHttpMethodFilter</filter-class>
</filter>
<filter-mapping>
    <filter-name>hiddenHttpMethodFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

###### 第二步：表单书写

```html
<!-- 对于不支持的请求方式,get，post之外的：form表单的method要指定为post
	并且要添加隐藏域。
-->
<form method="post" action="${pageContext.request.contextPath}/user">
    username:<input name="name" value="admin"/><br/>
    <input type="submit" value="注册"/>
</form>
<form method="post" action="${pageContext.request.contextPath}/user/1">
    <input type="hidden" name="_method" value="PUT">
    username:<input name="name" value="admin1"/><br/>
    <input type="submit" value="修改"/>
</form>
<form method="get" action="${pageContext.request.contextPath}/user/1">
    <input type="submit" value="查询"/>
</form>
<form method="post" action="${pageContext.request.contextPath}/user/1">
    <input type="hidden" name="_method" value="DELETE">
    <input type="submit" value="删除"/>
</form>
```

###### 第三步：处理器方法

```java
@RequestMapping(value = "/user",method = RequestMethod.POST)
public String regist(User user){return "success";}
//{id}参数的占位符
@RequestMapping(value = "/user/{id}",method = RequestMethod.PUT)
public String edit(@PathVariable("id") Integer uid, User user){return "success";}
@RequestMapping(value = "/user/{id}",method = RequestMethod.GET)
public String find(@PathVariable("id") Integer uid){return "success";}
@RequestMapping(value = "/user1{id}",method = RequestMethod.DELETE)
public String delete(@PathVariable("id") Integer uid){return "success";}
```

## 七.原始servletAPI、隐式对象和处理器返回值

#### 1.原始servletAPI

​	要使用原始的servletAPI，我们在处理器方法中直接定义参数即可，框架会为我们传入对象。

##### 第一步：导入servlet的jar包或Maven坐标

```xml
<dependency>
  <groupId>javax.servlet</groupId>
  <artifactId>javax.servlet-api</artifactId>
  <version>3.1.0</version>
  <scope>provided</scope>
</dependency>
```

##### 第二步：处理器方法定义参数

```java
@RequestMapping("/demo1")
public String demo1(HttpServletRequest req, HttpServletResponse res){
    System.out.println(req+"==="+res);
```

#### 2.隐式对象

​	SpringMVC为我们提供好了默认的隐式对象，我们直接使用即可。使用Model,ModelMap和Map可以封装数据到模型对象中（request域级别）。同使用servletAPI一样，我们在处理器方法上定义参数即可使用。

```java
//Model对象
@RequestMapping("/demo2")
public String demo2(Model model){
 	User user = new User();
    user.setName("user2");
    model.addAttribute("user",user);
    return "success";
}
//ModelMap对象    
@RequestMapping("/demo3")
public String demo3(ModelMap model){
	User user = new User();
    user.setName("user3");
    model.addAttribute("user",user);
    return "success";
}   
//Map对象
@RequestMapping("/demo4")
public String demo4(Map model){
    User user = new User();
    user.setName("user4");
    model.put("user",user);
    return "success";
}
```

**Tips:** 当然，我们也可以在处理器方法内部直接通过new创建其对象来使用。

#### 3.处理器返回值

​	controller方法的返回值有三种：String、void、ModelAndView。

##### 3.1String

```java
// 返回字符串指定逻辑视图名称，通过视图解析器解析为
// 真实视图地址。
@RequestMapping("/demo1")
public String demo1(){
    return "success";
}
```

##### 3.2void

```java
// 方法无返回值，我们可以用servletAPI完成转发、重定向、响应。
// 转发
request.getRequestDispatcher("1.jsp").forward(request,response);
//重定向:这里重定向到了当前处理器的demo1方法
response.sendRedirect("demo1")
//直接响应：如json数据
response.setCharacterEncoding("utf-8");
response.setContentType("application/json;charset=utf-8");
response.getWriter().write("我是json串");   
```

##### 3.3ModelAndView

```java
 @RequestMapping("/demo3")
public ModelAndView demo3(){
    User user = new User();
    user.setName("haha");
    ModelAndView modelAndView = new ModelAndView();
    modelAndView.setViewName("success");//指定视图
    modelAndView.addObject("user",user);//存放数据
    return modelAndView;
}
```

## 八.自定义类型转化器、拦截器和异常

#### 1.自定义类型转换器

​	SpringMVC内置简单类型转换器，所以我们在进行参数封装时SpringMVC会根据类型为我们进行自动转换。开发中，我们可根据需求自定义类型转化器，通过配置加入到SpringMVC的转换器列表中即可。比如我们常常需要将用户输入的日期字符串转换成日期类型的，即String -->java.util.Date的转换。

##### 第一步：自定义类实现Converter<S, T>接口

```java
//自定义类实现了org.springframework.core.convert.converter.Converter 接口
public class DateLocalConvertor implements Converter<String, Date> {
    public String pattern = "yyyy-MM-dd";
	//定义默认日期格式，并提供set方法使其可在配置文件中配置
    public void setPattern(String pattern) {
        this.pattern = pattern;
    }
	//实现接口的转换方法：Source -->Target
    @Override
    public Date convert(String s) {
        if (!StringUtils.isEmpty(s)) {
            try {
                SimpleDateFormat simpleDateFormat = new SimpleDateFormat(pattern);
                Date date = simpleDateFormat.parse(s);
                return date;
            } catch (ParseException e) {
                throw new RuntimeException("您输入的日期格式必须符合"+pattern+"格式");
            }
        } else {
            return null;
        }
    }
}
```

##### 第二步：注册自定义类型转换器到SpringMVC中

###### webApplicationContext.xml

```xml
<!-- 配置类型转换器工厂 -->
<bean id="converterService" class="org.springframework.context.support.
                                   ConversionServiceFactoryBean">
    <!-- 1.相当于调用了set方法,给工厂注入新的类型转换器  -->
    <property name="converters">
        <!-- 若将来有多个自定义的类型转化器,可以用array标签包裹多个bean -->
        <bean class="cn.dintalk.convertor.DateLocalConvertor"/>
    </property>
</bean>
<!-- 引用自定义类型转化器 -->
<mvc:annotation-driven conversion-service="converterService"/>
```

##### 第三步：使用

​	如此，我们在前端页面输入日期字符串后，处理器方法在进行封装时会用到我们自定义的类型转换器。否则没有日期类型转换器的话，而又有日期类型的数据封装的话会报400错误。

#### 2.自定义拦截器

​	SpringMVC的处理器拦截器类似于Servlet中的过滤器Filter，用于对处理器进行预处理和后处理。但又有区别：

- 过滤器：是servlet中的规范，任何java web工程都可使用。
- 拦截器：是SpringMVC框架的，使用了SpringMVC的工程才能用。
- 过滤器：配置了url-pattern为 /*后，可拦截所有的资源访问。
- 拦截器：只拦截访问的控制器的方法，不拦截jsp、html、img等。

我们要自定义拦截器，就必须实现接口：HandlerInterceptor。

##### 第一步：编写普通类实现HandlerInterceptor接口

```java
public class MyInterceptor implements HandlerInterceptor {
    @Override    
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        System.out.println("到达处理器前的方法执行了");
        return true; // 返回true这放行,否则拦截
    }
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        System.out.println("处理器方法执行之后执行的方法执行了");
    }
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        System.out.println("页面完成响应之前的方法执行了");
    }
}
```

##### 第二步：在webApplicationContext.xml中配置拦截器

```xml
<!-- 配置拦截器 -->
<mvc:interceptors>
    <mvc:interceptor>
        <!-- 拦截/inter开头的访问路径 -->
        <mvc:mapping path="/inter/**"/>
        <bean class="cn.dintalk.web.interceptor.MyInterceptor"/>
    </mvc:interceptor>
    <mvc:interceptor>
        <!-- 拦截/user开头的访问路径 -->
        <mvc:mapping path="/user/**"/>
        <!-- 排除/user/login的访问路径(不进行拦截) -->
        <mvc:exclude-mapping path="/user/login"/>
        <bean class="cn.dintalk.web.interceptor.CheckUserInterceptor"/>
    </mvc:interceptor>
</mvc:interceptors>
```

**Tips:** 自定义拦截器一般实现第一个方法,即在请求到达处理器前进行拦截作相应处理以决定是否放行;多个处理器的执行顺序按照其配置的上下顺序。

#### 3.自定义异常

##### 3.1异常处理思路

​	系统中异常包括两类：预期异常和运行时异常 RuntimeException，前者通过捕获异常从而获取异常信息，后者主要通过规范代码开发、测试通过手段减少运行时异常的发生。系统的 dao、service、controller 出现都通过 throws Exception 向上抛出，最后由 Springmvc 前端控制器交由异常处理器进行异常处理。

##### 3.2编写自定义异常类和错误页面

###### 自定义异常继承Exception,生成构造方法即可

```java
public class CustomException extends Exception {
    public CustomException() {
    }
    public CustomException(String message) {
        super(message);
    }
    public CustomException(String message, Throwable cause) {
        super(message, cause);
    }
    public CustomException(Throwable cause) {
        super(cause);
    }
    public CustomException(String message, Throwable cause, boolean enableSuppression, boolean writableStackTrace) {
        super(message, cause, enableSuppression, writableStackTrace);
    }
}
```

###### 编写错误页面error.jsp用以展示提示信息。

##### 3.3自定义异常处理器

```java
//编写普通类实现org.springframework.web.servlet.HandlerExceptionResolver接口
public class CustomExceptionResolver implements HandlerExceptionResolver {
@Override
public ModelAndView resolveException(HttpServletRequest httpServletRequest,
                                     HttpServletResponse httpServletResponse,
                                     Object o, Exception e) {
    CustomException exception = null;
    if (e instanceof CustomException){//若是自定义异常则转换
        exception = (CustomException)e;//使用时就封装好了消息
    }else {//其他的异常,用自定义异常提示系统繁忙
        exception = new CustomException("系统繁忙");
    }
    ModelAndView modelAndView = new ModelAndView();
    modelAndView.setViewName("error");
    modelAndView.addObject("msg",exception.getMessage());
    return modelAndView;
}
}
```

##### 3.4配置异常处理器

```xml
<!-- 配置异常处理器 -->
<bean id="customExceptionResolver" class="cn.dintalk.web.resolver.CustomExceptionResolver"/>
```

如此,当程序发生异常的时候,我们就可以截获异常,给用户一个友好的提示。

## 九.文件的上传

​	要实现上传文件，对form表单有一定的要求：

- method必须是：post
- 表单的enctype：必须是 "multipart/form-data"
- 表单中提供type="file"的上传输入域

**Tips:** 表单的enctype默认值为：application/x-www-form-urlencoded。修改为"multipart/form-data"后request.getParameter()等方法便获取不到数据了。

#### 1.准备上传页面

```html
<h1>上传练习</h1>
<form enctype="multipart/form-data" method="post" 
      action="${pageContext.request.contextPath}/upload/demo1">
    name:<input type="text" name="username" value="宋hui">
    file:<input type="file" name="photo">
    <input type="submit" value="上传">
</form>
```

#### 2.使用Commons-fileupload组件实现上传

##### 第一步：导入jar包或Maven坐标

```xml
<!-- 这里会自动导入其所依赖的commos-io包 -->
<dependency>
  <groupId>commons-fileupload</groupId>
  <artifactId>commons-fileupload</artifactId>
  <version>1.3.1</version>
</dependency>
```

##### 第二步：编写上传控制器

```java
@Controller
@RequestMapping("/upload")
public class UploadController {
    @RequestMapping("/demo1")
    public String demo1(HttpServletRequest request) throws Exception {
        //1.创建文件保存目录
        String rootDir = request.getServletContext().getRealPath("files");
        String chiledDir = new SimpleDateFormat("yyyy-MM-dd").format(new Date());
        File dir = new File(rootDir, chiledDir);
        if (!dir.exists())
            dir.mkdirs();
        //2.创建文件解析工厂
        DiskFileItemFactory diskFileItemFactory = new DiskFileItemFactory();
        //3.获取解析器
        ServletFileUpload upload = new ServletFileUpload(diskFileItemFactory);
        List<FileItem> fileItems = upload.parseRequest(request);
        for (FileItem fileItem : fileItems) {
            if (fileItem.isFormField()) {//是普通字段
                System.out.println(fileItem.getFieldName());//字段的name
                System.out.println(fileItem.getString("utf-8"));//字段值
            } else { // 上传的文件
                String fileName = fileItem.getName();//获取文件名
                String exName = fileName.substring(fileName.lastIndexOf("."));//截取扩展名
                String uuidName = UUID.randomUUID().toString().replace("-", "") + exName;
                System.out.println(uuidName);
                File target = new File(dir, uuidName);
                fileItem.write(target); //写入文件
                fileItem.delete();//清除临时目录中的缓存
            }
        }
        return "success";
    }
}
```

#### 3.使用SpringMVC提供的组件上传

​	底层使用的还是apache的commons-fileupload组件。

##### 第一步：导入jar包或Maven坐标

```xml
<!-- 这里会自动导入其所依赖的commos-io包 -->
<dependency>
  <groupId>commons-fileupload</groupId>
  <artifactId>commons-fileupload</artifactId>
  <version>1.3.1</version>
</dependency>
```

##### 第二步：编写上传控制器

```java
@Controller
@RequestMapping("/upload")
public class UploadController {
    @RequestMapping("/demo2")
    //参数名称需和表单的输入域名称保持一致
    public String demo2(HttpServletRequest request, String username, MultipartFile photo) throws IOException {
        //1.创建目录
        String rootDir = request.getServletContext().getRealPath("files");
        String chiledDir = new SimpleDateFormat("yyyy-MM-dd").format(new Date());
        File dir = new File(rootDir, chiledDir);
        if (!dir.exists())
            dir.mkdirs();
        //2.读取数据进行保存
        String fileName = photo.getOriginalFilename();//获取文件名
        String exName = fileName.substring(fileName.lastIndexOf("."));//截取扩展名
        String uuidName = UUID.randomUUID().toString().replace("-", "") + exName;
        System.out.println(uuidName);
        File target = new File(dir, uuidName);
        photo.transferTo(target);//存放文件
        return "success";
    }
}
```

##### 第三步：配置文件解析器

###### webApplicationContext.xml文件

```xml
<!-- 上传文件解析器配置：id值是固定的 -->
<bean id="multipartResolver" 
      class="org.springframework.web.multipart.commons.CommonsMultipartResolver">
    <!-- 设置默认编码 -->
    <property name="defaultEncoding" value="utf-8"/>
    <!-- 最大文件大小：字节为单位，这里配置文5M -->
    <property name="maxUploadSize" value="5242880"/>
</bean>
```
