
title: Android 时间同步原理分析
date: 2016-09-26
tags:
  - Android
---

### 前言
在 Android 手机中，我们打开设置可以看到自动确定时间和时区的功能，有时候我故意把手机网络关闭，但时间和时区的设置依然有效，总能把一个错误的时间或时区设置成当前正确的时间，这到底是为什么呢？看完这篇文章，相信你能找到答案。

### 结论
在分析之前，先把结论说了吧。Android 时间同步有两种方式，分别是从**运营商**和**网络**获取时间，其中运营商提供时间和时区，用的是 NITZ 协议，网络只能提供时间，用的是 SNTP 协议（NTP 协议的简单版本）。所以，我们的手机会通过营运商和网络两种手段来获取时间，从我的分析来看，如果手机已经从运营商获取时间了，那么就不会从网络获取时间了。
> NITZ, or Network Identity and Time Zone, is a mechanism for provisioning local time and date, time zone and DST offset, as well as network provider identity information, to mobile devices via a wireless network.

> NTP, Network Time Protocol is a networking protocol for clock synchronization between computer systems over packet-switched, variable-latency data networks.

> SNTP, A less complex implementation of NTP, using the same protocol but without requiring the storage of state over extended periods of time, is known as the Simple Network Time Protocol.

<!--more-->

### 结论验证视频
![从网络获取时间 | center](http://odc50o546.bkt.clouddn.com/c.gif)
*未插 sim 卡，从网络获取时间，验证无法改变时区，但能准确的改变时间*

![从运营商获取时间与时区 | center](http://odc50o546.bkt.clouddn.com/bb.gif)
*插入 sim 卡，但未连接网络，验证从运营商能获取时间和时区*

### 源码分析

#### Android 设置入口
[DateTimeSettings.java](https://android.googlesource.com/platform/packages/apps/Settings/+/master/src/com/android/settings/DateTimeSettings.java)
```java
public boolean onPreferenceChange(Preference preference, Object newValue) {
    if (preference.getKey().equals(KEY_AUTO_TIME)) {
        boolean autoEnabled = (Boolean) newValue;
        Settings.Global.putInt(getContentResolver(), Settings.Global.AUTO_TIME,
                autoEnabled ? 1 : 0);
        mTimePref.setEnabled(!autoEnabled);
        mDatePref.setEnabled(!autoEnabled);
    } else if (preference.getKey().equals(KEY_AUTO_TIME_ZONE)) {
        boolean autoZoneEnabled = (Boolean) newValue;
        Settings.Global.putInt(
                getContentResolver(), Settings.Global.AUTO_TIME_ZONE, autoZoneEnabled ? 1 : 0);
        mTimeZone.setEnabled(!autoZoneEnabled);
    }
    return true;
}
```
当我们选中自动更新时间，就会向`Settings.Global.AUTO_TIME`写入 1，此刻，ContentProvider中的值变化了，如果在其他地方对此处的内容变化有监听的话，势必会执行其他地方相应的函数。

#### 将运营商时间应用于手机中
[GsmServiceStateTracker.java](https://android.googlesource.com/platform/frameworks/opt/telephony/+/jb-mr2-dev/src/java/com/android/internal/telephony/gsm/GsmServiceStateTracker.java)
```java
private ContentObserver mAutoTimeObserver = new ContentObserver(new Handler()) {
    @Override
    public void onChange(boolean selfChange) {
        Rlog.i("GsmServiceStateTracker", "Auto time state changed");
        revertToNitzTime();
    }
};
private ContentObserver mAutoTimeZoneObserver = new ContentObserver(new Handler()) {
    @Override
    public void onChange(boolean selfChange) {
        Rlog.i("GsmServiceStateTracker", "Auto time zone state changed");
        revertToNitzTimeZone();
    }
};
mCr.registerContentObserver(
        Settings.Global.getUriFor(Settings.Global.AUTO_TIME), true,
        mAutoTimeObserver);
mCr.registerContentObserver(
        Settings.Global.getUriFor(Settings.Global.AUTO_TIME_ZONE), true,
        mAutoTimeZoneObserver);
```
上面的代码是监听设置入口内容变化的，我们可以看到内容变化后执行的操作是`revertToNitzTime()`和`revertToNitzTimeZone()`, 这两个函数类似，我们这里只分析前者。
```java
private void revertToNitzTime() {
    ......
    if (mSavedTime != 0 && mSavedAtTime != 0) {
        setAndBroadcastNetworkSetTime(mSavedTime
                + (SystemClock.elapsedRealtime() - mSavedAtTime));
    }
}
```
`revertToNitzTime()`执行的是`setAndBroadcastNetworkSetTime`，这里我们先不考虑`mSavedTime`和`mSavedAtTime`，我们把注意力转向`setAndBroadcastNetworkSetTime`。
```java
private void setAndBroadcastNetworkSetTime(long time) {
    if (DBG) log("setAndBroadcastNetworkSetTime: time=" + time + "ms");
    SystemClock.setCurrentTimeMillis(time);
    Intent intent = new Intent(TelephonyIntents.ACTION_NETWORK_SET_TIME);
    intent.addFlags(Intent.FLAG_RECEIVER_REPLACE_PENDING);
    intent.putExtra("time", time);
    mPhone.getContext().sendStickyBroadcastAsUser(intent, UserHandle.ALL);
}
```
该函数做了两件事情，第一件是设置了系统的时间，第二件是发送了一个广播。
现在我来解释一下`setAndBroadcastNetworkSetTime(mSavedTime + (SystemClock.elapsedRealtime() - mSavedAtTime))`中参数的含义。`mSavedTime`是从运营商获取的时间，`mSavedAtTime`表示从运营商获取时间的时候，手机从启动到现在所经过时间，值类型是`SystemClock.elapsedRealtime()`。通过下图，我们很清楚的知道`mSavedTime + (SystemClock.elapsedRealtime() - mSavedAtTime)`就是手机当前的时间。
![各个时间参数的示意图 | center](http://odc50o546.bkt.clouddn.com/demo.png)
#### 广播发往何处？
在上一步中，`setAndBroadcastNetworkSetTime`发出了一个`TelephonyIntents.ACTION_NETWORK_SET_TIME`的广播，从字面上理解，广播的意义就是通过网络设置系统。通过 Google 我们可以找到 [NetworkTimeUpdateService.java](https://android.googlesource.com/platform/frameworks/base.git/+/android-4.2.2_r1/services/java/com/android/server/NetworkTimeUpdateService.java) 文件。
```java
private void registerForTelephonyIntents() {
    IntentFilter intentFilter = new IntentFilter();
    intentFilter.addAction(TelephonyIntents.ACTION_NETWORK_SET_TIME);
    intentFilter.addAction(TelephonyIntents.ACTION_NETWORK_SET_TIMEZONE);
    mContext.registerReceiver(mNitzReceiver, intentFilter);
}
private BroadcastReceiver mNitzReceiver = new BroadcastReceiver() {
    @Override
    public void onReceive(Context context, Intent intent) {
        String action = intent.getAction();
        if (TelephonyIntents.ACTION_NETWORK_SET_TIME.equals(action)) {
            mNitzTimeSetTime = SystemClock.elapsedRealtime();
        } else if (TelephonyIntents.ACTION_NETWORK_SET_TIMEZONE.equals(action)) {
            mNitzZoneSetTime = SystemClock.elapsedRealtime();
        }
    }
};
```
我们可以看到，收到广播之后的操作不过是对一个成员变量进行赋值而已，其他并没有变化，那么我们在找找源码有没有其他广播接收器或者相关代码。
```java
/** Observer to watch for changes to the AUTO_TIME setting */
private static class SettingsObserver extends ContentObserver {
    ......
    void observe(Context context) {
        ContentResolver resolver = context.getContentResolver();
        resolver.registerContentObserver(Settings.Global.getUriFor(Settings.Global.AUTO_TIME),
                false, this);
    }
    ......
}
```
从代码中我们看到，此处对`Settings.Global.AUTO_TIME`内容变化也有监听，我们继续找代码。
```java
public void handleMessage(Message msg) {
    switch (msg.what) {
        case EVENT_AUTO_TIME_CHANGED:
        case EVENT_POLL_NETWORK_TIME:
        case EVENT_NETWORK_CONNECTED:
            onPollNetworkTime(msg.what);
            break;
    }
}
```
当`Settings.Global.AUTO_TIME`内容变化时，发送`EVENT_AUTO_TIME_CHANGED`给`handler`处理，最终的执行的函数是`onPollNetworkTime()`，好的，我们继续。
```java
private void onPollNetworkTime(int event) {
    // If Automatic time is not set, don't bother.
    if (!isAutomaticTimeRequested()) return;
    final long refTime = SystemClock.elapsedRealtime();
    // If NITZ time was received less than POLLING_INTERVAL_MS time ago,
    // no need to sync to NTP.
    if (mNitzTimeSetTime != NOT_SET && refTime - mNitzTimeSetTime < POLLING_INTERVAL_MS) {
        resetAlarm(POLLING_INTERVAL_MS);
        return; 
    }
    final long currentTime = System.currentTimeMillis();
    if (DBG) Log.d(TAG, "System time = " + currentTime);
    // Get the NTP time
    if (mLastNtpFetchTime == NOT_SET || refTime >= mLastNtpFetchTime + POLLING_INTERVAL_MS
            || event == EVENT_AUTO_TIME_CHANGED) {
        if (DBG) Log.d(TAG, "Before Ntp fetch");
        // force refresh NTP cache when outdated
        if (mTime.getCacheAge() >= POLLING_INTERVAL_MS) {
            mTime.forceRefresh();
        }
        // only update when NTP time is fresh
        if (mTime.getCacheAge() < POLLING_INTERVAL_MS) {
            final long ntp = mTime.currentTimeMillis();
            mTryAgainCounter = 0;
            // If the clock is more than N seconds off or this is the first time it's been
            // fetched since boot, set the current time.
            if (Math.abs(ntp - currentTime) > TIME_ERROR_THRESHOLD_MS
                    || mLastNtpFetchTime == NOT_SET) {
                // Set the system time
                if (DBG && mLastNtpFetchTime == NOT_SET
                        && Math.abs(ntp - currentTime) <= TIME_ERROR_THRESHOLD_MS) {
                    Log.d(TAG, "For initial setup, rtc = " + currentTime);
                }
                if (DBG) Log.d(TAG, "Ntp time to be set = " + ntp);
                // Make sure we don't overflow, since it's going to be converted to an int
                if (ntp / 1000 < Integer.MAX_VALUE) {
                    SystemClock.setCurrentTimeMillis(ntp);
                }
            } else {
                if (DBG) Log.d(TAG, "Ntp time is close enough = " + ntp);
            }
            mLastNtpFetchTime = SystemClock.elapsedRealtime();
        } else {
            // Try again shortly
            mTryAgainCounter++;
            if (mTryAgainCounter <= TRY_AGAIN_TIMES_MAX) {
                resetAlarm(POLLING_INTERVAL_SHORTER_MS);
            } else {
                // Try much later
                mTryAgainCounter = 0;
                resetAlarm(POLLING_INTERVAL_MS);
            }
            return;
        }
    }
    resetAlarm(POLLING_INTERVAL_MS);
}
```
这个函数比较长，那么我来逐句的进行解释一下，PS：该函数的环境是`android-4.2.2_r1`。 
```java
//mNitzTimeSetTime          手机设置运营商时间的时刻
//refTime                   手机当前时刻
//POLLING_INTERVAL_MS       24L * 60 * 60 * 1000; // 24 hrs
//所以在24小时内已经设置了运营商时间了，那么不设置网络时间
final long refTime = SystemClock.elapsedRealtime();
// If NITZ time was received less than POLLING_INTERVAL_MS time ago,
// no need to sync to NTP.
if (mNitzTimeSetTime != NOT_SET && refTime - mNitzTimeSetTime < POLLING_INTERVAL_MS) {
    resetAlarm(POLLING_INTERVAL_MS);
    return; 
}

//如果网络时间的缓存超过了一天，那么重新获取网络时间，具体的怎么获取在介绍完这个函数再谈。
// force refresh NTP cache when outdated
if (mTime.getCacheAge() >= POLLING_INTERVAL_MS) {
    mTime.forceRefresh();
}

// 抽出了 if 语句中的核心代码，ntp 为网络时间缓存中的时间，当 ntp 和本地时间的误差大于 TIME_ERROR_THRESHOLD_MS = 5000 时，设置时间。
if (mTime.getCacheAge() < POLLING_INTERVAL_MS) {
    final long ntp = mTime.currentTimeMillis();
    if (Math.abs(ntp - currentTime) > TIME_ERROR_THRESHOLD_MS
            || mLastNtpFetchTime == NOT_SET) {
        if (ntp / 1000 < Integer.MAX_VALUE) {
            SystemClock.setCurrentTimeMillis(ntp);
        }
    } 

//网络时间更新失败，进行定时重新更新
else {
// Try again shortly
mTryAgainCounter++;
if (mTryAgainCounter <= TRY_AGAIN_TIMES_MAX) {
    resetAlarm(POLLING_INTERVAL_SHORTER_MS);
} else {
    // Try much later
    mTryAgainCounter = 0;
    resetAlarm(POLLING_INTERVAL_MS);
}
return;
```
在上面我们看到了获取网络时间的关键代码是`mTime.forceRefresh()`，我们找到`mTime`的定义`mTime = NtpTrustedTime.getInstance(context)`，顺藤摸瓜，找`NtpTrustedTime`。我们就直接跳到网络请求的关键代码吧。
```java
final SntpClient client = new SntpClient();
if (client.requestTime(mServer, (int) mTimeout)) {
    mHasCache = true;
    mCachedNtpTime = client.getNtpTime();
    mCachedNtpElapsedRealtime = client.getNtpTimeReference();
    mCachedNtpCertainty = client.getRoundTripTime() / 2;
    return true;
} else {
    return false;
}
```
我们在文章的开头就介绍了`SNTP`，是一种简单的`NTP`协议，误差要比`NTP`大一点，`Android`帮我们封装了`SntpClient`，所以我们要自己获取网络时间的话，就可以直接拿来用了。这边我比较好奇的是`Android`从哪个服务器获取的时间。`defaultServer = res.getString(com.android.internal.R.string.config_ntpServer)`，我们再找对应的字符串[config.xml](https://android.googlesource.com/platform/frameworks/base/+/master/core/res/res/values/config.xml)。
```java
<!-- Remote server that can provide NTP responses. -->
<string translatable="false" name="config_ntpServer">2.android.pool.ntp.org</string>
```
服务器地址是`2.android.pool.ntp.org`，我`ping`了一下，一共有四个服务器可以使用，分别是：
> 0.android.pool.ntp.org
> 1.android.pool.ntp.org
> 2.android.pool.ntp.org
> 3.android.pool.ntp.org

除了这几个服务器之外，在`www.ntp.org`上面，大家还可以找到很多个服务器地址，使用哪个，就自己权衡一下了。

#### 小结
本文只是对时间同步做了一个很表层的分析而已，如果大家对文章哪一部分有疑问，可以给我留言或者自己查看源码，文章中如果有错误的话，也恳请大家指正。

### 参考文献

 1. http://blog.csdn.net/lindir/article/details/7973700
 2. https://android.googlesource.com/platform/packages/apps/Settings/+/master/src/com/android/settings/DateTimeSettings.java
 3. https://android.googlesource.com/platform/frameworks/opt/telephony/+/jb-mr2-dev/src/java/com/android/internal/telephony/gsm/GsmServiceStateTracker.java
 4. https://android.googlesource.com/platform/frameworks/base.git/+/android-4.2.2_r1/services/java/com/android/server/NetworkTimeUpdateService.java
 5. https://android.googlesource.com/platform/frameworks/base/+/android-4.4_r1/core/java/android/util/TrustedTime.java
 6. https://android.googlesource.com/platform/frameworks/base/+/master/core/java/android/util/NtpTrustedTime.java
 7. https://android.googlesource.com/platform/frameworks/base/+/master/core/res/res/values/config.xml
 8. https://android.googlesource.com/platform/frameworks/opt/telephony/+/tools_r22/src/java/com/android/internal/telephony/RIL.java
 9. http://blog.csdn.net/droyon/article/details/52211132

