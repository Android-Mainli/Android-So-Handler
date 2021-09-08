# Android-So-Handler
**`com.android.tools.build:gradle`版本需要小于等于`3.4.0`才可使用SoFilePlugin插件,SoLoadHookPlugin插件不受影响
新版`3.5.0`以及`3.6.0`Transform已经无法处理so文件，后续尝试别的方式处理so文件，这里如果有好的办法欢迎提issues交流**

**减包工具集合 , 通过处理so库实现减包**

### 特点如下:
1. 在**不修改代码**的情况下,完成对指定so库(包含引入的aar里的so库)进行**压缩**,并**代理加载方法**,在**第一次加载时解压加载**等一些列操作
2. 支持删除指定so库 方便自定义云端下发 需要自己编写侵入代码
> 说解压不要侵入代码,其实还需要**一行初始化** ~_ ~! `AssetsSoLoadBy7zFileManager.init(getContext())`  后边考虑**隐藏**

### 数据对比:

仅仅在加载时通过记录的MD5判断是否存在，不存在解压，存在则跳过直接加载。
这里通过压缩时记录的MD5判断是否需要解压更新不依赖apk版本减少解压次数。

|      so库名称      | apk包中所占大小 | 7z极限压缩大小 | 解压后实际大小 | 解压耗时(毫秒) |
| :----------------: | :-------------: | :------------: | :------------: | :------------: |
|   RTCStatistics    |  1331kb(1.3M)   |     958kb      | 2752kb(2.68M)  |      109       |
| flutter(Debug版本) | 10,547kb(10.3M) |     6360kb     | 23358kb(22.8M) |      700       |
| bd_idl_pass_token  |      9.6k       |      8kb       |      17kb      |       3        |
|    idl_license     |     63.3kb      |      51kb      |     113kb      |       6        |
|      FaceSDK       |     269.5kb     |     220kb      |     450kb      |       25       |
|      turbonet      | 1,638.4kb(1.6M) |     1258kb     |     2737kb     |      167       |
|   gnustl_shared    |     273.7kb     |     195kb      |     693kb      |       28       |
|     xiaoyuhost     |     426.9kb     |     309kb      |   1009kb(1M)   |       48       |
|    crab_native     |     57.7kb      |      44kb      |     109kb      |       7        |

> **apk包中所占大小:** apk属于zip压缩 所以apk包中已经为zip压缩后大小
>
> **7z极限压缩大小:** 7z极限压缩大小是执行7z a xxx.7z  libxxx.so -t7z -mx=9 -m0=LZMA2 -ms=10m -mf=on -mhc=on -mmt=on -mhcf 压缩后大小
>
> **解压后实际大小:** 指so文件实际大小,AndroidStudio中文件大小
>
> **解压耗时(毫秒):** 统计手机为谷歌**Pixel 2XL** 骁龙**835**处理器

### 接入方式如下:

ps:配置较多全可走默认 ~_ ~!

```groovy
//1.根build.gradle中加入
buildscript {
    repositories {
        maven { url "https://raw.githubusercontent.com/IMFWorks/Android-So-Handler/master/maven" }
        ...
    }
    dependencies {
        ...
        classpath "com.imf.so:load-hook-plugin:${SO_PLUGIN_VERSION}" 
        classpath "com.imf.so:file-plugin:${SO_PLUGIN_VERSION}"
        ...
    }
}
allprojects {
    repositories {
      ...
      maven { url "https://raw.githubusercontent.com/IMFWorks/Android-So-Handler/master/maven" }
      ...
    }
}

//2.app工程中加入
dependencies {
	...
  implementation "com.imf.so:load-assets-7z:${SO_PLUGIN_VERSION}"
  ...
}

apply plugin: 'SoFilePlugin'
SoFileConfig {
    //设置debug下不删除与压缩so库
    excludeBuildTypes = ['debug']
    /**
     * 强制保留所有依赖
     * 默认为false时
     * minSdkVersion <= 23 保留所有依赖
     * minSdkVersion > 23  只保留deleteSoLibs与compressSo2AssetsLibs中处理过的依赖
     */
    forceNeededRetainAllDependencies = true
    //设置要删除的so库
    deleteSoLibs = [
    ]
    //设置要压缩的库 注意libun7zip.so 为7z解压库不可压缩
    //这里名字要是带有lib开头与.so结尾与apk中so库名称一致
    //如果使用7z压缩请确保7z命令加入到环境变量
    //mac推荐使用brew install p7zip进行安装
    //windows 去https://www.7-zip.org/下载安装，别忘记配置7z到环境变量中
    compressSo2AssetsLibs = [
      'libxxx.so'
    ]
    /**
     * 配置自定义依赖
     * 用于解决 liba.so 并未声明依赖 libb.so 并且内部通过dlopen打开libb.so
     * 或者反射System.loadLibrary等跳过hook加载so库等场景
     * 如果没有这种情况可以不添加该配置,配置结构为Map<String,List<String>>
     */
    customDependencies = [
            'liba.so': ['libb.so',...]
    ]
}

apply plugin: 'SoLoadHookPlugin'
SoLoadHookConfig {
		//是否跳过R文件与BuildConfig
		isSkipRAndBuildConfig = true
		//设置跳过的包名,跳过的包不去hook 修改后请先clean
		excludePackage = ['com.imf.test']
}
//3.初始化 
AssetsSoLoadBy7zFileManager.init(getContext());
```

> SO_PLUGIN_VERSION 目前版本 `0.0.3-SNAPSHOT`

## 插件介绍

### 一、 SoLoadHookPlugin插件

1. 通过Hook `System.loadLibrary` 与 `System.load`实现加载转发具体步骤如下:

   * 通过`ASM`框架对Hook目标类进行字节码修改具体为 `System.loadLibrary` 与 `System.load`修改成`SoLoadHook.loadLibrary`与`SoLoadHook.load` 
   * `SoLoadHook`可设置`SoLoadProxy`完成对外代理

    > `SoLoadHook`有默认实现只是调用`System.loadLibrary` 与 `System.load`

2. 具体接入步骤如下:

* gradle配置

```groovy
//build.gradle中只加入
classpath "com.imf.so:load-hook-plugin:${SO_PLUGIN_VERSION}" 
//app.gradle中只配置
apply plugin: 'SoLoadHookPlugin'
SoLoadHookConfig {
		//是否跳过R文件与BuildConfig
		isSkipRAndBuildConfig = true
		//设置跳过的包名,跳过的包不去hook 修改后请先clean
		excludePackage = ['com.imf.test.']
}
dependencies {
  implementation "com.imf.so:load-hook:${SO_PLUGIN_VERSION}"
}
```

* java代码实现`SoLoadProxy`完成加载


```java
public interface SoLoadProxy {
    void loadLibrary(String libName);

    void load(String filename);
}
SoLoadHook.setSoLoadProxy(new XXXSoLoadProxy())
```

> 实现SoLoadProxy类后不会被修改`System.loadLibrary`与`System.load`字节码
> 如果不想在指定包名下修改 在excludePackage中配置报名
> 如果不想在指定类或方法下被修改字节码,请添加注解@KeepSystemLoadLib

### 二、SoFilePlugin插件依赖SoLoadHookPlugin

1. 通过transform插件对so库进行7z压缩(利用压缩差完成压缩apk),压缩后放入`asstes`下的`jniLib`
2. 根据压缩或删除so情况生成`info.json`
3. 运行时进行解压加载 so

> 1. so库依赖拷贝了[ReLinker](https://github.com/KeepSafe/ReLinker)中解析代码
> 2. 解压部分微调自[AndroidUn7z](https://github.com/hzy3774/AndroidUn7zip)

**接入方式参考顶部最开始部分**

### 三、TODO
1. 尝试对比压缩工具 zstd 与 7z
2. 兼容[facebook/SoLoader](https://github.com/facebook/SoLoader)库解压加载
3. 优化多线程加载so逻辑
4. 针对deleteSoLibs中删除so后,自动上传云端与云端下发方案完成code
> ps:前期先出下载列表,用于启动app时下载,让云端下发方案先跑起来

