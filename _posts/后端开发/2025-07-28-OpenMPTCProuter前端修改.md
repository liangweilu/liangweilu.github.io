---
layout: post
title: OpenMPTCProuter前端页面修改部分介绍
date: 2025-07-28
tags: 其它
categories:
  - 后端开发
---

## 1. 前置
### 1.1 OpenWrt
OpenWrt 是一个针对嵌入式设备的 Linux 操作系统，常用于路由器，它有以下特点：
- 高度模块化
- 文件系统可写
- 提供强大的网络组件：提供了丰富的网络功能，适用于各种网络设备；
- 广泛的硬件支持：支持多种处理器架构；

> OpenWrt地址：https://github.com/openwrt/openwrt

这里说明以下什么是feeds，feeds是一些第三方开发的包，而不是OpenWrt自己开发的包。

### 1.2 LuCI
  LuCI就是一个feed，它是OpenWrt的统一配置接口，就是OpenWrt的一个网页版配置界面，允许用户通过浏览器对路由器进行配置和管理。
  LuCI 是 Lua 和 UCI 的结合体。Lua 是一种轻量级的脚本语言，适合嵌入在其他程序中。UCI（Unified Configuration Interface）是 OpenWrt 中用于实现所有系统配置的统一接口。
  
> LuCI地址：https://github.com/openwrt/luci  
> LuCI的API文档地址：https://openwrt.github.io/luci/api/index.html

## 2. OpenMPTCProuter
OpenMPTCProuter 是一个开源解决方案，基于 OpenWrt 使用 Multipath TCP (MPTCP) 实现多网络（如4G、ADSL、VDSL、光纤等）的聚合。它可以将多个互联网连接聚合并加密，通过任何VPS终端，使用户受益于安全性、可靠性、网络中立性以及专用的公共IP。
> openmptcprouter仓库地址：https://github.com/Ysurac/openmptcprouter

OpenMPTCProuter的web页面就是一个它自己的feeds仓库中，也是我们修改页面需要用到的，其仓库地址如下：
> openmptcprouter页面相关的仓库地址：https://github.com/Ysurac/openmptcprouter-feeds

### 2.1 安装

在修改页面之前我们需要先安装一个OpenMPTCProuter，以便后续查看修改是否生效。首先需要去官网下载对应的包，如我们是x86-64-bits的架构，就选择对应的镜像文件即可。
![img.png](/images/openwrt/img.png)
其实我们这里下载下来的就是一个linux系统镜像了，直接导入虚拟机管理软件（VMware或KVM）就可以用了。

安装成功后，我们直接访问虚拟机的IP地址即可进入web管理页面了，web页面的系统设置中可以选择语言和主题，这里我选择的是OpenMPTCProuter主题，语言支持中文选择。
![img_1.png](/images/openwrt/img_1.png)[图片]

### 2.2 修改web页面
#### 2.2.1 直接修改
完成上一节的安装后，进入系统，LuCI的相关代码都在路径`/usr/lib/lua/luci`目录下，而JavaScript，CSS等文件都在`/www`目录下。当我们需要修改或新
增页面的时候，可以直接在`/usr/lib/lua/luci`目录下进行修改和添加，其目录内容为大致如下：
![img_3.png](/images/openwrt/img_3.png)
可以看到LuCI是一个MVC结构的框架:

| 层级         | 位置                            | 功能                   |
|------------|-------------------------------|----------------------|
| Model      | /usr/lib/lua/luci/model/      | 提供配置接口，例如 uci 配置     |
| View       | /usr/lib/lua/luci/view/       | HTML 模板（使用 Lua 模板语法) |
| Controller | /usr/lib/lua/luci/controller/ | 注册菜单、处理请求            |

由于框架带有缓存，所以我们修改后需要执行下面的命令才会生效：
`/etc/init.d/uhttpd restart`

当然我们也可以在进行开发的时候，关掉框架的缓存：
```
uci set luci.ccache.enable=0
uci commit luci
```

以添加一个最简单的demo页面为例：
```
# 1.在controller目录下新增demo.lua文件
# vi /usr/lib/lua/luci/controller/demo.lua
module("luci.controller.demo", package.seeall)

function index()
entry({"admin", "system", "demo"}, template("demo/index"), _("Demo page"), 60)
end

# 2.在view目录下新增demo目录和demo/index.htm文件
mkdir /usr/lib/lua/luci/view/demo
vi /usr/lib/lua/luci/view/demo/index.htm
<%+header%>

    <h1>XXX Demo page</h1>
    <p>this is a demo page</p>
    
<%+footer%>
```
完成后重启uhttpd即可在web页面看到新增的内容。
![img_4.png](/images/openwrt/img_4.png)
这里对上述代码块中的entry方法进行以下解释，它的原本函数定义为：
```
-- 参数依次含义：
-- 注册路径、处理函数、显示名、排序等
function entry(path, target, title, order)
...
end

-- target可以是alias，template，function等，表示页面跳转和函数调用等
```

如需修改title和footer，修改如下文件：
```
/usr/lib/lua/luci/view/themes/openmptcprouter/header.htm
```

#### 2.2.2 修改feeds库
这种方式是直接fork官方的feeds仓库，然后在里面新建一个app，可以参考官方已有的luci-app-openmptcprouter进行开发，app名称必须以luci-app-开头。
openmptcprouter官方feed库中提供的。
![img_5.png](/images/openwrt/img_5.png)
其中htdocs目录对应编译后系统上的`/www`目录；luasrc目录对应`/usr/lib/lua/luci`；其它的配置等信息就对应`root`目录，和服务器一致。
当我们在feeds库中新增的app，并提交到我们fork的仓库后，如何让我们的修改呈现在编译后的页面中呢？我们需要去修改`openmptcprouter`官方仓库中的
`build.sh`文件，其中有一个feeds源地址，改为我们fork的仓库地址就好了。
![img_6.png](/images/openwrt/img_6.png)
这种方式不好的是要重新编译openmptcprouter，编译时间很长，很麻烦。因为OpenWrt是一个高度模块化的项目，肯定有更好的方式，只是我还没有找到。

> PS: LuCI官方的wiki里推荐的是第一种方式进行修改。

#### 2.2.3 编译app包
```
# 1.进入openmptcprouter主目录，将编写的app放入如下目录，如：luci-app-seanet
./feeds/openmptcprouter/luci-app-seanet

# 2.进入编译之后的目录
cd ~/openmptcprouter/x86_64/6.12/source
./script/feeds update -a

# 3.执行命令选中需要编译的app
make menuconfig

# 4.编译app
make package/feeds/openmptcprouter/luci-app-seanet/compile V=s

# 5.编译后的文件在如下位置，是一个apk文件
./bin/packages/x86_64/openmptcprouter/
```

#### 2.2.4 国际化
修改po文件为lmo文件参考：https://github.com/openwrt/luci/wiki/i18n

## 3. 相关文档
1. [编译openMPTCProuter](https://docs.qq.com/doc/DZkRkdUJVbmtMTHV6)
2. [我的一些成功修改案例](https://fuys3rixxm.feishu.cn/docx/N2vhdfcYQozw13xNIsScnxk9nwc)
