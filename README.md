## 一、专题背景

最近使用了个自动化平台（详见[自动化运维平台Spug测试](https://blog.51cto.com/3241766/2537675)）进行每周的变更，效果很不错，平台将大量重复繁琐的操作通过脚本分发方式标准化自动化了，平台核心是下发到各个服务器的shell脚本，感觉有必要对shell脚本做个总结，所以有了写本专题的想法。本专题将结合运维实际介绍shell脚本的各项用法，预计10篇左右，将包括系统巡检、监控、ftp上传下载、数据库查询、日志清理、时钟同步、定时任务等，里面会涉及shell常用语法、注意事项、调试排错等。

## 二、本文前言

本文是该专题的第二篇。

文章主要介绍最近在日常生产运维中使用到的一些shell语句，主要有替换、删除、查找指定行和指定字符、文件传输、列与列互换等。

## 三、shell用例

### 1.禁止root直接登录

**需求：**为保证安全，生产上禁止root账号直接登录

**修改前：**

```bash
[root@ansible /etc/ssh]# ll|grep sshd_config
-rw-------. 1 root root       3907 4月  11 2018 sshd_config
[root@ansible /etc/ssh]# more /etc/ssh/sshd_config|grep PermitRootLogin
#PermitRootLogin yes
# the setting of "PermitRootLogin without-password".
```

修改前目录/etc/ssh下只有一个sshd_config文件且配置PermitRootLogin为注释状态

**修改后：**

```bash
[root@ansible /etc/ssh]# sed -i.bak 's/#PermitRootLogin yes/PermitRootLogin no/g' /etc/ssh/sshd_config
[root@ansible /etc/ssh]# more /etc/ssh/sshd_config|grep PermitRootLogin                               
PermitRootLogin no
# the setting of "PermitRootLogin without-password".
[root@ansible /etc/ssh]# ll|grep sshd_config                                                          
-rw-------  1 root root       3905 1月  22 15:26 sshd_config
-rw-------. 1 root root       3907 4月  11 2018 sshd_config.bak
```

修改后目录/etc/ssh多了个备份文件且PermitRootLogin参数解注释且为no状态

![image-20210122155247208](https://i.loli.net/2021/01/26/4PUB3LqnKoSTIdj.png)

### 2.更新sudoer列表

**需求：**某些账号需要root权限

**修改前：**

```bash
[root@ansible /etc]# cd
[root@ansible ~]# cd /etc
[root@ansible /etc]# ll|grep sudoers              
-r--r-----.  1 root root     4328 10月 30 2018 sudoers
drwxr-x---.  2 root root        6 10月 31 2018 sudoers.d
[root@ansible /etc]# more sudoers|grep 'ALL=(ALL)'
root    ALL=(ALL)       ALL
%wheel  ALL=(ALL)       ALL
# %wheel        ALL=(ALL)       NOPASSWD: ALL
```

修改前/etc下只有sudoers文件且除了root无其他账号有root权限

**修改后：**

```bash
[root@ansible /etc]# sed  -i.bak "/root.*ALL=(ALL).*ALL/a\app    ALL=(ALL)       ALL"  /etc/sudoers
[root@ansible /etc]# ll|grep sudoers                                                               
-r--r-----   1 root root     4355 1月  22 16:05 sudoers
-r--r-----   1 root root     4328 1月  22 15:53 sudoers.bak
drwxr-x---.  2 root root        6 10月 31 2018 sudoers.d
[root@ansible /etc]# more sudoers|grep 'ALL=(ALL)'                                                 
root    ALL=(ALL)       ALL
app    ALL=(ALL)       ALL
%wheel  ALL=(ALL)       ALL
# %wheel        ALL=(ALL)       NOPASSWD: ALL
```

修改后/etc多了个备份文件sudoers.bak且root下的一行多了app账号的信息，使app拥有root权限

![image-20210122160830976](https://i.loli.net/2021/01/26/s4GhrltUj6gedBm.png)

### 3.scp复制多个目录或文件

**需求：**复制多个本地文件到远端或将远端多个文件复制到本地

**本地复制到远程目录：**

```bash
[root@ansible ~]# touch  a.txt b.txt c.txt
[root@ansible ~]# mkdir d
[root@ansible ~]# scp -v -r a.txt b.txt c.txt d root@172.16.7.157:/tmp
```

本地新建文件a.txt b.txt c.txt和目录d，复制到远程主机的/tmp目录

![image-20210122165106801](https://i.loli.net/2021/01/26/BZm87vWTzNLKq35.png)

![image-20210122165121845](https://i.loli.net/2021/01/26/jSRXlZ25nAHmLPz.png)

![image-20210122165134795](https://i.loli.net/2021/01/26/aFgb1vtI7NBTwiM.png)

**远程目录复制到本地：**

```bash
[root@157 ~]# touch 01.sh 02.sh 03.sh
[root@157 ~]# mkdir 04
```

```bash
[root@ansible ~]# scp -v -r root@172.16.7.157:/root/\{01.sh,02.sh,03.sh,04\} /tmp
```

在157上新建文件01.sh 02.sh 03.sh和目录04，复制到本地的/tmp目录

![image-20210122165621821](https://i.loli.net/2021/01/26/uSr1ZT7EfzVULOQ.png)

![image-20210126154706589](https://i.loli.net/2021/01/26/GtXAqV6pBiFImw1.png)

![image-20210122165720838](https://i.loli.net/2021/01/26/8szouWvqdGNF3PH.png)

### 4.删除已存在的定时任务

**需求：**删除指定的定时任务

```bash
[root@ansible /var/spool/cron]# crontab -l
0 0 * * * /usr/sbin/ntpdate -u ntpserver  >> /tmp/ntp.log
[root@ansible /var/spool/cron]# sed -i.bak '/\/usr\/sbin\/ntpdate/d' /var/spool/cron/root

```

注意，匹配/usr/sbin/ntpdate时"/"需要转义

![image-20210126104339368](https://i.loli.net/2021/01/26/P8Kp69cfFrqYsGH.png)

### 5.行内列与列互换

**需求：**将/etc/hosts文件中ip和主机名互换，ansible中主机列表格式为主机名+ip

**修改前：**

```bash
[root@ansible ~]# cd /etc
[root@ansible /etc]# cp hosts hosts-ansible.txt 
[root@ansible /etc]# more hosts-ansible.txt 
10.17.6.137          loong576-file01
10.17.6.129          loong576-kxxl01
10.17.6.130          loong576-kxxl02
10.17.6.131          loong576-capp01
10.17.6.132          loong576-capp02
10.17.6.128          loong576-pbgw01
10.17.6.127          loong576-pbgw02
10.17.6.134          loong576-xucs01
10.17.6.133          loong576-xucs02
10.17.6.136          loong576-webc01
10.17.6.135          loong576-webc02
10.17.6.14           loong576-gwxx-1
10.17.6.15           loong576-gwxx-2
10.17.6.18           loong576-mysql1
10.17.6.17           loong576-mysql2
```

**修改后：**

```bash
[root@ansible /etc]# awk '{print $2,$1 > "hosts-ansible.txt"}' hosts-ansible.txt 
[root@ansible /etc]# more hosts-ansible.txt 
loong576-file01 10.17.6.137
loong576-kxxl01 10.17.6.129
loong576-kxxl02 10.17.6.130
loong576-capp01 10.17.6.131
loong576-capp02 10.17.6.132
loong576-pbgw01 10.17.6.128
loong576-pbgw02 10.17.6.127
loong576-xucs01 10.17.6.134
loong576-xucs02 10.17.6.133
loong576-webc01 10.17.6.136
loong576-webc02 10.17.6.135
loong576-gwxx-1 10.17.6.14
loong576-gwxx-2 10.17.6.15
loong576-mysql1 10.17.6.18
loong576-mysql2 10.17.6.17
```

![image-20210126105901043](https://i.loli.net/2021/01/26/KXUZ4mLyx7AS9d6.png)

这个脚本可以很方便的实现ip和主机名的位置互换

### 6.指定行新增

**需求：**在ip地址10.17.6前同时新增参数‘ansible_ssh_host=’

```bash
[root@ansible /etc]# sed -i 's/10.17.6/ansible_ssh_host=&/' hosts-ansible.txt 
```

![image-20210126160213473](https://i.loli.net/2021/01/26/1MA3rONywpBiI8e.png)通过5和6可以很方便的将/etc/hosts的ip+主机名格式转换为主机名+ansible_ssh_host=+ip的格式，满足ansible对主机名的格式要求

通过5和6可以很方便的将/etc/hosts的ip+主机名格式转换为主机名+ansible_ssh_host=+ip的格式，满足ansible对主机名的格式要求

### 7.find、xargs、rm删除找到的文件

**需求：**使用find查找满足条件的文件并删除

```bash
[root@ansible /]# find ./ -name *[0-9]\*.bak
./home/a001.bak
./opt/04.txt.bak
./opt/03.bak
./tmp/09.bak
./tmp/10.txt.bak
./usr/05.bak
./usr/06.txt.bak
./var/07.bak
./var/08.txt.bak
./root/02.txt.bak
./root/01.bak
[root@ansible /]# find ./ -name *[0-9]\*.bak|xargs rm -rf
[root@ansible /]# find ./ -name *[0-9]\*.bak
```

查找以数字开头以.bak结尾的所有文件，然后删除

![image-20210126111216741](https://i.loli.net/2021/01/26/rKX1QAvEOhG52LD.png)

### 8.sed、find、grep删除/替换文件中的指定字符

**需求：**查找所有文件中包含'loong576'的字符串并替换或者删除

**修改前：**

```bash
[root@ansible-awx os-check]# find .|xargs grep -rl 'loong576'                                  
./defaults/main.yaml
./files/check_linux.sh
./tasks/main.yaml
./defaults/main.yaml
./defaults/main.yaml
./files/check_linux.sh
./files/check_linux.sh
./tasks/main.yaml
./tasks/main.yaml
[root@ansible-awx os-check]# find .|xargs grep -ri 'loong576'                                
./defaults/main.yaml:# Created by loong576 2020.05 
./files/check_linux.sh:# Created by loong576 2020.05 
./tasks/main.yaml:# Created by loong576 2020.05 
./defaults/main.yaml:# Created by loong576 2020.05 
./defaults/main.yaml:# Created by loong576 2020.05 
./files/check_linux.sh:# Created by loong576 2020.05 
./files/check_linux.sh:# Created by loong576 2020.05 
./tasks/main.yaml:# Created by loong576 2020.05 
./tasks/main.yaml:# Created by loong576 2020.05 
```

查找包含loong576字样的文件列表并指出该文件包含的具体字符

**修改后：**将‘Created by loong576 2020.05’的注释更改为‘Created by loong576 2021.01’

```bash
[root@ansible-awx os-check]# sed -i "s/2020.05/2021.01/g"  `find .|xargs grep -rl 'loong576'` 
[root@ansible-awx os-check]# find .|xargs grep -ri 'loong576'                                 
./defaults/main.yaml:# Created by loong576 2021.01 
./files/check_linux.sh:# Created by loong576 2021.01 
./tasks/main.yaml:# Created by loong576 2021.01 
./defaults/main.yaml:# Created by loong576 2021.01 
./defaults/main.yaml:# Created by loong576 2021.01 
./files/check_linux.sh:# Created by loong576 2021.01 
./files/check_linux.sh:# Created by loong576 2021.01 
./tasks/main.yaml:# Created by loong576 2021.01 
./tasks/main.yaml:# Created by loong576 2021.01 
```

**删除时间：**

```bash
[root@ansible-awx os-check]# sed -i "s/2021.01//g"  `find .|xargs grep -rl 'loong576'`               
[root@ansible-awx os-check]# find .|xargs grep -ri 'loong576'                          
./defaults/main.yaml:# Created by loong576  
./files/check_linux.sh:# Created by loong576  
./tasks/main.yaml:# Created by loong576  
./defaults/main.yaml:# Created by loong576  
./defaults/main.yaml:# Created by loong576  
./files/check_linux.sh:# Created by loong576  
./files/check_linux.sh:# Created by loong576  
./tasks/main.yaml:# Created by loong576  
./tasks/main.yaml:# Created by loong576
```

![image-20210126144600270](https://i.loli.net/2021/01/26/8CaWt4Fr9S1QG2D.png)

### 9.指定字符最前面、上一行添加字符，最后一行新增一行

**需求：**在配置ntp服务器时需要在配置文件/etc/ntp.conf指定字符上一行新增行、注释某些默认配置（指定字符前加#）、配置文件/etc/hosts最后新增行

**指定字符上一行新增行：**

```bash
[root@ansible ~]# sed -i '/driftfile/i server ntpserver iburst' /etc/ntp.conf 
```

在指定行driftfile上一行新增‘server ntpserver iburst’

**注释某些默认配置（指定字符前加#）：**

```bash
[root@ansible ~]# sed -i '/centos.pool.ntp.org/s/^/#/' /etc/ntp.conf
```

注释‘server [0..3].centos.pool.ntp.org iburst’

更改前：

![image-20210126150628314](https://i.loli.net/2021/01/26/w2P1nThedoUu3JO.png)

更改后：

![image-20210126150816735](https://i.loli.net/2021/01/26/fJYMW8KVD3ArN6d.png)

**在最后一行新增：**

```bash
[root@ansible ~]# sed -i '$a 172.16.7.157    ntpserver' /etc/hosts
```

![image-20210126151241163](https://i.loli.net/2021/01/26/FeKjJsG9i7CUfrY.png)

## 四、本文总结

本文主要介绍了常用的一些shell用例，涉及日常的查找、替换、文件传输等，使用到的命令主要有find、sed、xargs、scp等。shell脚本就是将各个命令按不能使用目的有逻辑的的组合在一起，掌握好了这些命令会对后面的脚本编写起到事半功倍的效果。

&nbsp;

&nbsp;

更多请关注：[shell专题](https://blog.51cto.com/3241766/category18.html)

