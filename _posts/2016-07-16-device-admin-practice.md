---
layout: post
title: "DeviceAdmin简单实践"
tags: Android
comments: true
---

## **简介** ##
DeviceAdmin顾名思义设备管理，是Android在2.2版本中引进的。通过用户授权自己的应用设备管理权限后，可以在代码中修改很多系统设置。主要应用场景，例如公司给员工分配的机器，学校给学生提供的教学设备等等。
可查看[官方简介][1]，其中有代码样例。

### **基本功能** ###
 1. 设置锁屏密码
 2. 设置密码规则
 3. 设置密码试错次数
 4. 设置一定时长未操作设备后锁屏
 5. 强制锁屏
 6. 禁用相机
 7. 恢复出厂设置
 8. 锁屏时禁用某些功能


### **如何配置** ###

#### 1. 在资源目录的XML文件夹（没有此文件夹可自己新建一个）下创建一个xml文件，文件名可自定义。在其中添加你需要的权限。
{% highlight xml %}
<device-admin xmlns:android="http://schemas.android.com/apk/res/android" >
    <uses-policies>
        <!-- 设置密码规则 -->
        <limit-password />
        <!-- 监视屏幕解锁尝试次数 -->
        <watch-login />
        <!-- 更改解锁密码 -->
        <reset-password />
        <!-- 锁定屏幕 -->
        <force-lock />
        <!-- 清除数据，恢复出厂模式，在不发出警告的情况下 -->
        <wipe-data />
        <!-- 锁屏密码有效期 -->
        <expire-password />
        <!-- 对存储的应用数据加密 -->
        <encrypted-storage />
        <!-- 禁用锁屏信息 -->
        <disable-keyguard-features/>
        <!-- 禁用摄像头 -->
        <disable-camera />
    </uses-policies>
</device-admin>
{% endhighlight xml %}
#### 2. 注册一个广播继承DeviceAdminReceiver。有很多回调方法可供监听状态的改变。
{% highlight java %}
public class MyDeviceAdminReceiver extends DeviceAdminReceiver {

    @Override
    public void onProfileProvisioningComplete(Context context, Intent intent) {
        super.onProfileProvisioningComplete(context, intent);
    }

    @Override
    public void onEnabled(Context context, Intent intent) {
        super.onEnabled(context, intent);
    }

    @Override
    public CharSequence onDisableRequested(Context context, Intent intent) {
        return super.onDisableRequested(context, intent);
    }

    @Override
    public void onDisabled(Context context, Intent intent) {
        super.onDisabled(context, intent);
    }

    @Override
    public void onPasswordChanged(Context context, Intent intent) {
        super.onPasswordChanged(context, intent);
        Logger.d("onPasswordChanged");
    }

    @Override
    public void onPasswordFailed(Context context, Intent intent) {
        super.onPasswordFailed(context, intent);
        Logger.d("onPasswordFailed");
    }

    @Override
    public void onPasswordSucceeded(Context context, Intent intent) {
        super.onPasswordSucceeded(context, intent);
        Logger.d("onPasswordSucceeded");
    }

    @Override
    public void onPasswordExpiring(Context context, Intent intent) {
        super.onPasswordExpiring(context, intent);
        Logger.d("onPasswordExpiring");
    }

    /**
     * 获取ComponentName，DevicePolicyManager的大多数方法都会用到
     */
    public static ComponentName getComponentName(Context context) {
        return new ComponentName(context.getApplicationContext(), MyDeviceAdminReceiver.class);
    }
}
{% endhighlight java %}

#### 3. 在清单文件里注册广播
{% highlight xml %}
<receiver android:name=".MyDeviceAdminReceiver"
          android:description="@string/device_admin_description"
          android:permission="android.permission.BIND_DEVICE_ADMIN" >
          <meta-data
            android:name="android.app.device_admin"
            android:resource="@xml/device_policies" />
          <intent-filter>
            <action android:name="android.app.action.DEVICE_ADMIN_ENABLED" />
            <action android:name="android.app.action.DEVICE_ADMIN_DISABLE_REQUESTED" />
            <action android:name="android.app.action.DEVICE_ADMIN_DISABLED" />
          </intent-filter>
</receiver>
{% endhighlight xml %}

### **如何使用** ###
获取DevicePolicyManager
{% highlight java %}
DevicePolicyManager dpm = (DevicePolicyManager) getSystemService(Context.DEVICE_POLICY_SERVICE);
{% endhighlight java %}
检查应用是否已激活DeviceAdmin
{% highlight java %}
dpm.isAdminActive(mComponentName);
{% endhighlight java %}
没有激活的话，打开激活页面(device_admin_description是激活页面的描述信息，告知用户你为何需要这些应用，自定义)
{% highlight java %}
Intent intent = new Intent(DevicePolicyManager.ACTION_ADD_DEVICE_ADMIN);
intent.putExtra(DevicePolicyManager.EXTRA_DEVICE_ADMIN,
                mComponentName);
intent.putExtra(DevicePolicyManager.EXTRA_ADD_EXPLANATION,
                getString(R.string.device_admin_description));
startActivity(intent);
{% endhighlight java %}
移除DeviceAdmin，可以手动到手机的设置-安全-设备管理器中关闭，也可以通过代码。
{% highlight java %}
dpm.removeActiveAdmin(mComponentName);
{% endhighlight java %}
当你激活成功后，就可以使用[DevicePolicyManager][2]中那些强大的方法了，但是不包含方法描述中含有下面解释的。

> Called by profile or device owners

先不谈如何成为Device Owners。

我已经测试过上述提到的基本功能中的禁用相机，强制锁屏，一定时长未操作后锁屏，都基本没问题（未测试恢复出厂设置）。但是密码设置地方存在一些bug。

不管我在使用下面这个重设密码的语句之前有没有设置密码规则，这里存在一个bug就是，当你的密码小于5位或者说过短时，这个方法依然会返回true，告诉我设置成功。但当我锁屏后再输入我设置成功的密码却没有任何响应。有趣的是我输入错误的密码，输入框上方文字会提示我密码错误，输入正确的密码没有反应，为此我已经把自己的设备恢复数次出厂设置了。

我也在StackOverFlow上提了[问题][3]，并在Google查询了一番，暂时未找到解决办法，所以如果有人使用这个方法的话，建议在设置密码之前，检查一下密码长度。
{% highlight java %}
dpm.resetPassword(password,0);
{% endhighlight java %}
禁用锁屏信息似乎也无效果。

----------

## **DeviceOwner** ##
相信大家也注意到了在DevicePolicyManager这个类中有很多方法都需要你的应用成为DeviceOwner后才能使用。
那么什么是DeviceOwner呢，见[官方解释][4]：

> Android 5.0 引入了部署设备所有者应用的功能。“设备所有者”是一类特殊的设备管理员，具有在设备上创建和移除辅助用户以及配置全局设置的额外能力。您的设备所有者应用可以使用 DevicePolicyManager 类中的方法来对托管设备上的配置、安全性和应用进行精细控制。一个设备在任一时刻只能有一个处于活动状态的设备所有者。

>要部署并激活设备所有者，您必须在设备处于未配置状态时执行从编程应用到设备的 NFC 数据传输。此数据传输发送的信息与托管配置中描述的配置 intent 中的信息相同。

我们之前申请的DeviceAdmin可以对你的设备进行一些修改，而当你的应用成为DeviceOwner后，你就可以拥有更多的能力，可以对其他应用进行限制。
开发一个DeviceOwner应用的官方指南点[这里][5]，以及[样例][6]。

### **关于DeviceOwner需要注意的几点：** ###
1. 只在Android5.0以上有效。
2. 一个设备只能存在一个DeviceOwner。
3. DeviceOwner应用不能被卸载，也无法在设置->安全中关闭其权限。（具体办法下面讲）。

因为我的英语稀烂，导致很多文档和博客中的某些点可能理解的不是很透彻，所以总结的不是很到位。大家谨慎观看和使用。

### **创建DeviceOwner3种方式** ###

#### 1. 利用NFC功能在手机初始化的时候发送一个DeviceOwner应用到手机上。参考[链接][7]。**（未验证）**

#### 2. 利用ADB命令。**（已验证）**
先把你的DeviceOwner应用安装到设备上。然后运行下面这条ADB命令。
{% highlight bash %}
$adb shell dpm set-device-owner floatingmuseum.devicemanagersample/.MyDeviceAdminReceiver
{% endhighlight bash %}
如果出现错误提示
{% highlight bash %}
java.lang.IllegalStateException: Trying to set device owner but device is already provisioned.
{% endhighlight bash %}
可尝试到设置-账户中删光所有账户。然后重新尝试ADB设置，看到下面结果即设置成功。
{% highlight bash %}
Success: Device owner set to package floatingmuseum.devicemanagersample
Active admin set to component {floatingmuseum.devicemanagersample/floatingmuseum.devicemanagersample.MyDeviceAdminReceiver}
{% endhighlight bash %}

#### 3.  在已root设备上进行。**（已验证）**
先在/data/system/目录下创建一个device_owner.xml文件。
内容如下
{% highlight xml %}
<?xml version="1.0" encoding="utf-8" standalone="yes" ?>
<device-owner package="your owner app packagename" />
{% endhighlight xml %}
然后重启设备，一定要注意在重启设备之前安装好你的DeviceOwner应用，否则会出现无法启动的BUG。

### **部分API的测试使用** ###

1. 隐藏应用 setApplicationHidden()
被隐藏的应用可以在设置应用列表中找到。被隐藏的应用重新安装后，依然无法显示。未测试隐藏系统应用后有何效果。

2. 设置固定屏幕 setLockTaskPackages()
这个可以用来将屏幕锁定在你的应用上面，例如车站的电子展牌，可触摸的电子屏广告等等。通常我们可以在系统设置-安全-屏幕固定中来设置固定屏幕，或者使用ActivityManager的startLockTask()方法，以上方式会给用户发出信息通知用户即将固定屏幕，并提供退出固定屏幕的方法。
而如果在使用startLockTask之前调用setLockTaskPackages方法，会不发送通知直接固定屏幕，并且隐藏home键和菜单键（如果是虚拟按键的话）。无法通过长按返回键退出固定屏幕（测试系统为氧os安卓5.0上依然可以长按返回键退出）。在无法使用长按返回键的情况下有两种方法退出，重启设备或者使用Activity.stopLockTask()。

3. 设置用户限制 addUserRestrictions()
可以配置的功能有很多，包括限制音量调节，应用安装卸载，WIFI蓝牙设置，恢复出厂，添加删除用户，电话短信等等，很强大的的方法。
测试了限制音量调节和禁止应用安装卸载，都运行良好。
具体参数参考UserManager的[SUMMARY][8]。

4. 设置可以使用辅助功能的应用 setPermittedAccessibilityServices()
使用此方法后，所有不在此方法设置中的应用都不得使用辅助功能，在系统设置-辅助功能里的选项会变为灰色，但是对系统应用无效。
参数传null可以取消所有限定。

5. 设置可以使用的输入法 setPermittedInputMethods()
此方法只对第三方输入法有效，系统输入法不会被禁用，被禁用的输入法在设置里呈灰色显示。
参数传null可以取消限定。

6. 设置禁止屏幕捕获 setScreenCaptureDisabled()
使用此方法后，屏幕录像软件只能录取到黑屏，类似vysor这类屏幕捕获软件也只能看到黑屏。不过依然可以看到状态栏以及toast提示。

7. 全局设置 setGlobalSetting()
包括ADB开关，流量开关，手机连接电脑时是否可以开启USB存储设备模式，开发者模式开关等等。
ADB开关设置没有问题。而USB存储模式大多数新设备都已经没有了，开发者模式打开没问题，关闭没效果。

8. 安全设置 setSecureSetting()
包括默认输入法，阻止安装未知来源应用，地理位置等等。

### **在6.0上的变化** ###
在Android6.0版本上谷歌新增加了运行时权限的申请，因此DevicePolicyManager的API中也多出了两个对于权限申请的操作，分别为单一应用权限限制，以及全局权限限制。

#### 1. 单一应用权限设置 setPermissionGrantState()
说实话这个API不知道是我测试的方法不对，还是他有自已的特定顺序，使用起来非常麻烦，建议大家暂时放弃此方法。
测试场景
首先这里有一个与此方法对应的用来获取指定应用的指定权限的状态getPermissionGrantState()。然后新安装一个APP，需要写SD卡权限。

设定此应用写SD卡权限为自动拒绝。
获取状态显示依然为PERMISSION_GRANT_STATE_DEFAULT。进入应用设置中查看权限，写SD卡权限为灰色不可选，并且处于拒绝状态。进入应用中去申请权限，不弹申请权限提示框，直接失败。

好在以上步骤下我们继续，设定此应用写SD卡权限为自动获取。
获取状态仍旧显示为PERMISSION_GRANT_STATE_DEFAULT。应用设置查看权限为灰色，但是处于权限已给予的状态。进入应用中去申请权限，不弹框，提示失败。

继续以上步骤，设定写SD卡权限为默认
获取状态为PERMISSION_GRANT_STATE_DEFAULT，应用设置中可操作，并因为上一步是自动获取，这里显示权限已给予状态。应用中点申请，告知有开启此权限。

这只是其中一种情况，其他诡异BUG也会时常出现，所以如果有人需要使用权限限制，请谨慎。

#### 2. 全局动态权限设置 setPermissionPolicy()
对全部应用的权限进行初始限制，如果应用的权限在此限制之前已被设置过，无论是拒绝还是赋予，都不会再受此设置的限制，简单说就是只对未初始化过的权限进行限制。
三种状态，PERMISSION_POLICY_PROMPT，PERMISSION_POLICY_AUTO_GRANT，PERMISSION_POLICY_AUTO_DENY，申请，自动赋予，自动拒绝。

此设置状态为永久的，即你设置了自动拒绝，应用安装好所有动态权限默认拒绝无法修改，即使你该全局设置为自动赋予或者申请，应用依然遵照被安装时的权限设置。

这里需要注意文档里这句话：

> This also applies to new permissions declared by app updates. When a permission is denied or granted this way, the effect is equivalent to setting the permission grant state via setPermissionGrantState(ComponentName, String, String, int).

先说一个例子吧。
假如我安装一个应用之前设置了全局限制为PERMISSION_POLICY_PROMPT,然后安装此应用后，为申请权限前修改全局限制为PERMISSION_POLICY_GRANT,然后应用申请权限，会自动赋予，并且无法在设置里修改，类似设置单一应用权限的效果。
总的来说就是在我应用安装后权限未申请前，全局权限设置发生了改变，变更为自动赋予或者自动拒绝的话，我再申请权限，效果会和单一应用权限设置一样。全局设置是自动拒绝则我申请自动拒绝并且设置变灰无法修改，全局设置自动赋予，则我我申请自动赋予并且设置变灰无法修改。

#### 3.禁用状态栏 setStatusBarDisabled()
这个没问题可以放心使用。

### **在7.0上的变化** ###
[参考链接][9]

----------

### **总结** ###
总的来说DevicePolicyManager中的大部分API还是比较好用的，并且可以有效的管理用户设备，如果需要开发员工设备或者公共场合的大屏展示应用，可以在这里找到不少有用的设置设备的方法。
而已存在的一些坑，比如密码设置，权限设置等等，大家开发时多加小心，或者选择其他方式绕过。

另外还有一些关于Android证书方面的方法，以及Profile Owner相关的方法，对此了解太少，希望有了解的朋友可以PO文讲解一下。

----------
本文[Demo][10]

## **参考链接** ##
1. [Android Device Owner][11] by SDG systems
2. [Implementing Kiosk Mode in Android - Part 1][12] by SDG systems
3. [Implementing Kiosk Mode in Android - Part 2][13] by SDG systems
4. [Implementing Kiosk Mode in Android - Part 3: Android Lollipop (and Marshmallow)][14] by SDG systems
5. [Implementing Kiosk Mode in Android - Part 4: A Better Provisioning Method for DPC / Device Owner][15] by SDG systems
6. [An Introduction to Android for Work][16] by SDG systems
7. [How to make my app a device owner?][17] by StackOverflow
8. [10 things to know about Device Owner Apps in Android 5][18] by Florent Dupont
9. [Android 5 Screen Pinning][19] by Florent Dupont
10. [DevicePolicyManager][20] by Google Android Developer
11. [Device Administration][21] by Google Android Developer
12. [UserManager][22] by Google Android Developer
13. [办公场所和教育环境中的 Android][23] by Google Android Developer
14. [Android for work在Android M中的变化][24] by Google Android Developer
15. [Android for work在Android M中的变化(中文版)][25]
16. [Android for work在Android N中的变化][26] by Google Android Developer
17. [Chrome Policy List][27] by The Chromium Projects


  [1]: https://developer.android.com/guide/topics/admin/device-admin.html
  [2]: https://developer.android.com/reference/android/app/admin/DevicePolicyManager.html
  [3]: http://stackoverflow.com/questions/37672961/using-devicepolicymanager-resetpassword-succeeded-but-unlock-the-screen-failure
  [4]: https://developer.android.com/about/versions/android-5.0.html#DeviceOwner
  [5]: https://developers.google.com/android/work/build-dpc
  [6]: https://developer.android.com/samples/BasicManagedProfile/index.html
  [7]: https://sdgsystems.com/blog/implementing-kiosk-mode-android-part-3-android-lollipop
  [8]: https://developer.android.com/reference/android/os/UserManager.html
  [9]: https://developer.android.com/preview/features/afw.html
  [10]: https://github.com/floatingmuseum/DeviceAdminSample
  [11]: https://sdgsystems.com/blog/android-device-owner
  [12]: https://sdgsystems.com/blog/implementing-kiosk-mode-android-part-1
  [13]: https://sdgsystems.com/blog/implementing-kiosk-mode-android-part-2
  [14]: https://sdgsystems.com/blog/implementing-kiosk-mode-android-part-3-android-lollipop
  [15]: https://sdgsystems.com/blog/implementing-kiosk-mode-android-part-4-better-provisioning-method-dpc-device-owner
  [16]: https://sdgsystems.com/blog/introduction-android-work
  [17]: http://stackoverflow.com/questions/21183328/how-to-make-my-app-a-device-owner
  [18]: http://florent-dupont.blogspot.jp/2015/02/10-things-to-know-about-device-owner.html
  [19]: http://florent-dupont.blogspot.jp/2015/02/android-5-screen-pinning.html
  [20]: https://developer.android.com/reference/android/app/admin/DevicePolicyManager.html
  [21]: https://developer.android.com/guide/topics/admin/device-admin.html
  [22]: https://developer.android.com/reference/android/os/UserManager.html
  [23]: https://developer.android.com/about/versions/android-5.0.html#Enterprise
  [24]: https://developer.android.com/about/versions/marshmallow/android-6.0-changes.html#behavior-afw
  [25]: http://seewhy.leanote.com/post/Android-6.0-Changes
  [26]: https://developer.android.com/preview/features/afw.html
  [27]: http://www.chromium.org/administrators/policy-list-3