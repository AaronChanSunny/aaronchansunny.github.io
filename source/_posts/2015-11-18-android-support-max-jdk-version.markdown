---
layout: post
title: "Android添加Java依赖库遇到的一个问题"
date: 2015-11-18 17:02:07 +0800
comments: true
tags: Android
---

最近尝试在Android中引入MVP架构，其中Model作为一个独立的Android Studio模块，是以Java工程的形式存在的。<!--more-->单独编译运行这个工程并没有问题，但最后在app中引入时却一直报如下错误：

```
UNEXPECTED TOP-LEVEL EXCEPTION:
java.lang.RuntimeException: Exception parsing classes
	at com.android.dx.command.dexer.Main.processClass(Main.java:752)
	at com.android.dx.command.dexer.Main.processFileBytes(Main.java:718)
	at com.android.dx.command.dexer.Main.access$1200(Main.java:85)
	at com.android.dx.command.dexer.Main$FileBytesConsumer.processFileBytes(Main.java:1645)
	at com.android.dx.cf.direct.ClassPathOpener.processArchive(ClassPathOpener.java:284)
	at com.android.dx.cf.direct.ClassPathOpener.processOne(ClassPathOpener.java:166)
	at com.android.dx.cf.direct.ClassPathOpener.process(ClassPathOpener.java:144)
	at com.android.dx.command.dexer.Main.processOne(Main.java:672)
	at com.android.dx.command.dexer.Main.processAllFiles(Main.java:574)
	at com.android.dx.command.dexer.Main.runMonoDex(Main.java:311)
	at com.android.dx.command.dexer.Main.run(Main.java:277)
	at com.android.dx.command.dexer.Main.main(Main.java:245)
	at com.android.dx.command.Main.main(Main.java:106)
Caused by: com.android.dx.cf.iface.ParseException: bad class file magic (cafebabe) or version (0034.0000)
	at com.android.dx.cf.direct.DirectClassFile.parse0(DirectClassFile.java:472)
	at com.android.dx.cf.direct.DirectClassFile.parse(DirectClassFile.java:406)
	at com.android.dx.cf.direct.DirectClassFile.parseToInterfacesIfNecessary(DirectClassFile.java:388)
	at com.android.dx.cf.direct.DirectClassFile.getMagic(DirectClassFile.java:251)
	at com.android.dx.command.dexer.Main.parseClass(Main.java:764)
	at com.android.dx.command.dexer.Main.access$1500(Main.java:85)
	at com.android.dx.command.dexer.Main$ClassParserTask.call(Main.java:1684)
	at com.android.dx.command.dexer.Main.processClass(Main.java:749)
	... 12 more
1 error; aborting
Error:Execution failed for task ':app:preDexDebug'.
> com.android.ide.common.process.ProcessException: org.gradle.process.internal.ExecException: Process 'command '/Library/Java/JavaVirtualMachines/jdk1.8.0_45.jdk/Contents/Home/bin/java'' finished with non-zero exit value 1
```

关键信息在`java.lang.RuntimeException: Exception parsing classes`和`/Library/Java/JavaVirtualMachines/jdk1.8.0_45.jdk/`。可以发现，在类解析的过程中出错了，由于单独运行模块正常，初步判定是JDK版本问题。在`build.gradle`中显式指定JDK版本为1.7:

```
apply plugin: 'java'

sourceCompatibility = 1.7
targetCompatibility = 1.7

dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])

    compile 'com.squareup.retrofit:retrofit:2.0.0-beta2'
    compile 'com.squareup.retrofit:converter-gson:2.0.0-beta2'
}
```

在SO找到了[相应解答](http://stackoverflow.com/questions/30421389/android-studio-lambda-does-not-work)，之前一直没有注意，原来Android SDK支持的JDK最大版本只有到1.7。
