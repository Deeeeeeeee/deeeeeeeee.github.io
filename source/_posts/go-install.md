---
title: 【go】安装和配置
categories:
  - go
tags:
  - go
date: 2021-10-02 23:31:47
---

## 目录

- 安装 
- vim配置
- goncurses安装

## 安装

- 下载 [https://golang.org/dl/](https://golang.org/dl/)
- tar -zxvf goxxx.tar.gz 到 /usr/local/lib/go
- 环境变量 ~/.bash_profile
  ```
  export GOROOT=/usr/local/lib/go             # go安装路径
  export GOPATH=$HOME/.go                     # 工作目录. go get 和 go install 会使用的目录
  export PATH=$PATH:$GOPATH/bin:$GOROOT/bin   # 添加到path
  export GOPROXY=https://goproxy.cn,direct    # 代理加速
  ```
- go version 检测配置结果

<!-- more -->

## vim配置

前置需要配置vim插件管理等，可以参考 [https://gitee.com/sealde/dotfile](https://gitee.com/sealde/dotfile)

- 一个是使用 coc-go，[https://github.com/josa42/coc-go](https://github.com/josa42/coc-go)
- 另一个是使用 vim-go,
  - `Plug 'fatih/vim-go', { 'tag': '*' }`   # go 主要插件 [地址](https://github.com/fatih/vim-go)

我这里使用 vim-go 先安装 gopls
```
# go lsp(language server protocol)
go install golang.org/x/tools/gopls@latest
# 打开vim执行命令. vim-go依赖的工具自动安装。参考链接 https://zhuanlan.zhihu.com/p/51656877
:GoInstallBinaries
# 在 .vimrc 里加上. 参考 https://github.com/golang/tools/blob/master/gopls/doc/vim.md
let g:go_def_mode='gopls'
let g:go_info_mode='gopls'
```

自动补全
```
# coc-go. 在 .vimrc 加上 coc-go 插件
let g:coc_global_extensions = [
    \ "coc-go"]
# :ConConfig 加上lsp配置. 参考 https://github.com/golang/tools/blob/master/gopls/doc/vim.md
"languageserver": {
    "golang": {
        "command": "gopls",
        "rootPatterns": ["go.mod", ".vim/", ".git/", ".hg/"],
        "filetypes": ["go"],
        "initializationOptions": {
            "usePlaceholders": true
        }
    }
}
```

## goncurses安装

这个用来开发命令行ui的，不开发不需要安装

[https://github.com/rthornton128/goncurses](https://github.com/rthornton128/goncurses)

```
# 先安装ncurses库和pkg-config
sudo apt install libncurses5-dev
sudo apt install pkg-config
# 安装goncurses
go get github.com/rthornton128/goncurses
```

