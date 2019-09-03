---
title: javaEE
date: 2018-10-04 22:20:50
categories: java
description: javabean、servlet、jsp
---

# javabean
三要素
1.一个无参的构造函数
2.属性私有化。
3.get、set的方法。私有化的属性必须通过public类型的方法给其它程序，并且方法的命名也必须遵守一定的命名规范。

# servlet
## 写一个servlet
第一步：html表单提交请求
`<form action="login" method="post">`

第二步：servlet类中的请求处理
```
public class LoginServlet extends HttpServlet {
    protected void doPost(HttpServletRequest request, HttpServletResponse response)
```

第三步：配置web.xml
```
<servlet-name>L1</servlet-name>//类的缩写
    <servlet-class>Myservlet.LoginServlet</servlet-class>//类的名称
</servlet>

<servlet-mapping>
    <servlet-name>L1</servlet-name>//类的缩写，和上面一致就行
    <url-pattern>/login</url-pattern>//映射为的url地址，注意有“/”。对应<form action="login"
</servlet-mapping>
```
## 处理请求的三种方法
1、dopost、doget、service
doget：是默认方法，比如超链访问、地址栏直接输入某个地址。
特点：1.不安全，提交数据会在浏览器地址栏显示出来
2.不可以上传文件
3.有大小的限制

dopost：一般用于注册、修改请求
特点：1.安全
2.可以上传文件
3.没有限制


## request和reponse常用方法
### request


**request.getParameter(): 是常见的方法，用于获取单值的参数**
request.getParameterValues(): 用于获取具有多值得参数，比如注册的时候提交的爱好，可以使多选的。
request.getParameterMap(): 用于遍历所有的参数，并返回Map类型。

request.getRequestURL(): 浏览器发出请求时的完整URL，包括协议 主机名 端口(如果有)" +
request.getRequestURI(): 浏览器发出请求的资源名部分，去掉了协议和主机名" +
request.getQueryString(): 请求行中的参数部分，只能显示以get方式发出的参数，post方式的看不到
request.getRemoteAddr(): 浏览器所处于的客户机的IP地址
request.getRemoteHost(): 浏览器所处于的客户机的主机名
request.getRemotePort(): 浏览器所处于的客户机使用的网络端口
request.getLocalAddr(): 服务器的IP地址
request.getLocalName(): 服务器的主机名
request.getMethod(): 得到客户机请求方式一般是GET或者POST

request.getHeader() 获取浏览器传递过来的头信息。比如getHeader("user-agent") 可以获取浏览器的基本资料，这样就能判断是firefox、IE、chrome、或者是safari浏览器
request.getHeaderNames() 获取浏览器所有的头信息名称，根据头信息名称就能遍历出所有的头信息


### reponse(待完善)
用于提供给浏览器的响应信息

## 服务端跳转和客户端跳转

服务端跳转
`request.getRequestDispatcher("success.html").forward(request, response);`

客户端跳转
`response.sendRedirect("fail.html");`

区别在于**客户端跳转时**候浏览器地址发生了变化


## 中文问题
### 获取中文参数，只需三步
1. login.html中加上
 
`<meta http-equiv="Content-Type" content="text/html; charset=UTF-8">`
 
这句话的目的是告诉浏览器，等下发消息给服务器的时候，使用UTF-8编码

2. login.html
form的method修改为post

3. 在servlet进行解码和编码
 把下面这句代码放在request.getParameter()之前
`request.setCharacterEncoding("UTF-8"); `
这句话的目的UTF-8解码，然后用UTF-8编码

###  返回中文的响应
在Servlet中，加上
`response.setContentType("text/html; charset=UTF-8");`


## servlet的一些概念
### 生命周期
servlet生命周期有5步：实例化，初始化，提供服务，销毁，被回收 
初始化：（ init(ServletConfig) 实例方法，只会执行一次）
LoginSerlvet构造方法 只会执行一次，所以Serlvet是单实例的

## servlet和jsp
jsp就是在html里面写java代码，servlet就是在java里面写html代码

jsp更注重前端显示，servlet更注重模型和业务逻辑

jsp经过容器解释之后就是一个servlet类.
我们说HelloServlet是一个Servlet，不是因为它的类名里有一个"Servlet"，而是因为它继承了 HttpServlet
打开转译hello.jsp 后得到的hello_jsp.java，可以发现它继承了类HttpJspBase，而HttpJspBase 继承了HttpServlet
所以我们说hello_.jsp.java 是一个Servlet

# JSP

## 页面元素
```
1. 静态内容
就是html,css,javascript等内容
2. 指令
以<%@开始 %> 结尾，比如<%@page import="java.util.*"%>
3. 表达式 <%=%>
4. Scriptlet
在<%%> 之间，可以写任何java 代码
5. 声明
在<%!%> 之间可以声明字段或者方法。但是不建议这么做。
6. 动作
<jsp:include page="Filename" > 在jsp页面中包含另一个页面。在包含的章节有详细的讲解
7. 注释 <%-- -- %>
不同于 html的注释 <!-- --> 通过jsp的注释，浏览器也看不到相应的代码，相当于在servlet中注释掉了

8. 表达式
用于输出一段html，比如<%="hello jsp"%>相当于<%out.println("hello jsp");%>
再比如
<%for (String word : words) {%>
<tr>
    <td><%=word%></td>
</tr>
<%}%>
```
## 九大内置对象

request,response,out

pageContext, session,application作用域：页面、用户、全局

page,config,exception

使用javabean

# session
设置
`session.getAttribute("name");`
取得
`String name = (String)session.getAttribute("name");`

# Cookie
```
//创健
Cookie u= new Cookie("username",username);键，值对的关系存储`
//设置生存时间
u.setMaxAge(864000);`
//保存 
response.addCookie(u);`

//把所有cookie取出来装入数组 Cookie[] cookies = request.getCookies();
if(c.getName().equals("username")||c.getName().equals("password"))
  {c.setMaxAge(0); //设置Cookie失效
  response.addCookie(c); //重新保存。

//取键 
if(c.getName().equals("username"))
 
//取值
username = URLDecoder.decode(c.getValue(),"utf-8");
  ```