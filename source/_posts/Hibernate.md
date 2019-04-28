---
title: Hibernate
date: 2018-10-14 23:20:50
categories: java
description: Hibernate
---

# Hibernate
## Product.hbm.xml文件
作用是将javabean对应数据库中的表
```
<?xml version="1.0"?>
   <!DOCTYPE hibernate-mapping PUBLIC
           "-//Hibernate/Hibernate Mapping DTD 3.0//EN"
           "http://www.hibernate.org/dtd/hibernate-mapping-3.0.dtd">
    
   <hibernate-mapping package="com.how2java.pojo">
       <class name="Product" table="product_">//表示类Product对应表product_

           <id name="id" column="id">//表示属性id,映射表里的字段id
               <generator class="native">//意味着id的自增长方式采用数据库的本地方式
               </generator>
           </id>
           <property name="name" />//这里没有通过column="name" 显式的指定字段），因为属性和字段同名
           <property name="price" />
       </class>
        
   </hibernate-mapping>
 ```


##  hibernate.cfg.xml文件
访问数据库配置。注意，这个文件要放在**src目录**下
```
<?xml version='1.0' encoding='utf-8'?>
<!DOCTYPE hibernate-configuration PUBLIC
       "-//Hibernate/Hibernate Configuration DTD 3.0//EN"
"http://www.hibernate.org/dtd/hibernate-configuration-3.0.dtd">
 
<hibernate-configuration>
 
    <session-factory>
        <!-- Database connection settings -->
        <property name="connection.driver_class">com.mysql.jdbc.Driver</property>
        <property name="connection.url">jdbc:mysql://localhost:3306/test?characterEncoding=UTF-8</property>//这表示使用MYSQL方言
        <property name="connection.username">root</property>
        <property name="connection.password">admin</property>
        <!-- SQL dialect -->
        <property name="dialect">org.hibernate.dialect.MySQLDialect</property>
        <property name="current_session_context_class">thread</property>
        <property name="show_sql">true</property>//这表示是否在控制台显示执行的sql语句
        <property name="hbm2ddl.auto">update</property>//这表示是否会自动更新数据库的表结构，有这句话Hibernate会自动去创建表结构
        <mapping resource="com/how2java/pojo/Product.hbm.xml" />//识别Product这个实体类
    </session-factory>
 
</hibernate-configuration>
```
## 测试类

hibernate的基本步骤是：
1. 获取SessionFactory 
2. 通过SessionFactory 获取一个Session
3. 在Session基础上开启一个事务
4. 通过调用Session的save方法把对象保存到数据库
5. 提交事务
6. 关闭Session
7. 关闭SessionFactory
```
SessionFactory sf = new Configuration().configure().buildSessionFactory();//1.获取SessionFactory

Session s = sf.openSession();//2.通过SessionFactory 获取一个Session
s.beginTransaction();//3.在Session基础上开启一个事务

Product p = new Product();
p.setName("iphone7");
p.setPrice(7000);
s.save(p);//4. 通过调用Session的save方法把对象保存到数据库

s.getTransaction().commit();//5. 提交事务
s.close();//6. 关闭Session
sf.close();//7. 关闭SessionFactory
```