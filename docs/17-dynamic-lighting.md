# 动态光照

## 动态光照概述

动态光照是指在运行时实时计算的光照，与烘焙光照相对。动态光照支持移动的光源和物体，但性能消耗较大。

## 动态光照类型

### 可移动光源（Movable Lights）

```cpp
// 创建可移动光源
ADirectionalLight* Light = GetWorld()->SpawnActor<ADirectionalLight>();
Light->SetMobility(EComponentMobility::Movable);
Light->SetIntensity(2.0f);
```

**特点**:
- 可以移动和旋转
- 支持动态阴影
- 性能消耗大

### 静态光源（Static Lights）

```cpp
// 创建静态光源
ADirectionalLight* Light = GetWorld()->SpawnActor<ADirectionalLight>();
Light->SetMobility(EComponentMobility::Static);
```

**特点**:
- 不能移动
- 使用烘焙光照
- 性能最优

### 固定光源（Stationary Lights）

```cpp
// 创建固定光源
ADirectionalLight* Light = GetWorld()->SpawnActor<ADirectionalLight>();
Light->SetMobility(EComponentMobility::Stationary);
```

**特点**:
- 不能移动，但可以改变颜色/强度
- 混合烘焙和动态阴影
- 性能平衡

## 动态阴影

### 实时阴影贴图

```cpp
// 启用实时阴影
Light->bCastDynamicShadow = true;
Light->DynamicShadowDistances = {5000, 10000, 20000, 50000};
Light->DynamicShadowCascades = 4;
Light->ShadowMapResolution = 2048;
```

### 级联阴影贴图（CSM）

**原理**: 将视锥体分为多个级联，每个级联使用不同分辨率的阴影贴图。

```cpp
// 配置级联
Light->DynamicShadowCascades = 4;
Light->DynamicShadowDistances = {5000, 10000, 20000, 50000};

// 级联0: 0-5000m, 高分辨率
// 级联1: 5000-10000m, 中分辨率
// 级联2: 10000-20000m, 低分辨率
// 级联3: 20000-50000m, 极低分辨率
```

### 距离场阴影

```cpp
// 启用距离场阴影
r.DistanceFieldShadowing = 1;

// 配置距离场
r.DistanceFieldAO = 1;
```

**优点**:
- 高效
- 支持大范围阴影
- 可用于全局光照

## 光照探针

### 光照探针体积

```cpp
// 创建光照探针体积
ALightProbeVolume* ProbeVolume = GetWorld()->SpawnActor<ALightProbeVolume>();
ProbeVolume->SetActorScale3D(FVector(1000, 1000, 500));

// 生成探针
ProbeVolume->GenerateProbes();
```

### 探针放置策略

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
// 获取光照探针颜色
FVector WorldPos = Actor->GetActorLocation();
FLinearColor ProbeColor = GetLightProbeColor(WorldPos);

// 应用到动态物体
DynamicMesh->SetLightProbeColor(ProbeColor);
```

## 反射探针

### 反射捕捉

```cpp
// 创建球形反射捕捉
ASphereReflectionCapture* Capture = 
    GetWorld()->SpawnActor<ASphereReflectionCapture>();
Capture->SetActorScale3D(FVector(1000, 1000, 1000));

// 手动捕捉
Capture->CaptureScene();
```

### 反射更新

```cpp
// 定期更新反射
void UpdateReflections()
{
    for (TActorIterator<AReflectionCapture> It(GetWorld()); It; ++It) {
        It->CaptureScene();
    }
}
```

## 动态光照优化

### 光源剔除

```cpp
// 获取影响该物体的光源
TArray<FLightSceneInfo*> RelevantLights;
Scene->GetRelevantLights(Primitive, RelevantLights);

// 只计算相关光源
for (FLightSceneInfo* Light : RelevantLights) {
    // 计算光照
}
```

### 光源距离限制

```cpp
// 限制光源影响范围
Light->SetAttenuationRadius(1000.0f);

// 只影响近处物体
Light->bAffectsWorld = true;
```

### 阴影优化

```cpp
// 限制阴影距离
r.Shadow.MaxCSMCascades = 2;  // 只使用2个级联

// 降低阴影分辨率
Light->ShadowMapResolution = 1024;

// 禁用远处阴影
r.Shadow.MaxTranslucencyObjectsPerLight = 0;
```

## 性能考虑

### 动态光照成本

| 光源类型 | 成本 | 说明 |
|---------|------|------|
| 静态 | 低 | 烘焙 |
| 固定 | 中 | 混合 |
| 可移动 | 高 | 实时 |

### 优化建议

```cpp
// 使用固定光源而不是可移动
Light->SetMobility(EComponentMobility::Stationary);

// 限制动态光源数量
const int MaxDynamicLights = 4;

// 使用烘焙光照
StaticMeshComponent->SetMobility(EComponentMobility::Static);
```

## 常见问题

### 问题1: 动态阴影闪烁

**症状**: 阴影不稳定或闪烁

**原因**: 
- 级联过多
- 阴影偏差设置不当

**解决**:
```cpp
// 减少级联数量
Light->DynamicShadowCascades = 2;

// 调整阴影偏差
Light->ShadowBias = 0.01f;
```

### 问题2: 光照探针不准确

**症状**: 动态物体光照不正确

**原因**: 
- 探针放置不当
- 探针数量不足

**解决**:
```cpp
// 增加探针数量
ProbeVolume->SetActorScale3D(FVector(2000, 2000, 1000));

// 重新生成探针
ProbeVolume->GenerateProbes();
```

### 问题3: 性能下降

**症状**: 启用动态光照后FPS下降

**原因**: 
- 动态光源过多
- 阴影计算过多

**解决**:
```cpp
// 限制动态光源
const int MaxDynamicLights = 2;

// 使用固定光源
Light->SetMobility(EComponentMobility::Stationary);

// 禁用远处阴影
r.Shadow.MaxCSMCascades = 1;
```

## 最佳实践

1. **混合光照方案**
   - 静态物体: 烘焙光照
   - 动态物体: 光照探针
   - 特殊物体: 动态光照

2. **优化光源数量**
   - 限制可移动光源
   - 使用固定光源
   - 进行光源剔除

3. **优化阴影**
   - 限制级联数量
   - 降低分辨率
   - 限制距离

4. **性能监控**
   - 使用Stat命令
   - 分析光照成本
   - 优化关键路径

## 参考资源

- [UE动态光照文档](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Lighting/DynamicLighting/)
- [光照探针](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Lighting/LightProbes/)
- [反射捕捉](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Lighting/ReflectionCaptures/)
