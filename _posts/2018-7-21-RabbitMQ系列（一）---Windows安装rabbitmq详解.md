---
layout: post
title: RabbitMQ系列（一）---Windows安装rabbitmq详解
date: 2018-7-22
tags: RabbitMQ
---
### 前言  

&emsp;&emsp;RabbitMQ是一个在AMPQ(Advanced Message Queuing Protocol)协议基础上的企业级消息系统， 是一个建立在Erlang OPT平台上的开源框架。所以我们在安装RabbitMQ之前，需要安装Erlang环境，再安装RabbitMQ服务。本文只对如何在Windows环境下安装RabbitMQ做了介绍，对于其作用和原理以及如何以JAVA代码实现，我会在后续的系列中逐一写出。同时后面也会对基于RabbitMQ的Spring Cloud组件消息驱动进行介绍，这里就暂且当做是万里长征第一步吧。 由于本文是基于windows安装的，对于想在linux上安装的同学，这里我也给一个靠谱的链接。  

[Linux安装RabbitMQ](https://mp.weixin.qq.com/s/j4Qj_qTbdOfpyryVzlipqw)

### 正文
#### 1.安装Erlang环境  

&emsp;&emsp;首先，我们需要去[Erlang官网下载](http://www.erlang.org/downloads)我们所需要的安装包，我下载的是目前最新RELEASE的21.0版本。这里有几个可选的下载链接，由于我的电脑是64位的，所以选择`OTP 21.0.1 Windows 64-bit Binary File (91707927)`进行下载。官网提供了网页下载和二进制下载，这里我直接下载的是二进制，需要下载其他安装包的，请自行下载即可。官网的下载界面如下图所示：
![](/images/rabbitmq-install/step.png)

&emsp;&emsp;然后双击下载好的安装包，会出现如图所示的界面，直接点击下一步即可，如果需要安装到其他的路径，也可以自行选择，因为我的C盘是安装了SSD的，所以我选择了默认的安装路径。
![](/images/rabbitmq-install/step1.png)  
![](/images/rabbitmq-install/step2.png)

&emsp;&emsp;安装完成之后，需要我们进行系统变量设置，右键我的电脑`属性->高级系统设置->环境变量`，在系统变量那一栏选择新建，然后输入变量名称和安装的路径，最后再将我们新建的变量添加到系统变量`path`中，添加的规则是`;%ERLANG_HOME%\bin`。注意`;`需要和签名的系统变量隔离开喔，配置过环境变量的同学应该都很了解了。
![](/images/rabbitmq-install/step3.png)
![](/images/rabbitmq-install/step4.png)  
然后重启电脑，`win+R`然后`cmd`进入dos界面，输入`erl`出现如下界面就是安装成功了，至此我们的erlang环境就算安装完成。
![](/images/rabbitmq-install/step5.png)

### 2.安装RabbitMQ  
&emsp;&emsp;同样的，我们需要到[RabbitMQ官网](http://www.rabbitmq.com/install-windows.html)去下载Windows版的RabbitMQ Server的安装包进行安装。这里注意一下，我们需要看一下安装的Erlang版本是否和RabbitMQ的版本能对应，[官网版本对](http://www.rabbitmq.com/which-erlang.html)对比也给了我们提示的信息。我安装的RabbitMQ版本是3.7.7，对应Eralng的版本是21.0.x。下载页面如图所示
![](/images/rabbitmq-install/step0.png)  
&emsp;&emsp;然后双击安装包，使用默认的安装选择，即可完成安装。如下图。选择next即可  ![](/images/rabbitmq-install/step6.png)  
接下来就是配置RabbitMQ的环境变量，和上面类似，不多赘述，直接上图吧。  
![](/images/rabbitmq-install/step00.png)

![](/images/rabbitmq-install/step01.png)


### 3.启用RabbitMQ插件  

&emsp;&emsp;启动插件是为了便于我们RabbitMQ进行管理，以及查看我们服务的运行情况，也方便我们查看交换机和队列中的相关信息。交换机和队列的概念在后续博客中解释。打开命令行界面，输入`rabbitmq-plugins list`查看当前所有的插件信息，如图所示
![](/images/rabbitmq-install/step8.png)  
然后输入`rabbitmq-plugins enable rabbitmq_management`命令启用插件，如图所示
![](/images/rabbitmq-install/step10.png)  
至此，我们的插件已经成功启动了，这个时候我们再打开浏览器，输入`localhost:15672`即可进入web界面查看我们的相关信息，初始用户名和密码都是`guest`，输入即可登录，到这里基本上我们的RabbitMQ服务已经安装完成了。
![](/images/rabbitmq-install/step11.png)  
&emsp;&emsp;但是这时候的web界面我们只有使用`localhost:15672`才能登录访问，如果要使用ip进行登录，比如我的ip地址是`192.168.0.29`,如果我使用`192.168.0.29:15672`进行访问会出现如下所示的界面  
![](/images/rabbitmq-install/step13.png)  
&emsp;&emsp;这种情况下有两种解决办法。第一，进入安装目录下，找到目录下的文件`ebin\rabbit.app`，打开并修改其中的`loopback_users`，如下如中所述，去掉`<<"guest">>`即可。  
![](/images/rabbitmq-install/step15.png)  
第二，添加一个用户，赋予其administrator权限，并设置其`Virtual host`为`/`设置其读写权限为`*`。可以进入命令行添加用户并设置，也可以进入web界面设置。web界面的设置如图所示
![](/images/rabbitmq-install/step02.png)  ![](/images/rabbitmq-install/step03.png)  
这样我们就可以通过ip远程访问我们的RabbitMQ服务了(当然，需要在同一网段中，或者你的ip地址已经映射到公网了)。为什么需要远程呢？当你的服务部署在服务器上的时候，你总不能还使用`localhost:15672`去访问吧。另外一点需要注意的是，RabbitMQ服务web界面的端口是`15672`，而我们客户端访问的ip地址实际上是`5672`，所以在开发的时候要注意一下。  

### 参考
- RabbitMQ官网：<http://www.rabbitmq.com/>  
- Erlang官网：<http://www.erlang.org>  
- localhost无法登陆:<https://www.cnblogs.com/lazyboy/p/3853371.html>
- Linux安装:<https://mp.weixin.qq.com/s/j4Qj_qTbdOfpyryVzlipqw>
