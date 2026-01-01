# Docker简介
ps -ef 就能列出你当前工作的一些进程
yum install pstree: 装这个命令可以查看进程树

# Docker安装
## 环境
ubuntu 16.04 
## 步骤
 **1. 更新系统软件**
 ```yml
 sudo apt-get update
```
 **2. 安装软件包以允许apt通过HTTPS使用存储库**
 ```yml
 sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
```
 **3. 添加Docker的官方GPG密钥**
 ```yml
 curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```
　　<font color='red'> **注意！** </font>
在执行此条命令的时候会报错！需要将命令分开执行才行！具体如下：

 - wget wget https://download.docker.com/linux/ubuntu/gpg
 - sudo apt-key add gpg

显示OK的话,表示添加成功
 **4. 再次更新系统软件**
 ```yml
sudo apt-get update
```
 **5. 安装docker**
 ```yml
sudo apt-get install docker-ce
```
<font color='red'> **注意！** </font>
如果想指定安装某一版本，可使用 sudo apt-get install docker-ce=<VERSION>  命令，把<VERSION>替换为具体版本即可，如果没有指定版本，默认就会安装最新版。
 **6. 检查docker版本**
 ```yml
docker -v
```
 **7. docker info 查看docker的基本信息**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200115225917417.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbmF0XzIyNzk3NDI5,size_16,color_FFFFFF,t_70)
显示："Docker version 19.03.5, build 633a0ea838"，表示安装成功！
官网地址：[https://docs.docker.com/install/linux/docker-ce/ubuntu/](https://docs.docker.com/install/linux/docker-ce/ubuntu/)
## 其他 
 - 启动docker
systemctl start docker
 - 卸载docker和删除工作目录
yum remove docker-ce
rm -rf /var/lib/docker 
## 目录说明
 -  Docker Root Dir: /var/lib/docker
docker存放文件的位置
 -  Registry: https://index.docker.io/v1/
docker默认拉去镜像的地址

参考：
