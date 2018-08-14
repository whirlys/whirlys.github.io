---
title: 使用hexo+github pages搭建博客
categories:
  - 后端
tags:
  - 博客
date: 2018-05-23 11:35:46
---



## 为什么写博客


就如我在博客主页上所说，主要有三点：

1. 记录与分享
2. 锤炼技术，提高写作能力和表达能力
3. 树立个人品牌，提高影响力

而在此博客之前，我在CSDN上写过一些博客，截止于2018年5月23日，个人资料如下：

{% qnimg 201805/20180523_232554.png  title:小旋锋的csdn个人资料 alt:小旋锋的csdn个人资料 %}

当我在CSDN上写博客的时候，几乎每天都会去看看阅读量增加了多少，排名增加了多少，又增加了几个粉丝或者新评论，每每都会带给我兴奋感，让我感到写博客其实是一件很有意义的事情，并且反过来推动我学习和记录，写更多的博客。

而为什么现在要重新整一个博客呢？主要是因为之前CSDN的博客更多的是转载和低质量的，而博主即将毕业，正走在程序员的职业道路上，需要树立个人品牌，写博客是目前对我比较合适且能做到的方式。

而独立博客自由度更高，第三方博客平台推广则更快，所以最终决定采用独立博客首发，第三方平台分发引流的模式。

我的第三方平台账户：

* [小旋锋的简书](https://www.jianshu.com/u/ae269fd3620a)
* [小旋锋的csdn博客](https://blog.csdn.net/wwwdc1012)
* [小旋锋的知乎](https://www.zhihu.com/people/whirlys/activities)
* [小旋锋的微信公众号](http://image.laijianfeng.org/static/images/201805/20180523_230522.jpg)

## Hexo主题选择

[Hexo](https://hexo.io/zh-cn/docs/index.html) 是一个快速、简洁且高效的博客框架，可托管于github pages，可免去维护服务器的麻烦，博主们可更专注于内容的创作，并且Hexo主题众多，总有一款适合你。

我对主题的要求主要有：
1. 不要太大众
2. 大气美观
3. 功能齐全

经过了几天的搜索之后，筛选了几个比较满意的Hexo主题如下：

1. [Hueman](http://blog.zhangruipeng.me/hexo-theme-hueman/about/index.html)
2. [jacman](http://jacman.wuchong.me/2014/11/20/how-to-use-jacman/)
3. [大道至简](https://www.haomwei.com/)
4. [Loo's Blog](http://threehao.com/)
5. [3-hexo](http://yelog.org/2017/03/23/3-hexo-instruction/)

最终选择了 [3-hexo](http://yelog.org/2017/03/23/3-hexo-instruction/) 这款主题，当然还有很多不错的主题。


## 搭建步骤

### 1. 根据 [Hexo官网](https://hexo.io/zh-cn/docs/index.html) 步骤安装 git，node.js

### 2. 安装Hexo
``` javascript
npm install -g hexo-cli
```
安装 Hexo 完成后，新建一个博客的主目录，然后执行以下命令：
``` shell
hexo init <folder>
cd <folder>
npm install
```
新建完成之后该目录的目录结构如下:

.

├── _config.yml		# 网站的 配置 信息

├── package.json		# 应用程序的信息

├── scaffolds			# 模板文件夹

├── source			# 博文源文件目录

|   ├── _drafts		# 草稿文件夹

|   └── _posts			# 博文文件夹

└── themes			# 主题文件夹



再执行以下命令，访问 [http://localhost:4000](http://localhost:4000) 即可快速体验Hexo
``` shell
hexo g
hexo s
```
{% qnimg 201805/20180524_003246.png title: Hexo初体验 alt: Hexo初体验 %}

### 3. 根据 [Hexo文档](https://hexo.io/zh-cn/docs/configuration.html) 对网站做一些简单的配置，然后修改主题为 [3-hexo](https://github.com/yelog/hexo-theme-3-hexo)

安装
``` shell
git clone https://github.com/yelog/hexo-theme-3-hexo.git themes/3-hexo
```
修改hexo根目录的_config.yml中的theme参数
``` javascript
theme: 3-hexo
```
然后执行 hexo clean & hexo g & hexo s 即可看到效果

更多的主题配置可见 [3-hexo使用说明](http://yelog.org/2017/03/23/3-hexo-instruction/)

### 4. 配置 github pages

到github上创建一个新的空仓库，名字格式为 账户名.github.io，譬如我的github账户名是 [whirlys](https://github.com/whirlys)，所以我的github pages 仓库的名字应为 whirlys.github.io

安装插件
``` shell
npm install hexo-deployer-git --save
```

然后配置 Hexo根目录的 _config.yml，xxx为你的用户名，注意还需要加入你的 github 用户名和密码，不然后面推送失败（但是上传代码时注意防止密码泄露）
``` shell
deploy:
  type: git
  repo: https://[github用户名]:[github密码]@github.com/xxx/xxx.github.io.git
  branch: master
```

如果你是第一次配置 github 远程仓库，你还须将你电脑的ssh key 配置到 github 上，具体可参考 [git远程仓库](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/001374385852170d9c7adf13c30429b9660d0eb689dd43a000)

推送Hexo到github
``` shell
hexo deploy
```

访问 xxx.github.io 即可看到你的 github pages 博客了


### 5. 绑定私有域名

我的域名为 laijianfeng.org，是一年前买 腾讯云1元学生主机 时送的，当然可以选择其他域名提供商

在 hexo source 目录下新建一个 CNAME 文件（没有后缀名），在文件里填入你的域名，然后 hexo d 推送到github

登录域名提供商网站，进入域名解析页面，分别添加两条记录

| 主机记录 | 记录类型  | 线路类型 | 记录值               |
| ---- | ----- | ---- | ----------------- |
| @    | CNAME | 默认   | xxx.github.io     |
| www  | CNAME | 默认   | www.xxx.github.io |

等待十分钟之后，访问你的域名即可跳转到你的博客



### 6. 其他的配置

* 接入评论，3-hexo主题中已经集成了多种评论，我选择了gitment，具体的配置参考 [完美替代多说-gitment](http://yelog.org/2017/06/26/gitment/)，如果gitment遇到问题，譬如报Error：validation failed异常，可参考 [添加Gitment评论系统踩过的坑](http://xichen.pub/2018/01/31/2018-01-31-gitment/) 以及 [gitment issue](https://github.com/imsun/gitment/issues)上的解决方法
* 使用七牛云图床，参考 [使用七牛为Hexo存储图片](http://skyhacks.org/2017/08/02/UseQiniudnToStorePic/) 和 [Hexo七牛同步插件](https://github.com/gyk001/hexo-qiniu-sync)
* 代码高亮，字数统计，参考 [Hexo主题3-hexo](http://yelog.org/2017/03/07/3-hexo/)
