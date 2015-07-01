---
layout: post
title:  解决安装xhprof后 fpm报错问题(recv failed 104 nginx 502)
---

问题描述
---
**安装xhprof后，在代码中加入**

```
xhprof_enable(XHPROF_FLAGS_CPU+XHPROF_FLAGS_MEMORY); 
$xhprof_data = xhprof_disable();
var_dump($xhprof_data);exit;

```
**通过浏览器访问出现nginx 502报错**
![](http://luxiaotong-image.stor.sinaapp.com/nginx_502.png)

**nginx error log报错如下**

```
2015/06/29 21:30:26 [error] 27710#0: *140 recv() failed (104: Connection reset by peer) while reading response header from upstream, client: 10.44.112.83, server: , request: "GET /test.php HTTP/1.1", upstream: "fastcgi://127.0.0.1:8500", host: "cq01-rdqa-dev091.cq01.baidu.com:8001"
```

**fpm log日志信息如下**
```
[29-Jun-2015 21:30:26] WARNING: [pool www2] child 27270 exited on signal 11 (SIGSEGV - core dumped) after 7526.184909 seconds from start
```

问题解决过程
---
首先从上述日志并没有看出具体的问题，经过在谷组上搜索，『recv failed 104 connection resetby peer』

发现要在fpm日志输出更多错误信息。
[方法如下](http://stackoverflow.com/questions/18379323/failed-104-connection-reset-by-peer)

修改fpm配置文件fpm.conf，加入**catch\_workers_output = yes**

```
[global]
pid = run/fpm.pid
error_log = log/fpm.log
daemonize = yes 
[www2]
user=nobody
group=nobody
listen = ***
pm = dynamic
pm.max_children = 200 
pm.start_servers = 2 
pm.min_spare_servers = 1 
pm.max_spare_servers = 3 
request_terminate_timeout = 120s
rlimit_files = 1310
rlimit_core = unlimited
catch_workers_output = yes
```

重启fpm，再次请求，fpm log会输出如下信息

```
30-Jun-2015 19:02:20] WARNING: [pool www2] child 17892 said into stderr: "setaffinity: Invalid argument"
```

再次在谷姐上搜索『setaffinity: Invalid argument』
找到在[php bug中有对该问题的描述，以及对该问题的分析](https://bugs.php.net/bug.php?id=60078)，并且在评论中找到[修复此问题的代码](https://github.com/odoucet/xhprof/commit/2e74533746bf14b0bcfc9a6fae08e1bf9b4f724b)

于是**将xhprof.c按照提交中的样子修改**一下，之后将xhprof重新编译安装，就可以正常运行了。

```
array(2) {
  ["main()==>xhprof_disable"]=>
  array(5) {
    ["ct"]=>
    int(1)
    ["wt"]=>
    int(3)
    ["cpu"]=>
    int(0)
    ["mu"]=>
    int(840)
    ["pmu"]=>
    int(0)
  }
  ["main()"]=>
  array(5) {
    ["ct"]=>
    int(1)
    ["wt"]=>
    int(16)
    ["cpu"]=>
    int(0)
    ["mu"]=>
    int(1856)
    ["pmu"]=>
    int(0)
  }
}
```

参考文献
---
[http://stackoverflow.com/questions/18379323/failed-104-connection-reset-by-peer](http://stackoverflow.com/questions/18379323/failed-104-connection-reset-by-peer)

[https://bugs.php.net/bug.php?id=60078](https://bugs.php.net/bug.php?id=60078)

[https://github.com/odoucet/xhprof/commit/2e74533746bf14b0bcfc9a6fae08e1bf9b4f724b](https://github.com/odoucet/xhprof/commit/2e74533746bf14b0bcfc9a6fae08e1bf9b4f724b)