---
title: Android 中的序列化与反序列化
date: 2016-10-23
tags:
  - Android
  - Java
---

### 概念

#### 什么是序列化

> [中文wiki] 在数据储存与传送的部分是指将一个对象存储至一个储存媒介，例如档案或是记亿体缓冲等，或者透过网络传送资料时进行编码的过程，可以是字节或是XML等格式。

> [英文wiki] In computer science, in the context of data storage, serialization is the process of translating data structures or object state into a format that can be stored (for example, in a file or memory buffer, or transmitted across a network connection link) and reconstructed later in the same or another computer environment.

我觉得英文维基比中文维基更容易明白 ：) ，反序列化就是序列化的逆过程。

<!--more-->


#### 为什么要序列化

在上面的概念中，我们大概明白为什么要序列化。

 - 对象保存到本地
 - 对象在网络中传输
 - 对象在内存中传输

在《Java 编程思想》中还提到远程方法调用（Remote Method Invocation，RMI）这一特性，它使存活于其他计算机上的对象使用起来就像是存活于本机上一样。
 
#### 如何序列化

在 Android 中，有两种方法可以选择，一种是 Java 提供的 Serializable，另一种是 Android 提供的 Parceable。在内存中，Parceable 要比 Serializable 的效率高很多，同时，使用 Parceable 稍微会复杂点。

### Serializable 序列化

只要对象实现了 Serializable 接口（该接口仅是一个标记接口，不包括任何方法），那么该对象就是可以被序列化的了。例如有一个 Login 类，它实现了 Serializable 接口。

```java
public class Login {

    private static final long serialVersionUID = -8483115513871999402L;

    private String mUserName;

    private String mPassword;

    public Login() {
    }

    public Login(String userName, String password) {
        mUserName = userName;
        mPassword = password;
    }
}
```

然后使用 ObjectOutputStream 的 writeObject 将对象写入到文件中，或者是使用 ObjectInputObject 的 readObject 将对象从文件读取。

```java
public class SerializableDemo {

    public static void main(String[] args) throws Exception {

        ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("login"));
        Login login = new Login("zhengken", "zhengken.me");
        out.writeObject(login);
        System.out.println(login);

        System.out.println("--------------------------------------");

        ObjectInputStream in = new ObjectInputStream(new FileInputStream("login"));
        Login loginRead = (Login) in.readObject();
        System.out.println(loginRead);

    }
}
```

输出：

```java
mUserName = zhengken
mPassword = zhengken.me
--------------------------------------
mUserName = zhengken
mPassword = zhengken.me
```

#### serialVersionUID

> serialVersionUID 是用来辅助序列化和反序列化过程的，原则上序列化后的数据中的 serialVersionUID 只有和当前类的 serialVersionUID 相同才能够正常地被反序列化。serialVersionUID 的工作机制是这样的：序列化的时候系统会把当前类的 serialVersionUID 写入序列化的文件中，当反序列化的时候系统会去检测文件中的 serialVersionUID，看它是否和当前类的 serialVersionUID 一致，如果一致就说明序列化的类的版本和当前版本是相同的，这个时候可以成功发序列化；否则就说明当前类和序列化的类相比发生了某些变换，比如成员变量的数量、类型可能发生了改变，这个时候是无法正常反序列化的，因此会报如下错误：

```java
Exception in thread "main" java.io.InvalidClassException: Login; local class incompatible: stream classdesc serialVersionUID = -8483115513871999403, local class serialVersionUID = -84831155138719
```

serialVersionUID 既可以自己指定，也可以让编译器根据当前类的结构来生成它的 hash 值，这两者的效果是一致的。当然，我们可以不用指定 serialVersionUID 的值，在序列化的过程中，让编译器自动生成，不过作为好的编程习惯，我们还是应该显式的来指定。

#### static 成员

由于序列化只能保存对象的状态，而 static 是属于某个类的状态，所以 static 成员将不能被序列化。

#### transient 关键字

如果某个对象不希望将它的敏感信息（如密码）进行序列化，即使对象中的这些信息是 private 属性，一旦序列化以后，人们就可以通过读取文件或者拦截网络传输的方式来访问到它。在这种情况下，我们可以用 transient 关键字逐个字段地关闭序列化。例如，我们将 Login 类修改如下：

```java
private String mUserName;

private transient String mPassword;
```

#### 构造函数的调用

对于 Serializable 对象而言，对象完全以它存储的二进制位为基础来构造，而不调用构造器。

### Parcelable 序列化

和 Serializable 一样，Parcelable 也是一个接口，只要实现这个接口，一个类的对象就可以实现序列化并通过 Intent 和 Binder 传递。下面的代码是以 Parcelable 的方式对 Login 进行序列化。

```java
public class Login implements Parcelable {

    private String mUserName;

    private String mPassword;

    public Login() {
    }

    public Login(String userName, String password) {
        mUserName = userName;
        mPassword = password;
    }

    protected Login(Parcel in) {
        mUserName = in.readString();
        mPassword = in.readString();
    }

    public static final Creator<Login> CREATOR = new Creator<Login>() {
        @Override
        public Login createFromParcel(Parcel in) {
            return new Login(in);
        }

        @Override
        public Login[] newArray(int size) {
            return new Login[size];
        }
    };

    @Override
    public String toString() {
        return "mUserName = " + mUserName + "\nmPassword = " + mPassword;
    }

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(mUserName);
        dest.writeString(mPassword);
    }
}
```

在 Android Studio 中，实现 Parcelable 只要按两次 Alt + Enter 就可以了，很智能。Parcel 在序列化中起着核心的作用，Parcel 是一个存放读取数据的容器，Android 的进程间通信就是使用 Parcel 来进行客户端与服务端数据的交互。在代码中可以看到，在序列化中需要做三件事情，序列化、反序列化和内容描述。序列化功能由 writeToParcel 来完成；反序列化功能由 CREATOR 来完成，其内部标明了如何创建序列化对象和数组；内容描述功能由 describeContents 来完成，几乎在所有情况下这个方法都应该返回0，仅当当前对象中存在文件描述符时，此方法返回1。

### Serializable 与 Parcelable 的比较

Serializable 是 Java 中的序列化接口，使用简单但是开销很大，序列化和反序列化过程需要大量的 I/O 操作。而 Parcelable 是 Android 平台中的序列化接口，更适合于在 Android 中使用，它的效率很高，缺点是使用麻烦（但是有了 Android Studio 的 Alt + Enter 一点都不麻烦）。因此在 Android 中，我们首选 Parcelable。Parcelable 主要用在内存序列化中，Serializable 主要用于数据持久化和在网络中传输。有个谷歌的开发者 [Philippe Breault][1]对比了两者间的性能，Parcelable 与 Serializable 的性能竟然相差了十倍，下图是他的结论：

![此处输入图片的描述][2]

### 后记

在笔者的使用过程中，出现了一个非常低级的错误，我从程序 A 通过 Intent 发送序列化对象 MyObject 到程序 B，在这个过程中，我只是简单的把类 MyObject 从程序 A 复制到了程序 B 中，这时一直报 ClassNotFound 的错误，可以类已经复制过来了啊，怎么还是 ClassNotFound 呢？后来经同事提醒，两个类的包名需要一致。嗯，是这样的，之前一直是在同一个程序中使用序列化，所以包名肯定是一致的，写这篇文章算是自己对这个问题的总结，也加深一下对序列化的理解。

### 参考资料

 1. http://www.developerphil.com/parcelable-vs-serializable/
 2. https://developer.android.com/reference/android/os/Parcel.html
 3. https://developer.android.com/reference/java/io/Serializable.html
 4. [Java 编程思想][3]
 5. [Android 开发艺术探索][4]


  [1]: http://www.developerphil.com/parcelable-vs-serializable/
  [2]: http://odc50o546.bkt.clouddn.com/android-serializable-and-parcelable.png
  [3]: https://item.jd.com/10058164.html
  [4]: https://item.jd.com/11760209.html