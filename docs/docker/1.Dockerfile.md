@[toc]
# 前言
本文主要讲解了docker中Dockerfile的一些用法，学习Dockerfile就可以根据项目中需要的东西来构建自己的镜像，去创建容器来做一些工作。
# Dockerfile指令
指令|描述|举例
-|-|-|
FROM|构建的新镜像是基于哪个基础镜像|如：FROM centos:6(基础镜像名：版本号)|
MAINTAINER|镜像维护者姓名或者邮箱地址|如：MAINTAINER keweizhou
RUN|构建镜像时运行的Shell命令|如：RUN ["yum","install","httpd"] (通过数组的形式) RUN yum install httpd（通过bash命令的形式）
CMD|运行容器时执行的Shell命令|如：CMD["-e","/start.sh"]  CMD["/usr/sbin/sshd","-D"]  CMD /usr/sbin/sshd -D
EXPOSE|声明容器运行的服务端口|例如：EXPOSE 80 443
ENV|设置容器内环境变量|例如：ENV MYSQL_ROOT_PASSWORD 123456
ADD|拷贝文件或目录到镜像，如果是URL或者压缩包会**自动下载或者自动解压**|例如：ADD \<src>... \<dest> ADD["\<src>","\<dest>"]  ADD https://xx.yyy/html.tar.gz   /var/www/html  ADD html.tar.gz  /var/www/html 
COPY|拷贝文件或者目录到镜像，用法同上| 例如：COPY ./start.sh   /start.sh
ENTRYPOINT|运行容器执行的Shell命令|例如：ENTRYPOINT["/bin/bash","-c","/start.sh"]  ENTRYPOINT/bin/bash -c '/start.sh'
VOLUME|指定容器挂载点到宿主机自动生成的目录或其他容器| 例如：VOLUME [''/var/lib/mysql]
如果是拷贝压缩包需要解压的话用ADD，单纯拷贝一个文件用COPY。还有USER、WORKDIR、HEALTHCHECK、ARG等命令不一一列举了。
**注意：**

 - CMD如果写多条的话，只有最后一条生效！所以一般一个Dockerfile里面只有一个这个命令，要么启动一个脚本，要么直接启动一个服务。
# Build镜像命令
```xml
docker image build -t nginx:v1 -f /path/Dockerfile /path
```
其中：**nginx**代表repository，如果是私有仓库要指定仓库地址，**v1**表示Tag
-f指定文件名
查看nginx的配置文件：vi ../nginx/nginx.conf
两种启动服务的方式：
1. 将服务包放在宿主机上，然后挂载到容器中
2. 将服务包直接拷贝到镜像中，启动容器
# 构建JAVA网站环境镜像
## Dockerfile
```sh
FROM centos:7
MAINTAINER keweizhou@123.com

ADD jdk-8u45-linux-x64.tar.gz /usr/local
ENV JAVA_HOME /usr/local/jdk1.8.0_45

ADD apache-tomcat-8.0.46.tar.gz /usr/local
COPY server.xml /usr/local/apache-tomcat-8.0.46/conf

RUN rm -f /usr/local/*.tar.gz

WORKDIR /usr/local/apache-tomcat-8.0.46
EXPOSE 8080
ENTRYPOINT ["./bin/catalina.sh","run"]
```
## 构建镜像
## 创建容器