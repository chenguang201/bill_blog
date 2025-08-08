**<font style="color:#DF2A3F;">笔记来源：</font>**[**<font style="color:#DF2A3F;">黑马程序员 MySQL数据库入门到精通，从mysql安装到mysql高级、mysql优化全囊括</font>**](https://www.bilibili.com/video/BV1Kr4y1i7ru/?spm_id_from=333.337.search-card.all.click&vd_source=e8046ccbdc793e09a75eb61fe8e84a30)



DML英文全称是Data Manipulation Language(数据操作语言)，用来对数据库中表的数据记录进行增、删、改操作。

+ 添加数据（INSERT）
+ 修改数据（UPDATE）
+ 删除数据（DELETE）



## 1 添加数据
**① 给指定字段添加数据**

```sql
INSERT INTO 表名 (字段名1, 字段名2, ...) VALUES (值1, 值2, ...);
```

案例:

```sql
-- 给employee表所有的字段添加数据 ；
insert into employee(id,workno,name,gender,age,idcard,entrydate) 
values(1,'1','Itcast','男',10,'123456789012345678','2000-01-01');
```



插入数据完成之后，可以直接一条查询数据的SQL语句, 语句如下:

```plsql
select * from employee;
```



案例:

```sql
-- 给employee表所有的字段添加数据,执行如下SQL，添加的年龄字段值为-1。
insert into employee(id,workno,name,gender,age,idcard,entrydate) 
values(1,'1','Itcast','男',-1,'123456789012345678','2000-01-01');
```

执行上述的SQL语句时，报错了，具体的错误信息如下：  
![](images/17.png)  
因为 employee 表的age字段类型为 tinyint，而且还是无符号的 unsigned ，所以取值只能在0-255 之间。

  
 **② 给全部字段添加数据**

```sql
INSERT INTO 表名 VALUES (值1, 值2, ...);
```

案例：

```sql
-- 插入数据到employee表，具体的SQL如下：
insert into employee values(2,'2','张无忌','男',18,'123456789012345670','2005-01- 01');
```



**③ 批量添加数据**  
部分字段：

```sql
INSERT INTO 表名 (字段名1, 字段名2, ...) VALUES (值1, 值2, ...), (值1, 值2, ...), (值 1, 值2, ...) ;
```

  
全部字段：

```sql
INSERT INTO 表名 VALUES (值1, 值2, ...), (值1, 值2, ...), (值1, 值2, ...) ;
```

  
案例：

```sql
-- 批量插入数据到employee表，具体的SQL如下：
insert into employee values
(3,'3','韦一笑','男',38,'123456789012345670','2005-01- 01'),
(4,'4','赵敏','女',18,'123456789012345670','2005-01-01');
```

注意事项:

+ 插入数据时，指定的字段顺序需要与值的顺序是一一对应的。
+ 字符串和日期型数据应该包含在引号中。
+ 插入的数据大小，应该在字段的规定范围内。



## 2 修改数据
修改数据的具体语法为:

```sql
UPDATE 表名 SET 字段名1 = 值1 , 字段名2 = 值2 , .... [ WHERE 条件 ] ;
```

案例:

```sql
-- A. 修改id为1的数据，将name修改为itheima
update employee set name = 'itheima' where id = 1;
-- B. 修改id为1的数据, 将name修改为小昭, gender修改为 女
update employee set name = '小昭' , gender = '女' where id = 1;
-- C. 将所有的员工入职日期修改为 2008-01-01
update employee set entrydate = '2008-01-01';
```

> 注意事项:修改语句的条件可以有，也可以没有，如果没有条件，则会修改整张表的所有数据。
>



## 3 删除数据
删除数据的具体语法为：

```sql
DELETE FROM 表名 [ WHERE 条件 ] ;
```

案例:

```sql
-- A. 删除gender为女的员工
delete from employee where gender = '女';
-- B. 删除所有员工
delete from employee;
```

注意事项

+ DELETE 语句的条件可以有，也可以没有，如果没有条件，则会删除整张表的所有数据。
+ DELETE 语句不能删除某一个字段的值(可以使用UPDATE，将该字段值置为NULL即 可)。
+ 当进行删除全部数据操作时，datagrip会提示我们，询问是否确认删除，我们直接点击Execute即可。



