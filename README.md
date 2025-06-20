
# <center>Jekyll搭建个人博客教程<center>  

## 关于Github Pages
&emsp;&emsp; GIthub Pages则是github上的一项功能，可以放置网页文件到指定文件夹，然后给你一个专属域名用于展示一些项目，但现在大多用来开发制作个人博客网站。本教程就是基于此来搭建一个自己专属的博客。  

__本博客地址:__ [https://liangweilu.github.io](https://liangweilu.github.io/)

## 搭建步骤  

- Fork一个Jekyll主题的工程到你的Github账户下，你可以Fork[我的工程](https://github.com/byeluliangwei/byeluliangwei.github.io)，请务必在`__posts`目录下，删除我已有的文章。也可以自己去[JEkyll主题网站](http://jekyllthemes.org)找你喜欢的主题，然后放到你的Github仓库中。  

- 进入工程所在的`setting`中，修改工程名称为`{your github name}.github.io`。如下图所示：![](/images/readme/step1.png)

- 在`setting`页面中往下拉，找到`Github Pages`,任意选择一个主题，并找到自己的博客地址。如下图所示：![](/images/readme/step2.png)

- 在你fork的工程中，找到`__config`文件，并修改其中的个人信息为你自己的信息，然后提交保存。

- 至此一个简单的个人博客已经搭建好了，访问上面图片中标记的博客地址，就能看到你自己的博客主页了。  

- 最后就是发布文章，将你自己编写好的文章放到`__posts`目录下，提交到github之后，就完成了，进入你的博客，便能看见你所发布的文章。

## jekyll主题

[JEkyll主题网站](http://jekyllthemes.org) 使用方式说明
- 选择喜欢的主题
- `clone` 到本地
- 找到你的博客主目录，先copy备份，然后删除其中除了`.git、_posts`文件夹和`CNAME`文件之外的其他所有文件及文件夹
- 将你clone的主题的主目录下除`.git`文件夹外的其他文件及文件夹copy到你的博客主目录
- 在主目录下启动`jekyll server`
- 进入`localhost:4000`查看
- 如果失败也别担心，因为本地可能缺少某些插件，push变更到github，发布成功后去查看  

## 致谢

本博客是在[dawn1432](https://dawn1432.github.io) 博客模板上个性化修改而成，是一个简洁的博客模板。
