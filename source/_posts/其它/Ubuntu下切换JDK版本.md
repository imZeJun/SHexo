---
title: Ubuntu下切换JDK版本
date: 2017-02-20 23:47
categories : 其它
---
# 一、从官网下载压缩包
## 1.1 下载压缩包
进入下面的地址：
```
http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html
```
选择对应的`Linux`版本和格式，这里我们选择`*.tar.gz`
![下载.png](http://upload-images.jianshu.io/upload_images/1949836-15215e2d33185f0f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 1.2 解压到目录
下载完之后，拷贝到`/usr/lib/jvm/`目录下：
```
sudo cp jdk-8u121-linux-x64.tar.gz /usr/lib/jvm/
```
进入到对应目录进行解压：
```
sudo tar zxvf ./jdk-8u121-linux-x64.tar.gz
```
## 1.3 修改环境变量
修改环境变量文件  `/etc/profile`
```
sudo gedit /etc/profile 
```
修改`JAVA_HOME`的值为之前解压完成后的目录：
```
export JAVA_HOME=/usr/lib/jvm/jdk1.8.0_121
```
使环境变量生效：
```
source /etc/profile
sudo update-alternatives --install /usr/bin/javah javah "/usr/lib/jvm/jdk1.8.0_121/bin/javah" 300
sudo update-alternatives --install /usr/bin/javac javac "/usr/lib/jvm/jdk1.8.0_121/bin/javac" 300
sudo update-alternatives --install /usr/bin/java java "/usr/lib/jvm/jdk1.8.0_121/bin/java" 300
sudo update-alternatives --install /usr/bin/javac javac "/usr/lib/jvm/jdk1.8.0_121/bin/javac" 300
```
## 1.4 切换
```
sudo update-alternatives --config java
sudo update-alternatives --config javac
sudo update-alternatives --config javah
sudo update-alternatives --config jar
```
