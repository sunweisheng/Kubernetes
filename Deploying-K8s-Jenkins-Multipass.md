# 使用Multipass虚拟机搭建K8S构建部署环境

## 安装虚拟机
访问Multipass官方下载页面 https://canonical.com/multipass/install 下载并安装对应的版本。  
````shell
#安装之后检查是否安装成功
multipass version

multipass   1.16.3+mac
multipassd  1.16.3+mac

#查看基础系统环境内容
multipass get local.driver

qemu

multipass networks

Name   Type       Description
en0    wifi       Wi-Fi
en3    ethernet   Ethernet Adapter (en3)
en4    ethernet   Ethernet Adapter (en4)
````

## 创建虚拟机3台
三台虚拟机分别是192.168.0.10（Master）、192.168.0.11（Node1）、192.168.0.12（Node2）。  
### 创建Master
````shell
````

## 安装UTM虚拟机
https://mac.getutm.app

## 安装RouterOS虚拟路由器
https://mikrotik.com/download/chr