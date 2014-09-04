---
layout: post
title: "Hello Octopress"
date: 2014-09-05 01:35:36 +0800
comments: true
categories: octopress github blog 
---


想要有自己的博客这个想法很久很久了...行动上却很诚实，鄙视一下自己！

---

`2013-10-31` 号申请的域名 [lseek.me][1] 一直闲置到现在，着实是暴殄天物！想当初好不容易才找到这个为数甚少的未被注册的 C 标准库函数名 `lseek`， 而且 `lseek.me` 刚好带有点 `寻求自我` 的意味，那时还真有点小激动呢～～～导致手一抖 280 块大洋没了 >_<

前段时间无意知道 `github` + `octopress` 搭建 blog 这玩意，而且还是免费的，`github` 为你管理服务器，并提供无限流量，世界各地都有理想的访问速度，这还得了，叼炸天啊，不过过瘾怎么行！

于是一通搜索，看了诸多教程，基本上都是大同小异，照虎画猫很快把 blog 搭建了起来。本文简单记录一下整个过程：


----------


**准备工作**

> * 安装 [Git][2]
> * 安装 Ruby 1.9.3 或 更高版本 ( 通过 [rbenv][3] 或 [RVM][4] )

**搭建 Octopress 环境**

(1) 安装 Octopress
```
$ git clone git://github.com/imathis/octopress.git octopress
$ cd octopress
```
(2) 安装相关依赖
```
$ gem install bundler
$ rbenv rehash
$ bundle install
```
(3) 安装默认的 Octopress 主题
```
$ rake install
```

参考：[Initial Setup][5]

**配置 Octopress**

Octopress 的作者已经尽量让配置简化了。大多数情况下只需要配置 `_config.yml` 和 `Rakefile` 文件即可。其中 `Rakefile` 是跟博客部署相关，一般情况下并不需要修改这个文件，除非使用了 `rsync`。

这一块我就修改了一下 `_config.yml` 文件，把 `url` 和 `author` 加进去而已。其中 `url` 是必填项，内容是我在 github 上创建的一个仓库地址 http://ronnietsang.github.io

接着删除了里面的 twitter 相关的信息，否则据说是由于 GFW 的原因，将会造成页面加载很慢。

然后修改定制文件 `./.themes/classic/source/_includes/custom/head.html` 把 google 的自定义字体去掉，原因同上。

参考：[Basic Configuration][6]

**创建第一篇博客**

没错，现在就已经可以开始写博客了！

(1) 创建博文

Octopress 为我们提供了一些 task 来创建博文和页面。博文必须存储在 `source/_posts` 目录下，并且需要按照 Jekyll 的命名规范对文章进行命名：`YYYY-MM-DD-post-title.markdown`。文章的名字会被当做 `url` 的一部分，而其中的日期用于对博文的区分和排序。
```
$ rake new_post["Hello Octopress"]
```
其中 `Hello Octopress` 为博文的文件名，创建出来的文件默认是 `markdown` 格式。上面的命令会创建出这样一个文件：`./source/_posts/2014-09-04-hello-octopress.markdown`。打开这个文件，可以看到里面有如下一些内容了 ( 告诉Jekyll博客引擎如何处理博文和页面 )：
```
---
layout: post
title: "Hello Octopress"
date: 2014-09-04 19:13:48 +0800
comments: true
categories:
---
```

(2) 编辑博文

接下来使用你喜欢的 markdown 编辑器编写博文，完成后贴到上面的文件中就可以了。

> 我用的是 `作业部落` 出品的 [Cmd Markdown][7] 在线编辑阅读器，基本上兼容 github 的 markdown 文法，挺好用的，推荐！

(3) 本地预览

执行：
```
$ rake preview
```
然后就能在浏览器中进行本地预览访问了： `http://127.0.0.1:4000/` 或 `http://localhost:4000/`，效果跟仓库中的一样。

(4) 完整过程

下面是创建并部署博文的一个完整过程：
```
$ rake new_post["Hello Octopress"]
$ rake generate
$ git add .
$ git commit -m 'My first blog -- Hello Octopress'
$ git push origin source
$ rake deploy
```
先 mark ，现在还无法部署博文，要等后面部署博客完毕再参考这个过程来部署博文。

参考：[Blogging Basics][8]


----------


**部署到 Github Pages**

官方推荐了3种部署方式：
> * `GitHub Pages`： 部署允许自定义域名，免费，好处是多人开发更方面，坏处是文件随时可以被任何人拉下来
> * `Heroku`：部署允许自定义域名，免费，并且是私有的
> * `Rsync`：建议用来部署有自己服务器的个人博客


当然了，我选择的是 **[GitHub Pages][9]** 。

(1) 创建 github 仓库 [ronnietsang.github.io][10] ( 网上很多年代久远的教程都是说 com 后缀，现在改成 io 了 )
> * 保持新建仓库为空的，不要勾选创建 `README` 文件等选项( 这可能导致后面部署 `rake deploy` 失败，引出一系列问题 )
> * 一般来说，我们希望在将博客的源码放到 `source` 分支下，并把生成的内容提交到 `master` 分支

(2) 配置仓库

一键配置：只需要利用 octopress 的一个配置 `rake` 任务就可以自动配置上面创建的仓库，超级方便：
```
$ rake setup_github_pages
```

这个命令会要求你输入刚刚创建的仓库的 `url` ( `ssh` or `https` url )，拷贝过来，回车，接下来它就干了这么几件事：
> * 将指向远程分支 imathis/octopress 的指针 origin 改名为 octopress
> * 添加刚才创建的仓库为默认的远程分支
> * 将活跃分支从 master 转到 source
> * 根据 github仓库 配置博客的 url
> * 创建一个_deploy目录，用来存放部署到 master 分支的内容

(3) 部署仓库

命令如下：
```
$ rake generate
$ rake deploy
```
干了这么几件事：
> * 生成博客文件，并拷贝至 _deploy 目录下
> * 将这些内容添加到 git 中，并 commit 和 push 到仓库的 master 分支

不过博客的 `source` 需要单独提交，执行如下命令就可以将 `source` 提交到仓库的 `source` 分支下。
```
$ git add .
$ git commit -m 'Initial source commit'
$ git push origin source
```

至此，博客已经完成基本的部署。但是当我满心期待地使用前面配置的 url 来访问时，直接来了一个冷冰冰的 `404`！好吧，莫慌，原来是有延迟，稍等几分钟(我等了半个钟...)，终于可以正常访问了！


参考：[Deploying to Github Pages][12]

----------


**更多功能**

在搭建博客的时候，我们可能会对博客做一些配置，例如添加评论、域名解析、分享等。先 mark，以后再慢慢了解吧。

参考：
> * [Sharing Code Snippets][13]
> * [Blogging With Plugins][14]
> * [Theming & Customization][15]
> * [Updating Octopress][16]


----------

**承诺**

下个死命令：坚持下去，中秋回来以后，每周至少更新 `2` 篇技术博文！


  [1]: http://lseek.me
  [2]: http://git-scm.com/
  [3]: http://octopress.org/docs/setup/rbenv
  [4]: http://octopress.org/docs/setup/rvm
  [5]: http://octopress.org/docs/setup/
  [6]: http://octopress.org/docs/configuring/
  [7]: https://zybuluo.com/mdeditor
  [8]: http://octopress.org/docs/blogging
  [9]: https://pages.github.com/
  [10]: https://github.com/RonnieTsang/ronnietsang.github.io
  [11]: http://stackoverflow.com/questions/17609453/rake-gen-deploy-rejected-in-octopress
  [12]: http://octopress.org/docs/deploying/github/
  [13]: http://octopress.org/docs/blogging/code
  [14]: http://octopress.org/docs/blogging/plugins
  [15]: http://octopress.org/docs/theme
  [16]: http://octopress.org/docs/updating
