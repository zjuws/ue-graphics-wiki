# 渲染管线概述

## 什么是渲染管线？

渲染管线是将3D场景数据转换为2D屏幕图像的一系列步骤。虚幻引擎使用**延迟渲染**作为主要渲染方式（可选前向渲染）。

## 渲染管线的主要阶段

### 1. 应用阶段（Application Stage）

**发生位置**: CPU  
**主要任务**:
- 场景剔除（Culling）- 移除视锥体外的物体
- 批处理（Batching）- 合并相同材质的绘制调用
- 排序（Sorting）- 按材质、距离等排序
- 准备渲染命令（Render Commands）

```cpp
// 伪代码示例
for (AActor* Actor : Scene->GetActors()) {
    if (FrustumCull(Actor)) continue;  // 剔除
    RenderQueue.Add(Actor);
}
```

### 2. 几何处理阶段（Geometry Processing）

**发生位置**: GPU（顶点着色器）  
**主要任务**:
- 顶点变换（Model → World → View → Projection）
- 顶点动画（骨骼动画、顶点着色器动画）
- 法线变换
- 纹理坐标处理

```hlsl
// 顶点着色器示例
float4 VS_Main(FVertexInput Input) : SV_POSITION
{
    float4 WorldPos = mul(float4(Input.Position, 1), ObjectToWorld);
    float4 ViewPos = mul(WorldPos, ViewMatrix);
    float4 ProjPos = mul(ViewPos, ProjectionMatrix);
    return ProjPos;
}
```

### 3. 光栅化阶段（Rasterization）

**发生位置**: GPU（固定功能）  
**主要任务**:
- 三角形设置
- 扫描转换（Scan Conversion）
- 深度测试（Depth Test）
- 背面剔除（Back Face Culling）

### 4. 像素处理阶段（Pixel Processing）

**发生位置**: GPU（像素着色器）  
**主要任务**:
- 纹理采样
- 法线贴图应用
- 材质计算
- 光照计算（延迟渲染中为G-Buffer写入）

```hlsl
// 像素着色器示例（G-Buffer写入）
void PS_GBuffer(FPixelInput Input, out FGBufferData GBuffer)
{
    float3 BaseColor = Texture0.Sample(Sampler, Input.UV).rgb;
    float3 Normal = normalize(Texture1.Sample(Sampler, Input.UV).rgb * 2 - 1);
    float Metallic = Texture2.Sample(Sampler, Input.UV).r;
    float Roughness = Texture3.Sample(Sampler, Input.UV).r;
    
    GBuffer.BaseColor = BaseColor;
    GBuffer.Normal = Normal;
    GBuffer.Metallic = Metallic;
    GBuffer.Roughness = Roughness;
}
```

### 5. 光照通道（Lighting Pass）

**发生位置**: GPU（计算着色器或像素着色器）  
**主要任务**:
- 读取G-Buffer数据
- 计算光照贡献
- 应用阴影
- 写入最终颜色

### 6. 后处理阶段（Post-Processing）

**发生位置**: GPU（像素着色器）  
**主要任务**:
- 色调映射（Tone Mapping）
- 色彩分级（Color Grading）
- 景深（Depth of Field）
- 运动模糊（Motion Blur）
- 抗锯齿（Anti-Aliasing）

### 7. 合成与显示（Composition & Display）

**发生位置**: GPU/显示器  
**主要任务**:
- UI合成
- 帧缓冲交换
- 显示输出

## UE中的延迟渲染流程

```
┌─────────────────────────────────────────────────────────┐
│ 1. 场景剔除与排序 (CPU)                                  │
│    - 视锥体剔除                                          │
│    - 遮挡剔除                                            │
│    - 批处理                                              │
└──────────────────┬──────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────────┐
│ 2. G-Buffer通道 (GPU)                                    │
│    - 写入BaseColor                                       │
│    - 写入Normal                                          │
│    - 写入Metallic/Roughness                              │
│    - 写入深度                                            │
└──────────────────┬──────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────────┐
│ 3. 光照通道 (GPU)                                        │
│    - 方向光                                              │
│    - 点光源                                              │
│    - 聚光灯                                              │
│    - 阴影计算                                            │
└──────────────────┬──────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────────┐
│ 4. 透明物体通道 (GPU)                                    │
│    - 前向渲染透明物体                                    │
│    - 混合模式应用                                        │
└──────────────────┬──────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────────┐
│ 5. 后处理 (GPU)                                          │
│    - 色调映射                                            │
│    - 色彩分级                                            │
│    - 景深/运动模糊                                       │
└──────────────────┬──────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────────┐
│ 6. UI合成与显示                                          │
└─────────────────────────────────────────────────────────┘
```

## 关键概念

### 深度测试（Depth Testing）
- **Early-Z**: 在像素着色器前进行深度测试，提高性能
- **Z-Prepass**: 先写入深度，再进行颜色写入

### 模板测试（Stencil Testing）
用于：
- 镜面反射
- 阴影体积
- UI遮挡

### 混合模式（Blend Modes）
- **Opaque**: 完全不透明
- **Masked**: 二值透明（有或无）
- **Translucent**: 半透明
- **Additive**: 加法混合
- **Modulate**: 调制混合

## 性能考虑

| 阶段 | 瓶颈 | 优化方向 |
|------|------|--------|
| 应用 | CPU | 减少绘制调用、批处理 |
| 几何 | 顶点数量 | LOD、顶点着色器优化 |
| 光栅化 | 填充率 | 分辨率、MSAA |
| 像素 | 纹理带宽 | 纹理压缩、采样优化 |
| 光照 | 光源数量 | 光照剔除、烘焙 |

## 最佳实践

1. **使用Stat命令分析**
   ```
   stat unit        // 总体性能
   stat unitgraph   // 图表显示
   stat scenerendering  // 场景渲染详情
   ```

2. **启用GPU分析**
   - 使用GPU Insights
   - 查看各通道耗时

3. **合理使用LOD**
   - 远处物体使用低多边形模型
   - 减少顶点着色器负担

4. **优化材质复杂度**
   - 避免过多纹理采样
   - 使用材质函数复用代码

## 参考资源

- [UE官方渲染文档](https://docs.unrealengine.com/en-US/RenderingAndGraphics/)
- [Rendering Architecture](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Overview/)
