# 材质基础

## 材质系统概述

UE的材质系统是一个基于节点的着色器编辑器，允许美术师和程序员创建复杂的视觉效果而无需编写代码。

## 材质编辑器界面

### 主要区域

```
┌─────────────────────────────────────────────────────┐
│ 工具栏 (Toolbar)                                     │
├─────────────────────────────────────────────────────┤
│ 左侧面板          │ 中央编辑区        │ 右侧面板    │
│ - 详情面板        │ - 节点图          │ - 预览      │
│ - 参数列表        │ - 连接线          │ - 统计信息  │
│ - 函数库          │ - 主材质节点      │ - 性能      │
└─────────────────────────────────────────────────────┘
```

### 关键快捷键

| 快捷键 | 功能 |
|--------|------|
| 1 | 添加节点 |
| 2 | 连接节点 |
| Delete | 删除节点 |
| Ctrl+Z | 撤销 |
| Ctrl+Y | 重做 |
| F | 聚焦选中节点 |
| Home | 聚焦所有节点 |

## 材质输入

### 主材质节点（Main Material Node）

```
┌─────────────────────────────┐
│      Main Material Node     │
├─────────────────────────────┤
│ Base Color                  │
│ Normal                      │
│ Metallic                    │
│ Specular                    │
│ Roughness                   │
│ Ambient Occlusion           │
│ Emissive Color              │
│ Opacity                     │
│ Opacity Mask                │
│ World Position Offset       │
│ Subsurface Color            │
│ Custom Data 0-3             │
└─────────────────────────────┘
```

### 常用输入说明

| 输入 | 范围 | 说明 |
|------|------|------|
| Base Color | 0-1 | 基础颜色 |
| Normal | -1 to 1 | 法线贴图 |
| Metallic | 0-1 | 金属度 |
| Specular | 0-1 | 镜面反射强度 |
| Roughness | 0-1 | 粗糙度 |
| AO | 0-1 | 环境光遮挡 |
| Emissive | 0+ | 自发光 |
| Opacity | 0-1 | 透明度 |

## 常用节点

### 纹理节点

#### Texture Sample
采样纹理。

```
输入:
  - Coordinates (UV)
  - MipLevel
  - MipBias

输出:
  - Color (RGBA)
  - R, G, B, A (单通道)
```

#### Texture Object
引用纹理资源。

```cpp
// 在材质中使用
Texture2D MyTexture;
float4 Color = MyTexture.Sample(SamplerState, UV);
```

### 数学节点

#### Add
加法。

```
A + B
```

#### Multiply
乘法。

```
A * B
```

#### Lerp
线性插值。

```
lerp(A, B, Alpha) = A + (B - A) * Alpha
```

#### Clamp
限制范围。

```
clamp(Value, Min, Max)
```

### 向量节点

#### Append
合并向量分量。

```
Append(R, G, B, A) = float4(R, G, B, A)
```

#### ComponentMask
提取向量分量。

```
ComponentMask(Vector, R, G, B, A)
```

### 坐标节点

#### Texture Coordinate
获取UV坐标。

```
输出: UV0, UV1, UV2, UV3
```

#### World Position
获取世界坐标。

```
输出: X, Y, Z, W
```

#### Camera Position
获取摄像机位置。

```
输出: 摄像机世界坐标
```

## 材质类型

### Opaque（不透明）
- 完全不透明
- 性能最优
- 支持所有功能

### Masked（遮罩）
- 二值透明（有或无）
- 使用Opacity Mask
- 适合树叶、栅栏等

```hlsl
// 在材质中
Opacity Mask = Texture.A > 0.5 ? 1.0 : 0.0;
```

### Translucent（半透明）
- 支持半透明
- 性能消耗较大
- 需要排序

```hlsl
// 在材质中
Opacity = 0.5;  // 50%透明
```

### Additive（加法）
- 加法混合
- 适合光效、火焰

```hlsl
// 最终颜色 = 场景颜色 + 材质颜色
```

### Modulate（调制）
- 调制混合
- 适合阴影、变暗效果

```hlsl
// 最终颜色 = 场景颜色 * 材质颜色
```

## 材质参数

### 标量参数（Scalar Parameter）

```hlsl
// 定义
float MyValue = 1.0;

// 在代码中修改
Material->SetScalarParameterValue(FName("MyValue"), 0.5f);
```

### 向量参数（Vector Parameter）

```hlsl
// 定义
float3 MyColor = float3(1, 0, 0);

// 在代码中修改
Material->SetVectorParameterValue(FName("MyColor"), FLinearColor::Red);
```

### 纹理参数（Texture Parameter）

```hlsl
// 定义
Texture2D MyTexture;

// 在代码中修改
Material->SetTextureParameterValue(FName("MyTexture"), NewTexture);
```

## 材质实例

### 动态材质实例（Dynamic Material Instance）

```cpp
// 创建
UMaterialInstanceDynamic* MatInstance = 
    UMaterialInstanceDynamic::Create(BaseMaterial, this);

// 应用到组件
MeshComponent->SetMaterial(0, MatInstance);

// 修改参数
MatInstance->SetScalarParameterValue(FName("Roughness"), 0.5f);
MatInstance->SetVectorParameterValue(FName("BaseColor"), FLinearColor::Red);
```

### 材质实例常数（Material Instance Constant）

- 在编辑器中创建
- 参数值固定
- 性能更优

## 最佳实践

### 1. 使用材质函数复用代码

```hlsl
// 创建材质函数
MaterialFunction MyFunction(float Input)
{
    return Input * 2.0;
}

// 在材质中使用
float Result = MyFunction(0.5);
```

### 2. 优化采样次数

```hlsl
// 不好：多次采样同一纹理
float3 Color1 = Texture.Sample(UV);
float3 Color2 = Texture.Sample(UV);

// 好：采样一次，复用结果
float3 Color = Texture.Sample(UV);
float3 Color1 = Color;
float3 Color2 = Color;
```

### 3. 使用参数化设计

```hlsl
// 不好：硬编码值
float Roughness = 0.5;

// 好：使用参数
float Roughness = RoughnessParameter;
```

### 4. 性能考虑

| 操作 | 成本 | 建议 |
|------|------|------|
| 纹理采样 | 高 | 限制采样次数 |
| 数学运算 | 低 | 可自由使用 |
| 分支 | 中 | 避免复杂分支 |
| 导数 | 中 | 避免在分支中使用 |

## 常见问题

**Q: 如何创建闪烁效果？**  
A: 使用Time节点和Sine函数。

```hlsl
float Flicker = sin(Time * Frequency) * 0.5 + 0.5;
Emissive = BaseColor * Flicker;
```

**Q: 如何实现UV动画？**  
A: 修改纹理坐标。

```hlsl
float2 AnimatedUV = UV + float2(Time * Speed, 0);
Color = Texture.Sample(AnimatedUV);
```

**Q: 材质太复杂，性能下降？**  
A: 
- 减少纹理采样
- 使用材质函数优化
- 考虑使用烘焙纹理

## 参考资源

- [UE材质文档](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Materials/)
- [材质编辑器指南](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Materials/Editor/)
