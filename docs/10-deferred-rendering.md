# 延迟渲染

## 延迟渲染概述

延迟渲染（Deferred Rendering）是UE的主要渲染方式。它将渲染分为两个主要通道：几何通道和光照通道。

## 延迟渲染vs前向渲染

### 对比表

| 特性 | 延迟渲染 | 前向渲染 |
|------|---------|---------|
| 光源数量 | 支持多个 | 受限 |
| 透明物体 | 需要特殊处理 | 原生支持 |
| MSAA | 困难 | 简单 |
| 内存带宽 | 高 | 低 |
| 小屏幕 | 低效 | 高效 |
| 大屏幕 | 高效 | 低效 |

### 选择建议

**使用延迟渲染**:
- 大量动态光源
- 大屏幕分辨率
- 室内场景

**使用前向渲染**:
- 少量光源
- 透明物体多
- 移动平台

## 延迟渲染管线

### 第一通道：G-Buffer写入

**目的**: 将几何信息写入G-Buffer

**G-Buffer内容**:
```
GBuffer0 (RGBA8):
  R: Base Color R
  G: Base Color G
  B: Base Color B
  A: 材质类型

GBuffer1 (RGBA8):
  R: Normal X
  G: Normal Y
  B: Normal Z
  A: 金属度

GBuffer2 (RGBA8):
  R: 粗糙度
  G: AO
  B: 自发光
  A: 预留

GBuffer3 (RGBA8):
  R: 世界位置X
  G: 世界位置Y
  B: 世界位置Z
  A: 预留

深度缓冲:
  深度值
```

### 第二通道：光照计算

**目的**: 使用G-Buffer数据计算光照

**流程**:
1. 读取G-Buffer
2. 重建世界位置
3. 计算光照贡献
4. 写入最终颜色

### 第三通道：透明物体

**目的**: 使用前向渲染处理透明物体

**原因**: 延迟渲染不支持透明度混合

## G-Buffer详解

### 法线编码

**问题**: 法线是3D向量，但G-Buffer只有8位精度

**解决方案**:
```hlsl
// 编码法线
float3 Normal = normalize(NormalInput);
float2 EncodedNormal = Normal.xy;  // 丢弃Z，因为Z = sqrt(1 - x² - y²)

// 解码法线
float2 EncodedNormal = GBuffer1.xy;
float3 Normal;
Normal.xy = EncodedNormal * 2.0 - 1.0;
Normal.z = sqrt(1.0 - dot(Normal.xy, Normal.xy));
```

### 世界位置重建

**问题**: 存储完整的世界位置占用太多空间

**解决方案**: 从深度重建
```hlsl
// 从深度重建世界位置
float Depth = DepthBuffer.Load(PixelCoord);
float3 ViewPos = ReconstructViewPosition(PixelCoord, Depth);
float3 WorldPos = mul(float4(ViewPos, 1), ViewToWorld).xyz;
```

### 材质类型编码

**用途**: 区分不同的材质类型

```hlsl
// 材质类型
#define MATERIAL_TYPE_DEFAULT 0
#define MATERIAL_TYPE_SUBSURFACE 1
#define MATERIAL_TYPE_PREINTEGRATED_SKIN 2
#define MATERIAL_TYPE_CLEAR_COAT 3
#define MATERIAL_TYPE_CLOTH 4

// 编码
uint MaterialType = MATERIAL_TYPE_DEFAULT;
GBuffer0.a = MaterialType / 255.0;

// 解码
uint MaterialType = round(GBuffer0.a * 255.0);
```

## 光照通道

### 光源类型处理

#### 方向光

```hlsl
// 方向光计算
float3 LightDir = normalize(DirectionalLightDirection);
float3 LightColor = DirectionalLightColor;

float3 Lighting = CalculateLighting(
    BaseColor, Metallic, Roughness, Normal,
    ViewDir, LightDir, LightColor
);
```

#### 点光源

```hlsl
// 点光源计算
float3 LightPos = PointLightPosition;
float3 ToLight = LightPos - WorldPos;
float Distance = length(ToLight);
float3 LightDir = ToLight / Distance;

// 衰减
float Attenuation = 1.0 / (1.0 + (Distance / Radius) * (Distance / Radius));

float3 Lighting = CalculateLighting(
    BaseColor, Metallic, Roughness, Normal,
    ViewDir, LightDir, LightColor * Attenuation
);
```

#### 聚光灯

```hlsl
// 聚光灯计算
float3 LightDir = normalize(SpotLightPosition - WorldPos);
float3 SpotDir = normalize(SpotLightDirection);

// 圆锥衰减
float CosAngle = dot(LightDir, -SpotDir);
float ConeAttenuation = smoothstep(OuterConeAngle, InnerConeAngle, CosAngle);

float3 Lighting = CalculateLighting(...) * ConeAttenuation;
```

### 光照剔除

**问题**: 计算所有光源太昂贵

**解决方案**: 光照剔除
```hlsl
// 计算像素受影响的光源
uint LightCount = 0;
for (uint i = 0; i < TotalLights; ++i) {
    float Distance = length(LightPosition[i] - WorldPos);
    if (Distance < LightRadius[i]) {
        // 计算光照
        Lighting += CalculateLighting(...);
        LightCount++;
    }
}
```

### 瓦片光照剔除（Tiled Deferred）

**原理**: 将屏幕分为瓦片，每个瓦片计算影响它的光源

**优点**:
- 减少光源计算
- 提高缓存效率
- 支持更多光源

**实现**:
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
if (ViewDepth > CascadeDistances[2]) Cascade = 3;

// 采样对应级联的阴影贴图
float Shadow = ShadowMaps[Cascade].Sample(ShadowCoord).r;
```

## 性能优化

### G-Buffer优化

```cpp
// 减少G-Buffer数量
// 默认: 4个RT + 深度
// 优化: 2-3个RT + 深度

// 配置
r.GBufferFormat = 0;  // 标准格式
r.GBufferFormat = 1;  // 优化格式
```

### 光照优化

```cpp
// 限制光源数量
r.DeferredLighting.MaxLights = 256;

// 启用瓦片光照剔除
r.TiledDeferredShading = 1;

// 瓦片大小
r.TiledDeferredShading.TileSize = 16;
```

### 内存优化

```cpp
// 压缩G-Buffer
r.GBufferFormat = 1;  // 使用压缩格式

// 虚拟纹理
r.VirtualTexture.Enable = 1;
```

## 常见问题

### 问题1: 光照条纹

**症状**: 光照出现条纹或带状

**原因**: 
- G-Buffer精度不足
- 法线编码错误

**解决**:
```hlsl
// 使用更高精度的法线编码
float3 Normal = normalize(NormalInput);
// 使用Octahedron编码而不是简单的xy编码
```

### 问题2: 透明物体光照错误

**症状**: 透明物体光照不正确

**原因**: 透明物体使用前向渲染，不能访问G-Buffer

**解决**:
```hlsl
// 在透明物体着色器中手动计算光照
float3 Lighting = CalculateLighting(
    BaseColor, Metallic, Roughness, Normal,
    ViewDir, LightDir, LightColor
);
```

### 问题3: 性能下降

**症状**: 光源多时性能下降明显

**原因**: 
- 光照计算过多
- 内存带宽不足

**解决**:
```cpp
// 启用瓦片光照剔除
r.TiledDeferredShading = 1;

// 限制光源数量
r.DeferredLighting.MaxLights = 128;

// 使用烘焙光照
```

## 最佳实践

1. **合理使用光源** - 限制动态光源数量
2. **启用光照剔除** - 使用瓦片光照剔除
3. **优化G-Buffer** - 使用压缩格式
4. **处理透明物体** - 使用前向渲染
5. **监控性能** - 使用Stat命令分析

## 参考资源

- [UE延迟渲染文档](https://docs.unrealengine.com/en-US/RenderingAndGraphics/DeferredRendering/)
- [G-Buffer详解](https://docs.unrealengine.com/en-US/RenderingAndGraphics/DeferredRendering/Overview/)
- [光照剔除](https://docs.unrealengine.com/en-US/RenderingAndGraphics/DeferredRendering/LightCulling/)
