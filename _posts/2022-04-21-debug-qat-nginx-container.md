---
layout: post
title: [DEBUG] QAT Nginx for docker 部署时"--with-ld-opt"出错
subtitle: 记一次debug经历
tags: [debug, linux]
comments: true
---

## [DEBUG] QAT Nginx for docker 部署时"--with-ld-opt"出错

在将 Openssl + QAT + async-mode-nginx 部署至docker的container中时，在执行async-mode-nginx的`./configure`时，出现了如下报错：

```
checking for OS
+ Linux 4.18.0-305.el8.x86_64 x86_64
checking for C compiler ... found
+ using GNU C compiler
+ gcc version: 8.5.0 20210514 (Red Hat 8.5.0-4) (GCC)
checking for gcc -pipe switch ... found
checking for --with-ld-opt="-Wl,-rpath=/root/openssl/lib -L/root/openssl/lib -lz" ... not found
./configure: error: the invalid value in --with-ld-opt="-Wl,-rpath=/root/openssl/lib -L/root/openssl/lib -lz"
```

"not found" , 找不到/root/openssl/lib这个目录。但是实际上该目录是存在的。起初我认为是docker container环境本身不支持"--with-ld-opt",  在bing和百度上搜索，找到一些类似问题，解决方法是屏蔽了该参数。确实，屏蔽之后nginx顺利编译，但是如此，nginx就无法与openssl交叉编译。

于是查找了async-mode-nginx中与“--with-ld-opt”相关的代码，在该source目录下的`auto/lib/openssl/conf`文件中，有三段相似代码，第一段是这样：

```
        if [ $ngx_found = no ]; then

            # FreeBSD port

            ngx_feature="OpenSSL library in /usr/local/"
            ngx_feature_path="/usr/local/include"

            if [ $NGX_RPATH = YES ]; then
                ngx_feature_libs="-R/usr/local/lib -L/usr/local/lib -lssl -lcrypto"
            else
                ngx_feature_libs="-L/usr/local/lib -lssl -lcrypto"
            fi

            ngx_feature_libs="$ngx_feature_libs $NGX_LIBDL $NGX_LIBPTHREAD"

            . auto/feature
        fi
```

nginx的./configure会执行 check OS 过程，如果发现缺少某些库，则会报错退出，ngx_found这个变量如果为0，就会进入如下判断，configure程序会依次寻找/usr/local/lib, /usr/pkg/lib和/opt/local/lib, 三个路径如果都没有找到openssl/lib, 则configure就会报错退出。

我echo了ngx_found的值，发现是0，一般来说在host下，没有出现为0的情况，所以不会走这三个判断。

因此，我决定试一试把container中的nginx/auto/lib/openssl/conf文件修改一下，把/usr/local/改成了本地的openssl地址:

```
     if [ $ngx_found = no ]; then

            # FreeBSD port
		   openssl_path=/root/openssl
            ngx_feature="OpenSSL library in $openssl_path"
            ngx_feature_path="$openssl_path/include"

            if [ $NGX_RPATH = YES ]; then
                ngx_feature_libs="-R$openssl_path/lib -L$openssl_path/lib -lssl -lcrypto"
            else
                ngx_feature_libs="-L$openssl_path/lib -lssl -lcrypto"
            fi

            ngx_feature_libs="$ngx_feature_libs $NGX_LIBDL $NGX_LIBPTHREAD"

            . auto/feature
        fi
```

再次编译，nginx通过了，openssl和nginx顺利交叉编译。

这说明，async mode nginx在container中完全可以正常安装，一定是编译环节缺失了什么才导致了出现--with-ld-opt出错。

于是我搜索nginx的--with-ld-opt参数， 发现它和--with-cc-opt参数一样，都与PCRE库密切相关。PCRE是Perl5的软件库，Perl5在文本处理和编译安装，系统运维上有重要作用，是很多软件的辅助工具，在python出现之前Perl的使用范围非常巨大，以至于今天仍是Linux系统自带的语言。PCRE库不是所有Linux系统自带的，需要手动安装。

于是在container(该container基于redhat8.4 ubi:213制作)中先更新了repo源，再执行：

```
yum install pcre2-tools.x86_64 pcre-devel.x86_64 zlib-static.x86_64 zlib
```

保守起见，conf文件中还提到了zlib，所以一并安装。

这次nginx的configure执行加上--with-ld-opt之后顺利编译，没有再报错。于是该BUG顺利解决！唯一没搞懂的是，为什么当时执行的时候不是报找不到PCRE库而是报找不到openssl文件目录。

这次debug, 感受就是一定要具体定位到错误产生的源头，一定要尽可能直接看执行编译的conf文件是怎么写的，弄清楚编译逻辑，知道错从何处产生，大胆尝试修改代码check问题，才可能快速寻找到解决办法。

