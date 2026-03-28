# 粒子系统

## 粒子系统概述

粒子系统用于创建动态效果，如火焰、烟雾、爆炸等。UE提供了Niagara粒子系统，是现代的粒子解决方案。

## Niagara vs Cascade

| 特性 | Niagara | Cascade |
|------|---------|---------|
| 架构 | 现代 | 旧版 |
| 性能 | 高 | 中 |
| GPU粒子 | 原生 | 有限 |
| 编辑器 | 高级 | 基础 |
| 推荐 | 新项目 | 维护 |

## Niagara粒子系统

### 基本概念

**发射器（Emitter）**: 产生粒子的源

**模块（Module）**: 控制粒子行为的单元

**参数（Parameter）**: 可调节的值

### 创建粒子系统

```cpp
// 创建Niagara系统
UNiagaraSystem* ParticleSystem = LoadObject<UNiagaraSystem>(nullptr,
    TEXT("NiagaraSystem'/Game/Particles/MyParticleSystem.MyParticleSystem'"));

// 生成粒子效果
UNiagaraFunctionLibrary::SpawnSystemAtLocation(
    GetWorld(),
    ParticleSystem,
    SpawnLocation
);
```

### 粒子生命周期

```
初始化 → 更新 → 渲染 → 销毁
  ↓      ↓      ↓      ↓
生成   运动   显示   消失
```

## 粒子属性

### 位置和运动

```cpp
// 设置初始位置
Emitter->SetPosition(FVector(0, 0, 100));

// 设置速度
Emitter->SetVelocity(FVector(0, 0, 100));

// 设置加速度
Emitter->SetAcceleration(FVector(0, 0, -980));
```

### 生命周期

```cpp
// 设置粒子生命周期
Emitter->SetLifeTime(2.0f);  // 2秒

// 设置发射速率
Emitter->SetEmissionRate(100);  // 每秒100个粒子
```

### 外观

```cpp
// 设置颜色
Emitter->SetColor(FLinearColor::Red);

// 设置大小
Emitter->SetSize(FVector(10, 10, 10));

// 设置透明度
Emitter->SetOpacity(0.8f);
```

## 常用模块

### 初始化模块

#### Spawn Rate
控制粒子生成速率

```
参数:
- Spawn Rate: 每秒生成的粒子数
- Spawn Rate Over Time: 随时间变化的生成速率
```

#### Initial Position
设置粒子初始位置

```
参数:
- Position: 相对于发射器的位置
- Position Offset: 位置偏移
```

#### Initial Velocity
设置粒子初始速度

```
参数:
- Velocity: 初始速度
- Velocity Offset: 速度偏移
```

### 更新模块

#### Velocity
更新粒子速度

```
参数:
- Velocity: 当前速度
- Acceleration: 加速度
```

#### Drag
添加阻力

```
参数:
- Drag Coefficient: 阻力系数
```

#### Collision
处理碰撞

```
参数:
- Collision Enabled: 启用碰撞
- Collision Radius: 碰撞半径
```

### 渲染模块

#### Sprite Renderer
渲染粒子精灵

```
参数:
- Material: 粒子材质
- Alignment: 对齐方式
- Pivot: 中心点
```

#### Mesh Renderer
渲染粒子网格

```
参数:
- Mesh: 粒子网格
- Material: 粒子材质
```

## 粒子材质

### 基础粒子材质

```hlsl
// 粒子着色器
void MainPS(
    in float4 Color : COLOR0,
    in float2 UV : TEXCOORD0,
    out float4 OutColor : SV_Target0)
{
    // 采样纹理
    float4 TexColor = Texture.Sample(Sampler, UV);
    
    // 应用颜色
    float4 FinalColor = TexColor * Color;
    
    OutColor = FinalColor;
}
```

### 动态粒子材质

```hlsl
// 动态效果
void MainPS(
    in float4 Color : COLOR0,
    in float2 UV : TEXCOORD0,
    in float Age : TEXCOORD1,
    out float4 OutColor : SV_Target0)
{
    // 基于年龄的动画
    float2 AnimatedUV = UV + float2(Age * Speed, 0);
    
    // 采样纹理
    float4 TexColor = Texture.Sample(Sampler, AnimatedUV);
    
    // 基于年龄的淡出
    float Alpha = 1.0 - (Age / MaxAge);
    
    OutColor = TexColor * Color * Alpha;
}
```

## 常见粒子效果

### 火焰效果

```
发射器配置:
- 生成速率: 50-100
- 生成范围: 小圆形
- 初始速度: 向上
- 生命周期: 1-2秒
- 颜色: 红→黄→透明
- 大小: 小→大→消失
```

### 烟雾效果

```
发射器配置:
- 生成速率: 20-50
- 生成范围: 中等圆形
- 初始速度: 向上+随机
- 生命周期: 3-5秒
- 颜色: 白→灰→透明
- 大小: 中→大
```

### 爆炸效果

```
发射器配置:
- 生成速率: 200-500（瞬间）
- 生成范围: 球形
- 初始速度: 向外+随机
- 生命周期: 0.5-1秒
- 颜色: 黄→红→透明
- 大小: 大→小
```

## GPU粒子

### 优点

- 高性能
- 支持大量粒子
- 复杂计算

### 配置

```cpp
// 启用GPU粒子
Emitter->bUseGPUSimulation = true;

// 设置最大粒子数
Emitter->MaxGPUParticles = 10000;
```

### GPU计算着色器

```hlsl
// GPU粒子计算着色器
[numthreads(64, 1, 1)]
void UpdateParticles(uint3 DispatchThreadID : SV_DispatchThreadID)
{
    uint ParticleIndex = DispatchThreadID.x;
    
    // 读取粒子数据
    float3 Position = ParticleBuffer[ParticleIndex].Position;
    float3 Velocity = ParticleBuffer[ParticleIndex].Velocity;
    float Age = ParticleBuffer[ParticleIndex].Age;
    
    // 更新位置
    Position += Velocity * DeltaTime;
    
    // 应用重力
    Velocity.z -= 980 * DeltaTime;
    
    // 更新年龄
    Age += DeltaTime;
    
    // 写回数据
    ParticleBuffer[ParticleIndex].Position = Position;
    ParticleBuffer[ParticleIndex].Velocity = Velocity;
    ParticleBuffer[ParticleIndex].Age = Age;
}
```

## 性能优化

### 粒子数量

| 效果 | 粒子数 | 说明 |
|------|--------|------|
| 小效果 | 10-50 | 火花、碎片 |
| 中等效果 | 50-200 | 烟雾、火焰 |
| 大效果 | 200-1000 | 爆炸、雨 |
| 极端效果 | 1000+ | 特殊场景 |

### 优化建议

```cpp
// 根据距离调整粒子数量
float Distance = FVector::Dist(CameraPos, ParticlePos);

if (Distance > 5000) {
    Emitter->SetEmissionRate(10);  // 远处降低
} else if (Distance > 2000) {
    Emitter->SetEmissionRate(50);
} else {
    Emitter->SetEmissionRate(100);  // 近处高质量
}
```

### 内存优化

```cpp
// 使用GPU粒子
Emitter->bUseGPUSimulation = true;

// 限制最大粒子数
Emitter->MaxGPUParticles = 5000;

// 使用简单材质
Emitter->SetMaterial(SimpleMaterial);
```

## 常见问题

### 问题1: 粒子性能差

**症状**: 粒子效果导致FPS下降

**原因**: 
- 粒子数量过多
- 材质复杂
- CPU模拟

**解决**:
```cpp
// 使用GPU粒子
Emitter->bUseGPUSimulation = true;

// 减少粒子数量
Emitter->SetEmissionRate(50);

// 使用简单材质
```

### 问题2: 粒子看起来不对

**症状**: 粒子显示错误或消失

**原因**: 
- 材质问题
- 生命周期设置错误
- 渲染模块配置错误

**解决**:
```cpp
// 检查材质
Emitter->SetMaterial(CorrectMaterial);

// 调整生命周期
Emitter->SetLifeTime(2.0f);

// 检查渲染模块
```

### 问题3: 粒子碰撞不工作

**症状**: 粒子穿过物体

**原因**: 
- 碰撞模块未启用
- 碰撞设置错误

**解决**:
```cpp
// 启用碰撞
Emitter->bEnableCollision = true;

// 设置碰撞半径
Emitter->SetCollisionRadius(10.0f);
```

## 最佳实践

1. **使用GPU粒子** - 性能更好
2. **优化粒子数量** - 根据距离调整
3. **简化材质** - 使用高效的着色器
4. **测试性能** - 监控FPS
5. **重用系统** - 使用参数化设计

## 参考资源

- [UE Niagara文档](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Niagara/)
- [粒子系统指南](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Niagara/Overview/)
- [GPU粒子](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Niagara/GPUParticles/)
