---
title: Hadoop的安装与部署
date: 2018-07-28 23:24:33
tags:
---
#分布式环境配置
##（一）虚拟机配置
本项目我是用VMware Workstation Pro 建立三台Centos系统的虚拟机下完成的。一台作为从master主节点，另外两台作为slaves从节点。
###1、关闭防火墙
进入master节点，使用root权限，输入firewall-cmd --state即可
###2、修改主机名
修改/etc/hostname文件  
vi /etc/hostname
删除：localhost.localdomain  
添加：master  
保存退出:wq!  
其他两台分别为：  
Slave1,Slave2  
修改完成后，查看修改后的主机名。
###3、配置ip地址
因为所用VM虚拟机的网络设置为共享网络，故只需要在ens33配置文件将开机启动设置为yes即可，因每增加一台虚拟机，VM变会将ip进行自增，且下次开机不会改变。  
vi /etc/sysconfig/network-scripts/ifcfg-ens33  
配置完成后，重启网络服务：  
/etc/init.d/network restart    
此时，ip addr即可查看相应的ip  
本人搭建了三台虚拟机，其ip分别为：  
192.168.121.128    Master  
192.168.121.129    Slave1  
192.168.121.130    Slave2  
###4、hosts配置
vi /etc/hosts  
添加  
192.168.121.128     Master  
192.168.121.129     Slave1  
192.168.121.130     Slave2  
配置完成后，三台主机可以互PING域名。
###5、SSH无密码验证配置
Hadoop运行过程中需要管理远端Hadoop守护进程，在Hadoop启动以后，NameNode是通过SSH（Secure Shell）来启动和停止各个DataNode上的各种守护进程的。这就必须在节点之间执行指令的时候是不需要输入密码的形式，故我们需要配置SSH运用无密码公钥认证的形式，这样NameNode使用SSH无密码登录并启动DataName进程，同样原理，DataNode上也能使用SSH无密码登录到NameNode。  
####① 配置Master和所有的Slave机器的sshd_config  
vi /etc/ssh/sshd_config  
将下边三个选项进行配置  
RSAAuthentication yes # 启用 RSA 认证  
PubkeyAuthentication yes # 启用公钥私钥配对认证方式  
AuthorizedKeysFile .ssh/authorized_keys # 公钥文件路径  
保存退出:wq，后要将sshd服务重启  
systemctl restart  sshd.service  #启动服务  
然后分别在所有机器的hadoop用户的~/目录下分别建立.ssh文件夹并将其权限设为700。   
####②在每台机器上上生成密码对  
ssh-keygen -t rsa -P ''  
####③将Slaves中的公钥发送到Master中  
Slave1  
scp ~/.ssh/id_rsa.pub root@Master:~/.ssh/id_rsa1.pub  
Slave2  
scp ~/.ssh/id_rsa.pub root@Master:~/.ssh/id_rsa2.pub  
####④在Master中统一将公钥追加到authorized key中  
cat ~/.ssh/id_rsa1.pub >> ~/.ssh/authorized_keys  
cat ~/.ssh/id_rsa2.pub >> ~/.ssh/authorized_keys  
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys  
rm ~/.ssh/id_rsa1.pub  
rm ~/.ssh/id_rsa2.pub  
####⑤把公钥复制所有的Slave机器上  
scp ~/.ssh/authorized_keys root@Slave1:~/.ssh  
scp ~/.ssh/authorized_keys root@Slave2:~/.ssh  
####⑥修改文件”authorized_keys”权限  
chmod 600 ~/.ssh/authorized_keys  
配置完SSH无密码通信后，测试SSH已经成功进行无密码通信  
##（二）Hadoop分布式集群搭建 
###1、下载并安装JDK
下载JDK 7  
把下载的jdk复制到每台机器/usr/java目录下面，解压安装  
tar –zxvf  jdk-7u9-linux-i586.tar.gz   

配置JDK环境变量  
vi  /etc/profile    
在该文件最后一行添加：  
JAVA_HOME=/usr/java/jdk1.7.0_09    
PATH=$JAVA_HOME/bin:$JAVA_HOME/jre/bin:PATH    
CLASSPATH=.:$JAVA_HOME/lib:$JAVA_HOME/jre/lib   
  
使修改的操作生效  
source   /etc/profile       
  
验证是否已配置成功:  
java  -version   
###2、下载并安装Hadoop
我下载的hadoop版本是官网的原生2.5.0版本，把下载的包复制到虚拟机上，并解压：  
tar  –zxvf   hadoop-2.5.0.tar.gz  
###3、配置Hadoop
  进入配置目录，开始配置:  
cd  hadoop-2.5.0/conf  
   
1.配置core-site.xml   
```
<property>  
	<name>hadoop.tmp.dir</name>  
	<value>file:/home/ysw/cloud/project/tmp</value>  
</property>  
<property>  
	<name>fs.default.name</name>  
	<value>hdfs://master:9000</value>  
</property>  
```

2.配置hdfs-site.xml  
```  
<property>  
	<name>dfs.repplication</name>  
	<value>2</value>  
</property>  
<property>  
	<name>dfs.permissions.enabled</name>  
	<value>false</value>  
</property>  
<property>  
	<name>dfs.namenode.name.dir</name>  
	<value>file:/home/ysw/cloud/project/tmp/dfs/name</value>  
</property>  
<property>  
	<name>dfs.datanode.data.dir</name>  
	<value>file:/home/ysw/cloud/project/tmp/dfs/data</value>  
</property>  
```

3.配置mapred-site.xml  
```
<property>  
	<name>mapred.job.tracker</name>  
	<value>http://master:9001</value>  
</property>  
<property>  
	<name>mapred.local.dir</name>  
	<value>/home/ysw/cloud/project/var</value>  
</property>  
<property>  
	<name>mapreduce.framework.name</name>  
	<value>yarn</value>  
</property>  
```
  
4.配置hadoop-env.sh\yarn-env.sh的JAVA_HOME  
export JAVA_HOME=/usr/lib/java/jdk1.7.0_80  
  
5.配置slaves，删除默认的localhost，增加两个从节点  
  
6.配置master，添加主节点  
  
7.将配置好的hadoop复制到各个节点对应位置上，通过scp传送  
Scp –r /home/Hadoop slave1:/home  
Scp –r /home/Hadoop slave2:/home  
  
###4、启动Hadoop集群  
在Master服务器启动hadoop，从节点会自动启动。 
   
1.在master节点初始化namenode：  
bin/hadoop namenode   –format  
  
2.启动hadoop服务：  
Start-all.sh  
  
3.输入jps查看相关信息，查看hadoop服务是否启动。   
  浏览器地址栏输入192.168.1.201(master):50070查看HDFS相关信息  
  输入192.168.1.201(master):8088查看Yarn相关信息。
  
至此，Hadoop的安装部署已经完成。

