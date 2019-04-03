---
layout:     post
title:      Android Animation
subtitle:   Android动画和热门动画框架介绍
date:       2019-03-15
author:     Blank Ting
header-img: img/in-post/post-bg-android.jpg
catalog: 	 true
tags:
    - Android
    - APP
    - Animation
---

# Android动画

## 动画分类

- Frame Animation（逐帧动画）	       res/drawable
- View Animation（视图动画）	       res/anim
- Property Animation（属性动画）    res/animator

### Frame Animation

原理就是将一张张单独的图片连贯的进行播放，从而在视觉上产生一种动画的效果；有点类似于gif图的效果,更多的依赖于完善的UI资源,想要效果更顺畅,平滑,需要更多的图片资源,每个图片显示时间更短。

缺点：比较占用内存,比较消耗系统性能;

创建：res/drawable/bullet_anim.xml

引用：继承自Drawable,引用类似drawable,布局应用android:background="@drawable/bullet_anim" 
           AnimationDrawable   background = (AnimationDrawable) tv_start.getBackground();

启动：background.start();

注意事项: AnimationDrawable播放动画是依附在window上面的，而在Activity onCreate()中调用时Window还未初始化完毕，帧动画会停留在第一帧，要想实现播放必须在onCreate() 以后,比如onWindowFocusChanged()中;

### View Animation

又叫补间动画(tweened animation),是一种作用于View对象的动画，它支持4种动画效果，分别为平移（translate）、旋转（rotate）、缩放（scale）和透明度（alpha）。这四种动画分别对应于Android中的TranslateAnimation、RotateAnimation、ScaleAnimation和AlphaAnimation这四个类。

缺陷：不具备交互性，View动画作用的实际上是View的影像，而非真正改变了View的属性状态。也就是说，View动画结束后，即便使用setFillAfter（true）使得view保持在动画结束时的位置，view的真实位置依旧未发生变化，仍然处于最开始定义时的位置。因此，当view动画结束后，其响应位置仍然位于动画开始前的位置，这就使得其不具备交互性； View动画只能作用于View对象，且提供的动画种类有限；

使用技巧：在某些特殊情况,比如短信,进入多选模式,可以设置checkbox原位置定在需要交互位置(屏幕内),但是创建动画时,先把checkbox放到屏幕外,再动画平移到原位置,从而达到不影响交互的效果;

``` java
new TranslateAnimation(Animation.RELATIVE_TO_SELF, 0f,Animation.RELATIVE_TO_SELF, 1.0f, Animation.RELATIVE_TO_SELF, 0f, Animation.RELATIVE_TO_SELF, 0f);
```



### Property Animation

属性动画是API 11 新加入的特性,和View 动画不同,属性动画可以对任何对象做动画,甚至还可以没有对象;最为重要的是,打破了View动画的局限,真正改变属性,不存在交互局限;除此之外,属性动画的效果更丰富,不再像View动画只支持四种简单变换,可以对任意属性做动画(主要使用ObjectAnimator),只要满足两个条件:

- Object必须要有setXxx()方法,如果没有初始值,还要提供getXxx()方法,因为系统要获取Xxx属性的初始值;
  如果此条不满足,程序会直接Crash;
- (2)Object的setXxx()对属性Xxx所做的改变必须通过某种方法反映出来,比如UI改变之类的,否则看不到动画效果;
  Xml使用用:res/animator      AnimatorInflater.loadAnimator()生成动画对象;

#### 属性动画的使用

##### ObjectAnimator

最常用,动画原理是不停的调用setXXX方法更新属性值,一般对View的已知属性做动画,比如,alpha, translationX, translationY,x,y等等; 通过ofInt、ofFloat、ofObject这个三个常用的方法返回返回一个ObjectAnimator对象,传的参数包括一个对象和对象的属性名字以及变化值,例如:ObjectAnimator.ofFloat(view_start, View.TRANSLATION_X, 0, 300, 0);

管理多个属性使用PropertyValuesHolder,类似动画集合的效果:

``` java
PropertyValuesHolder pvhX = PropertyValuesHolder.ofFloat("alpha", 1f,  0f, 1f);

PropertyValuesHolder pvhY = PropertyValuesHolder.ofFloat("scaleX", 1f,  0, 1f);

PropertyValuesHolder pvhZ = PropertyValuesHolder.ofFloat("scaleY", 1f,  0, 1f);

ObjectAnimator.ofPropertyValuesHolder(view, pvhX, pvhY, pvhZ) .setDuration(1000).start();
```

##### ValueAnimator

本身不提供任何动画效果,更像一个数值发生器,在设定的时间内对设定区域的值进行变化,从而让调用者根据值控制动画实现,常用于自定义动画,例如:

``` java
vAnimator = ValueAnimator.ofFloat(2, 0).setDuration(1000);  
vAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
    @Override
    public void onAnimationUpdate(ValueAnimator animation) {
        float value = (float) animation.getAnimatedValue();
        view_start.setScaleX(value);
        view_start.setScaleY(value);
        view_start.setAlpha(value);
    }});
```

##### ViewPropertyAnimator

API11中提供的一个快捷高效的动画使用方式

``` java
/// 链式编程,多种动画同时执行
view_start.animate()
        .alpha(0.4f)
        .rotation(180)
        .scaleY(1.5f)
        .scaleX(1.5f)
        .translationX(100)
        .translationY(300)
        .setStartDelay(1000)
        .setDuration(2000);
```

AnimatorSet：动画集合,对多种属性动画的播放顺序精准的控制;
常用方法：playTogether()   ///同时执行  playSequentially(Animator... items) ///依次执行,一个接一个。

``` java
animatorSet.play(animator1)     //开始动画1
        .with(animator2)		//和动画2一起;
        .after(animator3)		//在动画3之后;
        .before(animator4);		//在动画4之前;
animatorSet.start();			//3>>1和2>>4
```



##### 属性动画的监听器
插值器：属性动画 和 View动画都可以使用, 直接setInterpolator();

 [Android中的动画插值器Interpolator：源码及图解](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0110/2292.html)

[Interpolator](https://my.oschina.net/banxi/blog/135633)

特别的动画：

布局动画,在ViewGroup指定 ,子view动画形式出现

LayoutAnimation  和View动画同时设计,不具备交互性;

android:layoutAnimation="@anim/my_layoutanimation“ 界面打开时,view创建的出场动画

android:animateLayoutChanges=“true“    子view的添加删除,出现消失,默认渐变和平移动画,不能作用于子view的子view。

##### LayoutTransition

 API Level 11 才出现的。LayoutTransition的动画效果，只有当ViewGroup中有View添加、删除、隐藏、显示的时候才会体现出来。

- LayoutTransition.APPEARING：当View出现或者添加的时候View出现的动画
- LayoutTransition.CHANGE_APPEARING：当添加View导致布局容器改变的时候整个布局容器的动画。  
- LayoutTransition.DISAPPEARING：当View消失或者隐藏的时候View消失的动画。 
- LayoutTransition.CHANGE_DISAPPEARING：当删除或者隐藏View导致布局容器改变的时候整个布局容器的动画。  
- LayoutTransition.CHANGE：当不是由于View出现或消失造成对其他View位置造成改变的时候整个布局容器的动画。

## 热门动画框架

### [NineOldAndroids  ](https://github.com/JakeWharton/NineOldAndroids)

由于属性动画3.0才出现,使用用它来兼容3.0以前的版本,3.0以前做View动画,以后做属性动画;使用上跟属性动画一致,方法名类名相同;但随着Android发展,4.0以下市场占有率不足1%,没有市场前景,基本已经过时,但在很多早期的框架还在使用,比如ListViewAnimations;

### [lottie-android](https://github.com/airbnb/lottie-android)

通过安装AE上的bodymovin的插件，能够将AE中的动画工程文件转换为通用的json格式描述文件(bodymovin插件本身是用于网页上呈现各种AE效果的一个开源库),lottie所做的事情就是实现在不同移动端平台上呈现AE动画的方式，从而达到动画文件的一次绘制、一次转换，随处可用的效果，这个跟java一次编译随处运行效果一样;

### [ListViewAnimations](https://github.com/nhaarman/ListViewAnimations)

去年宣布不维护更新了;让位给Recyclerview;

lib-core：库的核心，并包含外观动画。

lib-manipulation：包含项目操作选项，如“滑动到关闭”和“拖放”。

lib-core-slh：lib-core的扩展以支持StickyListHeaders。

### [recyclerview-animators](https://github.com/wasabeef/recyclerview-animators)

两种动画:

- Item的入场动画
- Item 添加删除动画