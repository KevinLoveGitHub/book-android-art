---
title: 《Android 开发艺术探索》- 第一章内容总结
date: '2018-01-14T00:39:08.000Z'
tags:
  - 读书笔记
  - toc: true
---

# 第一章

## 生命周期

![](http://ojxp1924f.bkt.clouddn.com/1515494514.png%20)

### onCreate\(\)

Activity 开始创建，并且只会被调用一次

### onStart\(\)

Activity 创建完成开始启动，此时的 Activity **已经可见**，但是还没有出现在前台，不能交互，可以理解为已经显示出来了，但是还看不到

### onRestart\(\)

Activity 重新启动，通常当 Activity 从不可见到可见会执行该声明周期函数，比如用户按下 Home 键或打开了一个新的 Activity 后再回到这个 Activity

### onResume\(\)

Activity 可见，Activity 此时在前台活动，onStart\(\) 执行的时候这个 Activity 还是后台运行，直到 onResume\(\) 这个 Activity 才在前台活动

### onPause\(\)

Activity 正在停止，这个 Activity 的 onPause\(\) 执行完毕才能执行新的 Activity 的 onResume\(\)，所以不建议在 onPause\(\) 里面做耗时操作，这样会影响打开新的 Activity 的速度

### onStop\(\)

Activity 即将停止，onPause\(\) 之后就会立即执行该生命周期函数，可以执行稍微重量级的回收工作。理想状态下 onPause\(\) 后快速回到当前 Activity，onStop\(\) 则不会执行

### onDestory\(\)

Activity 即将销毁，此时可以执行一些资源回收等操作

### 常见情况

* Activity 首次启动：onCreate\(\) -&gt; onStart\(\) -&gt; onResume\(\)
* 返回桌面或打开新的 Activity：onPause\(\) -&gt; onStop\(\)
* 重新回到这个 Activity：onRestart\(\) -&gt; onStart\(\) -&gt; onResume\(\)
* 返回键或者调用 finish\(\) 函数：onPause\(\) -&gt; onStop\(\) -&gt; onDestroy\(\)

#### A Activity 启动 B Activity

![](http://ojxp1924f.bkt.clouddn.com/1515504907.png%20)

| Activity | Method |
| :--- | :--- |
| A | onPause\(\) |
| B | onCrete\(\) |
| B | onStart\(\) |
| B | onResume\(\) |
| A | onStop\(\) |

* [ ] 需要对源码进行分析

### 异常情况

分别有两种情况：

* 系统配置发生变化，例如横竖屏切换
* 系统内存不足杀掉 Activity

这种情况下生命周期函数执行顺序就为：

![](http://ojxp1924f.bkt.clouddn.com/1515591652.png%20)

```text
s=>start: 横竖屏切换或者Activity被杀死
e=>end: 执行结束
op1=>operation: onPause()
op2=>operation: onSaveInstanceState()
op3=>operation: onStop()
op4=>operation: onDestroy()
op5=>operation: onCreate()
op6=>operation: onStart()
op7=>operation: onRestoreInstanceState()
op8=>operation: onResume()
s->op1->op2->op3->op4->op5->op6->op7->op8->e
```

由于 Activity 是因为异常原因才结束的，所以在结束的时候会调用 `onSaveInstanceState()` 函数保存状态，该函数的调用顺序和 `onPause()` 函数并没有既定的时序关系，可能是 `onPause()` 之前也可能是之后，Activity 被重新创建后会调用 `onRestoreInstanceState()` 函数中的 `Bundle` 就是之前保存在的 `onSaveInstanceState()` 中的数据，并且此时 `onCreate()` 中的参数也同样是该 `Bundle` 对象

Activity 异常结束并重新创建的过程中，系统同时把该 Activity 的视图结构也保存了下来，所以同 Activity 一样，所有的 View 也都具有 `onSaveInstanceState()` 和 `onRestoreInstanceState()` 函数

Activity 的优先级可以分为三级：

* 前台 Activity：可见并可交互，优先级最高
* 可见非前台 Activity：当 Activity 被弹窗遮盖时
* 后台 Activity：已经暂停的 Activity，优先级最低，系统内存不足时优先回收此类 Activity

如果不想在系统配置发生变化时，销毁并重新创建 Activity，可以在 `AndroidManifest.xml` 中配置该 Activity 的属性,从 Android 3.2（API 级别 13）开始，当设备在纵向和横向之间切换时，**屏幕尺寸**也会发生变化。因此，在开发针对 API 级别 13 或更高版本的应用时，若要避免由于设备方向改变而导致运行时重启，则除了 "orientation" 值以外，您还必须添加 "screenSize" 值：

```markup
<activity android:name="Activity"
            android:configChanges="orientation|screenSize">
</activity>
```

所有下列字符串均是该属性的有效值： ![](http://ojxp1924f.bkt.clouddn.com/1515601163.png%20)

### Dialog 弹出

弹出和隐藏 Dialog 时并不会执行所在 Activity 的任何生命周期，因为 Dialog 被看做是依附在 Activity 的一个 View，可以理解为它本来就是 Activity 的一部分

## 启动模式

### standard 标准模式

系统默认模式，每次启动 Activity 都会在当前 Activity 所在栈中创建一个实例，Activity 的生命周期函数正常执行，不管这个实例是否存在任务栈中，如果使用 ApplicationContext 启动一个 Activity，就会报错：

```text
Caused by: android.util.AndroidRuntimeException: Calling startActivity() from outside of an Activity  context requires the FLAG_ACTIVITY_NEW_TASK flag. Is this really what you want?
```

报错信息说的很清楚，startActivity\(\) 函数被非 Activity Context 调用需要加上 `FLAG_ACTIVITY_NEW_TASK` 标识，之所以会报这个错是因为，Application Context 并没有运行在一个承载 Activity 实例的任务栈中，加上该标识后，再次启动就会为此 Activity 实例单独创建一个任务栈，此时该 Activity 实际上是以 singleTask 模式启动

### singleTop 栈顶复用模式

该模式下，如果即将启动的 Activity 的实例已经存在任务栈中且位于栈顶，该 Activity 实例则不会被创建，并且位于栈顶的 Activity 的 `onNewIntent` 函数会被回调，如果新 Activity 的实例存在任务栈中，但并没有位于栈顶，新 Activity 实例依旧会正常创建

### singleTask 栈内复用模式

这是一种单例模式，只要新 Activity 实例在任务栈中存在，不管是否位于栈顶，都不会创建新的实例。当 singleTask 模式的 Activity 启动时，系统会先找是否存在该 Activity 实例所需的任务栈，如果不存在则新建一个任务栈存放该 Activity 实例，如果存在并且该任务栈中存在该 Activity 的实例，则把该实例调到栈顶并回调其 `onNewIntent` 函数同时会移除该任务栈中位于 Activity 实例上方的所有实例，如果不存在实例，就创建新 Activity 实例并压入栈内

如果 K 需要任务栈 T，T 中此时有 AKBC，任务栈中存在 K，所有就不会重新创建，系统会把 K 调到栈顶，同时回调 `onNewIntent` 函数，由于 singleTask 具有 clearTop 效果，所有此时的任务栈中情况为：AK

### singleInstance 单实例模式

可以理解为增强型的 singleTask 模式，它除了具有 singleTask 所有属性外，还有就是具有该模式的 Activity 实例都会拥有单独属于自己的任务栈

### 特殊情况

singleTask singletTop singleInstance 三种模式下，如果新 Activity 的实例没有创建，复用任务栈中的实例对象，生命周期只会执行 `onPause` `onResume`，以及回调 `onInstance` 函数

singleTask 模式下会有一种特殊情况：

![](http://ojxp1924f.bkt.clouddn.com/1515678806.png%20)

前台任务栈存在 12，后台任务栈存在 XY，当启动 Y 时，整个后台任务栈都会切换到前台，此时从 Y 返回，其实是到 X，并不是到 2。如果启动 X ，那么 Y 就会直接出栈

### 设置 Activity 启动模式

有两种设置方式：

* AndroidMinifest.xml 文件中设置

```markup
<activity android:name=".Activity"
    android:launchMode="singleInstance">
</activity>
```

* Java 文件中设置

```java
Intent intent = new Intent(this, SecondActivity.class);
intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
startActivity(intent);
```

如果两种方式同时存在，Java 文件中的设置方式优先级要高于 AndroidMinifest.xml 文件中的设置方式。其次上述两种设置方式的限定范围也有所不同，第一种方式不能直接为 Activity 设置 `FLAG_ACTIVITY_CLEAR_TOP` 标识，第二种方式设置 singleInstance 启动模式

### TaskAffinity

上提到 singleTask 启动模式中，Activity 会寻找所需的任务栈，taskAffinity 参数标识了 Activity 所需任务栈的名字，默认情况下所有 Activity 所需任务栈名称都是包名，taskAffinity 主要和 singleTask 模式或者 allowTaskReparenting 属性配合使用，下面有一个 taskAffinity 和 singleTask 模式配合使用的例子：

有 A、B、C 三个 Activity，包名是 `org.lovedev.chapter_1`，其中 A 没有设置任何参数，B 和 C 的启动模式都为 singleTask，taskAffinity 都设置为 `org.lovedev.kevin`。A 启动 B，B 启动 C，C 再启动 A，此时使用 `adb shell dumpsys activity` 命令查看下任务栈情况：

![](http://ojxp1924f.bkt.clouddn.com/1515852454.png%20)

可以看到当前具有两个任务栈，由于 A 的启动模式是 standard，所有在 C 启动 A 时，系统会为 A 创建一个新的实例，压入启动它的那个栈中，此时再从 A 启动 B，接着看一下任务栈情况：

![](http://ojxp1924f.bkt.clouddn.com/1515852833.png%20)

由于 B 的启动模式为 singleTask ，由于 singleTask 的 clearTop 特性，系统销毁任务栈中 B 上面的所有实例，此时如果按返回键，B 出栈，B 所在任务栈中已经没有实例，所以该任务栈也就不存在了，此时回到最初启动的 A 所在后台任务栈：

![](http://ojxp1924f.bkt.clouddn.com/1515853155.png%20)

### Activity 的 Flags

* FLAG\_ACTIVITY\_NEW\_TASK

  指定 singleTask 启动模式

* FLAG\_ACTIVITY\_SINGLE\_TOP

  指定 singleTop 启动模式

* FLAG\_ACTIVITY\_CLEAR\_TOP

  通常和 `FLAG_ACTIVITY_NEW_TASK` 配合使用，如果启动模式是 standard，那么该 Activity 实例和之上的所有实例都将出栈，系统会重新创建该 Activity 新的实例压入该栈中

## 隐式启动

想要隐式启动 Activity 需要 Intent 对象能够匹配上目标组件的 IntentFilter 中所设置的过滤条件，IntentFilter 过滤条件有 action、category、data：

```markup
<activity android:name=".IntentFilterActivity">
    <intent-filter>
        <action android:name="org.lovedev.kevin01"/>
        <action android:name="org.lovedev.kevin02"/>
        <action android:name="org.lovedev.kevin03"/>

        <category android:name="org.lovedev.category01"/>
        <category android:name="org.lovedev.category02"/>
        <category android:name="android.intent.category.DEFAULT"/>

        <data
            android:host="org.lovedev"
            android:scheme="http"/>
    </intent-filter>

    <intent-filter>
        <action android:name="org.lovedev.kevin04"/>

        <category android:name="android.intent.category.DEFAULT"/>

        <data android:mimeType="image/*"/>
    </intent-filter>
</activity>
```

需要注意的是，一个过滤列表可以有多个 action、category、data 属性，也可以有多个过滤列表

### action 匹配规则

intent 中的 action 只要和过滤规则中的任意一条 action 匹配即可，action 匹配区分大小写

### category 匹配规则

intent 可以设置多个 category，但是每个 category 都需要匹配上过滤规则中的 category 才算成功，也可以不设置 category，因为在 startActivity 默认添加了 `android.intent.category.DEFAULT` 这个 category， 所以过滤条件中必须添该 category

### data 匹配规则

data 语法如下：

```markup
<data android:scheme="string"
      android:host="string"
      android:port="string"
      android:path="string"
      android:pathPattern="string"
      android:pathPrefix="string"
      android:mimeType="string" />
```

其中包括两部分，URI 和 mimeType，mimeType 是指媒体类型，例如 image/jpeg、audio/mpeg4-generic、image/\*，URI 包含数据就比较多了，URI 结构如下：

```markup
<scheme>://<host>:<port>[<path>|<pathPrefix>|<pathPattern>]
```

示例如下：

```markup
http://www.lovedev.org/andrid/activity
```

* 如果 URI 中没有指定 scheme，整个 URI 参数无效
* 如果 host 未指定，整个 URI 参数同样无效

需要注意的是，如果过滤规则中只指定了 mimeType，未指定 URI，如下所示：

```markup
<intent-filter>
    <action android:name="org.lovedev.kevin"/>

    <category android:name="android.intent.category.DEFAULT"/>

    <data android:mimeType="image/*"/>
</intent-filter>
```

这种情况下会有默认的 URi，默认值为 content 和 file

### 判断规则是否匹配

如果不进行判断隐式启动是否是否匹配，就出抛出异常，导致应用崩溃，可以使用 Intent 的 resolveActivity 函数或者 PackageManager 的 resolveActivity 函数进行判断是否匹配，使用方法参考官方 API 文档：

\[PackageManager.resolveActivity\]\([https://developer.android.com/reference/android/content/pm/PackageManager.html\#resolveActivity\(android.content.Intent](https://developer.android.com/reference/android/content/pm/PackageManager.html#resolveActivity%28android.content.Intent), int\)\)

\[Intent.resolveActivity\]\([https://developer.android.com/reference/android/content/Intent.html\#resolveActivity\(android.content.pm.PackageManager](https://developer.android.com/reference/android/content/Intent.html#resolveActivity%28android.content.pm.PackageManager)\)\)

