# 风格化渲染案例

## 风格化渲染概述

风格化渲染追求非写实的艺术效果，如卡通、手绘、像素艺术等。

## 卡通渲染

### Cel Shading（卡通着色）

**原理**: 将光照分为几个离散的层级

```hlsl
// 卡通着色器
float3 CelShading(float3 BaseColor, float NdotL, float3 ShadowColor)
{
    // 阶梯化光照
    float ToonShade;
    if (NdotL > 0.8) ToonShade = 1.0;
    else if (NdotL > 0.4) ToonShade = 0.7;
    else if (NdotL > 0.0) ToonShade = 0.4;
    else ToonShade = 0.2;
    
    // 混合阴影颜色
    float3 FinalColor = lerp(ShadowColor * BaseColor, BaseColor, ToonShade);
    
    return FinalColor;
}
```

### 描边效果

**方法1: 背面扩展法**:
```hlsl
// 描边着色器（背面渲染）
float4 OutlineVS(float4 Position : POSITION) : SV_POSITION
{
    float3 Normal = mul((float3x3)WorldMatrix, Input.Normal);
    float4 WorldPos = mul(Position, WorldMatrix);
    
    // 沿法线扩展
    WorldPos.xyz += Normal * OutlineWidth;
    
    return mul(WorldPos, ViewProjection);
}

// 描边像素着色器
float4 OutlinePS() : SV_Target
{
    return OutlineColor;
}
```

**方法2: Sobel滤波**:
```hlsl
// Sobel描边
float3 SobelOutline(float2 UV)
{
    float3 Samples[9];
    // 采样周围像素
    for (int i = 0; i < 9; ++i) {
        Samples[i] = SceneTexture.Sample(Sampler, UV + Offsets[i]).rgb;
    }
    
    // Sobel算子
    float3 Horizontal = Samples[0] + 2 * Samples[1] + Samples[2] 
                       - Samples[6] - 2 * Samples[7] - Samples[8];
    float3 Vertical = Samples[0] + 2 * Samples[3] + Samples[6] 
                     - Samples[2] - 2 * Samples[5] - Samples[8];
    
    float3 Edge = sqrt(Horizontal * Horizontal + Vertical * Vertical);
    
    return Edge;
}
```

## 手绘风格

### 素描效果

```hlsl
// 素描着色器
float3 SketchEffect(float3 BaseColor, float NdotL, float2 UV)
{
    // 铅笔纹理
    float PencilTexture = Texture.Sample(Sampler, UV).r;
    
    // 根据光照调整密度
    float Density = 1.0 - NdotL;
    
    // 混合
    float3 SketchColor = float3(PencilTexture * Density, 
                                PencilTexture * Density, 
                                PencilTexture * Density);
    
    return SketchColor;
}
```

### 水彩效果

```hlsl
// 水彩着色器
float3 WatercolorEffect(float3 BaseColor, float2 UV, float Time)
{
    // 纸张纹理
    float PaperTexture = PaperTexture.Sample(Sampler, UV).r;
    
    // 颜色扩散
    float2 DistortedUV = UV + float2(
        sin(UV.y * 20.0 + Time) * 0.01,
        cos(UV.x * 20.0 + Time) * 0.01
    );
    
    float3 DistortedColor = BaseColor * PaperTexture;
    
    return DistortedColor;
}
```

## 像素艺术

### 像素化效果

```hlsl
// 像素化着色器
float3 PixelArt(float2 UV, float2 ScreenSize, float PixelSize)
{
    // 计算像素网格
    float2 PixelUV = floor(UV * ScreenSize / PixelSize) * PixelSize / ScreenSize;
    
    // 量化颜色
    float3 Color = Texture.Sample(Sampler, PixelUV).rgb;
    Color = floor(Color * ColorLevels) / ColorLevels;
    
    return Color;
}
```

### 有限调色板

```hlsl
// 调色板映射
float3 MapToPalette(float3 Color, float3 Palette[16])
{
    float MinDist = 1000.0;
    float3 NearestColor = Palette[0];
    
    for (int i = 0; i < 16; ++i) {
        float Dist = distance(Color, Palette[i]);
        if (Dist < MinDist) {
            MinDist = Dist;
            NearestColor = Palette[i];
        }
    }
    
    return NearestColor;
}
```

## 低多边形风格

### 平面着色

```hlsl
// 平面着色（不插值法线）
// 在顶点着色器中计算法线
float3 FaceNormal = cross(
    v1.Position - v0.Position,
    v2.Position - v0.Position
);

// 平面着色
float3 FlatShading(float3 BaseColor, float3 FaceNormal, float3 LightDir)
{
    float NdotL = max(dot(normalize(FaceNormal), LightDir), 0.0);
    return BaseColor * NdotL;
}
```

### 颜色渐变

```hlsl
// 根据高度渐变颜色
float3 GradientColor(float Height, float3 Colors[5], float Heights[5])
{
    for (int i = 0; i < 4; ++i) {
        if (Height >= Heights[i] && Height < Heights[i + 1]) {
            float t = (Height - Heights[i]) / (Heights[i + 1] - Heights[i]);
            return lerp(Colors[i], Colors[i + 1], t);
        }
    }
    return Colors[4];
}
```

## 性能优化

### 简化计算

```hlsl
// 风格化渲染通常比写实渲染更快
// 可以使用更简单的计算

// 不需要复杂的PBR
float3 SimpleDiffuse = BaseColor * NdotL;

// 不需要复杂的高光
float3 SimpleSpecular = SpecularColor * pow(NdotH, SpecularPower);
```

### 分辨率调整

```cpp
// 可以使用较低的分辨率
r.ScreenPercentage = 75;  // 对于像素艺术效果很好
```

## 最佳实践

1. **明确艺术方向** - 确定风格目标
2. **简化光照** - 使用阶梯化或简化模型
3. **后处理** - 使用后处理增强效果
4. **性能优势** - 利用风格化的性能优势
5. **一致性** - 保持整个项目的风格一致

## 参考资源

- [UE风格化渲染](https://docs.unrealengine.com/en-US/RenderingAndGraphics/StylizedRendering/)
- [卡通渲染](https://docs.unrealengine.com/en-US/RenderingAndGraphics/StylizedRendering/ToonShading/)
