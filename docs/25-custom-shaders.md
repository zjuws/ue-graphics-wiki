# 自定义着色器

## 自定义着色器概述

自定义着色器允许创建特殊的视觉效果和优化。

## 材质中的自定义代码

### 自定义表达式

```hlsl
// 在材质编辑器中添加自定义代码
// 使用 Custom 节点

float3 CustomEffect(float3 Input)
{
    return Input * 2.0;
}
```

### 自定义节点

```cpp
// 创建自定义节点
// 1. 右键 → Material Functions → New Material Function
// 2. 编写HLSL代码
// 3. 在材质中使用
```

## 着色器编写

### 顶点着色器

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

### 像素着色器

```hlsl
// 像素着色器示例
float4 PS_Main(FPixelInput Input) : SV_Target0
{
    float4 Color = Texture.Sample(Sampler, Input.UV);
    return Color;
}
```

### 计算着色器

```hlsl
// 计算着色器示例
[numthreads(8, 8, 1)]
void CS_Main(uint3 DispatchThreadID : SV_DispatchThreadID)
{
    uint Index = DispatchThreadID.y * Width + DispatchThreadID.x;
    
    // 计算
    OutputBuffer[Index] = InputBuffer[Index] * 2.0;
}
```

## 高级着色器技巧

### 条件编译

```hlsl
// 条件编译
#if PLATFORM_WINDOWS
    // Windows特定代码
#elif PLATFORM_ANDROID
    // Android特定代码
#endif
```

### 宏定义

```hlsl
// 定义宏
#define MAX_LIGHTS 256
#define PI 3.14159

// 使用宏
for (int i = 0; i < MAX_LIGHTS; ++i) {
    // ...
}
```

### 优化技巧

```hlsl
// 避免分支
// 不好
if (Condition) {
    Result = ExpensiveCalculation();
} else {
    Result = CheapCalculation();
}

// 好
Result = lerp(CheapCalculation(), ExpensiveCalculation(), Condition);
```

## 着色器调试

### 调试输出

```hlsl
// 输出调试颜色
return float4(DebugValue, DebugValue, DebugValue, 1.0);

// 输出法线
return float4(Normal * 0.5 + 0.5, 1.0);

// 输出UV
return float4(UV, 0, 1.0);
```

### 性能分析

```cpp
// 检查着色器复杂度
r.ShaderComplexity 1

// 检查指令计数
r.ShaderComplexityMode 1
```

## 常见问题

### 问题1: 着色器编译错误

**症状**: 着色器无法编译

**原因**: 
- 语法错误
- 类型不匹配

**解决**:
```hlsl
// 检查语法
// 检查类型
// 查看编译错误信息
```

### 问题2: 着色器性能差

**症状**: 使用自定义着色器后FPS下降

**原因**: 
- 指令过多
- 纹理采样过多

**解决**:
```hlsl
// 减少指令
// 减少纹理采样
// 使用低精度
```

### 问题3: 着色器结果不对

**症状**: 着色器输出错误

**原因**: 
- 逻辑错误
- 坐标系统错误

**解决**:
```hlsl
// 使用调试输出
// 逐步验证计算
// 检查坐标系统
```

## 最佳实践

1. **从简单开始** - 逐步增加复杂度
2. **性能优先** - 优化性能
3. **充分测试** - 在各种条件下测试
4. **文档化** - 记录着色器功能
5. **版本控制** - 使用版本控制

## 参考资源

- [UE着色器文档](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Shaders/)
- [HLSL参考](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl)
- [着色器优化](https://docs.unrealengine.com/en-US/RenderingAndGraphics/Performance/GPU/)
