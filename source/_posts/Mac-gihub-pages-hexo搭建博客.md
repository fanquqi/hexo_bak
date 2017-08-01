---
title: Mac+gihub pages+hexo搭建博客
date: 2017-06-03 22:48:39
tags: hexo 
categories: hexo 
---


# Mac下使用github+hexo建博客

 
> 很早之前就想建个博客，纠结于怎么建，有的同学使用LNMP，使用wordpress,Django，首先得有个服务器，还是不够方便。近期使用github托管一些平常使用的代码或者设置，而且平常习惯使用markdown写笔记，所以就想直接用github_pages+hexo建个博客。

**下边是过程。**
 
 

* * *

## 安装过程

**首先**需要在Mac本地把git装好，公钥传到github，然后装好node，npm

```
- brew install nodejs
- brew install npm 
- npm install -g hexo-cli
- (看下安装目录，可以做个软链到/usr/bin/)
- ln -s /usr/local/lib/node_modules/hexo-cli/bin/hexo /usr/bin/hexo
```

**然后**再自己Mac上建一个工作取名**Blog**，在github上面新建个工程参考[github-pages官网](https://pages.github.com/)必须以 **git用户名.github.io**这种格式命名工程名。之后再**Blog** 目录下clone该工程。

```
- mkdir Blog
- cd Blog && git clone https://github.com/fanquqi/fanquqi.github.io.git
```

## 在本地建立站点

安装 Hexo 完成后，请执行下列命令，Hexo 将会在指定文件夹中新建所需要的文件。
[Hexo相关操作参考](https://hexo.io/zh-cn/docs)

```
$ hexo init <folder>
$ cd <folder>
$ npm install
```

所以以上的`<folder>`我们 用Blog代替
操作可以改为这样,因为上文中我们已经手动建了一个`Blog`文件夹

```
cd Blog 
hexo init 
npm install
```

新建完成后我们的文件夹目录如下：

```
.
├── _config.yml
├── package.json
├── scaffolds
├── scripts
├── source
|   ├── _drafts
|   └── _posts
└── themes

```

之后运行 hexo start 可以在http://localhost:4000 看到自己的页面

**关联github**
>设置好本地之后,需要关联github，以后就可以在本地写作，传到github上了


修改_config.yml,在最后加上(注意不能有tab，':'后必须有个空格)

```
deploy:
  type: git
  repository: https://github.com/fanquqi/fanquqi.github.io.git
  branch: master
```

在blog文件夹目录下执行生成静态页面命令：

```
$ hexo generate     或者：hexo g
```
如果出现如下报错

```
ERROR Local hexo not found in ~/blog
ERROR Try runing: 'npm install hexo --save'
```

执行
```
npm install hexo --save
```
如果没有出现略过此步

然后deploy 一下 同步到远程github
```
hexo deploy
```
如果出现一下报错 没有找到git

``` 
➜  Blog hexo deploy
ERROR Deployer not found: git
```

安装hexo-deploy-git

```
➜  Blog npm install hexo-deployer-git --save
hexo-site@0.0.0 /Users/fanquanqing/workspace/Blog
└─┬ hexo-deployer-git@0.3.0
  └── moment@2.18.1
```

之后再次执行hexo generate和hexo deploy命令
如果没有报错，你就可以在https://fanquqi.github.io  看到之前自己4000端口相同的内容了。好棒哦。。。

------

##安装theme

>主题可以到hexo官网下载 [https://hexo.io/themes/ ](https://hexo.io/themes/) 
这里以Next为例。

```
cd Blog && git clone https://github.com/iissnan/hexo-theme-next themes/next
#替换Blog/_config.yml themes: landscape 为 themes: next
hexo clean //清除缓存
hexo generata
hexo deploy
hexo start
```
配置查看[http://theme-next.iissnan.com/getting-started.html ](http://theme-next.iissnan.com/getting-started.html)
**搜索功能添加**

```
# 安装hexo-generator-searchdb(给博客添加搜索功能)
➜  Blog npm install hexo-generator-searchdb --save
hexo-site@0.0.0 /Users/fanquanqing/workspace/Blog
└─┬ hexo-generator-searchdb@1.0.7
  └── striptags@3.0.1
#修改_config.yml,添加
search:
  path: search.xml
  field: post
  format: html
  limit: 10000
```

hexo start发现问题
```
➜  Blog hexo s
ERROR Plugin load failed: hexo-generator-searchdb
```
查看插件hexo-generator-searchdb 中的Redmine  发现node 需要降级到4.2.2以下

```
npm install -g n
n 4.2.2
```


## menu 添加

> 默认的有：首页，归档，分类，标签，关于, 但是next主题默认只显示了首页 归档和标签。
> 以** categories** 添加为例(tags 也是如此)

1.新建categories page 
```
hexo new page "categories"
```
会在source下面生成一个categories目录。

2.修改index.md

打开categories文件夹中的index.md页面，在头部添加 – type: “categories” （为了点击的时候链接生效）
修改完之后
```
---
title: 分类
date: 2017-06-03 22:17:53
type: "categories"
---
```

3.在hexo > theme > next > _config.yml 中修改menu下的categorues,去掉前面井号注释。



## 文章发表
发表步骤如下
- 新建文章
    + hexo new urlooker使用
- 编辑文章
    + 直接vim 或者用自己的markdown编辑器编辑，这里需要提醒下添加**tags**和 **categories** 使用
    + hexo中有Front-matter这个概念，是文件最上方以 — 分隔的区域，用于指定个别文件的变量。
```
---
title: urlooker使用
date: 2017-06-02 18:57:50
tags: open-falcon
categoris: 监控
---
```

举例如上

## 添加评论功能

> 多说在2017.6.1停止服务了，比较尴尬，但是新的next 集成了友言的评论功能，设置非常方便

- 注册[友言官网](http://www.uyan.cc/)，在管理后台记下UID
- 添加UID到_config.yml 格式为

```
youyan_uid: 2323232
```

## 静态文件处理
> 好像github-pages 最大存储为300M 所以不能尽情的存储，最好吧静态图片放到外面，我直接放到了七牛云存储上，每个图片生成一个外链，很方便。


重新部署下hexo即可。




好了，现在就可以随时markdown了。


参考：
- https://madongqiang2201.github.io/
- http://theme-next.iissnan.com/getting-started.html
- http://jeasonstudio.github.io/2016/05/26/Mac%E4%B8%8A%E6%90%AD%E5%BB%BA%E5%9F%BA%E4%BA%8EGitHub-Page%E7%9A%84Hexo%E5%8D%9A%E5%AE%A2/

