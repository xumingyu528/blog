---
title: Apache-httpd
category: Linux
tag: 
  - Linux
  - Apache
author: xmy
---




NCSA组织 httpd  
www.apache.org  
httpd.apache.org


## httpd特性
- 实现创建进程
- 按需维持适当的进程
- 模块化设计，核心比较小，各种功能都可以以模块方式添加  
  支持运行配置，支持单独编译模块
-   支持多种方式的虚拟主机配置
    Socket IP: Port

## httpd配置文件
- /usr/sbin/httpd(MPM：prefork)
  httpd：root,root (master process)
  httpd: apache,apache (worker process)
- /etc/rc.d/init.d/httpd  启动脚本
  Port：(80/tcp),(ssl：443/tcp)
- /etc/httpd：工作根目录，相当于程序安装目录
- /etc/httpd/conf：配置文件目录
  主配置文件：httpd.conf
  /etc/httpd/conf.d/*.conf
- /etc/httpd/modules：模块链接目录
- /etc/httpd/logs --> /var/log/httpd：日志目录
  日志文件有两类：访问日志access_log ，错误日志err_log
- /var/www/
  html
  cgi-bin
  cgi：Common Gateway Interface
        Client --> httpd(index.cgi) --> Spawn process(index.cgi) --> httpd --> client
        perl, python, java, (Servlet,JSP) php
  fastcgi


httpd-manual  #httpd 帮助文档
`#yum -y install httpd-manual`
安装完成后访问http://website/manual
/etc/httpd/conf.d/manual.conf  #httpd-manual配置文件

httpd配置文件分为三节  
`### Section 1: Global Environment`  全局配置  
`### Section 2: 'Main' server configuration`  主服务配置  
`### Section 3: Virtual Hosts`  虚拟主机配置  

`httpd -t`  检查httpd配置文件语法是否有误

---
### Section 1: Global Environment:
  由directive value（指令 值）组成

ServerTokens OS   #显示web相关信息 Major|Minor|Min|Prod|OS|Full

ServerRoot "/etc/httpd"  #httpd主目录，很多配置以此目录指定相对 路径

PidFile run/httpd.pid   #httpd pid配置文件

Timeout 60 #tcp/ip连接超时时间

Keepalive Off #长连接开关  
MaxKeepAliveRequests 100 #长连接最大连接数量  
KeepAliveTimeout 15 #长连接超时时间  

几种工作模型:可以使用httpd -l查看当前使用的模型  
MPM Multi Path Modules 多道处理模块  
    mpm_winnt  windowsNT使用的线程处理机制  
    prefork    预先生成工具    一个请求使用一个进程响应  
    worker     基于线程      一个请求使用一个线程响应  
    event      基于事件驱动模型   一个进程处理多个请求  

prefork模型配置相关选项
```
# prefork MPM
# StartServers: number of server processes to start
# MinSpareServers: minimum number of server processes which are kept spare
# MaxSpareServers: maximum number of server processes which are kept spare
# ServerLimit: maximum value for MaxClients for the lifetime of the server
# MaxClients: maximum number of server processes allowed to start
# MaxRequestsPerChild: maximum number of requests a server process serves
<IfModule prefork.c>
StartServers       8 #默认启动进程数
MinSpareServers    5 #最小空闲进程数
MaxSpareServers   20 #最大空闲进程数
ServerLimit      256 #MaxClients 的上限值
MaxClients       256 #最大客户端连接数
MaxRequestsPerChild  4000 #每个子进程可响应的数量
</IfModule>
```
worker模型配置相关选项
```
# worker MPM
# StartServers: initial number of server processes to start
# MaxClients: maximum number of simultaneous client connections
# MinSpareThreads: minimum number of worker threads which are kept spare
# MaxSpareThreads: maximum number of worker threads which are kept spare
# ThreadsPerChild: constant number of worker threads in each server process
# MaxRequestsPerChild: maximum number of requests a server process serves
<IfModule worker.c>
StartServers         4  #默认启动进程数
MaxClients         300  #最大客户端连接数
MinSpareThreads     25  #最小线程数
MaxSpareThreads     75  #最大线程数
ThreadsPerChild     25  #单个进程生成的线程数
MaxRequestsPerChild  0  #每个进程响应请求数
</IfModule>

```
```
listen 80 #监听端口  可以指定仅监听某IP地址，如listen 192.168.1.1:8080


LoadModule   #加载的模块

Include conf.d/*.conf    #包含的其他配置文件路径

work进程运行时使用的用户  
User apache  
Group apache
```

---
### Section 2: 'Main' server configuration
```
DocumentRoot "/var/www/html"  
>URL路径和和本地文件系统路径是两种概念
>URL路径是相对DocmentRoot而言  

<Directory "/var/www/html">  
    Options  选项 {None 不支持任何选项，默认为此选项| Indexes 允许索引目录| FollowSymLinks 允许访问符号链接指向的源文件| Includes 允许执行服务端包含（SSI）| ExecCGI 允许执行CGI脚本 | ALL 支持所有选项}  
    AllowOverride {None|AuthConfig|Limit}  
    Order allow,deny   用于定义基于主机的访问控制功能，先allow后deny，或者先deny后allow表示先后匹配的顺序   
    Allow from all   Allow from 192.168.0.0/24  
    Deny from all   Deny from 192.168.0.0/24  
</Directory>  

<IfModule mod_userdir.c>    #定义了用户目录，默认关闭
    UserDir disabled
</IfModule>

DirectoryIndex index.html index.htm   #默认文档  

AccessFileName .htaccess   #访问控制规则文件  

<Files ~ "\.ht">     定义以.ht开头的文件访问规则
    Order allow,deny
    Deny from all
</Files>

TypesConfig /etc/mime.types    #定义了非二进制文件类型  

HostnameLookup Off   #访问日志中是否记录主机名（默认记录IP）  

ErrorLog logs/error_log    #错误日志  

LogLevel warn      #日志级别  

LogFormat    #日志记录相关参数  

CustomLog 文件系统路径 #日志文件路径

Alias /别名/  "/真实路径/"  #别名路径  

<Location /status>  Handler 相关定义
    SetHandler server-status
    Order Deny,Allow
    Deny from all
    Allow from example.com
</Location>

```
---
### Section 3: Virtual Hosts
**中心主机**  
即Section 2 中定义的   
**虚拟主机**  
中心主机中的配置均可用
- 基于IP
```
<VirtualHost 10.0.0.2:80>
    DocumentRoot "/www/html"
</VirtualHost>

<VirtualHost 10.0.0.3:80>
    DocumentRoot "/www/html2"
</VirtualHost>
```
- 基于端口
```
<VirtualHost *:9090>
    DocumentRoot "/www/9090"
</VirtualHost>

<VirtualHost *:8080>
    DocumentRoot "/www/8080"
</VirtualHost>
```
- 基于域名，通过HTTP协议报文head中Host值来区分
```
NameVirtualHost *:80
<VirtualHost *:80>
    ServerName www.test.com
    DocumentRoot "/www/html/www"
</VirtualHost>

<VirtualHost *:80>
    ServerName b.test.com
    DocumentRoot "/www/html/b"
</VirtualHost>

```


举例：  
拒绝主机10.0.0.9访问 a.test.com;  
访问b.test.com需要认证;  
```
NameVirtualHost *:80
<VirtualHost *:80>
    ServerName a.test.com
    DocumentRoot "/web/a"
    CustomLog /web/log/a/access_log combined
    <Directory "/web/a">
        Options none
        AllowOverride none
        Order deny,allow
        Deny from 10.0.0.9
    </Directory>
</VirtualHost>

<VirtualHost *:80>
    ServerName b.test.com
    DocumentRoot "/web/b"
    CustomLog /web/log/b/access_log combined
    <Directory "/web/b">
      Options none
      AllowOverride authconfig
      AuthType basic
      AuthName "Restrict area."
      AuthUserFile "/etc/httpd/.htpasswd"
      Require valid-user
    </Directory>
</VirtualHost>
```
默认虚拟主机：可以解析到本地主机但不存在的地址会跳转至此虚拟主机定义的页面
```
<VirtualHost _default_:80>
    DocumentRoot "/web/default"
</VirtualHost>
```


---
### 基于openssl的https服务配置




curl  获取页面  
elinks  交互式模式显示网页   去除图片等格式  
    -dump  显示完成后退出，不进入交互式模式  
    -source 显示源码  

LoadRunner  HP公司测试工具  
PV：Page View  
UV：User View  