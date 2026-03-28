# 纹理基础

## 纹理系统概述

纹理是存储在2D或3D网格中的数据，用于为表面添加细节。UE的纹理系统支持多种格式、压缩方式和采样技术。

## 纹理类型

### 2D纹理（Texture 2D）

**最常用的纹理类型**

**用途**:
- 颜色贴图
- 法线贴图
- 粗糙度/金属度贴图
- 自发光贴图

**代码**:
```cpp
UTexture2D* Texture = LoadObject<UTexture2D>(nullptr, 
    TEXT("Texture2D'/Game/Textures/MyTexture.MyTexture'"));
```

### 立方体纹理（Cube Map）

**用于环境映射**

**用途**:
- 天空盒
- 环境反射
- 全向光照

**代码**:
```cpp
UTextureCube* CubeMap = LoadObject<UTextureCube>(nullptr,
    TEXT("TextureCube'/Game/Textures/SkyBox.SkyBox'"));
```

### 3D纹理（Volume Texture）

**用于体积效果**

**用途**:
- 体积雾
- 体积光
- 3D噪声

**代码**:
```cpp
UVolumeTexture* VolumeTexture = LoadObject<UVolumeTexture>(nullptr,
    TEXT("VolumeTexture'/Game/Textures/VolumeData.VolumeData'"));
```

### 渲染目标（Render Target）

**动态纹理**

**用途**:
- 动态反射
- 屏幕空间效果
- 实时计算

**代码**:
```cpp
UTextureRenderTarget2D* RenderTarget = NewObject<UTextureRenderTarget2D>();
RenderTarget->InitAutoFormat(1024, 1024);
```

## 纹理格式

### 像素格式（Pixel Format）

| 格式 | 通道 | 范围 | 用途 |
|------|------|------|------|
| RGBA8 | 4 | 0-255 | 颜色 |
| RGBA16F | 4 | 浮点 | HDR颜色 |
| R8 | 1 | 0-255 | 灰度 |
| R16F | 1 | 浮点 | 高精度灰度 |
| BC1/DXT1 | 3 | 压缩 | 颜色压缩 |
| BC4 | 1 | 压缩 | 法线压缩 |
| BC5 | 2 | 压缩 | 法线压缩 |

### 颜色空间

#### sRGB（标准RGB）
- 用于颜色数据
- 包含伽马校正
- 适合Base Color、Emissive

#### 线性（Linear）
- 用于数据纹理
- 无伽马校正
- 适合法线、粗糙度、金属度

**配置**:
```cpp
// 设置为线性空间
Texture->SRGB = false;
```

## 纹理采样

### 采样器状态（Sampler State）

**过滤模式**:
```hlsl
// 点采样（最近邻）
Texture.Sample(PointSampler, UV);

// 线性采样
Texture.Sample(LinearSampler, UV);

// 各向异性采样
Texture.Sample(AnisotropicSampler, UV);
```

**寻址模式**:
```hlsl
// 重复（Wrap）
// UV超出0-1范围时重复纹理

// 钳制（Clamp）
// UV超出0-1范围时使用边界颜色

// 镜像（Mirror）
// UV超出0-1范围时镜像纹理
```

### 采样代码

```hlsl
// 基础采样
float4 Color = Texture.Sample(Sampler, UV);

// 带偏移的采样
float4 Color = Texture.SampleBias(Sampler, UV, MipBias);

// 带梯度的采样
float4 Color = Texture.SampleGrad(Sampler, UV, ddx(UV), ddy(UV));

// 多重采样（MSAA）
float4 Color = Texture.Load(int3(PixelCoord, 0));
```

## 纹理分辨率

### 常见分辨率

| 分辨率 | 用途 | 说明 |
|--------|------|------|
| 512×512 | 小物体 | 低质量 |
| 1024×1024 | 标准 | 平衡 |
| 2048×2048 | 主角 | 高质量 |
| 4096×4096 | 特写 | 超高质量 |
| 8192×8192 | 电影级 | 极高质量 |

### 分辨率选择

```cpp
// 根据距离选择分辨率
float Distance = FVector::Dist(CameraPos, ActorPos);

int32 TextureResolution = 1024;
if (Distance < 500) TextureResolution = 2048;
if (Distance < 100) TextureResolution = 4096;
```

## 纹理压缩

### 压缩格式

#### DXT1（BC1）
- **压缩率**: 6:1
- **质量**: 低
- **用途**: 颜色贴图
- **特点**: 支持1位透明度

#### DXT5（BC3）
- **压缩率**: 4:1
- **质量**: 中
- **用途**: 颜色+透明度
- **特点**: 独立的透明度通道

#### BC4
- **压缩率**: 2:1
- **质量**: 高
- **用途**: 灰度贴图
- **特点**: 单通道压缩

#### BC5
- **压缩率**: 2:1
- **质量**: 高
- **用途**: 法线贴图
- **特点**: 双通道压缩

#### BC6H
- **压缩率**: 6:1
- **质量**: 高
- **用途**: HDR贴图
- **特点**: 浮点压缩

### 压缩配置

```cpp
// 在纹理属性中设置
Texture->CompressionSettings = TC_Default;  // 自动选择
Texture->CompressionSettings = TC_Masks;    // 用于遮罩
Texture->CompressionSettings = TC_Normal;   // 用于法线
Texture->CompressionSettings = TC_Grayscale; // 用于灰度
```

## 纹理流式加载

### 虚拟纹理（Virtual Texture）

**原理**: 按需加载纹理页面

**优点**:
- 减少内存占用
- 支持大型纹理
- 自动流式加载

**配置**:
```cpp
// 启用虚拟纹理
Texture->VirtualTextureStreaming = true;

// 设置页面大小
Texture->VirtualTextureSize = 128;  // 128x128像素
```

### 纹理流式加载

```cpp
// 强制加载所有MIP级别
Texture->SetForceMipLevelsToBeResident(true);

// 设置优先级
Texture->UpdateResource();
```

## MIP贴图

### 概念

MIP贴图是纹理的多分辨率版本，用于远距离采样。

**优点**:
- 提高性能
- 减少锯齿
- 改善缓存效率

### MIP级别

```
原始: 1024×1024
MIP1: 512×512
MIP2: 256×256
MIP3: 128×128
...
MIP10: 1×1
```

### 自动MIP选择

```hlsl
// GPU自动选择合适的MIP级别
float4 Color = Texture.Sample(Sampler, UV);

// 手动指定MIP级别
float4 Color = Texture.SampleLevel(Sampler, UV, MipLevel);

// 使用导数计算MIP级别
float4 Color = Texture.SampleGrad(Sampler, UV, ddx(UV), ddy(UV));
```

## 纹理打包

### 多通道打包

**原理**: 将多个灰度贴图打包到一个纹理的不同通道

**示例**:
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

**优点**:
- 减少纹理数量
- 提高缓存效率
- 降低带宽占用

## 常见问题

### 问题1: 纹理模糊

**症状**: 远处纹理看起来模糊

**原因**: 
- MIP贴图质量低
- 采样过滤不当

**解决**:
```cpp
// 提高MIP质量
Texture->MipGenSettings = TMGS_FromTextureGroup;

// 使用各向异性采样
Texture->Filter = TF_Aniso;
```

### 问题2: 纹理锯齿

**症状**: 纹理边界出现锯齿

**原因**: 
- 分辨率过低
- 采样不足

**解决**:
```hlsl
// 使用导数采样
float4 Color = Texture.SampleGrad(Sampler, UV, ddx(UV), ddy(UV));
```

### 问题3: 内存占用过大

**症状**: 游戏占用内存过多

**原因**: 
- 纹理分辨率过高
- 未压缩纹理

**解决**:
```cpp
// 启用虚拟纹理
Texture->VirtualTextureStreaming = true;

// 使用压缩格式
Texture->CompressionSettings = TC_Default;
```

## 性能优化

### 纹理优化检查清单

- [ ] 使用合适的分辨率
- [ ] 启用纹理压缩
- [ ] 使用虚拟纹理
- [ ] 打包多通道纹理
- [ ] 启用MIP贴图
- [ ] 使用各向异性采样
- [ ] 监控内存占用

### 内存计算

```
内存占用 = 宽 × 高 × 字节数 × (1 + 1/4 + 1/16 + ...)
         ≈ 宽 × 高 × 字节数 × 1.33
```

**示例**:
```
2048×2048 RGBA8:
2048 × 2048 × 4 × 1.33 ≈ 22 MB
```

## 参考资源

- [UE纹理文档](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Textures/)
- [纹理压缩指南](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Textures/Compression/)
- [虚拟纹理](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Textures/VirtualTextures/)
