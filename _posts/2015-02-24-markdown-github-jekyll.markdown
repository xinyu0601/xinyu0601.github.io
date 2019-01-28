---
title: 如何用Github搭建个人博客
layout: post
date: '2017-04-08 22:48:00'
image: "/assets/images/markdown.jpg"
tag:
- markdown
- github
- jekyll
category: blog
author: stanley huang
description: github博客搭建详细教程
---

## jekyll+github搭建个人博客
经过一天多的折腾，终于算是搭建好了自己的个人博客，看到有些社区评论说：在windows下用jekyll搭建静态博客，简直就自讨苦吃，但是都到一半了，有什么办法呢，只好坚持搭完咯~~
搭建github博客可以用hexo，也可以用jekyll，我用的是后者，hexo大家可以试试，在这里推荐一个用hexo搭建的教程：http://gaoxianglyx.top
下面就是我的搭建步骤了，希望可以帮到还在折腾的你：

- 下载ruby
- 安装jekyll
- 安装bundler
- 建立你的第一个静态博客
- 开启jekyll服务器
- 写一篇自己博文
- 用github pages 展示你的博客
	- 创建个人仓库
	- 克隆仓库到一个指定的文件目录
	- 把你本地的第一个博客文件里的所以文件复制到这个克隆下来的文件
	- 把这些文件push到远程仓库
	- 查看你的博客网站
	 接下来我们的操作都是在cmd命令行中进行的
	 
#### 下载ruby
什么是ruby：Ruby，一种简单快捷的面向对象（面向对象程序设计）脚本语言，安装Jekyll需要电脑上安装Ruby，以下是安装步骤：
- window系统下，可以使用rails install来安装ruby环境，下载地址为：http://rubyinstaller.org/downloads/ ，建议下载2.3以上的新版。
- 下载 RailsInstaller 之后，双击 railsinstaller-3.2.0 文件，启动 Ruby 安装向导点击next，向导完成安装，记得勾选 Add Ruby executables to your PATH，直到 Ruby 安装程序完成 Ruby 安装为止
- 安装后，在cmd中输入ruby -v和gem -v来看看是否安装成功，看到版本号就说明成功。
	 
#### 下载jekyll
jekyll：jekyll是一个简单的免费的Blog生成工具，类似WordPress。但是和WordPress又有很大的不同，原因是jekyll只是一个生成静态网页的工具，不需要数据库支持。但是可以配合第三方服务,例如Disqus。最关键的是jekyll可以免费部署在Github上，而且可以绑定自己的域名。
 ```
gem install jekyll
 ```
#### 下载bundle
在命令行输入
```
gem install bundler
```
bundler：就是一个打包机，他会连接rubygems.org（或者其他你声明的源），然后列出所有你指定的符合你需要的 gem。因为所有你在Gemfile里的依赖有它们自己的依赖，所以基于上面的Gemfile运行bundle install会安装相当多的的 gem。（我也不太了解，自己可以百度）

#### 开启jekyll内置服务器
实现转入blog的目录，输入：
```
jekyll serve  //开启服务器，可以按ctrl+c停止
```

>  Jekyll服务器默认端口是4000，所以打开浏览器输入：`http://localhost:4000` 就可以看到生成的博客页面。如下：

![](https://images2015.cnblogs.com/blog/1019973/201701/1019973-20170114223440525-1524406009.png)