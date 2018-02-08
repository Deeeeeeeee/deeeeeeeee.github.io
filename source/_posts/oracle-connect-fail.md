---
title: 记连接 oracle 数据库失败
date: 2018-02-08 14:44:40
categories:
  - 生产问题
tags:
  - oracle
  - connect-fail
  - 1153
---

plsql 客户端可以连上 oracle 数据库，程序连接不上。报错信息 oracle 1153 或 oracle 12505，connection refused

<!-- more -->

错误信息1：
{% codeblock  %}
org.apache.commons.dbcp.SQLNestedException: Cannot create PoolableConnectionFactory
 (Io 异常: Connection refused(DESCRIPTION=(ERR=1153)(VSNNUM=0)(ERROR_STACK=(ERROR=(CODE=1153)(EMFI=4)
 (ARGS='(ADDRESS=(PROTOCOL=TCP)(HOST=10.100.1.13)(PORT=1521))'))(ERROR=(CODE=305)(EMFI=1)))))
{% endcodeblock %}

错误信息2：
``` 
ORA-12505, TNS:listener does not currently know of SID given in connect descriptor
The Connection descriptor used by the client was:
xx.xxx.xx.xx:1521:xxx
```

{% note success %}
服务器是 windows 2008 server, oracle 是 10g, jdk 是 1.6
{% endnote %}

刚开始是报第一个错误，换了几个版本的驱动包，直到换成 ojdbc6_11.1.0.7.0，出现了第二个报错。
然后将配置信息从
```
<property name="url" value="jdbc:oracle:thin:@172.xx.xx.xxx:1521:xxxx" />
```
改成了
```
<property name="url" value="jdbc:oracle:thin:@(DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=TCP)(HOST=172.xx.xx.xxx)(PORT=1521)))(CONNECT_DATA=(SERVICE_NAME=xxxx)))" />
```

这个数据就是跟配置在 tnsnames.ora 上的，最后连接上了数据库