# 纹理优化

## 纹理优化概述

纹理优化是提高性能和降低内存占用的关键。优化包括分辨率、压缩、流式加载等多个方面。

## 分辨率优化

### 分辨率选择

| 用途 | 分辨率 | 说明 |
|------|--------|------|
| 主角 | 2K-4K | 高质量 |
| 环境 | 1K-2K | 平衡 |
| 远景 | 512-1K | 低质量 |
| UI | 512-1K | 固定大小 |

### 动态分辨率

```cpp
// 根据距离选择分辨率
float Distance = FVector::Dist(CameraPos, ActorPos);

if (Distance > 5000) {
    Texture->SetForcedMipLevel(3);  // 最低分辨率
} else if (Distance > 2000) {
    Texture->SetForcedMipLevel(2);
} else if (Distance > 500) {
    Texture->SetForcedMipLevel(1);
} else {
    Texture->SetForcedMipLevel(0);  // 最高分辨率
}
```

## 压缩优化

### 压缩格式选择

| 格式 | 压缩率 | 质量 | 用途 |
|------|--------|------|------|
| DXT1 | 6:1 | 低 | 颜色 |
| DXT5 | 4:1 | 中 | 颜色+透明 |
| BC4 | 2:1 | 高 | 灰度 |
| BC5 | 2:1 | 高 | 法线 |
| BC6H | 6:1 | 高 | HDR |

### 压缩配置

```cpp
// 设置压缩格式
Texture->CompressionSettings = TC_Default;  // 自动选择
Texture->CompressionSettings = TC_Masks;    // 遮罩
Texture->CompressionSettings = TC_Normal;   // 法线
Texture->CompressionSettings = TC_Grayscale; // 灰度
```

## 虚拟纹理

### 原理

按需加载纹理页面，减少内存占用。

```cpp
// 启用虚拟纹理
Texture->VirtualTextureStreaming = true;

// 设置页面大小
Texture->VirtualTextureSize = 128;  // 128x128像素

// 设置最大页面数
r.VirtualTexture.MaxPhysicalMemory = 512;  // 512MB
```

### 优点

- 减少内存占用
- 支持大型纹理
- 自动流式加载

## 纹理打包

### 多通道打包

```hlsl
// 打包纹理
// R: 粗糙度
// G: 金属度
// B: AO
// A: 自发光

float4 PackedTexture = Texture.Sample(UV);
float Roughness = PackedTexture.r;
float Metallic = PackedTexture.g;
float AO = PackedTexture.b;
float Emissive = PackedTexture.a;
```

### 打包优点

- 减少纹理数量
- 提高缓存效率
- 降低带宽占用

## 流式加载

### 纹理流式加载

```cpp
// 强制加载所有MIP级别
Texture->SetForceMipLevelsToBeResident(true);

// 设置优先级
Texture->StreamingPriority = 1;

// 更新资源
Texture->UpdateResource();
```

### 流式加载配置

```cpp
// 设置纹理池大小
r.Streaming.PoolSize = 2000;  // 2000MB

// 启用文件缓存
r.Streaming.UseTextureFileCache = 1;

// 设置最大纹理大小
r.Streaming.MaxTextureSize = 2048;
```

## MIP贴图优化

### MIP生成

```cpp
// 自动生成MIP
Texture->MipGenSettings = TMGS_FromTextureGroup;

// 手动生成MIP
Texture->MipGenSettings = TMGS_NoMipmaps;  // 禁用
```

### MIP质量

```cpp
// 提高MIP质量
Texture->MipGenSettings = TMGS_FromTextureGroup;

// 使用高质量过滤
Texture->Filter = TF_Aniso;
```

## 内存计算

### 内存占用公式

```
内存 = 宽 × 高 × 字节数 × (1 + 1/4 + 1/16 + ...)
     ≈ 宽 × 高 × 字节数 × 1.33
```

### 示例

```
2048×2048 RGBA8:
2048 × 2048 × 4 × 1.33 ≈ 22 MB

1024×1024 RGBA8:
1024 × 1024 × 4 × 1.33 ≈ 5.5 MB

512×512 RGBA8:
512 × 512 × 4 × 1.33 ≈ 1.4 MB
```

## 平台特定优化

### PC优化

```cpp
// 使用高分辨率纹理
r.Streaming.MaxTextureSize = 4096;

// 启用虚拟纹理
r.VirtualTexture.Enable = 1;
```

### 移动优化

```cpp
// 限制纹理分辨率
r.Streaming.MaxTextureSize = 1024;

// 使用ETC2压缩（Android）
Texture->CompressionSettings = TC_Default;

// 使用PVRTC压缩（iOS）
Texture->CompressionSettings = TC_Default;
```

## 优化检查清单

- [ ] 选择合适的分辨率
- [ ] 启用纹理压缩
- [ ] 打包多通道纹理
- [ ] 启用虚拟纹理
- [ ] 配置流式加载
- [ ] 优化MIP贴图
- [ ] 监控内存占用
- [ ] 测试性能

## 常见问题

### 问题1: 内存占用过大

**症状**: 游戏占用内存过多

**原因**: 
- 纹理分辨率过高
- 未压缩纹理

**解决**:
```cpp
// 降低分辨率
r.Streaming.MaxTextureSize = 1024;

// 启用虚拟纹理
r.VirtualTexture.Enable = 1;

// 使用压缩格式
Texture->CompressionSettings = TC_Default;
```

### 问题2: 纹理质量下降

**症状**: 压缩后纹理质量明显下降

**原因**: 
- 压缩格式不当
- 压缩质量设置过低

**解决**:
```cpp
// 使用更好的压缩格式
Texture->CompressionSettings = TC_Normal;  // 用于法线

// 提高压缩质量
Texture->CompressionQuality = 100;
```

### 问题3: 纹理加载缓慢

**症状**: 纹理加载时间长

**原因**: 
- 纹理分辨率过高
- 流式加载配置不当

**解决**:
```cpp
// 启用虚拟纹理
r.VirtualTexture.Enable = 1;

// 增加纹理池大小
r.Streaming.PoolSize = 3000;

// 提高优先级
Texture->StreamingPriority = 1;
```

## 最佳实践

1. **选择合适的分辨率** - 根据用途选择
2. **启用压缩** - 减少内存占用
3. **打包纹理** - 提高效率
4. **使用虚拟纹理** - 支持大型纹理
5. **监控内存** - 定期检查占用

## 参考资源

- [UE纹理优化](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Textures/Optimization/)
- [虚拟纹理](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Textures/VirtualTextures/)
- [纹理压缩](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Textures/Compression/)
