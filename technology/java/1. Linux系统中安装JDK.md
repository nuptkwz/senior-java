# 前言

> 本文记录了在Linux环境中安装Jdk的过程，为了方便后续快速安装。

# 下载

1. 进入Oracle官方网站的[下载页面](https://www.oracle.com/java/technologies/downloads/)
   ![下载地址.png](https://upload-images.jianshu.io/upload_images/9905084-99d4d1fa2baf7f1d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
2. 下载
   ![复制下载链接.png](https://upload-images.jianshu.io/upload_images/9905084-5c2c6d7c63696a76.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
   如果没能跳转到这个页面，可能是没有登陆，想下载JDK，必须有Oracle账号，在知乎上找到一个临时账号使用（此账号仅供下载JDK使用）：

```
用户名：OneMoreStudy@163.com
密码：One-More-Study-666
```

3. 创建一个目录放jdk

```
mkdir /usr/local/java/
```

4. 上传到Linux系统中

```
scp jdk-8u321-linux-x64.tar.gz root@192.168.30.130:/usr/local/java/
```

# 安装

```
tar -zxvf jdk-8u321-linux-x64.tar.gz
```

# 配置环境变量

执行以下命令，编辑 /etc/bash.bashrc文件：

```
vim /etc/profile
```

增加jdk的环境变量

```
export JAVA_HOME=/usr/local/java/jdk1.8.0_321
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
export PATH=${JAVA_HOME}/bin:$PATH
```

5. 使环境变量立即生效

```
source /etc/profile
```

# 验证

执行Java的打印版本命令：

```
java -version
```

显示：

```
java version "1.8.0_321"
Java(TM) SE Runtime Environment (build 1.8.0_321-b07)
Java HotSpot(TM) 64-Bit Server VM (build 25.321-b07, mixed mode)
```

# 参考

1. [详解在Linux系统中安装JDK](https://zhuanlan.zhihu.com/p/94010951)
2. [Linux下安装jdk的两种方法](https://www.cnblogs.com/Dr-wei/p/13339957.html)