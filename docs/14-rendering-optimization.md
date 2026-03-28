# 渲染优化

## 优化概述

渲染优化的目标是在保持视觉质量的前提下，最大化帧率和降低功耗。

## 性能分析工具

### Stat命令

```cpp
// 总体性能
stat unit

// 详细渲染统计
stat scenerendering

// 单位图表
stat unitgraph

// 内存统计
stat memory
```

### GPU分析

```cpp
// 启用GPU分析
r.GpuCrashDebugging 1

// 查看GPU时间
stat gpu
```

### 性能分析器（Profiler）

- 打开：Window → Developer Tools → Profiler
- 记录帧数据
- 分析CPU/GPU时间分布

## 批处理优化

### 静态批处理（Static Batching）

**原理**: 合并静态物体的顶点数据

**配置**:
```cpp
// 在Actor上
Actor->SetMobility(EComponentMobility::Static);
```

**优点**:
- 减少绘制调用
- 性能提升显著

**缺点**:
- 不支持动态变化
- 内存占用增加

### 动态批处理（Dynamic Batching）

**原理**: 运行时合并动态物体

**条件**:
- 相同材质
- 顶点数 < 900
- 相同光照贴图

**代码**:
```cpp
// 启用动态批处理
r.AllowDynamicBatching 1
```

### 实例化（Instancing）

**原理**: 一次绘制调用渲染多个实例

**代码**:
```cpp
// 创建实例化静态网格
UInstancedStaticMeshComponent* InstancedMesh = 
    NewObject<UInstancedStaticMeshComponent>(this);

// 添加实例
for (int i = 0; i < 1000; ++i) {
    FTransform Transform = FTransform(FVector(i * 100, 0, 0));
    InstancedMesh->AddInstance(Transform);
}
```

## LOD（细节级别）

### 静态LOD

**原理**: 根据距离使用不同分辨率的模型

**配置**:
```cpp
// 在编辑器中设置LOD
// 或在代码中
UStaticMesh* Mesh = GetStaticMesh();
Mesh->GetNumLODs();  // 获取LOD数量
```

### 动态LOD

```cpp
// 根据距离计算LOD
float Distance = FVector::Dist(CameraPos, ActorPos);
int32 LOD = 0;
if (Distance > 1000) LOD = 1;
if (Distance > 5000) LOD = 2;
if (Distance > 10000) LOD = 3;

SkeletalMesh->SetForcedLOD(LOD);
```

### LOD最佳实践

| 距离 | LOD | 顶点数 | 说明 |
|------|-----|--------|------|
| 0-100m | 0 | 100% | 高质量 |
| 100-500m | 1 | 50% | 中等质量 |
| 500-2000m | 2 | 25% | 低质量 |
| 2000m+ | 3 | 10% | 极低质量 |

## 遮挡剔除（Occlusion Culling）

### 预计算遮挡

**工作流**:
1. 标记遮挡物体
2. 构建遮挡数据
3. 运行时查询

**代码**:
```cpp
// 标记为遮挡物体
Actor->SetActorEnableCollision(true);
```

### 动态遮挡查询

```cpp
// 查询是否被遮挡
bool bIsOccluded = GetWorld()->LineTraceSingleByChannel(
    HitResult,
    CameraPos,
    ActorPos,
    ECC_Visibility
);
```

## 纹理优化

### 纹理压缩

**格式选择**:
| 格式 | 压缩率 | 质量 | 用途 |
|------|--------|------|------|
| DXT1 | 6:1 | 低 | 颜色 |
| DXT5 | 4:1 | 中 | 法线 |
| BC4 | 2:1 | 高 | 灰度 |
| BC6H | 6:1 | 高 | HDR |

**配置**:
```cpp
// 在纹理属性中设置压缩格式
Texture->CompressionSettings = TC_Default;
```

### 虚拟纹理（Virtual Texture）

**原理**: 按需加载纹理页面

**优点**:
- 减少内存占用
- 支持大型纹理
- 自动流式加载

**配置**:
```cpp
// 启用虚拟纹理
Texture->VirtualTextureStreaming = true;
```

### 纹理流式加载

```cpp
// 设置纹理优先级
Texture->UpdateResource();

// 强制加载
Texture->SetForceMipLevelsToBeResident(true);
```

## 着色器优化

### 指令计数

```hlsl
// 不好：复杂计算
float3 Result = normalize(cross(A, cross(B, C)));

// 好：简化计算
float3 Result = normalize(A);
```

### 分支优化

```hlsl
// 不好：动态分支
if (Condition) {
    Result = ExpensiveCalculation();
} else {
    Result = CheapCalculation();
}

// 好：使用lerp
Result = lerp(CheapCalculation(), ExpensiveCalculation(), Condition);
```

### 纹理采样优化

```hlsl
// 不好：多次采样
float3 Color1 = Texture.Sample(UV);
float3 Color2 = Texture.Sample(UV);

// 好：采样一次
float3 Color = Texture.Sample(UV);
float3 Color1 = Color;
float3 Color2 = Color;
```

## 光照优化

### 光照剔除

```cpp
// 只计算影响该物体的光源
TArray<FLightSceneInfo*> RelevantLights;
Scene->GetRelevantLights(Primitive, RelevantLights);
```

### 烘焙光照

**优点**:
- 性能最优
- 支持复杂光照

**缺点**:
- 不支持动态变化
- 需要预计算

### 光照探针

```cpp
// 创建光照探针
ALightProbeVolume* ProbeVolume = 
    GetWorld()->SpawnActor<ALightProbeVolume>();
```

## 移动平台优化

### 分辨率缩放

```cpp
// 动态调整分辨率
float TargetFPS = 60.0f;
float CurrentFPS = 1.0f / DeltaTime;

if (CurrentFPS < TargetFPS * 0.9f) {
    // 降低分辨率
    r.ScreenPercentage = 80;
} else if (CurrentFPS > TargetFPS * 1.1f) {
    // 提高分辨率
    r.ScreenPercentage = 100;
}
```

### 功耗管理

```cpp
// 限制帧率
r.VSync = true;
r.MaxFPS = 60;

// 降低质量
r.DefaultFeature.AntiAliasing = 0;
r.DefaultFeature.MotionBlur = 0;
```

## 性能目标

### 桌面平台
- **目标FPS**: 60-144
- **GPU时间**: < 16ms (60fps) 或 < 7ms (144fps)
- **CPU时间**: < 16ms

### 移动平台
- **目标FPS**: 30-60
- **GPU时间**: < 33ms (30fps) 或 < 16ms (60fps)
- **CPU时间**: < 33ms
- **内存**: < 2GB

### VR平台
- **目标FPS**: 90
- **GPU时间**: < 11ms
- **CPU时间**: < 11ms
- **延迟**: < 20ms

## 优化检查清单

- [ ] 使用Stat命令分析性能
- [ ] 启用批处理
- [ ] 配置LOD
- [ ] 使用遮挡剔除
- [ ] 压缩纹理
- [ ] 优化着色器
- [ ] 烘焙光照
- [ ] 测试移动平台
- [ ] 监控内存使用
- [ ] 定期性能测试

## 参考资源

- [UE性能优化指南](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Performance/)
- [GPU优化](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Performance/GPU/)
