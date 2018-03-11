title: hexo-next-travis-ci 构建自动化部署博客
tags:
  - hexo
  - next
  - travis-ci
  - blog
categories:
  - 博客
date: 2018-02-03 20:56:45
---
<video height="100%" width="100%" loop="loop" controls="controls">
  <source src="/post/hexo-next-travis-ci/show.mp4" type="video/mp4"></source>
</video>

构建效果如上面视频所示，如果浏览器不支持请戳一下链接: {% asset_link show.mp4 自动化部署构建效果 %} 。**只要将编辑的 .md 文件推送到 github 上，博客网站就可以更新这篇文章**。

<!-- more -->

其实差不多半年前也构建过一次，由于安装 travis-ci 失败终结了~这一次看到小伙伴也在使用，就想整好拿来玩玩。只能说，这些还挺有意思，挺牛逼的，哈哈哈。

------

# 稍微介绍一下所用的工具: **hexo 博客框架**、**NexT 主题**、**travis-ci 持续集成工具**
## hexo

{% note success %}
A fast, simple & powerful blog framework

**hexo** - [*hexo 官网*](https://hexo.io)
{% endnote %}

- hexo 一个基于 node.js 开源博客框架
- hexo 的插件和主题可以在文档中找到  [*中文文档*](https://hexo.io/zh-cn/)
- github 有相关的组织和开发主题的教程  [*hexojs/hexo*](https://github.com/hexojs/hexo)  [*hexojs/awesome-hexo*](https://github.com/hexojs/awesome-hexo)

## NexT

{% note info %}
精于心，简于形 Elegant Theme for Hexo

**NexT** - [*NexT使用文档*](http://theme-next.iissnan.com)
{% endnote %}

NexT 一个 hexo 下的高可定制化主题，并且有 Muse、Mist、Pisces、Gemini 几种模式。目前我使用的就是 Mist。	[*NexT github*](https://github.com/iissnan/hexo-theme-next)

## travis-ci

{% note warning %}
Free continuous integration platform for GitHub projects.

**travis-ci** - [*travis-ci 官网*](https://www.travis-ci.org)
{% endnote %}

一个开源持续集成项目，通过执行预先的脚本完成相应的部署  [*travis-ci github*](https://github.com/travis-ci)

------

# 安装 hexo 和建站

## 安装前提

安装 Hexo 相当简单。然而在安装前，您必须检查电脑中是否已安装下列应用程序：
- [*Node.js 官网*](https://nodejs.org/en/)
- [*Git 官网*](https://git-scm.com/)

对于 linux 用户，可以选择用命令行安装 Node.js 和 Git。我已经忘记当初我是怎么安装的了～可以参考 [*hexo 官网文档*](https://hexo.io/zh-cn/docs/)

## 安装 hexo

{% note success %}
如果下载速度缓慢，可以添加**淘宝镜像**。以下两者都可以添加镜像。
{% endnote %}

- 第一个可以用 npm config get registry 验证配置是否成功
- 第二个使用的时候用 cnpm 替换 npm， cnpm install express

{% codeblock  %}
$ npm config set registry https://registry.npm.taobao.org
或
$ npm install -g cnpm --registry=https://registry.npm.taobao.org
{% endcodeblock %}

安装 hexo 
{% codeblock %}
$ npm install -g hexo-cli
{% endcodeblock %}

## 建站准备

安装 Hexo 完成后，请执行下列命令，Hexo 将会在指定文件夹中新建所需要的文件。
{% codeblock %}
$ hexo init <folder>
$ cd <folder>
$ npm install
{% endcodeblock %}

新建完成后，指定文件夹的目录如下：
{% codeblock %}
.
├── _config.yml
├── package.json
├── scaffolds
├── source
|   ├── _drafts
|   └── _posts
└── themes
{% endcodeblock %}

### _config.yml
网站的**配置**信息，您可以在此配置大部分的参数。

### package.json
应用程序的信息。EJS, Stylus 和 Markdown renderer 已默认安装，您可以自由移除。
{% codeblock package.json lang:json %}
{
  "name": "hexo-site",
  "version": "0.0.0",
  "private": true,
  "hexo": {
    "version": ""
  },
  "dependencies": {
    "hexo": "^3.0.0",
    "hexo-generator-archive": "^0.1.0",
    "hexo-generator-category": "^0.1.0",
    "hexo-generator-index": "^0.1.0",
    "hexo-generator-tag": "^0.1.0",
    "hexo-renderer-ejs": "^0.1.0",
    "hexo-renderer-stylus": "^0.2.0",
    "hexo-renderer-marked": "^0.2.4",
    "hexo-server": "^0.1.2"
  }
}
{% endcodeblock %}

### scaffolds
模版文件夹。当您新建文章时，Hexo 会根据 **scaffold** 来建立文件。

Hexo的模板是指在新建的markdown文件中**默认填充**的内容。例如，如果您修改**scaffold/post.md**中的Front-matter内容，那么每次新建一篇文章时都会包含这个修改。

### source
资源文件夹是**存放用户资源**的地方。除 _posts 文件夹之外，开头命名为 _ (下划线)的文件 / 文件夹和隐藏的文件将会被忽略。Markdown 和 HTML 文件会被解析并放到 public 文件夹，而其他文件会被拷贝过去。

### themes
主题文件夹。Hexo 会根据主题来生成静态页面。Hexo 默认主题是 landscape。

## 本地开启服务
生成静态文件并启动服务器

{% codeblock %}
$ hexo generate && hexo server
简写
$ hexo g && hexo s
{% endcodeblock %}

清楚缓存文件(db.json)和已生成的静态文件(public)。
{% codeblock %}
$ hexo clean
{% endcodeblock %}

更多命令参考 [*hexo 指令*](https://hexo.io/zh-cn/docs/commands.html)

执行完 **hexo g && hexo s** 后你将会看到如下页面：
{% asset_img hexo-start.png %}

------

# 使用 NexT 主题

觉得默认的主题不满意，那就换个主题，比如 NexT。

在 Hexo 中有两份主要的配置文件，其名称都是 `_config.yml`。 其中，一份位于**站点根目录**下，主要包含 Hexo 本身的配置，称为**站点配置文件**；另一份位于**主题目录**下，这份配置由主题作者提供，主要用于配置主题相关的选项，称为**主题配置文件**。

## 安装 NexT

Hexo 安装主题的方式**非常简单**，只需要将主题文件拷贝至站点目录的 `themes` 目录下， 然后修改下配置文件即可。具体到 NexT 来说，安装步骤如下。

### 下载主题

{% codeblock %}
$ cd <folder>
$ git clone https://github.com/iissnan/hexo-theme-next themes/next
{% endcodeblock %}

### 启用主题

与所有 Hexo 主题启用的模式一样。 当 克隆/下载 完成后，打开 **站点配置文件**，即 &lt;folder&gt; 目录下的 `_config.yml` 文件， 找到 `theme` 字段，并将其值更改为 next。

{% codeblock %}
theme: next
{% endcodeblock %}

## 主题设定

### 选择 Scheme（模式）

Scheme 的切换通过更改 **主题配置文件**，即 **theme/next** 目录下的 `_config.yml` 文件， 搜索 scheme 关键字。 你会看到有四行 scheme 的配置，将你需用启用的 scheme 前面注释 # 去除即可。

{% codeblock %}
#scheme: Muse
scheme: Mist
#scheme: Pisces
#scheme: Gemini
{% endcodeblock %}

### 设置语言

编辑 **站点配置文件**， 将 `language` 设置成你所需要的语言，可以在 themes/next/languages 下找到。建议明确设置你所需要的语言，例如选用简体中文，配置如下：

{% codeblock %}
language: zh-Hans
{% endcodeblock %}

这里只是简单的配置，更多的配置请参考 [*NexT 使用文档*](http://theme-next.iissnan.com/getting-started.html)

## 验证主题

{% note success %}到此，NexT 主题安装和简单配置完成。{% endnote %}
下一步我们将验证主题是否正确启用。在切换主题之后、验证之前， 我们最好使用 `hexo clean` 来清除 Hexo 的缓存。

启动 Hexo 本地站点

{% codeblock %}
$ hexo clean
$ hexo g && hexo s
{% endcodeblock %}

启动时你将会看到类似如下日志
{% codeblock %}
INFO  Generated: index.html
INFO  Generated: archives/index.html
INFO  Generated: images/algolia_logo.svg
INFO  Generated: images/apple-touch-icon-next.png
INFO  Generated: archives/2018/index.html
INFO  Generated: images/avatar.gif
INFO  Generated: images/cc-by-nc-sa.svg
INFO  Generated: images/cc-by-nc-nd.svg
INFO  Generated: images/cc-by-nd.svg
INFO  Generated: images/cc-by-nc.svg
INFO  Generated: images/cc-by-sa.svg
INFO  Generated: images/cc-zero.svg
INFO  Generated: images/favicon-16x16-next.png
INFO  Generated: images/cc-by.svg
INFO  Generated: images/favicon-32x32-next.png
INFO  Generated: images/loading.gif
INFO  Generated: images/logo.svg
INFO  Generated: images/placeholder.gif
INFO  Generated: images/quote-r.svg
INFO  Generated: images/searchicon.png
INFO  Generated: images/quote-l.svg
INFO  Generated: lib/fastclick/LICENSE
INFO  Generated: archives/2018/02/index.html
INFO  Generated: lib/font-awesome/HELP-US-OUT.txt
INFO  Generated: lib/canvas-nest/canvas-nest.min.js
INFO  Generated: lib/canvas-ribbon/canvas-ribbon.js
INFO  Generated: lib/algolia-instant-search/instantsearch.min.css
INFO  Generated: lib/font-awesome/bower.json
INFO  Generated: lib/jquery_lazyload/CONTRIBUTING.html
INFO  Generated: lib/fastclick/bower.json
INFO  Generated: lib/jquery_lazyload/bower.json
INFO  Generated: lib/fastclick/README.html
INFO  Generated: lib/jquery_lazyload/README.html
INFO  Generated: lib/jquery_lazyload/jquery.lazyload.js
...
...
INFO  Hexo is running at http://localhost:4000/. Press Ctrl+C to stop.
{% endcodeblock %}

访问本地站点 http://localhost:4000/ 会得到如下页面
{% asset_img hexo-next.png %}

</br>  
  
{% note warning %}
在这里只是做最简单的配置，更多的配置请参考 [*NexT 使用文档-主题配置*](http://theme-next.iissnan.com/theme-settings.html)
{% endnote %}

## github io 部署

部署到网络有多种途径，可以部署到服务器上，也可以部署到 github 上，github pages 提供给用户部署静态页面，这里就部署到 github 上。

### 配置 github 仓库

创建仓库：名称（首字母小写）.github.io
如： 我的 github 名称为 Deeeeeeeee，那么创建仓库名为 deeeeeeeee.github.io

{% asset_img github-new-repository.png %}

</br>

进入仓库 ==》 <i class="fa fa-cog fa-lg"></i> Settings ==》 `GitHub Pages` ==》 选择 master 分支作为 github pages 分支

{% asset_img github-pages-setting.png %}

### 配置 hexo 部署信息

{% note warning %}
这里略过 SSH keys 配置的步骤，百度会告诉你
**更多部署操作参考 [*Hexo 文档-部署*](https://hexo.io/zh-cn/docs/deployment.html#Git)**
{% endnote %}

安装 [*hexo-deployer-git*](https://github.com/hexojs/hexo-deployer-git)

{% codeblock %}
$ npm install hexo-deployer-git --save
{% endcodeblock %}

修改站点配置
{% codeblock %}
deploy:
  type: git
  repo: <repository url>
  branch: [branch]
  message: [message]
{% endcodeblock %}

{% codeblock 我的配置 %}
deploy:
  type: git
  repo: https://github.com/Deeeeeeeee/deeeeeeeee.github.io
  branch: master
  message:
{% endcodeblock %}

### 推送到 github

如果配置了 github，hexo 部署的时候会将 public 文件夹下的文件推送到指定地址和分支
**执行部署命令**
{% codeblock %}
$ hexo deploy
简写
$ hexo d
{% endcodeblock %}

这时候可以在 github 仓库中看到更新的文件，访问页面你的 github io，如: https://deeeeeeeee.github.io

{% asset_img github-master.png %}

------

# 使用 Travis CI

{% note success %}
如果换工作环境，那么又需要安装 Node.js、Git、Hexo 等等。这时候可以使用 travis-ci 持续集成工具，虽然这比较鸡肋，因为你会在每个工作环境都装上 Node.js、Git、Hexo 等等。啊哈哈哈~
{% endnote %}

## Travis CI 官网配置

由于使用 travis-ci，所以需要在官网上配置 github 信息。[*Travis CI 官网地址*](https://www.travis-ci.org)

### 配置项目信息

进入官网并登录，选择添加仓库

{% asset_img travis-repository.png %}
</br>
项目构建配置

{% asset_img travis-build-setting.png %}

### 环境变量设置

由于推送代码需要 github 权限，又不能将账号密码暴露，所以可以将 github Token 设置为环境变量

<i class="fa fa-github fa-lg"></i> github ==》 <i class="fa fa-cog fa-lg"></i> Settings ==》 Developer settings ==》 Personal access tokens ==》 Generate new token

{% asset_img github-token.png %}
</br>

{% note success %}
将 token 保存好，因为 github 上只显示一次
{% endnote %}

在 travis 上设置 token 环境变量

{% asset_img travis-token-setting.png %}

## 编写 travis 脚本

{% codeblock lang:yml %}
language: node_js
node_js: stable

# S: Build Lifecycle
install:
  - npm install
  - npm install hexo-generator-searchdb --save

#before_script:

script:
  - hexo clean && hexo g

after_script:
  - cd ./public
  - git init
  - git config user.name "Deeeeeeeee"
  - git config user.email "seal.de@foxmail.com"
  - git add .
  - git commit -m "Update docs"
  - git push --force --quiet "https://${GIT_HUB_TOKEN}@${GH_REF}" master:master
# E: Build LifeCycle

branchs:
  only:
    - dev

env:
  global:
    - GH_REF: github.com/Deeeeeeeee/deeeeeeeee.github.io.git

{% endcodeblock %}

## 将文件推送到 dev 分支

{% note warning %}
推送到远程分之前，建议删除 themes/next 下的 .git 文件夹和 .gitignore 文件
{% endnote %}

给本地仓库添加 dev 分支
{% codeblock %}
$ git checkout -b dev1
{% endcodeblock %}

将分支推送到远程
{% codeblock %}
$ git push --force -u origin dev
{% endcodeblock %}

推送上去之后文件目录，如下图
{% asset_img github-dev.png %}

**最后在 travis 上就可以看到部署你的博客到 github pages 的日志**