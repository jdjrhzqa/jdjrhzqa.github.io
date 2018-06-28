---
title: hexo+github博客搭建
categories: 工具学习
author: 郑琳
tags: hexo
date: 2018-05-28
---

#### hexo 简介
Hexo 是一个快速、简洁且高效的博客框架。Hexo 使用 Markdown（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。

[hexo更多信息](https://hexo.io/docs/deployment.html)

#### 博客搭建
博客采用 hexo+github pages方式搭建，选择简洁的NexT主题，因网络上相关资料较多，本文只记录步骤，不做详细介绍。
##### 框架搭建
1. 在github上新建一个仓库，仓库名必须为 “账户名.github.io”
2. 本地安装nodejs 环境，新建文件夹 "账户名.github.io"，安装hexo环境
```
npm install hexo
npm hexo init
npm install 
npm install hexo-deployer-git --save 
```
3. 修改站点配置文件（根目录下_config.yml）,关联本地hexo和git仓库
```
deploy:
  type: git
  repo: git@github.com:账户名/账户名.github.io.git
  branch: master
  message: "hexo deploy"
```
  当然，配置这个前提是，github上已经配置过[ssh key 认证](https://help.github.com/articles/connecting-to-github-with-ssh/)。

4. 编辑站点配置文件_config.yml，添加个人设置
```
title: 京东金融杭州QA
subtitle:
description: 打造业内高精尖白盒测试团队！
keywords:
author: 白盒测试组
language:
timezone:
```
5. hexo 主题选择
hexo 提供了很多博客主题，在此选择简约的hexo主题。
编辑 hexo 站点配置文件 _config.yml
```
theme: next
```
  选择完成后，需要在一级目录themes中添加NexT的源码,具体方法为：
  ```
  cd 账户名.github.io
  git clone https://github.com/iissnan/hexo-theme-next themes/next
  ```

##### 属性设置
与博客主题相关的属性设置都在./themes/next/_config.yml文件中，包含博客样式设置，功能主题设置及关联第三方服务的进阶设置。
1. 样式设置，包含Scheme布局设置、头像设置、语言设置、菜单设置、侧栏设置等，具体详见[next官网](http://theme-next.iissnan.com/getting-started.html)
2. 功能主题设置：设置标签和分类页面。如果菜单设置中添加了tags和categories两个标签，则需要对应生成相应的页面
```
cd “账户名.github.io”
hexo new page tags
hexo new page categories
```
3. 进阶设置
- 博客阅读量统计功能设置
统计工具选择 leanCloud，在[官网](https://leancloud.cn/docs/storage_overview.html)上注册用户，新建一个Counter应用,记录应用对应的id 和key，修改主题配置文件
```
leancloud_visitors:
  enable: true
  app_id: 应用对应的id
  app_key: 应用对应的key
```
- 评论功能设置
曾经红极一时的友言、多说、网易云跟帖相继都关停服务了，而disqus 和facebook 的评论服务器在国外，由于网络限制，用起来也多有不便。调研了一些评论系统，发现leanCloud旗下的评论系统valine，宣称永不停服，相对还是很靠谱的。重要的是NexT 竟然也很好的支持了，意出望外~ 具体配置如下：
```
valine:
  enable: true
  appid:  同leanCloud
  appkey:  同leanCloud
```

- pv uv 统计功能设置
pv 和uv 统计功能使用busuanzi 来进行统计，最新版本的next对busuanzi的支持很好友，已经不需要像老版本那样通过修改post.ejs文件，关联busuanzi的js脚本来实现了，具体设置：
```
busuanzi_count:
  # count values only if the other configs are false
  enable: true
  # custom uv span for the whole site
  site_uv: true
  site_uv_header: 本站访客数
  site_uv_footer: 人次
  # custom pv span for the whole site
  site_pv: true
  site_pv_header: 本站总访问量
  site_pv_footer: 次
  # custom pv span for one page only
  #page_pv: true
  #page_pv_header: 本文总阅读量
  #page_pv_footer: 次

```
  因为已经有了learnCloud做阅读量统计，这里就不启用busuanzi阅读量的统计了。

- 分享功能设置
分享神器jiathis也停服了，暂时使用百度分享来实现博客内容分享功能。
```
baidushare:
 type: button
 baidushare: true
```

- blog author设置
nexT 没有展示博客作者的相关设置，查看了好多文章，没有找到合适的方式。但该功能对于团队博客来说是很重要的，所以去查看了相关源码，找到了markdown渲染的关键代码文件post.swig。对该文件的页面处理逻辑做一些修改，增加了author 标签，目前支持显示blog 的author，但与其他标签相比，缺少好看的样式支持，希望懂前端的同学可以加以补充完善。



#### 博客书写
##### 本地环境安装
1. 选择合适的文件夹，下载博客源码
前提条件: github账号关联jdjrhzqa Organizations，且github配置过[ssh key](https://help.github.com/articles/connecting-to-github-with-ssh/)
```
git clone git@github.com:jdjrhzqa/jdjrhzqa.github.io.git
git checkout hexo
```
2. hexo 环境安装
先安装nodejs，后执行npm install 安装相关依赖
```
npm install -g hexo-cli
npm install
npm install hexo-deployer-git
```
	如果npm速度较慢，建议切换至淘宝的npm 镜像cnpm来安装，具体方法为：
	```
	npm install -g cnpm --registry=https://registry.npm.taobao.org
	cnpm install -g hexo-cli
	cmpm install
	cnpm install hexo-deployer-git
	```
##### 博客书写与代码提交
环境安装完成之后，可以愉快写博客啦，博客书写发布需要掌握hexo的相关命令，具体的操作流程如下：
1. 新建一个提交页面
```
hexo new "my new post"
```
2. 在./source/_posts 目录下找到新建的md 文件，对文件进行编辑操作

3. md文件中需添加title、categories、tags及author字段，格式可参见已提交博客md文件
4. md文件编辑完成后，可以在本地运行博客服务，查看内容
```
hexo clean 
hexo generate（or hexo g）
hexo server （or hexo s）
```
  其中，hexo clean命令可清理已生成的静态页面，hexo generate 命令将markdown文件渲染成静态页面，可简写为hexo g。而hexo server 命令则在本地启动服务器，用于预览主题，可简写为hexo s。如果想直接预览主题，也可跳过前两个命令，直接执行hexo server。

5. 本地博客展示正常，将代码提交至远端仓库，而后将博客发布到远端服务器上
```
git commit -m "xxxx"
git push origin 
hexo deploy （or hexo d）
```
##### 书写博客注意事项
- 如果博客内容需要修改，建议直接在原文件中编辑，不要删除替换原有文件。因为生成时间是博客的标识之一，随意修改会影响到该篇博客的统计数据。
- 仓库里面hexo分支为源码分支，master分支为渲染后的静态页面内容，从github上拉取代码后，注意要切换到hexo分支，代码提交和hexo deploy都在此分支执行。
- 每次hexo deploy发布博客时，仓库中master分支的内容都会被完全覆盖掉，故如无特殊情况，建议大家不要随意修改或者删除hexo分支中已有的博客内容。
