---
title: IntentFilter 的匹配规则
date: 2016-10-16
tags:
  - Android
---
### IntentFilter 介绍

> Structured description of Intent values to be matched. An IntentFilter can match against actions, categories, and data (either via its type, scheme, and/or path) in an Intent. It also includes a "priority" value which is used to order multiple matching filters.



熟悉 Android 开发的朋友都知道，当我们创建了一个供其他组件或应用调用的 Activity 时，我们都会在 AndroidManifest.xml 中配置该 Activity 的  IntentFilter 节点。一个 Activity 可以拥有多个 IntentFilter，用于匹配不同的 Intent。IntentFilter 由三种数据类型组成，分别是 action，category，data，其中 data 由 mimeType 与 URI 组成。一个 IntentFilter 中的 action，category，data 也可以有多个，只有当 Intent 中的信息满足 IntentFilter 的过滤条件才算匹配成功，只有匹配成功后，才能启动目标组件。还有，Intent 只要能匹配多个 IntentFilter 中的任意一个，就能启动对应的组件。下面是一个过滤规则的示例：
<!--more-->
```java
<intent-filter>
    <action android:name="me.zhengken.a" />
    <action android:name="me.zhengken.b" />
    <category android:name="me.zhengken.c" />
    <category android:name="me.zhengken.d" />
    <category android:name="android.intent.category.DEFAULT" />
    <data android:mimeType="text/plain" />
</intent-filter>
```

### action 匹配规则

action 是一个字符串，且对大小写敏感，Android 系统预定义了很多 action，我们可以在 IntentFilter 中自己定义。IntentFilter 必须要有不少于一个的 action，前面我们讲了，一个 IntentFilter 可以有多个 action，那么 Intent 中的 action 只要能匹配 IntentFilter 诸多 action 的其中一个就算是匹配成功了。根据上面的示例，Intent 中的 action 等于 me.zhengken.a 或者是 me.zhengken.b，它都能和 IntentFilter 的 action 匹配成功。

### category 匹配规则

category 是一个字符串，Android 系统预定义了很多 category，我们在 IntentFilter 中定义自己的 category。category 匹配规则和 action 不同，Intent 中可以有 category，也可以没有 category，当没有为 Intent 指定 category，我们调用 startActivity 或者 startActivityForResult，系统会为 Intent 添加 android.intent.category.DEFAULT。如果 Intent 有一个或多个 category，那么每一个 category 都要在 IntentFilter 中找到与之对应的 category。根据上面的示例，intent.addCategory("me.zhengken.c") 或者 intent.addCategory("me.zhengken.d") 或者 intent.addCategory("me.zhengken.c")；intent.addCategory("me.zhengken.d") 或者 intent 没有 category，都能与示例中的 IntentFilter 匹配。需要注意的是，如果 IntentFilter 希望能被隐式调用的话，那么它必须要有 android.intent.category.DEFAULT ，因为隐式调用的 intent 一般只有 default 这个 category。

### data 匹配规则

data 的匹配规则与 action 类似，当 IntentFilter 定义了 data，那么 Intent 中也必须定义与之匹配的 data。在深入 data 之前，我们先介绍一下 data。
data 的语法如下：
```java
<data android:scheme="string"
      android:host="string"
      android:port="string"
      android:path="string"
      android:pathPattern="string"
      android:pathPrefix="string"
      android:mimeType="string" />
```

data 由 mimeType 与 URI 组成，当向 IntentFilter 加入 data 节点，data 可以只包括 mimeType 或者 URI 或者兼具 mimeType 和 URI。URI 的语法如下：
```java
<scheme>://<host>:<port>[<path>|<pathPrefix>|<pathPattern>]
```
举个例子：
```java
http://zhengken.me:80/2016/10/06/activity-launch-mode-analysis/index.html
```
组成 URI 的各个属性是可选的，但各个属性也都是线性相关：

 - scheme: URI 的模式，如 http、content、file，如果 URI 的 scheme 属性没有指定，那么其他的属性将被忽略。
 - host: URI 的主机名如果 URI 的 host 属性没有指定，那么 port 和所有的 path 属性将被忽略。
 - port: URI 的主机号，如 80，仅当 URI 中指定了 scheme 与 host，port 才是有意义的。
 - path: 表示完整的路径信息。
 - pathPrefix: 表示路径的前缀信息。
 - pathPattern: 表示完整的路径信息，但是它里面可以包含通配符 * ，表示 0 个或多个任意字符，在 Java 中，`\\` 的意思是"我要插入一个正则表达式的反斜线"，所以其后的字符具有特殊意义，表示字符串的 `*` 应该写 `\\*`，表示正常的反斜线应该写成 `\\\\`。

data 匹配规则示例：
```xml
<intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <data android:mimeType="image/*" />
</intent-filter>
```
该过滤规则指定了媒体类型为所有的图片，那么 Intent 中的媒体类型也必须为图片才能匹配，我们看到 data 并没有指定 URI，这种情况下是有默认值的，值为 content 和 file。笔者编程验证了一下，当 Intent 中的 data 没有指定 URI 也可以与该 IntentFilter 匹配，如果指定了 URI 的话，那么 URI 的 scheme 应为 content 或者 file。
```java
Intent intent = new Intent();
intent.setAction("android.intent.action.VIEW");
intent.setDataAndType(Uri.parse("file://abc"), "image/jpg");
startActivity(intent);
```
左图 Intent scheme 为 file，右图 scheme 为 http（直接跳到系统相册应用）
![此处输入图片的描述][1]

另外，如果要为 Intent 完整的指定 data，必须调用 setDataAndType()，不能分别调用 setData() 和 setType()，因为从源码我们可知 setData() 和 setType() 会互相清除对方的值。
```java
public Intent setType(String type) {
    mData = null;
    mType = type;
    return this;
}
```

另一种 data 示例
```java
<intent-filter>
    <data android:mimeType="audio/mp3" android:scheme="http" .../>
    <data android:mimeType="video/mpeg" android:scheme="http" .../>
    ...
</intent-filter>
```
该示例指定了两个 data，且每个 data 均有 mimeType 和 scheme。为了与之匹配，我们可以如下书写：
```java
intent.setDataAndType(Uri.parse("http://abc"), "audio/mp3");
```
或者
```java
intent.setDataAndType(Uri.parse("http://abc"), "video/mpeg");
```
除了两个示例，还要提一下 data 的特殊写法：
```java
<intent-filter . . . >
<data android:scheme="something" android:host="project.example.com" />
    . . .
</intent-filter>
```
等价于下面这种写法
```java
<intent-filter . . . >
    <data android:scheme="something" />
    <data android:host="project.example.com" />
    . . .
</intent-filter>
```
下面我们对 data 的用法做一个小结：

 - 仅当 IntentFilter 未指定任何 URI 或 MIME 类型时，不含 URI 和 MIME 类型的 Intent 才能匹配。
 - 当 Intent 包含 URI，但不含 MIME 类型时，仅当其 URI 与 IntentFilter 的 URI 格式匹配、且 IntentFilter 同样未指定 MIME 类型时，才会通过测试。
 - 当 Intent 包含 MIME，但不含 URI 类型时，仅当其 MIME 与 IntentFilter 的 MIME 格式匹配、且 IntentFilter 同样未指定 URI 类型时，才会通过测试。
 - 仅当 MIME 类型与过滤器中列出的类型匹配时，包含 URI 和 MIME 的 Intent 才会通过匹配的 MIME 类型部分。如果 Intent 的 URI 与过滤器中的 URI 匹配，或者如果 Intent 具有 content 或 file 且 IntentFilter 未指定 URI，则 Intent 会通过匹配的 URI 部分。换句话说，如果 IntentFilter 仅列出 MIME 类型，则假定组件支持 content 和 file 数据。

### Intent 匹配
我们在手机里面看到的 App 启动图标，是通过主页应用通过使用指定 `ACTION_MAIN` 操作和 `CATEGORY_LAUNCHER` 类别的 Intent 过滤器查找所有 Activity，以此填充应用启动器。

另外，PackageManager 提供了一整套 query...() 方法来返回能够接受特定 Intent 的组件，它还提供了一系列类似 resove...() 方法来确定相应 Intent 的最佳组件。我们可以用这些方法来确定系统是否有目标组件，以此来避免不必要的错误产生。

### 参考资料

本文是笔者阅读 Android 开发艺术探索和谷歌官网文档，经过个人思考及编程验证所得。

 1. https://developer.android.com/guide/components/intents-filters.html#Receiving
 2. https://developer.android.com/guide/topics/manifest/data-element.html
 3. https://developer.android.com/reference/android/content/IntentFilter.html
 4. [Android开发艺术探索 任玉刚 著][2]


  [1]: http://odc50o546.bkt.clouddn.com/android-intent-filter-analysis.png
  [2]: https://item.jd.com/11760209.html