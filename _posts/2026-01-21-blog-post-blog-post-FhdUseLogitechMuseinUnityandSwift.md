---
title: 'Precision Input in MR: 在Unity/Swift中使用LogitechMuse'
date: 2026-01-21
permalink: /posts/2026/01/blog-post-FhdUseLogitechMuseinUnityandSwift/
tags:
  - Unity
  - Swift
  - Logitech Muse
  - Interaction Design
  - MR
---
Introduction
======
  这是我的第一篇博客，思来想去，虽然是发布在github上，但还是打算用中文写，一方面碍于自己语法不精单词量少，另一方面的确是想先把自己这两年做出来的一些单点突破性质的工作贴出来，所以接下来就用中文来描述正文吧:)  
    
  书归正传，在虚拟手术或者工业设计等专业MR培训场景中，AppleVisionPro(AVP)有着绝佳的视觉效果和清晰度，所以我们也采用AVP作为显示设备。目前AVP支持了一些手柄，其中包括Logitech Muse(Muse)，我买了一支，主要是想使用它的6-DOF位姿和按键、震动等输入输出效果。  
  
  但实验发现，在Unity Build出来的Xcode工程里，不能直接使用Muse的6-DOF位姿，这在[apple官方插件](https://github.com/apple/unityplugins/issues/70)也有说明，而我们恰恰希望能用Muse的位姿去带动一个Unity GameObject的位姿跟随，所以，没有办法，我们作为开发者想去使用Muse只能舍近求远，既然需求是让GameObject跟随位姿，而一般常人使用都是像用笔一样持握Muse，所以我设计了一个使用方案：  
    
1. 利用Unity PolySpatial已有的XRHands手势识别方案，因为手势识别的延迟很低，故可以让GameObject跟随一个手指节点  
2. 因为手指节点无法做到自转(Roll)，但Muse支持悬停UI，获取其在UI上的位置和姿态信息，故可以使用Muse在UIKit上悬停的表现  

Talk is cheap, show me the code!
======



Aren't headings cool?
------
