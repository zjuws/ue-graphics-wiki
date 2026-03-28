# 快速参考卡片

## 🎨 材质参数速查

### PBR参数范围

```
Base Color:     0-1 (RGB)
Metallic:       0 (非金属) 或 1 (金属)
Roughness:      0 (光滑) - 1 (粗糙)
Normal:         -1 to 1 (法线向量)
AO:             0 (暗) - 1 (亮)
Emissive:       0+ (自发光强度)
Opacity:        0 (透明) - 1 (不透明)
```

### 常见材质设置

| 材质 | Base Color | Metallic | Roughness |
|------|-----------|----------|-----------|
| 铁 | 灰色 | 1.0 | 0.3-0.5 |
| 铜 | 橙红 | 1.0 | 0.2-0.3 |
| 金 | 金黄 | 1.0 | 0.1-0.2 |
| 木头 | 棕色 | 0.0 | 0.5-0.7 |
| 布料 | 颜色 | 0.0 | 0.8-1.0 |
| 石头 | 灰色 | 0.0 | 0.8-0.9 |
| 皮肤 | 肤色 | 0.0 | 0.3-0.4 |

## 📊 性能目标

### 桌面平台
- **FPS**: 60-144
- **GPU时间**: < 16ms (60fps) 或 < 7ms (144fps)
- **CPU时间**: < 16ms

### 移动平台
- **FPS**: 30-60
- **GPU时间**: < 33ms (30fps) 或 < 16ms (60fps)
- **CPU时间**: < 33ms
- **内存**: < 2GB

### VR平台
- **FPS**: 90
- **GPU时间**: < 11ms
- **CPU时间**: < 11ms
- **延迟**: < 20ms

## 🔧 常用命令

### 性能分析
```
stat unit           // 总体性能
stat unitgraph      // 图表显示
stat scenerendering // 场景渲染详情
stat gpu            // GPU时间
stat memory         // 内存统计
```

### 渲染调试
```
r.VisualizeLight 1              // 可视化光照
r.VisualizeNormalMap 1          // 可视化法线
r.ScreenPercentage 80           // 屏幕百分比
r.MaxFPS 60                     // 限制帧率
r.VSync 1                       // 垂直同步
```

### 后处理
```
r.ScreenSpaceReflectionIntensity 1.0
r.DynamicGlobalIlluminationIntensity 1.0
r.DepthOfFieldQuality 2
r.MotionBlurQuality 2
r.AmbientOcclusionIntensity 1.0
```

### 光照
```
r.DeferredLighting.MaxLights 256
r.TiledDeferredShading 1
r.Shadow.MaxCSMCascades 4
r.DistanceFieldAO 1
r.Lumen.Enabled 1
```

## 📐 坐标系统

### 变换管线
```
模型空间 → 世界空间 → 视图空间 → 投影空间 → 屏幕空间
```

### HLSL变换
```hlsl
// 模型到世界
float4 WorldPos = mul(float4(LocalPos, 1), ObjectToWorld);

// 世界到视图
float4 ViewPos = mul(WorldPos, ViewMatrix);

// 视图到投影
float4 ProjPos = mul(ViewPos, ProjectionMatrix);

// 法线变换（使用逆转置）
float3 WorldNormal = mul(LocalNormal, (float3x3)ObjectToWorldInvTranspose);
```

## 🎬 后处理效果成本

| 效果 | 成本 | 建议 |
|------|------|------|
| 色调映射 | 低 | 总是启用 |
| 色彩分级 | 低 | 总是启用 |
| 景深 | 中 | 有选择性 |
| 运动模糊 | 中 | 有选择性 |
| SSAO | 高 | 高端平台 |
| SSR | 高 | 高端平台 |
| DFGI | 高 | 高端平台 |

## 🌟 光照优化

### 光源数量限制
```
桌面: 256个
移动: 16-32个
VR: 32个
```

### 阴影配置
```
方向光: 4个级联
点光源: 1个阴影贴图
聚光灯: 1个阴影贴图
```

## 📦 纹理优化

### 分辨率选择
```
主角: 2K-4K
环境: 1K-2K
远景: 512-1K
移动: 512-1K
```

### 压缩格式
```
颜色: DXT1/BC1 (6:1)
法线: DXT5/BC5 (2:1)
灰度: BC4 (2:1)
HDR: BC6H (6:1)
```

## 🎯 LOD配置

### 距离设置
```
LOD0: 0-500m (100%)
LOD1: 500-1000m (50%)
LOD2: 1000-2000m (25%)
LOD3: 2000m+ (10%)
```

## 💾 内存预算

### 纹理内存
```
桌面: 2-4GB
移动: 512MB-1GB
VR: 1-2GB
```

### 计算公式
```
内存 = 宽 × 高 × 字节数 × 1.33 (含MIP)

示例:
2048×2048 RGBA8 = 2048 × 2048 × 4 × 1.33 ≈ 22MB
```

## 🔍 常见问题速解

### 法线贴图反向
```hlsl
Normal.g = 1.0 - Normal.g;  // 反向绿色通道
```

### 模型缩放后光照错误
```cpp
// 使用逆转置矩阵
FMatrix InvTranspose = ObjectToWorld.Inverse().Transpose();
```

### 屏幕空间计算错误
```hlsl
// 确保正确的透视除法
float2 ScreenUV = ProjPos.xy / ProjPos.w;
ScreenUV = ScreenUV * 0.5 + 0.5;  // 转换到0-1范围
```

### 纹理模糊
```cpp
// 提高MIP质量
Texture->MipGenSettings = TMGS_FromTextureGroup;
Texture->Filter = TF_Aniso;  // 各向异性采样
```

## 🚀 优化检查清单

- [ ] 使用Stat命令分析性能
- [ ] 启用批处理
- [ ] 配置LOD
- [ ] 使用遮挡剔除
- [ ] 压缩纹理
- [ ] 优化着色器
- [ ] 烘焙光照
- [ ] 限制动态光源
- [ ] 禁用昂贵效果
- [ ] 测试移动平台
- [ ] 监控内存使用
- [ ] 定期性能测试

## 📚 文档导航

### 快速开始
- README.md - 项目总览
- 01-rendering-pipeline.md - 渲染基础

### 核心知识
- 02-coordinate-systems.md - 坐标系统
- 03-lighting-shadows.md - 光照与阴影
- 04-material-basics.md - 材质系统
- 06-pbr.md - PBR工作流

### 高级主题
- 07-textures.md - 纹理系统
- 10-deferred-rendering.md - 延迟渲染
- 12-post-processing.md - 后处理效果
- 18-global-illumination.md - 全局光照

### 优化指南
- 14-rendering-optimization.md - 渲染优化
- 15-gpu-optimization.md - GPU优化
- 16-mobile-optimization.md - 移动优化
- 20-particle-systems.md - 粒子系统

---

**快速参考卡片** | 更新于 2026-03-28
