# 光照与阴影

## 光照基础

### 光照类型

#### 1. 方向光（Directional Light）
模拟太阳光，平行光线。

**特点**:
- 无衰减
- 影响整个场景
- 适合主光源

**参数**:
```cpp
ADirectionalLight* Light = GetWorld()->SpawnActor<ADirectionalLight>();
Light->SetIntensity(2.0f);
Light->SetLightColor(FLinearColor::White);
Light->SetActorRotation(FRotator(-45, 45, 0));
```

#### 2. 点光源（Point Light）
从一点向四周发射光线。

**特点**:
- 有衰减
- 影响范围有限
- 适合灯泡、火焰等

**参数**:
```cpp
APointLight* Light = GetWorld()->SpawnActor<APointLight>();
Light->SetIntensity(1000.0f);
Light->SetAttenuationRadius(1000.0f);
Light->SetLightColor(FLinearColor::Yellow);
```

#### 3. 聚光灯（Spot Light）
从一点向锥形区域发射光线。

**特点**:
- 有方向和角度
- 有衰减
- 适合手电筒、舞台灯等

**参数**:
```cpp
ASpotLight* Light = GetWorld()->SpawnActor<ASpotLight>();
Light->SetIntensity(2000.0f);
Light->SetOuterConeAngle(45.0f);
Light->SetInnerConeAngle(30.0f);
```

### 光照衰减

**距离衰减公式**:
```
Attenuation = 1 / (Distance / Radius)^2
```

**在材质中**:
```hlsl
float Distance = length(PixelWorldPos - LightWorldPos);
float Attenuation = 1.0 / (1.0 + (Distance / Radius) * (Distance / Radius));
```

## 阴影系统

### 阴影贴图（Shadow Mapping）

**原理**:
1. 从光源视角渲染场景到深度贴图
2. 在最终渲染时比较像素深度与阴影贴图深度
3. 如果像素深度 > 阴影贴图深度，则在阴影中

**优点**:
- 效率高
- 支持动态阴影
- 易于实现

**缺点**:
- 阴影边界锯齿
- 需要大量内存

### 级联阴影贴图（Cascaded Shadow Maps, CSM）

**解决问题**: 远处阴影分辨率低

**原理**:
- 将视锥体分为多个级联
- 每个级联使用不同分辨率的阴影贴图
- 近处高分辨率，远处低分辨率

**UE配置**:
```cpp
// 在光照设置中
DirectionalLight->DynamicShadowDistances = {5000, 10000, 20000, 50000};
DirectionalLight->DynamicShadowCascades = 4;
DirectionalLight->ShadowMapResolution = 2048;
```

### 距离场阴影（Distance Field Shadows）

**原理**:
- 预计算物体的距离场
- 使用距离场快速计算阴影

**优点**:
- 高效
- 支持大范围阴影
- 可用于全局光照

**缺点**:
- 需要预计算
- 内存占用

### 光线追踪阴影（Ray Traced Shadows）

**原理**:
- 使用GPU光线追踪计算精确阴影
- 支持软阴影、半影

**优点**:
- 物理准确
- 支持复杂几何
- 自动软阴影

**缺点**:
- 性能消耗大
- 需要RTX GPU

## 全局光照（Global Illumination）

### Lightmass烘焙

**工作流**:
1. 设置光照贴图UV
2. 配置Lightmass设置
3. 构建光照
4. 烘焙完成

**关键参数**:
```cpp
// 在Actor上
Actor->SetCastShadow(true);
Actor->SetLightmassSettings(
    true,  // bUseAsOccluder
    false, // bUseAsSourceSurfaceForLighting
    false  // bUseEmissiveForStaticLighting
);
```

**Lightmass设置**:
```
Num Indirect Lighting Bounces: 3-5
Num Sky Lighting Bounces: 1-2
Indirect Lighting Quality: 2-4
```

### 光照探针（Light Probes）

**用途**: 为动态物体提供全局光照

**放置**:
- 在关键位置手动放置
- 或使用自动生成

**代码**:
```cpp
ALightProbeVolume* ProbeVolume = GetWorld()->SpawnActor<ALightProbeVolume>();
ProbeVolume->SetActorScale3D(FVector(1000, 1000, 500));
```

### 反射探针（Reflection Probes）

**用途**: 捕捉环境反射

**类型**:
- **Box Reflection Capture**: 盒形捕捉
- **Sphere Reflection Capture**: 球形捕捉

**代码**:
```cpp
AReflectionCapture* Capture = GetWorld()->SpawnActor<AReflectionCapture>();
Capture->SetActorScale3D(FVector(1000, 1000, 1000));
Capture->CaptureScene();
```

## 光照计算

### PBR光照模型

**Cook-Torrance模型**:
```
Lo = ∫ (kd * c/π + ks * DFG/(4(ωo·n)(ωi·n))) * Li * (ωi·n) dωi
```

其中：
- `kd`: 漫反射系数
- `ks`: 镜面反射系数
- `D`: 法线分布函数（NDF）
- `F`: 菲涅尔方程
- `G`: 几何遮挡函数

### 在HLSL中实现

```hlsl
float3 CalculateLighting(
    float3 BaseColor,
    float Metallic,
    float Roughness,
    float3 Normal,
    float3 ViewDir,
    float3 LightDir,
    float3 LightColor)
{
    float3 H = normalize(ViewDir + LightDir);
    float NoH = max(dot(Normal, H), 0.0);
    float NoL = max(dot(Normal, LightDir), 0.0);
    float NoV = max(dot(Normal, ViewDir), 0.0);
    
    // 菲涅尔
    float3 F0 = lerp(float3(0.04, 0.04, 0.04), BaseColor, Metallic);
    float3 F = F0 + (1.0 - F0) * pow(1.0 - NoH, 5.0);
    
    // 法线分布（GGX）
    float alpha = Roughness * Roughness;
    float alpha2 = alpha * alpha;
    float denom = NoH * NoH * (alpha2 - 1.0) + 1.0;
    float D = alpha2 / (3.14159 * denom * denom);
    
    // 几何遮挡（Schlick-GGX）
    float k = (Roughness + 1.0) * (Roughness + 1.0) / 8.0;
    float G = NoL / (NoL * (1.0 - k) + k) * NoV / (NoV * (1.0 - k) + k);
    
    // 镜面反射
    float3 Specular = (D * F * G) / (4.0 * NoL * NoV + 0.001);
    
    // 漫反射
    float3 Diffuse = BaseColor / 3.14159;
    
    // 最终颜色
    return (Diffuse + Specular) * LightColor * NoL;
}
```

## 性能优化

### 光照剔除
```cpp
// 只计算影响该物体的光源
TArray<FLightSceneInfo*> RelevantLights;
Scene->GetRelevantLights(Primitive, RelevantLights);
```

### 烘焙vs动态
| 方案 | 优点 | 缺点 |
|------|------|------|
| 完全烘焙 | 性能最优 | 不支持动态变化 |
| 完全动态 | 完全动态 | 性能消耗大 |
| 混合 | 平衡 | 配置复杂 |

### 最佳实践

1. **使用烘焙光照**
   - 静态场景使用烘焙
   - 减少实时光源数量

2. **光照剔除**
   - 使用光照通道
   - 限制影响范围

3. **阴影优化**
   - 使用级联阴影贴图
   - 合理设置阴影距离

4. **移动平台**
   - 使用烘焙光照
   - 限制实时光源数量
   - 使用距离场阴影

## 常见问题

**Q: 如何改进阴影质量？**  
A: 增加阴影贴图分辨率、使用更多级联、启用软阴影。

**Q: 光照探针如何放置？**  
A: 在关键位置（转角、高度变化处）放置，间距1-2米。

**Q: 如何调试光照问题？**  
A: 使用`r.VisualizeLight`命令查看光照贡献。

## 参考资源

- [UE光照文档](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Lighting/)
- [Lightmass指南](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Lightmass/)
