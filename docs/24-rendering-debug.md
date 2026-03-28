# 渲染调试

## 渲染调试概述

渲染调试工具帮助诊断和优化渲染问题。

## 可视化模式

### 光照可视化

```cpp
// 可视化光照贡献
r.VisualizeLight 1

// 可视化光照类型
r.VisualizeLightingMode 1  // 0=默认, 1=光照, 2=阴影
```

### 法线可视化

```cpp
// 可视化法线贴图
r.VisualizeNormalMap 1

// 可视化世界法线
r.VisualizeWorldNormal 1
```

### 深度可视化

```cpp
// 可视化深度缓冲
r.VisualizeDepth 1

// 可视化深度范围
r.VisualizeDepthRange 1
```

### 材质复杂度

```cpp
// 可视化材质复杂度
r.ShaderComplexity 1

// 可视化指令计数
r.ShaderComplexityMode 1
```

## 性能分析

### Stat命令

```cpp
// 总体性能
stat unit

// 详细渲染统计
stat scenerendering

// GPU时间
stat gpu

// 内存统计
stat memory

// 单位图表
stat unitgraph
```

### 性能分析器

```cpp
// 启用性能分析
r.ProfileGPU 1

// 启用CPU分析
r.ProfileCPU 1
```

## 调试工具

### 绘制调试

```cpp
// 绘制调试线
DrawDebugLine(GetWorld(), Start, End, FColor::Red, false, 5.0f);

// 绘制调试球
DrawDebugSphere(GetWorld(), Center, Radius, 16, FColor::Green, false, 5.0f);

// 绘制调试框
DrawDebugBox(GetWorld(), Center, Extent, FColor::Blue, false, 5.0f);
```

### 屏幕调试

```cpp
// 在屏幕上显示文本
GEngine->AddOnScreenDebugMessage(-1, 5.0f, FColor::Yellow, 
    FString::Printf(TEXT("FPS: %.2f"), 1.0f / DeltaTime));
```

## 常见调试场景

### 调试光照问题

```cpp
// 1. 启用光照可视化
r.VisualizeLight 1

// 2. 检查光照贡献
stat scenerendering

// 3. 检查阴影
r.VisualizeShadows 1
```

### 调试材质问题

```cpp
// 1. 可视化法线
r.VisualizeNormalMap 1

// 2. 检查材质复杂度
r.ShaderComplexity 1

// 3. 检查纹理采样
r.VisualizeTextures 1
```

### 调试性能问题

```cpp
// 1. 检查总体性能
stat unit

// 2. 检查GPU时间
stat gpu

// 3. 检查内存
stat memory

// 4. 分析瓶颈
r.ProfileGPU 1
```

## 常见问题

### 问题1: 光照不正确

**症状**: 光照看起来不对

**调试步骤**:
```cpp
// 1. 启用光照可视化
r.VisualizeLight 1

// 2. 检查法线
r.VisualizeNormalMap 1

// 3. 检查阴影
r.VisualizeShadows 1
```

### 问题2: 性能下降

**症状**: FPS下降

**调试步骤**:
```cpp
// 1. 检查总体性能
stat unit

// 2. 检查GPU时间
stat gpu

// 3. 检查CPU时间
stat scenerendering

// 4. 找到瓶颈
r.ProfileGPU 1
```

### 问题3: 纹理问题

**症状**: 纹理显示错误

**调试步骤**:
```cpp
// 1. 检查UV
r.VisualizeTextures 1

// 2. 检查纹理分辨率
r.Streaming.MaxTextureSize

// 3. 检查压缩
Texture->CompressionSettings
```

## 最佳实践

1. **定期分析** - 定期检查性能
2. **使用可视化** - 利用可视化工具
3. **记录数据** - 记录性能数据
4. **对比分析** - 对比优化前后
5. **文档化** - 记录问题和解决方案

## 参考资源

- [UE调试文档](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Debugging/)
- [性能分析](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Performance/)
