---
title: "记一次数据库查询时间参数转换格式错误的BUG"
date: 2023-03-12T00:57:52+08:00
draft: true
---
数据库：MySQ5.7  

最近新开了一个服务，上线后发现定时任务失效了。  

经排查，发现JPA生成sql的日期参数格式为`"dd/MM/yyyy HH:mm:ss"`。  

通常数据库查询的时间参数在生成SQL时，格式为`"yyyy-MM-dd HH:mm:ss"`。  

对源码进行调试，  

![这是图片](/images/a-bug/2.png "2")  

发现是调用到了log4jdbc的这个类，正常的应该是这个：  

![这是图片](/images/a-bug/1.png "1")  

接着调试，发现  

![这是图片](/images/a-bug/3.png "3")  

想要获取到正确的类，需要在驱动名称为com.mysql.jdbc.Driver。当时的驱动名称是com.mysql.cj.jdbc.Driver，所以用的是默认的类（未截图）。  

经排查，发现Mysql连接为  

		<!--Mysql依赖包-->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
查看jar包，发现版本是8.0.30。将版本降至5.1，  

		<!--Mysql依赖包-->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.49</version>
            <!--            <scope>runtime</scope>-->
        </dependency>
驱动名变成了com.mysql.jdbc.Driver，可以获取到正确的类。运行程序，生成sql时时间转换格式正确。

可以看出，log4jdbc不支持MySQL连接8.0以上的。  

问题解决。  
