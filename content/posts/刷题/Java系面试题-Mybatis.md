---
title: "Java系面试题-Mybatis"
date: 2020-04-23T16:24:17+08:00
draft: false
featured_image: "https://oss.likeli.top/uPic/20200423191221!smail_img_likeli"
tags: [面试题, Mybatis]
categories: 面试题
---

## 什么是Mybatis？

1、Mybatis是一个半ORM框架，它内部封装了JDBC，开发时需要关注SQL语句本身，不需要话费精力去处理加载驱动、创建连接、创建statement等繁杂的过程。程序员直接编写原生的sql，可以严格控制sql执行性能，灵活度高。

2、Mybatis可以使用XML或者注解来配置和映射原生信息，将POJO映射成数据库的记录，避免了几乎所有的JDBC代码和手动设置参数以及获取结果集。

3、通过XML文件或主机的方式将要执行的各种statement配置起来，并通过java对象和statement中sql的动态参数进行映射成最终执行的SQL语句，最后由Mybatis框架执行sql并将结果映射为java对象并返回。（从执行sql到返回result的过程）。

## Mybatis的优点

1、基于SQL语句编程，相当灵活，不会对应用程序或者数据库的现有设计操作任何影响，SQL写在XML里，解除SQL与程序代码的耦合，便于统一管理；提供XML标签，支持编写动态SQL语句，并可重用。

2、与JDBC相比，减少了50%以上的代码量，消除了JDBC大量冗余代码，不需要手动开关连接；

3、很好的与各种数据库兼容（因为Mybatis使用JDBC来连接数据库，所以只要JDBC支持的数据库Mybatis都支持）。

4、能够与Spring很好的集成；

5、提供映射标签，支持对象与数据库的ORM字段关系映射；提供对象关系映射标签，支持对象关系组件维护；

## Mybatis框架的缺点

1、SQL语句编写的工作量较大，尤其当字段多，关联表多时，对开发人员编写SQL语句的功底有一定的要求；

2、SQL语句依赖数据库，导致数据库移植性差，不能随意更换数据库；

## Mybatis框架适用场合

1、Mybatis专注于SQL本身，是一个足够灵活的DAO层解决方案。

2、对性能要求很高，或者需求变化较多的项目，比如互联网项目，Mybatis将是不错的选择。

## Mybatis与Hibernate有哪些不同？

1、Mybatis和Hibernate不同，它不完全是一个ORM框架，因为Mybatis需要程序员自己编写SQL语句。

2、Mybatis只写编写原生态的SQL，可以严格控制SQL执行性能，灵活度高，非常适合对关系数据模型要求不高的软件开发，因为这类软件需求变化频繁，一旦需求变化要求迅速输出成果。但是灵活的前提是Mybatis无法做到数据库无关性，如果需要实现支持多种数据库的软件，则需要自定义多套SQL映射文件，工作量大。

3、Hibernate对象/关系映射能力强，数据库无关性好，对于关系模型要求高的软件，如果用Hibernate开发可以节省很多代码，提高效率。

## #{}和${}的区别是什么？

\#{}是预编译处理，${}是字符串替换。

> Mybatis在处理\#{}时，会将SQL中的\#{}替换为`?`号，调用PreparedStatement的set方法来赋值；
>
> Mybatis在处理${}时，就是把\${}替换成变量的值。
>
> 使用\#{}可以有效的防止SQL注入，提高系统安全性。



## 当实体类中的属性名和表中的字段名不一样，怎么办？

第 1 种： 通过在查询的 sql 语句中定义字段名的别名，让字段名的别名和实体类的属性名一致。

```xml
<select id=”selectorder” parametertype=”int” resultetype=”me.gacl.domain.order”>
	select order_id id, order_no orderno ,order_price price form orders where order_id=#{id};
</select>
```

第 2 种： 通过\<resultMap\>来映射字段名和实体类属性名的一一对应的关系。

```xml
<select id="getOrder" parameterType="int" resultMap="orderresultmap">
	select * from orders where order_id=#{id}
</select>

<resultMap type=”me.gacl.domain.order” id=”orderresultmap”>
  <!–用 id 属性来映射主键字段–>
  <id property=”id” column=”order_id”>
  <!–用 result 属性来映射非主键字段，property 为实体类属性名，column
  为数据表中的属性–>
  <result property = “orderno” column =”order_no”/>
  <result property=”price” column=”order_price” />
</reslutMap>
```

## 模糊查询like语句该怎么写？

第 1 种：在 Java 代码中添加 sql 通配符。

```java
String wildcardname = “%smi%”;
List<name> names = mapper.selectlike(wildcardname);
```

```xml
<select id=”selectlike”>
	select * from foo where bar like #{value}
</select>
```

第 2 种：在 sql 语句中拼接通配符，会引起 sql 注入

```java
String wildcardname = “smi”;
List<name> names = mapper.selectlike(wildcardname);
```

```xml
<select id=”selectlike”>
	select * from foo where bar like "%"#{value}"%"
</select>
```

## 通常一个XML映射文件，都会写一个DAO接口与之对应，请问，这个DAO接口的工作原理是什么？DAO接口里面的方法，参数不同时，方法能重载吗？

Dao 接口即 Mapper 接口。接口的全限名，就是映射文件中的 namespace 的值；接口的方法名，就是映射文件中 Mapper 的 Statement 的 id 值；接口方法内的参数，就是传递给 sql 的参数。

Mapper 接口是没有实现类的，当调用接口方法时，接口全限名+方法名拼接字符串作为 key 值，可唯一定位一个 MapperStatement。在 Mybatis 中，每一个\<select>、\<insert>、\<update>、\<delete>标签，都会被解析为一个MapperStatement 对象。

> 举例：com.mybatis3.mappers.StudentDao.findStudentById，可以唯一找到 namespace 为 com.mybatis3.mappers.StudentDao 下面 id 为findStudentById 的 MapperStatement。

Mapper 接口里的方法，是**不能重载**的，因为是使用**全限名+方法名** 的保存和寻找策略。Mapper 接口的工作原理是 JDK 动态代理，Mybatis 运行时会使用 JDK动态代理为 Mapper 接口生成代理对象 proxy，代理对象会拦截接口方法，转而执行 MapperStatement 所代表的 sql，然后将 sql 执行结果返回。

## Mybatis是如何进行分页的？分页插件的原理是什么？

Mybatis使用RowBounds对象进行分页，它是针对ResultSet结果集执行的**内存分页**，而**非物理分页**。

可以在sql内直接书写带有物理分页的参数来完成物理分页功能，也可以使用分页插件来完成物理分页。

分页插件的基本原理是使用Mybatis提供的插件接口，实现自定义插件，在插件的拦截方法内拦截待执行的sql，然后重写sql，根据dialect方言，添加对应的物理分页语句和物理分页参数。

## Mybatis是如何将SQL执行结果封装为目标对象并返回的？都有哪些映射形式？

第一种是使用\<resultMap>标签，逐一定义数据库列名和对象属性名之间的映射关系。

第二种是使用 sql 列的别名功能，将列的别名书写为对象属性名。

有了列名与属性名的映射关系后，Mybatis 通过反射创建对象，同时使用反射给对象的属性逐一赋值并返回，那些找不到映射关系的属性，是无法完成赋值的。



and so on ...