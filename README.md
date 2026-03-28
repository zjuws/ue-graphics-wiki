# UE 图形渲染 Wiki

> 虚幻引擎（Unreal Engine）图形渲染系统完整知识库

## 📚 目录结构

### 核心基础
- [渲染管线概述](./docs/01-rendering-pipeline.md) - 从顶点到像素的完整流程
- [坐标系统与变换](./docs/02-coordinate-systems.md) - 模型、世界、视图、投影坐标系
- [光照与阴影](./docs/03-lighting-shadows.md) - 光照模型、阴影技术、全局光照

### 材质系统
- [材质基础](./docs/04-material-basics.md) - 材质编辑器、节点系统、参数
- [高级材质技巧](./docs/05-advanced-materials.md) - 自定义节点、动画、特效
- [物理基础渲染 (PBR)](./docs/06-pbr.md) - 金属度、粗糙度、法线贴图

### 纹理与贴图
- [纹理基础](./docs/07-textures.md) - 纹理格式、采样、压缩
- [纹理映射技术](./docs/08-texture-mapping.md) - UV映射、法线贴图、视差贴图
- [纹理优化](./docs/09-texture-optimization.md) - 流式加载、虚拟纹理、内存管理

### 高级渲染技术
- [延迟渲染](./docs/10-deferred-rendering.md) - G-Buffer、光照通道、优缺点
- [前向渲染](./docs/11-forward-rendering.md) - 实时性、透明物体、移动平台
- [后处理效果](./docs/12-post-processing.md) - 色调映射、景深、运动模糊
- [屏幕空间技术](./docs/13-screen-space-techniques.md) - SSAO、SSR、SSGI

### 性能优化
- [渲染优化](./docs/14-rendering-optimization.md) - 批处理、LOD、遮挡剔除
- [GPU优化](./docs/15-gpu-optimization.md) - 着色器优化、纹理缓存、带宽优化
- [移动平台优化](./docs/16-mobile-optimization.md) - 移动渲染、功耗管理

### 实时光照
- [动态光照](./docs/17-dynamic-lighting.md) - 实时阴影、光照探针、反射探针
- [全局光照 (GI)](./docs/18-global-illumination.md) - Lightmass、GPU GI、混合方案
- [光照贴图](./docs/19-lightmaps.md) - 烘焙、打包、质量控制

### 特殊效果
- [粒子系统](./docs/20-particle-systems.md) - Niagara、GPU粒子、优化
- [体积效果](./docs/21-volumetric-effects.md) - 体积雾、光体积、体积光
- [水体渲染](./docs/22-water-rendering.md) - 波浪模拟、反射折射、性能

### 工具与工作流
- [材质编辑器](./docs/23-material-editor.md) - 界面、快捷键、最佳实践
- [渲染调试](./docs/24-rendering-debug.md) - 可视化模式、性能分析、问题排查
- [自定义着色器](./docs/25-custom-shaders.md) - HLSL/GLSL、自定义节点、编译

### 案例研究
- [写实渲染案例](./docs/26-realistic-rendering.md) - 电影级质量、角色渲染
- [风格化渲染案例](./docs/27-stylized-rendering.md) - NPR、卡通、手绘风格
- [VR/AR渲染](./docs/28-vr-ar-rendering.md) - 立体渲染、性能要求

## 🎯 快速开始

### 新手入门
1. 从 [渲染管线概述](./docs/01-rendering-pipeline.md) 开始
2. 学习 [坐标系统与变换](./docs/02-coordinate-systems.md)
3. 理解 [光照与阴影](./docs/03-lighting-shadows.md)
4. 实践 [材质基础](./docs/04-material-basics.md)

### 进阶学习
- 深入 [高级材质技巧](./docs/05-advanced-materials.md)
- 掌握 [PBR工作流](./docs/06-pbr.md)
- 学习 [延迟渲染](./docs/10-deferred-rendering.md)

### 性能优化
- 阅读 [渲染优化](./docs/14-rendering-optimization.md)
- 学习 [GPU优化](./docs/15-gpu-optimization.md)
- 针对平台 [移动优化](./docs/16-mobile-optimization.md)

## 📖 文档说明

每个文档包含：
- **概念解释** - 理论基础
- **实现细节** - UE中的具体实现
- **最佳实践** - 推荐做法
- **常见问题** - FAQ
- **参考资源** - 扩展阅读

## 🔗 相关资源

- [官方文档](https://docs.unrealengine.com/)
- [UE源代码](https://github.com/EpicGames/UnrealEngine)
- [Unreal Fest演讲](https://www.unrealengine.com/en-US/events)
- [社区论坛](https://forums.unrealengine.com/)

## 📝 贡献指南

欢迎提交Issue和Pull Request来改进这个Wiki！

## 📄 许可证

MIT License

---

**最后更新**: 2026-03-28  
**维护者**: 术哥 🦐
