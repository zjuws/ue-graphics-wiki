# 光照贴图

## 光照贴图概述

光照贴图是预计算的光照信息，存储在纹理中。用于静态物体的高质量光照。

## 光照贴图工作流

### 步骤1: 设置光照贴图UV

```cpp
// 在编辑器中设置
// 1. 选择静态网格
// 2. 打开细节面板
// 3. 设置 Generate Lightmap UVs = true
// 4. 设置 Lightmap Coordinate Index = 1
```

### 步骤2: 配置Lightmass

```cpp
// 在DefaultEngine.ini中配置
[/Script/Engine.Lightmass]
NumIndirectLightingBounces=3
NumSkyLightingBounces=1
IndirectLightingQuality=2.0
IndirectLightingSmoothness=1.0
```

### 步骤3: 构建光照

```
在编辑器中:
1. Build → Build Lighting Only
2. 等待烘焙完成
```

### 步骤4: 验证结果

```cpp
// 检查光照贴图
// 在编辑器中启用光照贴图可视化
r.VisualizeLightmapUVs 1
```

## 光照贴图参数

### 分辨率

```cpp
// 设置光照贴图分辨率
StaticMeshComponent->SetLightmapResolution(256);

// 常见分辨率
// 64: 低质量
// 128: 标准
// 256: 高质量
// 512: 超高质量
```

### 质量设置

| 参数 | 范围 | 说明 |
|------|------|------|
| 间接光照反弹 | 1-5 | 光线反弹次数 |
| 天光反弹 | 1-2 | 天空光反弹 |
| 质量 | 0.5-4.0 | 烘焙质量 |
| 平滑度 | 0.5-2.0 | 结果平滑度 |

## 光照贴图优化

### 打包优化

```cpp
// 启用光照贴图打包
r.LightmapPacking = 1;

// 设置打包质量
r.LightmapPackingQuality = 2;
```

### 分辨率优化

```cpp
// 根据物体大小调整分辨率
// 小物体: 64-128
// 中等物体: 128-256
// 大物体: 256-512
```

### 烘焙优化

```cpp
// 降低质量加快烘焙
r.Lightmass.IndirectLightingQuality = 1.0;

// 使用较少的反弹
r.Lightmass.NumIndirectLightingBounces = 2;

// 禁用某些功能
r.Lightmass.bUseAmbientOcclusion = false;
```

## 光照贴图问题

### 问题1: 接缝

**症状**: 光照贴图之间有明显接缝

**原因**: 
- 光照贴图UV重叠
- 分辨率过低

**解决**:
```cpp
// 增加光照贴图分辨率
StaticMeshComponent->SetLightmapResolution(512);

// 检查UV是否重叠
// 在编辑器中使用UV编辑器检查
```

### 问题2: 烘焙时间过长

**症状**: 构建光照需要很长时间

**原因**: 
- 质量设置过高
- 反弹次数过多

**解决**:
```cpp
// 降低质量
r.Lightmass.IndirectLightingQuality = 1.0;

// 减少反弹
r.Lightmass.NumIndirectLightingBounces = 2;
```

### 问题3: 光照贴图伪影

**症状**: 光照贴图出现条纹或斑点

**原因**: 
- 分辨率过低
- 烘焙设置不当

**解决**:
```cpp
// 增加分辨率
StaticMeshComponent->SetLightmapResolution(512);

// 提高烘焙质量
r.Lightmass.IndirectLightingQuality = 3.0;
```

## 光照贴图内存

### 内存计算

```
内存 = 宽 × 高 × 字节数

示例:
256×256 RGBA8 = 256 × 256 × 4 = 256KB
512×512 RGBA8 = 512 × 512 × 4 = 1MB
1024×1024 RGBA8 = 1024 × 1024 × 4 = 4MB
```

### 内存优化

```cpp
// 使用较低的分辨率
StaticMeshComponent->SetLightmapResolution(128);

// 启用光照贴图打包
r.LightmapPacking = 1;

// 使用压缩格式
r.LightmapCompressionQuality = 2;
```

## 最佳实践

1. **合理设置分辨率** - 根据物体大小
2. **优化UV布局** - 避免重叠
3. **适当的烘焙质量** - 平衡质量和时间
4. **定期检查** - 验证烘焙结果
5. **内存监控** - 监控占用

## 参考资源

- [UE光照贴图文档](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Lightmass/Lightmaps/)
- [Lightmass指南](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Lightmass/)
