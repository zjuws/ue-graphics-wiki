# 写实渲染案例

## 写实渲染概述

写实渲染追求照片级别的真实感，需要精确的物理模拟和高质量的资产。

## 角色渲染

### 皮肤材质

**次表面散射**:
```hlsl
// 皮肤材质着色器
void MainPS(
    in float2 UV : TEXCOORD0,
    in float3 Normal : TEXCOORD1,
    in float3 ViewDir : TEXCOORD2,
    out float4 OutColor : SV_Target0)
{
    // 皮肤基础颜色
    float3 BaseColor = Texture.Sample(Sampler, UV).rgb;
    
    // 法线贴图
    float3 NormalMap = NormalTexture.Sample(Sampler, UV).rgb * 2.0 - 1.0;
    Normal = normalize(Normal + NormalMap * 0.3);
    
    // 粗糙度贴图
    float Roughness = RoughnessTexture.Sample(Sampler, UV).r;
    
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

**参数设置**:
```
Base Color: 肤色贴图
Normal: 法线贴图
Roughness: 0.3-0.5
Subsurface Color: RGB(1.0, 0.3, 0.2)
```

### 眼睛渲染

**角膜和虹膜**:
```hlsl
// 眼睛着色器
float3 CalculateEye(float2 UV, float3 Normal, float3 ViewDir)
{
    // 角膜反射
    float3 Cornea = float3(1, 1, 1);
    float3 Reflection = reflect(ViewDir, Normal);
    float3 EnvReflection = EnvironmentTexture.Sample(Sampler, Reflection).rgb;
    
    // 虹膜
    float3 IrisColor = IrisTexture.Sample(Sampler, UV).rgb;
    
    // 瞳孔
    float PupilSize = 0.3;
    float Dist = length(UV - float2(0.5, 0.5));
    float Pupil = smoothstep(PupilSize, PupilSize * 0.9, Dist);
    
    // 混合
    float3 EyeColor = lerp(IrisColor, float3(0, 0, 0), Pupil);
    EyeColor += EnvReflection * 0.3;
    
    return EyeColor;
}
```

### 头发渲染

**Kajiya-Kay模型**:
```hlsl
// 头发着色器
float3 CalculateHair(float3 Tangent, float3 Normal, float3 ViewDir, float3 LightDir)
{
    // 切线空间光照
    float3 T = normalize(Tangent);
    float3 B = normalize(cross(Normal, T));
    
    // 镜面反射
    float3 H = normalize(LightDir + ViewDir);
    float TdotH = dot(T, H);
    float sinTH = sqrt(1.0 - TdotH * TdotH);
    
    float Specular = pow(sinTH, SpecularPower);
    
    // 双向高光
    float3 Specular1 = SpecularColor1 * Specular;
    float3 Specular2 = SpecularColor2 * Specular * shift(T, 0.5);
    
    // 漫反射
    float3 Diffuse = BaseColor * max(dot(Normal, LightDir), 0.0);
    
    return Diffuse + Specular1 + Specular2;
}
```

## 环境渲染

### 室内场景

**光照设置**:
```cpp
// 主光源（窗户光）
DirectionalLight->SetIntensity(3.0f);
DirectionalLight->SetLightColor(FLinearColor(1.0, 0.95, 0.9));  // 温暖

// 环境光
SkyLight->SetIntensity(0.5f);
SkyLight->SetLightColor(FLinearColor(0.7, 0.8, 1.0));  // 冷色

// 烘焙光照
LightmassSettings.NumIndirectLightingBounces = 5;
LightmassSettings.IndirectLightingQuality = 3.0;
```

**材质**:
```
墙壁: 
- Base Color: 贴图
- Normal: 法线贴图
- Roughness: 0.8
- Metallic: 0.0

地板:
- Base Color: 贴图
- Normal: 法线贴图
- Roughness: 0.3
- Metallic: 0.0
```

### 室外场景

**天空和大气**:
```cpp
// 大气设置
AtmosphericFog->SetDensity(0.001f);
AtmosphericFog->SetStartDistance(1000.0f);
AtmosphericFog->SetSunDiscScale(1.0f);

// 天空光
SkyLight->SetIntensity(1.0f);
SkyLight->SetLightColor(FLinearColor::White);
```

**地形**:
```hlsl
// 地形着色器
float3 CalculateTerrain(float2 UV, float Slope)
{
    // 根据坡度混合纹理
    float3 GrassTexture = Texture.Sample(Sampler, UV).rgb;
    float3 RockTexture = RockTexture.Sample(Sampler, UV).rgb;
    
    float GrassBlend = smoothstep(0.3, 0.7, 1.0 - Slope);
    
    float3 FinalColor = lerp(RockTexture, GrassTexture, GrassBlend);
    
    return FinalColor;
}
```

## 材质库

### 金属材质

**钢铁**:
```
Base Color: RGB(0.5, 0.5, 0.5)
Metallic: 1.0
Roughness: 0.3-0.5
```

**铜**:
```
Base Color: RGB(0.9, 0.5, 0.3)
Metallic: 1.0
Roughness: 0.2-0.3
```

**金**:
```
Base Color: RGB(1.0, 0.8, 0.0)
Metallic: 1.0
Roughness: 0.1-0.2
```

### 非金属材质

**木头**:
```
Base Color: 贴图
Metallic: 0.0
Roughness: 0.5-0.7
```

**布料**:
```
Base Color: 贴图
Metallic: 0.0
Roughness: 0.8-1.0
```

**石头**:
```
Base Color: 贴图
Metallic: 0.0
Roughness: 0.8-0.9
```

## 性能优化

### LOD配置

```cpp
// 角色LOD
// LOD0: 0-200m, 100%顶点
// LOD1: 200-500m, 50%顶点
// LOD2: 500-1000m, 25%顶点
// LOD3: 1000m+, 10%顶点
```

### 材质优化

```cpp
// 简化材质
// 远处使用更简单的材质
if (Distance > 2000) {
    // 禁用次表面散射
    // 降低法线贴图强度
    // 使用更简单的光照
}
```

## 最佳实践

1. **使用PBR工作流** - 确保物理正确
2. **高质量资产** - 使用高分辨率贴图
3. **合理的光照** - 平衡实时光照和烘焙
4. **性能优化** - 使用LOD和材质简化
5. **测试** - 在目标平台上测试

## 参考资源

- [UE写实渲染文档](https://docs.unrealengine.com/en-US/RenderingAndGraphics/RealisticRendering/)
- [角色渲染](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Characters/)
