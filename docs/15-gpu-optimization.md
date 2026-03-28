# GPU优化

## GPU架构基础

### GPU vs CPU

| 特性 | GPU | CPU |
|------|-----|-----|
| 核心数 | 多（1000+） | 少（8-16） |
| 时钟频率 | 低（1-2GHz） | 高（3-5GHz） |
| 缓存 | 小 | 大 |
| 延迟 | 高 | 低 |
| 吞吐量 | 高 | 低 |

### GPU执行模型

**SIMD（单指令多数据）**:
- 所有线程执行相同指令
- 不同数据
- 分支会导致性能下降

## 着色器优化

### 指令计数

**目标**: 减少着色器中的指令数

**常见优化**:

```hlsl
// 不好：复杂计算
float3 Result = normalize(cross(A, cross(B, C)));

// 好：简化计算
float3 Result = normalize(A);
```

### 纹理采样优化

**问题**: 纹理采样是昂贵的操作

**优化方案**:

```hlsl
// 不好：多次采样同一纹理
float3 Color1 = Texture.Sample(UV);
float3 Color2 = Texture.Sample(UV);
float3 Color3 = Texture.Sample(UV);

// 好：采样一次，复用结果
float3 Color = Texture.Sample(UV);
float3 Color1 = Color;
float3 Color2 = Color;
float3 Color3 = Color;
```

### 分支优化

**问题**: 分支导致GPU线程发散

**优化方案**:

```hlsl
// 不好：动态分支
if (Condition) {
    Result = ExpensiveCalculation();
} else {
    Result = CheapCalculation();
}

// 好：使用lerp消除分支
Result = lerp(CheapCalculation(), ExpensiveCalculation(), Condition);

// 好：使用step函数
Result = step(Threshold, Value) * ExpensiveCalculation() + 
         (1.0 - step(Threshold, Value)) * CheapCalculation();
```

### 精度优化

**问题**: 高精度计算消耗更多资源

**优化方案**:

```hlsl
// 不好：使用float64
double Result = double(A) * double(B);

// 好：使用float32
float Result = A * B;

// 好：使用half（如果支持）
half Result = half(A) * half(B);
```

## 内存优化

### 内存带宽

**问题**: 内存带宽是GPU的瓶颈

**计算**:
```
带宽需求 = 纹理大小 × 采样次数 × 帧率
```

**优化方案**:

```hlsl
// 不好：多次采样大纹理
float4 Color1 = LargeTexture.Sample(UV);
float4 Color2 = LargeTexture.Sample(UV + Offset1);
float4 Color3 = LargeTexture.Sample(UV + Offset2);

// 好：使用压缩纹理
float4 Color1 = CompressedTexture.Sample(UV);
float4 Color2 = CompressedTexture.Sample(UV + Offset1);
float4 Color3 = CompressedTexture.Sample(UV + Offset2);
```

### 纹理缓存

**原理**: GPU有纹理缓存，相邻像素的访问会命中缓存

**优化方案**:

```hlsl
// 好：顺序访问（缓存友好）
for (int i = 0; i < Width; ++i) {
    float4 Color = Texture.Load(int2(i, y));
}

// 不好：随机访问（缓存不友好）
for (int i = 0; i < Width; ++i) {
    float4 Color = Texture.Load(int2(Random(i), Random(i)));
}
```

### 寄存器压力

**问题**: 寄存器不足导致溢出到内存

**优化方案**:

```hlsl
// 不好：使用太多临时变量
float3 Temp1 = ...;
float3 Temp2 = ...;
float3 Temp3 = ...;
float3 Temp4 = ...;
float3 Temp5 = ...;
float3 Result = Temp1 + Temp2 + Temp3 + Temp4 + Temp5;

// 好：及时释放不需要的变量
float3 Temp1 = ...;
float3 Result = Temp1;
float3 Temp2 = ...;
Result += Temp2;
// Temp1已释放
```

## 填充率优化

### 概念

填充率是GPU每秒能写入的像素数。

**计算**:
```
填充率 = GPU频率 × 像素管道数
```

### 优化方案

```cpp
// 降低分辨率
r.ScreenPercentage = 80;  // 80%分辨率

// 使用MSAA而不是超采样
r.MSAA.CompileTarget = 4;  // 4x MSAA

// 启用Early-Z
r.EarlyZPass = 1;
```

## 计算着色器优化

### 工作组大小

**原理**: 工作组大小影响缓存效率

**优化方案**:

```hlsl
// 好的工作组大小
[numthreads(8, 8, 1)]  // 64线程
void ComputeShader(uint3 DispatchThreadID : SV_DispatchThreadID)
{
    // ...
}

// 避免
[numthreads(1, 1, 1)]  // 1线程（太小）
[numthreads(1024, 1, 1)]  // 1024线程（太大）
```

### 共享内存

**原理**: 共享内存比全局内存快得多

**优化方案**:

```hlsl
groupshared float SharedData[64];

[numthreads(8, 8, 1)]
void ComputeShader(uint3 DispatchThreadID : SV_DispatchThreadID,
                   uint3 GroupThreadID : SV_GroupThreadID)
{
    uint Index = GroupThreadID.y * 8 + GroupThreadID.x;
    
    // 写入共享内存
    SharedData[Index] = GlobalData[DispatchThreadID.y * Width + DispatchThreadID.x];
    
    // 同步所有线程
    GroupMemoryBarrierWithGroupSync();
    
    // 从共享内存读取
    float Value = SharedData[Index];
}
```

## 性能分析

### GPU分析工具

#### NVIDIA NSight

```cpp
// 启用NVIDIA分析
r.GPUCrashDebugging = 1;
```

#### AMD Radeon GPU Profiler

```cpp
// 启用AMD分析
r.GPUCrashDebugging = 1;
```

#### UE内置分析

```cpp
// 启用GPU时间统计
stat gpu

// 启用详细统计
stat scenerendering
```

### 性能指标

| 指标 | 目标 | 说明 |
|------|------|------|
| GPU时间 | < 16ms | 60fps |
| 填充率 | 高 | 像素吞吐量 |
| 内存带宽 | 低 | 内存使用 |
| 缓存命中率 | 高 | 缓存效率 |

## 平台特定优化

### PC优化

```cpp
// 启用高级功能
r.ScreenSpaceReflectionIntensity = 1.0;
r.DynamicGlobalIlluminationIntensity = 1.0;

// 使用高分辨率纹理
r.TextureStreamingPoolSize = 2000;
```

### 移动优化

```cpp
// 禁用昂贵效果
r.ScreenSpaceReflectionIntensity = 0.0;
r.DynamicGlobalIlluminationIntensity = 0.0;

// 降低分辨率
r.ScreenPercentage = 75;

// 使用低精度
r.ShaderPrecisionMode = 1;  // 低精度
```

### VR优化

```cpp
// 立体渲染
r.StereoRenderingMethod = 1;

// 降低分辨率
r.ScreenPercentage = 80;

// 启用异步时间扭曲
r.AllowAsyncTimeWarp = 1;
```

## 常见问题

### 问题1: GPU瓶颈

**症状**: GPU使用率100%，FPS低

**原因**: 
- 着色器复杂
- 纹理采样过多
- 分辨率过高

**解决**:
```cpp
// 降低分辨率
r.ScreenPercentage = 80;

// 禁用昂贵效果
r.ScreenSpaceReflectionIntensity = 0.0;

// 优化着色器
// 减少纹理采样
// 简化计算
```

### 问题2: 内存不足

**症状**: 内存占用过高

**原因**: 
- 纹理分辨率过高
- 未压缩纹理

**解决**:
```cpp
// 启用虚拟纹理
r.VirtualTexture.Enable = 1;

// 压缩纹理
Texture->CompressionSettings = TC_Default;

// 降低分辨率
r.ScreenPercentage = 75;
```

### 问题3: 缓存未命中

**症状**: 性能不稳定

**原因**: 
- 随机内存访问
- 工作组大小不当

**解决**:
```hlsl
// 使用顺序访问
for (int i = 0; i < Width; ++i) {
    // 顺序访问
}

// 调整工作组大小
[numthreads(8, 8, 1)]
```

## 最佳实践

1. **测量** - 使用分析工具找到瓶颈
2. **优化** - 针对瓶颈进行优化
3. **验证** - 确保优化有效
4. **平衡** - 在质量和性能之间平衡
5. **迭代** - 持续优化

## 参考资源

- [UE GPU优化](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Performance/GPU/)
- [NVIDIA优化指南](https://developer.nvidia.com/blog/optimizing-unreal-engine-5-for-nvidia-gpus/)
- [AMD优化指南](https://gpuopen.com/unreal-engine/)
