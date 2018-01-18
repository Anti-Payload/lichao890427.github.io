---
layout: post
title: 安卓布局适应任意分辨率（老帖勿喷）
categories: Android
description: 安卓布局适应任意分辨率
keywords: 
---

&emsp;&emsp;这个问题是我才发现的，上个礼拜有个同学跟我说她有个同事开发android，每个分辨率都要设计一套Layout布局文件，极其麻烦，开始我不以为然，我想android不是自动计算的么，用dip dp这种相对距离就ok了，然而我没想到的是自适应永远有局限性，我有2个手机，写出来的代码在320*240 上有一部分显示不了，而854*480却很适合。因为手机厂商竞争激烈因此出现了很多怪胎级别的屏幕。现在主流分辨率有：
* QVGA = 320 * 240; 
* WQVGA = 320 * 480; 
* WQVGA2 = 400 * 240; 
* WQVGA3 = 432 * 240; 
* HVGA = 480 * 320; 
* VGA = 640 * 480; 
* WVGA = 800 * 480; 
* WVGA2 = 768 * 480; 
* FWVGA = 854 * 480; 
* DVGA = 960 * 640; 
* PAL = 576 * 520; 
* NTSC = 486 * 440; 
* SVGA = 800 * 600; 
* WSVGA = 1024 * 576; 
* XGA = 1024 * 768; 
* XGAPLUS = 1152 * 864; 
* HD720 = 1280 * 720; 
* WXGA = 1280 * 768; 
* WXGA2 = 1280 * 800; 
* WXGA3 = 1280 * 854; 
* SXGA = 1280 * 1024; 
* WXGA4 = 1366 * 768; 

&emsp;&emsp;为此我在网上查了一下，得到的解决方法是：最稳妥的方案还是为每种特殊分辨率设计一套布局。否则你设计的布局很有可能不通用。这个结果令我相当不满意，因为这个比较麻烦。为此我探究了一番，最终创造出来一种前人没有的方法————(我命名为)二分法设计界面。  
&emsp;&emsp;所谓二分法就是对于任意屏幕某个控件的所在区域，通过重复每次将屏幕区域划分为2个部分(可以不均等)，然后在子区域中划分，总能刚好得到这个区域。不过你要能看准。划分的方法就是尽可能通过嵌套线性布局LinearLayout和他的权重属性layout_weight的合理分配，外加空隙调整layout_marginTop layout_marginBottom layout_marginLeft layout_marginRight控制子控件的位置达到原始效果。此外layout_height和layout_weight尽可能取fill_parent，且绝对不能出现数字。这样做的结果就是可以适应任何屏幕大小。不过缺点是eclipse会报layout_weight有可能降低性能(虽然这个性能降低我没感觉到)。不过这样的布局所见即所得，也就是说Graphical Layout怎么显示，手机上就是什么样的。   
&emsp;&emsp;android界面设计我觉得有许多bug，而我的二分法就是为了对抗这种bug，正像计算机采用2进制存储一样，二分法不会有错，当然我说的这种错是由于android自身bug导致的，不过可以通过二分法弥补，如果我采用三分法甚至四分法，则很有可能界面是不正常的。   
&emsp;&emsp;有的时候，控件可能无法正常居中，这个估计也是bug，这时你需要动脑筋，使用LinearLayout+layout_weight进行布局达到居中效果，下面这个例子简单的诠释了二分法设计界面的含义，也解决了这个问题：  
&emsp;&emsp;现在有个ImageView在某行居中，但是设置了居中后没效果，假设该控件相对该行大小为：1.5:1:1.5 那么我们先按照1.5:2.5分，然后把权重2.5的区域按照1:1.5分就可以得到目的区域，代码如下：
```xml
view plaincopy to clipboardprint?
<LinearLayout
android:layout_width="fill_parent"
android:layout_height="fill_parent"
android:layout_weight="5"
android:orientation="horizontal"/>
<LinearLayout
android:layout_width="fill_parent"
android:layout_height="fill_parent"
android:layout_weight="3"
android:orientation="horizontal">
<LinearLayout
android:layout_width="fill_parent"
android:layout_height="fill_parent"
android:layout_weight="3"
android:orientation="horizontal">
<ImageView
android:layout_width="80dip"
android:layout_height="60dip"
android:background="@drawable/mainpage_logo2"
android:layout_gravity="center"
android:gravity="center"
android:contentDescription="@string/imagedescrip"
android:scaleType="fitXY"/>
</LinearLayout>
<LinearLayout
android:layout_width="fill_parent"
android:layout_height="fill_parent"
android:layout_weight="2"
android:orientation="horizontal">
</LinearLayout>
</LinearLayout>
<LinearLayout android:layout_width="fill_parent" android:layout_height="fill_parent" android:layout_weight="5" android:orientation="horizontal"/> <LinearLayout android:layout_width="fill_parent" android:layout_height="fill_parent" android:layout_weight="3" android:orientation="horizontal"> <LinearLayout android:layout_width="fill_parent" android:layout_height="fill_parent" android:layout_weight="3" android:orientation="horizontal"> <ImageView android:layout_width="80dip" android:layout_height="60dip" android:background="@drawable/mainpage_logo2" android:layout_gravity="center" android:gravity="center" android:contentDescription="@string/imagedescrip" android:scaleType="fitXY"/> </LinearLayout> <LinearLayout android:layout_width="fill_parent" android:layout_height="fill_parent" android:layout_weight="2" android:orientation="horizontal"> </LinearLayout> </LinearLayout> 
```
&emsp;&emsp;对于已经使用正常布局格式写好的xml，改造成二分法布局相当麻烦，因为不是简简单单在外层layout处设置一下layout_weight就好了，如果你设置很可能你会发现界面莫名其妙的被搞坏了，这个极有可能是内部空间尺寸超过layout_weight，同时android处理的也不是很好导致的。一般改造以后布局代码量会扩充1/3以上，不过使用layout_weight以后会带来很多好处，包括可以轻松调整整体布局和单个控件而不是不断尝试哪个控件需要多少dp/dip合适。  
&emsp;&emsp;有些朋友跟我说他们调试都是用手机的，模拟器太慢了，确实不过如果要测试多分辨率，或者多android版本，模拟器是不可或缺的  

