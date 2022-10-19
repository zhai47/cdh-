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
```cat >> /etc/hosts << EOF
node1的address node1的hostname  
node2的address node2的hostname  
node3的address node3的hostname  
```

