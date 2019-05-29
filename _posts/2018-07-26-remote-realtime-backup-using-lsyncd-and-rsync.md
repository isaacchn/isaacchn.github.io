---
layout: article
title:  "使用lsyncd+rsync实现文件实时同步/增量备份"
tags: ops
modify_date: 2018-07-26 10:00:00
---
> 一种实现远程增量备份的方案。

## 0x00 前言
项目中需要对文件进行实时备份。看过诸多文章，踩过一系列坑之后，终于实现了预想中的需求。现将整个安装、配置、测试的流程汇总如下。
rsync简介，来自维基百科：
```
rsync是Unix下的一款应用软件，它能同步更新两处计算机的文件与目录，并适当利用差分编码以减少数据传输量。rsync中的一项同类软件不常见的重要特性是每个目标的镜像只需发送一次。rsync可以拷贝／显示目录内容，以及拷贝文件，并可选压缩以及递归拷贝。
```
lsyncd简介，来自项目主页：
```
Lsyncd watches a local directory trees event monitor interface (inotify or fsevents). It aggregates and combines events for a few seconds and then spawns one (or more) process(es) to synchronize the changes. By default this is rsync. Lsyncd is thus a light-weight live mirror solution that is comparatively easy to install not requiring new filesystems or block devices and does not hamper local filesystem performance.

Lsyncd监听本地文件系统事件监控接口(inotify或fsevents)。它将若干秒内的事件聚合，然后触发一个或多个进程以同步变更。默认的同步手段是rsync。Lysncd是一个轻量级、易安装的同步方案，它不需要新的文件系统或块设备，也不会影响本地文件系统性能。
```
## 0x01 需求
+ 实现对大量小文件的备份。文件数量为10万级，文件大小在4KB-1MB之间。这些文件被一次写入，多次读取，不会出现编辑的情况。
+ 要求实现增量备份。即原始文件夹将文件删除后，备份文件夹中不能将文件删除。
+ 原始文件夹文件增长速度不快，平均每天写入文件数小于1000，对网络带宽和磁盘IO都没有很高的性能需求。
+ 要求备份做到准实时级别，源始文件夹文件写入后，必须在1分钟内同步到备份文件夹。

## 0x02 方案选择
通过网上搜索，选用rsync作为备份过程的核心应用。为了做到实时备份，了解到了以下三个同步方案：
1. inotify+rsync
2. sersync+rsync
3. lsyncd+rsync

进行了对比：
1. inotify+rsync方案，有网友表示同步单个文件性能非常差。而且由于inotify的bug，必须让rsync同步整个目录而不是某个文件，否则可能出现文件遗漏。如果想让该方案保持性能且不遗漏文件，需要对整个体系进行认真的设计，时间成本会比较高。
2. sersync+rsync方案，开源项目，最近一次代码更新时间是2015年8月份，而且缺少社区支持。
3. lsyncd+rsync方案，同样是开源项目，最近一次代码更新时间是2018年5月。在网上有大量资料可供查询，在stackoverflow上也有较多关于这个方案的讨论。

最终选择lsyncd+rsync作为实时同步的方案。
## 0x03 安装及配置
刚开始部署的时候，没有明确“客户端”和“服务端”的概念，导致一开始的操作完全是错误的。在本文中，不使用这两个容易混淆的概念，取而代之的是“源端”和“目标端”。“源端”指的是需要被备份的原始文件所在服务器，“目标端”指的是备份后的文件所在服务器。
### 0 基础环境
源端：192.168.2.230 CentOS 7.4
目标端：192.168.2.249 CentOS 7.4

### 1 目标端安装rsync
rsync有多种运行方式，对于远程交互有两种：
1. 本地主机使用ssh和远程主机通信。
2. 本地主机使用socket连接远程主机上的rsync daemon。
“远程主机”指的就是“目标端”，我选择了第二种方式，源端将文件push到目标端。
目标端在CentOS下直接使用yum安装rsync。
`yum -y install rsync`

### 2 目标端配置rsync
#### 2.1 先创建一些需要的目录和文件。
```
mkdir /etc/rsync
mkdir /var/rsync
touch /etc/rsync/rsync.conf
touch /etc/rsync/rsync.passwd
chmod 600 /etc/rsync/rsync.passwd
touch /var/rsync/rsync.log
touch /var/rsync/rsync.motd
touch /opt/rsync/rsync.sh
```
**一定要注意密钥文件的权限，配置为600即可，否则会报错。**
#### 2.2 编辑/etc/rsync/rsync.conf文件

```properties
######### 全局配置参数 ##########
#rsync端口 默认873
port=873
#rsync服务的运行用户，默认是nobody，文件传输成功后属主将是这个uid
uid = 0
# rsync服务的运行组，默认是nobody，文件传输成功后属组将是这个gid
gid = 0
#rsync daemon在传输前是否切换到指定的path目录下，并将其监禁在内
use chroot = no
#最大连接数量，0表示没有限制
max connections = 200
#确保rsync服务器不会永远等待一个崩溃的客户端，0表示永远等待
timeout = 300
#客户端连接过来显示的消息
motd file = /var/rsync/rsync.motd
#rsync daemon的pid文件
pid file = /var/run/rsync.pid
#指定锁文件 
lock file = /var/run/rsync.lock
#指定rsync的日志文件，而不把日志发送给syslog
log file = /var/rsync/rsync.log
#指定哪些文件不用进行压缩传输
dont compress = *.gz *.tgz *.zip *.z *.Z *.rpm *.deb *.bz2
###########下面指定模块，并设定模块配置参数，可以创建多个模块###########
[rsync_test0] # 第一个模块的ID
#文件路径
path = /temp/rsync_test0
#忽略某些IO错误信息
ignore errors
#只读
read only = false
#不隐藏该模板
list = true
#允许IP
hosts allow = 192.168.0.0/16
#不允许IP
hosts deny = 0.0.0.0/32
#指定连接到该模块的用户列表，只有列表里的用户才能连接到模块，用户名和对应密码保存在secrts file中
#这里使用的不是系统用户，而是虚拟用户。不设置时，默认所有用户都能连接，但使用的是匿名连接
auth users = rsync
#保存auth users用户列表的用户名和密码，每行包含一个username:passwd。
#由于"strict modes"默认为true，所以此文件要求非rsync daemon用户不可读写。只有启用了auth users该选项才有效。
secrets file = /etc/rsync/rsync.passwd

[rsync_test1] # 第二个模块的ID
#文件路径
path = /temp/rsync_test1
#忽略某些IO错误信息
ignore errors
#只读
read only = false
#不隐藏该模板
list = true
#允许IP
hosts allow = 192.168.0.0/16
#不允许IP
hosts deny = 0.0.0.0/32
```
#### 2.3 将rsync用户认证信息写入到密码文件中
`echo 'rsync:123456' >> rsync.passwd`
其中`rsync`是用户名，需要和`rsync.conf`文件中的`auth users`保持一致，冒号后面的是该用户的密码。**注意：该用户为虚拟用户，与Linux系统中的用户无关。**
#### 2.4 编辑rsync.sh作为启动脚本
```sh
#!/bin/bash
#rsync启动脚本，注意这里的pid文件路径，conf文件路径一定要和自己的配置保持一致。
status1=$(ps -ef | egrep "rsync --daemon.*rsync.conf" | grep -v 'grep')
pidfile="/var/run/rsync.pid"
start_rsync="rsync --daemon --config=/etc/rsync/rsync.conf"

function rsyncstart() {
	if [ "${status1}X" == "X" ];then
		rm -f $pidfile
		$start_rsync
		status2=$(ps -ef | egrep "rsync --daemon.*rsync.conf" | grep -v 'grep')
		if [  "${status2}X" != "X"  ];then
			echo "rsync service start.......OK"
		fi
	else
		echo "rsync service is running !"
	fi
}
function rsyncstop() {
	if [ "${status1}X" != "X" ];then
		kill -9 $(cat $pidfile)
		status2=$(ps -ef | egrep "rsync --daemon.*rsync.conf" | grep -v 'grep')
		if [ "${statusw2}X" == "X" ];then
			echo "rsync service stop.......OK"
		fi
	else
		echo "rsync service is not running !"
	fi
}
function rsyncstatus() {
	if [ "${status1}X" != "X" ];then
		echo "rsync service is running !"
	else
		echo "rsync service is not running !"
	fi
}
function rsyncrestart() {
	if [ "${status1}X" == "X" ];then
		echo "rsync service is not running..."
		rsyncstart
	else
		rsyncstop
		rsyncstart
	fi
}
case $1 in
"start")
rsyncstart
;;
"stop")
rsyncstop
;;
"status")
rsyncstatus
;;
"restart")
rsyncrestart
;;
*)
echo
echo  "Usage: $0 start|stop|restart|status"
echo
esac
```
#### 2.5 运行rsync
```
chmod u+x /opt/rsync/rsync.sh
/opt/rsync/rsync.sh start
```

### 3 源端安装lsyncd+rsync
#### 3.1 源端安装rsync
`yum -y install rsync`

源端不需要对rsync进行配置。
使用源端rsync测试目标端rsync daemon运行情况。
```
mkdir /etc/lsyncd
touch /etc/lsyncd/rsync.passwd
echo '123456' > rsync.passwd
chmod 600 /etc/lsyncd/rsync.passwd
mkdir -p /temp/test0
touch /temp/test0/testfile
rsync -r /temp/test0/ rsync://rsync@192.168.2.230:873/rsync_test0/ --password-file=/etc/lsyncd/rsync.passwd
```
**一定要注意密钥文件的权限，配置为600即可，否则会报错。**
正常情况下，你目标端的rsync_test0对应目录下会出现
上面命令中的`rsync://rsync@192.168.2.230:873/rsync_test0/`，第一个rsync指的是这个地址是一个rsync URL，第二个rsync指的是配置在目标端的用户,rsync_test0是配置在目标端的module，password-file存储认证所需要的密码。**需要注意的是(1)这个密码文件中只填写密码即可，不要写用户名，否则会造成认证错误(2)源端目录后加与不加反斜杠，概念是不一样的。**
#### 3.2 源端安装lsyncd
先安装epel源，再使用yum安装lsyncd。
```
rpm -iUvh http://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-5.noarch.rpm
yum -y install lsyncd
```

### 4 源端配置lsyncd
#### 4.1 先创建一些需要的目录和文件
```
mkdir /etc/lsyncd
mkdir /var/lsyncd
touch /etc/lsyncd/lsyncd.conf
touch /var/lsyncd/lsyncd.log
touch /var/lsyncd/lsyncd.status
```
#### 4.2 编辑/etc/lsyncd/lsyncd.conf文件
```
settings {
	-- 日志文件存放位置
    logfile ="/var/lsyncd/lsyncd.log",
	-- 监控目录状态文件的存放位置
    statusFile ="/var/lsyncd/lsyncd.status",
	-- 指定要监控的事件,如,CloseWrite,Modify,CloseWrite or Modify
    inotifyMode = "CloseWrite or Modify",
	-- 指定同步时进程的最大个数
    maxProcesses = 10,
	-- 隔多少秒记录一次被监控目录的状态
    statusInterval = 10,
	-- false=启用守护模式
    nodaemon = false,
	-- 当事件被命中累计多少次后才进行一次同步(即使间隔低于statusInterval)
    maxDelays = 20
}
sync {
	-- lsyncd运行模式
	-- default.direct=本地目录间同步
	-- default.rsync=使用rsync
	-- default.rsyncssh=使用rsync的ssh模式
    default.rsync,
	-- 同步的源目录
    source = "/temp/test_src0/",
	-- 同步的目标目录
	target = "rsync://rsync@192.168.2.230/rsync_test0",
	-- 是否同步删除 true=同步删除 false=增量备份
    delete = false,
	-- 排除同步文件
    exclude = { "dir*" },
	-- 等待rsync同步延时时间(秒)
    delay = 10,
	-- init=false则只同步lsyncd启动后变更，不设置则lsyncd启动后进行全量的同步
    --init = false,
    rsync  = {
		bwlimit=200,
        binary = "/usr/bin/rsync",
        archive = true,
        compress = true,
        verbose = true,
       	perms = true,
       	-- rsync的用户密码，不是Linux系统的用户密码
	    password_file = "/etc/lsyncd/rsync.passwd"
    }
}
```
各字段意义见配置文件，不再赘述。详细的字段配置信息请见项目主页：https://axkibe.github.io/lsyncd/manual/config/file/。
#### 4.3 编辑/etc/init.d/lsyncd
因为我们没有使用官方默认的配置文件路径，需要对lsyncd的启动脚本进行修改。将该文件中的`/etc/lsyncd.conf`替换成`/etc/lsyncd/lsyncd.conf`，不要有遗漏。
#### 4.4 启动lsyncd
```
service lsyncd start
```
现在，你可以向源端被同步的文件下添加、编辑文件，并观察目标端备份文件夹及子文件的变化。

## 0x04 后记
lsyncd基于Linux Kernal的inotify机制，所以不支持nfs作为“源端”。项目实践中也证明了这一点。

本文偏重实践，对于理论性的东西没有过多涉及。比如目标端rsync的运行方式有多种，本文只涉及了其中的一种，读者可根据自己的需求做进一步了解。
实践过程中参考了以下资料，在此表示感谢。  
[骏马金龙rsync系列][1]  
[rsync --daemon模式的实现][2]  
[rsync 启动停止脚本][3]  


  [1]: http://www.cnblogs.com/f-ck-need-u/p/7220009.html "骏马金龙-rsync系列篇"
  [2]: http://blog.51cto.com/11265257/1769944 "rsync --daemon模式的实现"
  [3]: http://blog.51cto.com/linux5588/779000 "rsync 启动停止脚本"