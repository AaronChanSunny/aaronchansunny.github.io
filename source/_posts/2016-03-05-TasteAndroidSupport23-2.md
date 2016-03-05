---
title: Android Support23.2，带我们进入矢量图时代
date: 2016-03-05 18:02:07
tags: Android
---

## 1. 背景
2月底，谷歌发布了[Android Support Library 23.2](http://android-developers.blogspot.jp/2016/02/android-support-library-232.html)。最新的支持库特性包括以下：
- Support Vector Drawables and Animated Vector Drawables
- AppCompat DayNight theme
- Design Support Library: Bottom Sheets
- RecyclerView `WRAP_CONTENT `

接下来，让我们一步步建立示例代码库。<!--more-->
## 2. 全民矢量图时代
传统Android开发中，为了适配不同屏幕分辨率的手机，我们需要建立多个`drawable`目录，分别存放一套图标资源文件，使得App在不同手机上都达到最优的现实效果。

![](https://github.com/AaronChanSunny/TasteAndroidSupport23.2/blob/master/screenshot/2.PNG?raw=true)

如果引入矢量图，我们只需要定义一个文件就能适配所有屏幕，简直简单粗暴。我们先通过一个简单例子，体验一下矢量图的魅力。

### 2.1 新建工程
使用Android Studio创建一个新工程。为了引入最新支持库，需要在配置`app/build.gradle`：

```
// Gradle Plugin 1.5  
 android {  
   defaultConfig {  
     generatedDensities = []  
  }  

  // This is handled for you by the 2.0+ Gradle Plugin  
  aaptOptions {  
    additionalParameters "--no-version-vectors"  
  }  
 }  
```

如果你使用的是Android Studio2.0，则配置：

```
// Gradle Plugin 2.0+  
 android {  
   defaultConfig {  
     vectorDrawables.useSupportLibrary = true  
    }  
 }  
```

### 2.2 布局文件

```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:paddingBottom="@dimen/activity_vertical_margin"
    android:paddingLeft="@dimen/activity_horizontal_margin"
    android:paddingRight="@dimen/activity_horizontal_margin"
    android:paddingTop="@dimen/activity_vertical_margin"
    app:layout_behavior="@string/appbar_scrolling_view_behavior"
    tools:context="com.aaron.tasteandroidsupport.SimpleCompareActivity"
    tools:showIn="@layout/activity_simple_compare">

    <ImageView
        android:id="@+id/search"
        android:src="@drawable/ic_search_black_48dp"
        android:layout_width="200dp"
        android:layout_height="200dp"/>

    <ImageView
        android:id="@+id/vector_search"
        android:layout_below="@id/search"
        app:srcCompat="@drawable/ic_search_48dp"
        android:layout_width="200dp"
        android:layout_height="200dp"/>

</RelativeLayout>
```

> 对于矢量图，指定`src`需要使用`app:srcCompat`

### 2.3 效果对比

![](https://github.com/AaronChanSunny/TasteAndroidSupport23.2/blob/master/screenshot/1.PNG?raw=true)

可以看出，即使将ImageView的尺寸放大，矢量图也不会是真，这就是矢量图的意义。更重要的是，我们不再需要对图标做5套资源文件了，不仅仅减少了工作量，更重要是能减少最终发布时APK大小。

### 2.4 彩蛋

Android Studio已经集成Material Design所有图标，这里说明下使用方法：

选中`res`目录，鼠标右键`New`，选择`Vector Assert`，即可打开Vector Studio：

![](https://github.com/AaronChanSunny/TasteAndroidSupport23.2/blob/master/screenshot/3.PNG?raw=true)
