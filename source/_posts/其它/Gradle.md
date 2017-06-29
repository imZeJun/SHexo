---
title: Gradle
date: 2017-02-20 20:55
categories : 其它
---
# 一、统一管理依赖版本
## 1.1 在根目录下，新建`config.gradle`文件
```
ext {

    android = [
            compileSdkVersion: 23,
            buildToolsVersion: "23.0.3",
            applicationId: "com.example.lizejun.repogradle",
            minSdkVersion: 14,
            targetSdkVersion: 23,
            versionCode: 1,
            versionName: "1.0",
            testInstrumentationRunner: "android.support.test.runner.AndroidJUnitRunner"
    ]

    dependencies = [
            "support-v4"  : 'com.android.support:support-v4:23.2.0',
            "support-v7"  : 'com.android.support:appcompat-v7:23.2.0'
    ]

}
```
`android`用来管理`SDK`版本、版本号等，`dependencies`用来管理依赖库的版本，它们都是一个数组，用逗号来分割，数组的每个元素都是`key:value`的格式。
## 1.2 在根目录下的`build.gradle`，新增：
```
apply from: "config.gradle"
```
## 1.3 在根目录下的各个`Module`或者`Library`中的`build.gradle`文件中，引用对应的`key`：
```
android {
    compileSdkVersion rootProject.ext.android.compileSdkVersion
    buildToolsVersion rootProject.ext.android.buildToolsVersion
    defaultConfig {
        applicationId rootProject.ext.android.applicationId
        minSdkVersion rootProject.ext.android.minSdkVersion
        targetSdkVersion rootProject.ext.android.targetSdkVersion
        versionCode rootProject.ext.android.versionCode
        versionName rootProject.ext.android.versionName
        testInstrumentationRunner rootProject.ext.android.testInstrumentationRunner
    }
}

dependencies {
    compile rootProject.ext.dependencies["support-v7"]
}
```
# 二、查看依赖关系
## 2.1 工具查看
依赖树的查看可以通过`Android Studio`的插件`Gradle View`来查看：
![Gradle View.png](http://upload-images.jianshu.io/upload_images/1949836-a1aa8635f689f527.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
重启之后，点击`View -> Tool Windows -> Gradle View`，之后会在最下方生成一个窗口，我们通过这个窗口就可以看到对应项目的依赖关系：
![Gradle View Result.png](http://upload-images.jianshu.io/upload_images/1949836-2845695076cbd1a7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.2 命令查看
除此之外，我们也可以使用以下的命令：
```
./gradlew -q app:dependencies
```
同样可以获得类似的结果：
![命令.png](http://upload-images.jianshu.io/upload_images/1949836-61688bffb5000d3f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
注意上图，如果标记了`(*)`，那么表示这个依赖被忽略了，这是因为其他顶级依赖中也依赖了这个传递的依赖，`Gradle`会自动分析下载最合适的依赖。

# 三、标识符
由于多个顶级依赖当中，可能包含了相同的子依赖，但是它们的版本不同，这时候为了选择合适的版本，那么就需要使用一些必要的操作符来管理子依赖。
## 3.1 `Transitive`
默认为`true`，表示`gradle`自动添加子依赖项，形成一个多层树形结构；设置为`false`，则需要手动添加每个依赖项。
### 3.1.1 统一指定`Transitive`
```
configurations.all {
    transitive = false
}

dependencies {
    compile rootProject.ext.dependencies["support-v7"]
}
```
最终得到的结果是，可以看到前面的子依赖项都没有了：
![2017-02-09 15:27:38屏幕截图.png](http://upload-images.jianshu.io/upload_images/1949836-7b6b2ca32994cc69.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 3.1.2 单独指定`Transitive`
```
dependencies {
    compile ('com.android.support:appcompat-v7:23.2.0') {
        transitive = false
    }
}

```

## 3.2 `Force`
用来表示强制设置某个模块的版本：
```
configurations.all {
    resolutionStrategy {
        force 'com.android.support:support-v4:23.1.0'
    }
}
```
之后打印，发现其依赖的包版本变成了`23.1.0`：
![Force.png](http://upload-images.jianshu.io/upload_images/1949836-45686469a80385a6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 3.3 `exclude`
该标识符用来设置不编译指定的模块
### 3.3.1 全局配置某个模块
```
configurations {
    all *.exclude group : 'com.android.support', module: 'support-v4'
}
```
此时的依赖关系为：
![2017-02-09 15:54:02屏幕截图.png](http://upload-images.jianshu.io/upload_images/1949836-dc4456a9e09bb3d1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
当然，`exclude`后的参数有`group`和`module`，可以分别单独使用，会排除所有匹配项。
### 3.3.2 单独给某个模块`exclude`
```
    compile ('com.android.support:appcompat-v7:23.2.0') {
        exclude group : 'com.android.support', module: 'support-v4'
    }
```
也会和上面获得相同的结果

# 四、版本冲突
## 4.1 相同配置下的版本冲突
同一配置下，某个模块的不同版本同时被依赖时，默认使用最新版，`Gradle`同步时不会报错。

# 五、`Gradle`详解
基本配置：`AS`中的`Android`项目通常至少包含两个`build.gradle`，一个是`Project`范围的，另一个是`Module`范围的，由于一个`Project`可以有多个`Module`，所以每个`Module`下都会对应一个`build.gradle`。
## 5.1 `Project`下的`build.gradle`
```
// Top-level build file where you can add configuration options common to all sub-projects/modules.
apply from: "config.gradle"

buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:1.3.1'
    }
}

allprojects {
    repositories {
        jcenter()
    }
}

task clean(type: Delete) {
    delete rootProject.buildDir
}
```
- `buildscript`下的`repositories`是`gradle`脚本自身需要的资源，
- `allprojects`下的`repositories`是项目所有模块需要的资源。

## 5.2 模块的`build.gradle`
```
//声明插件,表明这是一个Android程序;如果是库,那么应当是com.android.library
apply plugin: 'com.android.application'
//Android构建过程需要配置的参数
android {
    //编译版本
    compileSdkVersion rootProject.ext.android.compileSdkVersion
    //buildTool版本
    buildToolsVersion rootProject.ext.android.buildToolsVersion
    //默认配置,同时应用到debug和release版本上
    defaultConfig {
        applicationId rootProject.ext.android.applicationId
        minSdkVersion rootProject.ext.android.minSdkVersion
        targetSdkVersion rootProject.ext.android.targetSdkVersion
        versionCode rootProject.ext.android.versionCode
        versionName rootProject.ext.android.versionName
        testInstrumentationRunner rootProject.ext.android.testInstrumentationRunner
    }
    //配置debug和release版本的一些参数,例如混淆,签名配置等
    buildTypes {
        //release版本
        release {
            minifyEnabled false //是否开启混淆
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro' //混淆文件位置
        }
    }
}

dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    compile rootProject.ext.dependencies["support-v7"]
    androidTestCompile('com.android.support.test.espresso:espresso-core:2.2.2', {
        exclude group: 'com.android.support', module: 'support-annotations'
    })
    testCompile 'junit:junit:4.12'
}
```
下面对`Android`构建过程中需要配置的参数做一些解释：
- `compileSdkVersion`：告诉`gradle`用那个`Android SDK`的版本编译你的应用，修改它不会改变运行时的行为，因为它不会被包含进入最终的`APK`中；因此，推荐使用最新的`SDK`编译；如果使用`Suppport Library`，那么`compileSdkVersion`必须要大于等于它的大版本号。
- `minSdkVersion`：应用最低可运行的要求；它必须要大于等于你所依赖的库的`minSdkVersion`；
- `targetSdkVersion`：`Android`提供向前兼容的重要依据。（`targetSdkVersion is the main way Android provides forward compatibility`）
因为随着`Android`系统的升级，某个`api`或者模块的行为有可能发生改变，但是为了保证老`APK`的行为和以前兼容，只要`APK`的`targetSdkVersion`不变，那么即使这个`APK`安装在新的`Android`系统上，那么行为还是保持老的系统上的行为。
系统在调用某个`api`或者模块的时候，会先检查调用的`APK`的`targetSdkVersion`，来决定执行什么行为。

`minSdkVersion`和`targetSdkVersion`最终会被包含进入最终的`APK`文件中，如果你查看生成的`AndroidManifest.xml `，那么会发现：
```
<uses-sdk android:targetSdkVersion="23" android:minSdkVersion="7" />
```
所以，我们一般遵循的规则是：
```
minSdkVersion (lowest possible) <= targetSdkVersion == compileSdkVersion (latest SDK)
```
## 5.3 `gradle`相关文件
- `gradle.properties`
配置文件，可以定义一些常量供`build.gradle`使用，比如可以配置签名相关信息，例如`keystore`位置、密码、`keyalias`等。
- `settings.gradle`
用来配置多模块的，如果你的项目有两个模块`Browser`和`ScannerSDK`，那么就需要：
```
include ':Browser'
include ':ScannerSDK'
project(':ScannerSDK').projectDir = new File(settingsDir, './ScannerSDK/')
```
- `gradle`文件夹
里面有两个文件，`gradle-wrapper.jar`和`gradle-wrapper.properties`，后者当中指定了`gradle`的版本。
```
distributionUrl=https\://services.gradle.org/distributions/gradle-2.8-all.zip
```
- `gradlew`和`gradlew.bat`
分别是`Linux`和`windows`下的批处理文件，它们的作用是根据`gradle-wrapper.properties`文件中的`distributionUrl`下载对应的`gradle`版本，这样即使环境没有安装`gradle`也可以编译。

## 5.4 `gradle`仓库
`gradle`有三种仓库：`maven/ivy/flat本地`。
```
maven{
    url  "..."
}
ivy{
    url  "..."
}
flatDir{
    dirs 'xxx'
}
```
有些仓库取了别名：
```
repositories{
    mavenCentral()
    jcenter()
    mavenLocal()
}
```

## 5.5 `gradle`任务
通过以下指令，可以看到支持的任务：
```
./gradlew tasks
```
- 
![Task1.png](http://upload-images.jianshu.io/upload_images/1949836-34ec9100c3d8e536.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 
![Task2.png](http://upload-images.jianshu.io/upload_images/1949836-216b12a96ed813c9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 六、常见问题
## 6.1 导入本地`Jar`包
导入单个`jar`文件：
```
compile files('libs/xxx.jar')
```
导入`libs`的多个`jar`文件：
```
compile fileTree(dir: 'libs', include: ['*.jar'])
```
## 6.2 导入`maven`库
```
compile 'groupId:artifactId:version'
```
## 6.3 导入`Project`
在导入之前，需要在`settings.gradle`中把模块包含进来，例如前面的`ScannerSDK`模块：
```
compile project(':ScannerSDK')
```
## 6.4 声明第三方`maven`库
如果项目中需要的一些库文件不再中央仓库中，而是在某个特定地址里，那么就需要在`Project`中的`build.gradle`中的`allprojects`结点下或者直接配到某个模块中：
```
allprojects {
    repositories {
        maven {
            url '地址'
        }
    }
}
```
## 6.5 依赖第三方`aar`
```
compile 'com.aaa.xxx:core:1.0.1@aar'
```
## 6.6 将库导出为`aar`
首先，你的项目必须是一个库项目，之后在`build.gradle`中进行配置：
```
apply plugin : 'com.android.library'
```
之后，进入到该项目中，执行`gradle`命令：
```
./gradlew assembleRelease
```
生成的`aar`放在`/build/output/aar`文件当中

## 6.7 引用本地`aar`
首先，将`aar`文件拷贝到对应目录下，然后在该模块的`build.gradle`中声明`flat`仓库：
```
repositories{
   flatDir{
      dirs '以build.gradle为根目录的相对路径'
   }
}
```
之后，在`dependencies`结点下依赖该`aar`模块：
```
dependencies{
    compile (name:'xxx',ext:'aar')
}
```

# 七、多版本打包
在此之前，我们先了解几个基本的概念：
- `buildTypes`：**构建类型**，`Android Studio`的`Gradle`组件默认提供了`debug`和`release`两个默认配置，这里主要用于是否需要混淆，是否调试。
- `productFlavors`：**产品渠道**，在实际发布中，根据不同渠道，可能需要使用不同的包名，甚至是不同的代码。
- `buildVaiants`：每个`buildTypes`和`productFlavors`组成一个`buildvariant`。

以上我们讨论的`buildTypes`和`productFlavors`可以通过每个`Module`中的`build.gradle`的`android`标签来配置。

下面，我们先看一下不同的`productFlavors`，分别用来支持读取不同的文件和替换不同的字符串。

## 7.1 引用不同的代码
我们首先在`app/src`目录下新建两个目录，分别对应两个`Flavor`，再在其中建立相同名字的文件`Constant.java`，对里面的某个常量赋予不同的值。
![Flavor.png](http://upload-images.jianshu.io/upload_images/1949836-d044465fbabe92a8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- `constantFlavor1`：

```
package com.example.lizejun.repogradle;

public class Constant {
    public static final String NAME = "flavor1";
}
```
- `constantFlavor2`：

```
package com.example.lizejun.repogradle;

public class Constant {
    public static final String NAME = "flavor2";
}
```
我们的`app`下的`build.gradle`中的`android`标签下添加如下几行代码，让两个`Flavor`分别引用不同的`Constant`文件：
```
    sourceSets {
        constantFlavor1 {
            java.srcDirs =  ['src/constantFlavor1', 'src/constantFlavor1/java', 'src/constantFlavor1/java/']
        }
        constantFlavor2 {
            java.srcDirs =  ['src/constantFlavor2', 'src/constantFlavor2/java', 'src/constantFlavor2/java/']
        }
    }

    productFlavors {
        constantFlavor1 {}
        constantFlavor2 {}
    }
```
之后我们进行打包，可以得到以下不同安装包，这两个`apk`如果在其中引用的了`NAME`，那么它会得到不同的值：

![2017-02-10 16:15:51屏幕截图.png](http://upload-images.jianshu.io/upload_images/1949836-4dd8447145078936.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 7.2 自定义`buildConfig`类
如果我们只需要定义一些简单的值，那么我们可以用`buildConfig`类：
```
    productFlavors {
        constantFlavor1 {
            buildConfigField "boolean", "BOOLEAN_VALUE", "true"
        }
        constantFlavor2 {
            buildConfigField "boolean", "BOOLEAN_VALUE", "false"
        }
    }
```
## 7.3 占位符
```
    productFlavors {
        constantFlavor1 {
            buildConfigField "boolean", "BOOLEAN_VALUE", "true"
            manifestPlaceholders = [label:"constantFlavor1"]
        }
        constantFlavor2 {
            buildConfigField "boolean", "BOOLEAN_VALUE", "false"
            manifestPlaceholders = [label:"constantFlavor2"]
        }
    }
```
之后，我们再在`AndroidManifest.xml`中引用它：
```
android:label="${label}"
```

## 7.4 签名配置
首先是在`android`标签下，我们使用`signingConfigs`来配置不同的签名类型
```
    signingConfigs {
        eng {
            keyAlias 'androiddebugkey'
            keyPassword 'android'
            storeFile file('debug.keystore')
            storePassword 'android'
        }
        prd {
            keyAlias 'androiddebugkey'
            keyPassword 'android'
            storeFile file('debug.keystore')
            storePassword 'android'
        }
    }
```
之后，再在`buildTypes`的各个分支中引用对应的签名配置：
```
    buildTypes {
        debug {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
            signingConfig signingConfigs.eng
        }
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
            signingConfig signingConfigs.prd
        }
    }
```
## 7.5 依赖
有时候，我们需要在不同的`buildTypes`下，引用不同的依赖，例如内存泄露的检测工具，我们希望在`debug`版本时检查内存泄露，并在发生时在桌面上生成图标，但是在`release`版本上我们并不希望这么做，这时候我们可以这么写：
```
debugCompile 'com.squareup.leakcanary:leakcanary-android:1.5'
releaseCompile 'com.squareup.leakcanary:leakcanary-android-no-op:1.5'
```
这样，`release`版本就不会在桌面生成内存泄露的图标。
