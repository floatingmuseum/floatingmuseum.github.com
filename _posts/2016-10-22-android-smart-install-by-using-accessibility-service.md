---
layout: post
title: "利用Android辅助功能(无障碍)实现应用自动点击安装"
tags: Android
comments: true
---

#### **前言** ####
我们可以在系统设置->辅助功能中自定义自己的服务，来实现监控系统的PackageInstaller(安装器)，并执行自动点击。
自动安装的功能比较常见，基本上各大应用市场都有这个服务。网上也有不少博文分析讲解，但是拿来的代码执行起来都有些缺陷，比如多次点击，控件获取不到，没有区分安装卸载等问题。
我自己通过在网上搜索以及测试，基本实现了一个比较完善的自动安装。

#### **配置文件** ####
首先在res目录下创建xml文件夹，在xml文件夹下创建一个xml文件，文件名自定义。服务的配置信息也可以在继承了AccessibilityService的类中的onServiceConnected()方法中动态配置。

{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<accessibility-service xmlns:android="http://schemas.android.com/apk/res/android"
    android:packageNames="com.android.packageinstaller,com.google.android.packageinstaller,com.lenovo.safecenter"
    android:description="@string/smart_install_accessibility_service_description"
    android:accessibilityEventTypes="typeAllMask"
    android:accessibilityFlags="flagDefault"
    android:accessibilityFeedbackType="feedbackGeneric"
    android:canRetrieveWindowContent="true"
    />
    <!--
    packageNames
    指定我们要监听哪个应用程序下的窗口活动，这里写com.android.packageinstaller表示监听Android系统的安装界面。部分Android Rom可能替换了系统的安装器，具体需要自己适配。
    description
    指定在无障碍服务当中显示给用户看的说明信息。
    accessibilityEventTypes
    指定我们在监听窗口中可以接收哪些事件，例如长按，点击，窗口内容变化等等，这里写typeAllMask表示所有的事件都能模拟。
    accessibilityFlags
    可以指定无障碍服务的一些附加参数，传默认值flagDefault就行。
    accessibilityFeedbackType
    指定无障碍服务的反馈方式，类似语音，震动等等。实际上无障碍服务这个功能是Android提供给一些残疾人士使用的，比如说盲人不方便使用手机，就可以借助无障碍服务配合语音反馈来操作手机，而我们其实是不需要反馈的，因此随便传一个值就可以，这里传入feedbackGeneric(普通回馈)。
    canRetrieveWindowContent
    指定是否允许我们的程序读取窗口中的节点和内容，必须写true。
    notificationTimeout
    响应事件的时间间隔
    -->
{% endhighlight xml %}
[详细参考链接][1]

#### **清单文件中配置** ####
{% highlight xml %}
        <service
            android:name=".smartinstall.SmartInstallService"
            android:label="辅助智能安装"
            android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE">
            <intent-filter>
                <action android:name="android.accessibilityservice.AccessibilityService" />
            </intent-filter>

            <meta-data
                android:name="android.accessibilityservice"
                android:resource="@xml/smart_install_accessibility_service_config" />
        </service>
        <!-- 
        创建一个Service类集成AccessibilityService来处理接收到的事件
        label:是你在系统设置辅助功能中显示的名称。
        resource：指向你配置的xml文件。
        其他都是固定的。
        -->
{% endhighlight xml %}

#### **代码** ####
{% highlight java %}
public class SmartInstallService extends AccessibilityService {

    private String log = "SmartInstallService";

    private boolean isUninstall = false;

    Map<Integer, Boolean> eventMap = new HashMap<>();
    private static final String INSTALL = "安装";
    private static final String INSTALL_FINISHED = "应用安装完成。";
    private static final String FINISHED = "完成";
    private static final String CONFIRMED = "确定";
    private static final String UNINSTALL = "卸载";

    //服务在系统设置中开启时触发此方法
    @Override
    protected void onServiceConnected() {
        super.onServiceConnected();
        //获取当前服务的配置信息
        Logger.d("智能安装onServiceConnected");
//        AccessibilityServiceInfo info = getServiceInfo();
        //配置设置信息
//        info.packageNames = new String[]{"com.android.packageinstaller","com.google.android.packageinstaller","com.lenovo.safecenter"};
//        setServiceInfo(info);
    }

    /**
     * 问题来源：https://coolpers.github.io/accessibility/install/uninstall/2015/04/27/Accessibility-automatically-install-and-uninstall.html
     * 1.部分手机无法自动点击“打开”按钮 在部分手机（例如HTC 手机）上发现安装过程最后一步显示“打开”按钮界面，
     * 使用AccessibilityEvent.getSource()[Added in API level 14]获取到的值为空，导致无法获取触发点击“打开”按钮。
     * 通过AccessibilityService.getRootInActiveWindow ()[Added in API level 16] 获取整个窗口的控件对象信息解决此问题。
     * <p>
     * 2.部分手机自动安装页面无任何反应 例子中判断需要点击的按钮对象为Button时才触发，但是一些手机上按钮是TextView，
     * 通过添加TextView判断条件解决。
     */
    //回调监听窗口的事件
    @Override
    public void onAccessibilityEvent(AccessibilityEvent event) {
        //遍历节点之前，现将卸载判定置为false;
        isUninstall = false;

//        AccessibilityNodeInfo nodeInfo = event.getSource();
        AccessibilityNodeInfo rootNodeInfo = getRootInActiveWindow();

        if (rootNodeInfo != null) {
            int eventType = event.getEventType();
            //响应窗口内容变化，窗口状态变化，控件滚动三种事件。
            if (eventType == AccessibilityEvent.TYPE_WINDOW_CONTENT_CHANGED
                    || eventType == AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED
                    || eventType == AccessibilityEvent.TYPE_VIEW_SCROLLED) {
                    //通过一个Map集合来过滤重复点击事件
                if (eventMap.get(event.getWindowId()) == null) {
                    boolean handled = handleEvent(rootNodeInfo);
                    if (handled) {
                        eventMap.put(event.getWindowId(), true);
                    }
                }
            }
        }
    }

    private boolean handleEvent(AccessibilityNodeInfo nodeInfo) {
        if (nodeInfo != null) {
        //检查是否是卸载操作
            checkIsUninstallAction(nodeInfo);

            if (!isUninstall) {

                /**
                 * 这里把执行操作的代码抽取了出来，在下面这个方法中进行递归。
                 * 如若在当前方法中递归，checkUninstall方法会多次执行，没意义。
                 */
                return startPerformNodeAction(nodeInfo);
            }
        }
        return false;
    }

    private boolean startPerformNodeAction(AccessibilityNodeInfo nodeInfo) {
        //获取子节点数量
        int childCount = nodeInfo.getChildCount();

        switch (nodeInfo.getClassName().toString()) {
            case "android.widget.Button":
                if (nodeInfo.getText() != null) {
                    String nodeContent = nodeInfo.getText().toString();
                    if (INSTALL.equals(nodeContent) || FINISHED.equals(nodeContent) || CONFIRMED.equals(nodeContent)) {
                        nodeInfo.performAction(AccessibilityNodeInfo.ACTION_CLICK);
                        return true;
                    }
                }
                break;
            case "android.widget.ScrollView":
                nodeInfo.performAction(AccessibilityNodeInfo.ACTION_SCROLL_FORWARD);
                break;
        }

        for (int i = 0; i < childCount; i++) {
            AccessibilityNodeInfo childNodeInfo = nodeInfo.getChild(i);
            if (startPerformNodeAction(childNodeInfo)) {
                return true;
            }
        }

        return false;
    }

    /**
     * 通过遍历界面节点，查看是否存在带有卸载的字符串。来判断是否是卸载操作。
     * 通过判断字符串的方式不够严谨，可能会出现误中安装操作的情况。
     */
    private void checkIsUninstallAction(AccessibilityNodeInfo nodeInfo) {
        int childCount = nodeInfo.getChildCount();
        if (childCount != 0) {
            for (int x = 0; x < childCount; x++) {
                checkIsUninstallAction(nodeInfo.getChild(x));
            }
        } else {
            if (nodeInfo.getText() != null) {
                String nodeContent = nodeInfo.getText().toString();
                if (nodeContent.contains(UNINSTALL)) {
                    isUninstall = true;
                }
            }
        }
    }

    //服务在设置中被关掉时调用
    @Override
    public void onInterrupt() {
        Logger.d("智能安装onInterrupt");
    }
}
{% endhighlight java %}
#### **目前可能存在的弊端** ####
1. 对于卸载操作的判定，若某个应用程序名称中包含卸载字样，在其安装时可能会判定为卸载操作。
2. 辅助功能开启后，即使退出应用，服务也依然运行有效，此时安装某个应用，依然会被此服务响应进行自动安装，可以在此服务中定义一个变量，放在onAccessibilityEvent()入口处，然后通过EventBus或者自定义广播修改其属性。
3. 华为Rom在安装完毕后会提示是否删除安装包，根据需求决定是否为用户执行此操作。其他Rom也可能存在类似自定义后的变化。

#### **已测试环境**
1. OnePlus OxygenOS 5.1.1
2. HuaWei Honor4X 6.0.1
3. Android4.4.4平板
4. Nexus S 4.4
5. Nexus 7Ⅱ  5.1

#### **参考链接** ####
1. [Android静默安装实现方案，仿360手机助手秒装和智能安装功能][2]
2. [通过Accessibility自动安装与卸载应用][3]
3. [深入了解AccessibilityService][4]


  [1]: https://developer.android.com/reference/android/accessibilityservice/AccessibilityServiceInfo.html
  [2]: http://blog.csdn.net/sinyu890807/article/details/47803149
  [3]: https://coolpers.github.io/accessibility/install/uninstall/2015/04/27/Accessibility-automatically-install-and-uninstall.html
  [4]: http://blog.csdn.net/dd864140130/article/details/51794318