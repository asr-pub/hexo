---
title: ScaleType 浅析
date: 2016-09-11
tags:
  - Android
---
### ScaleType 的概念
ScaleType 是 ImageView 中的属性，ScaleType 有众多的属性，它也是 ImageView 中 background 与 src 的一个重要区别，因为 background 是不支持 ScaleType 的，background 只是简单的把图片缩放至 ImageView 的大小，相比而言 src 的功能则要丰富的多。说了那么多，那么什么是 ScaleType 呢？

>ScaleType options are used for scaling the bounds of an image to the bounds of the image view.

ScaleType 的作用决定图片在 ImageView 是如何显示的。其实看完这句话的时候，对于 ScaleType 的概念还的懵懂的，下面我们结合实例来讲讲。
<!--more-->
### SacleType的取值
下面是 ScaleType 在 Android 应用中的8种取值
>CENTER : Center the image in the view, but perform no scaling. 
>
>CENTER_CROP : Scale the image uniformly (maintain the image's aspect ratio) so that both dimensions (width and height) of the image will be equal to or larger than the corresponding dimension of the view (minus padding). 
>
>CENTER_INSIDE : Scale the image uniformly (maintain the image's aspect ratio) so that both dimensions (width and height) of the image will be equal to or less than the corresponding dimension of the view (minus padding). 
>
>FIT_CENTER : Scale the image using CENTER. 
>
>FIT_END : Scale the image using END. 
>
>FIT_START : Scale the image using START. 
>
>FIT_XY : Scale the image using FILL. 
>
>MATRIX : Scale using the image matrix when drawing. 



### DEMO
![Demo中的大图](/images/dog.png)

![@Demo中的小图|center|100*0](/images/pig.jpg)


- CENTER : 把图片放在 ImageView 的中心，但是并不对图片进行缩放，所以当图片大于 ImageView 的尺寸时，便会对原始图片进行，比如说我们案例中 dog 这张图片就是比 ImageView 大。
``` xml
android:scaleType="center"
```
![@小图和大图设置 center 属性的效果|center](/images/1473578738218.png)



- CENTER_CROP : 保持图像原始比例，使图片的宽高大于等于 ImageView 的宽高。这个属性在 App 开发中也蛮有用的，比如说我们要做一个高斯模糊的效果，我们一般是将一个图片缩小进行模糊，然后再将模糊后的图片使用 centerCrop 放进相应的 ImageView 中，这样模糊的图片就能充满整个屏幕。
``` xml
android:scaleType="centerCrop"
```

![Alt text|center](/images/1473579243120.png)


- CENTER_INSIDE : 保持图像原始比例，使图片的宽高小于等于 ImageView 的宽高。
``` xml
android:scaleType="centerInside"
```
![Alt text|center](/images/1473579398815.png)  

- FIT_CENTER : ImageView 的默认属性，使得图片能够完全显示在 ImageView 中，大图等比例缩小，小图等比例放大
``` xml
android:scaleType="fitCenter"
```
![Alt text|center](/images/1473579695369.png)

- FIT_END : 与 FIT_CENTEN 显示效果一样，只是将图片显示在右方或下方，而不是居中
``` xml
android:scaleType="fitEnd"
```
![Alt text](/images/1473581101136.png)

- FIT_START : 与 FIT_CENTEN 显示效果一样，只是将图片显示在左方或上方，而不是居中
``` xml
android:scaleType="fitStart"
```
![Alt text](/images/1473580770811.png)

- FIT_XY : 这个最简单了，只是让图片充满整个 ImageView 而已
``` xml
android:scaleType="fitXY"
```
![Alt text](/images/1473580841847.png)

- MATRIX :  从 ImageView 的左上角绘制 bitmap，不对 bitmap 进行任何缩放
``` xml
android:scaleType="matrix"
```
![Alt text|center](/images/1473588241309.png)
### 源码分析
``` java
else if (ScaleType.CENTER == mScaleType) {
    // Center bitmap in view, no scaling.
    mDrawMatrix = mMatrix;
    mDrawMatrix.setTranslate(Math.round((vwidth - dwidth) * 0.5f),
                             Math.round((vheight - dheight) * 0.5f));
}
```
如果开发者没有调用 `setImageMatrix()`，那么 `mDrawMatrix = mMatrix`  中的 `mMatrix`  默认不作任何变换。其中`vwidth`和`vheight`分别是 ImageView 的宽高，`dwidth`和`dheight`分别是 bitmap 的宽高 ，所以我们得知， center 变换只是把 bitmap 移到 ImageView 中心点而已，正如源码中所注释的 Center bitmap in view, no scaling。
``` java
else if (ScaleType.CENTER_CROP == mScaleType) {
    mDrawMatrix = mMatrix;

    float scale;
    float dx = 0, dy = 0;

    if (dwidth * vheight > vwidth * dheight) {
        scale = (float) vheight / (float) dheight; 
        dx = (vwidth - dwidth * scale) * 0.5f;
    } else {
        scale = (float) vwidth / (float) dwidth;
        dy = (vheight - dheight * scale) * 0.5f;
    }

    mDrawMatrix.setScale(scale, scale);
    mDrawMatrix.postTranslate(Math.round(dx), Math.round(dy));
}
```
如果 bitmap 的尺寸大于或者小于 ImageView 的尺寸时，执行缩放和平移操作，具体是怎么操作的，大家可以研究一下
有一种特殊情况，那就是 bitmap 和 ImageView 中的尺寸是完全一样的，那么就什么都不做。

``` java
boolean fits = (dwidth < 0 || vwidth == dwidth) &&
                       (dheight < 0 || vheight == dheight);
else if (fits) {
	// The bitmap fits exactly, no transform needed.
	mDrawMatrix = null;
}
```





