# 物理基础渲染 (PBR)

## PBR概述

物理基础渲染（Physically Based Rendering, PBR）是一种基于物理原理的渲染方法，确保材质在不同光照条件下看起来真实一致。

## PBR工作流

### 金属工作流（Metallic Workflow）

这是UE默认使用的工作流。

#### 关键参数

| 参数 | 范围 | 说明 |
|------|------|------|
| Base Color | 0-1 | 基础颜色 |
| Metallic | 0-1 | 0=非金属，1=金属 |
| Roughness | 0-1 | 0=光滑，1=粗糙 |
| Normal | -1 to 1 | 表面法线细节 |
| AO | 0-1 | 环境光遮挡 |

#### 工作流图

```
纹理资源
  ├─ Base Color Map
  ├─ Normal Map
  ├─ Roughness Map
  ├─ Metallic Map
  └─ AO Map
       │
       ▼
  材质编辑器
       │
       ▼
  PBR计算
       │
       ▼
  最终渲染
```

### 金属度（Metallic）

**非金属（Metallic = 0）**:
- 漫反射主导
- 镜面反射弱
- 例：木头、布料、石头

**金属（Metallic = 1）**:
- 镜面反射主导
- 无漫反射
- 例：铁、铜、金

**混合（0 < Metallic < 1）**:
- 不推荐使用
- 会产生不真实的效果

### 粗糙度（Roughness）

**光滑（Roughness = 0）**:
- 镜面反射清晰
- 高光集中
- 例：抛光金属、镜子

**粗糙（Roughness = 1）**:
- 镜面反射分散
- 高光模糊
- 例：布料、混凝土

## 法线贴图

### 法线贴图格式

**RGB通道**:
- R: X方向（红色）
- G: Y方向（绿色）
- B: Z方向（蓝色）

**值范围**:
- 0 = -1
- 128 = 0
- 255 = 1

### 法线贴图类型

#### 切线空间（Tangent Space）
- 相对于表面
- 可重用
- UE默认格式

```hlsl
// 在材质中使用
float3 Normal = Texture.Sample(UV).rgb * 2.0 - 1.0;
```

#### 世界空间（World Space）
- 绝对坐标
- 不可重用
- 用于特殊效果

### 法线贴图强度

```hlsl
// 控制法线强度
float3 Normal = Texture.Sample(UV).rgb * 2.0 - 1.0;
Normal.xy *= NormalStrength;  // 调整强度
Normal = normalize(Normal);
```

## 环境光遮挡（AO）

### AO贴图

- 预计算的阴影信息
- 存储在纹理中
- 提高细节感

### 应用方法

```hlsl
// 方法1：乘法混合
float3 FinalColor = BaseColor * AO;

// 方法2：与光照结合
float3 FinalColor = BaseColor * (1.0 + AO * AOStrength);

// 方法3：仅在阴影中应用
float3 FinalColor = BaseColor * lerp(AO, 1.0, LightIntensity);
```

## 自发光（Emissive）

### 基础自发光

```hlsl
// 简单自发光
float3 Emissive = BaseColor * EmissiveIntensity;
```

### 动态自发光

```hlsl
// 基于时间的闪烁
float Flicker = sin(Time * Frequency) * 0.5 + 0.5;
float3 Emissive = BaseColor * EmissiveIntensity * Flicker;
```

### 自发光贴图

```hlsl
// 使用贴图控制自发光
float3 EmissiveMap = EmissiveTexture.Sample(UV).rgb;
float3 Emissive = EmissiveMap * EmissiveIntensity;
```

## 常见材质示例

### 金属材质

```hlsl
// 铁
Base Color: 灰色 (0.5, 0.5, 0.5)
Metallic: 1.0
Roughness: 0.3-0.5

// 铜
Base Color: 橙红色 (0.9, 0.5, 0.3)
Metallic: 1.0
Roughness: 0.2-0.3

// 金
Base Color: 金黄色 (1.0, 0.8, 0.0)
Metallic: 1.0
Roughness: 0.1-0.2
```

### 非金属材质

```hlsl
// 木头
Base Color: 棕色 (0.6, 0.4, 0.2)
Metallic: 0.0
Roughness: 0.5-0.7

// 布料
Base Color: 颜色变化
Metallic: 0.0
Roughness: 0.8-1.0

// 石头
Base Color: 灰色 (0.5, 0.5, 0.5)
Metallic: 0.0
Roughness: 0.8-0.9
```

### 皮肤材质

```hlsl
// 人类皮肤
Base Color: 肤色 (0.9, 0.7, 0.6)
Metallic: 0.0
Roughness: 0.3-0.4
Subsurface Color: 红色 (1.0, 0.3, 0.2)
```

## PBR最佳实践

### 1. 使用真实世界参考

- 测量真实材质的参数
- 参考照片和参考模型
- 保持一致性

### 2. 纹理分辨率

| 用途 | 分辨率 | 说明 |
|------|--------|------|
| 主角 | 2K-4K | 高质量 |
| 环境 | 1K-2K | 平衡 |
| 远景 | 512-1K | 低质量 |

### 3. 法线贴图质量

```hlsl
// 检查法线贴图质量
// 在材质编辑器中启用法线可视化
r.VisualizeNormalMap 1
```

### 4. 避免常见错误

| 错误 | 原因 | 解决 |
|------|------|------|
| 金属度0-1混合 | 不物理 | 使用0或1 |
| 过度光滑 | 不真实 | 增加粗糙度 |
| 缺少AO | 平面感 | 添加AO贴图 |
| 法线过强 | 不协调 | 调整法线强度 |

## 性能考虑

### 纹理采样优化

```hlsl
// 不好：多次采样
float3 Normal = NormalTexture.Sample(UV).rgb;
float Roughness = RoughnessTexture.Sample(UV).r;
float Metallic = MetallicTexture.Sample(UV).r;

// 好：打包纹理
float4 Packed = PackedTexture.Sample(UV);
float3 Normal = Packed.rgb;
float Roughness = Packed.a;
float Metallic = PackedTexture2.Sample(UV).r;
```

### 贴图打包

常见打包方案：
- **RGB**: 法线XYZ
- **A**: 粗糙度或AO

## 参考资源

- [UE PBR文档](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Materials/PhysicallyBased/)
- [Substance 3D Designer](https://www.adobe.com/products/substance/designer.html)
- [Quixel Megascans](https://quixel.com/megascans)
