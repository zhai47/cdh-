# 虚拟机搭建cdh平台

## 1. 准备工作  
虚拟机安装：虚拟机VMware Workstation Pro；  
镜像下载：Centos8； 
创建新的虚拟机：主节点配置为  
![image](https://user-images.githubusercontent.com/90238615/195975096-2d96d376-7a93-4fa9-8425-a5da118bb29b.png) 

启用虚拟机共享文件夹  
![image](https://user-images.githubusercontent.com/90238615/195975163-65228821-f5d3-48c5-8903-d17a91103b45.png) 
![image](https://user-images.githubusercontent.com/90238615/195975174-385204a4-b7ed-411e-9335-d6448fcd5423.png)

上传cdh包； 
解压cdh压缩包； 
` tar -xvf  cdh6.3.2.tar  -C /tmp/ `

因Centos8官方把源撤了，此处需要更改源  
` sudo sed -i -e "s|mirrorlist=|#mirrorlist=|g" /etc/yum.repos.d/CentOS-* `  
` sudo sed -i -e "s|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g" /etc/yum.repos.d/CentOS-* `  
yum就能用了  

## 2. 安装必要组件  
` yum install expect -y `  
` yum install psmisc -y `  
` yum install perl -y `  
` yum install httpd mod_ssl -y `  

## 3. 配置文件  
### 1. 配置静态ip  
找到你的网卡文件  
` cd /etc/sysconfig/network-scripts `  
找到后编辑它
` vi ifcfg-ens160 `  
![image](https://user-images.githubusercontent.com/90238615/195975747-c2cc358f-be43-49c5-b79e-d3a769b4aae8.png)  
CentOS8 重启网卡的语句好像换了，systemctl 不能用了，用下面这个
` nmcli c reload ens160 `  
` nmcli c up ens160 `  
` ifconfig `  
ifconfig检查一下  

主节点电源关机，克隆主节点，做两台结点机器  
![image](https://user-images.githubusercontent.com/90238615/195975866-0ec11495-606b-43e5-b9f3-f2a7d9b7d987.png)

同样，两台新机器，配置静态ip，配置好后用ifconfig检查下

### 2. 配置host文件
修改主节点名字为cdh-01；
` hostnamectl set-hostname cdh-01 `  
` hostname `  
hostname 检查下  

同样去修改另外两台机器，修改为 cdh-02、cdh-03；

/etc/hosts文件下面写入主机和地址们
`cat >> /etc/hosts << EOF  
node1的address node1的hostname  
node2的address node2的hostname  
node3的address node3的hostname  
`

## 4. 关闭selinux访问控制（在所有节点执行）
` grep ^SELINUX= /etc/selinux/config`  
` sed -i 's/^SELINUX=.*/SELINUX=disabled/g' /etc/selinux/config`  
` setenforce 0`  
` getenforce`  

## 5. 关闭iptables（在所有节点执行）
` systemctl disable firewalld`  
` systemctl stop firewalld`  

## 6. 配置ssh互信（在所有节点执行）
### 所有结点执行
` ssh-keygen -t rsa `  
### 在主节点上执行
` ssh-copy-id root@cdh-01`  
` ssh-copy-id root@cdh-02`
` ssh-copy-id root@cdh-03`  
` scp ~/.ssh/authorized_keys root@cdh-02:/root/.ssh/`  
` scp ~/.ssh/authorized_keys root@cdh-03:/root/.ssh/`
### 完成后验证效果
`ssh cdh-02`  
` exit`

## 7. 调整系统使用swap的策略（所有节点执行）
` sysctl -a | grep vm.swappiness `  
` echo 1 > /proc/sys/vm/swappiness`  
` sysctl -a | grep vm.swappiness`  
```shell
cat >> /etc/sysctl.conf << EOF  
  vm.swappiness=1  
  EOF
```  
` sysctl vm.swappiness=1 `  
``` shell 
cat >> /etc/rc.d/rc.local << EOF
if test -f /sys/kernel/mm/transparent_hugepage/enabled; then
  echo never > /sys/kernel/mm/transparent_hugepage/enabled
fi
if test -f /sys/kernel/mm/transparent_hugepage/defrag; then
  echo never > /sys/kernel/mm/transparent_hugepage/defrag
fi
EOF
```
` echo never > /sys/kernel/mm/transparent_hugepage/enabled`  
` echo never > /sys/kernel/mm/transparent_hugepage/defrag`  
` cat /sys/kernel/mm/transparent_hugepage/enabled`  
` cat /sys/kernel/mm/transparent_hugepage/defrag`  
` chmod 755 /etc/rc.d/rc.local`  
## 8. 关闭透明大页（所有节点执行）
## 9. 配置sudo相关（所有节点执行）
![image](https://user-images.githubusercontent.com/90238615/201812316-9a51f19e-49be-4c34-9809-78d32743e685.png)
## 10. 配置ntp时间同步（需求节点执行）
CentOS8，ntp没了，只要时间同步就行
## 11. 安装JDK（需求节点执行）
先把jdk包分发到其他节点  
![image](https://user-images.githubusercontent.com/90238615/201813078-e441453c-8380-42b6-9a02-be60c6b1af5e.png)  
安装一下，（后面[tab]默认为tab键自动补全，懒得写了）  
yum install ./oracle-[tab] -y  
ln -s /usr/java/jdk[tab] /usr/java/latest  
![image](https://user-images.githubusercontent.com/90238615/201814395-3105edb1-d910-40ec-8526-213fca6e4e55.png)  
![image](https://user-images.githubusercontent.com/90238615/201815970-59e47690-73ca-49c9-84bb-40bf0ef81b64.png)  
## 12. 安装、配置mysql（主节点执行即可）
CentOS 8 MySQL安装出问题，参考这个：https://blog.csdn.net/qq_42682745/article/details/123287788  
远程连接不上虚拟机看这个：https://blog.csdn.net/arlene12345/article/details/122933316  

## 13. 创建CM元数据库（主节点执行）
需要先修改你的MySQL密码配置项，或者使用强密码  
create database metastore default character set utf8;  
create user 'hive'@'%' identified by 'hivedemima';  
grant all privileges on metastore.* to 'hive'@'%';  

create database cm default character set utf8;  
create user 'cm'@'%' identified by 'cmdemima';  
grant all privileges on cm.* to 'cm'@'%';  

create database am default character set utf8;  
create user 'am'@'%' identified by 'amdemima';  
grant all privileges on am.* to 'am'@'%';  

create database rm default character set utf8;  
create user 'rm'@'%' identified by 'rmdemima';  
grant all privileges on rm.* to 'rm'@'%';  

create database hue default character set utf8;  
create user 'hue'@'%' identified by 'huedemima';  
grant all privileges on hue.* to 'hue'@'%';  

create database oozie default character set utf8;  
create user 'oozie'@'%' identified by 'ooziedemima';  
grant all privileges on oozie.* to 'oozie'@'%';  

create database sentry default character set utf8;  
create user 'sentry'@'%' identified by 'sentrydemima';  
grant all privileges on sentry.* to 'sentry'@'%';  

create database nas default character set utf8;  
create user 'nas'@'%' identified by 'nasdemima';  
grant all privileges on nas.* to 'nas'@'%';  

create database nms default character set utf8;   
create user 'nms'@'%' identified by 'nmsdemima';  
grant all privileges on nms.* to 'nms'@'%';  

grant all privileges on *.* to 'root'@'%' identified by 'root';  

flush privileges;  
exit;  

## 14. 配置http服务（主节点执行）
![image](https://user-images.githubusercontent.com/90238615/201835962-67e9e8e4-c20c-4d67-9221-c1752c27d1d1.png)
![image](https://user-images.githubusercontent.com/90238615/201835927-ae8ff627-c280-4384-8bbb-7730edaae1b3.png)
![image](https://user-images.githubusercontent.com/90238615/201835949-23370d80-6a92-4d2f-bfc7-149ca9cca3d2.png)

## 15. 创建cdh介质的本地源（主节点执行）
CentOS8 没有createrepo，制作yum源这步先过
## 16. 安装cdh介质的本地源（主节点执行）
![image](https://user-images.githubusercontent.com/90238615/201837661-bcbe676f-9b4e-47db-a43f-c477bf44326d.png)
