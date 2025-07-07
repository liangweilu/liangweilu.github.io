
# <center>Jekyll搭建个人博客教程<center>  

## 关于Github Pages
&emsp;&emsp; GIthub Pages则是github上的一项功能，可以放置网页文件到指定文件夹，然后给你一个专属域名用于展示一些项目，但现在大多用来开发制作个人博客网站。本教程就是基于此来搭建一个自己专属的博客。  

__本博客地址:__ [https://liangweilu.github.io](https://liangweilu.github.io/)

## 原型

本博客基于[jekyll-theme-yat](https://github.com/jeffreytse/jekyll-theme-yat)模板进行修改，主要包含如下内容：

- 修改博客字体为`Roboto, SourceHanSans, sans-serif, !default;`
- 修改字体大小和间距
- 增加基于[utterances](https://utteranc.es/)的评论插件
- 增加[highlight.js](https://github.com/highlightjs/highlight.js/)代码块高亮显示主题

## 搭建步骤  

- Fork或下载本工程到你的Github账户下，完成后务必在`__posts`目录下，删除原已有的文章。也可以自己去[JEkyll主题网站](http://jekyllthemes.org)找你喜欢的主题，然后放到你的Github仓库中。  

- 进入工程所在的`setting`中，修改工程名称为`{your github name}.github.io`。如下图所示：![](/images/readme/step1.png)

- 在`setting`页面中往下拉，找到`Github Pages`,任意选择一个主题，并找到自己的博客地址。如下图所示：![](/images/readme/step2.png)

- 在你fork的工程中，找到`__config`文件，并修改其中的个人信息为你自己的信息，然后提交保存。

- 至此一个简单的个人博客已经搭建好了，访问上面图片中标记的博客地址，就能看到你自己的博客主页了。  

- 最后就是发布文章，将你自己编写好的文章放到`__posts`目录下，提交到github之后，就完成了，进入你的博客，便能看见你所发布的文章。

## 开启评论
[jekyll-theme-yat](https://github.com/jeffreytse/jekyll-theme-yat) 支持[Disqus](https://disqus.com/)，[Gitment](https://github.com/imsun/gitment)，
[utterances](https://utteranc.es/)三种评论，打开`_config.yml`文件，取消对应的注释，并填入你自己的信息即可。
