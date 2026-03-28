# 前向渲染

## 前向渲染概述

前向渲染（Forward Rendering）是传统的渲染方式。每个物体对每个光源进行计算，然后混合结果。

## 前向渲染vs延迟渲染

### 详细对比

| 特性 | 前向渲染 | 延迟渲染 |
|------|---------|---------|
| 光源数量 | 受限（1-4个） | 支持多个（100+） |
| 透明物体 | 原生支持 | 需要特殊处理 |
| MSAA | 简单 | 困难 |
| 内存带宽 | 低 | 高 |
| 小屏幕 | 高效 | 低效 |
| 大屏幕 | 低效 | 高效 |
| 复杂材质 | 支持 | 受限 |
| 移动平台 | 推荐 | 不推荐 |

## 前向渲染管线

### 渲染流程

```
对每个物体:
  ├─ 对每个光源:
  │  ├─ 计算光照贡献
  │  ├─ 应用阴影
  │  └─ 累加到颜色
  ├─ 写入深度
  └─ 输出最终颜色
```

### 代码示例

```hlsl
// 前向渲染着色器
float4 MainPS(FPixelInput Input) : SV_Target0
{
    float3 BaseColor = Texture.Sample(Sampler, Input.UV).rgb;
    float3 Normal = normalize(NormalTexture.Sample(Sampler, Input.UV).rgb * 2.0 - 1.0);
    
    float3 FinalColor = float3(0, 0, 0);
    
    // 对每个光源计算
    for (int i = 0; i < NumLights; ++i) {
        float3 LightDir = normalize(LightPositions[i] - Input.WorldPos);
        float3 LightColor = LightColors[i];
        
        // 计算光照
        float3 Lighting = CalculateLighting(
            BaseColor, Normal, LightDir, LightColor
        );
        
        // 应用阴影
        float Shadow = GetShadow(Input.WorldPos, i);
        
        FinalColor += Lighting * Shadow;
    }
    
    return float4(FinalColor, 1.0);
}
```

## 光源处理

### 光源分类

#### 主光源（Main Light）
- 通常是方向光
- 总是计算
- 支持阴影

#### 附加光源（Additional Lights）
- 点光源和聚光灯
- 有条件计算
- 可能没有阴影

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

## 阴影处理

### 阴影贴图

```hlsl
// 从光源视角渲染深度
float ShadowDepth = ShadowMap.Sample(ShadowCoord).r;
float CurrentDepth = ShadowCoord.z;

// 比较深度
float Shadow = CurrentDepth > ShadowDepth ? 0.0 : 1.0;
```

### 级联阴影贴图

```hlsl
// 根据距离选择级联
uint Cascade = 0;
if (ViewDepth > CascadeDistances[0]) Cascade = 1;
if (ViewDepth > CascadeDistances[1]) Cascade = 2;

// 采样对应级联
float Shadow = ShadowMaps[Cascade].Sample(ShadowCoord).r;
```

## 透明物体处理

### 透明度混合

```hlsl
// 透明物体着色器
float4 MainPS(FPixelInput Input) : SV_Target0
{
    float4 Color = Texture.Sample(Sampler, Input.UV);
    
    // 计算光照
    float3 Lighting = CalculateLighting(...);
    
    // 应用透明度
    return float4(Lighting, Color.a);
}
```

### 混合模式

| 模式 | 公式 | 用途 |
|------|------|------|
| Alpha Blend | Src * SrcAlpha + Dst * (1 - SrcAlpha) | 标准透明 |
| Additive | Src + Dst | 光效、火焰 |
| Modulate | Src * Dst | 阴影、变暗 |
| Multiply | Src * Dst | 调制 |

## 性能优化

### 光源数量限制

```cpp
// 限制光源数量
const int MaxLights = 4;  // 前向渲染通常限制4个

// 根据距离排序光源
SortLightsByDistance(Lights, CameraPos);

// 只使用最近的N个光源
for (int i = 0; i < MaxLights && i < Lights.Num(); ++i) {
    // 计算光照
}
```

### 早期深度测试

```cpp
// 启用Early-Z
r.EarlyZPass = 1;

// 先渲染不透明物体
// 再渲染透明物体
```

### 批处理

```cpp
// 合并相同材质的物体
r.AllowDynamicBatching = 1;

// 减少绘制调用
```

## 前向+渲染优化（Forward+）

### 概念

结合前向和延迟的优点。

**原理**:
1. 使用瓦片光照剔除（延迟）
2. 对每个瓦片计算相关光源
3. 使用前向渲染（前向）

### 优点

- 支持多个光源
- 支持透明物体
- 支持复杂材质
- 性能较好

### 实现

```hlsl
// 瓦片大小
#define TILE_SIZE 16

// 计算瓦片索引
uint2 TileCoord = DispatchThreadID.xy / TILE_SIZE;

// 获取该瓦片的光源列表
uint LightCount = TileLightCount[TileCoord];
for (uint i = 0; i < LightCount; ++i) {
    uint LightIndex = TileLightList[TileCoord * MAX_LIGHTS_PER_TILE + i];
    // 计算光照
}
```

## 常见问题

### 问题1: 光源过多导致性能下降

**症状**: 多个光源时FPS下降明显

**原因**: 
- 光源数量超过限制
- 没有进行光源剔除

**解决**:
```cpp
// 限制光源数量
const int MaxLights = 4;

// 进行光源剔除
GetRelevantLights(Primitive, RelevantLights);
```

### 问题2: 透明物体排序错误

**症状**: 透明物体显示顺序不对

**原因**: 
- 没有按深度排序
- 混合模式错误

**解决**:
```cpp
// 按深度排序透明物体
SortTransparentObjects(Objects, CameraPos);

// 设置正确的混合模式
Material->SetBlendMode(BLEND_Translucent);
```

### 问题3: 阴影伪影

**症状**: 阴影边界出现锯齿或条纹

**原因**: 
- 阴影贴图分辨率低
- 阴影偏差设置不当

**解决**:
```cpp
// 增加阴影贴图分辨率
Light->ShadowMapResolution = 2048;

// 调整阴影偏差
Light->ShadowBias = 0.01f;
```

## 最佳实践

1. **选择合适的渲染方式**
   - 少量光源 → 前向渲染
   - 多个光源 → 延迟渲染
   - 混合 → Forward+

2. **优化光源数量**
   - 限制动态光源
   - 使用烘焙光照
   - 进行光源剔除

3. **处理透明物体**
   - 按深度排序
   - 使用正确的混合模式
   - 避免过度透明

4. **性能监控**
   - 使用Stat命令
   - 分析光源成本
   - 优化关键路径

## 参考资源

- [UE前向渲染文档](https://docs.unrealengine.com/en-US/RenderingAndGraphics/ForwardRendering/)
- [光照文档](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Lighting/)
- [透明度处理](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Materials/HowTo/Transparency/)
