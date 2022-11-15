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
