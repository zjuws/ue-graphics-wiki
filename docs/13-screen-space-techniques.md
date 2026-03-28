# 屏幕空间技术

## 屏幕空间技术概述

屏幕空间技术是指在屏幕空间（2D）中进行的计算，而不是在世界空间（3D）中。这些技术通常性能高效，但质量可能受限。

## 屏幕空间环境光遮挡（SSAO）

### 原理

基于深度缓冲计算环境光遮挡。

**步骤**:
1. 从深度缓冲重建世界位置
2. 在周围采样深度
3. 比较深度确定遮挡
4. 累加遮挡值

### 实现

```hlsl
// SSAO计算
float CalculateSSAO(float2 UV, float Depth)
{
    float3 WorldPos = ReconstructWorldPosition(UV, Depth);
    float3 Normal = GetNormal(UV);
    
    float Occlusion = 0.0;
    
    // 在周围采样
    for (int i = 0; i < NumSamples; ++i) {
        float2 SampleUV = UV + SampleOffsets[i];
        float SampleDepth = DepthBuffer.Sample(SampleUV).r;
        float3 SamplePos = ReconstructWorldPosition(SampleUV, SampleDepth);
        
        float3 ToSample = SamplePos - WorldPos;
        float Distance = length(ToSample);
        float3 Direction = ToSample / Distance;
        
        // 检查是否被遮挡
        float Occlusion += max(0.0, dot(Normal, Direction) - Bias) / (Distance + Epsilon);
    }
    
    return 1.0 - (Occlusion / NumSamples);
}
```

### 配置

```cpp
// 启用SSAO
r.AmbientOcclusionIntensity = 1.0;
r.AmbientOcclusionRadius = 100.0;
r.AmbientOcclusionQuality = 2;
```

## 屏幕空间反射（SSR）

### 原理

在屏幕空间中进行光线追踪计算反射。

**步骤**:
1. 计算反射光线
2. 在屏幕空间中追踪光线
3. 采样反射颜色
4. 混合到最终颜色

### 实现

```hlsl
// SSR光线追踪
float3 CalculateSSR(float3 WorldPos, float3 Normal, float3 ViewDir)
{
    // 计算反射方向
    float3 ReflectionDir = reflect(ViewDir, Normal);
    
    // 投影到屏幕空间
    float3 ReflectionPos = WorldPos + ReflectionDir * RayDistance;
    float4 ProjPos = mul(float4(ReflectionPos, 1), ViewProjection);
    float2 ScreenCoord = ProjPos.xy / ProjPos.w * 0.5 + 0.5;
    
    // 光线追踪
    float3 ReflectionColor = float3(0, 0, 0);
    float Confidence = 0.0;
    
    for (int i = 0; i < NumSteps; ++i) {
        float2 CurrentCoord = ScreenCoord + ReflectionDir.xy * i * StepSize;
        
        // 检查是否超出屏幕
        if (CurrentCoord.x < 0 || CurrentCoord.x > 1 ||
            CurrentCoord.y < 0 || CurrentCoord.y > 1) {
            break;
        }
        
        // 采样深度
        float SampleDepth = DepthBuffer.Sample(CurrentCoord).r;
        float CurrentDepth = GetDepth(CurrentCoord);
        
        // 检查是否命中
        if (SampleDepth < CurrentDepth) {
            ReflectionColor = SceneTexture.Sample(CurrentCoord).rgb;
            Confidence = 1.0;
            break;
        }
    }
    
    return ReflectionColor * Confidence;
}
```

### 配置

```cpp
// 启用SSR
r.ScreenSpaceReflectionIntensity = 1.0;
r.ScreenSpaceReflectionQuality = 50;
r.ScreenSpaceReflectionMaxRoughness = 0.6;
```

## 屏幕空间全局光照（SSGI）

### 原理

在屏幕空间中计算全局光照。

**特点**:
- 实时计算
- 性能相对高效
- 质量受屏幕空间限制

### 配置

```cpp
// 启用SSGI
r.ScreenSpaceGlobalIllumination = 1;
r.ScreenSpaceGlobalIlluminationQuality = 2;
```

## 屏幕空间阴影（Screen Space Shadows）

### 原理

使用屏幕空间信息计算阴影。

**优点**:
- 高效
- 支持动态物体
- 无需阴影贴图

### 实现

```hlsl
// 屏幕空间阴影
float CalculateScreenSpaceShadow(float3 WorldPos, float3 LightDir)
{
    // 投影到屏幕空间
    float4 ProjPos = mul(float4(WorldPos, 1), ViewProjection);
    float2 ScreenCoord = ProjPos.xy / ProjPos.w * 0.5 + 0.5;
    
    // 沿光线方向追踪
    float Shadow = 1.0;
    
    for (int i = 0; i < NumSteps; ++i) {
        float2 SampleCoord = ScreenCoord + LightDir.xy * i * StepSize;
        
        // 采样深度
        float SampleDepth = DepthBuffer.Sample(SampleCoord).r;
        float CurrentDepth = GetDepth(ScreenCoord);
        
        // 检查是否在阴影中
        if (SampleDepth < CurrentDepth) {
            Shadow = 0.0;
            break;
        }
    }
    
    return Shadow;
}
```

## 屏幕空间法线重建

### 原理

从深度缓冲重建法线。

### 实现

```hlsl
// 从深度重建法线
float3 ReconstructNormal(float2 UV)
{
    float Depth = DepthBuffer.Sample(UV).r;
    float DepthLeft = DepthBuffer.Sample(UV - float2(PixelSize, 0)).r;
    float DepthUp = DepthBuffer.Sample(UV - float2(0, PixelSize)).r;
    
    float3 Pos = ReconstructWorldPosition(UV, Depth);
    float3 PosLeft = ReconstructWorldPosition(UV - float2(PixelSize, 0), DepthLeft);
    float3 PosUp = ReconstructWorldPosition(UV - float2(0, PixelSize), DepthUp);
    
    float3 Normal = cross(PosLeft - Pos, PosUp - Pos);
    return normalize(Normal);
}
```

## 屏幕空间位置重建

### 原理

从深度和屏幕坐标重建世界位置。

### 实现

```hlsl
// 从深度重建世界位置
float3 ReconstructWorldPosition(float2 UV, float Depth)
{
    // 转换到NDC坐标
    float2 NDC = UV * 2.0 - 1.0;
    
    // 转换到视图空间
    float4 ViewPos = mul(float4(NDC, Depth, 1), InverseProjection);
    ViewPos /= ViewPos.w;
    
    // 转换到世界空间
    float4 WorldPos = mul(ViewPos, InverseView);
    
    return WorldPos.xyz;
}
```

## 性能考虑

### 屏幕空间技术成本

| 技术 | 成本 | 质量 | 说明 |
|------|------|------|------|
| SSAO | 中 | 中 | 常用 |
| SSR | 高 | 中 | 昂贵 |
| SSGI | 高 | 中 | 昂贵 |
| 屏幕空间阴影 | 中 | 低 | 快速 |

### 优化建议

```cpp
// 降低采样数量
r.ScreenSpaceReflectionQuality = 25;  // 低质量

// 限制追踪距离
r.ScreenSpaceReflectionMaxDistance = 5000;

// 使用较低的分辨率
r.ScreenPercentage = 80;
```

## 常见问题

### 问题1: SSAO伪影

**症状**: SSAO出现条纹或斑点

**原因**: 
- 采样不足
- 偏差设置不当

**解决**:
```cpp
// 增加采样数量
r.AmbientOcclusionQuality = 3;

// 调整半径
r.AmbientOcclusionRadius = 150.0;
```

### 问题2: SSR不准确

**症状**: 反射位置不对或缺失

**原因**: 
- 屏幕空间限制
- 追踪步数不足

**解决**:
```cpp
// 增加质量
r.ScreenSpaceReflectionQuality = 100;

// 增加最大距离
r.ScreenSpaceReflectionMaxDistance = 10000;
```

### 问题3: 性能下降

**症状**: 启用屏幕空间技术后FPS下降

**原因**: 
- 采样过多
- 追踪距离过长

**解决**:
```cpp
// 降低质量
r.ScreenSpaceReflectionQuality = 25;

// 禁用昂贵技术
r.ScreenSpaceGlobalIllumination = 0;
```

## 最佳实践

1. **选择合适的技术**
   - SSAO: 总是启用
   - SSR: 高端平台
   - SSGI: 特殊场景

2. **质量和性能平衡**
   - 根据平台调整质量
   - 监控性能成本
   - 使用质量等级

3. **处理屏幕空间限制**
   - 屏幕边界处理
   - 深度不连续处理
   - 信心值混合

4. **性能监控**
   - 使用Stat命令
   - 分析成本
   - 优化关键路径

## 参考资源

- [UE屏幕空间技术](https://docs.unrealengine.com/en-US/RenderingAndGraphics/ScreenSpaceTechniques/)
- [SSAO文档](https://docs.unrealengine.com/en-US/RenderingAndGraphics/ScreenSpaceTechniques/SSAO/)
- [SSR文档](https://docs.unrealengine.com/en-US/RenderingAndGraphics/ScreenSpaceTechniques/SSR/)
