---
title: "本地编译Doris遇到的问题"
date: 2021-08-09T11:52:03+08:00
lastmod: 2021-08-09T11:52:03+08:00
keywords: ["doris"]
categories: ["doris"]
tags: ["doris"]
author: "小十一狼"
---

# 问题一：except-md5与actual-md5不一致

执行`sh build.sh`之后，会根据`thirdparty/vars.sh`去下载相应的包并进行校验，有时会提示except-md5与actual-md5不一致，错误类似如下：

```
Failed to download boost_1_64_0.tar.gz. attemp: 1
/root/work/incubator-doris/thirdparty/src/boost_1_64_0.tar.gz md5sum check failed!
except-md5 319c6ffbbeccc366f14bb68767a6db79
actual-md5 d41d8cd98f00b204e9800998ecf8427e  /root/work/incubator-doris/thirdparty/src/boost_1_64_0.tar.gz
Archive boost_1_64_0.tar.gz will be removed and download again.
Failed to download boost_1_64_0.tar.gz
Failed to download boost_1_64_0.tar.gz
```

这种情况下，如果我们确信下载的是完整的包，那么可以手动修改`thirdparty/vars.sh`中对应包的MD5SUM值。

# 问题二：DataTables.zip下载链接的问题

`thirdparty/vars.sh`中提供的DataTables.zip下载链接`https://datatables.net/download/builder?bs-3.3.7/jq-3.3.1/dt-1.10.24`不能下载到dt-1.10.24版本，将链接拷贝到浏览器上查询会提示：

```
Error: Only libraries of the current release version can be used by the . package builder. The library DataTables' s current release version is 1.10.25. Please reload the download builder page to have it use the latest libraries.
```

基于该提示，将`thirdparty/vars.sh`中DataTables的下载链接修改为：`https://datatables.net/download/builder?bs-3.3.7/jq-3.3.1/dt-1.10.25`，并修改`DATATABLES_SOURCE="DataTables-1.10.25"`。

另外发现docker镜像编译中已经针对该问题进行了修复，其`thirdparty/vars.sh`中关于DataTables的内容如下：

```
# datatables, bootstrap 3 and jQuery 3
# The origin download url is always changing: https://datatables.net/download/builder?bs-3.3.7/jq-3.3.1/dt-1.10.25
# So we put it in our own http server.
# If someone can offer an official url for DataTables, please update this.
DATATABLES_DOWNLOAD="https://doris-thirdparty-repo.bj.bcebos.com/thirdparty/DataTables.zip"
DATATABLES_NAME="DataTables.zip"
DATATABLES_SOURCE="DataTables-1.10.25"
DATATABLES_MD5SUM="c8fd73997c9871e213ee4211847deed5"
```

大家如果直接编译遇到问题二，可以直接使用上面的内容替换本地`thirdparty/vars.sh`该部分内容。

# 问题三：`sh build.sh`每次下载都从最新开始，无法续传

可以进入目录`thirdparty/src`，通过`wget -c [url]`连续下载，最后通过`md5sum`命令求出软件包的md5sum值，然后在`thirdparty/vars.sh`中修改对应的MD5SUM。

# 问题四：s2n-0.10.0.tar.gz软件包解压后目录名称与`thirdparty/vars.sh`中的AWS_S2N_SOURCE不一致

s2n-0.10.0.tar.gz解压后，目录是s2n-tls-0.10.0，我的方法是将修改目录名为s2n-0.10.0。另外一种方法则是修改AWS_S2N_SOURCE的值为s2n-tls-0.10.0。

看到docker镜像编译里，`AWS_S2N_SOURCE=s2n-tls-0.10.0`。

# 问题五：提示`FLAGS_log_filenum_quota`与`FLAGS_log_split_method`未在该范围内声明

错误信息如下：

```
[  1%] Building CXX object src/common/CMakeFiles/Common.dir/status.cpp.o
[  1%] Building CXX object src/common/CMakeFiles/Common.dir/resource_tls.cpp.o
[  1%] Building CXX object src/common/CMakeFiles/Common.dir/logconfig.cpp.o
/root/work/apache-doris-0.14.0-incubating-src/be/src/common/logconfig.cpp: In function 'bool doris::init_glog(const char*, bool)':
/root/work/apache-doris-0.14.0-incubating-src/be/src/common/logconfig.cpp:70:5: error: 'FLAGS_log_filenum_quota' was not declared in this scope
   70 |     FLAGS_log_filenum_quota = config::sys_log_roll_num;
      |     ^~~~~~~~~~~~~~~~~~~~~~~
/root/work/apache-doris-0.14.0-incubating-src/be/src/common/logconfig.cpp:101:9: error: 'FLAGS_log_split_method' was not declared in this scope
  101 |         FLAGS_log_split_method = "day";
      |         ^~~~~~~~~~~~~~~~~~~~~~
/root/work/apache-doris-0.14.0-incubating-src/be/src/common/logconfig.cpp:104:9: error: 'FLAGS_log_split_method' was not declared in this scope
  104 |         FLAGS_log_split_method = "hour";
      |         ^~~~~~~~~~~~~~~~~~~~~~
/root/work/apache-doris-0.14.0-incubating-src/be/src/common/logconfig.cpp:107:9: error: 'FLAGS_log_split_method' was not declared in this scope
  107 |         FLAGS_log_split_method = "size";
      |         ^~~~~~~~~~~~~~~~~~~~~~
make[2]: *** [src/common/CMakeFiles/Common.dir/build.make:119：src/common/CMakeFiles/Common.dir/logconfig.cpp.o] 错误 1
make[1]: *** [CMakeFiles/Makefile2:540：src/common/CMakeFiles/Common.dir/all] 错误 2
make: *** [Makefile:147：all] 错误 2
```

观察错误，发现是glog没有打上patch，阅读`third_party/download-thirdparty.sh`，发现对于patch是否被打上的判断依据是是否存在文件`patched_mark`，于是删除目录下所有的`patched_mark`，重新进行编译，问题解决。

# 问题六：npm与node版本老旧

老旧的npm和node在编译过程中，会报js的SyntaxError。

需要升级到最新的npm和node，可在`https://nodejs.org/dist/`查找最新版本，如当前是node-v16.6.1-linux-x64。
