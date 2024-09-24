---
tags:
  - ubuntu
  - 教程
  - 技巧
title: Ubuntu
createTime: 2024/09/24 15:41:06
permalink: /article/vcq0fx99/
---
## Ubuntu使用技巧

### 修改大陆源（阿里云）

```bash
# 备份文件
cd /etc/apt
sudo cp sources.list sources.list.bak
# 替换文件内容，并保存
vim sources.list
# 文件内容如下：
deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe  multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse

# 更新软件列表
sudo apt-get update
sudo apt-get upgrade
```

### 开启SSH链接

```bash
# 更新安装源
sudo apt-get update
#安装ssh server 
sudo apt-get  install openssh-server
#启动 ssh
sudo /etc/init.d/ssh start
#查看ssh服务状态
sudo service ssh status
#关闭服务
sudo service ssh stop
#重启服务
sudo service ssh restart 
```

### 开启root用户支持远程登录

```bash
#修改root账户密码
sudo passwd root
#修改下面文件，找到 PermitRootLogin  并修改为 yes
sudo vim /etc/ssh/sshd_config
#重启系统
sudo reboot
```

### 重新获取IP地址

```bash
#释放ip地址
dhclient -r
#重新获取IP
dhclient 
#查看IP
ip addr
```

### 永久修改主机名

```bash
# 查看当前主机名
hostname
# 修改主机名
vim /etc/hostname
# 修改本地host
vim /etc/hosts
```

### 设置固定IP

#### netplan

```bash
# 修改netplan 文件
vim /etc/netplan/50-cloud-init.yaml
#修改文件对应内容，注意yaml格式, 192.168.195.141 固定IP地址，192.168.195.2 网关地址
network:
    renderer: networkd
    ethernets:
        ens32:
                addresses: [192.168.195.141/24]
                gateway4: 192.168.195.2
                nameservers:
                        addresses: [114.114.114.114,8.8.8.8]
                dhcp4: false
    version: 2
# 修改完成后,应用生效
netplan apply
#检测网络是否连通
curl -v www.baidu.com
```

#### interfaces文件修改

```bash
#查看网卡，一般以ens开头
ifconfig
#安装高级网络配置
apt install ifupdown
#修改文件
vim /etc/network/interfaces
# 文件内容
auto ens32
iface ens32 inet static
address 192.168.195.141
netmask 255.255.255.0
gateway 192.168.195.2
# 文件内容
# 重启生效
service networking restart
# 若重启服务可生效，则无需重启系统
reboot
#检测网络是否连通
curl -v www.baidu.com
```

### 关闭防火墙

```bash
#关闭防火墙
ufw disable
#关闭交换分区
swapoff -a
#永久关闭交换分区
vim /etc/fstab
#注释下面内容
/swap.img     none    swap    sw      0       0
```