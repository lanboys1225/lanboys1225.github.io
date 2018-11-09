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

```
 AndroidManifest.xml
  //authorities 不一定是这种格式, 可自定义
        <provider
            android:name="android.support.v4.content.FileProvider"
            android:authorities="${applicationId}.updateFileProvide"
            android:exported="false"
            android:grantUriPermissions="true">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/file_paths"/>
        </provider>
  
  file_paths.xml
 <resources>
    <paths>
        <!--代表外部存储区域的根目录下的文件 Environment.getExternalStorageDirectory()/path 值-->
        <external-path name="root" path="."/>
		<!--/storage/emulated/0/--> 
        <external-path name="app_update" path="AppUpdate/apkdir" />
        <!--/storage/emulated/0/AppUpdate/apkdir--> 
        
        <!--代表app 私有的存储区域 Context.getFilesDir()目录下的images目录 -->
        <files-path name="private_files" path="imagesfiles" />
        <!--/data/user/0/com.xxx.xxx/files/imagesfiles-->
        
        <!--代表app 私有的存储区域 Context.getCacheDir()目录下的images目录-->
   		<cache-path name="private_cache" path="imagescache" />
   		<!-- /data/user/0/com.xxx.xxx/cache/imagescache-->
        
        <!--代表app 外部存储区域根目录下的文件 Context.getExternalFilesDir(Environment.DIRECTORY_PICTURES)目录下的Pictures目录-->
   		<external-files-path name="external_files" path="Pictures/dir" />
    	<!--/storage/emulated/0/Android/data/com.xxx.xxx/files/Pictures/dir-->
    	
    	
    
        <!--content:// com.xxx.xxx.fileprovider / app_update / xx.apk-->
        <!--content:// FileProvider的authority / name值 / xx.apk-->
        
        举例：
        File file = new File("/storage/emulated/0/AppUpdate/apkdir")
        Uri uri = FileProvider.getUriForFile(this, "com.xxx.fileprovider", file);
        
        file 根据file_paths.xml提供的键值对来匹配替换出 uri, file_paths.xml 就相当于一个替换表格
     </paths>
</resources>
  
  
  
  
  
```



 











