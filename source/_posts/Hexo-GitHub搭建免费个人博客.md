---
title: Hexo+GitHub搭建免费个人博客
categories: Hexo博客
date: 2019-10-09 16:42:07
cover: /img/post_cover/Hexo+GitHub搭建免费个人博客.jpg
tags:
    - GitHub
    - Hexo
    - 博客
---
<head> 
    <script defer src="https://use.fontawesome.com/releases/v5.0.13/js/all.js"></script> 
    <script defer src="https://use.fontawesome.com/releases/v5.0.13/js/v4-shims.js"></script> 
</head> 
<link rel="stylesheet" href="https://use.fontawesome.com/releases/v5.11.2/css/all.css">  

# Hexo+GitHub免费搭建个人博客  

> 作者：大大蜡笔小小新    

## 1. <i class="fas fa-blog"></i> Hexo框架的介绍   

`Hexo`的官方介绍：`A fast,simple&powerful blog framework`。  

`Hexo`框架具有以下特点：  

- 快速搭建
- 支持`Markdown`语法  
- 支持一键部署  
- 丰富的主题和插件  

## 2. <i class="fas fa-home"></i> 环境准备  

### 2.1申请一个`GitHub`账户    

  [`GitHub`账户申请](https://github.com/)

  [`GitHub`疑问解答](https://help.github.com/cn)

### 2.2搭建`Hexo`需要的环境  

  如何安装`Hexo`及安装`Hexo`需要的环境，可以参考`Hexo`官方文档  

  [Hexo官方文档](https://hexo.io/docs/index.html)  
  
### 2.3环境验证  

`Node.js`和`Git`安装是否成功?  

```cmd
Microsoft Windows [版本 10.0.17134.590]  
(c) 2018 Microsoft Corporation。保留所有权利。    
C:\Users\lenovopc>node -v  
v8.11.2    
C:\Users\lenovopc>npm -v  
5.6.0

E:\Git>git --version  
git version 2.16.2.windows.1  
E:\Git>
```

`Node.js`和`npm`的关系  
> <i class="fas fa-quote-left fa-3x fa-pull-left"></i>`Node.js`是`javascript`的一种运行环境，是对`Google V8`引擎进行的封装。是一个服务器端的`javascript`的解释器。  
 包含关系:`Node.js`中含有`npm`，比如说你安装好`Node.js`，你打开`cmd`输入`npm -v`会发现npm的版本号，说明`npm`已经安装好。  
引用大神的总结:
其实`npm`是`Node.js`的包管理器（`package manager`）。我们在`Node.js`上开发时，会用到很多别人已经写好的`javascript`代码，
如果每当我们需要别人的代码时，都根据名字搜索一下，下载源码，解压，再使用，会非常麻烦。于是就出现了包管理器`npm`。
大家把自己写好的源码上传到`npm`官网上，如果要用某个或某些个，直接通过`npm`安装就可以了，不用管那个源码在哪里。
并且如果我们要使用模块`A`，而模块`A`又依赖模块`B`，模块`B`又依赖模块`C`和`D`，此时`npm`会根据依赖关系， 把所有依赖的包都下载下来并且管理起来。试想如果这些工作全靠我们自己去完成会多么麻烦！其实就是类似于`Java`中的`Maven`。  

## 3. <i class="fas fa-cloud-download-alt"></i> 下载及安装Hexo

### 3.1下载及安装`Hexo`

  在`cmd`终端窗口执行下载安装命令  

  ``` javascript
  npm install hexo-cli -g
  ```

  安装完成如下图：  

  ![安装Hexo](安装Hexo.png)

  执行命令判断`Hexo`安装是否成功

  ```javascript
  hexo -v
  ```

  安装成功如下：  

  ![安装Hexo成功](安装Hexo成功.png)

### 3.2初始化博客

  > 以下操作尽可能都在`Git`终端操作

  首先在自己本地磁盘创建一个安装目录，这里以`Hexo`为例；

  ![Hexo安装目录](Hexo安装目录.png)

  切换到此文件夹目录下执行初始化命令  

  ```javascript
  hexo init
  ```

  ![Hexo初始化中](Hexo初始化中.png)

  如果你未自己手动创建文件夹，也可通过以下的命令去初始化
  ```javascript
  // 建立一个博客文件夹，并初始化博客，<folder>为文件夹的名称，可以随便起名字
  hexo init <folder>
  // 进入博客文件夹，<folder>为文件夹的名称
  cd <folder>
  // node.js的命令，根据博客既定的dependencies配置安装所有的依赖包
  npm install
  ```

  初始化完成  

  ```javascript
  hexo g // 生成网页。
  hexo s // 将生成的网页放在了本地服务器。
  ```

  ![初始化完成](初始化完成.png)

  通过提示的信息，访问本地服务器  

  ![博客](博客.png)

## 4. <i class="fas fa-cog fa-spin"></i> 配置博客  

当然，目前的博客界面不是很美观，如果想做的比较有点逼格，当然还是得个性化定制下，我们可以去`Hexo`官网下载自己喜欢的主题。  

[Hexo官方主题](https://hexo.io/themes/)

选择你喜欢的主题，复制它的链接，`clone`到本地博客的`themes`目录下  

![克隆主题中](克隆主题中.png)

![主题克隆完成](主题克隆完成.png)

![克隆到本地的主题](克隆到本地的主题.png)

配置博客我们首先就得了解博客文件结构

```javascript
|node_modules:node.js的相关组件
|scaddolds: 定义的一些东西，固定的
|soorce: Markdown博客文档
|themes: 一些主题
	|主题名-|
    	   |主题内容（包含README.md,config.yml）
|config: 配置文件（博客全局配置）
|db.json: 生成的一些东西
|package.json:  当前npm的相关的包
|package-lock.json: npm管理的一些东西
```

按照主题的`README.md`文档进行配置。  
例如我使用的博客主题是[Annie](https://github.com/Sariay/hexo-theme-Annie)  
## 5. <i class="fas fa-user-cog"></i> 博客个性化设置
我们按照自己选择的`Hexo`主题进行配置后，如果想根据自己的喜好做相应的修改当然也是可以的。比如可以给博客添加图片、视频、音乐播放器等等。  
### 5.1博客中添加图片[1]   
博客中的图片添加有以下几种方式：
#### 本地引用    
- 绝对路径  
直接在主题下的`img`（存储图片文件夹，不同的主题存储图片的名称可能不同）文件夹下(themes/所选主题文件夹/source/img),`/img/图片名称.jpg`这张图片，就可以使用以下方式访问： 
```cmd
![图片说明](/img/图片名称.jpg)  
```
eg:    
![wechat](/img/wechat.jpg)   

- 相对路径  
图片除了可以放在统一的`img`文件夹中，还可以放在文章自己的目录中。文章的目录可以通过配置博客根目录下的`_config.yml`来生成。  

```javascript
post_asset_folder: true 
```  

将_config.yml文件中的配置项`post_asset_folder`设为`true`后，执行命令`$ hexo new post_name`，在`source/_posts`中会生成文章`post_name.md`和同名文件夹`post_name`。将图片资源放在`post_name`文件夹中，文章就可以使用相对路径引用图片资源了。`_posts/post_name/图片名称.jpg`这张图片可以用以下方式访问：  

```cmd
![图片说明](图片名称.jpg) 
```   

eg:  
![微信公众号](微信公众号.jpg)  
#### CDN引用
除了在本地存储图片，还可以将图片上传到一些免费的 `CDN`服务中。因国内访问`GitHub`速度较慢，所以将突破放到国内图床上，然后引用外链接是常用的方法。  
常用图床总结：https://sspai.com/post/40499  
常用的图床有：七牛云、腾讯云、微博图床等。  
#### GitHub  
使用`github`存储博客图片
 1. 创建一个空的仓库  
 2. 将图片`push`到仓库中
 3. 点击图片进去，有个`download`，右键复制链接
 4. 将链接插入文章  
引用格式：  

```cmd
![logo](https://github.com/xxxx/xx.jpg)
```  

#### 使用插件  
 1. 首先把`blog（hexo）`目录下的`_config.yml`里的`psot_asset_folder:`设置为`true`
 2. 在`blog（hexo）`目录下执行:  

```cmd  
npm install hexo-asset-image --save
```  

 3. 在`blog（hexo）`目录下`Git Bash Here`，运行`hexo n "博客文章名"`来生成`md`博客时，会在`_post`目录下看到一个与博客同名的文件夹。  
 4. 将想要上传的图片先扔到文件夹下，然后在博客中使用`markdown`的格式引入图片：  

```cmd  
![你想要输入的替代文字](xxxx/图片名.jpg) 
```   

>因为博客名和文件夹名字相同，所以不需要绝对路径，只要xxxx是文件夹的名字就可以了。
### 5.2博客中添加视频[2] 
> 以`bilibili`为例，B站无广告   

- 去B站获取视频外链  
![获取视频外链](获取视频外链.png)  
- 在文章中插入视频外链   
我们知道在`md`中可以直接插入`html`代码。这里我们就插入视频外链。代码如下：  

```html
<iframe src="//player.bilibili.com/player.html?aid=68662896&cid=118997493&page=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"> </iframe>
```   

我们可以看到效果令人很不满意。  

- 修改代码，美化播放器样式。  
代码如下：  

```html 
<div style="position: relative; width: 100%; height: 0;padding-bottom: 75%;" >
<iframe src="//player.bilibili.com/player.html?aid=68662896&cid=118997493&page=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true" style="position: absolute; width: 100%; height: 100%; left: 0; top: 0;"> </iframe></div>
```   

### 5.3博客中添加网易云音乐歌单    
- （以 Chrome 为例，其他浏览器类似）打开歌单页面，在“生成外链播放器”上右击，点击检查（审查元素 `ctrl+shift+i`）；  
![网易云音乐外链](网易云音乐外链.png)
- 接着找到生成外链播放器这段文字直接双击复制前面的`/outchain/0/170792779/`  
![外链id](外链id.png)  
- 然后修改歌单链接示例：http://music.163.com/#/outchain/0/170792779/（可以修改自己喜欢的播放器尺寸，播放模式后再复制代码） 
![网易云音乐歌单](网易云音乐歌单.png)    

>由于版权限制，好多歌曲可能在播放器中无法播放，毕竟没有收费，将就用吧！😂
### 5.4博客中实现在线联系功能  
在线联系功能可以使访客及时，快捷的与博主交流，也能帮助博主及时的解决访客提出的博文中的问题。`Hexo`实现在线联系功能主要有以下两种方式:  
#### DaoVoice实现在线联系  
- 注册登录`DaoVoice`  
[注册登录DaoVoice](http://dashboard.daovoice.io/get-started?invite_code=75159429)  
- `DaoVoice`接入  
[`DaoVoice`接入](http://guide.daocloud.io/daovoice/daovoice-9151028.html)  
- `Daovoice`绑定微信（可选）  
`DaoVoice`虽然可以很好的与访客交流，但是还是不能像微信聊天一样方便，所以我们绑定微信，瞬间秒回访客消息，不再等待！  
![DaoVoice绑定微信](DaoVoice绑定微信.png)
#### HEXO的博客添加gitter在线交流  
[给基于HEXO的博客添加gitter在线交流](https://blog.csdn.net/u011606307/article/details/89504541)
## 6. <i class="fab fa-dev"></i> 持续集成Hexo博客  
#### 6.1使用Jenkins持续集成Hexo博客[3]    
[使用Jenkins持续集成Hexo博客](http://www.sevenyuan.cn/2019/03/18/%E4%BD%BF%E7%94%A8Jenkins%E6%8C%81%E7%BB%AD%E9%9B%86%E6%88%90Hexo%E5%8D%9A%E5%AE%A2/)  
#### 6.2用TravisCI持续集成Hexo博客    
[Travis CI个人免费站点](https://travis-ci.org/)  
[Travis CI企业收费站点](https://travis-ci.com/)  
##### 6.2.1核心概念 
- 什么是持续集成（CI）？[4]    
CI(`Continuous Integration`)翻译为持续集成。  
持续集成是经常合并小的代码更改的实践，而不是在开发周期结束时合并大的更改。目的是通过以较小的增量开发和测试来构建更健康的软件。这就是`Travis CI`出现的地方。  
作为一个持续集成平台，`Travis CI`通过自动构建和测试代码更改来支持您的开发过程，并提供有关更改成功的即时反馈。`Travis CI`还可以通过管理部署和通知来自动化开发过程的其他部分。  
##### 6.2.2准备条件  
要开始使用Travis CI，请确保您具有：

- 一个[GitHub](https://github.com/)的帐户。  
- [托管在GitHub上的项目](https://help.github.com/en/github/importing-your-projects-to-github)的所有者权限。  
  - 博客源码仓库  
  - 博客部署仓库  
##### 6.2.3 关联仓库
使用GitHub登录到[Travis CI个人免费站点](https://travis-ci.org/)  
![travis login](travis_login.png)  
关联到持续集成的仓库  
![持续集成的仓库](repo.png)  
**配置 Access Token**  
如下图，Environment Variables 区域就是用来添加权限信息的。我们需要填写一个Token的名称和值，该名称可以在配置文件中以 ${变量名} 来引用，该Token我们需要从Github中获取。  
![](travis_setting.png)  
**从Github获取Access Token**  
在Github的setting页面，左侧面板选择Developer settings然后Personal access tokens, 右上角点击Generate new token。生成token时候需要确定访问scope，这里我们选择我们的repo即可。 
![](travis_token.png) 
重要：生成的token只有第一次可见，一定要保存下来备用。  
![token](token.png)  
**在Travis CI中配置**
将上面获取到的token添加到 Environment Variables 部分，值为该 token ,而名称即为上面设置的 Travis_Token (请更改为个人所设置名称)。不勾选后面的 Display value in build log . 否则会在日志文件中暴露你的 token 信息，而日志文件是公开可见的。

至此我们已经配置好了要构建的仓库和访问的token，接下来就是如何构建的问题了。  
**配置.travis.yml（如果没有，新建)**  
我个人的.travis.yml 可供参考
```yml
# 指定构建环境是Node.js，当前版本是稳定版
sudo: false
language: node_js
node_js:
  - 10

env:
 global:
   - URL_REPO: github.com/10veU/10veU.github.io.git

# 设置钩子只检测blog-source分支的push变动
branches:
  only:
    - master

# 设置缓存文件
cache:
  directories:
    - node_modules

#在构建之前安装hexo环境
before_install:
  - npm install -g hexo-cli

#安装git插件和搜索功能插件
install:
  - npm install

# 执行清缓存，生成网页操作
script:
  - hexo clean
  - hexo generate
deploy:
  provider: pages
  skip_cleanup: true
  github_token: $GH_TOKEN  # Set in the settings page of your repository, as a secure variable
  keep_history: true
  on:
    branch: master
  local-dir: public
# configure notifications (email, IRC, campfire etc)
# please update this section to your needs!
# https://docs.travis-ci.com/user/notifications/
notifications:
  email:
    - 514084647@qq.com
  on_success: change
  on_failure: always
```
注意：需要将配置文件中的 GH_TOKEN 换成我们自己设定的名称，这里我的配置应该是 Travis_token 即 - git push --force --quiet "https://${Travis_token}@${GH_REF}" master:master # GH_TOKEN是在Travis中配置token的名称。 还要更改 GH_REF 中我们的博客仓库的地址。  
![travis_success](travis_success.png)


[1]: https://vwin.github.io/2018/08/07/Hexo%E5%8D%9A%E5%AE%A2%E6%96%87%E7%AB%A0%E6%8F%92%E5%85%A5%E5%9B%BE%E7%89%87/  
[2]: https://baijiahao.baidu.com/s?id=1623914788952059989&wfr=spider&for=pc   
[3]: https://www.karlzhou.com/2016/05/28/travis-ci-deploy-blog/  
[4]: https://docs.travis-ci.com/user/for-beginners/#breaking-the-build



