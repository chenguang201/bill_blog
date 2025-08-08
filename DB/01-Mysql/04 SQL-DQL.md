**<font style="color:#DF2A3F;">笔记来源：</font>**[**<font style="color:#DF2A3F;">黑马程序员 MySQL数据库入门到精通，从mysql安装到mysql高级、mysql优化全囊括</font>**](https://www.bilibili.com/video/BV1Kr4y1i7ru/?spm_id_from=333.337.search-card.all.click&vd_source=e8046ccbdc793e09a75eb61fe8e84a30)



# 4. 前言
DQL英文全称是Data Query Language(数据查询语言)，数据查询语言，用来查询数据库中表的记录。  
查询关键字: **<font style="color:#E8323C;">SELECT</font>**  
在一个正常的业务系统中，查询操作的频次是要远高于增删改的，当我们去访问企业官网、电商网站，在这些网站中我们所看到的数据，实际都是需要从数据库中查询并展示的。而且在查询的过程中，可能还会涉及到条件、排序、分页等操作。  
那么，接下来我们主要学习的就是如何进行数据的查询操作。 我们先来完成如下数据准备工作:

```sql
drop table if exists employee;

create table emp( 
      id int comment '编号', 
      workno varchar(10) comment '工号', 
      name varchar(10) comment '姓名', 
      gender char(1) comment '性别', 
      age tinyint unsigned comment '年龄', 
      idcard char(18) comment '身份证号', 
      workaddress varchar(50) comment '工作地址', 
      entrydate date comment '入职时间' 
)comment '员工表';

INSERT INTO emp (id, workno, name, gender, age, idcard, workaddress, entrydate) VALUES (1, '00001', '柳岩666', '女', 20, '123456789012345678', '北京', '2000-01- 01');
INSERT INTO emp (id, workno, name, gender, age, idcard, workaddress, entrydate) VALUES (2, '00002', '张无忌', '男', 18, '123456789012345670', '北京', '2005-09- 01'); 
INSERT INTO emp (id, workno, name, gender, age, idcard, workaddress, entrydate) VALUES (3, '00003', '韦一笑', '男', 38, '123456789712345670', '上海', '2005-08- 01'); 
INSERT INTO emp (id, workno, name, gender, age, idcard, workaddress, entrydate) VALUES (4, '00004', '赵敏', '女', 18, '123456757123845670', '北京', '2009-12-01'); 
INSERT INTO emp (id, workno, name, gender, age, idcard, workaddress, entrydate) VALUES (5, '00005', '小昭', '女', 16, '123456769012345678', '上海', '2007-07-01'); 
INSERT INTO emp (id, workno, name, gender, age, idcard, workaddress, entrydate) VALUES (6, '00006', '杨逍', '男', 28, '12345678931234567X', '北京', '2006-01-01'); 
INSERT INTO emp (id, workno, name, gender, age, idcard, workaddress, entrydate) VALUES (7, '00007', '范瑶', '男', 40, '123456789212345670', '北京', '2005-05-01');
INSERT INTO emp (id, workno, name, gender, age, idcard, workaddress, entrydate) VALUES (8, '00008', '黛绮丝', '女', 38, '123456157123645670', '天津', '2015-05- 01'); 
INSERT INTO emp (id, workno, name, gender, age, idcard, workaddress, entrydate) VALUES (9, '00009', '范凉凉', '女', 45, '123156789012345678', '北京', '2010-04- 01'); 
INSERT INTO emp (id, workno, name, gender, age, idcard, workaddress, entrydate) VALUES (10, '00010', '陈友谅', '男', 53, '123456789012345670', '上海', '2011-01- 01'); 
INSERT INTO emp (id, workno, name, gender, age, idcard, workaddress, entrydate) VALUES (11, '00011', '张士诚', '男', 55, '123567897123465670', '江苏', '2015-05- 01'); 
INSERT INTO emp (id, workno, name, gender, age, idcard, workaddress, entrydate) VALUES (12, '00012', '常遇春', '男', 32, '123446757152345670', '北京', '2004-02- 01'); 
INSERT INTO emp (id, workno, name, gender, age, idcard, workaddress, entrydate) VALUES (13, '00013', '张三丰', '男', 88, '123656789012345678', '江苏', '2020-11- 01'); 
INSERT INTO emp (id, workno, name, gender, age, idcard, workaddress, entrydate) VALUES (14, '00014', '灭绝', '女', 65, '123456719012345670', '西安', '2019-05- 01');
INSERT INTO emp (id, workno, name, gender, age, idcard, workaddress, entrydate) VALUES (15, '00015', '胡青牛', '男', 70, '12345674971234567X', '西安', '2018-04- 01'); 
INSERT INTO emp (id, workno, name, gender, age, idcard, workaddress, entrydate) VALUES (16, '00016', '周芷若', '女', 18, null, '北京', '2012-06-01');
```



# 1 基本语法
DQL 查询语句，语法结构如下：

```sql
SELECT
      字段列表 
FROM
      表名列表 
WHERE
      条件列表 
GROUP BY 
      分组字段列表 
HAVING
      分组后条件列表 
ORDER BY 
      排序字段列表 
LIMIT
      分页参数
```

学习这部分内容的时候，会将上面的完整语法进行拆分，分为以下几个部分：

+ 基本查询（不带任何条件）
+ 条件查询（WHERE）
+ 聚合函数（count、max、min、avg、sum）
+ 分组查询（group by）
+ 排序查询（order by）
+ 分页查询（limit）

# 2 基础查询
在基本查询的DQL语句中，不带任何的查询条件，查询的语法如下：  
**<font style="color:#DF2A3F;">查询多个字段</font>**

```sql
SELECT 字段1, 字段2, 字段3 ... FROM 表名 ;
```

或

```sql
SELECT * FROM 表名 ;
```

> 注意 : * 号代表查询所有字段，在实际开发中尽量少用（不直观、影响效率）。
>



**<font style="color:#DF2A3F;">字段设置别名</font>**

```sql
SELECT 字段1 [ AS 别名1 ] , 字段2 [ AS 别名2 ] ... FROM 表名;
```

或

```sql
SELECT 字段1 [ 别名1 ] , 字段2 [ 别名2 ] ... FROM 表名;
```

> as可以省略
>



**<font style="color:#DF2A3F;">去除重复记录</font>**

案例：

```sql
--A. 查询指定字段 name, workno, age并返回
select name,workno,age from emp;
--B. 查询返回所有字段
select id ,workno,name,gender,age,idcard,workaddress,entrydate from emp;
select * from emp;
--C. 查询所有员工的工作地址,起别名
select workaddress as '工作地址' from emp;
-- as可以省略
select workaddress '工作地址' from emp;
--D. 查询公司员工的上班地址有哪些(不要重复)
select distinct workaddress '工作地址' from emp;
```

# 3 条件查询
**语法**

```sql
SELECT 字段列表 FROM 表名 WHERE 条件列表 ;
```

**条件**  
常用的**<font style="color:#DF2A3F;">比较运算符</font>**如下:

| 比较运算符 | 功能 |
| --- | --- |
| > | 大于 |
| >= | 大于等于 |
| < | 小于 |
| <= | 小于等于 |
| = | 等于 |
| <> 或 != | 不等于 |
| BETWEEN ... AND ... | 在某个范围之内(含最小、最大值) |
| IN(...) | 在in之后的列表中的值，多选一 |
| LIKE 占位符 | 模糊匹配(_匹配单个字符, %匹配任意个字符) |
| IS NULL | 是NULL |




常用的**<font style="color:#DF2A3F;">逻辑运算符</font>**如下:

| 逻辑运算符 | 功能 |
| --- | --- |
| AND 或 && | 并且 (多个条件同时成立) |
| OR 或 | | | 或者 (多个条件任意一个成立) |
| NOT 或 ! | 非 , 不是 |




案例:

```sql
--A. 查询年龄等于 88 的员工
select * from emp where age = 88;

--B. 查询年龄小于 20 的员工信息
select * from emp where age < 20;

--C. 查询年龄小于等于 20 的员工信息
select * from emp where age <= 20;

--D. 查询没有身份证号的员工信息
select * from emp where idcard is null;

--E. 查询有身份证号的员工信息
select * from emp where idcard is not null;

--F. 查询年龄不等于 88 的员工信息
select * from emp where age != 88; 
select * from emp where age <> 88;

--G. 查询年龄在15岁(包含) 到 20岁(包含)之间的员工信息
select * from emp where age >= 15 && age <= 20; 
select * from emp where age >= 15 and age <= 20; 
select * from emp where age between 15 and 20;

--H. 查询性别为 女 且年龄小于 25岁的员工信息
select * from emp where gender = '女' and age < 25;

--I. 查询年龄等于18 或 20 或 40 的员工信息
select * from emp where age = 18 or age = 20 or age =40; 
select * from emp where age in(18,20,40);

--J. 查询姓名为两个字的员工信息 _ %
select * from emp where name like '__';

--K. 查询身份证号最后一位是X的员工信息
select * from emp where idcard like '%X'; 
select * from emp where idcard like '_________________X';
```

# 4 聚合函数
将一列数据作为一个整体，进行纵向计算。

| 函数 | 功能 |
| --- | --- |
| count | 统计数量 |
| max | 最大值 |
| min | 最小值 |
| avg | 平均值 |
| sum | 求和 |


语法：

```sql
SELECT 聚合函数(字段列表) FROM 表名 ;
```

> 注意 : NULL值是不参与所有聚合函数运算的。
>



案例：

```sql
--A. 统计该企业员工数量
select count(*) from emp; --统计的是总记录数 
select count(idcard) from emp; --统计的是idcard字段不为null的记录数
--对于count聚合函数，统计符合条件的总记录数，还可以通过 count(数字/字符串)的形式进行统计查询，比如：
select count(1) from emp;
--对于count(*) 、count(字段)、 count(1) 的具体原理，我们在进阶篇中SQL优化部分会详细讲解，此处只需要知道如何使用即可。


--B. 统计该企业员工的平均年龄
select avg(age) from emp;

--C. 统计该企业员工的最大年龄
select max(age) from emp;

--D. 统计该企业员工的最小年龄
select min(age) from emp;

--E. 统计西安地区员工的年龄之和
select sum(age) from emp where workaddress = '西安';
```



# 5 分组查询
语法：

```sql
SELECT 字段列表 FROM 表名 [ WHERE 条件 ] GROUP BY 分组字段名 [ HAVING 分组 后过滤条件 ];
```

`where`与`having`区别:

+ 执行时机不同：where是分组之前进行过滤，不满足where条件，不参与分组；而having是分组之后对结果进行过滤。
+ 判断条件不同：where不能对聚合函数进行判断，而having可以。

注意事项:

> 分组之后，查询的字段一般为聚合函数和分组字段，查询其他字段无任何意义。  
执行顺序: where > 聚合函数 > having 。  
支持多字段分组, 具体语法为 : group by columnA,columnB
>



案例

```sql
--A. 根据年龄对公司的员工进行升序排序
select gender, count(*) from emp group by gender ;

--B. 根据入职时间, 对员工进行降序排序
select gender, avg(age) from emp group by gender ;

--C. 查询年龄小于45的员工 , 并根据工作地址分组 , 获取员工数量大于等于3的工作地址
select workaddress, count(*) address_count from emp where age < 45 group by workaddress having address_count >= 3;

--D. 统计各个工作地址上班的男性及女性员工的数量
select workaddress, gender, count(*) '数量' from emp group by gender , workaddress ;
```



# 6 排序查询
语法:

```sql
SELECT 字段列表 FROM 表名 ORDER BY 字段1 排序方式1 , 字段2 排序方式2 ;
```

排序方式

+ ASC : 升序(默认值)
+ DESC: 降序



注意事项：

> 如果是升序, 可以不指定排序方式ASC ;  
如果是多字段排序，当第一个字段值相同时，才会根据第二个字段进行排序 ;
>



案例:

```sql
--A. 根据年龄对公司的员工进行升序排序
select * from emp order by age asc; select * from emp order by age;

--B. 根据入职时间, 对员工进行降序排序
select * from emp order by entrydate desc;

--C. 根据年龄对公司的员工进行升序排序 , 年龄相同 , 再按照入职时间进行降序排序
select * from emp order by age asc , entrydate desc;
```

# 7 分页查询
语法

```sql
SELECT 字段列表 FROM 表名 LIMIT 起始索引, 查询记录数;
```

注意事项:

> 起始索引从0开始，起始索引 = （查询页码 - 1）* 每页显示记录数。  
分页查询是数据库的方言，不同的数据库有不同的实现，MySQL中是LIMIT。  
如果查询的是第一页数据，起始索引可以省略，直接简写为 limit 10。
>



案例:

```sql
--A. 查询第1页员工数据, 每页展示10条记录
select * from emp limit 0,10; 
select * from emp limit 10;

--B. 查询第2页员工数据, 每页展示10条记录 --------> (页码-1)*页展示记录数
select * from emp limit 10,10;
```



综合案例

```sql
--1). 查询年龄为20,21,22,23岁的女员工信息。
select * from emp where gender = '女' and age in(20,21,22,23);

--2). 查询性别为 男 ，并且年龄在 20-40 岁(含)以内的姓名为三个字的员工。
select * from emp where gender = '男' and ( age between 20 and 40 ) and name like '___';

--3). 统计员工表中, 年龄小于60岁的 , 男性员工和女性员工的人数。
select gender, count(*) from emp where age < 60 group by gender;

--4). 查询所有年龄小于等于35岁员工的姓名和年龄，并对查询结果按年龄升序排序，如果年龄相同按入职时间降序排序。
select name , age from emp where age <= 35 order by age asc , entrydate desc;

--5). 查询性别为男，且年龄在20-40 岁(含)以内的前5个员工信息，对查询的结果按年龄升序排序，年龄相同按入职时间升序排序。
select * from emp where gender = '男' and age between 20 and 40 order by age asc , entrydate asc limit 5 ;
```



# 8 执行顺序
在讲解DQL语句的具体语法之前，我们已经讲解了DQL语句的完整语法，及编写顺序，接下来，我们要来说明的是DQL语句在执行时的执行顺序，也就是先执行那一部分，后执行那一部分。  
![](images/18.png)  
验证：查询年龄大于15的员工姓名、年龄，并根据年龄进行升序排序。

```plsql
select name , age from emp where age > 15 order by age asc;
```



在查询时，我们给emp表起一个别名 e，然后在select 及 where中使用该别名。

```plsql
select e.name , e.age from emp e where e.age > 15 order by age asc;
```

执行上述SQL语句后，我们看到依然可以正常的查询到结果，此时就说明： from 先执行, 然后where 和 select 执行。那 where 和 select 到底哪个先执行呢?  
此时，此时我们可以给select后面的字段起别名，然后在 where 中使用这个别名，然后看看是否可以执行成功。

```plsql
select e.name ename , e.age eage from emp e where eage > 15 order by age asc;
```

执行上述SQL报错了:  
![](images/19.png)  
由此我们可以得出结论: from 先执行，然后执行 where ， 再执行select 。  
接下来，我们再执行如下SQL语句，查看执行效果：

```plsql
select e.name ename , e.age eage from emp e where e.age > 15 order by eage asc;
```

结果执行成功。 那么也就验证了: order by 是在select 语句之后执行的。  
综上所述，我们可以看到DQL语句的执行顺序为： `from ... where ... group by ...having ... select ... order by ... limit ...`

