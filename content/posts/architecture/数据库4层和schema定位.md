---
title: "数据库4层和schema定位"
date: "2025-08-23"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

mysql数据库库是3层的  
jdbc:mysql://localhost:3306/mydatabase?useUnicode=true&characterEncoding=UTF-8  
mydatabase 是schema 下面在是表 表下面的是列  
我们写sql是 select * from a a表是在当前的schema mydatabase下面  
当然也可以写成这种 select * from mydatabase2.b  mydatabase2是另外一个schema  

pgsql数据库是4层的  
jdbc:postgresql://localhost:5432/mydb  
mydb是db db下面有schema 下面在是表 表下面的是列  
我们写sql是 select * from schema.a 需要加上schema  
如果不加上schema  
PostgreSQL 使用 search_path 参数来确定查找对象的顺序  
SHOW search_path; -- 默认值： "$user", public  
"$user"：首先查找与当前用户名同名的 schema public：然后查找 public schema  
在所有用户 schema 查找完成后，会搜索系统 catalog  


oracle也有他自己的做法  
当执行 SELECT * FROM mytable 时，Oracle 按以下顺序查找：  
当前用户 schema 中的表  
当前用户 schema 中的私有同义词  
公有同义词  

创建私有同义词 CREATE SYNONYM employees FOR hr.employees;  
SELECT * FROM employees; 会访问hr这个schema下面的表  

公有同义词（如果存在）  
CREATE PUBLIC SYNONYM employees FOR hr.employees;  
SELECT * FROM employees;会访问hr这个schema下面的表  