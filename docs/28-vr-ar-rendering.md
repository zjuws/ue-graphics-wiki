# VR/AR渲染

## VR/AR渲染概述

VR/AR渲染需要特殊的考虑，包括立体渲染、高帧率、低延迟等。

## VR渲染基础

### 立体渲染

**原理**: 为每只眼睛渲染一个视角

```cpp
// 立体渲染配置
r.StereoRenderingMethod = 1;  // 0=单通道, 1=双通道

// 眼睛位置
FVector LeftEyePos = CameraPos + LeftEyeOffset;
FVector RightEyePos = CameraPos + RightEyeOffset;

// 为每只眼睛渲染
RenderEye(LeftEyePos, LeftEyeViewMatrix, LeftEyeProjection);
RenderEye(RightEyePos, RightEyeViewMatrix, RightEyeProjection);
```

### 单通道立体渲染

```hlsl
// 单通道立体渲染着色器
void MainVS(
    float4 Position : POSITION,
    uint InstanceID : SV_InstanceID,
    out float4 OutPosition : SV_POSITION)
{
    // 根据实例ID选择眼睛
    float4x4 ViewMatrix = (InstanceID == 0) ? LeftViewMatrix : RightViewMatrix;
    float4x4 ProjMatrix = (InstanceID == 0) ? LeftProjection : RightProjection;
    
    OutPosition = mul(mul(Position, ViewMatrix), ProjMatrix);
}
```

## VR性能要求

### 目标规格

| 规格 | 目标 | 说明 |
|------|------|------|
| FPS | 90 | 防止眩晕 |
| GPU时间 | < 11ms | 90fps |
| CPU时间 | < 11ms | 90fps |
| 延迟 | < 20ms | 运动到光子 |

### 性能优化

#### 降低分辨率

```cpp
// VR可以使用较低的分辨率
r.ScreenPercentage = 80;  // 80%分辨率

// 推荐设置
// Oculus Quest: 70-80%
// PC VR: 80-90%
```

#### 禁用昂贵效果

```cpp
// 禁用或降低效果质量
r.ScreenSpaceReflectionIntensity = 0.0;  // 禁用SSR
r.DynamicGlobalIlluminationIntensity = 0.0;  // 禁用动态GI
r.DepthOfFieldQuality = 0;  // 禁用景深
r.MotionBlurQuality = 0;  // 禁用运动模糊
```

#### 使用前向渲染

```cpp
// VR推荐使用前向渲染
r.ForwardRenderer = 1;
```

## AR渲染

### 相机背景

```cpp
// AR相机设置
ARCamera->bUseCameraBackground = true;
ARCamera->BackgroundTexture = CameraFeed;
```

### 光照估计

```cpp
// AR光照估计
float LightIntensity = ARSession->GetLightIntensity();
FLinearColor LightColor = ARSession->GetLightColor();

DirectionalLight->SetIntensity(LightIntensity);
DirectionalLight->SetLightColor(LightColor);
```

### 平面检测

```cpp
// AR平面检测
TArray<FARPlane> Planes = ARSession->GetDetectedPlanes();

for (const FARPlane& Plane : Planes) {
    // 可视化平面
    DrawDebugPlane(Plane.Center, Plane.Extent, Plane.Rotation);
}
```

## VR特效

### 注视点渲染

```hlsl
// 注视点渲染（中心高分辨率，边缘低分辨率）
float2 FoveatedRendering(float2 UV, float2 CenterUV)
{
    float Distance = length(UV - CenterUV);
    
    float ResolutionScale = lerp(1.0, 0.5, smoothstep(0.0, 0.5, Distance));
    
    return UV;
}
```

### 时间扭曲

```cpp
// 异步时间扭曲
r.AllowAsyncTimeWarp = 1;

// 时间扭曲补偿延迟
// 通过重投影减少感知延迟
```

### 运动模糊

```hlsl
// VR中的运动模糊（减少闪烁）
float3 VRMotionBlur(float3 CurrentColor, float3 PreviousColor, float2 Velocity)
{
    float BlurAmount = length(Velocity);
    return lerp(CurrentColor, PreviousColor, BlurAmount * 0.5);
}
```

## 交互和UI

### VR UI

```cpp
// VR UI渲染
// 使用世界空间UI
UWidgetComponent* VRWidget = CreateWidgetComponent();
VRWidget->SetWidgetSpace(EWidgetSpace::World);
VRWidget->SetDrawSize(FVector2D(500, 500));
```

### 手部追踪

```cpp
// 手部追踪
FVector HandPosition = GetHandPosition();
FRotator HandRotation = GetHandRotation();

// 渲染手部模型
HandMesh->SetWorldLocationAndRotation(HandPosition, HandRotation);
```

## 优化技术

### 实例化

```cpp
// VR中使用实例化减少绘制调用
UInstancedStaticMeshComponent* InstancedMesh = 
    NewObject<UInstancedStaticMeshComponent>();

for (int i = 0; i < NumInstances; ++i) {
    InstancedMesh->AddInstance(Transforms[i]);
}
```

### 剔除优化

```cpp
// 视锥体剔除
r.FrustumCulling = 1;

// 遮挡剔除
r.OcclusionCulling = 1;

// 距离剔除
r.DistanceCull = 1;
```

## 常见问题

### 问题1: VR眩晕

**症状**: 用户感到恶心或头晕

**原因**: 
- 帧率不稳定
- 延迟过高
- 运动不适

**解决**:
```cpp
// 稳定帧率
r.VSync = 1;
r.MaxFPS = 90;

// 降低延迟
r.AllowAsyncTimeWarp = 1;

// 减少运动
// 提供移动选项
```

### 问题2: 性能不足

**症状**: 无法达到90fps

**原因**: 
- 场景过于复杂
- 效果过于昂贵

**解决**:
```cpp
// 降低分辨率
r.ScreenPercentage = 70;

// 简化场景
// 减少绘制调用

// 禁用效果
r.ScreenSpaceReflectionIntensity = 0.0;
```

### 问题3: 延迟高

**症状**: 动作和视觉不同步

**原因**: 
- 渲染延迟
- 传感器延迟

**解决**:
```cpp
// 启用时间扭曲
r.AllowAsyncTimeWarp = 1;

// 优化渲染
r.ForwardRenderer = 1;

// 简化场景
```

## 最佳实践

1. **保持90fps** - 关键性能目标
2. **低延迟** - 使用时间扭曲
3. **简化场景** - 减少复杂度
4. **优化UI** - 世界空间UI
5. **测试** - 在目标设备上测试
6. **舒适度** - 提供移动选项

## 平台特定

### Oculus

```cpp
// Oculus特定设置
r.Oculus.EnableDash = 1;
r.Oculus.EnablePassthrough = 0;
```

### SteamVR

```cpp
// SteamVR特定设置
r.SteamVR.EnableHome = 0;
r.SteamVR.ReprojectionMode = 1;
```

### ARKit/ARCore

```cpp
// ARKit/ARCore特定设置
r.AR.EnablePlaneDetection = 1;
r.AR.EnableLightEstimation = 1;
```

## 参考资源

- [UE VR文档](https://docs.unrealengine.com/en-US/SharingAndReleasing/XRDevelopment/VR/)
- [UE AR文档](https://docs.unrealengine.com/en-US/SharingAndReleasing/XRDevelopment/AR/)
- [性能优化](https://docs.unrealengine.com/en-US/SharingAndReleasing/XRDevelopment/VR/DevelopVR/Performance/)
