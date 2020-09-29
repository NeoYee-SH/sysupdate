#### 背景
朋友公司的一台服务器，阿里云ecs，2核4G，centos7.5。 上面运行了一个企业站， 一套erp系统，企业站和erp是找外包团队开发的，环境都是 nginx + php， 对应mysql两个库，和一个redis库。

这台机器我之前也登入进去过，当时是朋友直接跟我说账号密码，目的明确的解决某项问题，之后就没咋关注了。

#### 发现问题

某天（25日晚）朋友跟我讲他的erp系统变得很卡，加载一个列表需要好几秒才出来，让我看看是不是数据变多了， 是不是要升级系统。

然后我就root直接登入了。
用 ps aux  命令，看到了占用CPU和内存较高的任务
```
[root@localhost ~]# ps -aux --sort -pcpu | head -n 5
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root     19614  101  0.1 2957748 5012 ?        Sl   Sep24 3337:04 ./sysupdate
root     23241 85.8  1.4 808840 56436 ?        Sl   Sep24 2788:44 /etc/networkservice
root     23246 11.7  1.4 855720 56260 ?        SNl  Sep24 382:42 ./networkservice 15
root      9002  0.4  0.9 164616 38248 ?        S<sl Sep03 140:13 /usr/local/aegis/aegis_client/aegis_10_85/AliYunDun
[root@localhost ~]# ps -aux --sort -pmem | head -n 5
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
mysql    21420  0.0  2.6 1516944 101324 ?      Sl   Aug04  30:31 /usr/libexec/mysqld --basedir=/usr --datadir=/var/lib/mysql --plugin-dir=/usr/lib64/mysql/plugin --user=mysql --log-error=/var/log/mariadb/mariadb.log --pid-file=/var/run/mariadb/mariadb.pid --socket=/var/lib/mysql/mysql.sock
root     23241 85.8  1.4 808840 56436 ?        Sl   Sep24 2788:54 /etc/networkservice
root     23246 11.7  1.4 855720 56260 ?        SNl  Sep24 382:44 ./networkservice 15
nobody   16686  0.0  1.0 1276400 39428 ?       S    Aug31   2:11 php-fpm: pool www

```

sysupdate 和 networkservice 不知道啥玩意，确实是最近起的， 就问了下朋友是不是最近给账号给别人用过， 加了啥服务， 得到回答没有。
<!--more-->
#### 分析问题
通过进程id找到命令
```
[root@localhost ~]# ll /proc/19614/cwd
lrwxrwxrwx 1 root root 0 Sep 25 16:02 /proc/19614/cwd -> /etc
[root@skyland ~]# ll /proc/19614/exe
lrwxrwxrwx 1 root root 0 Sep 25 14:55 /proc/19614/exe -> /etc/localhost
[root@skyland ~]# ll /etc/sysupdate 
-rwxrwxrwx 1 root root 1102480 Sep 24 16:34 /etc/sysupdate
```
同目录(/etc)发现异常文件。
1. last 命令查看了一下登录历史，发现已被清空。
2. history 记录也仅剩下我当时的操作记录。
3. 查看  /etc/passwd 无异常。
4. 查看 /tmp 发现有异常文件 。
5. 查看 ~/.ssh/authorized_keys，发现被更改，时间接近。
![image.png](https://i.loli.net/2020/09/27/mJXAk9jr814zUiQ.png)


于是安装 [Rootkit Hunter](https://www.yihuaiyuan.com/archives/30.html) ,做了个系统扫描， 发现多处异常。

然后通过一些线索跟进，找到了侵入者留下来的 update.sh和 config.json 等文件， 具体也没细看，大概发现是跟 门罗币 有关， 应该是个挖矿程序。

具体文件在 [github- sysupdate](https://github.com/yihuaiyuan/sysupdate) , 这里截几个图示例

![image _1_.png](https://i.loli.net/2020/09/27/6W5neI2q3kwSova.png)

![image _2_.png](https://i.loli.net/2020/09/27/HjdQmYJiPaVzCMR.png)

![image _3_.png](https://i.loli.net/2020/09/27/tMq6uY2vIDjRcoJ.png)

#### 反馈
跟朋友分析之后， 对方很震惊，表示可以花钱解决， 问是否需要买阿里的安全服务。

我表示我可以再花点时间处理，阿里的安全服务，等项目做大了再买不迟。

#### 处理

考虑到修复工作量太大，怕是搞不定。 何况这上面也没什么东西，重装系统重新部署或换台机器迁移过去肯定是比较easy的。 

因为朋友那台主机还有很久才到期， 直接放弃换新的就要浪费钱了，但是备份数据重装又担心玩砸了，所以折中了一下。 以下是处理流程简述：
1. 租一台阿里云同样配置的ecs新机器，一个月。
    
    在ecs实例详情 -> 付费信息 -> 更多里面有个快捷键
    ![image _4_.png](https://i.loli.net/2020/09/27/rAEnBLKIHqVCdRY.png)

2. 新机器修改 ssh 默认端口， 配置禁用密码登录，仅可通过密钥登录。
    
    [SSH免密登录远程Linux主机](https://www.yihuaiyuan.com/archives/27.html)
3. 在阿里云后台设置新机器IP白名单。
    ![image _5_.png](https://i.loli.net/2020/09/27/o9fAC43Llki7pJR.png)

4. 新机器上搭建 [openresty + php](https://github.com/yihuaiyuan/openresty_php) + redis + mysql，php改为sock通信。 

    修改 redis 默认端口，伪装危险命令，非root账户启动。
    [Redis 安全](https://redis.yihuaiyuan.com/senior/secure/)
    
5. 迁移 网站 和 erp 的代码到新机器。
    
    这里使用了[ scp](https://www.yihuaiyuan.com/archives/24.html)
6. 通知停服。之后停服。
7. 迁移 redis 库到新机器。 RMT 。
8. 迁移 两个数据库 到新机器。
    ```
    [root@new ~]# mysqldump -h原机器 -u用户名 -p密码 --routines --skip-lock-tables --single-transaction --default-character-set=utf8 --databases web > /tmp/web.dev         
    [root@new ~]# mysqldump -h原机器 -u用户名 -p密码 --routines --skip-lock-tables --single-transaction --default-character-set=utf8 --databases erp > /tmp/erp.dev     
    
    [root@new ~]# time mysql -u用户名 -p密码 --default-character-set=utf8 < /tmp/web.dev;
    [root@new ~]# time mysql -u用户名 -p密码 --default-character-set=utf8 < /tmp/erp.dev;
    ```
9. 新机器启动服务。
9. 本地配置host，指向新机器，访问确认一切正常。
    
    可先随意配置一个域名。
10. 原机器重装系统， 搭建服务，迁移回去代码和库文件。
11. 原机器启动服务，测试一切正常。
12. 通知恢复服务。

#### 最后
朋友很高兴，说有时间请我吃饭。

本故事纯属虚构。
