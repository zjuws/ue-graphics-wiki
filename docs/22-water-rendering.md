# 水体渲染

## 水体渲染概述

水体渲染包括波浪模拟、反射折射、泡沫等效果。

## 波浪模拟

### Gerstner波

```hlsl
// Gerstner波计算
float3 GerstnerWave(float3 Position, float4 Wave, float Time)
{
    float Frequency = Wave.x;
    float Amplitude = Wave.y;
    float Phase = Wave.z;
    float Direction = Wave.w;
    
    float k = 2.0 * 3.14159 / Frequency;
    float c = sqrt(9.8 / k);
    float d = Phase + c * Time;
    
    float x = Direction * k * Position.x + d;
    
    float3 Result = float3(0, 0, 0);
    Result.x = Amplitude * sin(x);
    Result.z = Amplitude * cos(x);
    Result.y = Amplitude * sin(x);
    
    return Result;
}
```

### 多波叠加

```hlsl
// 多波叠加
float3 CalculateWaves(float3 Position, float Time)
{
    float3 TotalDisplacement = float3(0, 0, 0);
    
    for (int i = 0; i < NumWaves; ++i) {
        TotalDisplacement += GerstnerWave(Position, Waves[i], Time);
    }
    
    return TotalDisplacement;
}
```

## 反射和折射

### 反射计算

```hlsl
// 水面反射
float3 CalculateReflection(float3 Normal, float3 ViewDir)
{
    float3 ReflectionDir = reflect(ViewDir, Normal);
    float3 ReflectionColor = EnvironmentTexture.Sample(Sampler, ReflectionDir).rgb;
    
    return ReflectionColor;
}
```

### 折射计算

```hlsl
// 水下折射
float3 CalculateRefraction(float3 Normal, float3 ViewDir, float IOR)
{
    float3 RefractionDir = refract(ViewDir, Normal, 1.0 / IOR);
    float3 RefractionColor = UnderwaterTexture.Sample(Sampler, RefractionDir).rgb;
    
    return RefractionColor;
}
```

### 菲涅尔混合

```hlsl
// 菲涅尔效应
float Fresnel = pow(1.0 - max(dot(Normal, -ViewDir), 0.0), 5.0);

float3 FinalColor = lerp(RefractionColor, ReflectionColor, Fresnel);
```

## 泡沫效果

### 泡沫贴图

```hlsl
// 泡沫效果
float3 CalculateFoam(float2 UV, float Time)
{
    // 泡沫贴图
    float Foam = FoamTexture.Sample(Sampler, UV + Time * FoamSpeed).r;
    
    // 泡沫颜色
    float3 FoamColor = float3(1, 1, 1) * Foam;
    
    return FoamColor;
}
```

### 泡沫位置

```hlsl
// 根据波浪计算泡沫位置
float FoamAmount = max(0.0, WaveHeight - FoamThreshold);
float3 FoamColor = float3(1, 1, 1) * FoamAmount;
```

## 水体材质

### 完整的水体着色器

```hlsl
// 水体着色器
void MainPS(
    in float2 UV : TEXCOORD0,
    in float3 Normal : TEXCOORD1,
    in float3 ViewDir : TEXCOORD2,
    in float3 WorldPos : TEXCOORD3,
    out float4 OutColor : SV_Target0)
{
    // 计算波浪
    float3 WaveDisplacement = CalculateWaves(WorldPos, Time);
    float3 WaveNormal = CalculateWaveNormal(WaveDisplacement);
    
    // 混合法线
    float3 Normal = normalize(Normal + WaveNormal);
    
    // 计算反射和折射
    float3 ReflectionColor = CalculateReflection(Normal, ViewDir);
    float3 RefractionColor = CalculateRefraction(Normal, ViewDir, 1.33);
    
    // 菲涅尔混合
    float Fresnel = pow(1.0 - max(dot(Normal, -ViewDir), 0.0), 5.0);
    float3 WaterColor = lerp(RefractionColor, ReflectionColor, Fresnel);
    
    // 添加泡沫
    float3 FoamColor = CalculateFoam(UV, Time);
    WaterColor += FoamColor * 0.3;
    
    // 水的颜色
    float3 WaterTint = float3(0.1, 0.3, 0.5);
    WaterColor *= WaterTint;
    
    OutColor = float4(WaterColor, 1.0);
}
```

## 性能优化

### 波浪优化

```hlsl
// 减少波浪数量
const int NumWaves = 2;  // 而不是4

// 使用低精度
half3 WaveDisplacement = half3(0, 0, 0);
```

### 反射折射优化

```cpp
// 使用屏幕空间反射
r.ScreenSpaceReflectionIntensity = 1.0;

// 限制反射分辨率
r.ScreenPercentage = 80;
```

### 采样优化

```hlsl
// 减少纹理采样
// 使用打包纹理
float4 PackedData = PackedTexture.Sample(Sampler, UV);
```

## 常见问题

### 问题1: 波浪不真实

**症状**: 波浪看起来不自然

**原因**: 
- 波浪参数不当
- 波浪数量不足

**解决**:
```hlsl
// 调整波浪参数
// 增加波浪数量
const int NumWaves = 4;

// 调整频率和幅度
```

### 问题2: 性能下降

**症状**: 水体渲染导致FPS下降

**原因**: 
- 反射折射计算过多
- 波浪计算复杂

**解决**:
```cpp
// 使用屏幕空间反射
r.ScreenSpaceReflectionIntensity = 1.0;

// 减少波浪数量
const int NumWaves = 2;
```

### 问题3: 反射不准确

**症状**: 反射位置不对

**原因**: 
- 屏幕空间限制
- 法线计算错误

**解决**:
```hlsl
// 检查法线计算
// 使用更高质量的反射
r.ScreenSpaceReflectionQuality = 100;
```

## 最佳实践

1. **合理的波浪数量** - 平衡质量和性能
2. **优化反射** - 使用屏幕空间反射
3. **性能监控** - 定期检查成本
4. **质量等级** - 根据平台调整
5. **测试** - 在目标平台上测试

## 参考资源

- [UE水体文档](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Water/)
- [波浪模拟](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Water/WaveSimulation/)
