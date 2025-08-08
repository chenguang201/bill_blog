**<font style="color:#DF2A3F;">笔记来源：</font>**[**<font style="color:#DF2A3F;">黑马程序员python教程，8天python从入门到精通，学python看这套就够了</font>**](https://www.bilibili.com/video/BV1qW4y1a7fU/?spm_id_from=333.337.search-card.all.click&vd_source=e8046ccbdc793e09a75eb61fe8e84a30)



# 1 安装第三方库
在Python中，通过使用第三方库：pymysql，完成对MySQL数据库的操作。

安装指令：

```python
pip install pymysql
```

# 2 在Python中使用
## 2.1 基本使用
导包 -> 建立连接 -> 进行xx操作 -> 关闭连接

```python
# 1.导入操作包
from pymysql import Connection

# 2.获取到MySQL数据库的连接对象
conn = Connection(
    host='localhost',  # 主机名或IP地址
    port=3306,  # 端口号，默认3306
    user='root',  # MySQL账号
    password='root'  # MySQL密码
)

# 231. 打印MySQL版本信息
print(conn.get_server_info())

# 3.关闭到数据库的连接
conn.close()

```

## 2.2 执行建表SQL
导包 -> 建立连接 -> 获取游标对象 -> 选择数据库 -> 执行相应sql -> 关闭连接

```python
from pymysql import Connection

# 232. 获取到MySQL数据库的连接对象
conn = Connection(
    host='localhost',  # 主机名或IP地址
    port=3306,  # 端口号，默认3306
    user='root',  # MySQL账号
    password='root'  # MySQL密码
)

"""
执行非查询性质SQL
"""
# 233. 获取游标对象(用于操作数据库)
cursor = conn.cursor()
# 234. 选择要操作的数据库
conn.select_db("db1")
# 235. 使用游标对象，执行建表sql语句
cursor.execute("CREATE TABLE tb_user(id INT,name VARCHAR(8),age int)")

# 236. 关闭到数据库的连接
conn.close()

```

## 2.3 执行查询SQL
导包 -> 建立连接 -> 获取游标对象 -> 选择数据库 -> 执行相应sql -> 获取查询数据执行xx操作 -> 关闭连接

```python
from pymysql import Connection

# 237. 获取到MySQL数据库的连接对象
conn = Connection(
    host='localhost',  # 主机名或IP地址
    port=3306,  # 端口号，默认3306
    user='root',  # MySQL账号
    password='root'  # MySQL密码
)

"""
执行查询性质SQL
"""
# 238. 获取游标对象(用于操作数据库)
cursor = conn.cursor()
# 239. 选择要操作的数据库
conn.select_db("db1")
# 240. 使用游标对象，执行sql语句
cursor.execute("SELECT * FROM tb_user")
# 241. 获取查询结果，返回元组对象
results: tuple = cursor.fetchall()
for result in results:
    print(result)

# 242. 关闭到数据库的连接
conn.close()

```

## 2.4 执行插入SQL
导包 -> 建立连接 -> 获取游标对象 -> 选择数据库 -> 执行相应sql -> 提交行为 -> 关闭连接

```python
from pymysql import Connection

# 243. 获取到MySQL数据库的连接对象
conn = Connection(
    host='localhost',  # 主机名或IP地址
    port=3306,  # 端口号，默认3306
    user='root',  # MySQL账号
    password='root',  # MySQL密码
    autocommit=True  # 设置自动提交(commit)
)

"""
执行插入SQL
"""
# 244. 获取游标对象(用于操作数据库)
cursor = conn.cursor()
# 245. 选择要操作的数据库
conn.select_db("db1")
# 246. 使用游标对象，执行sql语句
cursor.execute("Insert into tb_user values(1,'hhy','250')")
# 247. 确认插入行为
# 248. 如果在获取连接对象时设置自动提交可以不用再写。
conn.commit()

# 249. 关闭到数据库的连接
conn.close()

```

## 2.5 执行修改SQL
导包 -> 建立连接 -> 获取游标对象 -> 选择数据库 -> 执行相应sql -> 提交行为 -> 关闭连接

```python
from pymysql import Connection

# 250. 获取到MySQL数据库的连接对象
conn = Connection(
    host='localhost',  # 主机名或IP地址
    port=3306,  # 端口号，默认3306
    user='root',  # MySQL账号
    password='root',  # MySQL密码
    autocommit=True  # 设置自动提交(commit)
)

"""
执行修改SQL
"""
# 251. 获取游标对象(用于操作数据库)
cursor = conn.cursor()
# 252. 选择要操作的数据库
conn.select_db("db1")
# 253. 使用游标对象，执行sql语句
cursor.execute("UPDATE tb_user set username='hhy' where username = 'fsp'")
# 254. 确认修改行为
# 255. 如果在获取连接对象时设置自动提交可以不用再写。
conn.commit()

# 256. 关闭到数据库的连接
conn.close()

```

## 2.6 执行删除SQL
导包 -> 建立连接 -> 获取游标对象 -> 选择数据库 -> 执行相应sql -> 提交行为 -> 关闭连接

```python
from pymysql import Connection

# 257. 获取到MySQL数据库的连接对象
conn = Connection(
    host='localhost',  # 主机名或IP地址
    port=3306,  # 端口号，默认3306
    user='root',  # MySQL账号
    password='root',  # MySQL密码
    autocommit=True  # 设置自动提交(commit)
)


"""
执行删除SQL
"""
# 258. 获取游标对象(用于操作数据库)
cursor = conn.cursor()
# 259. 选择要操作的数据库
conn.select_db("db1")
# 260. 使用游标对象，执行sql语句
cursor.execute("DELETE from tb_user WHERE username = 'hhy'")
# 261. 确认删除行为
# 262. 如果在获取连接对象时设置自动提交可以不用再写。
conn.commit()

# 263. 关闭到数据库的连接
conn.close()

```



小结

+ pymysql在执行数据插入或其它产生数据更改的SQL语句时，默认是需要提交更改的，即，需要通过代码“确认”这种更改行为。
+ 如果不想手动commit确认，可以在构建连接对象的时候，设置自动commit的属性。
+ 查询后，使用游标对象.fetchall()可得到全部的查询结果封装入嵌套元组内
+ 可使用游标对象.execute()执行SQL语句



