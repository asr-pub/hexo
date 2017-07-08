---
title: 对于字符集的个人总结
date: 2016-11-06
tags: 
  - 编程基础
---

无论是在上学时，还是工作中，无论是 PC 还是移动端，无论是 Windows 还是 Linux，都会遇到乱码的情况，每次遇到该问题都是上网搜索，寻求解决方案。一个问题最好的解决方案，往往看透它的本质，从它的本质来寻找解决方案。下面的内容是我翻阅了很多的资料，总结了几个关键性的问题，弄清了这几个关键性问题，那么关于乱码的问题会有一个清晰的认识。

### 什么是 Unicode

 > Unicode is a computing industry standard for the consistent encoding, representation, and handling of text expressed in most of the world's writing systems.

Unicode 其实就是一个**字符表**，它规定了每个字符的序号，比如说 A 在 Unicode 的序号是 65，表示成十六进制为 0x41。但是 Unicode 并没有规定字符的存储方式，对于 A 来说，用单个字节可以表示为 0x41，两个字节可以表示为 0x00 0x41，所以这就引出了第二个问题。
<!--more-->
### Unicode 和 UTF-8 的联系

UTF（Unicode Transfer Format），**UTF-8 是 Unicode 的实现方式之一**。除了 UTF-8 之外，还有 UTF-16 等，前者以 8 位为编码单位，后者以 16 位作为编码单位。
UTF-8 具体的实现方式如下：

|Unicode 序号范围|UTF-8 存储方式|
|:---|:-|
|0000 0000 - 0000 007F|0xxxxxxx|
|0000 0080 - 0000 07FF|110xxxxx 10xxxxxx|
|0000 0800 - 0000 FFFF|1110xxxx 10xxxxxx 10xxxxxx|
|0001 0000 - 0010 FFFF|11110xxx 10xxxxxx 10xxxxxx 10xxxxxx|

第一个字节是 0 开头的话，那么该字符占一个字节；如果第一个字节有 n 个连续的 1，那么该字符由 n 个字节组成，且后面 n - 1 个字节均由 10 开头。

从上表中，我们可以知道，UTF-8 能表示的字符数为 2^7 + 2^11 + 2^16 + 2^21 = 2,164,992 这个数字对于可预测的将来，应该是够用的，不够用的话还可以继续扩充哈。这里让我想到了哈夫曼编码，最好的字符集设计是用最少的字节来表示使用最频繁的字符。

### UTF-8 和 ISO-8859-1 的联系

UTF-8 是可变长度的字符集，而 ISO-8859-1 是单字节的字符集；UTF-8 可以表示 Unicode 的任何字符，而 ISO-8859-1 只能表示 Unicode 前面的 256 个字符；如果要存储 128 ~ 256 的字符，那么 UTF-8 就需要两个字节。

### 中文字符编码

中文编码字符集按照发布的时间有 GB2312、GBK、GB18030，后者是前者的超集，现在用的最多的是 GBK 字符集。在这里，我把 Unicode 叫做字符表，UTF-8 叫做字符编码，那么 GB2312、GBK、GB18030 既是字符表，又是字符编码。


### 参考资料

 1. http://www.ruanyifeng.com/blog/2007/10/ascii_unicode_and_utf-8.html





