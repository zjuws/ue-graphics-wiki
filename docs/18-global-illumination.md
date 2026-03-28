# 全局光照 (Global Illumination)

## 全局光照概述

全局光照（GI）是指光线在场景中多次反弹的结果。它提供了更真实的光照效果，但计算成本很高。

## GI方案对比

| 方案 | 质量 | 性能 | 动态 | 成本 |
|------|------|------|------|------|
| 烘焙 | 高 | 高 | 否 | 预计算 |
| 实时 | 中 | 低 | 是 | 高 |
| 混合 | 高 | 中 | 部分 | 中 |
| 距离场 | 中 | 中 | 是 | 中 |

## Lightmass烘焙

### 工作流

```
1. 设置光照贴图UV
   ↓
2. 配置Lightmass设置
   ↓
3. 构建光照
   ↓
4. 烘焙完成
```

### 光照贴图UV

**重要**: 每个静态物体需要唯一的光照贴图UV

```cpp
// 在编辑器中设置
// 1. 选择静态网格
// 2. 打开细节面板
// 3. 设置 Generate Lightmap UVs = true
// 4. 设置 Lightmap Coordinate Index = 1
```

### Lightmass配置

```cpp
// 在DefaultEngine.ini中配置
[/Script/Engine.Engine]
bUseFixedPoolSize=False

[/Script/Engine.Lightmass]
NumIndirectLightingBounces=3
NumSkyLightingBounces=1
IndirectLightingQuality=2.0
IndirectLightingSmoothness=1.0
```

### 烘焙参数

| 参数 | 范围 | 说明 |
|------|------|------|
| 间接光照反弹 | 1-5 | 光线反弹次数 |
| 天光反弹 | 1-2 | 天空光反弹 |
| 质量 | 0.5-4.0 | 烘焙质量 |
| 平滑度 | 0.5-2.0 | 结果平滑度 |

### 代码配置

```cpp
// 在Actor上配置
Actor->SetCastShadow(true);
Actor->SetLightmassSettings(
    true,   // bUseAsOccluder
    false,  // bUseAsSourceSurfaceForLighting
    false   // bUseEmissiveForStaticLighting
);

// 设置光照贴图分辨率
StaticMeshComponent->SetLightmassSettings(
    true,   // bUseAsOccluder
    false,  // bUseAsSourceSurfaceForLighting
    false   // bUseEmissiveForStaticLighting
);
```

## 光照探针（Light Probes）

### 概念

光照探针捕捉场景中的光照信息，用于动态物体。

### 放置策略

```
关键位置:
- 转角处
- 高度变化处
- 光照变化处
- 物体周围

间距: 1-2米
```

### 代码使用

```cpp
// 创建光照探针体积
ALightProbeVolume* ProbeVolume = GetWorld()->SpawnActor<ALightProbeVolume>();
ProbeVolume->SetActorScale3D(FVector(1000, 1000, 500));

// 生成探针
ProbeVolume->GenerateProbes();
```

### 光照探针插值

```hlsl
// 在着色器中使用光照探针
float3 ProbeColor = GetLightProbeColor(WorldPos);
float3 FinalColor = BaseColor * ProbeColor;
```

## 反射探针（Reflection Probes）

### 概念

反射探针捕捉环境反射。

### 类型

#### 球形反射捕捉
```cpp
ASphereReflectionCapture* Capture = 
    GetWorld()->SpawnActor<ASphereReflectionCapture>();
Capture->SetActorScale3D(FVector(1000, 1000, 1000));
```

#### 盒形反射捕捉
```cpp
ABoxReflectionCapture* Capture = 
    GetWorld()->SpawnActor<ABoxReflectionCapture>();
Capture->SetActorScale3D(FVector(1000, 1000, 500));
```

### 捕捉和更新

```cpp
// 手动捕捉
Capture->CaptureScene();

// 设置更新频率
Capture->CaptureOffsetDistance = 0;
```

## 距离场全局光照（DFGI）

### 原理

使用预计算的距离场进行全局光照计算。

### 配置

```cpp
// 启用距离场
r.DistanceFieldAO = 1;
r.DistanceFieldGI = 1;

// 设置质量
r.DistanceFieldAO.Quality = 2;
```

### 优点

- 高质量
- 支持动态物体
- 相对高效

### 缺点

- 需要预计算
- 内存占用
- 不支持复杂几何

## Lumen全局光照

### 概念

Lumen是UE5的实时全局光照系统。

### 特点

- 完全实时
- 支持动态物体
- 高质量
- 性能消耗大

### 配置

```cpp
// 启用Lumen
r.Lumen.Enabled = 1;

// 设置质量
r.Lumen.Quality = 2;  // 0=低, 1=中, 2=高

// 设置距离
r.Lumen.MaxTraceDistance = 10000;
```

### 代码使用

```cpp
// 启用Lumen反射
r.Lumen.Reflections = 1;

// 启用Lumen全局光照
r.Lumen.GlobalIllumination = 1;
```

## 混合GI方案

### 策略

```
静态物体: 使用烘焙光照
动态物体: 使用光照探针
特殊物体: 使用反射探针
```

### 实现

```cpp
// 静态物体
StaticMeshComponent->SetMobility(EComponentMobility::Static);
StaticMeshComponent->SetCastShadow(true);

// 动态物体
SkeletalMeshComponent->SetMobility(EComponentMobility::Dynamic);
SkeletalMeshComponent->SetCastShadow(true);

// 使用光照探针
SkeletalMeshComponent->bUseAsOccluder = false;
```

## 性能优化

### 烘焙优化

```cpp
// 降低质量加快烘焙
r.Lightmass.IndirectLightingQuality = 1.0;

// 使用较少的反弹
r.Lightmass.NumIndirectLightingBounces = 2;
```

### 运行时优化

```cpp
// 限制探针数量
r.LightProbeMaxCount = 1000;

// 禁用昂贵的GI
r.Lumen.GlobalIllumination = 0;
```

## 常见问题

### 问题1: 烘焙时间过长

**症状**: 构建光照需要很长时间

**原因**: 
- 质量设置过高
- 反弹次数过多
- 场景过大

**解决**:
```cpp
// 降低质量
r.Lightmass.IndirectLightingQuality = 1.0;

// 减少反弹
r.Lightmass.NumIndirectLightingBounces = 2;
```

### 问题2: 光照贴图接缝

**症状**: 光照贴图之间有明显接缝

**原因**: 
- 光照贴图UV重叠
- 光照贴图分辨率过低

**解决**:
```cpp
// 增加光照贴图分辨率
StaticMeshComponent->SetLightmapResolution(256);

// 检查UV是否重叠
// 在编辑器中使用UV编辑器检查
```

### 问题3: 动态物体光照不正确

**症状**: 动态物体光照看起来不对

**原因**: 
- 光照探针放置不当
- 探针数量不足

**解决**:
```cpp
// 增加探针数量
ProbeVolume->SetActorScale3D(FVector(2000, 2000, 1000));

// 重新生成探针
ProbeVolume->GenerateProbes();
```

## 最佳实践

1. **混合方案** - 结合烘焙和实时GI
2. **探针放置** - 在关键位置放置探针
3. **质量平衡** - 在质量和性能之间平衡
4. **测试** - 在目标平台上测试
5. **优化** - 持续监控和优化

## 参考资源

- [UE全局光照文档](https://docs.unrealengine.com/en-US/RenderingAndGraphics/GlobalIllumination/)
- [Lightmass指南](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Lightmass/)
- [Lumen文档](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Lumen/)
