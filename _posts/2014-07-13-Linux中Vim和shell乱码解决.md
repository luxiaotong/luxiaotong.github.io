---
layout: post
title:  Linux中Vim和shell乱码解决
---

- 问题: 使用Xshell连接到开发机, Vim和shell中的中文都出现乱码

- 解决: 
    1. Xshell编码改成UTF-8
    2. vim ~/.bash_profile, 增加一行:
        `export LC_ALL=en_US.utf8`
    3. source ~/.bash_profile

- 备注:
    1. 由于没有sudo和root权限, 所以只能修改当前登录用户的~/.bash_profile, 如果有权限可以尝试修改/etc/sysconfig/i18n
    2. source的作用是把刚才修改的内容加载一下.否则修改完需要重启或重新登录.
    3. 修改前可以使用locale -a, 查看当前支持的编码.
        

- 参考:
    - [http://blog.csdn.net/wangjun_pfc/article/details/4009205](http://blog.csdn.net/wangjun_pfc/article/details/4009205 "http://blog.csdn.net/wangjun_pfc/article/details/4009205")
    - [http://www.cnblogs.com/Jack47/p/3551584.html](http://www.cnblogs.com/Jack47/p/3551584.html "http://www.cnblogs.com/Jack47/p/3551584.html")
