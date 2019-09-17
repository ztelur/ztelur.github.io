---
title: Android文件存储位置简述
tags:
  - 文件系统
categories:
  - Android
abbrlink: b17c24a6
date: 2015-12-12 17:11:54
---

&emsp;最近一段时间,工作和学习方面都比较忙,所以,博客方面有一段时间没有投入时间啦,今天学习了一下android文件存储方面的知识,主要是`Internal Storage`和`External Storage`的相关特性.主要知识来自android的官方文档和其他人的博客.
#### Internal Storage
&emsp;一般来说,你可以直接存储文件在机器的internal storage中,存储在这个位置的文件是私有的,其他应用无法获得.但是当用户卸载你的应用时,文件就被删除啦.
>通过`openFileOutput()`传入文件的名字和操作模式,就可以获得`FileOutputStream`,然后就可以`write()`,然后`close`啦.
``` java
String FILENAME = "hello_file";
String string = "hello world!";

FileOutputStream fos = openFileOutput(FILENAME, Context.MODE_PRIVATE);
fos.write(string.getBytes());
fos.close();
```
> `MODE_PRIVATE`模式会创建或者替换同名文件,并让文件变为私有的,其他的一些模式还有`MODE_APPEND`(追加模式),`MODE_WORLD_READABLE`(全局可读)和`MODE_WORLD_WRITEABLE`(全局可写).
> 通过`OpenFileInput()`函数可以进行文件的读取.

&emsp;Android的内部存储路径为/data/data/packagename/,加入你的应用名为com.example.test,那么这个路径就为/data/data/com.example.test,这个路径下一般会有files,cache和你自己生成的文件夹.那么如下的操作返回的路径如下

- Context.getFileDir(),获得/data/data/com.example.test/files这个文件夹的File对象
- Context.openFileInput()和Context.openFileOutput,读取的是files文件夹下的文件
- Context.fileList(),返回的是files下的所有文件名
- Context.deleteFile(),删除files下指定名称的文件
- Context.getCacheDir(),该方法返回的是/data/data/com.example.test/cache的File对象.当Android的内部存储容量过低时,android会自动清除缓存文件.
- getDir(String name,int mode),返回的是/data/data/com.example.test/下指定名称的文件夹的File对象

#### External Storage
&emsp;所有android设备都会提供外部存储,你可以用来保存文件,但是存储在外部存储的文件是完全公开的,并且可以被用户修改,外部存储可能无法获得,并且存储的文件的安全性很低,会被修改或者删除.
&emsp;获得	外部存储的访问权必须需要申请`READ_EXTERNAL_STORAGE`或者`WRITE_EXTERNAL_STORAGE`权限,如果申请了写权限,那么相应的读权限也获得啦.如果是Android 6.0,那么权限的申请可能就更加麻烦 :[6.0新的权限管理系统](http://jijiaxin89.com/2015/08/30/Android-s-Runtime-Permission/)
&emsp;在使用外部存储设备之前,必须先检查外部存储设备的挂载情况,然后再进行文件操作
``` java
/* Checks if external storage is available for read and write */
public boolean isExternalStorageWritable() {
    String state = Environment.getExternalStorageState();
    if (Environment.MEDIA_MOUNTED.equals(state)) {
        return true;
    }
    return false;
}

/* Checks if external storage is available to at least read */
public boolean isExternalStorageReadable() {
    String state = Environment.getExternalStorageState();
    if (Environment.MEDIA_MOUNTED.equals(state) ||
        Environment.MEDIA_MOUNTED_READ_ONLY.equals(state)) {
        return true;
    }
    return false;
}
```
&emsp;如果你想存储一些可以和其他应用共享的文件时,一般存储在共享的文件夹中,比如`Music/`,'Pictures/','Ringtones/'通过`Environment.getExternalStoragePublicDirectory`,传递给其文件夹的类型,比如`DIRECTORY_MUSIC`,`DIRECTORY_PICTURES`,就可以获得响应文件夹的File对象.
&emsp;当你不想其他应用读取你的文件时,你可能需要使用私有文件夹.通过`getExternalFilesDir()`,并传递给其子文件夹的type,就可以打开响应的文件夹,在4.4之后,读取私有文件夹下的文件,是不需要外部存储设备权限的.
&emsp;外部存储设备的路径一般都以/mnt/sdcard开始,如下的一些函数获得路径如下:
- getExternalCacheDir() 获得/mnt/sdcard/Android/data/com.example.test/cache 文件夹的File对象
- getExternalFilesDir(type)获得/mnt/sdcard/Android/data/com.example.test/files文件夹下响应子文件夹的File对象
- Environment.getExternalStorageDiretory() 获得的是/mnt/sdcard文件夹的File对象
- Environment.getDataDirectory() 获得是的/data文件夹的File对象,需要注意的是,/data/data/Android/就是内部存储文件夹啦.
- Environment.getDownloadCacheDirectory() 获得的是/cache文件夹的File对象.







参考文章:
- [android 系统文件路径.sdcard路径.外部路径](http://blog.csdn.net/ljh347917444/article/details/16984199) 
- [Android Doc](http://developer.android.com/intl/zh-cn/guide/topics/data/data-storage.html#filesInternal)
