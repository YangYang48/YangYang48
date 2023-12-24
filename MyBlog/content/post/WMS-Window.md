---
title: "WMS-Window"
date: 2023-04-02
thumbnailImagePosition: left
thumbnailImage: WMS/window/window_thumb.jpg
coverImage: WMS/window/window_cover.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- service call
- 2023
- April
tags:
- Android
- 源码
- Binder
- WMS
- Window
- WindowManager
- ViewRootImpl
- DecorView
- WindowToken
- WindowState
- Window层级
- Window容器
showSocial: false
---

Android framework层离不开AMS、WMS、PMS等，其中WMS中Window这个概念比较抽象，且很多博客作者写了很多关于Window的文章，我个人感觉还缺点什么，本文尽可能通过通俗易懂的方式来描述（android11和13源码）。

<!--more-->
# 0快问快答

1）Window，WMS，Activity之间的联系（什么时候Window和Activity和WMS产生联系）

2）Window和View区别和联系（什么时候Window和View产生联系）

3）Window的类别和层次关系（为什么PopWindow会遮住App的显示，锁屏为什么看不到Launcher）

4）activity与 PhoneWindow与DecorView关系 （如果我添加一个View，会添加到系统层面去吗）

# 1介绍

Window 是一个窗口的概念，是所有视图的载体，不管是 Activity，Dialog，还是 Toast，他们的视图都是附加在 Window 上面的。例如在桌面显示一个悬浮窗，就需要用到 Window 来实现。



# 2源码跟踪分析问题

## 2.1Window，WMS，Activity之间的联系

- **Window**： 在Android视图体系中Window就是一个窗口的概念。 Android中所有的视图都是依赖于Window显示的。
- **WindowManager**： 对Window的管理， 包括新增、更新和删除等。
- **WMS(WindowManagerService)**： 窗口的最终管理者， 它负责窗口的启动、添加和删除， 另外窗口的大小和层级也是由WMS进行管理  

具体关于三者之间的联系，请翻看2.5.1

## 2.2Window和View区别和联系

WindowManagerImpl实际控制对应的PhoneWindow

ViewRootImpl与View树进行关联，这样ViewRootImpl就可以指挥View树的具体工作  

Activity#onCreate()中调用setContentView()方法，这个方法内部创建一个DecorView实例作为PhoneWindow的内容。WindowManagerImpl决定管理DecorView， 并创建一个ViewRootImpl实例

具体关于Window和View的联系，请翻看2.5.2和2.5.3 

## 2.3Window的类别和层次关系

通常我们开发应用的时候会用到各种View、布局，但是这和Window并不是一回事。打开一个应用，出现界面，我们可以理解出现了一个窗口。实际上View和Window是光年和年的关系。

> 一个 Activity 可以理解 对应一个 Window，ViewRootImpl 是对应一个 Window。

查看window方式，笔者这里使用手机打开浏览器的设置操作

```shell
$ dumpsys window windows
# 输入法
Window #0 Window{6c306bc u0 InputMethod}:
mBaseLayer=141000 mSubLayer=0    mToken=WindowToken{25f6be3 android.os.Binder@5319812}
isVisible=false
# 圆角
Window #1 Window{6474d57 u0 RoundCorner}:
mBaseLayer=311000 mSubLayer=0    mToken=WindowToken{7706d6 android.os.BinderProxy@359caf1}
isVisible=true
# 圆角
Window #2 Window{98435fe u0 RoundCorner}:
mBaseLayer=311000 mSubLayer=0    mToken=WindowToken{a6ec0b2 android.os.BinderProxy@1af12bd}
isVisible=true
# 导航栏
Window #3 Window{987738d u0 NavigationBar}:
mBaseLayer=231000 mSubLayer=0    mToken=WindowToken{cdfac24 android.os.BinderProxy@1bf9cb7}
isVisible=true
# 状态栏
Window #4 Window{85d4239 u0 StatusBar}:
mBaseLayer=181000 mSubLayer=0    mToken=WindowToken{e3e1a00 android.os.BinderProxy@bf7a332}
isVisible=true
# 圆角
Window #5 Window{5c8904f u0 RoundCorner}:
mBaseLayer=171000 mSubLayer=0    mToken=WindowToken{c281aae android.os.BinderProxy@485e729}
isVisible=false
# SYSTEM_ALERT_WINDOW
Window #6 Window{e12503c u0 Aspect}:
mBaseLayer=111000 mSubLayer=0    mToken=WindowToken{cb4622f android.os.BinderProxy@bba880e}
isVisible=false
# 协助预览盘
Window #7 Window{31fbd71 u0 AssistPreviewPanel}:
mBaseLayer=41000 mSubLayer=0    mToken=WindowToken{f8d1f18 android.os.BinderProxy@c09ec8a}
isVisible=false
# 分屏窗口
Window #8 Window{67faf21 u0 DockedStackDivider}:
mBaseLayer=21000 mSubLayer=0    mToken=WindowToken{f740988 android.os.BinderProxy@d47412b}
isVisible=false
# 浏览器设置
Window #9 Window{297fd5f u0 PopupWindow:9fecaec}:
mBaseLayer=21000 mSubLayer=2    mToken=AppWindowToken{380207c token=Token{d724b6f ActivityRecord{b054e4e u0 com.android.chrome/org.chromium.chrome.browser.ChromeTabbedActivity t220}}}
isVisible=true
# 浏览器
Window #10 Window{7a87747 u0 com.android.chrome/org.chromium.chrome.browser.ChromeTabbedActivity}:
mBaseLayer=21000 mSubLayer=0    mToken=AppWindowToken{380207c token=Token{d724b6f ActivityRecord{b054e4e u0 com.android.chrome/org.chromium.chrome.browser.ChromeTabbedActivity t220}}}
isVisible=true
# Launcher悬浮窗口
Window #11 Window{8c10cde u0 LauncherOverlayWindow:com.miui.personalassistant}:
mBaseLayer=21000 mSubLayer=0    mToken=AppWindowToken{e921607 token=Token{361c446 ActivityRecord{6d5b688 u0 com.miui.home/.launcher.Launcher t1}}}
isVisible=false
# Launcher
Window #12 Window{6163c5a u0 com.miui.home/com.miui.home.launcher.Launcher}:
mBaseLayer=21000 mSubLayer=0    mToken=AppWindowToken{e921607 token=Token{361c446 ActivityRecord{6d5b688 u0 com.miui.home/.launcher.Launcher t1}}}
isVisible=false
# 壁纸，类似透明板（动态壁纸）
Window #13 Window{f63516 u0 com.android.keyguard.wallpaper.service.MiuiKeyguardLiveWallpaper}:
mBaseLayer=11000 mSubLayer=0    mToken=WallpaperWindowToken{f661030 token=android.os.BinderProxy@3481973}
isVisible=false
# 壁纸
Window #14 Window{cf10cf2 u0 com.android.systemui.ImageWallpaper}:
mBaseLayer=11000 mSubLayer=0    mToken=WallpaperWindowToken{5182c91 token=android.os.Binder@2b356b8}
isVisible=false
```

这里的Window数字越小，那么显示距离屏幕越近。

### 2.3.1Window类别

1. Application Window： Activity就是一个典型的应用程序窗口

2. Sub Window： 子窗口， 顾名思义， 它不能独立存在， 需要附着在其他窗口才可以， PopupWindow就属于子窗口
3. System Window： 输入法窗口、 系统音量条窗口、系统错误窗口都属于系统窗口

Application Window【type 取值范围 [0,999]】

应用程序窗口，主要是应用中的出现的Window，也就是对应我们说的Activity。

```java
//frameworks/base/core/java/android/view/WindowManager.java
public static final int FIRST_APPLICATION_WINDOW = 1;
public static final int TYPE_BASE_APPLICATION   = 1;
public static final int TYPE_APPLICATION        = 2;
public static final int TYPE_APPLICATION_STARTING = 3;
public static final int TYPE_DRAWN_APPLICATION = 4;
public static final int LAST_APPLICATION_WINDOW = 99;
```

Sub window【type 取值范围 [1000,1999]】

不能独立存在，需依附其他窗口的Window，也就是Subwindow可以依赖于Application Window或System Window，也可以依赖于Subwindow，可以存在嵌套的子窗口。实际上学习完Window类别和层次关系，可以知道其实是两个Window属于同一个WindowToken容器，距离谷歌浏览器中打开右上角「设置」，「设置」属于子窗口。

```java
//frameworks/base/core/java/android/view/WindowManager.java
public static final int FIRST_SUB_WINDOW = 1000;
public static final int TYPE_APPLICATION_PANEL = FIRST_SUB_WINDOW;
public static final int TYPE_APPLICATION_MEDIA = FIRST_SUB_WINDOW + 1;
public static final int TYPE_APPLICATION_SUB_PANEL = FIRST_SUB_WINDOW + 2;
public static final int TYPE_APPLICATION_ATTACHED_DIALOG = FIRST_SUB_WINDOW + 3;
public static final int TYPE_APPLICATION_MEDIA_OVERLAY  = FIRST_SUB_WINDOW + 4;
public static final int TYPE_APPLICATION_ABOVE_SUB_PANEL = FIRST_SUB_WINDOW + 5;
public static final int LAST_SUB_WINDOW = 1999;
```

System Window【type 取值范围 [2000,2999]】

如 Toast，ANR 窗口，输入法，StatusBar，NavigationBar 等。

```java
//frameworks/base/core/java/android/view/WindowManager.java
public static final int TYPE_STATUS_BAR         = FIRST_SYSTEM_WINDOW;
public static final int TYPE_SEARCH_BAR         = FIRST_SYSTEM_WINDOW+1;
public static final int TYPE_PHONE              = FIRST_SYSTEM_WINDOW+2;
...
public static final int TYPE_STATUS_BAR_ADDITIONAL = FIRST_SYSTEM_WINDOW + 41;
public static final int LAST_SYSTEM_WINDOW      = 2999;
```



> type 值越大，层级越高， **这个观点是不完全正确的**。
>
> 相对而言，确实type值越大会越接近屏幕，但是相同type或者是存在SubWindow情况，用Type是无法满足的。
>
> 因此，引入Window层级，**层级越高，最终在屏幕上显示时就越靠近用户**。

### 2.3.2Window层次关系

首先，先介绍一下Window容器中的WindowToken和WindowState

> **WindowToken**
>
> WindowToken理解成是一个**显示令牌**，无论是系统窗口还是应用窗口，添加新的窗口的时候必须使用这个令牌向WMS表明自己的身份，添加窗口的时候会创建WindowToken，销毁窗口的时候移除WindowToken(removeWindowToken方法)。
>
> ＷMS使用WindowToken将同一个应用组件(Activity,InputMethod,Wallpaper,Dream)的窗口组织在一起，换句话说，每一个窗口都会对应一个WindowToken，并且这个窗口中的所有子窗口将会对应同一个WindowToken，这些窗口的WindowToken都是一样的。
>
> **WindowState**
>
> WindowState用于窗口管理，这个是**WMS中事实的窗口**，包含了一个窗口的所有的属性，WindowState对象都存放在mWindowMap里面，mWindowMap是所有窗口的一个全集，如果梳理WindowState的一些增加、移动和删除等操作，会更加理解这个类。举一个例子，对于Activity而言，我们看到的App的中的Activity就是一个个Window；对于系统WMS服务而言，这些Activity一个个对应的Window实际上是WindowState。

为了更好地说清楚层次关系，需要引入两个变量

```java
//frameworks/base/services/core/java/com/android/server/wm/WindowState.java
class WindowState extends WindowContainer<WindowState> implements WindowManagerPolicy.WindowState,
InsetsControlTarget, InputTarget {
    final int mBaseLayer;//主要层级
    final int mSubLayer;//子层级
}
```

#### 2.3.2.1层级顺序 

先看主要层级的顺序，范围是[1, 36]

```java
//frameworks/base/services/core/java/com/android/server/policy/WindowManagerPolicy.java
default int getWindowLayerFromTypeLw(int type) {
    if (isSystemAlertWindowType(type)) {
        throw new IllegalArgumentException("Use getWindowLayerFromTypeLw() or"
                                           + " getWindowLayerLw() for alert window types");
    }
    return getWindowLayerFromTypeLw(type, false /* canAddInternalSystemWindow */);
}
//这里数字越大，层级越大，最小的是1，最大是36
default int getWindowLayerFromTypeLw(int type, boolean canAddInternalSystemWindow,
                                     boolean roundedCornerOverlay) {
    // Always put the rounded corner layer to the top most.
    if (roundedCornerOverlay && canAddInternalSystemWindow) {
        return getMaxWindowLayer();//这里是36，android版本不一样大小不一样，层级顺序也不一样
    }
    //这里先判断type是否是APPLICATION Window中的，如果是返回2
    if (type >= FIRST_APPLICATION_WINDOW && type <= LAST_APPLICATION_WINDOW) {
        return APPLICATION_LAYER;
    }

    switch (type) {
        //壁纸层级最低
        case TYPE_WALLPAPER:
            return  1;
        case TYPE_PRESENTATION:
        case TYPE_PRIVATE_PRESENTATION:
        case TYPE_DOCK_DIVIDER:
        case TYPE_QS_DIALOG:
        case TYPE_PHONE:
            return  3;
        case TYPE_SEARCH_BAR:
            return  4;
        case TYPE_INPUT_CONSUMER:
            return  5;
        case TYPE_SYSTEM_DIALOG:
            return  6;
        case TYPE_TOAST:
            // toasts and the plugged-in battery thing
            return  7;
        case TYPE_PRIORITY_PHONE:
            // SIM errors and unlock.  Not sure if this really should be in a high layer.
            return  8;
        case TYPE_SYSTEM_ALERT:
            // like the ANR / app crashed dialogs
            // Type is deprecated for non-system apps. For system apps, this type should be
            // in a higher layer than TYPE_APPLICATION_OVERLAY.
            return  canAddInternalSystemWindow ? 12 : 9;
        case TYPE_APPLICATION_OVERLAY:
            return  11;
        case TYPE_INPUT_METHOD:
            // on-screen keyboards and other such input method user interfaces go here.
            return  13;
        case TYPE_INPUT_METHOD_DIALOG:
            // on-screen keyboards and other such input method user interfaces go here.
            return  14;
        case TYPE_STATUS_BAR:
            return  15;
        case TYPE_STATUS_BAR_ADDITIONAL:
            return  16;
        case TYPE_NOTIFICATION_SHADE:
            return  17;
        case TYPE_STATUS_BAR_SUB_PANEL:
            return  18;
        case TYPE_KEYGUARD_DIALOG:
            return  19;
        case TYPE_VOICE_INTERACTION_STARTING:
            return  20;
        case TYPE_VOICE_INTERACTION:
            // voice interaction layer should show above the lock screen.
            return  21;
        case TYPE_VOLUME_OVERLAY:
            // the on-screen volume indicator and controller shown when the user
            // changes the device volume
            return  22;
        case TYPE_SYSTEM_OVERLAY:
            // the on-screen volume indicator and controller shown when the user
            // changes the device volume
            return  canAddInternalSystemWindow ? 23 : 10;
        case TYPE_NAVIGATION_BAR:
            // the navigation bar, if available, shows atop most things
            return  24;
        case TYPE_NAVIGATION_BAR_PANEL:
            // some panels (e.g. search) need to show on top of the navigation bar
            return  25;
        case TYPE_SCREENSHOT:
            // screenshot selection layer shouldn't go above system error, but it should cover
            // navigation bars at the very least.
            return  26;
        case TYPE_SYSTEM_ERROR:
            // system-level error dialogs
            return  canAddInternalSystemWindow ? 27 : 9;
        case TYPE_MAGNIFICATION_OVERLAY:
            // used to highlight the magnified portion of a display
            return  28;
        case TYPE_DISPLAY_OVERLAY:
            // used to simulate secondary display devices
            return  29;
        case TYPE_DRAG:
            // the drag layer: input for drag-and-drop is associated with this window,
            // which sits above all other focusable windows
            return  30;
        case TYPE_ACCESSIBILITY_OVERLAY:
            // overlay put by accessibility services to intercept user interaction
            return  31;
        case TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY:
            return 32;
        case TYPE_SECURE_SYSTEM_OVERLAY:
            return  33;
        case TYPE_BOOT_PROGRESS:
            return  34;
        case TYPE_POINTER:
            // the (mouse) pointer layer
            return  35;
        default:
            Slog.e("WindowManager", "Unknown window type: " + type);
            return 3;
    }
}
```

接着看先看次层级的顺序，范围是[-2, 3]

```java
//frameworks/base/services/core/java/com/android/server/policy/WindowManagerPolicy.java
default int getSubWindowLayerFromTypeLw(int type) {
    switch (type) {
        case TYPE_APPLICATION_PANEL:
        case TYPE_APPLICATION_ATTACHED_DIALOG:
            return APPLICATION_PANEL_SUBLAYER;// 1
        case TYPE_APPLICATION_MEDIA:
            return APPLICATION_MEDIA_SUBLAYER;// -2
        case TYPE_APPLICATION_MEDIA_OVERLAY:
            return APPLICATION_MEDIA_OVERLAY_SUBLAYER;// -1
        case TYPE_APPLICATION_SUB_PANEL:
            return APPLICATION_SUB_PANEL_SUBLAYER;// 2
        case TYPE_APPLICATION_ABOVE_SUB_PANEL:
            return APPLICATION_ABOVE_SUB_PANEL_SUBLAYER;// 3
    }
    Slog.e("WindowManager", "Unknown sub-window type: " + type);
    return 0;
}
```

#### 2.3.2.2Window层级计算

在WindowState的构造函数中会对Window层级计算

```java
//frameworks/base/services/core/java/com/android/server/wm/WindowState.java
//将阈值扩大 10000 倍，系统中可能存在相同类型的窗口有很多
int TYPE_LAYER_MULTIPLIER = 10000;
int TYPE_LAYER_OFFSET = 1000;

WindowState(WindowManagerService service, Session s, IWindow c, WindowToken token,
            WindowState parentWindow, int appOp, WindowManager.LayoutParams a, int viewVisibility,
            int ownerId, int showUserId, boolean ownerCanAddInternalSystemWindow,
            PowerManagerWrapper powerManagerWrapper) {
    ...
    //如果是子窗口走这里
    if (mAttrs.type >= FIRST_SUB_WINDOW && mAttrs.type <= LAST_SUB_WINDOW) {
        mBaseLayer = mPolicy.getWindowLayerLw(parentWindow)
            * TYPE_LAYER_MULTIPLIER + TYPE_LAYER_OFFSET;
        mSubLayer = mPolicy.getSubWindowLayerFromTypeLw(a.type);
        mIsChildWindow = true;

        mLayoutAttached = mAttrs.type !=
            WindowManager.LayoutParams.TYPE_APPLICATION_ATTACHED_DIALOG;
        mIsImWindow = parentWindow.mAttrs.type == TYPE_INPUT_METHOD
            || parentWindow.mAttrs.type == TYPE_INPUT_METHOD_DIALOG;
        mIsWallpaper = parentWindow.mAttrs.type == TYPE_WALLPAPER;
    } else {
        //如果是非子窗口走这里
        mBaseLayer = mPolicy.getWindowLayerLw(this)
            * TYPE_LAYER_MULTIPLIER + TYPE_LAYER_OFFSET;
        mSubLayer = 0;
        mIsChildWindow = false;
        mLayoutAttached = false;
        mIsImWindow = mAttrs.type == TYPE_INPUT_METHOD
            || mAttrs.type == TYPE_INPUT_METHOD_DIALOG;
        mIsWallpaper = mAttrs.type == TYPE_WALLPAPER;
    }
    ...
}
```

如 Activity，type 是 TYPE_BASE_APPLICATION ，getWindowLayerLw 计算返回 APPLICATION_LAYER（2），mBaseLayer = 2 * 10000 + 1000 = 21000

#### 2.3.2.3Window 排序

主Window排序

```java
//frameworks/base/services/core/java/com/android/server/wm/WindowToken.java
private final Comparator<WindowState> mWindowComparator =
    (WindowState newWindow, WindowState existingWindow) -> {
    final WindowToken token = WindowToken.this;
    if (newWindow.mToken != token) {
        throw new IllegalArgumentException("newWindow=" + newWindow
                                           + " is not a child of token=" + token);
    }

    if (existingWindow.mToken != token) {
        throw new IllegalArgumentException("existingWindow=" + existingWindow
                                           + " is not a child of token=" + token);
    }
    //判断新的Window和原来的Window关系，符合要求新Window会放在原来Window上面
    return isFirstChildWindowGreaterThanSecond(newWindow, existingWindow) ? 1 : -1;
};

protected boolean isFirstChildWindowGreaterThanSecond(WindowState newWindow,
            WindowState existingWindow) {
        // New window is considered greater if it has a higher or equal base layer.
        return newWindow.mBaseLayer >= existingWindow.mBaseLayer;
}
```

按照 mBaseLayer 大小排序，如果是新插入的，且相等，放在上方

子Window排序

```java
//frameworks/base/services/core/java/com/android/server/wm/WindowState.java
private static final Comparator<WindowState> sWindowSubLayerComparator =
    new Comparator<WindowState>() {
    @Override
    public int compare(WindowState w1, WindowState w2) {
        final int layer1 = w1.mSubLayer;
        final int layer2 = w2.mSubLayer;
        if (layer1 < layer2 || (layer1 == layer2 && layer2 < 0 )) {
            //w1是原来的sublayer，w2是新的子Window
            //如果新的sublayer大，那么放在原来子Window上面
            //如果新的sublayer相等，且sublayer小于0，放到原来子Window下面
            //如果新的sublayer相等且为正值，放在原来子Window上面
            return -1;
        }
        return 1;
    };
};
```

以上面的例子来具体说明

下图**红色部分**才是展示出来的Window叠加起来真正出现的手机画面

{{< image classes="fancybox center fig-100" src="/WMS/window/window18.png" thumbnail="/WMS/window/window18.png" title="">}}

{{< image classes="fancybox center fig-100" src="/WMS/window/window19.png" thumbnail="/WMS/window/window19.png" title="">}}

> 用demo来进一步理解运算逻辑
>
> {{< image classes="fancybox center fig-100" src="/WMS/window/window20.png" thumbnail="/WMS/window/window20.png" title="">}}
>
> {{< image classes="fancybox center fig-100" src="/WMS/window/window21.png" thumbnail="/WMS/window/window21.png" title="">}}
>
> {{< image classes="fancybox center fig-100" src="/WMS/window/window22.png" thumbnail="/WMS/window/window22.png" title="">}}
>
> {{< image classes="fancybox center fig-100" src="/WMS/window/window23.png" thumbnail="/WMS/window/window23.png" title="">}}
>
> {{< image classes="fancybox center fig-100" src="/WMS/window/window24.png" thumbnail="/WMS/window/window24.png" title="">}}

# 2.4WindowContainer

Android窗口是根据显示屏幕来管理，每个显示屏幕的窗口层级分为36层，1-36层。每层可以放置多个窗口，上层窗口覆盖下面的。要理解窗口的结构，需要学习下WindowContainer、RootWindowContainer、DisplayContent、TaskDisplayArea、Task、ActivityRecord、WindowToken、WindowState、WindowContainer等类。

### 2.4.1Window容器架构和继承关系图

#### 2.4.1.1Window容器架构

android11和android13两个版本架构差异比较大，所以为了更好地理解，将分别展示出两个版本的架构

android11 Window容器架构图

{{< image classes="fancybox center fig-100" src="/WMS/window/window3.png" thumbnail="/WMS/window/window3.png" title="">}}

android13 Window容器架构图

{{< image classes="fancybox center fig-100" src="/WMS/window/window3_1.png" thumbnail="/WMS/window/window3_1.png" title="">}}

#### 2.4.1.2Window容器继承关系

android11 Window容器继承关系图

{{< image classes="fancybox center fig-100" src="/WMS/window/window6.png" thumbnail="/WMS/window/window6.png" title="">}}

android13 Window容器继承关系图

{{< image classes="fancybox center fig-100" src="/WMS/window/window6_1.png" thumbnail="/WMS/window/window6_1.png" title="">}}

### 2.4.2源码解析Window容器

#### 2.4.2.1WindowContainer

Window容器的源头，为直接包含窗口或者通过孩子层级形式包含窗口的类，定义了普遍功能。它作为基类被继承，像RootWindowContainer、DisplayContent、TaskDisplayArea、Task、ActivityRecord、WindowToken、WindowState都是直接或间接的继承该类。

```java
//frameworks/base/services/core/java/com/android/server/wm/WindowContainer.java
class WindowContainer<E extends WindowContainer> extends ConfigurationContainer<E>
    implements Comparable<WindowContainer>, Animatable, SurfaceFreezer.Freezable,
InsetsControlTarget {
    //mParent是当前WindowContainer容器的父容器
    private WindowContainer mParent = null;
    //mChildren是一个继承WindowContainer的集合对象，说明当前WindowContainer可能会有好几个子容器
    protected final WindowList<E> mChildren = new WindowList<E>();
};
```

对于WindowContainer需要了解两个成员变量，mParent和mChildren，一个是容器本身的父容器，另一个是容器的子容器（类似上有老下有小的概念，老人只能有一个，小孩可以是多个）

{{< image classes="fancybox center fig-100" src="/WMS/window/window26.png" thumbnail="/WMS/window/window26.png" title="">}}

#### 2.4.2.2窗口结构层级相关类

从Android12开始引入Window的**特色模式**，更好地管理Window的各个层级相关类

- RootWindowContainer：根窗口容器，树的根是它。通过它遍历寻找，可以找到窗口树上的窗口。它的孩子是DisplayContent。

- DisplayContent：该类是对应着显示屏幕的，Android是支持多屏幕的，所以可能存在多个DisplayContent对象。上图只画了一个对象的结构，其他对象的结构也是和画的对象的结构是相似的。
- DisplayArea：该类是对应着显示屏幕下面的，代表一组窗口合集,具有多个子类，如Tokens，TaskDisplayArea等
- TaskDisplayArea：它为DisplayContent的孩子，对应着窗口层次的第2层。第2层作为应用层，看它的定义：int APPLICATION_LAYER = 2，应用层的窗口是处于第2层。TaskDisplayArea的孩子是Task类，其实它的孩子类型也可以是TaskDisplayArea。而Task的孩子则可以是ActivityRecord，也可以是Task。
- Tokens：代表专门包含WindowTokens的容器，它的孩子是WindowToken，而WindowToken的孩子则为WindowState对象。
- ImeContainer：它是输入法窗口的容器，它的孩子是WindowToken类型。WindowToken的孩子为WindowState类型，而WindowState类型则对应着输入法窗口。
- Task：任务，它的孩子可以是Task，也可以是ActivityRecord类型。
- ActivityRecord：是对应着应用进程中的Activity的。ActivityRecord是继承WindowToken的，它的孩子类型为WindowState。
- WindowState：WindowState是对应着一个窗口的。

上面的文字介绍可能会懵，没关系，先了解一下大概，下面会一一介绍内容。

#### 2.4.2.3RootWindowContainer

1）**RootWindowContainer构造**

RootWindowContainer的构造是在WindowManagerService中，其中RootWindowContainer对象作为WindowManagerService的成员变量mRoot 存在。

```java
//frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java
public class WindowManagerService extends IWindowManager.Stub
    implements Watchdog.Monitor, WindowManagerPolicy.WindowManagerFuncs {
    RootWindowContainer mRoot;
    private WindowManagerService(Context context, InputManagerService inputManager,
                                 boolean showBootMsgs, boolean onlyCore, WindowManagerPolicy policy,
                                 ActivityTaskManagerService atm, DisplayWindowSettingsProvider
                                 displayWindowSettingsProvider, Supplier<SurfaceControl.Transaction>                                          transactionFactory,Supplier<Surface> surfaceFactory,
                                 Function<SurfaceSession, SurfaceControl.Builder> surfaceControlFactory) {
        ...
        mRoot = new RootWindowContainer(this);
        mDisplayAreaPolicyProvider = DisplayAreaPolicy.Provider.fromResources(
                mContext.getResources());
        ...
    }
}
```

2）**setWindowManager**

在ActivityTaskManagerService类中，会调用setWindowManager(WindowManagerService wm)方法

```java
//frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java
public class ActivityTaskManagerService extends IActivityTaskManager.Stub {
    RootWindowContainer mRootWindowContainer;
    public void setWindowManager(WindowManagerService wm) {
        synchronized (mGlobalLock) {
            mWindowManager = wm;
            mRootWindowContainer = wm.mRoot;
            mRootWindowContainer.setWindowManager(wm);
            ...
        }
    }
}
```

ActivityTaskManagerService的成员变量mRootWindowContainer 也赋值为RootWindowContainer根对象。最后会调用RootWindowContainer的setWindowManager(wm)方法

```java
//frameworks/base/services/core/java/com/android/server/wm/RootWindowContainer.java
class RootWindowContainer extends WindowContainer<DisplayContent>
    implements DisplayManager.DisplayListener {
    void setWindowManager(WindowManagerService wm) {
        //这里的mWindowManager指的是WindowManagerservice的实例
        mWindowManager = wm;
        //这里的mDisplayManager对应DisplayManagerService服务
        mDisplayManager = mService.mContext.getSystemService(DisplayManager.class);
        mDisplayManager.registerDisplayListener(this, mService.mUiHandler);
        mDisplayManagerInternal = LocalServices.getService(DisplayManagerInternal.class);
        //这里的Display是屏幕，包括虚拟屏幕，默认只有一个屏幕
        final Display[] displays = mDisplayManager.getDisplays();//1
        for (int displayNdx = 0; displayNdx < displays.length; ++displayNdx) {
            final Display display = displays[displayNdx];
            final DisplayContent displayContent = new DisplayContent(display, this);
            addChild(displayContent, POSITION_BOTTOM);//2
            if (displayContent.mDisplayId == DEFAULT_DISPLAY) {
                mDefaultDisplay = displayContent;//3
            }
        }
        //添加Home Task,本文不作讨论
        final TaskDisplayArea defaultTaskDisplayArea = getDefaultTaskDisplayArea();
        defaultTaskDisplayArea.getOrCreateRootHomeTask(ON_TOP);//4
        positionChildAt(POSITION_TOP, defaultTaskDisplayArea.mDisplayContent,
                        false /* includingParents */);//5
    }
}
```

这里只介绍Window层级关系，setWindowManager后边的添加Home Task不在本文讨论，详情可以看[这里](https://blog.csdn.net/q1165328963/article/details/127841486?spm=1001.2014.3001.5502)。

1. 通过屏幕管理对象mDisplayManager得到所有的显示屏幕，然后构造DisplayContent对象
2. 通过addChild方法将DisplayContent对象添加到RootWindowContainer根对象的树状结构中
3. 默认显示屏幕的mDisplayId 是DEFAULT_DISPLAY，mDisplayId 为0作为基本显示屏幕
4. 获取默认屏幕的TaskDisplayArea，创建一个根Home任务
5. 最后把默认屏幕放在RootWindowContainer根对象的孩子的最上面

{{< image classes="fancybox center fig-100" src="/WMS/window/window27.png" thumbnail="/WMS/window/window27.png" title="">}}

#### 2.4.2.4DisplayContent

源码的内容比较多，这里只挑选部分Window容器源码部分

```java
//frameworks/base/services/core/java/com/android/server/wm/DisplayContent.java
class DisplayContent extends RootDisplayArea implements WindowManagerPolicy.DisplayContentInfo {
    final ActivityTaskManagerService mAtmService;
    DisplayContent(Display display, RootWindowContainer root) {
        //DisplayContent继承于RootDisplayArea
        super(root.mWindowManager, "DisplayContent", FEATURE_ROOT);
        ...
        mRootWindowContainer = root;
        //mAtmService是ActivityTaskManagerService实例
        mAtmService = mWmService.mAtmService;
        mDisplay = display;
        mDisplayPolicy = new DisplayPolicy(mWmService, this);
        final Transaction pendingTransaction = getPendingTransaction();
        configureSurfaces(pendingTransaction);
        pendingTransaction.apply();
        // Sets the display content for the children.
        setWindowingMode(WINDOWING_MODE_FULLSCREEN);
        mWmService.mDisplayWindowSettings.applySettingsToDisplayLocked(this);
        ...
    }
}
```

其中configureSurfaces是窗口构建的部分

```java
//frameworks/base/services/core/java/com/android/server/wm/DisplayContent.java
//这个是输入法Window容器
private final ImeContainer mImeWindowsContainer = new ImeContainer(mWmService);
private void configureSurfaces(Transaction transaction) {
    if (mDisplayAreaPolicy == null) {
        //设置策略并构建显示区域层次结构,只有在创建Surface之后才能构建层次结构，这样才能正确地重新生成
        //这里的mWmService是wms
        mDisplayAreaPolicy = mWmService.getDisplayAreaPolicyProvider().instantiate(
            mWmService, this /* content */, this /* root */,
            mImeWindowsContainer);
    }
    ...
}
```

#### 2.4.2.5DisplayAreaPolicy

```java
//frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java
DisplayAreaPolicy.Provider getDisplayAreaPolicyProvider() {
    return mDisplayAreaPolicyProvider;
}
```

这里的mDisplayAreaPolicyProvider实际上是上面2.4.2.3RootWindowContainer构造的时候生成的，DisplayAreaPolicy.Provider.fromResources

```java
//frameworks/base/services/core/java/com/android/server/wm/DisplayAreaPolicy.java
public abstract class DisplayAreaPolicy {
    protected final WindowManagerService mWmService;
    //这个mRoot会在后续的内容中用到
    protected final RootDisplayArea mRoot;
    public interface Provider {
        DisplayAreaPolicy instantiate(WindowManagerService wmService, DisplayContent content,
                                      RootDisplayArea root, DisplayArea.Tokens imeContainer);
        static Provider fromResources(Resources res) {
            //这个name在/frameworks/base/core/res/res/values/config.xml中定义，里面为空
            String name = res.getString(
                com.android.internal.R.string.config_deviceSpecificDisplayAreaPolicyProvider);
            if (TextUtils.isEmpty(name)) {
                return new DisplayAreaPolicy.DefaultProvider();
            }
            //上面的name定义了的话用反射方式来实例化一个Provider
            try {
                return (Provider) Class.forName(name).newInstance();
            } catch (ReflectiveOperationException | ClassCastException e) {
                throw new IllegalStateException("Couldn't instantiate class " + name
                                                + " for config_deviceSpecificDisplayAreaPolicyProvider:"
                                                + " make sure it has a public zero-argument constructor"
                                                + " and implements DisplayAreaPolicy.Provider", e);
            }
        }
    }
}
```

可以看到mDisplayAreaPolicyProvider 的具体类型是可以在系统资源文件中配置的，资源字符串属性为config_deviceSpecificDisplayAreaPolicyProvider，如果为空，类型则为DefaultProvider类型。目前查看，资源文件里没有配置字符值。所以mDisplayAreaPolicyProvider类型则为DefaultProvider类型。

```xml
<!-- /frameworks/base/core/res/res/values/config.xml -->
<resources xmlns:xliff="urn:oasis:names:tc:xliff:document:1.2">
    ...
    <string translatable="false" name="config_deviceSpecificDisplayAreaPolicyProvider"></string>
    ...
</resources>
```

上面的mDisplayAreaPolicy实际上就是new DisplayAreaPolicy.DefaultProvider().instantiate

```java
//frameworks/base/services/core/java/com/android/server/wm/DisplayAreaPolicy.java
public abstract class DisplayAreaPolicy {
    static final class DefaultProvider implements DisplayAreaPolicy.Provider {
        @Override
        public DisplayAreaPolicy instantiate(WindowManagerService wmService,
                                             DisplayContent content, RootDisplayArea root,
                                             DisplayArea.Tokens imeContainer) {
            final TaskDisplayArea defaultTaskDisplayArea = new TaskDisplayArea(content, wmService,
                                                                               "DefaultTaskDisplayArea", FEATURE_DEFAULT_TASK_CONTAINER);//1
            final List<TaskDisplayArea> tdaList = new ArrayList<>();
            tdaList.add(defaultTaskDisplayArea);
            //在整个逻辑显示的根下定义将被支持的特性。该策略将基于此构建DisplayArea层次结构
            //这个特性就是Android12引入的特性，这里的root是上面传入的当前的DisplayContent的this指针
            final HierarchyBuilder rootHierarchy = new HierarchyBuilder(root);//2
            // 设置基本容器(即使显示器不支持输入法)。
            rootHierarchy.setImeContainer(imeContainer).setTaskDisplayAreas(tdaList);//2
            if (content.isTrusted()) {
                //只有信任的显示可以有特色模式建造
                configureTrustedHierarchyBuilder(rootHierarchy, wmService, content);//3
            }
            // 使用上面定义的层次结构实例化策略，这将创建并附加所有必要的显示区域到根目录
            return new DisplayAreaPolicyBuilder().setRootHierarchy(rootHierarchy).build(wmService);//4
        }
    }
}
```

1. 新建一个TaskDisplayArea 实例defaultTaskDisplayArea
2. 接着创建HierarchyBuilder 实例， 将输入法窗口容器和defaultTaskDisplayArea 都设置到它的成员变量里面
3. 接着判断显示屏是否是可信任的，主要是添加一些特色模式
4. 采用建造者模式新建DisplayAreaPolicyBuilder对象，返回的是一个DisplayAreaPolicy实例，即Result对象

1）构造TaskDisplayArea

```java
//frameworks/base/services/core/java/com/android/server/wm/TaskDisplayArea.java
//这里传入的name为DefaultTaskDisplayArea
final class TaskDisplayArea extends DisplayArea<WindowContainer> {
    TaskDisplayArea(DisplayContent displayContent, WindowManagerService service, String name,
                    int displayAreaFeature) {
        this(displayContent, service, name, displayAreaFeature, false /* createdByOrganizer */,
             true /* canHostHomeTask */);
    }

    TaskDisplayArea(DisplayContent displayContent, WindowManagerService service, String name,
                    int displayAreaFeature, boolean createdByOrganizer,
                    boolean canHostHomeTask) {
        super(service, Type.ANY, name, displayAreaFeature);
        mDisplayContent = displayContent;
        mRootWindowContainer = service.mRoot;
        mAtmService = service.mAtmService;
        mCreatedByOrganizer = createdByOrganizer;
        mCanHostHomeTask = canHostHomeTask;
    }
}
```

2）构造HierarchyBuilder

Android12引入的特色模式

```java
//frameworks/base/services/core/java/com/android/server/wm/DisplayAreaPolicyBuilder.java
class DisplayAreaPolicyBuilder {
    static class HierarchyBuilder {
        //这里只有三种Token类型，TASK，IME和TOKENS
        private static final int LEAF_TYPE_TASK_CONTAINERS = 1;
        private static final int LEAF_TYPE_IME_CONTAINERS = 2;
        private static final int LEAF_TYPE_TOKENS = 0;
        //这个是根Root容器
        private final RootDisplayArea mRoot;
        private final ArrayList<DisplayAreaPolicyBuilder.Feature> mFeatures = new ArrayList<>();
        private final ArrayList<TaskDisplayArea> mTaskDisplayAreas = new ArrayList<>();
        @Nullable
        private DisplayArea.Tokens mImeContainer;
        //这里构造传入的root实际上是DisplayContent的实例对象
        HierarchyBuilder(RootDisplayArea root) {
            mRoot = root;
        }

        HierarchyBuilder addFeature(DisplayAreaPolicyBuilder.Feature feature) {
            mFeatures.add(feature);
            return this;
        }

        HierarchyBuilder setTaskDisplayAreas(List<TaskDisplayArea> taskDisplayAreas) {
            mTaskDisplayAreas.clear();
            mTaskDisplayAreas.addAll(taskDisplayAreas);
            return this;
        }

        HierarchyBuilder setImeContainer(DisplayArea.Tokens imeContainer) {
            mImeContainer = imeContainer;
            return this;
        }

        private void build() {
            build(null /* displayAreaGroupHierarchyBuilders */);
        }
        ...
    }
}
```

3）特色模式建造

可以看到这里实际上还是特色模式的构建，其中里面传入的参数是Feature.Builder，显然又用到了建造者的设计模式

```java
//frameworks/base/services/core/java/com/android/server/wm/DisplayAreaPolicy.java
private void configureTrustedHierarchyBuilder(HierarchyBuilder rootHierarchy,
                                              WindowManagerService wmService, DisplayContent content) {
    rootHierarchy.addFeature(new Feature.Builder(wmService.mPolicy, "WindowedMagnification",
                                                 FEATURE_WINDOWED_MAGNIFICATION)
                             .upTo(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY)
                             .except(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY)
                             // Make the DA dimmable so that the magnify window also mirrors the dim layer.
                             .setNewDisplayAreaSupplier(DisplayArea.Dimmable::new)
                             .build());
    if (content.isDefaultDisplay) {
        rootHierarchy.addFeature(new Feature.Builder(wmService.mPolicy, "HideDisplayCutout",
                                                     FEATURE_HIDE_DISPLAY_CUTOUT)
                                 .all()
                                 .except(TYPE_NAVIGATION_BAR, TYPE_NAVIGATION_BAR_PANEL, TYPE_STATUS_BAR,
                                         TYPE_NOTIFICATION_SHADE)
                                 .build())
            .addFeature(new Feature.Builder(wmService.mPolicy, "OneHanded",
                                            FEATURE_ONE_HANDED)
                        .all()
                        .except(TYPE_NAVIGATION_BAR, TYPE_NAVIGATION_BAR_PANEL,
                                TYPE_SECURE_SYSTEM_OVERLAY)
                        .build());
    }
    rootHierarchy
        .addFeature(new Feature.Builder(wmService.mPolicy, "FullscreenMagnification",
                                        FEATURE_FULLSCREEN_MAGNIFICATION)
                    .all()
                    .except(TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY, TYPE_INPUT_METHOD,
                            TYPE_INPUT_METHOD_DIALOG, TYPE_MAGNIFICATION_OVERLAY,
                            TYPE_NAVIGATION_BAR, TYPE_NAVIGATION_BAR_PANEL)
                    .build())
        .addFeature(new Feature.Builder(wmService.mPolicy, "ImePlaceholder",
                                        FEATURE_IME_PLACEHOLDER)
                    .and(TYPE_INPUT_METHOD, TYPE_INPUT_METHOD_DIALOG)
                    .build());
}
```

关于Feature.Builder，内部持有Builder，用于建造Feature类

{{< image classes="fancybox center fig-100" src="/WMS/window/window28.png" thumbnail="/WMS/window/window28.png" title="">}}

Feature分类有5种，分别为"**WindowedMagnification**"、"**HideDisplayCutout**"、"**OneHanded**"、"**FullscreenMagnification**"、"**ImePlaceholder**"

{{< image classes="fancybox center fig-100" src="/WMS/window/window29.png" thumbnail="/WMS/window/window29.png" title="">}}

4）根据上面5种Feature，具体显示区域到根目录

```java
//frameworks/base/services/core/java/com/android/server/wm/DisplayAreaPolicyBuilder.java
Result build(WindowManagerService wmService) {
    //让Window容器的特色模式生效
    validate();
    //在向组层次结构添加窗口之前，将DA组根附加到屏幕层次结构。
    mRootHierarchyBuilder.build(mDisplayAreaGroupHierarchyBuilders);//1
    List<RootDisplayArea> displayAreaGroupRoots = new ArrayList<>(
        mDisplayAreaGroupHierarchyBuilders.size());
    //这个size大小根据上文所知是5，然后构建
    for (int i = 0; i < mDisplayAreaGroupHierarchyBuilders.size(); i++) {
        HierarchyBuilder hierarchyBuilder = mDisplayAreaGroupHierarchyBuilders.get(i);
        hierarchyBuilder.build();//2
        displayAreaGroupRoots.add(hierarchyBuilder.mRoot);//2
    }
    // Use the default function if it is not specified otherwise.
    if (mSelectRootForWindowFunc == null) {
        mSelectRootForWindowFunc = new DefaultSelectRootForWindowFunction(
            mRootHierarchyBuilder.mRoot, displayAreaGroupRoots);//3
    }
    return new Result(wmService, mRootHierarchyBuilder.mRoot, displayAreaGroupRoots,
                      mSelectRootForWindowFunc, mSelectTaskDisplayAreaFunc);//4
}
```

1. 调用mRootHierarchyBuilder.build构造层级
2. 这里没有使用到displayAreaGroupRoots，所以跳过2
3. 如果mSelectRootForWindowFunc 没有设置值的话，设置一个默认的对象
4. 新生成一个Result对象返回，即Result为DisplayAreaPolicy的实例类

#### 2.4.2.6HierarchyBuilder.build

这个内容比较多，分成上下两段来说明

1）上半段

```java
//frameworks/base/services/core/java/com/android/server/wm/DisplayAreaPolicyBuilder.java
private void build(@Nullable List<HierarchyBuilder> displayAreaGroupHierarchyBuilders) {
    final WindowManagerPolicy policy = mRoot.mWmService.mPolicy;
    final int maxWindowLayerCount = policy.getMaxWindowLayer() + 1;
    final DisplayArea.Tokens[] displayAreaForLayer =
        new DisplayArea.Tokens[maxWindowLayerCount];
    final Map<Feature, List<DisplayArea<WindowContainer>>> featureAreas =
        new ArrayMap<>(mFeatures.size());
    for (int i = 0; i < mFeatures.size(); i++) {
        featureAreas.put(mFeatures.get(i), new ArrayList<>());
    }

    PendingArea[] areaForLayer = new PendingArea[maxWindowLayerCount];
    final PendingArea root = new PendingArea(null, 0, null);
    Arrays.fill(areaForLayer, root);

    // 创建显示区域以覆盖所有已定义的功能
    final int size = mFeatures.size();
    for (int i = 0; i < size; i++) {
        final Feature feature = mFeatures.get(i);
        PendingArea featureArea = null;
        for (int layer = 0; layer < maxWindowLayerCount; layer++) {
            if (feature.mWindowLayers[layer]) {
                // 该特性将应用于该窗口层。
                //我们需要为它找到一个显示区域:
                //我们可以重用现有的一个，如果它是为前一层的这个特性创建的
                //并且应用到前一层的最后一个特性与应用到当前层的特性相同(所以它们可以共享相同的父DisplayArea)
                if (featureArea == null || featureArea.mParent != areaForLayer[layer]) {
                    // 没有合适的显示区域:
                    //在之前的区域下创建一个新的图层(作为父层)。
                    featureArea = new PendingArea(feature, layer, areaForLayer[layer]);
                    areaForLayer[layer].mChildren.add(featureArea);
                }
                areaForLayer[layer] = featureArea;
            } else {
                featureArea = null;
            }
        }
    }
    ...
}
```

根据上面的Feature类，创建对应的联系

对应的PendingArea结构

{{< image classes="fancybox center fig-100" src="/WMS/window/window30.png" thumbnail="/WMS/window/window30.png" title="">}}

第一个循环

{{< image classes="fancybox center fig-100" src="/WMS/window/window31.png" thumbnail="/WMS/window/window31.png" title="">}}

第二个循环

{{< image classes="fancybox center fig-100" src="/WMS/window/window32.png" thumbnail="/WMS/window/window32.png" title="">}}

为了更加方便的理解，简化上述过程，用PA代表PendingArea的实例对象，最终上半段的结构如下所示

{{< image classes="fancybox center fig-100" src="/WMS/window/window33.png" thumbnail="/WMS/window/window33.png" title="">}}

2）下半段

```java
//frameworks/base/services/core/java/com/android/server/wm/DisplayAreaPolicyBuilder.java
private void build(@Nullable List<HierarchyBuilder> displayAreaGroupHierarchyBuilders) {
    ...
    // Create Tokens as leaf for every layer.
    PendingArea leafArea = null;
    int leafType = LEAF_TYPE_TOKENS;
    for (int layer = 0; layer < maxWindowLayerCount; layer++) {
        int type = typeOfLayer(policy, layer);
        // Check whether we can reuse the same Tokens with the previous layer. This happens
        // if the previous layer is the same type as the current layer AND there is no
        // feature that applies to only one of them.
        if (leafArea == null || leafArea.mParent != areaForLayer[layer]
            || type != leafType) {
            // Create a new Tokens for this layer.
            leafArea = new PendingArea(null /* feature */, layer, areaForLayer[layer]);
            areaForLayer[layer].mChildren.add(leafArea);
            leafType = type;
            if (leafType == LEAF_TYPE_TASK_CONTAINERS) {
                // We use the passed in TaskDisplayAreas for task container type of layer.
                // Skip creating Tokens even if there is no TDA.
                addTaskDisplayAreasToApplicationLayer(areaForLayer[layer]);
                addDisplayAreaGroupsToApplicationLayer(areaForLayer[layer],
                                                       displayAreaGroupHierarchyBuilders);
                leafArea.mSkipTokens = true;
            } else if (leafType == LEAF_TYPE_IME_CONTAINERS) {
                // We use the passed in ImeContainer for ime container type of layer.
                // Skip creating Tokens even if there is no ime container.
                leafArea.mExisting = mImeContainer;
                leafArea.mSkipTokens = true;
            }
        }
        leafArea.mMaxLayer = layer;
    }
    root.computeMaxLayer();

    // 构建了一个PendingAreas树来表示层次结构，现在创建并将真正的DisplayAreas附加到根节点上
    //这个root是PendingArea的实例，
    root.instantiateChildren(mRoot, displayAreaForLayer, 0, featureAreas);

    // 通知根节点我们已经完成了所有displayarea的附加。缓存所有与特性相关的集合，以便快速访问
    // 这个mRoot是RootDisplayArea类型，传入的mRoot是上文传入的DisplayContent的this指针
    mRoot.onHierarchyBuilt(mFeatures, displayAreaForLayer, featureAreas);
}
```

下半段添加了一些叶子结点

{{< image classes="fancybox center fig-100" src="/WMS/window/window34.png" thumbnail="/WMS/window/window34.png" title="">}}

再次简化如下

{{< image classes="fancybox center fig-100" src="/WMS/window/window35.png" thumbnail="/WMS/window/window35.png" title="">}}

子节点的PendingArea对象树形结构构建完成，接着root.instantiateChildren构建窗口层级图

```java
//frameworks/base/services/core/java/com/android/server/wm/DisplayAreaPolicyBuilder.java
class DisplayAreaPolicyBuilder { 
    ...
    static class PendingArea {
        ...
        void instantiateChildren(DisplayArea<DisplayArea> parent, DisplayArea.Tokens[] areaForLayer,
                                 int level, Map<Feature, List<DisplayArea<WindowContainer>>> areas) {
            mChildren.sort(Comparator.comparingInt(pendingArea -> pendingArea.mMinLayer));
            for (int i = 0; i < mChildren.size(); i++) {
                final PendingArea child = mChildren.get(i);
                final DisplayArea area = child.createArea(parent, areaForLayer);
                if (area == null) {
                    // TaskDisplayArea and ImeContainer can be set at different hierarchy, so it can
                    // be null.
                    continue;
                }
                parent.addChild(area, WindowContainer.POSITION_TOP);
                if (child.mFeature != null) {
                    areas.get(child.mFeature).add(area);
                }
                child.instantiateChildren(area, areaForLayer, level + 1, areas);
            }
        }
        
        @Nullable
        private DisplayArea createArea(DisplayArea<DisplayArea> parent,
                DisplayArea.Tokens[] areaForLayer) {
            if (mExisting != null) {
                if (mExisting.asTokens() != null) {
                    // Store the WindowToken container for layers
                    fillAreaForLayers(mExisting.asTokens(), areaForLayer);
                }
                return mExisting;
            }
            if (mSkipTokens) {
                return null;
            }
            DisplayArea.Type type;
            if (mMinLayer > APPLICATION_LAYER) {
                type = DisplayArea.Type.ABOVE_TASKS;
            } else if (mMaxLayer < APPLICATION_LAYER) {
                type = DisplayArea.Type.BELOW_TASKS;
            } else {
                type = DisplayArea.Type.ANY;
            }
            if (mFeature == null) {
                final DisplayArea.Tokens leaf = new DisplayArea.Tokens(parent.mWmService, type,
                        "Leaf:" + mMinLayer + ":" + mMaxLayer);
                fillAreaForLayers(leaf, areaForLayer);
                return leaf;
            } else {
                return mFeature.mNewDisplayAreaSupplier.create(parent.mWmService, type,
                        mFeature.mName + ":" + mMinLayer + ":" + mMaxLayer, mFeature.mId);
            }
        }

        private void fillAreaForLayers(DisplayArea.Tokens leaf, DisplayArea.Tokens[] areaForLayer) {
            for (int i = mMinLayer; i <= mMaxLayer; i++) {
                areaForLayer[i] = leaf;
            }
        }
    }
}
```

这里能看到PendingArea类的mExisting 和mSkipTokens属性的使用。

- mExisting 存在直接就返回它，不会重新生成新的

  如果mExisting 不存在，并且mSkipTokens= true，这个时候，会返回null

  如果mExisting 不存在，并且mSkipTokens= false，则会新生成DisplayArea对象

- 新生成的DisplayArea对象，如果mFeature 为null，则为DisplayArea.Tokens对象

  不然，则会调用生成具体的对象

- 前面添加特色模式时，mNewDisplayAreaSupplier设置为DisplayArea.Dimmable::new

  如果不设置，它默认为DisplayArea::new

- 我们还看到，新生成的DisplayArea对象的type，是根据PendingArea类的mMinLayer 和mMaxLayer 来决定

  如果新生成的DisplayArea对象是Tokens 类型，会填充参数areaForLayer数组，将对应层级的Tokens对象放入其中





#### 2.4.2.7窗口层级分析WindowContainer相关子类关联

这里补充上面提到但没有讲述的Window容器，WindowState的容器，直接持有WindowState容器，WindowToken的容器，ActivityRecord的容器，Task的容器，DisplayArea相关容器，RootWindowContainer容器。

##### 2.4.2.7.1WindowState的容器

直接持有WindowState的类，根据上面的android13 Window容器架构图可以初步得知，WindowToken、ActivityRecord和WallpaperWindowToken

##### 2.4.2.7.2 WindowToken

```java
//frameworks/base/services/core/java/com/android/server/wm/WindowToken.java
class WindowToken extends WindowContainer<WindowState> {}
```

WMS中存放一组相关联的窗口的容器。通常是ActivityRecord，它是Activity在WMS的表示，就如WindowState是Window在WMS的表示，用来显示窗口。

比如StatusBar：

```shell
$ dumpsys activity containers
#0 WindowToken{ef15750 android.os.BinderProxy@4562c02} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
#0 d3cbd49 StatusBar type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
```

{{< image classes="fancybox center fig-100" src="/WMS/window/window37.png" thumbnail="/WMS/window/window37.png" title="">}}

##### 2.4.2.7.3 ActivityRecord

```java
//frameworks/base/services/core/java/com/android/server/wm/ActivityRecord.java
final class ActivityRecord extends WindowToken implements WindowManagerService.AppFreezeListener {}
```

ActivityRecord是WindowToken的子类，如上面所说，在WMS中一个**ActivityRecord对象就代表一个Activity对象**。

这里有几点需要说明：

1. 可以根据窗口的添加方式将窗口分为App窗口和非App窗口，或者换个说法，Activity窗口和非Activity窗口，我个人习惯叫App窗口和非App窗口。

   **App窗口**(Activity或者Dialog)父容器为**ActivityRecord**

   这类窗口由系统自动创建，不需要App主动去调用ViewManager.addView去添加一个窗口，比如我写一个Activity或者Dialog，系统就会在合适的时机为Activity或者Dialog调用ViewManager.addView去向WindowManager添加一个窗口。这类窗口在创建的时候，其父容器为ActivityRecord。

   **非App窗口**(NavigationBar)父容器为**WindowToken**

   这类窗口需要App主动去调用ViewManager.addView来添加一个窗口，比如NavigationBar窗口的添加，需要SystemUI主动去调用ViewManager.addView来为NavigationBar创建一个新的窗口。这类窗口在创建的时候，其父容器为WindowToken。

2. WindowToken既然是WindowState的直接父容器，那么每次添加窗口的时候，就需要创建一个WindowToken，或者一个ActivityRecord。

3. 上述情况也有例外，即存在WindowToken的复用，因为一个WindowToken中可能不只一个WindowState对象，说明第二个WindowState创建的时候，复用了第一个WindowState创建的时候生成的WindowToken。如果两个WindowState能被放进一个WindowToken中，那么这两个WindowState之间就必须有联系。

   {{< image classes="fancybox center fig-100" src="/WMS/window/window5.png" thumbnail="/WMS/window/window5.png" title="">}}

   在WMS.addWindow方法的部分片段：

```java
//frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java
public int addWindow(Session session, IWindow client, LayoutParams attrs, int viewVisibility,
                     int displayId, int requestUserId, InsetsVisibilities requestedVisibilities,
                     InputChannel outInputChannel, InsetsState outInsetsState,
                     InsetsSourceControl[] outActiveControls) {
    ...
    WindowToken token = displayContent.getWindowToken(
        hasParent ? parentWindow.mAttrs.token : attrs.token);
    ...
}
```

可知，如果两个窗口的WindowManager.LayoutParams.token指向同一个对象，那么这两个WindowState就会被放入同一个WindowToken中。但是通常来说，只有由**同一个Activity的两个窗口**才有可能被放入到一个WindowToken中，**比如典型的Launcher**

```shell
$ dumpsys activity containers
#0 ActivityRecord{31ce9e6 u0 com.google.android.apps.nexuslauncher/.NexusLauncherActivity t191}
#1 220afda com.google.android.apps.nexuslauncher/com.google.android.apps.nexuslauncher.NexusLauncherActivity
#0 4cf47c3 com.google.android.apps.nexuslauncher/com.google.android.apps.nexuslauncher.NexusLauncherActivity
```
{{< image classes="fancybox center fig-100" src="/WMS/window/window38.png" thumbnail="/WMS/window/window38.png" title="">}}

此外分别来自**两个不同的Activity的两个窗口是不会被放入同一个WindowToken中**的，因此更一般的情况是，一个WindowToken或ActivityRecord只包含一个WindowState。

##### 2.4.2.7.4 WallpaperWindowToken

```java
//frameworks/base/services/core/java/com/android/server/wm/WallpaperWindowToken.java
class WallpaperWindowToken extends WindowToken {}
```

其实WindowToken还有一个子类，WallpaperWindowToken，这类的WindowToken用来存放和Wallpaper相关的窗口。

```shell
$ dumpsys activity containers
#0 WallpaperWindowToken{dc5772f token=android.os.Binder@985f410} type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
#0 1ea91d com.android.systemui.ImageWallpaper type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
```

{{< image classes="fancybox center fig-100" src="/WMS/window/window39.png" thumbnail="/WMS/window/window39.png" title="">}}

上面讨论ActivityRecord的时候，我们将窗口划分为App窗口和非App窗口。在引入了WallpaperWindowToken后，我们继续将非App窗口划分为两类，Wallpaper窗口和非Wallpaper窗口。

Wallpaper窗口的层级是比App窗口的层级低的，因此这里我们可以按照层级这一角度将窗口划分为：

- **App之上的窗口**，父容器为WindowToken，如StatusBar和NavigationBar（这个层级大于2）
- **App窗口**，父容器为ActivityRecord，如Launcher（这个层级为2）
- **App之下的窗口**，父容器为WallpaperWindowToken，如ImageWallpaper窗口（这个层级为1）

##### 2.4.2.7.5 WindowToken的容器 （DisplayArea.Tokens）

可以容纳WindowToken的容器，搜索之后，发现为定义在DisplayArea中的内部类Tokens。关于DisplayArea的内容放在2.4.2.7.10中

```java
//frameworks/base/services/core/java/com/android/server/wm/DisplayArea.java
public class DisplayArea<T extends WindowContainer> extends WindowContainer<T> {
    public static class Tokens extends DisplayArea<WindowToken> {}
}
```

##### 2.4.2.7.6 ActivityRecord的容器 （Task）

```java
//frameworks/base/services/core/java/com/android/server/wm/Task.java
class Task extends TaskFragment {}
//frameworks/base/services/core/java/com/android/server/wm/TaskFragment.java
class TaskFragment extends WindowContainer<WindowContainer> {}
```

> 关于Task相关
>
> 如果该应用没有Task存在（应用最近没有使用过），则会创建一个新的Task，并且该应用的“主”Activity 将会作为堆栈的根 Activity 打开。在当前 Activity 启动另一个 Activity 时，新的 Activity 将被推送到堆栈顶部并获得焦点。上一个 Activity 仍保留在堆栈中，但会停止。当 Activity 停止时，系统会保留其界面的当前状态。当用户按返回按 钮时，当前 Activity 会从堆栈顶部退出（该 Activity 销毁），上一个 Activity 会恢复（界面会恢复到上一个状态）。堆栈中的 Activity 永远不会重新排列，只会被送入和退出，在当前 Activity 启动时被送入堆栈，在用户使用返回按钮离开时从堆栈中退出。因此，返回堆栈按照“后进先出”的对象结构运作。图 1 借助一个时间轴直观地显示了这种行为。该时间轴显示了 Activity 之间的进展以及每个时间点的当前返回堆栈。
> {{< image classes="fancybox center fig-100" src="/WMS/window/window41.png" thumbnail="/WMS/window/window41.png" title="">}}

WMS中的Task类的作用和以上说明基本一致，用来管理ActivityRecord。

```shell
$ dumpsys activity containers
#6 Task=67 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
#0 ActivityRecord{cb9b141 u0 com.google.android.apps.messaging/.ui.ConversationListActivity t67} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
#0 d01e570 com.google.android.apps.messaging/com.google.android.apps.messaging.ui.ConversationListActivity type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
```

{{< image classes="fancybox center fig-100" src="/WMS/window/window40.png" thumbnail="/WMS/window/window40.png" title="">}}

接着再启动Message的另外一个界面

```shell
$ dumpsys activity containers
#6 Task=67 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
#1 ActivityRecord{77ab155 u0 com.google.android.apps.messaging/.conversation.screen.ConversationActivity t67} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
#0 e50283c com.google.android.apps.messaging/com.google.android.apps.messaging.conversation.screen.ConversationActivity type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
#0 ActivityRecord{cb9b141 u0 com.google.android.apps.messaging/.ui.ConversationListActivity t67} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
#0 d01e570 com.google.android.apps.messaging/com.google.android.apps.messaging.ui.ConversationListActivity type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
```

{{< image classes="fancybox center fig-100" src="/WMS/window/window42.png" thumbnail="/WMS/window/window42.png" title="">}}

Task也支持嵌套的，从Task类的定义也可以看出来，Task的嵌套多用于多窗口模式，如**分屏**

```shell
$ dumpsys activity containers
#2 Task=5 type=standard mode=split-screen-primary override-mode=split-screen-primary requested-bounds=[0,0][720,770] bounds=[0,0][720,770]
#0 Task=67 type=standard mode=split-screen-primary override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,770]
#0 ActivityRecord{cb9b141 u0 com.google.android.apps.messaging/.ui.ConversationListActivity t67} type=standard mode=split-screen-primary override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,770]
#0 d01e570 com.google.android.apps.messaging/com.google.android.apps.messaging.ui.ConversationListActivity type=standard mode=split-screen-primary override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,770]
```

{{< image classes="fancybox center fig-100" src="/WMS/window/window43.png" thumbnail="/WMS/window/window43.png" title="">}}

##### 2.4.2.7.7 Task的容器 （TaskDisplayArea）

```java
//frameworks/base/services/core/java/com/android/server/wm/TaskDisplayArea.java
final class TaskDisplayArea extends DisplayArea<WindowContainer> {}
```

TaskDisplayArea，代表了屏幕上一块专门用来存放App窗口的区域。它的子容器可能是**Task**或者是**TaskDisplayArea**。

##### 2.4.2.7.8 DisplayArea容器

```java
//frameworks/base/services/core/java/com/android/server/wm/DisplayArea.java
public class DisplayArea<T extends WindowContainer> extends WindowContainer<T> {}    
```

DisplayArea，是DisplayContent之下的对WindowContainer进行分组的容器。

DisplayArea**受DisplayAreaPolicy管理**，DisplayArea可以**包含嵌套DisplayArea**。

DisplayArea有三种风格，用来保证窗口能够拥有正确的Z轴顺序：

- **BELOW_TASKS**，只能包含Task之下的的DisplayArea和WindowToken
- **ABOVE_TASKS**，只能包含Task之上的DisplayArea和WindowToken
- **ANY**，能包含任何种类的DisplayArea、WindowToken或是Task容器

这里和我们之前分析WindowToken的时候逻辑是一样的，DisplayArea的子类有一个专门用来存放Task的容器类，TaskDisplayArea。层级高于TaskDisplayArea的DisplayArea，会被归类为ABOVE_TASKS，层级低于TaskDisplayArea则属于ABOVE_TASKS。

DisplayArea有三个直接子类，**TaskDisplayArea**，**DisplayArea.Tokens**和**DisplayArea.Dimmable**。

##### 2.4.2.7.9 TaskDisplayArea

2.4.2.7.7中已经列出了具体的类，TaskDisplayArea代表了屏幕上的一个包含App类型的WindowContainer的区域。它的子节点可以是Task，或者是TaskDisplayArea。目前在代码中，创建TaskDisplayArea的地方只有一处，即**TaskDisplayArea存放Task的容器**。

在手机的近期任务列表界面将所有App都清掉后，查看一下此时的TaskDisplayArea情况

```shell
$ dumpsys activity containers
#0 DefaultTaskDisplayArea type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
#5 Task=1 type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
#0 Task=191 type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
#0 ActivityRecord{31ce9e6 u0 com.google.android.apps.nexuslauncher/.NexusLauncherActivity t191} type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
#1 220afda com.google.android.apps.nexuslauncher/com.google.android.apps.nexuslauncher.NexusLauncherActivity type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
#0 4cf47c3 com.google.android.apps.nexuslauncher/com.google.android.apps.nexuslauncher.NexusLauncherActivity type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
#4 Task=4 type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
#3 Task=3 type=undefined mode=split-screen-secondary override-mode=split-screen-secondary requested-bounds=[0,1498][1440,2960] bounds=[0,1498][1440,2960]
#2 Task=2 type=undefined mode=split-screen-primary override-mode=split-screen-primary requested-bounds=[0,0][1440,1464] bounds=[0,0][1440,1464]
#1 Task=5 type=undefined mode=multi-window override-mode=multi-window requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
#0 Task=6 type=undefined mode=multi-window override-mode=multi-window requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
```

正常来说，App端调用startActivity后，WMS会为新创建的ActivityRecord对象创建Task，用来存放这个ActivityRecord。当Task中的所有ActivityRecord都被移除后，这个Task就会被移除，就如Task#1，或者Task#191。但是对于Task#4之类的Task来说并不是这样。这些Task是WMS启动后就由TaskOrganizerController创建的，这些Task并没有和某一具体的App联系起来，因此当它里面的子WindowContainer被移除后，这个Task也不会被销毁，比如分屏Task#3和Task#2
{{< image classes="fancybox center fig-100" src="/WMS/window/window44.png" thumbnail="/WMS/window/window44.png" title="">}}

##### 2.4.2.7.10 DisplayArea.Tokens

```java
//frameworks/base/services/core/java/com/android/server/wm/DisplayArea.java
public class DisplayArea<T extends WindowContainer> extends WindowContainer<T> {
    public static class Tokens extends DisplayArea<WindowToken> {}
}
```

Tokens是DisplayArea的内部类，从其定义即可看出，它是一个只能包含WindowToken对象的DisplayArea类型的容器，那么其内部层级结构就相对简单，比如StatusBar

```shell
$ dumpsys activity containers
#0 Leaf:17:17 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
#0 WindowToken{ef15750 android.os.BinderProxy@4562c02} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
#0 d3cbd49 StatusBar type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
```

{{< image classes="fancybox center fig-100" src="/WMS/window/window45.png" thumbnail="/WMS/window/window45.png" title="">}}

##### 2.4.2.7.11DisplayArea.Dimmable

```java
//frameworks/base/services/core/java/com/android/server/wm/DisplayArea.java
public class DisplayArea<T extends WindowContainer> extends WindowContainer<T> {
    static class Dimmable extends DisplayArea<DisplayArea> {}
}
```

Dimmable也是DisplayArea的内部类，从名字可以看出，这类的DisplayArea可以**添加模糊效果**，并且Dimmable也是一个DisplayArea类型的DisplayArea容器。

它内部有一个Dimmer对象

```java
//frameworks/base/services/core/java/com/android/server/wm/DisplayArea.java
private final Dimmer mDimmer = new Dimmer(this);
```

可以通过Dimmer对象施加模糊效果，模糊图层可以插入到以该Dimmable对象为根节点的层级结构之下的任意两个图层之间。

它有一个直接子类，RootDisplayArea。

##### 2.4.2.7.12 DisplayContent.ImeContainer

DisplayArea.Tokens有一个子类DisplayContent.ImeContainer

```java
//frameworks/base/services/core/java/com/android/server/wm/DisplayContent.java
class DisplayContent extends RootDisplayArea implements WindowManagerPolicy.DisplayContentInfo {
    private static class ImeContainer extends DisplayArea.Tokens {}
}
```

DisplayContent的内部类，ImeContainer，是存放输入法窗口的容器。它继承的是DisplayArea.Tokens，说明它是一个只能存放WindowToken容器。

```shell
$ dumpsys activity containers
#0 ImeContainer type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
#0 WindowToken{5e01794 android.os.Binder@579cfe7} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
#0 4435738 InputMethod type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
```

{{< image classes="fancybox center fig-100" src="/WMS/window/window46.png" thumbnail="/WMS/window/window46.png" title="">}}

##### 2.4.2.7.13 RootDisplayArea

```java
//frameworks/base/services/core/java/com/android/server/wm/RootDisplayArea.java
class RootDisplayArea extends DisplayArea.Dimmable {}
```

RootDisplayArea，是一个DisplayArea层级结构的根节点。

它可以是：

- **DisplayContent**，作为整个屏幕的DisplayArea层级结构根节点。
- **DisplayAreaGroup**，作为屏幕上部分区域对应的DisplayArea层级结构的根节点。

这又引申出了RootDisplayArea的两个子类，**DisplayContent**和**DisplayAreaGroup**(这个是车载用到的，暂时不考虑)。

##### 2.4.2.7.14 DisplayContent

```java
//frameworks/base/services/core/java/com/android/server/wm/DisplayContent.java
class DisplayContent extends RootDisplayArea implements WindowManagerPolicy.DisplayContentInfo {}
```

DisplayContent，代表了一个实际的屏幕，那么它作为一个屏幕上的DisplayArea层级结构的根节点也没有毛病。

隶属于同一个DisplayContent的窗口将会被显示在同一个屏幕中。每一个DisplayContent都对应着唯一ID，比如**默认屏幕是0**

那么以DisplayContent为根节点的DisplayArea层级结构可能是

{{< image classes="fancybox center fig-100" src="/WMS/window/window48.png" thumbnail="/WMS/window/window48.png" title="">}}

##### 2.4.2.7.15 RootWindowContainer

```java
//frameworks/base/services/core/java/com/android/server/wm/RootWindowContainer.java
class RootWindowContainer extends WindowContainer<DisplayContent>
        implements DisplayManager.DisplayListener {}
```

从名字也可以看出，这个类代表当前设备的根WindowContainer，就像View层级结构中的DecorView一样，它是DisplayContent的容器。由于DisplayContent代表了一个屏幕，且RootWindowContainer能够作为DisplayContent的父容器，这也说明了Android是支持多屏幕的，展开来说就是包括一个内部屏幕（内置于手机或平板电脑中的屏幕）、一个外部屏幕（如通过 HDMI 连接的电视）以及一个或多个虚拟屏幕。

如果我们在开发者选项里通过“Simulate secondary displays”开启另一个虚拟屏幕(或者是辅助模拟显示设备)，此时的情况是：

```shell
$ dumpsys activity containers
ROOT type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
  #1 Display 0 name="Built-in screen" type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][720,1612] bounds=[0,0][720,1612]
  #0 Display 2 name="Overlay #1" type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][720,480] bounds=[0,0][720,480]
```

“Root”即RootWindowContainer，“Display 0”代表默认屏幕，“Display 2”是我们开启的虚拟屏幕。

{{< image classes="fancybox center fig-100" src="/WMS/window/window47.png" thumbnail="/WMS/window/window47.png" title="">}}

#### 2.4.2.8最终构建窗口层级图

{{< image classes="fancybox center fig-100" src="/WMS/window/window36.png" thumbnail="/WMS/window/window36.png" title="">}}

#### 2.4.2.9 总结

除了WindowState可以显示图像以外，大部分的WindowContainer，如WindowToken、TaskDisplayArea是不会有内容显示的，都只是一个**抽象的容器概念**。WMS如果只为了管理窗口，WMS也可以不创建这些个WindowContainer类，直接用一个类似列表的东西，将屏幕上显示的窗口全部添加到这个列表中，通过这一个列表来对所有的窗口进行管理。但是为了更有逻辑地管理屏幕上显示的窗口，还是需要创建各种各样的窗口容器类，即WindowContainer及其子类，来对WindowState进行分类，从而对窗口进行系统化的管理。

|                  设计复杂的Window容器意义                   |                           详细描述                           |
| :---------------------------------------------------------: | :----------------------------------------------------------: |
| `WindowContainer`类都有着鲜明的上下级关系，一般不能越级处理 | 比如`DefaultTaskDisplayArea`只用来管理调度`Task`，`Task`用来管理调度`ActivityRecord`，而`DefaultTaskDisplayArea`不能直接越过`Task`去调度`Task`中的`ActivityRecord`。这样`TaskDisplayArea`只需要关注它的子容器们，即Task的管理，`ActivityRecord`相关的事务让`Task`去操心就好，每一级与每一级之间的边界都很清晰，不会在管理逻辑上出现混乱，比如`DefaultTaskDisplayArea`强行去调整一个`ActivityRecord`的位置，导致这个`ActivityRecord`跳出它所在的`Task`，变成和`Task`一个层级 |
|           保证了每一个`WiindowContainer`不会越界            | 比如我点击`HOME`键回到`Launcher`，此时`DefaultTaskDisplayArea`就会把`Launcher`对应的`Task#1`，移动到它内部栈的栈顶，而这仅限于`DefaultTaskDisplayArea`内部的调整，这一点保证了`Launcher`的窗口将永远不可能高于`StatusBar`窗口，也不会低于`Wallpaper`窗口 |




### 2.4.3实例分析Window容器

这里举一个实例，以android13设备手机为例

| 手机型号    | 三星Galaxy F52 5G      |
| ----------- | ---------------------- |
| Android版本 | Android13              |
| 测试场景    | 屏幕点亮的Launcher界面 |
| 后台应用    | 无                     |

```shell
dumpf52x:/ $ dumpsys activity containers
ACTIVITY MANAGER CONTAINERS (dumpsys activity containers)
ROOT type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
  #0 Display 0 name="内置屏幕" type=undefined mode=fullscreen override-mode=fullscreen
 requested-bounds=[0,0][1080,2408] bounds=[0,0][1080,2408]
   #2 Leaf:36:36 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
   #1 HideDisplayCutout:32:35 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
    #2 OneHanded:34:35 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
     #0 FullscreenMagnification:34:35 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
      #0 Leaf:34:35 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
    #1 FullscreenMagnification:33:33 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
     #0 Leaf:33:33 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
      #0 WindowToken{258c658 type=2015 android.os.BinderProxy@d28af3b} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
       #0 c4b73b1 LockscreenShortcutBlur type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
    #0 OneHanded:32:32 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
     #0 Leaf:32:32 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
   #0 WindowedMagnification:0:31 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
    #6 HideDisplayCutout:26:31 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
     #0 OneHanded:26:31 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
      #2 FullscreenMagnification:29:31 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
       #0 Leaf:29:31 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
        #0 WindowToken{b23fa21 type=2016 android.os.BinderProxy@b432f6e} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
         #0 31d42b ShellDropTarget type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
      #1 Leaf:28:28 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
      #0 FullscreenMagnification:26:27 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
       #0 Leaf:26:27 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
    #5 Leaf:24:25 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
     #2 WindowToken{d14c3be type=2024 android.os.BinderProxy@da64a79} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
      #0 a381535 SecondaryHomeHandle0 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
     #1 WindowToken{23be81 type=2024 android.os.BinderProxy@3ab3a68} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
      #0 c099b26 EdgeBackGestureHandler0 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
     #0 WindowToken{62fdc7 type=2019 android.os.BinderProxy@1c389e1} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
      #0 2aa9419 NavigationBar0 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
    #4 HideDisplayCutout:18:23 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
     #0 OneHanded:18:23 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
      #0 FullscreenMagnification:18:23 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
       #0 Leaf:18:23 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
        #0 WindowToken{ece377f type=2226 android.os.BinderProxy@d8c209e} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
         #0 895a8aa com.samsung.android.app.cocktailbarservice/com.samsung.android.app.cocktailbarservice.CocktailBarService type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
    #3 OneHanded:17:17 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
     #0 FullscreenMagnification:17:17 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
      #0 Leaf:17:17 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
       #0 WindowToken{11fdd2b type=2040 android.os.BinderProxy@e1219a5} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
        #0 6f61b46 NotificationShade type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
    #2 HideDisplayCutout:16:16 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
     #0 OneHanded:16:16 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
      #0 FullscreenMagnification:16:16 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
       #0 Leaf:16:16 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
    #1 OneHanded:15:15 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
     #0 FullscreenMagnification:15:15 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
      #0 Leaf:15:15 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
       #0 WindowToken{b620a94 type=2000 android.os.BinderProxy@7d4e01} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
        #0 d5e5283 StatusBar type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
    #0 HideDisplayCutout:0:14 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
     #0 OneHanded:0:14 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
      #1 ImePlaceholder:13:14 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
       #0 ImeContainer type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
        #1 WindowToken{33d4588 type=2011 android.os.Binder@1bfed2b} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
         #0 dd5ca43 InputMethod type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
        #0 WindowToken{d802601 type=2011 android.os.Binder@7a28fe8} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
      #0 FullscreenMagnification:0:12 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
       #2 Leaf:3:12 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
        #0 WindowToken{6102e33 type=2038 android.os.BinderProxy@81db1a2} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
       #1 DefaultTaskDisplayArea type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
        #2 Task=22 type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
         #0 Task=23 type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
          #0 ActivityRecord{ba66eb5 u0 com.sec.android.app.launcher/.activities.LauncherActivity} t23} type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
           #0 6e2f047 com.sec.android.app.launcher/com.sec.android.app.launcher.activities.LauncherActivity type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
            #0 bec7e6c com.samsung.android.app.spage type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
        #1 Task=2 type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
        #0 Task=3 type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
         #1 Task=5 type=undefined mode=multi-window override-mode=multi-window requested-bounds=[0,0][1080,1222] bounds=[0,0][1080,1222]
         #0 Task=4 type=undefined mode=multi-window override-mode=multi-window requested-bounds=[0,1245][1080,2408] bounds=[0,1245][1080,2408]
       #0 Leaf:0:1 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
        #0 WallpaperWindowToken{c835f1c token=android.os.Binder@7d6e08f} type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
         #0 3d43d44 com.android.systemui.ImageWallpaper type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2408]
```

window层次逻辑图如下所示，标红的为叶子结点，同色的为同一层级

{{< image classes="fancybox center fig-100" src="/WMS/window/window8.png" thumbnail="/WMS/window/window8.png" title="">}}

## 2.5addView/updateViewLayout/removeView

在 Android 系统中，**屏幕的抽象是 DisplayContent** ，在屏幕的抽象上，通过不同的窗口，展示不同的应用程序页面和一些其他UI 组件（例如 Dialog、状态栏等）。Window 通过在 Z 轴叠放，从而实现了复杂的交互逻辑。

Window 实际是 View 的**直接管理者**，例如笔者点击屏幕，对应 Activity 里面收到点击事件后，会首先通过Activity对应的 window 将事件传递到 DecorView，最后再分发到我们的 View 上。而Window 是一个处理顶级窗口外观和行为策略的抽象基类，它的具体实现是 PhoneWindow 类，**PhoneWindow 对 View 进行管理**。

### 2.5.1Window和Activity以及WindowManager建立联系

这里以Activity为例，其他Dialog、DreamOverlay、FloatWindow等可以自行查找源码阅读。

Activity在启动的过程中，会调用它的attach方法进行Window的创建

```java
//frameworks/base/core/java/android/app/Activity.java
final void attach(Context context, ActivityThread aThread,
                  Instrumentation instr, IBinder token, int ident,
                  Application application, Intent intent, ActivityInfo info,
                  CharSequence title, Activity parent, String id,
                  NonConfigurationInstances lastNonConfigurationInstances,
                  Configuration config, String referrer, IVoiceInteractor voiceInteractor,
                  Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken,
                  IBinder shareableActivityToken) {
    ...
    //[2.5.1.1]
    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    ...
    //[2.5.1.2]    
    mWindow.setWindowManager(
        (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
        mToken, mComponent.flattenToString(),
        (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
}
```

#### 2.5.1.1PhoneWindow

**创建了一个PhoneWindow，并传入当前Activity的this对象**， PhoneWindow类，继承了Window

```java
//frameworks/base/core/java/com/android/internal/policy/PhoneWindow.java
public class PhoneWindow extends Window {
    public PhoneWindow(Context context, Window preservedWindow,
            ActivityConfigCallback activityConfigCallback) {
        //这里最终会调父类Window的构造方法，并传入Activity的实例context
        this(context);
        //只有Activity的Window才会使用Decor context,其他依赖使用的Context
        mUseDecorContext = true;
        ...
   }

}
```

调用父类方法，让抽象父类内部持有Activity的实例化对象context

```java
//frameworks_base/core/java/android/view/Window.java
public abstract class Window {
     public Window(@UiContext Context context) {
        mContext = context;
        mFeatures = mLocalFeatures = getDefaultFeatures(context);
    }   
}
```

PhoneWindow的构造方法中传入当前Activity对象， 并最终在父类Window构造方法中给mContext赋值，**Window得到Activity的引用，由此就建立了Window和Activity的联系。**

#### 2.5.1.2setWindowManager

```java
//frameworks_base/core/java/android/view/Window.java
/**
     *用Window来设置窗口管理器，例如，显示面板。Window本身不能用于显示，必须通过客户端设置
     *window manager 用来添加新的Windows
     */
public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
                             boolean hardwareAccelerated) {
    //前三个参数是基本配置，包括AppToken，AppName和是否硬件加速
    mAppToken = appToken;
    mAppName = appName;
    mHardwareAccelerated = hardwareAccelerated;
    if (wm == null) {
        //[2.5.1.3]
        wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
    }
    //[2.5.1.4]
    mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
}
```

#### 2.5.1.3 getSystemService(Context.WINDOW_SERVICE)

这里就是获取系统注册的服务，主要是SystemServer，属于服务应用层的服务，其他包括AMS，PMS等

注意到SystemServiceRegistry的**static 方法**，这个方法内部注册了很多服务，我们可以在方法里面找到name = Context.WINDOW_SERVICE的服务

```java
//frameworks/base/core/java/android/app/SystemServiceRegistry.java
static {
    registerService(Context.WINDOW_SERVICE, WindowManager.class,
                    new CachedServiceFetcher<WindowManager>() {
                        @Override
                        public WindowManager createService(ContextImpl ctx) {
                            //最终会回调到这
                            return new WindowManagerImpl(ctx);
                        }});
}
```

registerService

```java
//frameworks/base/core/java/android/app/SystemServiceRegistry.java
private static final Map<Class<?>, String> SYSTEM_SERVICE_NAMES =
    new ArrayMap<Class<?>, String>();
private static final Map<String, ServiceFetcher<?>> SYSTEM_SERVICE_FETCHERS =
    new ArrayMap<String, ServiceFetcher<?>>();
private static final Map<String, String> SYSTEM_SERVICE_CLASS_NAMES = new ArrayMap<>();

private static <T> void registerService(@NonNull String serviceName,
                                        @NonNull Class<T> serviceClass, @NonNull ServiceFetcher<T> serviceFetcher) {
    //WindowManager.class和Context.WINDOW_SERVICE，组成kv的ArrayMap
    SYSTEM_SERVICE_NAMES.put(serviceClass, serviceName);
    //Context.WINDOW_SERVICE和Fetcher(CachedServiceFetcher)组成kv的ArrayMap
    SYSTEM_SERVICE_FETCHERS.put(serviceName, serviceFetcher);
    //Context.WINDOW_SERVICE和WindowManager，组成kv的ArrayMap
    SYSTEM_SERVICE_CLASS_NAMES.put(serviceName, serviceClass.getSimpleName());
}
```

这里的**mContext是ContextImpl类**的对象

```java
//frameworks/base/core/java/android/app/ContextImpl.java
@Override
public Object getSystemService(String name) {
    ...
    return SystemServiceRegistry.getSystemService(this, name);
}
```

进入SystemServiceRegistry的getSystemService方法，**name = Context.WINDOW_SERVICE**

```java
//frameworks/base/core/java/android/app/SystemServiceRegistry.java
public static Object getSystemService(ContextImpl ctx, String name) {
    ...
    //SYSTEM_SERVICE_FETCHERS是一个ArrayMap的集合
    final ServiceFetcher<?> fetcher = SYSTEM_SERVICE_FETCHERS.get(name);
    ...
    final Object ret = fetcher.getService(ctx);
    return ret;
}
```

先通过**Context.WINDOW_SERVICE**找到对应的fetcher(上述的CachedServiceFetcher)，然后从fetcher中找出其对应的服务

```java
static abstract class CachedServiceFetcher<T> implements ServiceFetcher<T> {
    @Override
    @SuppressWarnings("unchecked")
    public final T getService(ContextImpl ctx) {
        ...
        if (doInitialize) {
            T service = null;
            @ServiceInitializationState int newState = ContextImpl.STATE_NOT_FOUND;
            try {
                service = createService(ctx);
                newState = ContextImpl.STATE_READY;
            } catch (ServiceNotFoundException e) {
                ...
            }
            return ret;
        }
    }

    public abstract T createService(ContextImpl ctx) throws ServiceNotFoundException;
}
```

最终回到上述的回调，即对应服务为**WindowManagerImpl的实例**

#### 2.5.1.4 createLocalWindowManager

这里也是创建了一个WindowManagerImpl，和前面getSystemService创建的WindowManagerImpl区别：这里传入了parentWindow，让WindowManagerImpl持有了Window的引用，**这里持有的引用实际上是PhoneWindow**，可以对此Window进行操作了。

```java
//frameworks/base/core/java/android/view/WindowManagerImpl.java
public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
    return new WindowManagerImpl(mContext, parentWindow, mWindowContextToken);
}
```

#### 2.5.1.5总结

1. new一个PhoneWindow对象并传入当前Activity引用，建立Window和Activity的一一对应关系。此Window是Window类的子类PhoneWindow的实例。Activity在Window中是以mContext属性存在。
2. 调用PhoneWindow的setWindowManager方法，在这个方法中让Window和WindowManager建立了一一关系。此WindowManager是WindowManagerImpl的实例

**UML类图**

{{< image classes="fancybox center fig-100" src="/WMS/window/window49.png" thumbnail="/WMS/window/window49.png" title="">}}



**流程图**

{{< image classes="fancybox center fig-100" src="/WMS/window/window50.png" thumbnail="/WMS/window/window50.png" title="">}}

### 2.5.2Window和View关联

关于Window和View关联，详细可以阅读2.6.1关于activity的xml怎么跟屏幕关联起来的

这里主要梳理一下类图关系，App直接继承**Activity**和App继承**AppCompatActivity**流程类似，都是在DecorView内部操作的。

{{< image classes="fancybox center fig-100" src="/WMS/window/window51.png" thumbnail="/WMS/window/window51.png" title="">}}

> 简单概述：( **Window和View进行关联**)
>
> 1. 我们就给当前Activity的Window**创建了一个DecorView**
> 2. 这个DecorView就是当前Window的**rootView**
> 3. 并对rootView的ContentView设置了mContentParent 实现了将**Window和DecorView**绑定

```java
//frameworks/base/core/java/android/app/Activity.java
//1.设置主题
public void setTheme(int resid) {
    super.setTheme(resid);
    mWindow.setTheme(resid);
}
//2.设置Window属性等
public final boolean requestWindowFeature(int featureId) {
    return getWindow().requestFeature(featureId);
}

//3.获取Layout解析器
public LayoutInflater getLayoutInflater() {
    return getWindow().getLayoutInflater();
}
```

{{< image classes="fancybox center fig-100" src="/WMS/window/window52.png" thumbnail="/WMS/window/window52.png" title="">}}

可以发现Activity所有的对**View进行的操作**几乎都是通过**Window**进行的。而我们的Activity又是通过AMS来控制生命周期，所以AMS和View的通讯其实就是通过WindowManager这个介子进行的。



### 2.5.3View上屏、更新、删除

这里的代码比较多，比较还是以精简的方式来叙述，详细具体可以阅读参考文章

屏幕中所有的View首先需要经过**WindowManager**的处理，最后提交给**WMS**来处理。

WindowManager的父类为**ViewManager**（当然ViewManager也是GroupView的接口）

```java
//frameworks/base/core/java/android/view/ViewManager.java
public interface ViewManager{
    public void addView(View view, ViewGroup.LayoutParams params);
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    public void removeView(View view);
}
```

而这几个方法具体实现是在WindowManagerImpl类中

```java
//frameworks/base/core/java/android/view/WindowManagerImpl.java
public final class WindowManagerImpl implements WindowManager{
    private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
    @Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyTokens(params);
        mGlobal.addView(view, params, mContext.getDisplayNoVerify(), mParentWindow,
                mContext.getUserId());
    }

    @Override
    public void updateViewLayout(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyTokens(params);
        mGlobal.updateViewLayout(view, params);
    }
    @Override
    public void removeView(View view) {
        mGlobal.removeView(view, false);
    }
    ...
}
```



#### 2.5.3.1addView

根据上面的流程，这里是以WindowManager实现的函数调用为例

##### 2.5.3.1.1WindowManagerGlobal.addView

```java
//frameworks/base/core/java/android/view/WindowManagerGlobal.java
public final class WindowManagerGlobal {
    private final ArrayList<View> mViews = new ArrayList<View>();
    private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
    private final ArrayList<WindowManager.LayoutParams> mParams =
            new ArrayList<WindowManager.LayoutParams>();
    public void addView(View view, ViewGroup.LayoutParams params,
                        Display display, Window parentWindow, int userId) {
        ...
        //将params强转为WindowManager.LayoutParams类型
        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
        ...
        ViewRootImpl root;
        View panelParentView = null;

        synchronized (mLock) {
            ...
            IWindowSession windowlessSession = null;
            // 如果有一个父集，但是我们找不到它，它可能来自一个SurfaceControlViewHost层次结构
            if (wparams.token != null && panelParentView == null) {
                for (int i = 0; i < mWindowlessRoots.size(); i++) {
                    ViewRootImpl maybeParent = mWindowlessRoots.get(i);
                    if (maybeParent.getWindowToken() == wparams.token) {
                        windowlessSession = maybeParent.getWindowSession();
                        break;
                    }
                }
            }

            if (windowlessSession == null) {
                //创建一个ViewRootImpl对象
                root = new ViewRootImpl(view.getContext(), display);
            } else {
                root = new ViewRootImpl(view.getContext(), display,
                                        windowlessSession, new WindowlessWindowLayout());
            }

            view.setLayoutParams(wparams);
			//存储view到mViews列表,存储root到mRoots列表,存储wparams到mParams列表
            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);

            // do this last because it fires off messages to start doing things
            try {
                //这里的root是ViewRootImpl
                root.setView(view, wparams, panelParentView, userId);
            } catch (RuntimeException e) {
                final int viewIndex = (index >= 0) ? index : (mViews.size() - 1);
                // BadTokenException or InvalidDisplayException, clean up.
                if (viewIndex >= 0) {
                    removeViewLocked(viewIndex, true);
                }
                throw e;
            }
        }
    }   
}
```

- 将params强转为WindowManager.LayoutParams类型
- 创建一个ViewRootImpl对象root。
- 然后将当前view，当前params以及1中创建的ViewRootImpl对象root分别存储到对应的List列表中
- 调用root的setView方法

##### 2.5.3.1.2ViewRootImpl.setView

ViewRootImpl有很多作用

1. **管理View树，且其是View的根**
2. **触发三大绘制流程：测量，布局，绘制**
3. **输入事件中转站**
4. **管理Surface**
5. **负责与WMS通讯**

```java
//frameworks/base/core/java/android/view/ViewRootImpl.java
final IWindowSession mWindowSession;
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
                    int userId) {
    synchronized (this) {
        if (mView == null) {
            mView = view;
            // 在添加到窗口管理器之前安排第一个布局，以确保我们在从系统接收任何其他事件之前执行布局。
            requestLayout();
            ...
            try {
                res = mWindowSession.addToDisplayAsUser(mWindow, mWindowAttributes,
                                                        getHostVisibility(), mDisplay.getDisplayId(),
                                                        userId,mInsetsController.getRequestedVisibilities(), 
                                                        inputChannel, mTempInsets,mTempControls,
                                                        attachedFrame, compatScale);

        }
    }
}
```

这里主要是requestLayout方法和addToDisplayAsUser方法。

前者方法是屏幕绘制部分，这里不展开说明，主要为requestLayout内部主要使用垂直同步信号VSync的方式，在收到GPU提供的VSync信号后，会触发View的三大绘制**layout**，**mesure**，**draw**流程。

{{< image classes="fancybox center fig-100" src="/WMS/window/window54.png" thumbnail="/WMS/window/window54.png" title="">}}

##### 2.5.3.1.3mWindowSession.addToDisplayAsUser

后面是addToDisplayAsUser方法。mWindowSession是IWindowSession类型的

```java
//frameworks/base/core/java/android/view/ViewRootImpl.java
final IWindowSession mWindowSession;
public ViewRootImpl(Context context, Display display) {
    this(context, display, WindowManagerGlobal.getWindowSession(), new WindowLayout());
}

public ViewRootImpl(@UiContext Context context, Display display, IWindowSession session,
                    WindowLayout windowLayout) {
    mContext = context;
    mWindowSession = session;
    ...
}
```

可以看到mWindowSession实际上是是一个binder对象，用于进程间通讯，IWindowSession是C端代理，在S端使用的是Session类实现

```java
//frameworks/base/core/java/android/view/WindowManagerGlobal.java
public static IWindowSession getWindowSession() {
    synchronized (WindowManagerGlobal.class) {
        if (sWindowSession == null) {
            try {
                InputMethodManager.ensureDefaultInstanceForDefaultDisplayIfNecessary();
                //这里获取的是WindowManagerService的服务的代理类
                IWindowManager windowManager = getWindowManagerService();
                //真正获取到sWindowSession的对象代理类
                sWindowSession = windowManager.openSession(
                    new IWindowSessionCallback.Stub() {
                        @Override
                        public void onAnimatorScaleChanged(float scale) {
                            ValueAnimator.setDurationScale(scale);
                        }
                    });
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        }
        return sWindowSession;
    }
}

public static IWindowManager getWindowManagerService() {
    synchronized (WindowManagerGlobal.class) {
        if (sWindowManagerService == null) {
            sWindowManagerService = IWindowManager.Stub.asInterface(
                ServiceManager.getService("window"));
                ...
        }
        return sWindowManagerService;
    }
}
```

openSession

```java
//frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java
public IWindowSession openSession(IWindowSessionCallback callback) {
    return new Session(this, callback);
}
```

调用的Session的构造，并将回调监听也加入

```java
//frameworks/base/services/core/java/com/android/server/wm/Session.java
class Session extends IWindowSession.Stub implements IBinder.DeathRecipient {
    final WindowManagerService mService;

    public Session(WindowManagerService service, IWindowSessionCallback callback) {
        mService = service;
        mCallback = callback;
        ...
    }

    @Override
    public int addToDisplayAsUser(IWindow window, WindowManager.LayoutParams attrs,
            int viewVisibility, int displayId, int userId, InsetsVisibilities requestedVisibilities,
            InputChannel outInputChannel, InsetsState outInsetsState,
            InsetsSourceControl[] outActiveControls, Rect outAttachedFrame,
            float[] outSizeCompatScale) {
        //这里的mService就是构造注入的wms的this指针
        return mService.addWindow(this, window, attrs, viewVisibility, displayId, userId,
                requestedVisibilities, outInputChannel, outInsetsState, outActiveControls,
                outAttachedFrame, outSizeCompatScale);
    }
}
```

##### 2.5.3.1.4mService.addWindow

所以之前的Session类的addToDisplayAsUser方法，会调用addWindow方法，根据前面学习的Window容器，可以知道这里实际上Window容器添加到添加上去。（需要明白一点，先前的View绘制三部曲，**测量，布局，绘制**完成后，需要后续的图像的合成和显示）

{{< image classes="fancybox center fig-100" src="/WMS/window/window55.png" thumbnail="/WMS/window/window55.png" title="">}}

在WMS眼里，**一切View都是以Window形式**存在的， 剩下的工作就交由WMS进行处理了：在WMS中会为这个Window分配Surface，并确定显示层级， 可见负责显示界面的是画布Surface，而不是窗口本身，WMS将他管理的Surface交由SurfaceFlinger处理，SurfaceFlinger将这些Surface合并后放入到buffer中，屏幕会定时从buffer中获取显示数据，显示到屏幕上。

{{< image classes="fancybox center fig-100" src="/WMS/window/window56.png" thumbnail="/WMS/window/window56.png" title="">}}

查看源码addWindow，这里正好和上面的Window层级联系上了

```java
//frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java
final HashMap<IBinder, WindowState> mWindowMap = new HashMap<>();

public int addWindow(Session session, IWindow client, LayoutParams attrs, int viewVisibility,
                     int displayId, int requestUserId, InsetsVisibilities requestedVisibilities,
                     InputChannel outInputChannel, InsetsState outInsetsState,
                     InsetsSourceControl[] outActiveControls, Rect outAttachedFrame,
                     float[] outSizeCompatScale) {
   

    WindowState parentWindow = null;
    ...
    synchronized (mGlobalLock) {
        final DisplayContent displayContent = getDisplayContentOrCreate(displayId, attrs.token);

		//type类型获取
        if (type >= FIRST_SUB_WINDOW && type <= LAST_SUB_WINDOW) {
            parentWindow = windowForClientLocked(null, attrs.token, false);
            if (parentWindow == null) {
                return WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN;
            }
            if (parentWindow.mAttrs.type >= FIRST_SUB_WINDOW
                && parentWindow.mAttrs.type <= LAST_SUB_WINDOW) {
                return WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN;
            }
        }

        if (type == TYPE_PRIVATE_PRESENTATION && !displayContent.isPrivate()) {
            return WindowManagerGlobal.ADD_PERMISSION_DENIED;
        }

        if (type == TYPE_PRESENTATION && !displayContent.getDisplay().isPublicPresentation()) {
            return WindowManagerGlobal.ADD_INVALID_DISPLAY;
        }
		//token获取
        if (token == null) {
            if (hasParent) {
                // Use existing parent window token for child windows.
                token = parentWindow.mToken;
            } else if (mWindowContextListenerController.hasListener(windowContextToken)) {
                // Respect the window context token if the user provided it.
                final IBinder binder = attrs.token != null ? attrs.token : windowContextToken;
                final Bundle options = mWindowContextListenerController
                    .getOptions(windowContextToken);
                token = new WindowToken.Builder(this, binder, type)
                    .setDisplayContent(displayContent)
                    .setOwnerCanManageAppTokens(session.mCanAddInternalSystemWindow)
                    .setRoundedCornerOverlay(isRoundedCornerOverlay)
                    .setFromClientToken(true)
                    .setOptions(options)
                    .build();
            } else {
                final IBinder binder = attrs.token != null ? attrs.token : client.asBinder();
                token = new WindowToken.Builder(this, binder, type)
                    .setDisplayContent(displayContent)
                    .setOwnerCanManageAppTokens(session.mCanAddInternalSystemWindow)
                    .setRoundedCornerOverlay(isRoundedCornerOverlay)
                    .build();
            }
        } else if (rootType >= FIRST_APPLICATION_WINDOW
                   && rootType <= LAST_APPLICATION_WINDOW) {
            //token获取
        }
		//[2.5.3.1.5]
        final WindowState win = new WindowState(this, session, client, token, parentWindow,
                                                appOp[0], attrs, viewVisibility, session.mUid, userId,
                                                session.mCanAddInternalSystemWindow);
        // 这里筛选子窗口类型，因为子窗口必须附加到父窗口，子窗口的Token和父窗口一致
        //[2.5.3.1.6]
        win.mToken.addWindow(win);
        ...
    }

    Binder.restoreCallingIdentity(origId);

    return res;
}
```

流程比较复杂

1. 通过 WindowManagerPolicy 的 checkAddPermission 检查添加权限

2. 通过 displayId 获取所添加到的屏幕抽象 DisplayContent 对象

3. 根据 Window 是否是子窗口，创建 WindowToken

4. 如果上一步创建 token 失败，重新根据是否存在父窗口创建 WindowToken

5. 1. 子窗口直接取父窗口的 WindowToken
   2. 新的窗口直接构造一个新的 WindowToken

6. 根据根窗口的 type 来处理不同的情况（Toast、键盘、无障碍浮层等不同类型返回不同的结果）

7. **创建 WindowState**

8. 检查客户端状态，判断是否可以将 **Window 添加到系统中**

9. 以 session 为 key ，windowState 为 value ，存入到 mWindowMap 中

10. 将 WindowState 添加到相应的 WindowToken 中

    1. 若 WindowState 代表一个子窗口，直接 return
    2. 若仍没有 SurfaceControl ，为该 token 创建 Surface
    3. 更新 Layer
    4. 根据 Z 轴排序顺序将 WindowState 所代表的 Window 添加到合适的位置，此数据结构保存在 WindowToken 的父类 WindowContainer 的 mChildren 中：

11. 检查是否需要转换动画

12. 更新窗口焦点

13. 最后返回执行结果

最重要的就是[2.5.3.1.5]和[2.5.3.1.6]的操作，也是添加Window过程中Window容器之间的添加。

笔者这里之阐述跟Window容器相关的流程，关于**WMS中的Surface，还有Input流程**等需要到具体WMS章节阅读，可以点击[这里](https://juejin.cn/post/7172598258064162830)。

##### 2.5.3.1.5WindowState

```java
//frameworks/base/services/core/java/com/android/server/wm/WindowState.java
class WindowState extends WindowContainer<WindowState> implements WindowManagerPolicy.WindowState,
InsetsControlTarget, InputTarget {
    WindowState(WindowManagerService service, Session s, IWindow c, WindowToken token,
                WindowState parentWindow, int appOp, WindowManager.LayoutParams a, int viewVisibility,
                int ownerId, int showUserId, boolean ownerCanAddInternalSystemWindow,
                PowerManagerWrapper powerManagerWrapper) {
        //下面这些信息都会通过dumpsys window windows出现
        super(service);
        mSession = s;
        mClient = c;
        mAppOp = appOp;
        mToken = token;
        mActivityRecord = mToken.asActivityRecord();
        mWindowId = new WindowId(this);
        mViewVisibility = viewVisibility;
        mPolicy = mWmService.mPolicy;
        mContext = mWmService.mContext;
        ...
        //这一段在上述Window层级中出现，现在回过头再看就眼熟很多
        if (mAttrs.type >= FIRST_SUB_WINDOW && mAttrs.type <= LAST_SUB_WINDOW) {
            // The multiplier here is to reserve space for multiple
            // windows in the same type layer.
            mBaseLayer = mPolicy.getWindowLayerLw(parentWindow)
                * TYPE_LAYER_MULTIPLIER + TYPE_LAYER_OFFSET;
            mSubLayer = mPolicy.getSubWindowLayerFromTypeLw(a.type);
            mIsChildWindow = true;

            mLayoutAttached = mAttrs.type !=
                WindowManager.LayoutParams.TYPE_APPLICATION_ATTACHED_DIALOG;
            mIsImWindow = parentWindow.mAttrs.type == TYPE_INPUT_METHOD
                || parentWindow.mAttrs.type == TYPE_INPUT_METHOD_DIALOG;
            mIsWallpaper = parentWindow.mAttrs.type == TYPE_WALLPAPER;
        } else {
            // The multiplier here is to reserve space for multiple
            // windows in the same type layer.
            mBaseLayer = mPolicy.getWindowLayerLw(this)
                * TYPE_LAYER_MULTIPLIER + TYPE_LAYER_OFFSET;
            mSubLayer = 0;
            mIsChildWindow = false;
            mLayoutAttached = false;
            mIsImWindow = mAttrs.type == TYPE_INPUT_METHOD
                || mAttrs.type == TYPE_INPUT_METHOD_DIALOG;
            mIsWallpaper = mAttrs.type == TYPE_WALLPAPER;
        }
        ...
        // 确保在添加到parentwwindow之前初始化所有字段，以防止onDisplayChanged期间出现异常
        if (mIsChildWindow) {
            ProtoLog.v(WM_DEBUG_ADD_REMOVE, "Adding %s to %s", this, parentWindow);
            parentWindow.addChild(this, sWindowSubLayerComparator);
        }
    }
}
```

这里构造只会让WindowState和父WindowState联系起来，**并不会将WindowToken和WindowState关联起来**

##### 2.5.3.1.6win.mToken.addWindow

会将WindowToken和WindowState关联起来

```java
//frameworks/base/services/core/java/com/android/server/wm/WindowToken.java
void addWindow(final WindowState win) {
    if (win.isChildWindow()) {
        // 子窗口被添加到它们的父窗口
        return;
    }
    ...
    if (!mChildren.contains(win)) {
        addChild(win, mWindowComparator);
        mWmService.mWindowsChanged = true;
    }
}
```

最终会调用到父类WindowContainer的addChild方法

```java
//frameworks/base/services/core/java/com/android/server/wm/WindowContainer.java
protected final WindowList<E> mChildren = new WindowList<E>();
private WindowContainer<WindowContainer> mParent = null;
@CallSuper
protected void addChild(E child, Comparator<E> comparator) {
    int positionToAdd = -1;
    if (comparator != null) {
        final int count = mChildren.size();
        for (int i = 0; i < count; i++) {
            if (comparator.compare(child, mChildren.get(i)) < 0) {
                positionToAdd = i;
                break;
            }
        }
    }

    if (positionToAdd == -1) {
        mChildren.add(child);
    } else {
        mChildren.add(positionToAdd, child);
    }

    // Set the parent after we've actually added a child in case a subclass depends on this.
    child.setParent(this);
}

final protected void setParent(WindowContainer<WindowContainer> parent) {
    final WindowContainer oldParent = mParent;
    mParent = parent;

    if (mParent != null) {
        mParent.onChildAdded(this);
    } else if (mSurfaceAnimator.hasLeash()) {
        mSurfaceAnimator.cancelAnimation();
    }
    if (!mReparenting) {
        onSyncReparent(oldParent, mParent);
        if (mParent != null && mParent.mDisplayContent != null
            && mDisplayContent != mParent.mDisplayContent) {
            onDisplayChanged(mParent.mDisplayContent);
        }
        onParentChanged(mParent, oldParent);
    }
}
```

上述添加规则mWindowComparator，实际上就是上面层级关系2.3.2.3的WindowState添加规则

{{< image classes="fancybox center fig-100" src="/WMS/window/window57.png" thumbnail="/WMS/window/window57.png" title="">}}



##### 2.5.3.1.7流程图

{{< image classes="fancybox center fig-100" src="/WMS/window/window72.png" thumbnail="/WMS/window/window72.png" title="">}}

补充UML图

{{< image classes="fancybox center fig-100" src="/WMS/window/window73.png" thumbnail="/WMS/window/window73.png" title="">}}

#### 2.5.3.2updateViewLayout

和Window 添加类似，主要简单介绍为主，感兴趣的读者可以自行查看源码。它的实现也是在 WindowManagerImpl 中，同样的，实际调用的是 WindowManagerGlobal 对应的方法 updateViewLayout

```java
//frameworks/base/core/java/android/view/WindowManagerGlobal.java
public void updateViewLayout(View view, ViewGroup.LayoutParams params) {
    if (view == null) {
        throw new IllegalArgumentException("view must not be null");
    }
    if (!(params instanceof WindowManager.LayoutParams)) {
        throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
    }

    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;

    view.setLayoutParams(wparams);

    synchronized (mLock) {
        int index = findViewLocked(view, true);
        ViewRootImpl root = mRoots.get(index);
        mParams.remove(index);
        mParams.add(index, wparams);
        root.setLayoutParams(wparams, false);
    }
}
```

这里的root直接调用对应的setLayoutParams方法，其中后一个参数为false，表示无需重绘

```java
//frameworks/base/core/java/android/view/ViewRootImpl.java
public void setLayoutParams(WindowManager.LayoutParams attrs, boolean newView) {
    synchronized (this) {
        final int oldInsetLeft = mWindowAttributes.surfaceInsets.left;
        final int oldInsetTop = mWindowAttributes.surfaceInsets.top;
        final int oldInsetRight = mWindowAttributes.surfaceInsets.right;
        final int oldInsetBottom = mWindowAttributes.surfaceInsets.bottom;
        final int oldSoftInputMode = mWindowAttributes.softInputMode;
        final boolean oldHasManualSurfaceInsets = mWindowAttributes.hasManualSurfaceInsets;
        // Keep track of the actual window flags supplied by the client.
        mClientWindowLayoutFlags = attrs.flags;

        // Preserve compatible window flag if exists.
        final int compatibleWindowFlag = mWindowAttributes.privateFlags
            & WindowManager.LayoutParams.PRIVATE_FLAG_COMPATIBLE_WINDOW;

        // Preserve system UI visibility.
        final int systemUiVisibility = mWindowAttributes.systemUiVisibility;
        final int subtreeSystemUiVisibility = mWindowAttributes.subtreeSystemUiVisibility;

        // Preserve appearance and behavior.
        final int appearance = mWindowAttributes.insetsFlags.appearance;
        final int behavior = mWindowAttributes.insetsFlags.behavior;
        final int appearanceAndBehaviorPrivateFlags = mWindowAttributes.privateFlags
            & (PRIVATE_FLAG_APPEARANCE_CONTROLLED | PRIVATE_FLAG_BEHAVIOR_CONTROLLED);

        final int changes = mWindowAttributes.copyFrom(attrs);
        if ((changes & WindowManager.LayoutParams.TRANSLUCENT_FLAGS_CHANGED) != 0) {
            // Recompute system ui visibility.
            mAttachInfo.mRecomputeGlobalAttributes = true;
        }
        if ((changes & WindowManager.LayoutParams.LAYOUT_CHANGED) != 0) {
            // Request to update light center.
            mAttachInfo.mNeedsUpdateLightCenter = true;
        }
        if (mWindowAttributes.packageName == null) {
            mWindowAttributes.packageName = mBasePackageName;
        }

        // Restore preserved flags.
        mWindowAttributes.systemUiVisibility = systemUiVisibility;
        mWindowAttributes.subtreeSystemUiVisibility = subtreeSystemUiVisibility;
        mWindowAttributes.insetsFlags.appearance = appearance;
        mWindowAttributes.insetsFlags.behavior = behavior;
        mWindowAttributes.privateFlags |= compatibleWindowFlag
            | appearanceAndBehaviorPrivateFlags
            | WindowManager.LayoutParams.PRIVATE_FLAG_USE_BLAST;

        if (mWindowAttributes.preservePreviousSurfaceInsets) {
            // Restore old surface insets.
            mWindowAttributes.surfaceInsets.set(
                oldInsetLeft, oldInsetTop, oldInsetRight, oldInsetBottom);
            mWindowAttributes.hasManualSurfaceInsets = oldHasManualSurfaceInsets;
        } else if (mWindowAttributes.surfaceInsets.left != oldInsetLeft
                   || mWindowAttributes.surfaceInsets.top != oldInsetTop
                   || mWindowAttributes.surfaceInsets.right != oldInsetRight
                   || mWindowAttributes.surfaceInsets.bottom != oldInsetBottom) {
            mNeedsRendererSetup = true;
        }

        applyKeepScreenOnFlag(mWindowAttributes);
		//这里无需重绘
        if (newView) {
            mSoftInputMode = attrs.softInputMode;
            requestLayout();
        }

        // Don't lose the mode we last auto-computed.
        if ((attrs.softInputMode & SOFT_INPUT_MASK_ADJUST)
            == WindowManager.LayoutParams.SOFT_INPUT_ADJUST_UNSPECIFIED) {
            mWindowAttributes.softInputMode = (mWindowAttributes.softInputMode
                                               & ~SOFT_INPUT_MASK_ADJUST) | (oldSoftInputMode & SOFT_INPUT_MASK_ADJUST);
        }
		//如果softInputMode有调整，会触发这里的scheduleTraversals
        if (mWindowAttributes.softInputMode != oldSoftInputMode) {
            requestFitSystemWindows();
        }

        mWindowAttributesChanged = true;
        scheduleTraversals();
    }
}
```

这里的scheduleTraversals在上述addView也有，这里不做展开，申请下一帧进行绘制。

{{< image classes="fancybox center fig-100" src="/WMS/window/window60.png" thumbnail="/WMS/window/window60.png" title="">}}

#### 2.5.3.3removeView

Window 的删除过程通过 WindowManager 的 removeView 方法进行的

```java
//frameworks/base/core/java/android/view/WindowManagerImpl.java
@Override
public void removeView(View view) {
    mGlobal.removeView(view, false);
}

@Override
public void removeViewImmediate(View view) {
    mGlobal.removeView(view, true);
}
```

上面在WindowManagerImpl将remove操作划分，通过是否立刻来判断是否是同步，false为异步，true为同步。通常Activity走的是异步，Dialog会走同步。

实际逻辑在 WindowManagerGlobal 的实现中。和Window 添加类似，主要简单介绍为主，感兴趣的读者可以自行查看源码。

```java
//frameworks/base/core/java/android/view/WindowManagerGlobal.java
public void removeView(View view, boolean immediate) {
    if (view == null) {
        throw new IllegalArgumentException("view must not be null");
    }

    synchronized (mLock) {
        //[2.5.3.3.1]
        int index = findViewLocked(view, true);
        //[2.5.3.3.2]
        View curView = mRoots.get(index).getView();
        //[2.5.3.3.3]
        removeViewLocked(index, immediate);
        if (curView == view) {
            return;
        }

        throw new IllegalStateException("Calling with view " + view
                                        + " but the ViewAncestor is attached to " + curView);
    }
}
```

##### 2.5.3.3.1findViewLocked

获取view在mViews中的索引

```java
//frameworks/base/core/java/android/view/WindowManagerGlobal.java
private int findViewLocked(View view, boolean required) {
    final int index = mViews.indexOf(view);
    if (required && index < 0) {
        throw new IllegalArgumentException("View=" + view + " not attached to window manager");
    }
    return index;
}
```

##### 2.5.3.3.2mRoots.get(index).getView()

从 mRoots 中根据索引找出 ViewRootImpl ，通过 ViewRootImpl 找到它的 View。通常我们访问的View可能是一个子View，然后直接添加的View可能是一个View树，这个时候如果删除那么就是删除这个子View对应的之前setView添加的View树。当然可能这个现在的View就是之前的setView进来的View

```java
//frameworks/base/core/java/android/view/ViewRootImpl.java
View mView;
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
                    int userId) {
    synchronized (this) {
        if (mView == null) {
            mView = view;
        }
    }
    ...
}

public View getView() {
    return mView;
}
```

>  说明一下addView传入的View和removeView传入的View
>
> {{< image classes="fancybox center fig-100" src="/WMS/window/window59.png" thumbnail="/WMS/window/window59.png" title="">}}

##### 2.5.3.3.3removeViewLocked

```java
//frameworks/base/core/java/android/view/WindowManagerGlobal.java
private void removeViewLocked(int index, boolean immediate) {
    ViewRootImpl root = mRoots.get(index);
    View view = root.getView();

    if (root != null) {
        //调用去清除输入焦点，结束处理输入法相关的逻辑
        root.getImeFocusController().onWindowDismissed();
    }
    boolean deferred = root.die(immediate);
    if (view != null) {
        view.assignParent(null);
        if (deferred) {
            mDyingViews.add(view);
        }
    }
}
```

在这个方法中，首先也是获取 ViewRootImpl 和 View 对象，然后调用 ViewRootImpl 对象的 die 方法，其中传入的immediate是根据外部传入的值来判断，如果为true同步移除才执行后续的dodie方法。

```java
//frameworks/base/core/java/android/view/ViewRootImpl.java
boolean die(boolean immediate) {
    // 如果是同步移除，则立即调用doDie方法
    if (immediate && !mIsInTraversal) {
        doDie();
        return false;
    }

    if (!mIsDrawing) {
        destroyHardwareRenderer();
    } else {
        Log.e(mTag, "Attempting to destroy the window while drawing!\n" +
              "  window=" + this + ", title=" + mWindowAttributes.getTitle());
    }
    //异步移除，发出一个msg进行移除，在主线程移除
    mHandler.sendEmptyMessage(MSG_DIE);
    return true;
}
```

##### 2.5.3.3.4doDie

```java
//frameworks/base/core/java/android/view/ViewRootImpl.java
void doDie() {
    ...
    synchronized (this) {
        //mAdded必须在setView之后，且还没有doDie之前，不然再次调用doDie，mAdded也为false
        if (mAdded) {
            //[2.5.3.3.5]	
            dispatchDetachedFromWindow();
        }
        ..
    }
    //[2.5.3.3.6]
    WindowManagerGlobal.getInstance().doRemoveView(this);
}
```

##### 2.5.3.3.5dispatchDetachedFromWindow

```java
//frameworks/base/core/java/android/view/ViewRootImpl.java
void dispatchDetachedFromWindow() {
    //这里可以看出我们可以在View销毁前，在View视图树分发的dispatchDetachedFromWindow做一些资源释放操作
    if (mView != null && mView.mAttachInfo != null) {
        mAttachInfo.mTreeObserver.dispatchOnWindowAttachedChange(false);
        mView.dispatchDetachedFromWindow();
    }
	//硬件渲染的销毁释放
    destroyHardwareRenderer();
	//将mView的root置为null
    mView.assignParent(null);
    mView = null;
    mAttachInfo.mRootView = null;
	//释放当前Surface
    destroySurface();

    try {
        //[2.5.3.3.7]通过Session IPC调用了WMS移除Window,这里移除的实际上是Window容器
        mWindowSession.remove(mWindow);
    } catch (RemoteException e) {
    }
    // 销毁对应的input接受者
    if (mInputEventReceiver != null) {
        mInputEventReceiver.dispose();
        mInputEventReceiver = null;
    }
	//删除待执行的 Traversals
    unregisterListeners();
    unscheduleTraversals();
}
```

dispatchDetachedFromWindow 方法中

1. 开始向视图树中的 View 分发 DetachedFromWindow ，
2. 然后释放一些资源，释放 Surface 
3. 通过 Session IPC 调用了 WMS 移除 Window
4. 删除待执行的 Traversals

这里的unscheduleTraversals和上面的scheduleTraversals恰好相反，这里是移除同步屏障，使得上述scheduleTraversals**还未执行的消息队列失效**。

##### 2.5.3.3.6doRemoveView

这里主要是对WindowManagerGlobal保存的数据（mRoots、mParams、mViews）进行移除

```java
//frameworks/base/core/java/android/view/WindowManagerGlobal.java
void doRemoveView(ViewRootImpl root) {
    boolean allViewsRemoved;
    synchronized (mLock) {
        final int index = mRoots.indexOf(root);
        if (index >= 0) {
            mRoots.remove(index);
            mParams.remove(index);
            final View view = mViews.remove(index);
            mDyingViews.remove(view);
        }
        allViewsRemoved = mRoots.isEmpty();
    }
    if (ThreadedRenderer.sTrimForeground) {
        doTrimForeground();
    }

    if (allViewsRemoved) {
        InsetsAnimationThread.release();
    }
}
```

##### 2.5.3.3.7remove

根据addView分析，mWindowSession是WMS进程中的Session类

```java
//frameworks/base/services/core/java/com/android/server/wm/Session.java
public void remove(IWindow window) {
    mService.removeWindow(this, window);
}
```

##### 2.5.3.3.8removeWindow

```java
//frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java
void removeWindow(Session session, IWindow client) {
    synchronized (mGlobalLock) {
        //[2.5.3.3.9]通过windowid找到对应的WindowState容器
        WindowState win = windowForClientLocked(session, client, false);
        if (win != null) {
            //[2.5.3.3.10]移除相关的WindowState容器
            win.removeIfPossible();
            return;
        }

        //如果令牌属于EmbeddedWindow，则删除EmbeddedWindow映射
        mEmbeddedWindowController.remove(client);
    }
}
```

##### 2.5.3.3.9windowForClientLocked

```java
//frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java
final HashMap<IBinder, WindowState> mWindowMap = new HashMap<>();
public int addWindow(Session session, IWindow client, LayoutParams attrs, int viewVisibility,
                     int displayId, int requestUserId, InsetsVisibilities requestedVisibilities,
                     InputChannel outInputChannel, InsetsState outInsetsState,
                     InsetsSourceControl[] outActiveControls, Rect outAttachedFrame,
                     float[] outSizeCompatScale) {
    final WindowState win = new WindowState(this, session, client, token, parentWindow,
                    appOp[0], attrs, viewVisibility, session.mUid, userId,
                    session.mCanAddInternalSystemWindow);
    //addWindow中传入windowstate
    mWindowMap.put(client.asBinder(), win);
    win.mToken.addWindow(win);
}

final WindowState windowForClientLocked(Session session, IBinder client, boolean throwOnError) {
    //从hashmap中去除WindowState
    WindowState win = mWindowMap.get(client);
    //校验WindowState的合法性
    if (win == null) {
        if (throwOnError) {
            throw new IllegalArgumentException(
                "Requested window " + client + " does not exist");
        }
        ProtoLog.w(WM_ERROR, "Failed looking up window session=%s callers=%s", session,
                   Debug.getCallers(3));
        return null;
    }
    if (session != null && win.mSession != session) {
        if (throwOnError) {
            throw new IllegalArgumentException("Requested window " + client + " is in session "
                                               + win.mSession + ", not " + session);
        }
        ProtoLog.w(WM_ERROR, "Failed looking up window session=%s callers=%s", session,
                   Debug.getCallers(3));
        return null;
    }

    return win;
}
```

##### 2.5.3.3.10removeIfPossible

获取到WindowState容器，开始移除

```java
//frameworks/base/services/core/java/com/android/server/wm/WindowState.java
@Override
void removeIfPossible() {
    //如果当前的WindowState存在mChildren，那么也需要对mChildren中的Window容器移除
    super.removeIfPossible();
    removeIfPossible(false /*keepVisibleDeadWindow*/);
}

private void removeIfPossible(boolean keepVisibleDeadWindow) {
    final DisplayContent displayContent = getDisplayContent();
    final long origId = Binder.clearCallingIdentity();
    try {
        //[2.5.3.3.11]条件判断过滤，满足条件会直接 return 延迟删除操作
        removeImmediately();
        ...
        //删除一个可见窗口将影响计算方向，如果需要，更新方向
        if (needToSendNewConfiguration) {
            displayContent.sendNewConfiguration();
        }
        //更新 WMS 的 Window 焦点
        mWmService.updateFocusedWindowLocked(isFocused()
                                             ? UPDATE_FOCUS_REMOVING_FOCUS
                                             : UPDATE_FOCUS_NORMAL,
                                             true /*updateInputWindows*/);
        ...
    } finally {
        Binder.restoreCallingIdentity(origId);
    }
}
```

##### 2.5.3.3.11removeImmediately

```java
//frameworks/base/services/core/java/com/android/server/wm/WindowState.java
@Override
void removeImmediately() {
    //确保同时只能处理一个
    if (mRemoved) {
        // Nothing to do.
        ProtoLog.v(WM_DEBUG_ADD_REMOVE,
                   "WS.removeImmediately: %s Already removed...", this);
        return;
    }

    mRemoved = true;
    super.removeImmediately();
    final DisplayContent dc = getDisplayContent();
    final int type = mAttrs.type;
    //判断windowstate类型，只支持五种
    //STATUS_BAR
    //NOTIFICATION_SHADE
    //NAVIGATION_BAR
    //INPUT_METHOD_DIALOG
    //VOLUME_OVERLAY
    if (WindowManagerService.excludeWindowTypeFromTapOutTask(type)) {
        dc.mTapExcludedWindows.remove(this);
    }

    // Remove this window from mTapExcludeProvidingWindows. If it was not registered, this will
    // not do anything.
    dc.mTapExcludeProvidingWindows.remove(this);
    //这里移除对应的STATUS_BAR、NAVIGATION_BAR、CLIMATE_BAR、EXTRA_NAVIGATION_BAR之类的Window
    dc.getDisplayPolicy().removeWindowLw(this);
	//处理 Session 相关的逻辑，从 mSessions 中删除了当前 mSession ，并清除 mSession 对应的 SurfaceSession 资源
    //SurfaceSession 是 SurfaceFlinger 的一个连接，通过这个连接可以创建多个 Surface 并渲染到屏幕上
    mSession.windowRemovedLocked();
    try {
        //解除与客户端的连接
        mClient.asBinder().unlinkToDeath(mDeathRecipient, 0);
    } catch (RuntimeException e) {

    }
	//执行一些集中清理的工作
    mWmService.postWindowRemoveCleanupLocked(this);
}
```

{{< image classes="fancybox center fig-100" src="/WMS/window/window61.png" thumbnail="/WMS/window/window61.png" title="">}}

### 2.5.4与View相关总结

#### 2.5.4.1activity与 PhoneWindow与DecorView关系  

DecorView是FrameLayout的子类， 它可以被认为是Android视图树的**根节点视图**  

#### 2.5.4.2WindowManagerGlobal  作用

设计模式中的**桥接模式**，在WindowManagerImpl和WMS之间起到了桥梁作用

{{< image classes="fancybox center fig-100" src="/WMS/window/window62.png" thumbnail="/WMS/window/window62.png" title="">}}

#### 2.5.4.3ViewRootImpl  作用

1. View树的树根并管理View树
2. 触发View的测量、 布局和绘制
3. 输入响应的中转站
4. 负责与WMS进行进程间通信  

{{< image classes="fancybox center fig-100" src="/WMS/window/window63.png" thumbnail="/WMS/window/window63.png" title="">}}

#### 2.5.4.4View/ViewGroup结构

View/ViewGroup主要是树型结构

{{< image classes="fancybox center fig-100" src="/WMS/window/window64.png" thumbnail="/WMS/window/window64.png" title="">}}

#### 2.5.4.5绘制流程

{{< image classes="fancybox center fig-100" src="/WMS/window/window65.png" thumbnail="/WMS/window/window65.png" title="">}}

#### 2.5.4.6输入事件流程 

{{< image classes="fancybox center fig-100" src="/WMS/window/window66.png" thumbnail="/WMS/window/window66.png" title="">}}

#### 2.5.4.7WMS通信

{{< image classes="fancybox center fig-100" src="/WMS/window/window67.png" thumbnail="/WMS/window/window67.png" title="">}}

#### 2.5.4.8屏幕刷新流程 

ViewRootImpl会调用scheduleTraversals准备重绘， 但是，重绘一般不会立即执行， 而是往Choreographer的Choreographer.CALLBACK_TRAVERSAL队列中添加了一个mTraversalRunnable，同时申请VSYNC， 这个mTraversalRunnable要一直等到申请的VSYNC到来后才会被执行  

{{< image classes="fancybox center fig-100" src="/WMS/window/window25.png" thumbnail="/WMS/window/window25.png" title="">}}

{{< image classes="fancybox center fig-100" src="/WMS/window/window53.png" thumbnail="/WMS/window/window53.png" title="">}}

## 2.6补充Window相关问题

### 2.6.1关于activity的xml怎么跟屏幕关联起来的

#### 2.6.1.1performLaunchActivity

这里从应用启动开始，加载ActivityThread调用**performLaunchActivity**方法中的onCreate，直接调用的应用的onCreate方法

```java
//frameworks/base/core/java/android/app/ActivityThread.java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ...
    if (r.isPersistable()) {
        mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
    } else {
        mInstrumentation.callActivityOnCreate(activity, r.state);
    }
    ...
}
```

#### 2.6.1.2App#onCreate

应用的代码如下

```kotlin
//App,这里用到的setContentView是AppCompatActivity中的
class MainActivity : AppCompatActivity() {
    private lateinit var binding: ActivityMainBinding
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
                // 使用了 ViewBinding
        binding = ActivityMainBinding.inflate(layoutInflater)
                // ① 加载 activity 的布局
        setContentView(binding.root)
    }
}
```

#### 2.6.1.3AppCompatActivity#setContentView

```java
//androidx.appcompat:appcompat:1.3.12aar
//androidx/appcompat/app/AppComPatActivity.java
@Override
public void setContentView(View view) {
    //initViewTreeOwners方法作用
    //在设置内容视图之前设置视图树所有者，以便膨胀过程和附加侦听器将看到它们已经存在
    initViewTreeOwners();
    getDelegate().setContentView(view);
}
```

getDelegate

```java
//androidx.appcompat:appcompat:1.3.12aar
//androidx/appcompat/app/AppComPatActivity.java
public AppCompatDelegate getDelegate() {
    if (mDelegate == null) {
        //传入的两个参数都是AppComPatActivity
        mDelegate = AppCompatDelegate.create(this, this);
    }
    return mDelegate;
}
```

AppCompatActivity 将很多事都委托给了别人，就是 AppCompatDelegateImpl

```java
//androidx.appcompat:appcompat:1.3.12aar
//androidx/appcompat/app/AppCompatDelegate.java
public static AppCompatDelegate create(@NonNull Activity activity,
                                       @Nullable AppCompatCallback callback) {
    return new AppCompatDelegateImpl(activity, callback);
}
```

#### 2.6.1.4AppCompatDelegateImpl#setContentView

```java
//androidx.appcompat:appcompat:1.3.12aar
//androidx/appcompat/app/AppCompatDelegateImpl.java
@Override
public void setContentView(View v) {
    ensureSubDecor();
    ViewGroup contentParent = mSubDecor.findViewById(android.R.id.content);
    contentParent.removeAllViews();
    //这里的addView跟动态添加View类似，都是父布局中添加布局，这的父布局是contentParent，子布局是v
    contentParent.addView(v);
    ...
}
```

#### 2.6.1.5AppCompatDelegateImpl#ensureSubDecor

创建一个 SubDecor，然后找到里面 id 为 content 的控件，将从 xml 创建出来的 View 添加到 content 里面去。

我们看看这个 SubDecor 是啥，创建 SubDecor 的逻辑也不多：

```java
//androidx.appcompat:appcompat:1.3.12aar
//androidx/appcompat/app/AppCompatDelegateImpl.java
private void ensureSubDecor() {
    if (!mSubDecorInstalled) {
        mSubDecor = createSubDecor();
        ...
    }
}

private ViewGroup createSubDecor() {
    ...
    //下面会根据不同的情况去加载不同的布局，即应用布局并不是固定不变的   
    final LayoutInflater inflater = LayoutInflater.from(mContext);
    ViewGroup subDecor = null;
    if (!mWindowNoTitle) {
        if (mIsFloating) {
            // If we're floating, inflate the dialog title decor
            subDecor = (ViewGroup) inflater.inflate(
                R.layout.abc_dialog_title_material, null);
        } else if (mHasActionBar) {
            subDecor = (ViewGroup) LayoutInflater.from(themedContext)
                .inflate(R.layout.abc_screen_toolbar, null);

            mDecorContentParent = (DecorContentParent) subDecor
                .findViewById(R.id.decor_content_parent);
            mDecorContentParent.setWindowCallback(getWindowCallback());
            ...
        }
    } else {
        if (mOverlayActionMode) {
            subDecor = (ViewGroup) inflater.inflate(
                R.layout.abc_screen_simple_overlay_action_mode, null);
        } else {
            //本文是这里
            subDecor = (ViewGroup) inflater.inflate(R.layout.abc_screen_simple, null);
        }
    }

    return subDecor;
}
```

首先它会根据设置的 theme 来加载不同的 layout，比如这里调试的发现它使用的是 R.layout.abc_screen_simple，搜索一下，发现它的内容如下：

```xml
<!--androidx.appcompat:appcompat:1.3.12aar -->
<!--/res/layout/abc_screen_simple.xml -->
<androidx.appcompat.widget.FitWindowsLinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/action_bar_root"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:fitsSystemWindows="true">

    <androidx.appcompat.widget.ViewStubCompat
        android:id="@+id/action_mode_bar_stub"
        android:inflatedId="@+id/action_mode_bar"
        android:layout="@layout/abc_action_mode_bar"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />

    <include layout="@layout/abc_screen_content_include" />

</androidx.appcompat.widget.FitWindowsLinearLayout>
```

include 里面就是一个 androidx.appcompat.widget.ContentFrameLayout 没有其他布局。SubDecor 的整个布局如下

{{< image classes="fancybox center fig-100" src="/WMS/window/window9.png" thumbnail="/WMS/window/window9.png" title="">}}

#### 2.6.1.6AppCompatDelegateImpl#createSubDecor

上面已经提到了subDecor的初步创建

```java
//androidx.appcompat:appcompat:1.3.12aar
//androidx/appcompat/app/AppCompatDelegateImpl.java
private ViewGroup createSubDecor() {
    subDecor = (ViewGroup) inflater.inflate(R.layout.abc_screen_simple, null);
    ...
    //1.找到xml中的名称为action_bar_activity_content，类型为ContentFrameLayout的布局
    final ContentFrameLayout contentView = (ContentFrameLayout) subDecor.findViewById(
        R.id.action_bar_activity_content);
    //2.这个mWindow布局来自框架screen_simple.xml，找到对应的布局android.R.id.content
    final ViewGroup windowContentView = (ViewGroup) mWindow.findViewById(android.R.id.content);
    if (windowContentView != null) {
        //3.这里是Window窗口的添加，具体见上文2.4
        while (windowContentView.getChildCount() > 0) {
            final View child = windowContentView.getChildAt(0);
            windowContentView.removeViewAt(0);
            contentView.addView(child);
        }
        //4.将布局windowContentView对应的的id从android.R.id.content变成View.NO_ID
        windowContentView.setId(View.NO_ID);
        //5.将对应布局contentView中的id从R.id.action_bar_activity_content变成android.R.id.content
        contentView.setId(android.R.id.content);
    }
    //把子布局放到父布局中
    mWindow.setContentView(subDecor);
    return subDecor;
}
```

其中abc_screen_simple.xml存在abc_screen_content_include布局，这个布局中有action_bar_activity_content布局

```xml
<!--androidx.appcompat:appcompat:1.3.12aar -->
<!--/res/layout/abc_screen_content_include.xml -->
<merge xmlns:android="http://schemas.android.com/apk/res/android">

    <androidx.appcompat.widget.ContentFrameLayout
            android:id="@id/action_bar_activity_content"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:foregroundGravity="fill_horizontal|top"
            android:foreground="?android:attr/windowContentOverlay" />

</merge>
```

其中，上述的mWindow实际上PhoneWindow，mWindow.findViewById

```java
//frameworks/base/core/java/android/view/Window.java
//可以得知这个mWindow布局是依托在DecorView中的
@Nullable
public <T extends View> T findViewById(@IdRes int id) {
    return getDecorView().findViewById(id);
}
```

getDecorView生成布局

```java
//frameworks/base/core/java/com/android/internal/policy/PhoneWindow.java
protected ViewGroup generateLayout(DecorView decor) {
    //这里桶前文createSubDecor类似，也是根据不同的情况生成对应的布局
    int layoutResource;
    int features = getLocalFeatures();
    ...
    if ((features & (1 << FEATURE_NO_TITLE)) == 0) {
        // If no other features and not embedded, only need a title.
        // If the window is floating, we need a dialog layout
        if (mIsFloating) {
            TypedValue res = new TypedValue();
            getContext().getTheme().resolveAttribute(
                R.attr.dialogTitleDecorLayout, res, true);
            layoutResource = res.resourceId;
        } else if ((features & (1 << FEATURE_ACTION_BAR)) != 0) {
            layoutResource = a.getResourceId(
                R.styleable.Window_windowActionBarFullscreenDecorLayout,
                R.layout.screen_action_bar);
        } else {
            layoutResource = R.layout.screen_title;
        }
    } else if ((features & (1 << FEATURE_ACTION_MODE_OVERLAY)) != 0) {
        layoutResource = R.layout.screen_simple_overlay_action_mode;
    } else {
        //本文的布局在这里
        layoutResource = R.layout.screen_simple;
    }
    return contentParent;
}
```

即getDecorView对应的布局是screen_simple.xml

```xml
<!--/frameworks/base/core/res/res/layout/screen_simple.xml -->
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    android:orientation="vertical">
    <ViewStub android:id="@+id/action_mode_bar_stub"
              android:inflatedId="@+id/action_mode_bar"
              android:layout="@layout/action_mode_bar"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:theme="?attr/actionBarTheme" />
    <FrameLayout
         android:id="@android:id/content"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
         android:foregroundInsidePadding="false"
         android:foregroundGravity="fill_horizontal|top"
         android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>
```

> 这里我们发现，subDecor 被添加到了 mContentParent 里面，mContentParent 是啥呢？它是DecorView里面的 id 叫 Content 的一个FrameLayout 控件。但是 subDecor 里面不也有一个 id 叫 Content 的控件吗？
>
> 这里解释一下，subDecor-content 在xml中的ID是 action_bar_activity_content，经过一番操作后，它的 id 会被代码替换成 R.id.content，而 decorView-content 控件的 id 被设置为了 NO_ID。由于 mContentParent  已经保存了 decorView-content 控件的引用，所以设置成 NO_ID 也没关系。
>
> {{< image classes="fancybox center fig-100" src="/WMS/window/window10.png" thumbnail="/WMS/window/window10.png" title="">}}

#### 2.6.1.7完整的界面的布局如下

原理都是一样，只是使用AppCompatActivity继承的布局会多2层嵌套

应用程序继承**Activity**

{{< image classes="fancybox center fig-100" src="/WMS/window/window11.png" thumbnail="/WMS/window/window11.png" title="">}}

应用程序继承**AppCompatActivity**

{{< image classes="fancybox center fig-100" src="/WMS/window/window4.png" thumbnail="/WMS/window/window4.png" title="">}}

上屏流程如下

{{< image classes="fancybox center fig-100" src="/WMS/window/window13.png" thumbnail="/WMS/window/window13.png" title="">}}

### 2.6.2关于锁屏之后为什么看不到Launcher

锁屏理论上也只是一个window盖在最顶层，理论即使这个锁屏window透明，那么透看到的也是有桌面Activity的情况，但现实情况是我们只看到了壁纸，并没有看到桌面。

首先我们知道壁纸属于系统中一个特殊窗口，一直处于系统最底层。这个窗口在系统中有专门类进行他的显示情况

```java
//frameworks/services/core/java/com/android/server/wm/WallpaperController.java
if (DEBUG_WALLPAPER) {
    Slog.v(TAG, "Wallpaper visibility: " + visible + " at display "
            + mDisplayContent.getDisplayId());
}
```

这里我们把DEBUG_WALLPAPER就可以把log打开了

```shell
WindowManager: Win Window{60c77f1 u0 ScreenDecorOverlayBottom}: isOnScreen=true mDrawState=4
WindowManager: Win Window{df0a78b u0 ScreenDecorOverlay}: isOnScreen=true mDrawState=4
WindowManager: Win Window{b7a35a u0 NavigationBar0}: isOnScreen=true mDrawState=4
WindowManager: Win Window{3cd76e8 u0 NotificationShade}: isOnScreen=true mDrawState=4
WindowManager: Found wallpaper target: Window{3cd76e8 u0 NotificationShade}
WindowManager: New wallpaper target: Window{3cd76e8 u0 NotificationShade} prevTarget: null
WindowManager: Report new wp offset Window{803c2b8 u0 com.android.systemui.ImageWallpaper} x=0.0 y=0.5 zoom=0.0
WindowManager: Wallpaper visibility: true at display 0
WindowManager: Wallpaper token android.os.Binder@3a6eda2 visible=true
WindowManager: New wallpaper: target=Window{3cd76e8 u0 NotificationShade} prev=null
WindowManager: powerPress: eventTime=118823847 interactive=true count=0 beganFromNonInteractive=true mShortPressOnPowerBehavior=1
```

打开后我们点亮锁屏和在桌面锁屏发现都有类似以下打印

WindowManager: Found wallpaper target: Window{3cd76e8 u0 NotificationShade}

看名字大家大概就知道这个log是在寻找一个wallpaper target:，而且找到的是NotificationShade即锁屏窗口

前面疑惑中就写到正常应该是桌面

WindowManager: Found wallpaper target: Window{8d79f3 u0 com.android.launcher3/com.android.launcher3.uioverrides.QuickstepLauncher}

这日志其实就可以给我们非常重要线索，可以找出对应代码在哪

```java
//frameworks/services/core/java/com/android/server/wm/WallpaperController.java
final boolean hasWallpaper = w.hasWallpaper() || animationWallpaper;
boolean hasWallpaper() {
    return (mAttrs.flags & FLAG_SHOW_WALLPAPER) != 0
        || (mActivityRecord != null && mActivityRecord.hasWallpaperBackgroudForLetterbox());
}
https://blog.csdn.net/learnframework/article/details/127483545
private final ToBooleanFunction<WindowState> mFindWallpaperTargetFunction = w -> {
    ...
    else if (hasWallpaper && w.isOnScreen()
    ...
};
```

这里其实也就是判断window中是否有FLAG_SHOW_WALLPAPER属性，而且经过额外加log打印发现hasWallpaper值变化在锁屏window解锁前后

那么其实我们可以猜测是不是锁屏window会去动态改变自己的FLAG_SHOW_WALLPAPER属性，在有桌面显示时候锁屏的window实际是没有这个属性，在锁屏状态下是有这个属性。根据猜想去systemui代码中grep相关FLAG_SHOW_WALLPAPER关键字

```java
//frameworks/packages/SystemUI/src/com/android/systemui/shade/NotificationShadeWindowControllerImpl.java
private void applyKeyguardFlags(State state) {
    ...
    if ((keyguardOrAod && !state.mBackdropShowing && !scrimsOccludingWallpaper)
        || mKeyguardViewMediator.isAnimatingBetweenKeyguardAndSurfaceBehindOrWillBe()) {
        mLpChanged.flags |= LayoutParams.FLAG_SHOW_WALLPAPER;
    } else {
        mLpChanged.flags &= ~LayoutParams.FLAG_SHOW_WALLPAPER;
    }
}
```

这里其实就是代表有锁屏时候需要有 mLpChanged.flags |= LayoutParams.FLAG_SHOW_WALLPAPER;

锁屏退出时清除锁屏的FLAG_SHOW_WALLPAPER

mLpChanged.flags &= ~LayoutParams.FLAG_SHOW_WALLPAPER;

### 2.6.3Activity究竟对应几个Window

上面提到一个Activity对应一个PhoneWindow，那么为什么图还存在一个Activity可以出现两个Window，似乎有所矛盾

提到这个问题，我们必须要先复现当前问题

| 名称        | 说明                   |
| ----------- | ---------------------- |
| 手机型号    | 红米K30pro             |
| Android版本 | Android10              |
| 测试场景    | 屏幕点亮的Launcher界面 |
| 后台应用    | Wechat                 |

如下图所示

{{< image classes="fancybox center fig-100" src="/WMS/window/window0.jpg" thumbnail="/WMS/window/window0.jpg" title="">}}

```shell
# 这里输出简化表示
picasso:/ $ dumpsys window windows
WINDOW MANAGER WINDOWS (dumpsys window windows)
Window #0 Window{54fd848 u0 InputMethod}:
  mBaseLayer=141000 mSubLayer=0    mToken=WindowToken{226fcc5 android.os.Binder@3f373c}
  isVisible=false
Window #1 Window{6474d57 u0 RoundCorner}:
  mBaseLayer=311000 mSubLayer=0    mToken=WindowToken{7706d6 android.os.BinderProxy@359caf1}
  isVisible=true
Window #2 Window{98435fe u0 RoundCorner}:
  mBaseLayer=311000 mSubLayer=0    mToken=WindowToken{a6ec0b2 android.os.BinderProxy@1af12bd}
  isVisible=true
Window #3 Window{987738d u0 NavigationBar}:
  mBaseLayer=231000 mSubLayer=0    mToken=WindowToken{cdfac24 android.os.BinderProxy@1bf9cb7}
  isVisible=true
Window #4 Window{85d4239 u0 StatusBar}:
  mBaseLayer=181000 mSubLayer=0    mToken=WindowToken{e3e1a00 android.os.BinderProxy@bf7a332}
  isVisible=true
Window #5 Window{5c8904f u0 RoundCorner}:
  mBaseLayer=171000 mSubLayer=0    mToken=WindowToken{c281aae android.os.BinderProxy@485e729}
  isVisible=false
Window #6 Window{e12503c u0 Aspect}:
  mBaseLayer=111000 mSubLayer=0    mToken=WindowToken{cb4622f android.os.BinderProxy@bba880e}
  isVisible=false
Window #7 Window{31fbd71 u0 AssistPreviewPanel}:
  mBaseLayer=41000 mSubLayer=0    mToken=WindowToken{f8d1f18 android.os.BinderProxy@c09ec8a}
  isVisible=false
Window #8 Window{67faf21 u0 DockedStackDivider}:
  mBaseLayer=21000 mSubLayer=0    mToken=WindowToken{f740988 android.os.BinderProxy@d47412b}
  isVisible=false
Window #9 Window{19af82e u0 LauncherOverlayWindow:com.miui.personalassistant}:
  mBaseLayer=21000 mSubLayer=0    mToken=AppWindowToken{e921607 token=Token{361c446 ActivityRecord{6d5b688 u0 com.miui.home/.launcher.Launcher t1}}}
  isVisible=false
Window #10 Window{6163c5a u0 com.miui.home/com.miui.home.launcher.Launcher}:
  mBaseLayer=21000 mSubLayer=0    mToken=AppWindowToken{e921607 token=Token{361c446 ActivityRecord{6d5b688 u0 com.miui.home/.launcher.Launcher t1}}}
  isVisible=true
Window #11 Window{d7d87d0 u0 com.tencent.mm/com.tencent.mm.ui.LauncherUI}:
  mBaseLayer=21000 mSubLayer=0    mToken=AppWindowToken{e299b34 token=Token{4ee9e07 ActivityRecord{ebb6c46 u0 com.tencent.mm/.ui.LauncherUI t87}}}
  isVisible=false
Window #12 Window{f63516 u0 com.android.keyguard.wallpaper.service.MiuiKeyguardLiveWallpaper}:
  mBaseLayer=11000 mSubLayer=0    mToken=WallpaperWindowToken{f661030 token=android.os.BinderProxy@3481973}
  isVisible=true
Window #13 Window{cf10cf2 u0 com.android.systemui.ImageWallpaper}:
  mBaseLayer=11000 mSubLayer=0    mToken=WallpaperWindowToken{5182c91 token=android.os.Binder@2b356b8}
  isVisible=true
```

可以看到有14个Window，但不是所有Window都显示的，只有7个Window才显示出来的，其中可以看到有9，10两个Window对应同一个WindowToken，并且mBaseLayer和mSubLayer都相同，根据上面的Window层级关系，实际上就是一个Activity会存在两个Window，似乎跟上文的一个一个Activity对应一个PhoneWindow矛盾了。

实际上不然，9和10两个Window这个Activity相同，WindowToken相同，这个类似悬浮窗，虽然是同一个Activity，前者9属于一个最小化的图标，后者10才是真正意义上的一个Activity对应一个PhoneWindow，这是一种**Overlay机制**。并且9通常是隐藏的，10才是可以显示出来的。Window可以两者存在，并且可以同时显示。

最终显示结果从上往下

{{< image classes="fancybox center fig-100" src="/WMS/window/window7.png" thumbnail="/WMS/window/window7.png" title="">}}

### 2.6.4Activity、Dialog、PopupWindow和Toast 的Window创建

#### 2.6.4.1 Activity的Window创建

Activity在ActivityThread中调用performLaunchActivity方法中的Activity#attach

##### 2.6.4.1.1Activity#attach 

通过代码可以知道在 attach 方法中，创建了 Window

```java
//frameworks/base/core/java/android/app/Activity.java
private Window mWindow;
final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken,
            IBinder shareableActivityToken) {
	//创建 PhoneWindow
    mWindow = new PhoneWindow(this, window, activityConfigCallback);
	...
}
```

##### 2.6.4.1.2Activity#setContentView 

Activity 的视图是通过 setContentView 方法提供的，如果是AppCompatActivity类中的setContentView ，请参考2.6.1的流程，这里阐述直接的Activity 类中的setContentView

```java
//frameworks/base/core/java/android/app/Activity.java
public void setContentView(@LayoutRes int layoutResID) {
    //这里的getWindow对应PhoneWindow
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
```

##### 2.6.4.1.3PhoneWindow#setContentView

```java
//frameworks/base/core/java/com/android/internal/policy/PhoneWindow.java
@Override
public void setContentView(int layoutResID) {
    if (mContentParent == null) {
        // 1，创建 DecorView
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                                                       getContext());
        transitionTo(newScene);
    } else {
        //2 添加 activity 的布局文件
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    mContentParent.requestApplyInsets();
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        //通知 window 的callback 接口回调
        cb.onContentChanged();
    }
    mContentParentExplicitlySet = true;
}
```

如果没有 DecorView 就创建它，一般来说它内部包含了标题栏和内容栏，但是这个会随着主题的改变而发生改变。但是不管怎么样，内容栏是一定存在的，并且内容栏有固定的 id `content`，完整的 id 是 `android.R.id.content` 。

1. 通过 `generateDecor` 创建了 DecorView，接着会调用 `generateLayout` 来加载具体的布局文件到 DecorView 中，这个要加载的布局就和系统版本以及定义的主题有关了。加载完之后就会将内容区域的 View 返回出来，也就是 `mContentParent`
2. 将 activity 需要显示的布局添加到 `mcontentParent` 中
3. 由于 activity 实现了 window 的callback 接口，这里表示 activity 的布局文件已经被添加到 decorView 的 mParentView 中了，于是通知 接口回调

##### 2.6.4.1.4 handlerResumeActivity

ActivityThread 的 handlerResumeActivity 中，会调用 activity 的 onResume 方法，接着就会将 DecorView 添加到 Window 中

```java
@Override
public void handleResumeActivity(ActivityClientRecord r, boolean finalStateRequest,
                                 boolean isForward, boolean shouldSendCompatFakeFocus, String reason) {
    if (!performResumeActivity(r, finalStateRequest, reason)) {
        return;
    }

    final Activity a = r.activity;
    if (r.window == null && !a.mFinished && willBeVisible) {
        r.window = r.activity.getWindow();
        View decor = r.window.getDecorView();
        decor.setVisibility(View.INVISIBLE);
        ViewManager wm = a.getWindowManager();
        WindowManager.LayoutParams l = r.window.getAttributes();
        a.mDecor = decor;
        //这里可以看到type是TYPE_BASE_APPLICATION
        l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
        l.softInputMode |= forwardBit;
        
        if (a.mVisibleFromClient) {
            if (!a.mWindowAdded) {
                a.mWindowAdded = true;
                //这里就是addView流程了，不在展开描述
                wm.addView(decor, l);
            } 
        }
    }
}
```

可以看到type类型是TYPE_BASE_APPLICATION，有上文的层级关系流程，在WindowManagerPolicy类中查找对应的主次层级的序号

```java
//frameworks/base/services/core/java/com/android/server/policy/WindowManagerPolicy.java
default int getWindowLayerFromTypeLw(int type, boolean canAddInternalSystemWindow,
                                     boolean roundedCornerOverlay) {
	...
    //TYPE_BASE_APPLICATION值为1，正好在[1,99]范围内，返回APPLICATION_LAYER，值为2
    if (type >= FIRST_APPLICATION_WINDOW && type <= LAST_APPLICATION_WINDOW) {
        return APPLICATION_LAYER;
    }
}
```

即Activity所在层级为App层级，主层级为2

#### 2.6.4.2Dialog的Window创建

##### 2.6.4.2.1Dialog::Dialog

Dialog 中创建 `Window` 是在其构造方法中完成

```java
//frameworks/base/core/java/android/app/Dialog.java
private final WindowManager mWindowManager;
final Context mContext;
final Window mWindow;

public Dialog(@UiContext @NonNull Context context, @StyleRes int themeResId) {
    this(context, themeResId, true);
}

Dialog(@UiContext @NonNull Context context, @StyleRes int themeResId,
       boolean createContextThemeWrapper) {
	//获取 WindowManager
    mWindowManager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
	//创建 Window
    final Window w = new PhoneWindow(mContext);
    mWindow = w;
    ...
}
```

##### 2.6.4.2.2Dialog#setContentView

初始化 DecorView，将 dialog 的视图添加到 DecorView 中。这个和 activity 的类似，都是通过 Window 去添加指定的布局文件

```java
//frameworks/base/core/java/android/app/Dialog.java
public void setContentView(@LayoutRes int layoutResID) {
    mWindow.setContentView(layoutResID);
}
```

##### 2.6.4.2.3Dialog#show

将 DecorView 添加到 Window 中显示

```java
//frameworks/base/core/java/android/app/Dialog.java
public void show() {
	//...
    mDecor = mWindow.getDecorView();
    WindowManager.LayoutParams l = mWindow.getAttributes();
    mWindowManager.addView(mDecor, l);
	//发送回调消息
    sendShowMessage();
}
```

##### 2.6.4.2.4mWindow.getAttributes

这里没有显示type类型，追溯mWindow.getAttributes()

```java
//frameworks/base/core/java/android/view/Window.java
private final WindowManager.LayoutParams mWindowAttributes =
        new WindowManager.LayoutParams();
```

##### 2.6.4.2.5WindowManager.LayoutParams构造

这里可以看出默认如果没有设置type类型，默认的类型是TYPE_APPLICATION，跟上面Activity类似，都属于主层级为2

```java
//frameworks/base/core/java/android/view/WindowManager.java
public LayoutParams() {
    super(LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT);
    type = TYPE_APPLICATION;
    format = PixelFormat.OPAQUE;
}
```

从上面几个步骤可以发现，Dialog 的 Window 创建和 Activity 的 Window 创建很类似，二者几乎没有什么区别。

##### 2.6.4.2.6Dialog#dismissDialog

当 dialog 关闭时，它会通过 WindowManager来移除 DecorView

```java
//frameworks/base/core/java/android/app/Dialog.java
@UnsupportedAppUsage
void dismissDialog() {
	...
    try {
        //移除操作
        mWindowManager.removeViewImmediate(mDecor);
    } finally {
        if (mActionMode != null) {
            mActionMode.finish();
        }
        mDecor = null;
        mWindow.closeAllPanels();
        onStop();
        mShowing = false;

        sendDismissMessage();
    }
}
```

普通的 Dialog 有一个特殊的地方，就是必须采用 Activity 的 Context，如果采用 Application 的 Context，就会报错。

错误信息很明确，是没有 Token 导致的，而 Token 一般只有 Activity 拥有，所以这里只需要用 Activity 作为 Context 即可。

```txt
Caused by: android.view.WindowManager$BadTokenException: Unable to add window -- token null is not valid; is your activity running?
```

##### 2.6.4.2.7window#setType

系统 Window 比较特殊，他可以不需要 Token，我们可以将 Dialog 的 Window Type 修改为系统类型就可以了

可以调用Dialog实例化对象的window.setType()。Type类型可以是系统的`WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY`（对应主层级11）或者是`WindowManager.LayoutParams.TYPE_SYSTEM_ALERT`（对应主层级9）

额外需要申请需要申请悬浮窗权限即可

```java
//frameworks/base/core/java/android/view/Window.java
public void setType(int type) {
    final WindowManager.LayoutParams attrs = getAttributes();
    attrs.type = type;
    dispatchWindowAttributesChanged(attrs);
}
```



#### 2.6.4.3PopupWindow的Window创建

##### 2.6.4.3.1PopupWindow#PopupWindow

```java
//frameworks/base/core/java/android/widget/PopupWindow.java
public PopupWindow(Context context, AttributeSet attrs) {
    this(context, attrs, com.android.internal.R.attr.popupWindowStyle);
}

public PopupWindow(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
    mContext = context;
    mWindowManager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
    ...
}
```

##### 2.6.4.3.2PopupWindow#setContentView

可以看到，和刚才看到的构造函数基本相同，保存了`ContentView`变量后，获取`Context`和`WindowManger`对象。

```java
//frameworks/base/core/java/android/widget/PopupWindow.java
public void setContentView(View contentView) {
    mContentView = contentView;
    if (mContext == null && mContentView != null) {
        mContext = mContentView.getContext();
    }

    if (mWindowManager == null && mContentView != null) {
        mWindowManager = (WindowManager) mContext.getSystemService(Context.WINDOW_SERVICE);
    }

    if (mContext != null && !mAttachedInDecorSet) {

        setAttachedInDecor(mContext.getApplicationInfo().targetSdkVersion
                           >= Build.VERSION_CODES.LOLLIPOP_MR1);
    }

}
```

##### 2.6.4.3.3PopupWindow#showAsDropDown

```java
//frameworks/base/core/java/android/widget/PopupWindow.java
public void showAsDropDown(View anchor) {
    showAsDropDown(anchor, 0, 0);
}

public void showAsDropDown(View anchor, int xoff, int yoff, int gravity) {
    TransitionManager.endTransitions(mDecorView);
	//绑定监听，设置变量
    attachToAnchor(anchor, xoff, yoff, gravity);

    mIsShowing = true;
    mIsDropdown = true;
	//创建LayoutParams
    final WindowManager.LayoutParams p =
        createPopupLayoutParams(anchor.getApplicationWindowToken());
    //准备Popupwindow
    preparePopup(p);

    final boolean aboveAnchor = findDropDownPosition(anchor, p, xoff, yoff,
                                                     p.width, p.height, gravity, mAllowScrollingAnchorParent);
    updateAboveAnchor(aboveAnchor);
    p.accessibilityIdOfAnchor = (anchor != null) ? anchor.getAccessibilityViewId() : -1;
	//添加布局到Window中
    invokePopup(p);
}
```

##### 2.6.4.3.4PopupWindow#createPopupLayoutParams

可以看到创建默认的LayoutParams时候类型为TYPE_APPLICATION_PANEL，说明PopupWindow是一个子Window，主层级关系就是2，次层级关系是1，也就是说会覆盖在App层级之上

```java
//frameworks/base/core/java/android/widget/PopupWindow.java
private int mWindowLayoutType = WindowManager.LayoutParams.TYPE_APPLICATION_PANEL;
@UnsupportedAppUsage
protected final WindowManager.LayoutParams createPopupLayoutParams(IBinder token) {
    final WindowManager.LayoutParams p = new WindowManager.LayoutParams();

    p.gravity = computeGravity();
    p.flags = computeFlags(p.flags);
    p.type = mWindowLayoutType;
    p.token = token;
    p.softInputMode = mSoftInputMode;
    p.windowAnimations = computeAnimationResource();
    
    p.privateFlags = PRIVATE_FLAG_WILL_NOT_REPLACE_ON_RELAUNCH
        | PRIVATE_FLAG_LAYOUT_CHILD_WINDOW_IN_PARENT_FRAME;
    return p;
}
```

##### 2.6.4.3.5PopupWindow#preparePopup

```java
//frameworks/base/core/java/android/widget/PopupWindow.java
@UnsupportedAppUsage
private void preparePopup(WindowManager.LayoutParams p) {
    //设置Background包裹
    if (mBackground != null) {
        mBackgroundView = createBackgroundView(mContentView);
        mBackgroundView.setBackground(mBackground);
    } else {
        mBackgroundView = mContentView;
    }
	//这里创建根布局，包裹mBackground
    mDecorView = createDecorView(mBackgroundView);
    mDecorView.setIsRootNamespace(true);
    ...
}
```

##### 2.6.4.3.6PopupWindow#createBackgroundView

显然它的布局就是PopupBackgroundView

```java
//frameworks/base/core/java/android/widget/PopupWindow.java
private PopupBackgroundView createBackgroundView(View contentView) {
    final ViewGroup.LayoutParams layoutParams = mContentView.getLayoutParams();
    final int height;
    if (layoutParams != null && layoutParams.height == WRAP_CONTENT) {
        height = WRAP_CONTENT;
    } else {
        height = MATCH_PARENT;
    }

    final PopupBackgroundView backgroundView = new PopupBackgroundView(mContext);
    final PopupBackgroundView.LayoutParams listParams = new PopupBackgroundView.LayoutParams(
        MATCH_PARENT, height);
    backgroundView.addView(contentView, listParams);

    return backgroundView;
}
```



##### 2.6.4.3.7PopupWindow#createDecorView

显然它的布局就是PopupDecorView

```java
//frameworks/base/core/java/android/widget/PopupWindow.java
private PopupDecorView createDecorView(View contentView) {
    final ViewGroup.LayoutParams layoutParams = mContentView.getLayoutParams();
    final int height;
    
    if (layoutParams != null && layoutParams.height == WRAP_CONTENT) {
        height = WRAP_CONTENT;
    } else {
        height = MATCH_PARENT;
    }

    final PopupDecorView decorView = new PopupDecorView(mContext);
    decorView.addView(contentView, MATCH_PARENT, height);
    decorView.setClipChildren(false);
    decorView.setClipToPadding(false);

    return decorView;
}
```

{{< image classes="fancybox center fig-100" src="/WMS/window/window69.png" thumbnail="/WMS/window/window69.png" title="">}}

##### 2.6.4.3.8PopupWindow#invokePopup

最终添加window，可以看出根布局是PopupDecorView，里面包裹着PopupBackgroundView，mContentView就是传入的View

```java
//frameworks/base/core/java/android/widget/PopupWindow.java
@UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.P)
private void invokePopup(WindowManager.LayoutParams p) {
    if (mContext != null) {
        p.packageName = mContext.getPackageName();
    }

    final PopupDecorView decorView = mDecorView;
    decorView.setFitsSystemWindows(mLayoutInsetDecor);

    setLayoutDirectionFromAnchor();
	//addView
    mWindowManager.addView(decorView, p);

    if (mEnterTransition != null) {
        decorView.requestEnterTransition(mEnterTransition);
    }
}
```



PopupWindow它是一个子Window，需要设置ContentView

利用自定义View包裹我们的ContentView，自定义View重写了键盘事件和触摸事件分发，实现了点击外部消失

最终利用WindowManger的addView加入布局，并没有创建新的Window

#### 2.6.4.4Toast的Window创建

Toast 也是基于 Window 来实现的，但是他的工作过程有些复杂。Toast有进程间的通信，Toast对应服务为NotificationManagerService和ITransientNotification。

Toast 属于系统 Window，内部视图有两种定义方式，一种是系统默认的，另一种是通过 setView 方法来指定一个 View(setView 方法在 android 11 以后已经废弃了，不会再展示自定义视图)，他们都对应 Toast 的一个内部成员 `mNextView`。

##### 2.6.4.4.3Toast#makeText

```java
//frameworks/base/core/java/android/widget/Toast.java
public static Toast makeText(@NonNull Context context, @Nullable Looper looper,
                             @NonNull CharSequence text, @Duration int duration) {
    //这里主要是构造[2.6.4.4.2]
    Toast result = new Toast(context, looper);

    if (Compatibility.isChangeEnabled(CHANGE_TEXT_TOASTS_IN_THE_SYSTEM)) {
        result.mText = text;
    } else {
        //然后将mNextView赋值为getTextToastView[2.6.4.4.6]
        result.mNextView = ToastPresenter.getTextToastView(context, text);
    }

    result.mDuration = duration;
    return result;
}
```



##### 2.6.4.4.2Toast#Toast

在Toast构造中，可以发现内部维护了一个mTN的类属性，里面还有回调的可扩容数组

```java
//frameworks/base/core/java/android/widget/Toast.java
final TN mTN;
public Toast(Context context) {
    this(context, null);
}

public Toast(@NonNull Context context, @Nullable Looper looper) {
    //context是外部App端调用传入的值
    mContext = context;
    mToken = new Binder();
    looper = getLooper(looper);
    mHandler = new Handler(looper);
    //回调的可扩容数组
    mCallbacks = new ArrayList<>();
    mTN = new TN(context, context.getPackageName(), mToken,
                 mCallbacks, looper);
   ...
}
```

##### 2.6.4.4.3TN#TN

这里的ToastPresenter实例化对象很关键，是后续Toast内部维护View的桥梁，真正加载布局的地方

```java
//frameworks/base/core/java/android/widget/Toast.java
//这个就是上述服务类的实现，实现在Toast类的内部
private static class TN extends ITransientNotification.Stub {
    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.P)
    private final WindowManager.LayoutParams mParams;
    private final ToastPresenter mPresenter;
    TN(Context context, String packageName, Binder token, List<Callback> callbacks,
       @Nullable Looper looper) {
        IAccessibilityManager accessibilityManager = IAccessibilityManager.Stub.asInterface(
            ServiceManager.getService(Context.ACCESSIBILITY_SERVICE));
        //初始化ToastPresenter
        mPresenter = new ToastPresenter(context, accessibilityManager, getService(),
                                        packageName);
        mParams = mPresenter.getLayoutParams();
        mPackageName = packageName;
        mToken = token;
        mCallbacks = callbacks;
        ...
    }
}
```

##### 2.6.4.4.4ToastPresenter#ToastPresenter

```java
//frameworks_base/core/java/android/widget/ToastPresenter.java
public ToastPresenter(Context context, IAccessibilityManager accessibilityManager,
                      INotificationManager notificationManager, String packageName) {
    mContext = context;
    mResources = context.getResources();
    mWindowManager = context.getSystemService(WindowManager.class);
    mNotificationManager = notificationManager;
    mPackageName = packageName;
    mAccessibilityManager = accessibilityManager;
    //创建默认的LayoutParams
    mParams = createLayoutParams();
}
```

##### 2.6.4.4.5ToastPresenter#createLayoutParams

找到Toast的window type类型为TYPE_TOAST，对应层级关系主层级为7。结果上面2.6.4.2的Dialog，如果没有设置Type那么Toast会在Dialog上面，如果设置系统Type，那么Dialog会覆盖Toast。

```java
//frameworks_base/core/java/android/widget/ToastPresenter.java
private WindowManager.LayoutParams createLayoutParams() {
    WindowManager.LayoutParams params = new WindowManager.LayoutParams();
    params.height = WindowManager.LayoutParams.WRAP_CONTENT;
    params.width = WindowManager.LayoutParams.WRAP_CONTENT;
    params.format = PixelFormat.TRANSLUCENT;
    params.windowAnimations = R.style.Animation_Toast;
    //这里的type类型是TYPE_TOAST
    params.type = WindowManager.LayoutParams.TYPE_TOAST;
    params.setFitInsetsIgnoringVisibility(true);
    params.setTitle(WINDOW_TITLE);
    params.flags = WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON
        | WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
        | WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE;
    setShowForAllUsersIfApplicable(params, mPackageName);
    return params;
}
```

##### 2.6.4.4.6ToastPresenter#getTextToastView

这里可以发现Toast的布局，实际上是在ToastPresenter中添加对应布局，根布局为R.layout.transient_notification，我们所在的text就是内部的com.android.internal.R.id.message所在位置。

```java
//frameworks_base/core/java/android/widget/ToastPresenter.java
@VisibleForTesting
public static final int TEXT_TOAST_LAYOUT = R.layout.transient_notification;
public static View getTextToastView(Context context, CharSequence text) {
    View view = LayoutInflater.from(context).inflate(TEXT_TOAST_LAYOUT, null);
    TextView textView = view.findViewById(com.android.internal.R.id.message);
    textView.setText(text);
    return view;
}
```

其中xml为

```xml
<!-- //frameworks/base/core/res/res/layout/transient_notification.xml-->
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:orientation="horizontal"
    android:gravity="center_vertical"
    android:maxWidth="@dimen/toast_width"
    android:background="?android:attr/toastFrameBackground"
    android:elevation="@dimen/toast_elevation"
    android:layout_marginEnd="16dp"
    android:layout_marginStart="16dp"
    android:paddingStart="16dp"
    android:paddingEnd="16dp">

    <TextView
        android:id="@android:id/message"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:ellipsize="end"
        android:maxLines="2"
        android:paddingTop="12dp"
        android:paddingBottom="12dp"
        android:textAppearance="@style/TextAppearance.Toast"/>
</LinearLayout>
```

##### 2.6.4.4.7Toast#show

```java
//frameworks/base/core/java/android/widget/Toast.java
public void show() {
    INotificationManager service = getService();
    String pkg = mContext.getOpPackageName();
    TN tn = mTN;
    tn.mNextView = mNextView;
    final int displayId = mContext.getDisplayId();

    try {
        if (Compatibility.isChangeEnabled(CHANGE_TEXT_TOASTS_IN_THE_SYSTEM)) {
            if (mNextView != null) {
                //mNextView文字存在
                service.enqueueToast(pkg, mToken, tn, mDuration, displayId);
            } else {
               ...
            }
        } 
    } catch (RemoteException e) {
        // Empty
    }
}
```

##### 2.6.4.4.8NotificationManagerService#enqueueTextToast

上面代码中对给定应用的 toast 数量进行判断，如果超过 50 条，就直接退出，这是为了防止 DOS ，如果某个应用一直循环弹出 toast 就会导致其他应用无法弹出，这显然是不合理的。

```java
//frameworks_base/services/core/java/com/android/server/notification/NotificationManagerService.java
@Override
public void enqueueToast(String pkg, IBinder token, ITransientNotification callback,
                         int duration, int displayId) {
    enqueueToast(pkg, token, null, callback, duration, displayId, null);
}
//传入上面的callback，类型为ITransientNotificationCallback,指的是mTN
private void enqueueToast(String pkg, IBinder token, @Nullable CharSequence text,
                          @Nullable ITransientNotification callback, int duration, int displayId,
                          @Nullable ITransientNotificationCallback textCallback) {

    synchronized (mToastQueue) {
        try {
            //创建对应的 ToastRecord
            ToastRecord record;
            int index = indexOfToastLocked(pkg, token);
            // If it's already in the queue, we update it in place, we don't
            // move it to the end of the queue.
            if (index >= 0) {
                //实际上就是获取队列中的record元素
                record = mToastQueue.get(index);
                record.update(duration);
            } else {
                ...
                    //如果超过 50 条，就直接退出    
                    if (count >= MAX_PACKAGE_TOASTS) {
                        Slog.e(TAG, "Package has already queued " + count
                               + " toasts. Not showing more. Package=" + pkg);
                        return;
                    }
                //创建对应的 ToastRecord
                record = getToastRecord(callingUid, callingPid, pkg, isSystemToast, token,text, callback, duration, windowToken, displayId, textCallback);
            }
            // 0 表示只有一个 toast了，直接显示，否则就是还有toast，排队等待显示
            if (index == 0) {
                showNextToastLocked(false);
            }
        } finally {
            Binder.restoreCallingIdentity(callingId);
        }
    }
}
```

##### 2.6.4.4.9NotificationManagerService#getToastRecord

说明这里的ToastRecord有两种类型，分别为TextToastRecord和CustomToastRecord，我们这里存在callback，所以为CustomToastRecord

```java
//frameworks_base/services/core/java/com/android/server/notification/NotificationManagerService.java
private ToastRecord getToastRecord(int uid, int pid, String packageName, boolean isSystemToast,
                                   IBinder token, @Nullable CharSequence text, @Nullable ITransientNotification callback,
                                   int duration, Binder windowToken, int displayId,
                                   @Nullable ITransientNotificationCallback textCallback) {
    if (callback == null) {
        return new TextToastRecord(this, mStatusBar, uid, pid, packageName,
                                   isSystemToast, token, text, duration, windowToken, displayId, textCallback);
    } else {
        return new CustomToastRecord(this, uid, pid, packageName,
                                     isSystemToast, token, callback, duration, windowToken, displayId);
    }
}
```

##### 2.6.4.4.10NotificationManagerService#showNextToastLocked

展示下一个Toast

```java
//frameworks_base/services/core/java/com/android/server/notification/NotificationManagerService.java
@GuardedBy("mToastQueue")
void showNextToastLocked(boolean lastToastWasTextRecord) {
    //获取队头元素
    ToastRecord record = mToastQueue.get(0);
    while (record != null) {
        //存在队头元素，处理元素
        if (tryShowToast(
            record, rateLimitingEnabled, isWithinQuota, isPackageInForeground)) {
            scheduleDurationReachedLocked(record, lastToastWasTextRecord);
            mIsCurrentToastShown = true;
            if (rateLimitingEnabled && !isPackageInForeground) {
                mToastRateLimiter.noteEvent(userId, record.pkg, TOAST_QUOTA_TAG);
            }
            return;
        }
    }
}

private boolean tryShowToast(ToastRecord record, boolean rateLimitingEnabled,
                             boolean isWithinQuota, boolean isPackageInForeground) {
    return record.show();
}
```

##### 2.6.4.4.11CustomToastRecord#show

显然，这里有回调到对应的Callback中，callback.show

```java
//frameworks/base/services/core/java/com/android/server/notification/toast/CustomToastRecord.java
public final ITransientNotification callback;
public boolean show() {
    if (DBG) {
        Slog.d(TAG, "Show pkg=" + pkg + " callback=" + callback);
    }
    try {
        //这里的callback指的是mTN
        callback.show(windowToken);
        return true;
    } catch (RemoteException e) {
		...
    }
}
```

##### 2.6.4.4.12Toast#TN#show

最终回调到Toast内部类TN的show

```java
//frameworks/base/core/java/android/widget/Toast.java
private static class TN extends ITransientNotification.Stub {
    @Override
    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.P, trackingBug = 115609023)
    public void show(IBinder windowToken) {
        if (localLOGV) Log.v(TAG, "SHOW: " + this);
        mHandler.obtainMessage(SHOW, windowToken).sendToTarget();
    }
}

public void handleShow(IBinder windowToken) {
    if (mView != mNextView) {
        ...
        //最终在mPresenter中show
        mPresenter.show(mView, mToken, windowToken, mDuration, mGravity, mX, mY,
                        mHorizontalMargin, mVerticalMargin,
                        new CallbackBinder(getCallbacks(), mHandler));
    }
}
```

Toast 的窗口类型是 `TYPE_TOAST`，属于系统类型，Toast 有自己的 `token`，不受 Activity 控制。

Toast 通过 WindowManager 将 view 直接添加到了 Window 中，并没有创建 `PhoneWindow` 和 `DecorView`，这点和 Activity 与 Dialog 不同。

#### 2.6.4.5总结

Activity和Dialog都会创建PhoneWindow和DecorView，即这两个是有根布局的，而Toast不创建PhoneWindow和DecorView，其根布局是transient_notification.xml。PopupWindow是子窗口，也没有所谓的PhoneWindow和DecorView，层级关系是主层级为2，子层级为1。

{{< image classes="fancybox center fig-100" src="/WMS/window/window70.png" thumbnail="/WMS/window/window70.png" title="">}}

Activity和Dialog流程类似，详看2.6.1

PopupWindow流程

{{< image classes="fancybox center fig-100" src="/WMS/window/window71.png" thumbnail="/WMS/window/window71.png" title="">}}

Toast流程

{{< image classes="fancybox center fig-100" src="/WMS/window/window68.png" thumbnail="/WMS/window/window68.png" title="">}}





# 3总结

关于专业名词的罗列

|             名称              | 含义                                                         |
| :---------------------------: | ------------------------------------------------------------ |
|           `Window`            | 一个窗口，处理顶级窗口外观和行为策略的抽象类，这个窗口承载了需要绘制的`View` |
|         `PhoneWindow`         | `Window`的具体实现类                                         |
|        `WindowManager`        | 一个接口类，继承`ViewManager`，对`Window`进行管理            |
|      `WindowManagerImpl`      | `WindowManager`的具体实现类，一个`app`层面的`WMS`和`Window`的中间人 |
|  `WMS(WindowManagerService)`  | 真正对`Window`添加，负责管理窗口的启动、添加、删除、大小和层级、处理窗口的触摸事件、处理窗口之间切换动画等 |
|     `WindowManagerGlobal`     | 一个单例，一个进程只有一个实例，这里属于桥接模式             |
|        `ViewRootImpl`         | `Android` 视图层次结构的顶部，`View`树的树根并管理`View`树  ，但它**不是View视图树**的一部分。它是`DecorView ` 和 `WindowManager` 之间的桥梁，是输入响应的中转站，并且触发View的测量、 布局和绘制 |
|         `DecorView `          | `FrameLayout`的子类，是`Android View`视图树的**根布局**      |
|           `Session`           | 应用程序和`WMS`通信之间的通道，应用程序都会有一个`Session`，保存在`WMS`中的`mSessions` |
|       `WindowContainer`       | `Window`容器抽象，定义了一组可以直接或通过其子类以层次结构形式保存 `Window` 的类的常用功能。 |
|         `WindowState`         | `Window`容器，`WMS`管理的一个窗口，可以认为是`Window`容器在`WMS`中的一种**事实窗口**的表示 |
|         `WindowToken`         | `Window`容器，`WMS` 中一组` Window` 的容器，主要作用向`WMS`提供令牌、将属于同一个应用组件的窗口组织在一起 |
|     `RootWindowContainer`     | `Window`容器，根窗口容器。它的孩子是`DisplayContent`         |
|       `ActivityRecord`        | `Window`容器，`WindowToken`的子类，`WMS`中一个`ActivityRecord`对象就代表一个`Activity`对象 |
|    `WallpaperWindowToken`     | `Window`容器，`WindowToken`的子类，用来存放和`Wallpaper`相关的窗口 |
| `DisplayContent.ImeContainer` | `Window`容器，`DisplayArea.Tokens`的子类，用来存放输入法相关的窗口 |

根据android11和android13Window容器不同，这里区分开来

android11

| 名称                     | 含义                                                         |
| ------------------------ | ------------------------------------------------------------ |
| `DisplayContent`         | `Window`容器，`Window`容器，管理一个逻辑屏上的所有窗口，有几个屏幕就会有几个`DisplayContent`，使用`displayId`来区分 |
| `WindowContainers`       | `Window`容器，`DisplayContent`内部类，用于`App`中`Window`的管理集 |
| `NonAppWindowContainers` | `Window`容器，`DisplayContent`内部类，用于非`App`中`Window`的管理集 |
| `DisplayArea`            | `Window`容器，`DisplayContent`之下的对`WindowContainer`进行分组的容器，受`DisplayAreaPolicy`管理 |
| `DisplayArea.Root`       | `Window`容器，`App Window`的根容器                           |
| `TaskDisplayArea`        | `Window`容器，代表了屏幕上一块专门用来存放`App`窗口的区域    |
| `DisplayArea.Tokens`     | `Window`容器，`DisplayArea`的内部类，一个只能包含`WindowToken`对象的`DisplayArea`类型的容器 |
| `Task`                   | `Window`容器，存放`ActivityRecord`的容器，用来管理`ActivityRecord` |
| `ActivityStack`          | `Window`容器，`Task`的子类，用来记录`Activity`历史           |

继承关系如下

{{< image classes="fancybox center fig-100" src="/WMS/window/window6.png" thumbnail="/WMS/window/window6.png" title="">}}

android13

|             名称              | 含义                                                         |
| :---------------------------: | ------------------------------------------------------------ |
|         `DisplayArea`         | `Window`容器，`DisplayContent`之下的对`WindowContainer`进行分组的容器，受`DisplayAreaPolicy`管理 |
|       `TaskDisplayArea`       | `Window`容器，代表了屏幕上一块专门用来存放`App`窗口的区域    |
|            `Task`             | `Window`容器，存放`ActivityRecord`的容器，用来管理`ActivityRecord` |
|     `DisplayArea.Tokens`      | `Window`容器，`DisplayArea`的内部类，一个只能包含`WindowToken`对象的`DisplayArea`类型的容器 |
| `DisplayContent.ImeContainer` | `Window`容器，一个只能包含`WindowToken`对象的`DisplayArea`类型的容器，存放输入法窗口的容器 |
|    `DisplayArea.Dimmable`     | `Window`容器，`Dimmable`也是一个`DisplayArea`类型的`DisplayArea`容器，用于模糊效果。 |
|       `RootDisplayArea`       | `Window`容器，`DisplayArea`层级结构的根节点                  |
|       `DisplayContent`        | `Window`容器，`Window`容器，管理一个逻辑屏上的所有窗口，有几个屏幕就会有几个`DisplayContent`，使用`displayId`来区分 |
|      `DisplayAreaGroup`       | `Window`容器，屏幕上的部分区域对应的`DisplayArea`层级结构的根节点，用于**车载** |

继承关系如下

{{< image classes="fancybox center fig-100" src="/WMS/window/window6_1.png" thumbnail="/WMS/window/window6_1.png" title="">}}

**关于Window和WIndow相关总结**

Window 的管理是基于 C/S 架构，依赖系统服务。服务端是 WMS ，客户端是 WindowManager 。

Window 的更新过程不涉及 WMS ，而添加和删除需要服务端与客户端一同合作，WindowManager 来请求执行，并处理 UI 渲染刷新，WMS 则是负责管理 Window 与系统其他能力结合。

Window 的概念不同于 View ，尽管看起来十分相似，但它们是两种概念。

- Window 代表一套视图树的容器，而 View 是视图树中的节点。

- ViewRootImpl 是视图树对象。

- WindowState 用来描述一个 Window。

- WindowToken 用来表示一组 Window 的容器，可以代表 Activity 和嵌套窗口的父容器。


WindowManager 是我们访问 Window 的入口，Window 的具体实现位于 `WindowManagerService` 中。`WindowManager` 和 `WindowManagerService` 交互是一个 IPC 的过程，最终的 IPC 是在 `RootViewImpl` 中完成的。

# 4问题解答

1）Window，WMS，Activity之间的联系（什么时候Window和Activity和WMS产生联系）

> 时机：Activity和Window之间的联系是在Activity.attach的时候产生的。
>
> 其中new一个PhoneWindow对象并传入当前Activity引用，建立Window和Activity的一一对应关系。此Window是Window类的子类PhoneWindow的实例。Activity在Window中是以mContext属性存在。
>
> 时机：AMS和Window之间的联系是在addView/removeView产生的。通过

2）Window和View区别和联系（什么时候Window和View产生联系）

> 联系：每一个 Window 都对应着一个 View 和 一个 ViewRootImpl 。Window 表示一个窗口的概念，也是一个抽象的概念，它并不是实际存在的，它是以 View 的方式存在的。
>
> 区别：Window 的概念不同于 View ，尽管看起来十分相似，但它们是两种概念。Window 代表一套视图树的容器，而 View 是视图树中的节点。
>
> 时机：Window和View产生联系是在外部App调用AppCompatActivity#setContent时候，最终会生成DecorView。

3）Window的类别和层次关系（为什么PopWindow会遮住App的显示，锁屏为什么看不到Launcher）

> Window分为不同容器，其中主要是WindowToken和WindowSate，一个ActivityRecord（WindowToken容器）可以包含多个WindowState容器。
>
> 每种不同的容器代表不同的层级，主要分成主层级和次层级，主层级分为[1,36]，次层级为[-2,-3]。主层级越大，离用户越近，主层级相同，次层级越大离用户越近。主层级主要看WindowToken相关容器，次层级主要看WindowState相关容器。
>
> PopWindow会遮住App的显示：App主层级为2，次层级默认为0，popWindow是不能独立存在，也就是主层级和App相同为2，次层级为1大于App所在的Window，会遮住App显示
>
> 锁屏为什么看不到Launcher：锁屏之后显示的是锁屏界面，这个界面的主层级在App层之上，那么会遮住App的显示，即看不到Launcher。

4）activity与 PhoneWindow与DecorView关系 （如果我添加一个View，会添加到系统层面去吗）

> PhoneWindow内部持有DecorView，而且所有跟当前Window相关的View处理，最终都在DecorView里面进行。
>
> 添加一个View，包括自定义View，实际上就是所在的DecorView里面添加的一个App层的布局，是不会跟系统层面相关的。



# 源码

查看方式可以直接访问下面的网站，点击[这里](http://aospxref.com/)

直接搜索对应变量或者方法名

{{< image classes="fancybox center fig-100" src="/WMS/window/window1.png" thumbnail="/WMS/window/window1.png" title="">}}

除了上述方式之外，如果大量的查找相关的方法，上面的方式反而比较笨拙，可以下载对应的源码，/frameworks/base点击[这里](https://github1s.com/aosp-mirror/platform_frameworks_base)，/frameworks/native点击[这里](https://github1s.com/GrapheneOS/platform_frameworks_native)

目前看来网页内置了VSCode，搜索还是联想确实会比直接在github上看快多了，不过有一个缺点就是直接打开文件功能比较弱

{{< image classes="fancybox center fig-100" src="/WMS/window/window2.png" thumbnail="/WMS/window/window2.png" title="">}}

# demo下载

另外，处理本文举例的几个简单例子，可能有读者并没有android13的设备，可以下载笔者准备的几个demo，分析具体的Window容器、层级关系等，可以点击[这里](https://github.com/YangYang48/project/tree/master/window-13demo)。

{{< image classes="fancybox center fig-100" src="/WMS/window/window12.png" thumbnail="/WMS/window/window12.png" title="">}}

# 参考

源码下载

[[1] 星辰之力, 官网 Android framework源码git地址, 2022.](https://www.bbsmax.com/A/D8548N7w5E/)

[[2] GrapheneOS, Android framework_native, 2023.](https://github.com/GrapheneOS/platform_frameworks_native)

[[3] aosp-mirror, Android framework_base, 2023.](https://github.com/aosp-mirror/platform_frameworks_base)





AMS启动

[[1] Pingred, 手把手带你搞懂AMS启动原理, 2023.](https://mp.weixin.qq.com/s/8Fii9ofRpAJscXJC2bIwfQ)

[[2] Avengong, Android渲染(一)_系统服务WMS启动过程(基于Android10), 2022.](https://juejin.cn/post/7125057980420063268)

[[3] 345丶, Android | WMS 解析 (一), 2022.](https://juejin.cn/post/7160870305051705352)



window and windowmanager

[[1] Chunyu J, Android Window 机制, 2022.](https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650269748&idx=1&sn=33b385cc4ebc9a9b563d500aae3867e6&chksm=8863195bbf14904d961e5ef578785424f37fc00103d9947d102f9588a1939fb51c084a5fde68&scene=27)

[[2] 小余的自习室, 源码学习时间，Window Manager in Android, 2023.](https://mp.weixin.qq.com/s/iidzG_xV9RhWuCiNilGtMQ)

[[3] 345丶, Android | 理解 Window 和 WindowManager, 2022.](https://juejin.cn/post/7076274407416528909)



addView/addWindow

[[1] a5right, Android View从绘制到上屏全过程解析, 2023.](https://mp.weixin.qq.com/s/q_V6JI6Ng6Huj-EIG2o24w)

[[2] Anderson大码渣, What is DecorView and android.R.id.content?, 2017.](https://www.jianshu.com/p/579e9cc8b83c)

[[3] 刘望舒, Android解析WindowManagerService（二）WMS的重要成员和Window的添加过程, 2017.](https://juejin.cn/post/6844903506541805582)

[[4] 刘望舒, Android解析WindowManagerService（三）Window的删除过程, 2018.](https://juejin.cn/post/6844903560262451213)

[[5] learnframework, android 13 WMS/AMS系统开发-SplashScreen的添加与移除分析, 2023.](https://blog.csdn.net/learnframework/article/details/129051608?spm=1001.2014.3001.5502)

[[6] 被代码淹没的小伙子, 【Window系列】——PopupWindow的前世今生, 2019.](https://www.jianshu.com/p/9dafea9cb3c0)



window层级/windowcontainer

[[1] TechMerger, 你知道 Android 是如何管理复杂的 Window 层级的？, 2022.](https://m.ithome.com/html/647058.htm)

[[2] 小余的自习室, “一文读懂”系列：无处不在的WMS, 2022.](https://juejin.cn/post/7172598258064162830#heading-4)

[[3] Looperjing, Android窗口系统第一篇---Window的抽象概念和WMS相关数据结构理解, 2017.](https://juejin.cn/post/6844903489366130701)

[[4] aofan9566, WindowState注意事项, 2015.](https://blog.csdn.net/aofan9566/article/details/101821620)

[[5] weixin_41591109, android 11 的Window的屏幕的架构图, 2022.](https://blog.csdn.net/weixin_41591109/article/details/128025338?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1-128025338-blog-126748372.235%5Ev27%5Epc_relevant_recovery_v2&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1-128025338-blog-126748372.235%5Ev27%5Epc_relevant_recovery_v2&utm_relevant_index=2)

[[6] qluka, Android 窗口结构(一) 窗口层级构造, 2022.](https://blog.csdn.net/q1165328963/article/details/127746382)

[[7] qluka, Android 窗口结构(二) 添加Home Task, 2022.](https://blog.csdn.net/q1165328963/article/details/127841486?spm=1001.2014.3001.5502)

[[8] Geralt_z_Rivii, 2【Android 12】WindowContainer类, 2022.](https://blog.csdn.net/ukynho/article/details/126748176?spm=1001.2014.3001.5502)

[[9] Geralt_z_Rivii, 3【Android 12】DisplayArea层级结构, 2022.](https://blog.csdn.net/ukynho/article/details/126748372)

[[10] TigherGuo, 安卓12窗口层次: DisplayArea树, 2022.](https://blog.csdn.net/jieliaoyuan8279/article/details/123157937?spm=1001.2101.3001.6650.15&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-15-123157937-blog-126748176.235%5Ev27%5Epc_relevant_recovery_v2&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-15-123157937-blog-126748176.235%5Ev27%5Epc_relevant_recovery_v2&utm_relevant_index=20)

[[11] HansChen_, Android 12 - WMS 层级结构 && DisplayAreaGroup 引入, 2021.](https://blog.csdn.net/shensky711/article/details/121530510?spm=1001.2101.3001.6650.9&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-9-121530510-blog-126748176.235%5Ev27%5Epc_relevant_recovery_v2&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-9-121530510-blog-126748176.235%5Ev27%5Epc_relevant_recovery_v2&utm_relevant_index=14)

[[12] learnframework, android 13 WMS/AMS系统开发-窗口层级相关DisplayArea,WindowContainer, 2023.](https://blog.csdn.net/learnframework/article/details/129101781?spm=1001.2014.3001.5502)

[[13] learnframework, android 13 WMS/AMS系统开发-窗口层级相关DisplayArea,WindowContainer第二节, 2023.](https://blog.csdn.net/learnframework/article/details/129115725?spm=1001.2014.3001.5502)

[[14] learnframework, android 13 WMS/AMS系统开发-窗口层级相关Task/ActivityRecord/WindowState/WindowToken放置图层创建 第三节, 2023.](https://blog.csdn.net/learnframework/article/details/129175890?spm=1001.2014.3001.5502)

[[15] learnframework, android 13 WMS/AMS系统开发-窗口层级相关SurfaceFlinger图层创建 第三节, 2023.](https://blog.csdn.net/learnframework/article/details/129132667?spm=1001.2014.3001.5502)



锁屏相关

[[1] learnframework, Android 12中系统Wallpaper详解1--锁屏透看壁纸和桌面透看壁纸的切换, 2022.](https://blog.csdn.net/learnframework/article/details/127483545)



view相关

[[1] 凶残的程序员, Android 屏幕绘制机制及硬件加速, 2019.](https://blog.csdn.net/qian520ao/article/details/81144167)

[[2] 沧浪之水, Android GPU硬件加速渲染流程（上）, 2022.](https://zhuanlan.zhihu.com/p/464492155)

[[3] 沧浪之水, Android GPU硬件加速渲染流程（下）, 2022.](https://zhuanlan.zhihu.com/p/464564859)

[[4] 𝓑𝓮𝓲𝓨𝓪𝓷𝓰, 软件绘制 & 硬件加速绘制 【DisplayList & RenderNode】, 2022.](https://juejin.cn/post/7128779284503592991)

[[5] 𝓑𝓮𝓲𝓨𝓪𝓷𝓰, 硬件加速绘制基础知识, 2022.](https://juejin.cn/post/7129330919990624269)

[[6] 失落夏天, View绘制流程3-Vsync信号是如何发送和接受的, 2022.](https://blog.csdn.net/rzleilei/article/details/126118441)

[[7] houliang120, Android垂直同步信号VSync的产生及传播结构详解, 2016.](https://blog.csdn.net/houliang120/article/details/50908098)





