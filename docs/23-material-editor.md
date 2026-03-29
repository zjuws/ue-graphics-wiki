# 材质编辑器高级

## 材质编辑器概述

材质编辑器是UE的核心工具，用于创建复杂的材质效果。

## 高级界面功能

### 材质图表

**节点组织**:
- 使用注释框分组相关节点
- 使用 reroute 节点整理连线
- 使用 material function 复用代码

**快捷键**:
```
Ctrl+C: 复制节点
Ctrl+V: 粘贴节点
Delete: 删除节点
Ctrl+Z: 撤销
Ctrl+Y: 重做
F: 聚焦选中节点
Home: 聚焦所有节点
```

### 材质实例

**创建材质实例**:
```cpp
// 基于材质创建实例
UMaterialInstanceDynamic* MatInstance = 
    UMaterialInstanceDynamic::Create(BaseMaterial, this);

// 应用到组件
MeshComponent->SetMaterial(0, MatInstance);
```

**动态参数**:
```cpp
// 设置标量参数
MatInstance->SetScalarParameterValue(FName("Roughness"), 0.5f);

// 设置向量参数
MatInstance->SetVectorParameterValue(FName("BaseColor"), FLinearColor::Red);

// 设置纹理参数
MatInstance->SetTextureParameterValue(FName("BaseTexture"), NewTexture);
```

## 高级材质函数

### 自定义材质函数

**创建材质函数**:
1. 右键 → Material Functions → New Material Function
2. 编写节点逻辑
3. 设置输入和输出参数
4. 保存并使用

**示例 - Fresnel函数**:
```hlsl
// 输入: Normal, ViewDir, Power
// 输出: FresnelValue

float NdotV = max(dot(Normal, -ViewDir), 0.0);
return pow(1.0 - NdotV, Power);
```

### 材质函数库

**常用函数**:

#### 法线混合
```hlsl
float3 BlendNormals(float3 Normal1, float3 Normal2)
{
    return normalize(Normal1 + Normal2 - float3(0, 0, 1));
}
```

#### 三平面映射
```hlsl
float3 TriplanarMapping(float3 WorldPos, float3 Normal)
{
    float3 ColorX = Texture.Sample(Sampler, WorldPos.yz).rgb;
    float3 ColorY = Texture.Sample(Sampler, WorldPos.xz).rgb;
    float3 ColorZ = Texture.Sample(Sampler, WorldPos.xy).rgb;
    
    float3 BlendWeights = abs(Normal);
    BlendWeights /= (BlendWeights.x + BlendWeights.y + BlendWeights.z);
    
    return ColorX * BlendWeights.x + ColorY * BlendWeights.y + ColorZ * BlendWeights.z;
}
```

## 材质分层

### 分层材质

**原理**: 使用多个材质层叠加

```hlsl
// 材质分层
float3 BaseMaterial = BaseColor;
float3 DetailMaterial = DetailColor;
float3 OverlayMaterial = OverlayColor;

float BlendFactor = 0.5;
float3 FinalMaterial = lerp(BaseMaterial, DetailMaterial, BlendFactor);
FinalMaterial = lerp(FinalMaterial, OverlayMaterial, OverlayBlend);
```

### 材质混合

**高度混合**:
```hlsl
// 基于高度的材质混合
float Height1 = HeightTexture1.Sample(UV).r;
float Height2 = HeightTexture2.Sample(UV).r;

float Blend = smoothstep(Height1, Height2, BlendFactor);
float3 FinalColor = lerp(Color1, Color2, Blend);
```

## 材质优化

### 指令优化

```hlsl
// 不好：复杂计算
float3 Result = normalize(cross(A, cross(B, C)));

// 好：简化计算
float3 Result = normalize(A);
```

### 纹理采样优化

```hlsl
// 不好：多次采样
float3 Color1 = Texture.Sample(UV);
float3 Color2 = Texture.Sample(UV);

// 好：采样一次
float3 Color = Texture.Sample(UV);
float3 Color1 = Color;
float3 Color2 = Color;
```

### 分支优化

```hlsl
// 不好：动态分支
if (Condition) {
    Result = ExpensiveCalculation();
} else {
    Result = CheapCalculation();
}

// 好：使用lerp
Result = lerp(CheapCalculation(), ExpensiveCalculation(), Condition);
```

## 高级技巧

### 材质参数动画

```hlsl
// 时间驱动的动画
float Time = GetTime();
float AnimatedValue = sin(Time * Speed) * 0.5 + 0.5;

// UV动画
float2 AnimatedUV = UV + float2(Time * Speed, 0);
float3 AnimatedColor = Texture.Sample(Sampler, AnimatedUV).rgb;
```

### 材质LOD

```cpp
// 根据距离选择材质复杂度
float Distance = FVector::Dist(CameraPos, ActorPos);

if (Distance > 5000) {
    // 简化材质
    UseLowQualityMaterial();
} else if (Distance > 2000) {
    // 中等质量
    UseMediumQualityMaterial();
} else {
    // 高质量
    UseHighQualityMaterial();
}
```

## 常见问题

### 问题1: 材质编译错误

**症状**: 材质无法编译

**原因**: 
- 语法错误
- 节点连接错误
- 参数类型不匹配

**解决**:
```
1. 检查错误信息
2. 验证节点连接
3. 检查参数类型
```

### 问题2: 材质性能差

**症状**: 材质导致FPS下降

**原因**: 
- 指令过多
- 纹理采样过多
- 复杂计算

**解决**:
```hlsl
// 简化材质
// 减少纹理采样
// 优化计算
```

### 问题3: 材质显示错误

**症状**: 材质看起来不对

**原因**: 
- UV坐标错误
- 法线计算错误
- 参数设置错误

**解决**:
```
1. 检查UV坐标
2. 验证法线计算
3. 调整参数
```

## 最佳实践

1. **使用材质函数** - 复用代码
2. **组织节点** - 使用注释和分组
3. **优化性能** - 减少复杂度
4. **测试** - 在各种条件下测试
5. **文档化** - 记录参数含义

## 参考资源

- [UE材质编辑器文档](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Materials/Editor/)
- [材质函数](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Materials/Functions/)
