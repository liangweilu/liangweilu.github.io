---
layout: post
title: Windows环境下在Docker容器中运行一个Java服务
date: 2018-11-05
tags: Docker
---
#### 预备工作
- 了解Docker相关的基本概念，容器，镜像等。这里有一个中文版的[Docker入门实践](https://yeasy.gitbooks.io/docker_practice/content/)，介绍了docker相关的基本概念和知识。
- 已经安装好了docker服务并确保可用。本文基于win 10，这里是安装教程：[Install Docker for Windows](https://docs.docker.com/docker-for-windows/install/)
- 准备好一个可用的Java服务（本文基于springboot建立的一个服务）

#### LET'S GO
###### 1、用Dockerfile制作基础的ubuntu14.04镜像
&emsp;&emsp;Dockerfile是我们用来构建和制作docker镜像的基本文件，它定义了镜像的构成。新建一个文件夹命名为`/docker`，在此目录下新建文件夹`/ubuntu-14.04`，然后在此目录下新建一个文件命名为`Dockerfile`（虽然可以命名为其他名称，但是不建议，因为这个是约定俗成的）。这个时候你的目录应该是和下图一样，不一样也没关系，只要能进入这个`Dockerfile`所在的目录即可，我这样是为了看着清晰明了。  
![](https://byeluliangwei.github.io/images/docker/1.png)  
现在我们需要编写这个`Dockerfile`，需要写入一些基本的命令和信息，复制的信息到文件中即可：
```
# IMAGES REPOSITORY   ubuntu:14.04
# DESCRIPTION         基于ubuntu14.04构建镜像并设置了系统编码和时区
# VERSION             1.0.0

# 使用此ubuntu基础镜像开始构建
FROM ubuntu:14.04

# 指定镜像创建者
MAINTAINER 1874

# 编码问题
ENV LANG            zh_CN.UTF-8
ENV LANGUAGE        zh_CN:zh
ENV LC_ALL          zh_CN.UTF-8

RUN locale-gen zh_CN.UTF-8 && \
    DEBIAN_FRONTEND=noninteractive dpkg-reconfigure locales && \
    locale-gen zh_CN.UTF-8

# 时区问题
ENV TZ Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
```
然后进入`Dockerfile`这个文件所在的目录，在这个目录下打开Windows PowerShell或者DOS，这里我是用的PowerShell。然后输入命令:`docker build -t luliangwei/ubuntu:14.04 .`来构建镜像。注意命令中带有一个`.`，这个是代表docker引擎的上下文环境，而并不是我们平时所理解的当前目录。举个例子，我们现在所在的文件位置是`/docker/ubuntu-14.04/Dockerfile`,docker构建镜像的时候默认是从`Dockerfile`所在的位置开始进行构建的，也就是说`.`代表的是`Dockerfile`所在的位置。其实并不是，假如docker引擎上下文的路径是`/var/lib/21888342/`，那么此时`.`代表的位置就是`/var/lib/21888342/...`。要是还没有明白的话，就看上文中所说的[Docker入门实践](https://yeasy.gitbooks.io/docker_practice/content/)，上面有讲解的。回到这里，我们运行了命令后，就会出现如下的输出：
```
PS D:\docker\ubuntu-14.04> docker build -t ubuntu:14.04 .
Sending build context to Docker daemon   2.56kB
Step 1/8 : FROM ubuntu:14.04
14.04: Pulling from library/ubuntu
027274c8e111: Pull complete
d3f9339a1359: Pull complete
872f75707cf4: Pull complete
dd5eed9f50d5: Pull complete
Digest: sha256:e6e808ab8c62f1d9181817aea804ae4ba0897b8bd3661d36dbc329b5851b5637
Status: Downloaded newer image for ubuntu:14.04
 ---> f216cfb59484
Step 2/8 : MAINTAINER 1874
 ---> Running in 38756fb563cd
Removing intermediate container 38756fb563cd
 ---> 099c8fb7f5c4
Step 3/8 : ENV LANG            zh_CN.UTF-8
 ---> Running in 178ac47cdb17
Removing intermediate container 178ac47cdb17
 ---> e8384ffc6de7
Step 4/8 : ENV LANGUAGE        zh_CN:zh
 ---> Running in 89b4d41bcb4e
 ...
```
这代表这镜像正在一步步的构建，执行完成之后输入`docker images`可以看到本地的所有镜像，如下图所示
![](https://byeluliangwei.github.io/images/docker/2.png)  
图中TAG为`<none>`的镜像时ubuntu的基础镜像，上面TAG为`14.04`的镜像时我们基于这个基础镜像，添加了时间和编码问题的镜像。然后我们运行这个镜像，就相当于运行了一个ubuntu操作系统，这个运行起来的镜像，叫做容器，`ls`一下即可看到一些基本的文件夹，如图所示
![](https://byeluliangwei.github.io/images/docker/3.png)
###### 2、基于步骤1中的镜像添加s6-overlay进程管理工具
&emsp;&emsp;步骤和前面一样，进入该`Dockerfile`文件所在的位置，在`Dockerfile`文件中写入面的信息。关于[s6-overlay更详细的信息](https://github.com/just-containers/s6-overlay)，可以去github上看，
```
# IMAGES REPOSITORY   luliangwei/ubuntu:14.04
# DESCRIPTION         安装S6进程管理工具
# VERSION             1.0.0

# 使用此ubuntu基础镜像开始构建
FROM luliangwei/ubuntu:14.04

# 指定镜像创建者和联系方式
MAINTAINER 1874

# 添加s6-overlay进程管理工具
ENV S6_OVERLAY_VERSION          v1.19.1.1

# 安装s6-overlay
ADD https://github.com/just-containers/s6-overlay/releases/download/${S6_OVERLAY_VERSION}/s6-overlay-amd64.tar.gz /tmp/

RUN tar xzf /tmp/s6-overlay-amd64.tar.gz -C / && \
    rm -rf /tmp/*

ENTRYPOINT ["/init"]
CMD []
```
同样执行命令`docker build -t luliangwei/ubuntu:s6-overlay .`构建镜像，构建完成后查看镜像可以看到多了一个，如下图所示：
![](https://byeluliangwei.github.io/images/docker/4.png)
其实这个镜像就是在上一个镜像的基础上安装了一个进程管理工具，为什么要这样来制作镜像，不直接在上一步安装呢，相信看完[Docker入门实践](https://yeasy.gitbooks.io/docker_practice/content/)你会明白的。
###### 3、基于步骤2中的镜像制作Java环境的镜像
&emsp;&emsp;步骤和上面一样，在`Dockerfile`文件中写入相关信息，然后构建镜像，当然，这个`Dockerfile`仍然需要和前面两个一样，单独的放在一个文件夹中。复制下面的信息到该`Dockerfile`文件中并在命令行中执行`docker build -t luliangwei/ubuntu:java8 .`即可。  
```
# IMAGES REPOSITORY   luliangwei/ubuntu:java8
# DESCRIPTION         安装Java8环境
# VERSION             1.0.0

# 使用此ubuntu基础镜像开始构建
FROM luliangwei/ubuntu:s6-overlay

# 指定镜像创建者和联系方式
MAINTAINER 1874

# 安装Java8运行环境
RUN echo "deb http://ppa.launchpad.net/webupd8team/java/ubuntu trusty main"\
    > /etc/apt/sources.list.d/webupd8team-java.list \
    && echo "deb-src http://ppa.launchpad.net/webupd8team/java/ubuntu trusty main"\
    >> /etc/apt/sources.list.d/webupd8team-java.list \
    && apt-key adv --keyserver keyserver.ubuntu.com --recv-keys EEA14886 \
    && apt-get update -y \
    && echo oracle-java8-installer shared/accepted-oracle-license-v1-1 select true | /usr/bin/debconf-set-selections \
    && apt-get install -y --no-install-recommends oracle-java8-installer=8u191-1~webupd8~1 \
    && apt-get autoremove \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* /var/cache/oracle-jdk8-installer

# 设置Java8系统环境变量
ENV JAVA_HOME /usr/lib/jvm/java-8-oracle
```
如果制作镜像过程中出现java版本找不到之类的错误，只需换一个版本号即可。完成之后运行镜像，`docker run -it --rm 镜像ID /bin/bash`然后输入`java -version` 即可看到相关信息，代表安装java环境成功。至此，我们就完成了一个基础的带有Java环境的镜像了，后续我们都可以基于这个镜像来运行Java服务。
###### 4、基于步骤3中的镜像制作特定服务的镜像
&emsp;&emsp;这一步我们就开始为想要运行的特定服务制作镜像了。本例中我使用springboot写了一个简单的可以运行的服务，端口为：1874。首先还是要制作一个`Dockerfile`,内容如下：
```
# IMAGES REPOSITORY   luliangwei/docker-demo:0.0.1
# DESCRIPTION         安装docker-demo相关环境
# VERSION             0.0.1

# 指定基础镜像
FROM luliangwei/ubuntu:java8

# 指定镜像创建者和联系方式
MAINTAINER 1874

# 环境变量定义
ENV SERVER_SERVICE              docker-demo
ENV SERVER_VERSION              0.0.1-SNAPSHOT
ENV SERVER_CONTEXT              /
ENV SERVER_TYPE                 jar
ENV SERVER_ROOT_HOME            /usr/local/bin/${SERVER_SERVICE}
#ENV SERVER_BIN                  ${SERVER_ROOT_HOME}/${SERVER_SERVICE}.${SERVER_TYPE}
ENV SERVER_BIN                  ${SERVER_ROOT_HOME}/${SERVER_SERVICE}-${SERVER_VERSION}.${SERVER_TYPE}
ENV SERVER_DEFAULT_PORT         1874
ENV SERVER_DEFAULT_SSL_PORT     41874

#
ADD ./run /etc/services.d/${SERVER_SERVICE}/

RUN mkdir -p ${SERVER_ROOT_HOME}

# 将jar包复制到容器内（这种做法不好），实际生产中应该从nexus库中取
ADD ./docker-demo-0.0.1-SNAPSHOT.jar ${SERVER_ROOT_HOME}

# 给run文件执行权限
RUN chmod +x /etc/services.d/${SERVER_SERVICE}/run

# 执行run脚本，运行jar包
ENTRYPOINT sh /etc/services.d/${SERVER_SERVICE}/run

#需要暴露的端口
EXPOSE ${SERVER_DEFAULT_PORT}
```
构建镜像之前，我们需要将打包好的可运行jar包放入`Dockerfile`所在的位置，同时还需要写一个`run`脚本来启动这个jar包，脚本内容如下所示：  
```
#!/usr/bin/with-contenv sh

# 启动服务
java -Xms256m -Xmx256m -XX:MaxNewSize=256m -XX:MaxPermSize=256m -jar ${SERVER_BIN}
```
其实就是一个执行jar包的命令，如：java -jar *** 。当前目录下的文件如图所示
![](https://byeluliangwei.github.io/images/docker/5.png)  
现在我们就可以运行这个镜像，例如我的镜像ID是4b4开头的ID，所以我直接执行`docker run -it --rm -d -p 8080:1874 4b4`，然后直接通过浏览器获取curl即可访问http://localhost:1874/users?name=luliangwei 就可以得到对应的信息。如果你的jar包是自己写的服务，直接访问自己对应的地址即可。
#### 总结
&emsp;&emsp;windows环境下可能会遇到挂载数据卷的时候出现找不到文件的问题，可以考虑在PowerShell下设置环境变量`COMPOSE_CONVERT_WINDOWS_PATHS`为true，方法就是直接运行命令`$Env:COMPOSE_CONVERT_WINDOWS_PATHS=1`即可。PS：可以使用命令`ls env:`查看所有的环境变量及其值。另外，本例工程在github中已改成基于docker-compose来编排服务，镜像也是基于一个mini版的linux系统alpine来制作的。

相关Dockerfile文件：<https://github.com/byeluliangwei/docker>  
本例中使用的工程：<https://github.com/byeluliangwei/docker-demo>
