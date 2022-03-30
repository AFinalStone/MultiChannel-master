## 多渠道打包之动态修改App名称，图标，applicationId，版本号，添加资源

近来公司有需求，同一套代码，要打包N套APP，而且这些APP的软件名称，软件图标，applicationId，版本号，甚至主页都不一样。之前都是单次修改，单次打包，可随着需求越来越多，需要打的包也会越来越多，单次打包费时费力，很明显已经不再适合，于是研究了一下，使用gradle成功实现了需要的功能，打包过程也变的更为简单。

gradle是一个基于Apache Ant和Apache Maven概念的项目自动化建构工具。他可以帮助我们轻松实现多渠道打包的功能。

- **效果图**

![多渠道打包.gif](http://upload-images.jianshu.io/upload_images/2761423-45b3ea86b630ad7e.gif?imageMogr2/auto-orient/strip)

- **项目结构图**

![项目结构.png](http://upload-images.jianshu.io/upload_images/2761423-0ac8db9394a40b44.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- **项目结构中build.gradle的具体内容**

```groovy
plugins {
    id 'com.android.application'
}

android {
    compileSdk 28

    defaultConfig {
        applicationId "com.shi.androidstudio.multichannel.baidu"
        minSdk 21
        targetSdk 28
        versionCode 1
        versionName "1.0"
        multiDexEnabled true//支持dex分包

        manifestPlaceholders = [CHANNEL_VALUE: "baidu"]//AndroidManifest.xml 里渠道变量
        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
        flavorDimensions "default"//默认渠道
        resConfigs "en"//只保留默认的国际化标准

        ndk {//ndk的so库过滤
            abiFilters "armeabi"/*, "armeabi-v7a"*/, "arm64-v8a"
        }

    }

    // 使用签名文件进行签名的两种方式

    //第一种：使用gradle直接签名打包
    signingConfigs {
        config {
            storeFile file('keyTest.jks')
            storePassword '123456'
            keyAlias 'HomeKey'
            keyPassword '123456'
        }
    }
    //第二种：为了保护签名文件，把它放在local.properties中并在版本库中排除
    // ，不把这些信息写入到版本库中（注意，此种方式签名文件中不能有中文）
//    signingConfigs {
//        config {
//            storeFile file(properties.getProperty("keystroe_storeFile"))
//            storePassword properties.getProperty("keystroe_storePassword")
//            keyAlias properties.getProperty("keystroe_keyAlias")
//            keyPassword properties.getProperty("keystroe_keyPassword")
//        }
//    }

    //多渠道打包
    productFlavors {
        baidu {//QQ
            applicationId "com.shi.androidstudio.multichannel.baidu"
//             动态修改 常量 字段
            buildConfigField "String", "ENVIRONMENT", '"我是百度首页"'
//             修改 AndroidManifest.xml 里渠道变量
            manifestPlaceholders = [CHANNEL_VALUE: "baidu"]
        }
        qq {//百度
            applicationId "com.shi.androidstudio.multichannel.qq"
//             动态修改 常量 字段
            buildConfigField "String", "ENVIRONMENT", '"我是腾讯首页"'
//             修改 AndroidManifest.xml 里渠道变量
            manifestPlaceholders = [CHANNEL_VALUE: "qq"]
        }
        xiaomi {//小米
            applicationId "com.shi.androidstudio.multichannel.xiaomi"
//             动态修改 常量 字段
            buildConfigField "String", "ENVIRONMENT", '"我是小米首页"'
//             修改 AndroidManifest.xml 里渠道变量
            manifestPlaceholders = [CHANNEL_VALUE: "xiaomi"]
        }
    }

    buildTypes {
        debug {
            // debug模式下，显示log
            buildConfigField("boolean", "LOG_DEBUG", "true")
            //为已经存在的applicationId添加后缀
            applicationIdSuffix ".debug"
            // 为版本名添加后缀
            versionNameSuffix "-debug"
            // 不开启混淆
            minifyEnabled false
            // 不移除无用的resource文件
//            shrinkResources false
            // 使用config签名
            signingConfig signingConfigs.config

        }
        release {
            // release模式下，不显示log
            buildConfigField("boolean", "LOG_DEBUG", "false")
            // 为版本名添加后缀
            versionNameSuffix "-relase"
            // 不开启混淆
            minifyEnabled false
            // 移除无用的resource文件
//            shrinkResources true
            // 使用config签名
            signingConfig signingConfigs.config
            // 混淆文件位置
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }

    }
    //移除lint检测的error
    lintOptions {
        checkReleaseBuilds false
        abortOnError false
    }

    aaptOptions {
        //防止java.io.FileNotFoundException: This file can not be opened as a file descriptor; it is probably compressed
        noCompress 'mp3', 'aac', 'mp4', 'wav'
        cruncherEnabled false //用来关闭Android Studio的PNG合法性检查的，直接不让它检查。
        useNewCruncher(false)
    }


    compileOptions {//知名使用的JDK版本
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }

    // 批量打包
    applicationVariants.all { variant ->
        variant.outputs.each { output ->
            if (output.name == 'release') {
                variant.packageApplicationProvider.get().outputDirectory = new File(project.rootDir.absolutePath + "/app")
                //输出apk名称为：版本名_版本code_时间.apk
                def fileName = "MiChat_v${defaultConfig.versionName}_code${defaultConfig.versionCode}_t${releaseTime()}_release.apk"
                output.outputFileName = fileName
            }
        }
    }


}

dependencies {
    implementation 'androidx.appcompat:appcompat:1.3.0'
    implementation 'com.google.android.material:material:1.4.0'
}
```

- **项目结构中local.properties的具体内容**

```java
## This file must*NOT*be checked into Version Control Systems,
        # as it contains information specific to your local configuration.
        #
        # Location of the SDK.This is only used by Gradle.
        # For customization when using a Version Control System,please read the
        # header note.
        #Tue Mar 29 17:45:25CST 2022
        sdk.dir=D\:\\Android_SDK
        #对应自己实际的证书路径和名字，在这里由于签名文件是放在app目录下，因为没有写绝对路径。
        keystroe_storeFile=keyTest.jks
        keystroe_storePassword=123456
        keystroe_keyAlias=HomeKey
        keystroe_keyPassword=123456
```

看完build.gradle和local.properties的具体内容之后，我们再来挑选几个地方来具体说一下。

### 一. signingConfigs

在signingConfigs中主要是为打包配置签名文件具体信息的，这里我使用了两种方式，第一种方式把签名文件的位置，storePassword ，keyAlias，keyPassword
等具体内容都直接写在其中，然后使用gradle进行打包，第二种是通过通过使用local.properties文件来间接加载签名文件的具体信息。一般我们更倾向于第二种方法，这样有助于保护我们的签名文件（在local.properties中不能有中文）。

### 二. productFlavors

不同渠道的设置基本都是在 productFlavors 里设置的，在里面想要添加多少个渠道都可以。

### 修改app名称、图标

当我们在productFlavors 中添加了不同渠道环境名称之后，我们还可以mian文件夹同层级中建立和baidu，qq，xiaomi名称对应的文件夹，  
并放入特定的string.xml文件，图标文件，当然我们还可以放入其他color.xml、dimen.xml等资源文件，甚至AndroidManifest.xml都可以放入，  
Gradle在构建应用时，会优先使用flavor所属dataSet中的同名资源，这样就能达到不同环境不同软件图标的功能。

![修改软件图标.png](http://upload-images.jianshu.io/upload_images/2761423-3aa0b6c4326db3a7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

简单介绍就到这里了，做个笔记方便以后使用，如果"不小心"帮到别人了当然也是极好的了。
[最后附上github上的项目地址](https://github.com/AFinalStone/MultiChannel-master)以及[整个demo下载地址](http://download.csdn.net/detail/abc6368765/9650523)

### 最后再推荐一个批量打马甲包的脚本项目，使用该脚本可以起到批量修改app名字和logo的作用

### [点我进入传送门](https://github.com/AFinalStone/jiagu)