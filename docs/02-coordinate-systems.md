# 坐标系统与变换

## 坐标系统概述

UE使用多个坐标系统来处理3D空间中的位置和方向。理解这些系统对于正确的渲染至关重要。

## 主要坐标系统

### 1. 模型空间（Model/Local Space）

**定义**: 相对于模型自身的坐标系

**特点**:
- 原点在模型中心
- 与模型一起旋转和缩放
- 顶点数据存储在此空间

**示例**:
```cpp
// 模型空间中的顶点
FVector LocalVertex = FVector(0, 0, 100);  // 模型上方100单位
```

### 2. 世界空间（World Space）

**定义**: 整个游戏世界的全局坐标系

**特点**:
- 固定的全局坐标系
- 所有物体的最终位置
- 光源、摄像机都在世界空间

**示例**:
```cpp
// 获取Actor的世界位置
FVector WorldPos = Actor->GetActorLocation();
```

### 3. 视图空间（View/Camera Space）

**定义**: 相对于摄像机的坐标系

**特点**:
- 摄像机在原点
- 摄像机前方是Z轴正方向
- 用于光照计算

**转换**:
```hlsl
// 在顶点着色器中
float4 ViewPos = mul(WorldPos, ViewMatrix);
```

### 4. 投影空间（Projection Space）

**定义**: 投影到屏幕的坐标系

**特点**:
- 范围 -1 到 1（NDC - Normalized Device Coordinates）
- 用于光栅化
- 深度范围 0 到 1

**转换**:
```hlsl
// 完整的顶点变换
float4 ProjPos = mul(WorldPos, ViewMatrix);
ProjPos = mul(ProjPos, ProjectionMatrix);
```

### 5. 屏幕空间（Screen Space）

**定义**: 最终的2D屏幕坐标

**特点**:
- 范围 0 到分辨率
- 用于UI和后处理
- 像素坐标

**转换**:
```cpp
// 世界坐标转屏幕坐标
FVector ScreenPos = Camera->GetViewportClient()->Project(WorldPos);
```

## 坐标系统转换

### 完整的变换管线

```
模型空间 → 世界空间 → 视图空间 → 投影空间 → 屏幕空间
   ↓          ↓          ↓          ↓          ↓
 顶点      位置/旋转    相对摄像机   透视除法    光栅化
```

### 变换矩阵

#### 模型到世界（Model to World）

```hlsl
float4 WorldPos = mul(float4(LocalPos, 1), ObjectToWorld);
```

**包含**:
- 平移（Translation）
- 旋转（Rotation）
- 缩放（Scale）

#### 世界到视图（World to View）

```hlsl
float4 ViewPos = mul(WorldPos, ViewMatrix);
```

**特点**:
- 将世界坐标转换为相对摄像机的坐标
- ViewMatrix = inverse(CameraTransform)

#### 视图到投影（View to Projection）

```hlsl
float4 ProjPos = mul(ViewPos, ProjectionMatrix);
```

**类型**:
- **透视投影**: 近处大，远处小
- **正交投影**: 平行投影，无透视

### 代码示例

```cpp
// C++中的完整变换
FVector LocalPos = FVector(0, 0, 100);

// 1. 模型到世界
FVector WorldPos = Actor->GetActorTransform().TransformPosition(LocalPos);

// 2. 世界到视图
FVector ViewPos = Camera->GetViewMatrix().TransformPosition(WorldPos);

// 3. 视图到投影
FVector4 ProjPos = Camera->GetProjectionMatrix().TransformPosition(ViewPos);

// 4. 透视除法
FVector2D ScreenPos = FVector2D(
    ProjPos.X / ProjPos.W,
    ProjPos.Y / ProjPos.W
);

// 5. 转换到屏幕坐标
FVector2D PixelPos = FVector2D(
    (ScreenPos.X + 1) * ViewportWidth / 2,
    (1 - ScreenPos.Y) * ViewportHeight / 2
);
```

## 法线变换

**重要**: 法线不能直接使用模型矩阵变换！

### 正确的法线变换

```hlsl
// 错误：直接使用模型矩阵
float3 WorldNormal = mul(LocalNormal, (float3x3)ObjectToWorld);

// 正确：使用逆转置矩阵
float3 WorldNormal = mul(LocalNormal, (float3x3)ObjectToWorldInvTranspose);
```

### 为什么需要逆转置？

当模型有非均匀缩放时，法线会扭曲。逆转置矩阵修正这个问题。

```cpp
// C++中计算逆转置矩阵
FMatrix ObjectToWorld = Actor->GetActorTransform().ToMatrixWithScale();
FMatrix ObjectToWorldInvTranspose = ObjectToWorld.Inverse().Transpose();
```

## 切线空间（Tangent Space）

### 概念

切线空间是相对于表面的局部坐标系，用于法线贴图。

**三个轴**:
- **法线（Normal）**: 垂直于表面
- **切线（Tangent）**: 沿着表面，通常沿U方向
- **副切线（Bitangent）**: 沿着表面，通常沿V方向

### TBN矩阵

```hlsl
// 构建TBN矩阵
float3 T = normalize(Tangent);
float3 B = normalize(Bitangent);
float3 N = normalize(Normal);

float3x3 TBN = float3x3(T, B, N);
```

### 法线贴图应用

```hlsl
// 从切线空间转换到世界空间
float3 NormalTS = Texture.Sample(UV).rgb * 2.0 - 1.0;
float3 NormalWS = mul(NormalTS, TBN);
```

## 常见坐标系统问题

### 问题1: 法线贴图反向

**症状**: 法线贴图看起来反向

**原因**: 
- 绿色通道反向
- TBN矩阵计算错误

**解决**:
```hlsl
// 反向绿色通道
float3 Normal = Texture.Sample(UV).rgb;
Normal.g = 1.0 - Normal.g;
```

### 问题2: 模型缩放后光照错误

**症状**: 缩放模型后光照不正确

**原因**: 法线没有正确变换

**解决**:
```cpp
// 使用逆转置矩阵
FMatrix InvTranspose = ObjectToWorld.Inverse().Transpose();
```

### 问题3: 屏幕空间计算错误

**症状**: 屏幕空间效果位置不对

**原因**: 坐标系统转换错误

**解决**:
```hlsl
// 确保正确的透视除法
float2 ScreenUV = ProjPos.xy / ProjPos.w;
ScreenUV = ScreenUV * 0.5 + 0.5;  // 转换到0-1范围
```

## 性能考虑

### 矩阵乘法成本

| 操作 | 成本 | 说明 |
|------|------|------|
| 向量-矩阵乘法 | 低 | 常见操作 |
| 矩阵-矩阵乘法 | 中 | 避免在着色器中 |
| 矩阵求逆 | 高 | 预计算 |
| 矩阵转置 | 低 | 廉价操作 |

### 优化建议

```cpp
// 不好：每帧计算逆转置矩阵
FMatrix InvTranspose = ObjectToWorld.Inverse().Transpose();

// 好：缓存矩阵
class FMeshComponent {
    FMatrix CachedObjectToWorldInvTranspose;
    
    void UpdateTransform() {
        CachedObjectToWorldInvTranspose = 
            GetActorTransform().ToMatrixWithScale().Inverse().Transpose();
    }
};
```

## 最佳实践

1. **理解坐标系统** - 知道数据在哪个空间
2. **正确变换法线** - 使用逆转置矩阵
3. **缓存矩阵** - 避免重复计算
4. **验证坐标** - 使用调试可视化
5. **文档化** - 标记变量的坐标空间

## 调试技巧

### 可视化坐标系统

```cpp
// 绘制坐标轴
void DrawCoordinateSystem(const FVector& Origin, const FMatrix& Transform)
{
    FVector X = Transform.TransformVector(FVector(100, 0, 0));
    FVector Y = Transform.TransformVector(FVector(0, 100, 0));
    FVector Z = Transform.TransformVector(FVector(0, 0, 100));
    
    DrawDebugLine(GetWorld(), Origin, Origin + X, FColor::Red);
    DrawDebugLine(GetWorld(), Origin, Origin + Y, FColor::Green);
    DrawDebugLine(GetWorld(), Origin, Origin + Z, FColor::Blue);
}
```

### 验证变换

```cpp
// 验证变换是否正确
void VerifyTransform(const FVector& LocalPos, const FVector& ExpectedWorldPos)
{
    FVector ActualWorldPos = Actor->GetActorTransform().TransformPosition(LocalPos);
    
    if (!ActualWorldPos.Equals(ExpectedWorldPos, 0.01f)) {
        UE_LOG(LogWarning, Warning, TEXT("Transform mismatch!"));
    }
}
```

## 参考资源

- [UE坐标系统文档](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Coordinates/)
- [矩阵数学](https://docs.unrealengine.com/en-US/ProgrammingAndScripting/ProgrammingWithCPP/UnrealMathematicsLibrary/)
- [法线贴图指南](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Materials/HowTo/NormalMaps/)
