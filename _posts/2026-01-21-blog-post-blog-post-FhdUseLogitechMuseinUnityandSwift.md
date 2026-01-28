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

<p align="center">
  <img src="https://HaldonFu.github.io/images/FHDUseLogitechMuseinUNITYSWIFT.png?raw=true" alt="Photo" style="width: 450px;"/> 
</p>  
  
如图中所示，红色的UI是SwiftUI，用于Muse的悬停，使用的方法是UIHoverGestureRecognizer，在其前方1-3cm处会检测到笔的悬停，并且获取到其在UI上的位姿，其中:  
x:触控笔在红色面板上的水平坐标  
y:触控笔在红色面板上的垂直坐标  
z:深度坐标(这里被编码为0，但如果swiftui设置为体积ui的话，也可以获取到Z)  
alt:笔杆与屏幕平面之间的夹角（仰角）  
azi: 笔杆在屏幕平面上的投影朝向（笔尖指向哪里）  
roll: 笔围绕自身中轴线旋转的角度（就像拧螺丝一样的动作）  
我在图中的log只展示了部分参数，但其实代码中全部发送给了unity。
  

Talk is cheap, show the code!
======
  
那么首先我们应该先实现物体跟随手的方法，这里代码贴的稍微全一些。
```csharp
using UnityEngine;
using UnityEngine.XR.Hands;
using UnityEngine.XR.Management;

public class HandProbeBinder_Smart : MonoBehaviour
{
    [Header("基础绑定")]
    public Handedness handedness = Handedness.Right;
    public XRHandJointID trackJointID = XRHandJointID.Wrist;

    [Header("捏合控制")]
    public bool showOnlyWhenPinching = true;
    public float pinchThreshold = 0.03f;

    [Header("校准偏移")]
    public Vector3 positionOffset;
    public Vector3 rotationOffset;

    //内部变量
    private XRHandSubsystem m_Subsystem;
    private Renderer[] m_Renderers;

    void Start()
    {
        m_Renderers = GetComponentsInChildren<Renderer>();
        GetHandSubsystem();
    }

    void Update()
    {
        
        if (m_Subsystem == null || !m_Subsystem.running) { GetHandSubsystem(); return; }
        XRHand hand = (handedness == Handedness.Left) ? m_Subsystem.leftHand : m_Subsystem.rightHand;
        if (!hand.isTracked) { SetVisibility(false); return; }

        XRHandJoint anchorJoint = hand.GetJoint(trackJointID);
        if (anchorJoint.TryGetPose(out Pose anchorPose))
        {
            //计算手部基础姿态
            Vector3 targetPos = anchorPose.position + (anchorPose.rotation * positionOffset);
            Quaternion handBaseRot = anchorPose.rotation * Quaternion.Euler(rotationOffset);
            //捏合显隐
            if (showOnlyWhenPinching) CheckPinchDistance(hand);
            else SetVisibility(true);
        }
        else SetVisibility(false);
    }
    
    void CheckPinchDistance(XRHand hand)
    {
        var thumbTip = hand.GetJoint(XRHandJointID.ThumbTip);
        var indexTip = hand.GetJoint(XRHandJointID.IndexTip);

        if (thumbTip.TryGetPose(out Pose thumbPose) && indexTip.TryGetPose(out Pose indexPose))
        {
            float distance = Vector3.Distance(thumbPose.position, indexPose.position);
            bool isHolding = distance <= pinchThreshold;
            SetVisibility(isHolding);
        }
        else { SetVisibility(false); }
    }

    void SetVisibility(bool visible)
    {
        if (m_Renderers.Length > 0 && m_Renderers[0].enabled != visible)
        {
            foreach (var r in m_Renderers) r.enabled = visible;
        }
    }

    void GetHandSubsystem()
    {
        var subSystems = new System.Collections.Generic.List<XRHandSubsystem>();
        SubsystemManager.GetSubsystems(subSystems);
        if (subSystems.Count > 0) m_Subsystem = subSystems[0];
    }
}
```
实验发现，这里跟随时有抖动，所以我们必须加入平滑效果，这里采用lerp的方式
```CSharp
  transform.position = Vector3.Lerp(transform.position, targetPos, Time.deltaTime * posSmoothSpeed);
  transform.rotation = Quaternion.Slerp(transform.rotation, finalRot, Time.deltaTime * rotSmoothSpeed);
```
在此刻，我们就发现了，跟随手指的话无法完成自转（roll），所以此刻该利用swift的UIHover来实现自转。
于是我们需要在swift端创建一个ui，用于悬停使用，并定义一个函数，用于发送数据给unity，unity接收数据后再叠加自转。

```swift
import Foundation
import SwiftUI
import UnityFramework
import PolySpatialRealityKit
import UIKit

// 1. 定义一个专门用于调试的 UIView
class MuseDebugView: UIView {
    
    // 回调函数，把数据传回给 SwiftUI 显示
    var onUpdate: ((String) -> Void)?
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        setupGesture()
    }
    
    required init?(coder: NSCoder) {
        super.init(coder: coder)
        setupGesture()
    }
    
    private func setupGesture() {
        let hover = UIHoverGestureRecognizer(target: self, action: #selector(handleHover(_:)))
        
        //允许 笔(pencil) 和 手(direct) 触发，方便测试
        hover.allowedTouchTypes = [
            NSNumber(value: UITouch.TouchType.pencil.rawValue), // Muse
            //NSNumber(value: UITouch.TouchType.direct.rawValue)  // 手指，可以删除
        ]
        
        self.addGestureRecognizer(hover)
        
        //设置一个明显红色
        self.backgroundColor = UIColor.red.withAlphaComponent(0.2)
        self.isUserInteractionEnabled = true
    }
    

        @objc func handleHover(_ gesture: UIHoverGestureRecognizer) {
            
            if gesture.state == .ended || gesture.state == .cancelled || gesture.state == .failed {
                        return
                    }
            //获取原始数据
            let location = gesture.location(in: self)
            let alt = Float(gesture.altitudeAngle)
            let azi = Float(gesture.azimuthAngle(in: self)) //别忘了(in: self)
            
            //let roll: Float = 0.0
            let rawRoll = gesture.rollAngle
            let roll = Float(rawRoll)
            
            let rollDeg = rawRoll * 180 / .pi
            
            //假设窗口大小约 30cm，UI 坐标约 300pt -> 比例约 0.001
            //注意：VisionOS 窗口原点在左上角，Y轴向下；Unity Y轴向上。
            let unityX = Float(location.x) * 0.001
            let unityY = Float(location.y) * 0.001
            
            //构建 Log 字符串 (方便调试)
            let log = "P:(\(String(format: "%.2f", unityX)), \(String(format: "%.2f", unityY))) A:\(String(format: "%.2f", alt)) R:\(String(format: "%.2f", rollDeg))"
            //print("MuseSending: \(log)") //保持 Xcode 端也能看到
            onUpdate?(log) // 更新 Swift UI 上的文字
            
            //发送给 Unity
            //最后一个参数 1.0 代表“正在追踪”
            SendMusePoseToUnity(
                x: unityX,
                y: unityY,
                z: 0,
                alt: alt,
                azi: azi,
                roll: roll,
                status: 1.0
            )
        }
}

//连接 UIView 和 SwiftUI
struct MuseDebugViewWrapper: UIViewRepresentable {
    @Binding var logText: String
    
    func makeUIView(context: Context) -> MuseDebugView {
        let view = MuseDebugView()
        //绑定回调
        view.onUpdate = { newLog in
            //必须在主线程更新 UI
            DispatchQueue.main.async {
                self.logText = newLog
            }
        }
        return view
    }
    
    func updateUIView(_ uiView: MuseDebugView, context: Context) {
        // 不需要反向更新
    }
}

@available(visionOS 1.0, *)
struct MuseTrackerView: View {
    @State private var logText: String = "Waiting for input...\nPlease try hovering and rotating your stylus here"
    
    //@State private var dragOffset: CGSize = .zero
    
    var body: some View {
        VStack(spacing: 15) {
            ZStack {
                //磨砂玻璃 + 阴影
                RoundedRectangle(cornerRadius: 30, style: .continuous)
                    .fill(.regularMaterial)
                    .shadow(color: .black.opacity(0.9), radius: 10, y: 5)
                
                //红色检测区
                MuseDebugViewWrapper(logText: $logText)
                    .opacity(1.0)
                    .clipShape(RoundedRectangle(cornerRadius: 30))
                
                //内容
                VStack(spacing: 12) {
                    Text("Stylus Input Area")
                        .font(.headline)
                        .padding(.top, 25)
                        .foregroundColor(.primary)
                    
                    Divider().padding(.horizontal)
                    
                    Text(logText)
                        .font(.system(size: 13, design: .monospaced))
                        .padding()
                        .background(Color.black.opacity(0.5))
                        .cornerRadius(12)
                        .padding(.horizontal)
                        .frame(maxHeight: .infinity)
                    
                    Spacer()
                }
            }
            .frame(width: 300, height: 300)
        }
    }
}


```
经过以上Swift这一系列操作之后呢，就可以生成一个ui，只需要在Swift调用窗口的位置添加上这个ui即可，无论是attachment还是windowgroup都支持。  
下一步就是我们该如何在unity使用这个自转数据。  
其实很简单，只需要在更新位姿时加入自转姿态的影响即可  
```csharp
        //检测 Muse 数据更新 (判定是“预览”还是“锁定”)
        if (useMuseRoll)
        {
            // 检查MuseDebugSync 是否有新数据
            float currentRawRoll = MuseDebugSync.LatestRollRadians;
            
            //只要数据源在更新，就更新 _savedRollAngle
            _savedRollAngle = currentRawRoll * Mathf.Rad2Deg;
            //处理坐标系反转
            if (invertRoll) _savedRollAngle = -_savedRollAngle;
        }

        //叠加保存好的 Roll 角度
        Quaternion rollRot = Quaternion.Euler(0, 0, _savedRollAngle);
        Quaternion finalRot = handBaseRot * rollRot;
```

写在结尾
------
关于Logitech Muse来控制虚拟工具Roll的方法就写完了，当然这里如果使用attachment的话，还可以用unity来控制苹果原生SwiftUI的位姿，这就留到后面再开一篇帖子来写吧...
