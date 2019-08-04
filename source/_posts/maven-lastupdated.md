---
title: 为什么有时候maven无法更新jar包
date: 2019-08-04 13:20:02
category: maven
tags: [maven, pom]
---

在使用maven管理pom项目的时候，多多少少会遇到一些奇葩的问题，比如网络良好，idea非离线状态，科学上网模式，maven就是死活无法更新jar包，很令人头大，无从下手找原因，当我们去查看maven本地仓库的时候，发现有些包里会多了一些以 .lastUpdated 结尾的文件，那么问题来了

#### 为什么maven仓库会出现这些文件? 又会导致什么问题?
> 在idea网络状态很差或者离线状态时，需要从远程仓库下载某些jar包到本地参库时，因网络差，无法下载，导致本地仓库出现 `xx.jar.lastUpdated` 或者 `xx.pom.lastUpdated` 文件(文件具体作用暂时不清楚)， 由于这些文件的存在，即使网络变好后，项目仍然报错， 无法重新下载需要的jar包

#### 解决方法:
> 方法非常简单粗暴，找到本地仓库对应jar包目录位置将 `.lastUpdated` 文件删除, 刷新项目重新下载即可


有的小伙伴可能会问，仓库那么多，总不会让我一个个找到然后删除吧，机智，下面给大家提供了两个一键删除脚本： 

###### 1、cleanLastUpdated.bat（windows版本）
```
rem 这里写你的仓库路径
set REPOSITORY_PATH=D:\Java\maven-repository
rem 正在搜索...
for /f "delims=" %%i in ('dir /b /s "%REPOSITORY_PATH%\*lastUpdated*"') do (
    echo %%i
    del /s /q "%%i"
)
rem 搜索完毕
pause
```
###### 2、cleanLastUpdated.sh（linux版本）
```
# 这里写你的仓库路径
REPOSITORY_PATH=~/Documents/tools/repository
echo 正在搜索...
find $REPOSITORY_PATH -name "*lastUpdated*" | xargs rm -fr
echo 搜索完
```

参考：[maven仓库中的LastUpdated文件生成原因及删除](https://blog.csdn.net/u011990675/article/details/80066897 )

