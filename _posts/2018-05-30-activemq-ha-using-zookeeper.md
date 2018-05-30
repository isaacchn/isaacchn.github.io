---
layout: article
title:  "使用ZooKeeper构建ActiveMQ高可用集群"
tags: ops
---
> 一种构建ActiveMQ高可用集群的方案。

## 0x00 前言

使用ZooKeeper实现ActiveMQ HA是官方给出的集群方案的一种，其他的方案见[ActiveMQ Clustering]。官方对于ZooKeeper+ActiveMQ方案的介绍见[Replicated LevelDB Store]。

## 0x01 原理

该方案的核心思想在于在多个ActiveMQ之间实现主从复制，使用zookeeper监控这些ActiveMQ。正常状态下，这些ActiveMQ节点中会有一个节点被选举为Master节点，接受来自客户端的连接。其余Slave节点不接受来自客户端的连接。当主节点失效后，获得最近更新的Slave节点被选举为主节点。原Master节点重新上线后会成为Slave节点。

假设集群内服务器数量为`replicas`，则至少有`(replicas/2)+1`个ActiveMQ正常运行时，可以保证集群仍能响应来自客户端的请求。不难得出在该方案下，若想保证在1台服务器失效的情况下集群正常运行，至少需要3台服务器。

ActiveMQ1被钦定为Master节点：
![1]  

ActiveMQ1失效后，ActiveMQ2被选举为Master节点：
![2]

## 0x02 实施步骤
### 环境
三台CentOS7主机，IP地址分别为10.188.102.149/10.188.102.24/10.188.102.210。
### 配置防火墙
配置iptables，使各节点之间内网互通
`-A INPUT -s 10.188.102.0/24 -j ACCEPT`
### 安装ZooKeeper和ActiveMQ
在三台机器上分别安装ZooKeeper及ActiveMQ，该部分不再赘述。
### 配置ZooKeeper
编辑zoo.cfg文件：
```properties
#zoo.cfg
#Zookeeper服务器之间或客户端与服务器之间维持心跳的时间间隔
tickTime=2000
#Zookeeper接受客户端初始化连接时最长能忍受多少个心跳时间间隔数
initLimit=10
#标识Leader与Follower之间发送消息,请求和应答时间长度，最长不能超过多少个tickTime的时间长度
syncLimit=5
#Zookeeper保存数据的目录,默认情况下Zookeeper将写数据的日志文件也保存在这个目录里
dataDir=/data/zookeeper
#客户端（应用程序）连接Zookeeper服务器的端口,Zookeeper会监听这个端口接受客户端的访问请求
clientPort=2181
#zookeeper服务器 格式为server.A=B:C:D
#A位数字与myid文件值对应，B服务器地址，C集群成员信息交换端口，D是Leader挂掉后用来选举Leader的端口
#本机IP填0.0.0.0
server.1=0.0.0.0:2888:3888
server.2=10.188.102.24:2888:3888
server.3=10.188.102.210:2888:3888
```
在zookeeper的data目录新建myid文件，编辑myid文件，填入zookeeper服务器的id，与zoo.cfg中保持一致。
### 配置ActiveMQ
编辑`ACTIVEMQ_HOME/conf/activemq.xml`:
```xml
<!-- 设定持久化方案 -->
<persistenceAdapter>
    <!-- HA将持续化存储方案由默认的kahaDB改为LevelDB -->
    <replicatedLevelDB
		directory="${activemq.data}/leveldb"
		replicas="3"
		bind="tcp://0.0.0.0:0"
		zkAddress="10.188.102.149:2181,10.188.102.24:2181,10.188.102.210:2181"
		hostname="10.188.102.149"
		/>
</persistenceAdapter>

```
zkAddress填写服务器集群ip及zookeeper端口，hostname填写本机IP，replicas填写集群内机器数量。配置的时候一定要注意三台服务器填写相同的`brokerName`。

## 0x03 测试
分别启动三台服务器的zookeeper  
`ZOOKEEPER_HOME/bin/zkServer.sh start`  
分别启动三台服务器的ActiveMQ  
`ACTIVEMQ_HOME/bin/activemq start`  
查看ActiveMQ Master日志(具体哪一台服务器成为Master是随机的)
```
2018-05-16 17:40:30,047 | INFO  | Master started: tcp://10.188.102.210:43904 | org.apache.activemq.leveldb.replicated.MasterElector | ActiveMQ BrokerService[ganinfo_mq] Task-2
2018-05-16 17:40:30,207 | INFO  | Slave has connected: af091087-cd05-490c-ac33-254ccfc56490 | org.apache.activemq.leveldb.replicated.MasterLevelDBStore | hawtdispatch-DEFAULT-2
2018-05-16 17:40:30,271 | INFO  | Slave has now caught up: af091087-cd05-490c-ac33-254ccfc56490 | org.apache.activemq.leveldb.replicated.MasterLevelDBStore | hawtdispatch-DEFAULT-2
```
查看ActiveMQ Slave日志
```
2018-05-16 17:40:30,136 | INFO  | Slave started | org.apache.activemq.leveldb.replicated.MasterElector | ActiveMQ BrokerService[ganinfo_mq] Task-1
2018-05-16 17:40:30,224 | INFO  | Slave skipping download of: log/0000000000000000.log | org.apache.activemq.leveldb.replicated.SlaveLevelDBStore | hawtdispatch-DEFAULT-1
2018-05-16 17:40:30,228 | INFO  | Slave requested: 0000000000000031.index/CURRENT | org.apache.activemq.leveldb.replicated.SlaveLevelDBStore | hawtdispatch-DEFAULT-1
2018-05-16 17:40:30,230 | INFO  | Slave requested: 0000000000000031.index/MANIFEST-000004 | org.apache.activemq.leveldb.replicated.SlaveLevelDBStore | hawtdispatch-DEFAULT-1
2018-05-16 17:40:30,231 | INFO  | Slave requested: 0000000000000031.index/000006.log | org.apache.activemq.leveldb.replicated.SlaveLevelDBStore | hawtdispatch-DEFAULT-1
2018-05-16 17:40:30,232 | INFO  | Slave requested: 0000000000000031.index/000005.sst | org.apache.activemq.leveldb.replicated.SlaveLevelDBStore | hawtdispatch-DEFAULT-1
2018-05-16 17:40:30,243 | INFO  | Attaching... Downloaded 0.02/1.43 kb and 1/4 files | org.apache.activemq.leveldb.replicated.SlaveLevelDBStore | hawtdispatch-DEFAULT-1
2018-05-16 17:40:30,246 | INFO  | Attaching... Downloaded 0.11/1.43 kb and 2/4 files | org.apache.activemq.leveldb.replicated.SlaveLevelDBStore | hawtdispatch-DEFAULT-1
2018-05-16 17:40:30,248 | INFO  | Attaching... Downloaded 0.81/1.43 kb and 3/4 files | org.apache.activemq.leveldb.replicated.SlaveLevelDBStore | hawtdispatch-DEFAULT-1
2018-05-16 17:40:30,251 | INFO  | Attaching... Downloaded 1.43/1.43 kb and 4/4 files | org.apache.activemq.leveldb.replicated.SlaveLevelDBStore | hawtdispatch-DEFAULT-1
2018-05-16 17:40:30,252 | INFO  | Attached | org.apache.activemq.leveldb.replicated.SlaveLevelDBStore | hawtdispatch-DEFAULT-1
```
可以看到`10.188.102.210`这台机器被选举为主机，可以访问`http://10.188.102.210:8161`进入控制台。
停止`10.188.102.210`上的ActiveMQ服务，查看两台Slave的日志
```
2018-05-16 17:51:30,013 | INFO  | Master started: tcp://10.188.102.24:42206 | org.apache.activemq.leveldb.replicated.MasterElector | ActiveMQ BrokerService[ganinfo_mq] Task-3
```
可以看到`10.188.102.24`这台机器被选举为Master。
继续关闭`10.188.102.149`，查看日志
```
2018-05-16 17:52:58,079 | INFO  | Slave has disconnected: 4a9f578c-7240-42a5-a70b-7141b7eec515 | org.apache.activemq.leveldb.replicated.MasterLevelDBStore | hawtdispatch-DEFAULT-2
2018-05-16 17:53:01,027 | INFO  | Not enough cluster members connected to elect a master. | org.apache.activemq.leveldb.replicated.MasterElector | main-EventThread
2018-05-16 17:53:01,035 | INFO  | Not enough cluster members connected to elect a master. | org.apache.activemq.leveldb.replicated.MasterElector | main-EventThread
2018-05-16 17:53:01,037 | INFO  | Demoted to slave | org.apache.activemq.leveldb.replicated.MasterElector | main-EventThread
2018-05-16 17:53:01,040 | INFO  | Apache ActiveMQ 5.15.3 (ganinfo_mq, ID:activemq_2-56827-1526464290093-0:1) is shutting down | org.apache.activemq.broker.BrokerService | ActiveMQ BrokerService[ganinfo_mq] Task-6
2018-05-16 17:53:01,048 | INFO  | Connector openwire stopped | org.apache.activemq.broker.TransportConnector | ActiveMQ BrokerService[ganinfo_mq] Task-6
2018-05-16 17:53:01,062 | INFO  | Stopped LevelDB[/opt/activemq/data/leveldb] | org.apache.activemq.leveldb.LevelDBStore | ActiveMQ BrokerService[ganinfo_mq] Task-5
2018-05-16 17:53:01,065 | INFO  | Master stopped | org.apache.activemq.leveldb.replicated.MasterElector | ActiveMQ BrokerService[ganinfo_mq] Task-5
2018-05-16 17:53:01,066 | INFO  | Not enough cluster members connected to elect a master. | org.apache.activemq.leveldb.replicated.MasterElector | ActiveMQ BrokerService[ganinfo_mq] Task-5
```
可以看到由于可用节点数量不足(<2)，导致整个集群失效。

至此，我们的ActiveMQ高可用集群配置工作正式完成。

[1]:https://markdown-1251260884.cos.ap-shanghai.myqcloud.com/ActiveMQ%2Bzookeeper%E5%AE%9E%E7%8E%B0%E9%AB%98%E5%8F%AF%E7%94%A8%E9%9B%86%E7%BE%A4/ActiveMQ%2Bzookeeper%E5%AE%9E%E7%8E%B0%E9%AB%98%E5%8F%AF%E7%94%A8%E9%9B%86%E7%BE%A42.png
[2]:https://markdown-1251260884.cos.ap-shanghai.myqcloud.com/ActiveMQ%2Bzookeeper%E5%AE%9E%E7%8E%B0%E9%AB%98%E5%8F%AF%E7%94%A8%E9%9B%86%E7%BE%A4/ActiveMQ%2Bzookeeper%E5%AE%9E%E7%8E%B0%E9%AB%98%E5%8F%AF%E7%94%A8%E9%9B%86%E7%BE%A41.png
[ActiveMQ Clustering]:http://activemq.apache.org/clustering.html
[Replicated LevelDB Store]:http://activemq.apache.org/replicated-leveldb-store.html