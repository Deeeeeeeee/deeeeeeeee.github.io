---
title: wsl2-proxy
categories:
  - 杂项
tags:
  - wsl2
date: 2021-10-06 13:21:31
---

## 目录

- 获取主机ip和打开wsl防火墙
- 一键代理

## 获取主机ip和打开wsl防火墙

获取主机ip命令

```
ip route | grep default | awk '{print $3}'
# 或者
cat /etc/resolv.conf | grep nameserver | awk '{ print $2 }'
```

v2rayN 运行局域网的连接

```
参数设置 -> v2rayN设置 -> 勾选 来自局域网的连接 -> 确定
```

<!-- more -->

代理工具设置正确，仍访问不了，要打开wsl防火墙。参考 [https://github.com/microsoft/WSL/issues/4585](https://github.com/microsoft/WSL/issues/4585)

```
# 放开 `vEthernet (WSL)` 的防火墙
New-NetFirewallRule -DisplayName "WSL" -Direction Inbound -InterfaceAlias "vEthernet (WSL)" -Action Allow
```

## 一键代理

参考 [https://lengthmin.me/posts/wsl2-network-tricks/#一键设置代理](https://lengthmin.me/posts/wsl2-network-tricks/#%E4%B8%80%E9%94%AE%E8%AE%BE%E7%BD%AE%E4%BB%A3%E7%90%86)

在 .zshrc 中配置如下，然后执行 proxy 命令设置代理，unpro 命令取消代理设置

```
# Proxy configuration
getIp() {
    export winip=$(ip route | grep default | awk '{print $3}')
    export wslip=$(hostname -I | awk '{print $1}')
    export PROXY_SOCKS5="socks5://${winip}:1080"
    export PROXY_HTTP="http://${winip}:1081"
}

proxy_git() {
    ssh_proxy="${winip}:1080"
    git config --global http.https://github.com.proxy ${PROXY_SOCKS5}
    if ! grep -qF "Host github.com" ~/.ssh/config ; then
        echo "Host github.com" >> ~/.ssh/config
        echo "    User git" >> ~/.ssh/config
        echo "    ProxyCommand nc -X 5 -x ${ssh_proxy} %h %p" >> ~/.ssh/config
    else
        lino=$(($(awk '/Host github.com/{print NR}'  ~/.ssh/config)+2))
        sed -i "${lino}c\    ProxyCommand nc -X 5 -x ${ssh_proxy} %h %p" ~/.ssh/config
    fi
}

winip_() {
    getIp
    echo ${winip}
}

wslip_() {
    getIp
    echo ${wslip}
}

ip_() {
    getIp
    https --follow -b https://api.ip.sb/geoip/$1
    echo "WIN ip: ${winip}"
    echo "WSL ip: ${wslip}"
}

proxy() {
    getIp
    # pip can read http_proxy & https_proxy
    export http_proxy="${PROXY_HTTP}"
    export HTTP_PROXY="${PROXY_HTTP}"
    export https_proxy="${PROXY_HTTP}"
    export HTTPS_PROXY="${PROXY_HTTP}"
    export ftp_proxy="${PROXY_HTTP}"
    export FTP_PROXY="${PROXY_HTTP}"
    export rsync_proxy="${PROXY_HTTP}"
    export RSYNC_PROXY="${PROXY_HTTP}"
    export ALL_PROXY="${PROXY_SOCKS5}"
    export all_proxy="${PROXY_SOCKS5}"
    proxy_git
    if [ ! $1 ]; then
        ip_
    fi
    echo "Acquire::http::Proxy \"${PROXY_HTTP}\";" | sudo tee /etc/apt/apt.conf.d/proxy.conf >/dev/null 2>&1
    echo "Acquire::https::Proxy \"${PROXY_HTTP}\";" | sudo tee -a /etc/apt/apt.conf.d/proxy.conf >/dev/null 2>&1
}

unpro () {
    unset http_proxy
    unset HTTP_PROXY
    unset https_proxy
    unset HTTPS_PROXY
    unset ftp_proxy
    unset FTP_PROXY
    unset rsync_proxy
    unset RSYNC_PROXY
    unset ALL_PROXY
    unset all_proxy
    sudo rm /etc/apt/apt.conf.d/proxy.conf
    git config --global --unset http.https://github.com.proxy
    ip_
}
```

