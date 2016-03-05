---
layout: post
title: "如何在Android Service中开启对话框"
date: 2015-05-13 09:45:11 +0800
comments: true
tags: Android
---

最近碰到一个需求，需要在其他应用前台运行时弹出自己应用的对话框，通知用户信息。当然，这么做是完全和Android设计模式相悖的。通常情况下，当应用处于后台时，要以通知栏的形式和用户交互。但是，具体要如何实现了？让我们一起试试。<!-- more -->

## 直接使用AlertDialog并显示

最初的想法很简单，我在Service中直接`create`一个对话框，然后调用`show`方法不就行了吗？代码如下：

```java
public class RunningService extends Service {

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        showDialog();
        new Handler().postDelayed(new Runnable() {
            @Override
            public void run() {
                showDialog();
            }
        }, 5 * 1000);
        return START_STICKY;
    }

    private void showDialog() {
        AlertDialog dialog = new AlertDialog.Builder(App.getContext()).setTitle(R.string.prompt)
                .setMessage(getString(R.string.remove_ads_push_dialog_text))
                .setCancelable(false)
                .setPositiveButton(getString(R.string.confirm), new DialogInterface.OnClickListener() {
                    public void onClick(DialogInterface dialog, int id) {

                    }
                })
                .create();
        dialog.show();
    }

}
```

为了方便测试，我们让对话框弹出时间延迟5s。 
在AndroidManifest注册该Service：

```xml
<service android:name=".aaron.service.RunningService"></service>
```

并在应用入口开启Service：

```java
startService(new Intent(this, RunningService.class));
```

但是发现，应用在切回桌面后，并没有弹出对话框，而且Service也关闭了。其实是因为程序抛出了异常。上网找相关资料，原来只有依附于Activity或Fragment才能开启对话框，否则将抛出异常。那是不是意味着没办法在Service启动对话框呢？实际上并不是，我们需要额外做一个设置即可。

## Dialog设置成TYPE_SYSTEM_ALERT

将Dialog Type设置成`TYPE_SYSTEM_ALERT`：

```java
private void showDialog() {
    AlertDialog dialog = new AlertDialog.Builder(App.getContext()).setTitle(R.string.prompt)
            .setMessage(getString(R.string.remove_ads_push_dialog_text))
            .setCancelable(false)
            .setPositiveButton(getString(R.string.confirm), new DialogInterface.OnClickListener() {
                public void onClick(DialogInterface dialog, int id) {
                }
            })
            .create();
    dialog.getWindow().setType(WindowManager.LayoutParams.TYPE_SYSTEM_ALERT);
    dialog.show();
}
```

关键代码`dialog.getWindow().setType(WindowManager.LayoutParams.TYPE_SYSTEM_ALERT);` 
并在AndroidManifest中添加权限：

```xml
<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />
```

启动应用后，发现这次不会跑异常了，但返回桌面后，还是没见到弹出对话框。是程序出错了吗？如果在`dialog.show();`之后打印**Log**，是能正常打印的，说明对话框确实已经显示。如果此时返回我们的应用，会发现刚才没显示的对话框显示了。可以知道，虽然这种方案可以显示出对话框，但并不是我们想要的效果。我们需要在其他应用中显示对话框，而不需要回到自己的应用。

## 使用对话框主题Activity

我们知道，在Service中开启一个Activity是没有问题的，为何不用Activity作为我们的对话框呢？

- 定义Activity Dialog样式

```xml
<style name="AppTheme.Dialg" parent="Theme.AppCompat.Light.Dialog">

</style>
```

- 在Service中启动对话框Activity

```java
new Handler().postDelayed(new Runnable() {
	@Override
	public void run() {
		Intent dialogIntent = new Intent(SendActivePackageService.this, PushDialogActivity.class);
		dialogIntent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
		startActivity(dialogIntent);
	}
}, Config.DELAY_SHOW_PUSH_DIALOG);
```

> 需要注意`setFlags(Intent.FLAG_ACTIVITY_NEW_TASK)`不能少，否则开启对话框时会强制跳转到对话框所属的应用。

举个简单的例子，比如现在正使用优酷客户端。这时，后台服务弹出对话框了，该对话框并不是直接覆盖在优酷客户端上，而是先跳转到你的应用，再弹出对话框。这显然不是我们想要的，大家可以试试。

目前，最后一种解决方案真正解决了我们的需求。但还是重复一遍，因为非常重要：  
**切忌在其他应用中弹出自己应用的对话框，类似的场景应该采用Notification来解决。**

> 参考资料：
> [How to display a Dialog from a Service](http://stackoverflow.com/questions/7918571/how-to-display-a-dialog-from-a-service)