---
title: 在ubuntu18.04下编译OpenJDK8
date: 2020-02-13 18:15:02
category: jvm
tags: [jvm,openjdk,ubuntu]
---
最近一直在折腾 jvm，想自己编译一番，可是在 Windows 上用 CLion 怎么也搞不定，CMakeLists.txt 不对导致各种报错，后面换到 centos 上终于编译成功了，但是又想调试，就用 CLion 的远程调试功能，虽然可以调试，但是因为网速或者其他不知道的原因导致调试速度太慢了，毕竟项目这么大而且还是远程调试，只好作罢，另寻他法。

想到周老师的《深入理解Java虚拟机》里有提到如何自己编译JDK, 于是参照周老师的方法终于编译成功了，还实现了调试功能，以后就能好好学习 jvm 源码了。

### 准备工作

- 系统：ubuntu18.04.2 (我是在虚拟机中安装的，也可以直接在电脑中安装, 官网下载即可)
- 调试软件：CLion2019.3.3 (官网下载即可)
- 下载 openjdk 源码

方法1. 可以通过Mercurial工具
```
hg clone http://hg.openjdk.java.net/jdk8u/jdk8u
cd jdk8u
sh get_source.sh
```
>ps: 一直报错，只好改用其他方式


方法2. 在 https://hg.openjdk.java.net/jdk8u/jdk8u/ 中, 点击左边的 “browse”, 然后再点你需要的格式 bz2 / zip / gz 进行下载

>ps: 方法是对的，但事实证明，如何下载都下载不完全，多次尝试，要么报错下载不下来，要么不报错下载下来的压缩文件只有几百k, 文件损坏无法解压，只好改用其他方式

方法3. 在github参库 https://github.com/unofficial-openjdk/openjdk 中把jdk8u/jdk8u分支拉下来就可以了

>ps: 真是的，下载个源码也那么费劲, 还是github好


### 搭建环境

###### 1. 安装一个jdk作为 bootstrap jdk

```
sudo apt-get install openjdk-8-jdk
```
###### 2. 安装编译调试工具(如在安装CLion时安装过了下面工具可直接跳到下一步)

```
sudo apt-get install gcc
sudo apt-get install g++
sudo apt-get install gdb
sudo apt-get install make
```
###### 3. 安装其他依赖包

```
sudo apt-get install ccache

sudo apt-get install libfreetype6-dev
sudo apt-get install libcups2-dev
sudo apt-get install libx11-dev libxext-dev libxrender-dev libxrandr-dev libxtst-dev libxt-dev
sudo apt-get install libasound2-dev
sudo apt-get install libffi-dev
sudo apt-get install autoconf
```

### 编译
> 编译一般没那么顺利，可提前看看下一小节的坑

1 . 在源码根目录下执行`./configure`命令，进行依赖项检查、参数配置和构建输出目录结构等工作，如果一切顺利的话，将会看到如下配置成功的提示。如果不顺利可按后面提示安装工具或依赖即可。

```
lanbing@ubuntu1804:~/openjdk/openjdk-jdk8u$ ./configure

Running generated-configure.sh
configure: Configuration created at Wed Feb 12 22:27:32 EST 2020.
configure: configure script generated at timestamp 1468207795.
checking for basename... /usr/bin/basename
checking for bash... /bin/bash
...
...
...
====================================================
A new configuration has been successfully created in
/home/lanbing/openjdk/openjdk-jdk8u/build/linux-x86_64-normal-server-release
using default settings.

Configuration summary:
* Debug level:    release
* JDK variant:    normal
* JVM variants:   server
* OpenJDK target: OS: linux, CPU architecture: x86, address length: 64

Tools summary:
* Boot JDK:       openjdk version "1.8.0_242" OpenJDK Runtime Environment (build 1.8.0_242-8u242-b08-0ubuntu3~18.04-b08) OpenJDK 64-Bit Server VM (build 25.242-b08, mixed mode)  (at /usr/lib/jvm/java-8-openjdk-amd64)
* C Compiler:     x86_64-linux-gnu-gcc-7 (Ubuntu 7.4.0-1ubuntu1~18.04.1) version 7.4.0 (at /usr/bin/x86_64-linux-gnu-gcc-7)
* C++ Compiler:   x86_64-linux-gnu-g++-7 (Ubuntu 7.4.0-1ubuntu1~18.04.1) version 7.4.0 (at /usr/bin/x86_64-linux-gnu-g++-7)

Build performance summary:
* Cores to use:   1
* Memory limit:   2963 MB
* ccache status:  installed, but disabled (version older than 3.1.4)

WARNING: The result of this configuration has overridden an older
configuration. You *should* run 'make clean' to make sure you get a
proper build. Failure to do so might result in strange build problems.
```

2 . 执行`make`命令进行编译，编译成功， 将会出现下面提示

```
----- Build times -------
Start 2020-02-17 12:00:21
End   2020-02-17 12:18:17
00:00:37 corba
00:11:06 hotspot
00:00:21 jaxp
00:00:30 jaxws
00:05:22 jdk
00:00:00 langtools
00:17:56 TOTAL
-------------------------
Finished building OpenJDK for target 'default'
```
3 . 在编译好的jdk目录`~/openjdk/openjdk-jdk8u/build/linux-x86_64-normal-server-release/jdk/bin`下执行`./java -version`命令进行验证
```
lanbing@ubuntu1804:~/openjdk/openjdk-jdk8u/build/linux-x86_64-normal-server-release/jdk/bin$ ./java -version

openjdk version "1.8.0-internal"
OpenJDK Runtime Environment (build 1.8.0-internal-lanbing_2020_02_17_11_43-b00)
OpenJDK 64-Bit Server VM (build 25.71-b00, mixed mode)
```

### 坑

###### 1. 编译内核版本问题

打开 /hotspot/make/linux/Makefile 文件
```
# /hotspot/make/linux/Makefile 第236行
SUPPORTED_OS_VERSION = 2.4% 2.5% 2.6% 3% 4%
后面加上 5% 变成
SUPPORTED_OS_VERSION = 2.4% 2.5% 2.6% 3% 4% 5%
```

从下面报错信息看到，系统内核版本为：`Linux ubuntu1804 5.3.0-28-generic` ，并不在上面支持列表中，所以在后面加上 `5%` 就可以了

```
lanbing@ubuntu1804:~/openjdk/openjdk-jdk8u$ make 

Building OpenJDK for target 'default' in configuration 'linux-x86_64-normal-server-release'

## Starting langtools
Compiling 2 files for BUILD_TOOLS
Compiling 32 properties into resource bundles
Compiling 782 files for BUILD_BOOTSTRAP_LANGTOOLS
Note: Some input files use or override a deprecated API.
Note: Recompile with -Xlint:deprecation for details.
Creating langtools/dist/bootstrap/lib/javac.jar
Compiling 785 files for BUILD_FULL_JAVAC
Note: Some input files use or override a deprecated API.
Note: Recompile with -Xlint:deprecation for details.
Creating langtools/dist/lib/classes.jar
Updating langtools/dist/lib/src.zip
## Finished langtools (build time 00:01:04)

## Starting hotspot

*** This OS is not supported: Linux ubuntu1804 5.3.0-28-generic #30~18.04.1-Ubuntu SMP Fri Jan 17 06:14:09 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
/home/lanbing/openjdk/openjdk-jdk8u/hotspot/make/linux/Makefile:242: recipe for target 'check_os_version' failed
/home/lanbing/openjdk/openjdk-jdk8u/hotspot/make/linux/Makefile:263: recipe for target 'linux_amd64_compiler2/debug' failed
Makefile:230: recipe for target 'generic_build2' failed
Makefile:177: recipe for target 'product' failed
HotspotWrapper.gmk:44: recipe for target '/home/lanbing/openjdk/openjdk-jdk8u/build/linux-x86_64-normal-server-release/hotspot/_hotspot.timestamp' failed
make[5]: *** [check_os_version] Error 1
make[4]: *** [linux_amd64_compiler2/debug] Error 2
make[3]: *** [generic_build2] Error 2
make[2]: *** [product] Error 2
make[1]: *** [/home/lanbing/openjdk/openjdk-jdk8u/build/linux-x86_64-normal-server-release/hotspot/_hotspot.timestamp] Error 2
/home/lanbing/openjdk/openjdk-jdk8u//make/Main.gmk:108: recipe for target 'hotspot-only' failed
make: *** [hotspot-only] Error 2

```
###### 2. -Werror=deprecated-declarations问题，什么原因导致的暂不清楚
打开 /hotspot/make/linux/makefiles/gcc.make 文件
```
# /hotspot/make/linux/makefiles/gcc.make 第200行
WARNINGS_ARE_ERRORS = -Werror
改成
WARNINGS_ARE_ERRORS = -Wno-all
```
按上面方式修改好文件，许多警告就会被忽略了，直到编译成功。

```
/home/lanbing/openjdk/openjdk-jdk8u/hotspot/src/os/linux/vm/os_linux.inline.hpp:127:42: error: 'int readdir_r(DIR*, dirent*, dirent**)' is deprecated [-Werror=deprecated-declarations]
   if((status = ::readdir_r(dirp, dbuf, &p)) != 0) {
                                          ^
In file included from /home/lanbing/openjdk/openjdk-jdk8u/hotspot/src/os/linux/vm/jvm_linux.h:44:0,
                 from /home/lanbing/openjdk/openjdk-jdk8u/hotspot/src/share/vm/prims/jvm.h:30,
                 from /home/lanbing/openjdk/openjdk-jdk8u/hotspot/src/share/vm/utilities/debug.hpp:29,
                 from /home/lanbing/openjdk/openjdk-jdk8u/hotspot/src/share/vm/runtime/globals.hpp:28,
                 from /home/lanbing/openjdk/openjdk-jdk8u/hotspot/src/share/vm/memory/allocation.hpp:28,
                 from /home/lanbing/openjdk/openjdk-jdk8u/hotspot/src/share/vm/memory/iterator.hpp:28,
                 from /home/lanbing/openjdk/openjdk-jdk8u/hotspot/src/share/vm/memory/genOopClosures.hpp:28,
                 from /home/lanbing/openjdk/openjdk-jdk8u/hotspot/src/share/vm/oops/klass.hpp:28,
                 from /home/lanbing/openjdk/openjdk-jdk8u/hotspot/src/share/vm/runtime/handles.hpp:28,
                 from /home/lanbing/openjdk/openjdk-jdk8u/hotspot/src/share/vm/memory/universe.hpp:28,
                 from /home/lanbing/openjdk/openjdk-jdk8u/hotspot/src/share/vm/code/oopRecorder.hpp:28,
                 from /home/lanbing/openjdk/openjdk-jdk8u/hotspot/src/share/vm/asm/codeBuffer.hpp:28,
                 from /home/lanbing/openjdk/openjdk-jdk8u/hotspot/src/share/vm/asm/assembler.hpp:28,
                 from /home/lanbing/openjdk/openjdk-jdk8u/hotspot/src/share/vm/precompiled/precompiled.hpp:29:
/usr/include/dirent.h:183:12: note: declared here
 extern int readdir_r (DIR *__restrict __dirp,
            ^~~~~~~~~
cc1plus: all warnings being treated as errors
make[6]: *** [precompiled.hpp.gch] Error 1
/home/lanbing/openjdk/openjdk-jdk8u/hotspot/make/linux/makefiles/vm.make:309: recipe for target 'precompiled.hpp.gch' failed
/home/lanbing/openjdk/openjdk-jdk8u/hotspot/make/linux/makefiles/top.make:119: recipe for target 'the_vm' failed
/home/lanbing/openjdk/openjdk-jdk8u/hotspot/make/linux/Makefile:298: recipe for target 'product' failed
Makefile:230: recipe for target 'generic_build2' failed
Makefile:177: recipe for target 'product' failed
HotspotWrapper.gmk:44: recipe for target '/home/lanbing/openjdk/openjdk-jdk8u/build/linux-x86_64-normal-server-release/hotspot/_hotspot.timestamp' failed
make[5]: *** [the_vm] Error 2
make[4]: *** [product] Error 2
make[3]: *** [generic_build2] Error 2
make[2]: *** [product] Error 2
make[1]: *** [/home/lanbing/openjdk/openjdk-jdk8u/build/linux-x86_64-normal-server-release/hotspot/_hotspot.timestamp] Error 2
/home/lanbing/openjdk/openjdk-jdk8u//make/Main.gmk:108: recipe for target 'hotspot-only' failed
make: *** [hotspot-only] Error 2
```

### 调试
1.我们用 CLion 进行调试，新建项目，选择 “New CMake Project from Sources”，然后再选源码根目录，CLion 帮我们选好了要导入的源码，直接点 `ok`(如图), 此时会自动创建 CMakeLists.txt, 不过不能使用，我们暂时也不需要使用到它

![](https://pic.downk.cc/item/5e4555dd48b86553eec4dd91.png)

2.在 `Run/Debug Configurations` 中编辑或者新增一个 CMake Application , Executable 中选择我们前面编译出来的 `java` 命令, arguments 加上 `-version`, 把下方 Before launch 中的 Build 任务去掉，如图

![](https://pic.downk.cc/item/5e45566a48b86553eec4f7b2.png)

3.在 `jdk/src/share/bin/java.c` 第 354 行的 `JavaMain()` 里设置断点，运行就可以进行调试了

![](https://pic.downk.cc/item/5e4556fa48b86553eec51033.png)
