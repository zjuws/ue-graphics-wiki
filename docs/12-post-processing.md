# 后处理效果

## 后处理概述

后处理是在渲染完成后对最终图像进行的处理。UE提供了丰富的后处理效果，可以显著改善视觉质量。

## 后处理管线

```
延迟渲染完成
    ↓
后处理通道1 (色调映射)
    ↓
后处理通道2 (色彩分级)
    ↓
后处理通道3 (景深)
    ↓
后处理通道4 (运动模糊)
    ↓
后处理通道5 (抗锯齿)
    ↓
最终输出
```

## 常用后处理效果

### 1. 色调映射（Tone Mapping）

**目的**: 将HDR颜色映射到LDR显示范围

**原理**:
```hlsl
// 简单的色调映射
float3 ToneMap(float3 Color)
{
    // Reinhard色调映射
    return Color / (Color + 1.0);
}

// 更复杂的色调映射（ACES）
float3 ACESFilm(float3 x)
{
    float a = 2.51;
    float b = 0.03;
    float c = 2.43;
    float d = 0.59;
    float e = 0.14;
    return clamp((x*(a*x+b))/(x*(c*x+d)+e), 0.0, 1.0);
}
```

**配置**:
```cpp
// 在后处理体积中设置
PostProcessVolume->Settings.bOverride_ToneCurveAmount = true;
PostProcessVolume->Settings.ToneCurveAmount = 1.0;
```

### 2. 色彩分级（Color Grading）

**目的**: 调整图像的色彩和对比度

**参数**:
- **饱和度**: 颜色强度
- **对比度**: 亮度范围
- **亮度**: 整体亮度
- **色温**: 冷暖色调

**代码**:
```cpp
// 创建色彩分级LUT
UTexture* ColorGradingLUT = LoadObject<UTexture>(nullptr,
    TEXT("Texture2D'/Game/ColorGrading/CinematicLUT.CinematicLUT'"));

PostProcessVolume->Settings.bOverride_ColorGradingLUT = true;
PostProcessVolume->Settings.ColorGradingLUT = ColorGradingLUT;
PostProcessVolume->Settings.ColorGradingIntensity = 1.0;
```

### 3. 景深（Depth of Field）

**目的**: 模拟摄像机焦点效果

**参数**:
- **焦点距离**: 清晰的距离
- **焦点半径**: 焦点范围
- **模糊大小**: 模糊强度

**实现**:
```hlsl
// 景深计算
float GetDepthOfFieldBlur(float Depth, float FocusDistance, float FocusRadius)
{
    float Distance = abs(Depth - FocusDistance);
    return smoothstep(0, FocusRadius, Distance);
}

// 应用景深
float3 FinalColor = lerp(SharpColor, BlurredColor, DepthOfFieldBlur);
```

**配置**:
```cpp
PostProcessVolume->Settings.bOverride_DepthOfFieldFocalDistance = true;
PostProcessVolume->Settings.DepthOfFieldFocalDistance = 1000.0;
PostProcessVolume->Settings.DepthOfFieldFocalRegion = 500.0;
PostProcessVolume->Settings.DepthOfFieldScale = 1.0;
```

### 4. 运动模糊（Motion Blur）

**目的**: 模拟快速运动时的模糊效果

**原理**:
```hlsl
// 运动模糊
float3 MotionBlur(float3 CurrentColor, float3 PreviousColor, float BlurAmount)
{
    return lerp(CurrentColor, PreviousColor, BlurAmount);
}
```

**配置**:
```cpp
PostProcessVolume->Settings.bOverride_MotionBlurAmount = true;
PostProcessVolume->Settings.MotionBlurAmount = 0.5;
PostProcessVolume->Settings.MotionBlurMax = 5.0;
```

### 5. 抗锯齿（Anti-Aliasing）

**类型**:

#### FXAA（快速近似抗锯齿）
- 廉价的后处理抗锯齿
- 可能导致模糊

#### SMAA（子像素形态学抗锯齿）
- 更好的质量
- 中等成本

#### TAA（时间抗锯齿）
- 最好的质量
- 可能产生鬼影

**配置**:
```cpp
PostProcessVolume->Settings.bOverride_AntiAliasingMethod = true;
PostProcessVolume->Settings.AntiAliasingMethod = AAM_TemporalAA;
```

### 6. 环境光遮挡（Ambient Occlusion）

**目的**: 增强角落和缝隙的阴影

**类型**:

#### SSAO（屏幕空间AO）
- 基于深度
- 实时计算
- 可能产生伪影

#### DFAO（距离场AO）
- 基于距离场
- 质量更好
- 需要预计算

**配置**:
```cpp
PostProcessVolume->Settings.bOverride_AmbientOcclusionIntensity = true;
PostProcessVolume->Settings.AmbientOcclusionIntensity = 1.0;
PostProcessVolume->Settings.AmbientOcclusionRadius = 100.0;
```

### 7. 屏幕空间反射（Screen Space Reflection）

**目的**: 实时计算屏幕空间的反射

**原理**:
```hlsl
// SSR光线追踪
float3 ReflectionRay = reflect(ViewDir, Normal);
float3 ReflectionPos = WorldPos + ReflectionRay * RayDistance;

// 投影到屏幕空间
float2 ScreenCoord = Project(ReflectionPos);

// 采样反射颜色
float3 ReflectionColor = SceneTexture.Sample(ScreenCoord);
```

**配置**:
```cpp
PostProcessVolume->Settings.bOverride_ScreenSpaceReflectionIntensity = true;
PostProcessVolume->Settings.ScreenSpaceReflectionIntensity = 1.0;
PostProcessVolume->Settings.ScreenSpaceReflectionQuality = 50;
```

### 8. 全局光照（Global Illumination）

**目的**: 实时全局光照

**类型**:

#### DFGI（距离场全局光照）
- 基于距离场
- 质量好
- 需要预计算

#### LSGI（Lumen全局光照）
- 实时全局光照
- 高质量
- 性能消耗大

**配置**:
```cpp
PostProcessVolume->Settings.bOverride_DynamicGlobalIlluminationIntensity = true;
PostProcessVolume->Settings.DynamicGlobalIlluminationIntensity = 1.0;
```

## 后处理体积（Post Process Volume）

### 创建和配置

```cpp
// 创建后处理体积
APostProcessVolume* PostProcessVolume = GetWorld()->SpawnActor<APostProcessVolume>();

// 设置为无界（影响整个场景）
PostProcessVolume->bUnbound = true;

// 配置效果
PostProcessVolume->Settings.bOverride_BloomIntensity = true;
PostProcessVolume->Settings.BloomIntensity = 1.0;
```

### 优先级

```cpp
// 设置优先级（高优先级覆盖低优先级）
PostProcessVolume->Priority = 100;
```

### 混合

```cpp
// 在多个体积之间混合
PostProcessVolume->BlendWeight = 0.5;  // 50%混合
```

## 自定义后处理

### 创建自定义后处理材质

```hlsl
// 自定义后处理着色器
void MainPS(
    in float4 UVAndScreenPos : TEXCOORD0,
    out float4 OutColor : SV_Target0)
{
    float2 UV = UVAndScreenPos.xy;
    
    // 采样场景颜色
    float3 SceneColor = SceneTexture.Sample(UV).rgb;
    
    // 应用自定义效果
    float3 Result = ApplyCustomEffect(SceneColor);
    
    OutColor = float4(Result, 1.0);
}
```

### 应用自定义后处理

```cpp
// 创建后处理材质实例
UMaterialInstanceDynamic* PostProcessMaterial = 
    UMaterialInstanceDynamic::Create(BaseMaterial, this);

// 应用到后处理体积
PostProcessVolume->AddOrUpdateBlendable(PostProcessMaterial);
```

## 性能优化

### 后处理成本

| 效果 | 成本 | 说明 |
|------|------|------|
| 色调映射 | 低 | 简单计算 |
| 色彩分级 | 低 | LUT查询 |
| 景深 | 中 | 多次采样 |
| 运动模糊 | 中 | 历史缓冲 |
| SSAO | 高 | 多次采样 |
| SSR | 高 | 光线追踪 |
| DFGI | 高 | 复杂计算 |

### 优化建议

```cpp
// 根据质量等级调整后处理
switch (QualityLevel) {
    case EQualityLevel::Low:
        // 禁用昂贵效果
        PostProcessVolume->Settings.bOverride_ScreenSpaceReflectionIntensity = true;
        PostProcessVolume->Settings.ScreenSpaceReflectionIntensity = 0.0;
        break;
        
    case EQualityLevel::High:
        // 启用所有效果
        PostProcessVolume->Settings.ScreenSpaceReflectionIntensity = 1.0;
        break;
}
```

## 常见问题

### 问题1: 后处理效果不显示

**症状**: 配置的后处理效果不显示

**原因**: 
- 后处理体积未启用
- 摄像机不在体积内

**解决**:
```cpp
// 确保体积启用
PostProcessVolume->bEnabled = true;

// 设置为无界
PostProcessVolume->bUnbound = true;
```

### 问题2: 性能下降

**症状**: 启用后处理后FPS下降

**原因**: 
- 后处理效果过多
- 分辨率过高

**解决**:
```cpp
// 禁用昂贵效果
PostProcessVolume->Settings.bOverride_ScreenSpaceReflectionIntensity = true;
PostProcessVolume->Settings.ScreenSpaceReflectionIntensity = 0.0;

// 降低质量
PostProcessVolume->Settings.ScreenSpaceReflectionQuality = 25;
```

### 问题3: 效果过度

**症状**: 后处理效果太强

**原因**: 
- 参数设置过高
- 多个体积叠加

**解决**:
```cpp
// 调整参数
PostProcessVolume->Settings.BloomIntensity = 0.5;  // 降低强度

// 检查优先级
PostProcessVolume->Priority = 0;  // 降低优先级
```

## 最佳实践

1. **合理使用效果** - 不是所有效果都需要启用
2. **性能监控** - 使用Stat命令监控成本
3. **质量等级** - 根据平台调整效果
4. **参数调优** - 找到质量和性能的平衡
5. **测试** - 在目标平台上测试

## 参考资源

- [UE后处理文档](https://docs.unrealengine.com/en-US/RenderingAndGraphics/PostProcessEffects/)
- [后处理材质](https://docs.unrealengine.com/en-US/RenderingAndGraphics/PostProcessEffects/PostProcessMaterials/)
- [性能优化](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Performance/)
