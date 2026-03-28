# 体积效果

## 体积效果概述

体积效果包括体积雾、体积光、体积阴影等，用于创建大气和光线效果。

## 体积雾

### 原理

在3D空间中模拟雾气效果。

```hlsl
// 体积雾计算
float3 CalculateVolumeFog(float3 WorldPos, float3 CameraPos)
{
    float Distance = length(WorldPos - CameraPos);
    float FogDensity = exp(-Distance * FogAbsorption);
    
    float3 FogColor = GetFogColor(WorldPos);
    return FogColor * (1.0 - FogDensity);
}
```

### 配置

```cpp
// 启用体积雾
r.VolumetricFog = 1;

// 设置质量
r.VolumetricFog.GridPixelSize = 8;  // 网格大小

// 设置距离
r.VolumetricFog.MaxDistance = 10000;
```

### 体积雾体积

```cpp
// 创建体积雾体积
AVolumetricFogVolume* FogVolume = GetWorld()->SpawnActor<AVolumetricFogVolume>();
FogVolume->SetActorScale3D(FVector(1000, 1000, 500));

// 设置雾参数
FogVolume->SetDensity(0.1f);
FogVolume->SetAbsorption(0.5f);
```

## 体积光

### 原理

光线在体积中的散射效果。

```hlsl
// 体积光计算
float3 CalculateVolumetricLight(float3 WorldPos, float3 LightDir)
{
    float3 ToLight = normalize(LightDir);
    float3 ToCamera = normalize(CameraPos - WorldPos);
    
    // 相位函数
    float Phase = dot(ToLight, ToCamera);
    
    // 体积光
    float3 VolumetricLight = LightColor * Phase;
    
    return VolumetricLight;
}
```

### 配置

```cpp
// 启用体积光
r.VolumetricLighting = 1;

// 设置质量
r.VolumetricLighting.GridPixelSize = 8;
```

## 体积阴影

### 原理

阴影在体积中的投影。

```hlsl
// 体积阴影计算
float CalculateVolumeShadow(float3 WorldPos, float3 LightDir)
{
    // 从光源视角追踪
    float Shadow = 1.0;
    
    for (int i = 0; i < NumSteps; ++i) {
        float3 SamplePos = WorldPos + LightDir * i * StepSize;
        float SampleDepth = GetDepth(SamplePos);
        
        if (SampleDepth < GetCurrentDepth(SamplePos)) {
            Shadow *= 0.9;  // 逐步减弱
        }
    }
    
    return Shadow;
}
```

## 体积纹理

### 3D纹理

```cpp
// 创建3D纹理
UVolumeTexture* VolumeTexture = NewObject<UVolumeTexture>();
VolumeTexture->Init(128, 128, 128);  // 128x128x128

// 填充数据
for (int z = 0; z < 128; ++z) {
    for (int y = 0; y < 128; ++y) {
        for (int x = 0; x < 128; ++x) {
            // 设置体素数据
        }
    }
}
```

### 体积采样

```hlsl
// 采样3D纹理
float3 VolumeColor = VolumeTexture.Sample(Sampler, float3(UV, Depth)).rgb;
```

## 性能优化

### 分辨率优化

```cpp
// 降低网格分辨率
r.VolumetricFog.GridPixelSize = 16;  // 更大的像素

// 限制距离
r.VolumetricFog.MaxDistance = 5000;
```

### 采样优化

```hlsl
// 减少采样步数
const int NumSteps = 16;  // 而不是64

// 使用低精度计算
half3 VolumeColor = half3(0, 0, 0);
```

## 常见问题

### 问题1: 性能下降

**症状**: 启用体积效果后FPS下降

**原因**: 
- 分辨率过高
- 采样步数过多

**解决**:
```cpp
// 降低分辨率
r.VolumetricFog.GridPixelSize = 16;

// 减少采样
const int NumSteps = 16;
```

### 问题2: 效果不明显

**症状**: 体积效果看不清

**原因**: 
- 参数设置过低
- 距离过远

**解决**:
```cpp
// 增加密度
FogVolume->SetDensity(0.5f);

// 减少距离
r.VolumetricFog.MaxDistance = 5000;
```

## 最佳实践

1. **合理设置分辨率** - 平衡质量和性能
2. **优化采样** - 减少采样步数
3. **限制距离** - 只计算近处
4. **性能监控** - 定期检查成本
5. **质量等级** - 根据平台调整

## 参考资源

- [UE体积效果文档](https://docs.unrealengine.com/en-US/RenderingAndGraphics/VolumetricEffects/)
- [体积雾](https://docs.unrealengine.com/en-US/RenderingAndGraphics/VolumetricEffects/VolumetricFog/)
