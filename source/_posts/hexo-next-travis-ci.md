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
<video height="100%" width="100%" loop="loop"  controls="controls">
  <source src="/post/hexo-next-travis-ci/show.mp4" type="video/mp4"></source>
</video>

构建效果如上面视频所示，如果浏览器不支持请戳一下链接: {% asset_link show.mp4 自动化部署构建效果 %} 。**只要将编辑的 .md 文件推送到 github 上，博客网站就可以更新这篇文章**。

其实差不多半年前也构建过一次，由于安装 travis-ci 失败终结了~这一次看到小伙伴也在使用，就想整好拿来玩玩。只能说，这些还挺有意思，挺牛逼的，哈哈哈。

------

# 稍微介绍一下所用的工具: **hexo 博客框架**、**NexT 主题**、**travis-ci 持续集成工具**
## hexo

{% blockquote hexo https://hexo.io hexo 官网 %}
A fast, simple & powerful blog framework
{% endblockquote %}

- hexo 一个基于 node.js 开源博客框架
- hexo 的插件和主题可以在文档中找到 --- [中文文档](https://hexo.io/zh-cn/)
- github 有相关的组织和开发主题的教程 --- [hexojs/hexo](https://github.com/hexojs/hexo)  [hexojs/awesome-hexo](https://github.com/hexojs/awesome-hexo)

## NexT

{% blockquote NexT http://theme-next.iissnan.com NexT使用文档 %}
精于心，简于形 Elegant Theme for Hexo
{% endblockquote %}

NexT 一个 hexo 下的高可定制化主题，并且有 Muse、Mist、Pisces、Gemini 几种模式。目前我使用的就是 Mist。

## travis-ci

{% blockquote travis-ci https://www.travis-ci.org travis-ci 官网 %}
Free continuous integration platform for GitHub projects.
{% endblockquote %}

一个开源持续集成项目，通过执行预先的脚本完成相应的部署 --- [travis-ci github](https://github.com/travis-ci)

------

# 安装 hexo 和建站

## 安装前提

安装 Hexo 相当简单。然而在安装前，您必须检查电脑中是否已安装下列应用程序：
- [Node.js 官网](https://nodejs.org/en/)
- [Git 官网](https://git-scm.com/)

对于 linux 用户，可以选择用命令行安装 Node.js 和 Git。我已经忘记当初我是怎么安装的了～可以参考 [hexo 官网文档](https://hexo.io/zh-cn/docs/)

## 安装 hexo

如果下载速度缓慢，可以添加**淘宝镜像**。以下两者都可以添加镜像。
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
简写
$ hexo c
{% endcodeblock %}

更多命令参考 [hexo 指令](https://hexo.io/zh-cn/docs/commands.html)

执行完 **hexo g && hexo s** 后你将会看到如下页面：
{% asset_img hexo-start.png %}

.. 未完待续