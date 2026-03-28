# 高级材质技巧

## 高级材质概述

高级材质技巧包括动画、自定义节点、复杂效果等，用于创建专业级的视觉效果。

## 材质动画

### 基于时间的动画

```hlsl
// 使用Time节点创建动画
float Time = GetTime();

// 闪烁效果
float Flicker = sin(Time * Frequency) * 0.5 + 0.5;
float3 Emissive = BaseColor * Flicker;

// 旋转UV
float2 RotatedUV = RotateUV(UV, Time * RotationSpeed);
float3 Color = Texture.Sample(Sampler, RotatedUV).rgb;

// 波浪效果
float Wave = sin(UV.x * Frequency + Time * Speed) * Amplitude;
float2 WavedUV = UV + float2(0, Wave);
float3 Color = Texture.Sample(Sampler, WavedUV).rgb;
```

### 顶点动画

```hlsl
// 顶点着色器中的动画
float4 VS_Main(FVertexInput Input) : SV_POSITION
{
    float3 WorldPos = mul(float4(Input.Position, 1), ObjectToWorld).xyz;
    
    // 风动画
    float Wind = sin(WorldPos.x * 0.01 + Time * WindSpeed) * WindStrength;
    WorldPos.y += Wind;
    
    // 投影
    float4 ProjPos = mul(float4(WorldPos, 1), ViewProjection);
    return ProjPos;
}
```

## 自定义节点

### 创建自定义节点

```cpp
// 在材质编辑器中创建自定义节点
// 1. 右键 → Material Functions → New Material Function
// 2. 编写HLSL代码
// 3. 在材质中使用
```

### 自定义节点示例

```hlsl
// 自定义Fresnel节点
float3 CustomFresnel(float3 Normal, float3 ViewDir, float Power)
{
    float NdotV = max(dot(Normal, -ViewDir), 0.0);
    return pow(1.0 - NdotV, Power);
}

// 自定义噪声节点
float CustomNoise(float2 UV, float Scale)
{
    float2 i = floor(UV * Scale);
    float2 f = frac(UV * Scale);
    
    float a = random(i);
    float b = random(i + float2(1, 0));
    float c = random(i + float2(0, 1));
    float d = random(i + float2(1, 1));
    
    float2 u = f * f * (3.0 - 2.0 * f);
    
    return mix(mix(a, b, u.x), mix(c, d, u.x), u.y);
}
```

## 复杂效果

### 玻璃材质

```hlsl
// 玻璃材质
void MainPS(
    in float2 UV : TEXCOORD0,
    in float3 Normal : TEXCOORD1,
    in float3 ViewDir : TEXCOORD2,
    out float4 OutColor : SV_Target0)
{
    // 基础颜色（略微着色）
    float3 BaseColor = float3(0.9, 0.95, 1.0);
    
    // 法线扰动（玻璃表面不完全光滑）
    float3 NormalNoise = Texture.Sample(Sampler, UV).rgb * 2.0 - 1.0;
    Normal = normalize(Normal + NormalNoise * 0.1);
    
    // 菲涅尔效应
    float Fresnel = pow(1.0 - max(dot(Normal, -ViewDir), 0.0), 5.0);
    
    // 反射和折射混合
    float3 ReflectionColor = GetReflection(Normal, ViewDir);
    float3 RefractionColor = GetRefraction(Normal, ViewDir);
    
    float3 FinalColor = lerp(RefractionColor, ReflectionColor, Fresnel);
    
    OutColor = float4(FinalColor, 0.9);
}
```

### 金属材质

```hlsl
// 高质量金属材质
void MainPS(
    in float2 UV : TEXCOORD0,
    in float3 Normal : TEXCOORD1,
    in float3 ViewDir : TEXCOORD2,
    out float4 OutColor : SV_Target0)
{
    // 金属颜色
    float3 BaseColor = float3(0.9, 0.8, 0.3);  // 金色
    
    // 法线贴图
    float3 NormalMap = Texture.Sample(Sampler, UV).rgb * 2.0 - 1.0;
    Normal = normalize(Normal + NormalMap * 0.5);
    
    // 粗糙度贴图
    float Roughness = RoughnessTexture.Sample(Sampler, UV).r;
    
    // 金属度
    float Metallic = 1.0;
    
    // PBR计算
    float3 Lighting = CalculatePBRLighting(
        BaseColor, Normal, ViewDir, Metallic, Roughness
    );
    
    OutColor = float4(Lighting, 1.0);
}
```

### 皮肤材质

```hlsl
// 皮肤材质（次表面散射）
void MainPS(
    in float2 UV : TEXCOORD0,
    in float3 Normal : TEXCOORD1,
    in float3 ViewDir : TEXCOORD2,
    out float4 OutColor : SV_Target0)
{
    // 皮肤颜色
    float3 BaseColor = Texture.Sample(Sampler, UV).rgb;
    
    // 法线贴图
    float3 NormalMap = NormalTexture.Sample(Sampler, UV).rgb * 2.0 - 1.0;
    Normal = normalize(Normal + NormalMap * 0.3);
    
    // 次表面散射颜色
    float3 SubsurfaceColor = float3(1.0, 0.3, 0.2);
    
    // 计算光照
    float3 LightDir = normalize(LightDirection);
    float NdotL = max(dot(Normal, LightDir), 0.0);
    
    // 次表面散射
    float Backlight = max(dot(Normal, -LightDir), 0.0);
    float3 Subsurface = SubsurfaceColor * Backlight * 0.5;
    
    // 最终颜色
    float3 FinalColor = BaseColor * NdotL + Subsurface;
    
    OutColor = float4(FinalColor, 1.0);
}
```

## 材质函数库

### 常用材质函数

#### Fresnel函数
```hlsl
float3 Fresnel(float3 Normal, float3 ViewDir, float3 F0)
{
    float NdotV = max(dot(Normal, -ViewDir), 0.0);
    return F0 + (1.0 - F0) * pow(1.0 - NdotV, 5.0);
}
```

#### 法线混合
```hlsl
float3 BlendNormals(float3 Normal1, float3 Normal2)
{
    return normalize(Normal1 + Normal2 - float3(0, 0, 1));
}
```

#### 视差映射
```hlsl
float2 ParallaxMapping(float2 UV, float3 ViewDir, float HeightScale)
{
    float Height = HeightTexture.Sample(Sampler, UV).r;
    float2 Offset = ViewDir.xy * (Height * HeightScale);
    return UV - Offset;
}
```

#### 三平面映射
```hlsl
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

## 性能优化

### 材质复杂度

```cpp
// 检查材质复杂度
// 在编辑器中启用材质复杂度可视化
r.ShaderComplexity = 1
```

### 优化建议

```hlsl
// 不好：过多纹理采样
float3 Color1 = Texture1.Sample(Sampler, UV);
float3 Color2 = Texture2.Sample(Sampler, UV);
float3 Color3 = Texture3.Sample(Sampler, UV);
float3 Color4 = Texture4.Sample(Sampler, UV);

// 好：打包纹理
float4 Packed = PackedTexture.Sample(Sampler, UV);
float3 Color1 = Packed.rgb;
float3 Color2 = Packed.aaa;
```

### 材质实例

```cpp
// 使用材质实例而不是动态材质实例
// 性能更好
UMaterialInstance* MatInstance = LoadObject<UMaterialInstance>(nullptr,
    TEXT("MaterialInstance'/Game/Materials/MyMaterial_Inst.MyMaterial_Inst'"));

// 应用到组件
MeshComponent->SetMaterial(0, MatInstance);
```

## 常见问题

### 问题1: 材质看起来不对

**症状**: 材质显示错误或不符合预期

**原因**: 
- 纹理坐标错误
- 法线计算错误
- 参数设置不当

**解决**:
```hlsl
// 检查UV坐标
float3 DebugColor = float3(UV, 0);

// 检查法线
float3 DebugColor = Normal * 0.5 + 0.5;

// 检查参数
float3 DebugColor = float3(Metallic, Roughness, 0);
```

### 问题2: 材质性能差

**症状**: 使用该材质的物体FPS下降

**原因**: 
- 纹理采样过多
- 计算过于复杂

**解决**:
```hlsl
// 减少纹理采样
// 简化计算
// 使用材质函数优化
```

### 问题3: 动画不流畅

**症状**: 材质动画卡顿或闪烁

**原因**: 
- 时间精度不足
- 动画参数不当

**解决**:
```hlsl
// 使用高精度时间
float Time = GetTime();

// 调整动画参数
float Frequency = 2.0;  // 调整频率
float Speed = 1.0;      // 调整速度
```

## 最佳实践

1. **使用材质函数** - 复用代码
2. **优化采样** - 减少纹理采样
3. **使用参数** - 参数化设计
4. **测试性能** - 监控复杂度
5. **文档化** - 记录参数含义

## 参考资源

- [UE材质文档](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Materials/)
- [材质函数](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Materials/Functions/)
- [材质编辑器](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Materials/Editor/)
