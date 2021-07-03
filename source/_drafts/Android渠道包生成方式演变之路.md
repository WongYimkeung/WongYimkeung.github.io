---
title: Android渠道包生成方式演变之路
tags: Android
---

### 前言

小强作为一名重度手机患者，梦想着有一天也能开发出一个属于自己的爆款应用，于是刚从大学毕业的他就投入到了轰轰烈烈的移动浪潮中，成为了一名光荣的Android开发攻城狮，从此走上了一条不归之路。

<!-- more -->

人物简介：老黄，资深项目经理；小强，技术开发萌新。

### 简单的任务

​	这天，小强被老黄叫到了会议室喝茶。

​	老黄：“怎么样，小强，来到公司还适应不？”

​	小强：“适应适应，现在正在熟悉项目代码。”

​	老黄：“挺好，项目代码可以慢慢熟悉，现在有个简单的任务交给你来实现以下，听说过渠道包吗？”

​	小强：“听说过，现在国内Android应用市场百家争鸣，百花齐放，为了标识用户是从哪个应用市场/渠道获取的，在发布APP的时候，会生成多个渠道包发布到对应的应用市场上去，APP上报数据的时候会把对应的渠道信息上传，这样子就可以统计用户都是从哪个应用市场/渠道下载我们的应用，方便后续的统计和运营等工作。”

​	老黄拍了拍小强的肩膀说：“很好，现在就是让你负责一下生成渠道包的这个事情，以后APP的打包就交给你啦！”

​	小强一脸激动地说道：“我一定会好好完成这个任务的！”

​	小强回到工位上就开始完成任务，他想起可以在`AndroidManifest.xml`文件中的`application`标签里面添加`meta-data`信息并使用占位符，然后在`app/build.gralde`文件中的`android`闭包里面添加`productFlavors`并对每个渠道都设置对应的渠道信息替换占位符，这样打包出来的每个渠道包的`AndroidManifest.xml`文件中就会包含对应的渠道信息，只需要读取这个文件获取渠道信息进行数据上报就可以了。“真是简单的任务啊”，小强想道。

对应实现方式如下：

```
// AndroidManifest.xml
<manifest ...>
	<application ...>
		<meta-data
			android:name="channel"
			android:value="${channelValue}" />
	</aplication>
</manifest>

// app/build.gradle
android {
		...
    flavorDimensions "defalut"
    productFlavors {
        xiaomi {
            manifestPlaceholders = [channelValue: "xiaomi"]
        }
        huawei {
            manifestPlaceholders = [channelValue: "huawei"]
        }
        oppo {
            manifestPlaceholders = [channelValue: "oppo"]
        }
        vivo {
            manifestPlaceholders = [channelValue: "vivo"]
        }
    }
}

// ChannelUtil.java
    /**
     * 从AndroidManifest文件中获取渠道信息
     *
     * @param context Android上下文
     * @return 渠道信息
     */
    private static String getChannelFromManifest(Context context) {
        ApplicationInfo applicationInfo = null;
        try {
            applicationInfo = context.getPackageManager()
                    .getApplicationInfo(context.getPackageName(), PackageManager.GET_META_DATA);
        } catch (PackageManager.NameNotFoundException e) {
            e.printStackTrace();
        }
        if (null == applicationInfo) {
            return "default";
        }

        Bundle metaDataBundle = applicationInfo.metaData;
        if (null == metaDataBundle) {
            return "default";
        }

        String channel = metaDataBundle.getString("channel");
        if (TextUtils.isEmpty(channel)) {
            return "default";
        }

        return channel;
    }
```

### 你能快点吗？

​	小强通过`Gradle`的方式实现了渠道包的打包之后，

每个渠道包都要打包一次才能生成，太慢了，能否快点？
反编译，脚本修改AndroidManifest.xnl，重新生成新包，再次签名

### 你能再快点吗？

虽然提高了速度，但是再次签名也是需要时间的，能否再快点？
找到V1不校验的地方，往META-INF里面添加渠道文件，重新生成新包即可，因为V1方式不会校验，所以不需要重新签名。

### 为什么渠道信息会丢失？

文件特别大的时候，读取META-INF下的文件也慢，过去渠道信息慢，导致数据已经上报了结果渠道信息还没获取。
V1 Apk也是Zip文件，添加内容到Comment块中不会影响校验，使用RandomAccessFile可以快速读书，也不需要重新签名。

### 都是Android 7.0惹得祸害

V2签名，对整个文件进行签名，修改一处也需要重新签名，导致无法安装。
build.gradle 设置不使用V2签名，真是个天才，要直面困难
找到V2签名不校验的地方，添加渠道信息。判断支持V1还是V2，选择对应写入方式。



### 总结

