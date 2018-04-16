---
title: 为什么Android 7.0开始要使用FileProvider(应用之间共享文件)
---
###为什么要使用FileProvider?

Android 7.0 开始App对外无法暴露`file://xxx/xxx`类型的URI了, 如果你使用Intent携带这样的URI去打开外部App(比如：打开系统相机拍照)，那么会抛出FileUriExposedException异常。


官方给出解决这个问题的方案，就是使用FileProvider。

例子：apk安装流程

1. App内下载 apk 文件
2. 带着 apk 的 uri 地址调用系统安装apk的Activity
3. 系统安装apk的Activity 通过 uri 拿到apk文件 进行安装

问题就出在第3步，7.0开始通过  `Uri.fromFile(file)` 拿到的uri 已经不起作用了，需要通过 `FileProvider.getUriForFile(this, "com.xxx.fileprovider", file)` 拿到的uri才起作用。所以通常为了兼容7.0我们需要添加 FileProvider

apk安装代码如下：

```
            Intent intent = new Intent(Intent.ACTION_VIEW);
            intent.setFlags(FLAG_ACTIVITY_NEW_TASK);
            Uri uri;
            if (Build.VERSION.SDK_INT >= 24) {
            	//7.0开始的 uri: content://xxxxxxxx
                uri = FileProvider.getUriForFile(this, "com.xxx.fileprovider", file);
            } else {
            	//7.0之前的 uri: file://xxxxxxxx
                uri = Uri.fromFile(file);
            }
            intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
            intent.setDataAndType(uri, "application/vnd.android.package-archive");
            startActivityForResult(intent, REQUEST_CODE_INSTALL_APP);
```

###如何使用FileProvider?

我们就不在本文描述了，网上资源很多











