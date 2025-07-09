---
title: centos6升级nginx
published: 2025-07-09
description: '在老版本centos6系统中升级nginx版本'
image: ''
tags: ['centos6','nginx']
category: '记录'
draft: false 
lang: ''
---
## 安装新版本openssl
```shell
wget https://www.openssl.org/source/openssl-1.1.1w.tar.gz

tar -xzf openssl-1.1.1w.tar.gz

cd openssl-1.1.1w

./config --prefix=/usr/local/openssl-1.1.1w --openssldir=/usr/local/openssl-1.1.1t shared zlib
make -j 2
make install
```
## 安装新版本gcc环境
### 先更新还在支持centos6的yum.repo
```
sudo tee /etc/yum.repos.d/CentOS-Vault.repo <<EOF
[base]
name=CentOS-6 - Base
baseurl=http://vault.centos.org/6.10/os/x86_64/
enabled=1
gpgcheck=0

[updates]
name=CentOS-6 - Updates
baseurl=http://vault.centos.org/6.10/updates/x86_64/
enabled=1
gpgcheck=0

[extras]
name=CentOS-6 - Extras
baseurl=http://vault.centos.org/6.10/extras/x86_64/
enabled=1
gpgcheck=0

[centos-sclo-rh]
name=CentOS-6 - SCLo rh
baseurl=http://vault.centos.org/6.10/sclo/x86_64/rh/
enabled=1
gpgcheck=0

[centos-sclo-sclo]
name=CentOS-6 - SCLo sclo
baseurl=http://vault.centos.org/6.10/sclo/x86_64/sclo/
enabled=1
gpgcheck=0
EOF

sudo yum clean all
sudo yum makecache
```
### 安装devetoolset-9 为nginx编译启用gcc新版本
```
# 安装devtoolset-9
sudo yum install devtoolset-9 -y
# 启用 gcc 9
scl enable devtoolset-9 bash
# 验证
gcc -v

#如果没有切换成功执行：
source /opt/rh/devtoolset-9/enable

# 再次检测成功输出gcc version 9.1.1 20190605 (Red Hat 9.1.1-2) (GCC)
```

## 安装新版本luajit
```
### 最保险方式 → 用OpenResty官方LuaJIT2
重新编译 LuaJIT，用 openresty 维护的版本：

git clone https://github.com/openresty/luajit2.git
cd luajit2
make && make install PREFIX=/usr/local/luajit

sudo make install

# 完整设置环境变量
export LUAJIT_LIB=/usr/local/luajit/lib
export LUAJIT_INC=/usr/local/luajit/include/luajit-2.1
export LD_LIBRARY_PATH=/usr/local/luajit/lib:$LD_LIBRARY_PATH
export PATH=/usr/local/luajit/bin:$PATH
export PKG_CONFIG_PATH=/usr/local/luajit/lib/pkgconfig


# 检测是否支持FFI
luajit -e "local ffi = require('ffi'); print('FFI is available')"
```

## 安装新版本pcre
```
wget wget https://sourceforge.net/projects/pcre/files/pcre/8.45/pcre-8.45.tar.gz/download -O pcre-8.45.tar.gz

tar -xzf pcre-8.45.tar.gz

cd pcre-8.45

./configure --prefix=/usr/local/pcre-8.45

make && sudo make install
```

## 下载nginx-1.26.3
```shell
wget https://nginx.org/download/nginx-1.26.3.tar.gz
tar xzf nginx-1.26.3.tar.gz
cd nginx-1.26.3
mkdir exp
cd exp
```
## 下载扩展
```shell
# 在exp目录下
git clone -b v0.3.1 https://github.com/vision5/ngx_devel_kit.git

git clone -b v0.10.25 https://github.com/openresty/lua-nginx-module.git

```

## 编译
```
# 可以添加 -O3 -march=native 让nginx性能更好，但要考虑cpu架构的兼容性
-O3 -march=native

./configure \
--prefix=/usr/local/nginx-1.26.3 \
--with-http_v2_module \
--with-http_sub_module \
--with-http_stub_status_module \
--with-http_ssl_module \
--with-openssl=exp/openssl-1.1.1w/ \
--with-openssl-opt=enable-tls1_3 \
--with-http_auth_request_module \
--with-http_gzip_static_module \
--with-http_realip_module \
--add-module=exp/echo-nginx-module/ \
--add-module=exp/ngx_devel_kit/ \
--add-module=exp/lua-nginx-module/ \
--with-pcre=exp/pcre-8.45 \
--add-module=exp/set-misc-nginx-module/ \
--add-module=exp/ngx_cache_purge/ \
--add-module=exp/nginx-push-stream-module/ \
--add-module=exp/redis2-nginx-module/ \
--add-module=exp/ngx_http_consistent_hash/ \
--add-module=exp/nginx_upstream_check_module/ \
--with-cc-opt=-I/usr/local/luajit/include/luajit-2.1 \
--with-ld-opt=-L/usr/local/luajit/lib

# make的时候加上，编译出来的文件会变大，但能提升nginx的速度
make CFLAGS='-O3 -march=native'

```
### NDK 必须在 lua 模块前面添加

这是官方要求的顺序，NDK 是 lua-nginx-module 的依赖，顺序不能错：
```bash
--add-module=exp/ngx_devel_kit/ \
--add-module=exp/lua-nginx-module/ \
```

