# 移动平台优化

## 移动平台特点

### 硬件限制

| 方面 | 限制 | 说明 |
|------|------|------|
| GPU | 弱 | 1/10 PC性能 |
| CPU | 弱 | 多核但频率低 |
| 内存 | 有限 | 2-6GB |
| 电池 | 有限 | 需要省电 |
| 屏幕 | 小 | 1080p-2K |

### 目标

- **帧率**: 30-60 FPS
- **功耗**: 低
- **内存**: < 2GB
- **加载时间**: < 5秒

## 渲染优化

### 分辨率缩放

**原理**: 动态调整渲染分辨率

```cpp
// 设置屏幕百分比
r.ScreenPercentage = 100;  // 100%分辨率

// 根据性能调整
if (CurrentFPS < TargetFPS * 0.9f) {
    r.ScreenPercentage = 80;  // 降低到80%
} else if (CurrentFPS > TargetFPS * 1.1f) {
    r.ScreenPercentage = 100;  // 提高到100%
}
```

### 禁用昂贵效果

```cpp
// 禁用屏幕空间反射
r.ScreenSpaceReflectionIntensity = 0.0;

// 禁用全局光照
r.DynamicGlobalIlluminationIntensity = 0.0;

// 禁用景深
r.DepthOfFieldQuality = 0;

// 禁用运动模糊
r.MotionBlurQuality = 0;

// 禁用SSAO
r.AmbientOcclusionIntensity = 0.0;
```

### 简化材质

```hlsl
// 移动平台材质（简化版）
void MainPS(
    in float2 UV : TEXCOORD0,
    out float4 OutColor : SV_Target0)
{
    // 只采样一次
    float4 BaseColor = Texture.Sample(Sampler, UV);
    
    // 简单光照计算
    float3 Normal = normalize(NormalTexture.Sample(Sampler, UV).rgb * 2.0 - 1.0);
    float3 LightDir = normalize(LightDirection);
    float Diffuse = max(dot(Normal, LightDir), 0.0);
    
    OutColor = BaseColor * Diffuse;
}
```

## 纹理优化

### 分辨率限制

```cpp
// 限制纹理分辨率
r.Streaming.MaxTextureSize = 1024;  // 最大1024x1024

// 强制压缩
r.Streaming.UseTextureFileCache = 1;
```

### 纹理压缩

```cpp
// 使用ETC2压缩（Android）
Texture->CompressionSettings = TC_Default;

// 使用PVRTC压缩（iOS）
Texture->CompressionSettings = TC_Default;
```

### 虚拟纹理

```cpp
// 启用虚拟纹理
r.VirtualTexture.Enable = 1;

// 设置页面大小
r.VirtualTexture.PageSize = 128;
```

## 几何优化

### LOD配置

```cpp
// 设置LOD距离
StaticMesh->LODDistances = {
    0,      // LOD0: 0-500m
    500,    // LOD1: 500-1000m
    1000,   // LOD2: 1000-2000m
    2000    // LOD3: 2000m+
};

// 设置LOD顶点数
// LOD0: 100%
// LOD1: 50%
// LOD2: 25%
// LOD3: 10%
```

### 批处理

```cpp
// 启用动态批处理
r.AllowDynamicBatching = 1;

// 启用静态批处理
StaticMeshComponent->SetMobility(EComponentMobility::Static);
```

### 遮挡剔除

```cpp
// 启用遮挡剔除
r.OcclusionCulling = 1;

// 设置遮挡剔除距离
r.OcclusionCullingDistance = 5000;
```

## 光照优化

### 烘焙光照

```cpp
// 使用烘焙光照
StaticMeshComponent->SetMobility(EComponentMobility::Static);

// 禁用实时光照
DirectionalLight->SetMobility(EComponentMobility::Static);
```

### 限制动态光源

```cpp
// 限制动态光源数量
r.DeferredLighting.MaxLights = 32;  // 移动平台限制

// 使用光照剔除
r.TiledDeferredShading = 1;
```

### 阴影优化

```cpp
// 禁用动态阴影
DirectionalLight->bCastDynamicShadow = false;

// 使用烘焙阴影
DirectionalLight->bCastStaticShadow = true;

// 限制阴影距离
r.Shadow.MaxCSMCascades = 1;  // 只使用1个级联
```

## 内存优化

### 内存预算

```cpp
// 设置内存预算
r.Streaming.PoolSize = 512;  // 512MB纹理池

// 设置音频内存
r.Audio.PoolSize = 64;  // 64MB音频池
```

### 资源流式加载

```cpp
// 启用资源流式加载
r.Streaming.UseTextureFileCache = 1;

// 设置流式加载优先级
Texture->StreamingPriority = 1;
```

### 垃圾回收

```cpp
// 定期垃圾回收
GetWorld()->ForceGarbageCollection(true);

// 设置垃圾回收间隔
r.GarbageCollection.TimeBetweenPasses = 60;  // 60秒
```

## 功耗管理

### 帧率限制

```cpp
// 限制帧率
r.VSync = true;
r.MaxFPS = 60;  // 60fps

// 移动平台可以降低到30fps
r.MaxFPS = 30;
```

### 省电模式

```cpp
// 检测电池状态
bool bLowBattery = FPlatformMisc::GetBatteryLevel() < 0.2f;

if (bLowBattery) {
    // 降低质量
    r.ScreenPercentage = 75;
    r.MaxFPS = 30;
    r.ScreenSpaceReflectionIntensity = 0.0;
}
```

## 平台特定优化

### Android优化

```cpp
// Android特定设置
#if PLATFORM_ANDROID
    // 使用ETC2压缩
    r.Streaming.MaxTextureSize = 1024;
    
    // 限制光源
    r.DeferredLighting.MaxLights = 16;
    
    // 禁用昂贵效果
    r.ScreenSpaceReflectionIntensity = 0.0;
#endif
```

### iOS优化

```cpp
// iOS特定设置
#if PLATFORM_IOS
    // 使用PVRTC压缩
    r.Streaming.MaxTextureSize = 1024;
    
    // 限制光源
    r.DeferredLighting.MaxLights = 16;
    
    // 使用Metal API
    r.RHI.PreferredAPI = "Metal";
#endif
```

## 性能分析

### 移动分析工具

#### Android Profiler
```cpp
// 启用Android分析
r.GPUCrashDebugging = 1;
```

#### Xcode Instruments
```cpp
// 启用iOS分析
r.GPUCrashDebugging = 1;
```

### 性能指标

| 指标 | 目标 | 说明 |
|------|------|------|
| FPS | 30-60 | 帧率 |
| GPU时间 | < 33ms | 30fps |
| CPU时间 | < 33ms | 30fps |
| 内存 | < 2GB | 总内存 |
| 功耗 | 低 | 电池消耗 |

## 优化检查清单

- [ ] 设置合适的分辨率
- [ ] 启用纹理压缩
- [ ] 配置LOD
- [ ] 使用烘焙光照
- [ ] 限制动态光源
- [ ] 禁用昂贵效果
- [ ] 优化材质
- [ ] 启用批处理
- [ ] 监控内存
- [ ] 测试性能

## 常见问题

### 问题1: 帧率不稳定

**症状**: FPS波动大

**原因**: 
- 垃圾回收
- 资源加载
- 动态分配

**解决**:
```cpp
// 预加载资源
PreloadAssets();

// 定期垃圾回收
GetWorld()->ForceGarbageCollection(true);

// 避免动态分配
```

### 问题2: 内存不足

**症状**: 应用崩溃

**原因**: 
- 纹理分辨率过高
- 未压缩资源

**解决**:
```cpp
// 降低纹理分辨率
r.Streaming.MaxTextureSize = 512;

// 启用虚拟纹理
r.VirtualTexture.Enable = 1;
```

### 问题3: 功耗过高

**症状**: 电池快速耗尽

**原因**: 
- 帧率过高
- 昂贵效果

**解决**:
```cpp
// 限制帧率
r.MaxFPS = 30;

// 禁用昂贵效果
r.ScreenSpaceReflectionIntensity = 0.0;
```

## 最佳实践

1. **从低质量开始** - 逐步提高质量
2. **定期测试** - 在真实设备上测试
3. **监控性能** - 使用分析工具
4. **优化关键路径** - 优先优化瓶颈
5. **平衡质量** - 在质量和性能之间平衡

## 参考资源

- [UE移动优化](https://docs.unrealengine.com/en-US/SharingAndReleasing/Mobile/)
- [Android优化](https://docs.unrealengine.com/en-US/SharingAndReleasing/Mobile/Android/)
- [iOS优化](https://docs.unrealengine.com/en-US/SharingAndReleasing/Mobile/iOS/)
