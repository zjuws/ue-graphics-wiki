# 纹理映射技术

## 纹理映射概述

纹理映射是将2D纹理应用到3D表面的技术。不同的映射方式适合不同的场景。

## UV映射

### 基本概念

UV坐标是2D纹理坐标系统，范围通常为0-1。

```hlsl
// 在像素着色器中使用UV
float2 UV = Input.TexCoord;
float4 Color = Texture.Sample(Sampler, UV);
```

### UV展开

**原理**: 将3D模型表面展开到2D平面

**工具**:
- Substance 3D Painter
- Marmoset Toolbag
- xNormal

### UV优化

```cpp
// 检查UV重叠
// 在编辑器中使用UV编辑器

// 优化UV布局
// 1. 最大化纹理空间利用率
// 2. 避免重叠
// 3. 保持一致的密度
```

## 法线贴图映射

### 法线贴图类型

#### 切线空间法线贴图
- 相对于表面
- 可重用
- UE默认格式

```hlsl
// 应用切线空间法线贴图
float3 NormalTS = Texture.Sample(Sampler, UV).rgb * 2.0 - 1.0;
float3 NormalWS = mul(NormalTS, TBN);
```

#### 世界空间法线贴图
- 绝对坐标
- 不可重用
- 特殊效果

### TBN矩阵构建

```hlsl
// 构建TBN矩阵
float3 T = normalize(Tangent);
float3 B = normalize(Bitangent);
float3 N = normalize(Normal);

float3x3 TBN = float3x3(T, B, N);

// 应用法线贴图
float3 NormalTS = Texture.Sample(Sampler, UV).rgb * 2.0 - 1.0;
float3 NormalWS = mul(NormalTS, TBN);
```

## 视差映射

### 基本视差映射

```hlsl
// 视差映射
float2 ParallaxMapping(float2 UV, float3 ViewDir, float HeightScale)
{
    float Height = HeightTexture.Sample(Sampler, UV).r;
    float2 Offset = ViewDir.xy * (Height * HeightScale);
    return UV - Offset;
}
```

### 陡峭视差映射

```hlsl
// 陡峭视差映射（更准确）
float2 SteepParallaxMapping(float2 UV, float3 ViewDir, float HeightScale)
{
    float2 P = ViewDir.xy * HeightScale;
    float2 CurrentUV = UV;
    float CurrentHeight = 1.0;
    float CurrentDepth = 0.0;
    
    for (int i = 0; i < NumSteps; ++i) {
        CurrentDepth += 1.0 / NumSteps;
        float SampleHeight = HeightTexture.Sample(Sampler, CurrentUV).r;
        
        if (CurrentDepth > SampleHeight) {
            CurrentUV -= P / NumSteps;
        }
    }
    
    return CurrentUV;
}
```

## 三平面映射

### 原理

在三个方向上采样纹理，根据法线混合。

```hlsl
// 三平面映射
float3 TriplanarMapping(float3 WorldPos, float3 Normal)
{
    // 三个方向的纹理采样
    float3 ColorX = Texture.Sample(Sampler, WorldPos.yz).rgb;
    float3 ColorY = Texture.Sample(Sampler, WorldPos.xz).rgb;
    float3 ColorZ = Texture.Sample(Sampler, WorldPos.xy).rgb;
    
    // 根据法线混合
    float3 BlendWeights = abs(Normal);
    BlendWeights /= (BlendWeights.x + BlendWeights.y + BlendWeights.z);
    
    return ColorX * BlendWeights.x + ColorY * BlendWeights.y + ColorZ * BlendWeights.z;
}
```

## 投影映射

### 平面投影

```hlsl
// 平面投影映射
float2 PlanarProjection(float3 WorldPos, float3 ProjectionDir)
{
    float2 UV = float2(
        dot(WorldPos, ProjectionDir),
        dot(WorldPos, cross(ProjectionDir, float3(0, 0, 1)))
    );
    return UV;
}
```

## 高级映射技术

### 细节映射

```hlsl
// 细节映射
float3 DetailMapping(float2 UV, float3 BaseColor)
{
    float3 DetailColor = DetailTexture.Sample(Sampler, UV * DetailScale).rgb;
    return BaseColor * DetailColor * 2.0;
}
```

### 分层映射

```hlsl
// 分层映射
float3 LayeredMapping(float2 UV, float BlendFactor)
{
    float3 Layer1 = Texture1.Sample(Sampler, UV).rgb;
    float3 Layer2 = Texture2.Sample(Sampler, UV).rgb;
    return lerp(Layer1, Layer2, BlendFactor);
}
```

## 性能优化

### 纹理坐标优化

```hlsl
// 不好：多次计算UV
float2 UV1 = ComputeUV(Input.Position);
float2 UV2 = ComputeUV(Input.Position);

// 好：计算一次
float2 UV = ComputeUV(Input.Position);
float2 UV1 = UV;
float2 UV2 = UV;
```

### 采样优化

```hlsl
// 不好：多次采样
float3 Color1 = Texture.Sample(Sampler, UV);
float3 Color2 = Texture.Sample(Sampler, UV);

// 好：采样一次
float3 Color = Texture.Sample(Sampler, UV);
float3 Color1 = Color;
float3 Color2 = Color;
```

## 常见问题

### 问题1: UV接缝

**症状**: 纹理接缝明显

**原因**: 
- UV展开不当
- 纹理边界处理不当

**解决**:
```hlsl
// 使用边界处理
float2 UV = frac(Input.TexCoord);  // 重复
float2 UV = clamp(Input.TexCoord, 0, 1);  // 钳制
```

### 问题2: 纹理扭曲

**症状**: 纹理在某些区域扭曲

**原因**: 
- UV密度不一致
- 法线贴图应用错误

**解决**:
```cpp
// 优化UV布局
// 确保UV密度一致
// 检查法线贴图方向
```

## 最佳实践

1. **优化UV布局** - 最大化利用率
2. **一致的密度** - 保持纹理密度一致
3. **避免重叠** - 除非有意
4. **测试接缝** - 检查边界处理
5. **性能监控** - 监控采样成本

## 参考资源

- [UE纹理映射文档](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Textures/TextureMapping/)
- [UV展开指南](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Textures/UVMapping/)
