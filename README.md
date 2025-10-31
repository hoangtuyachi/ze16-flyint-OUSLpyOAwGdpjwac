> [【从UnityURP开始探索游戏渲染】](https://github.com)**专栏-直达**

色差(Chromatic Aberration)是Unity URP后处理系统中的一种视觉效果，模拟真实相机镜头因折射率差异导致不同波长光线无法聚焦在同一点而产生的彩色边缘现象。（类似抖音Log的颜色分离偏差的效果）

# **核心功能与用途**

## ‌**视觉效果**‌：

* 在图像高对比度边缘（如物体轮廓）产生RGB通道分离的彩色条纹，常见红/蓝偏移

## ‌**应用场景**‌：

* 模拟老旧相机、镜头缺陷的复古风格
* 表现角色醉酒、中毒等特殊状态
* 科幻场景中增强高科技设备的光学畸变感

# **发展历史**

* ‌**早期实现**‌：2017年前通过Shader手动分离RGB通道采样实现基础效果
* ‌**URP集成**‌：2019年随URP 7.2+版本成为标准后处理效果，基于Volume系统管理
* ‌**性能优化**‌：2021年后引入Quality分级控制，支持不同硬件配置

# **实现原理**

色差（Chromatic Aberration）在URP后处理中的底层实现基于RGB通道分离的屏幕空间着色技术，其核心原理是通过对红、绿、蓝三通道进行差异化偏移采样，模拟光线折射率差异导致的波长分离现象.

## **物理光学基础**

* 色差效应源于镜头对不同波长光线的折射率差异（n(λ)），导致短波长（蓝光）比长波长（红光）产生更大折射角，最终在成像平面形成RGB通道的像素级偏移

## **URP实现流程**

技术实现基于屏幕空间着色，核心步骤：

* 计算像素到屏幕中心的归一化距离
* 根据距离对R/G/B通道进行不同偏移采样
* 混合原始像素与偏移采样结果

### ‌**坐标归一化**‌

计算当前像素到屏幕中心的UV向量，并归一化为径向距离值（0-1范围）：

```
|  |  |
| --- | --- |
|  | hlsl |
|  | float2 centerUV = (i.uv - 0.5) * 2.0; // 中心坐标系 |
|  | float radius = length(centerUV);      // 径向距离 |
```

### ‌**通道偏移计算**‌

根据距离应用非线性强度曲线，生成各通道偏移向量（红蓝通道反向偏移）：

```
|  |  |
| --- | --- |
|  | hlsl |
|  | float3 offset = radius * _Intensity * float3( |
|  | -_SpectralTex.r,  // 红通道左偏 |
|  | 0,                // 绿通道不偏移 |
|  | _SpectralTex.b    // 蓝通道右偏 |
|  | ); |
```

### ‌**多通道采样混合**‌

对原始纹理进行三次采样并加权混合：

```
|  |  |
| --- | --- |
|  | hlsl |
|  | half4 frag(v2f i) : SV_Target { |
|  | half4 r = tex2D(_MainTex, i.uv + offset.rg); |
|  | half4 g = tex2D(_MainTex, i.uv); |
|  | half4 b = tex2D(_MainTex, i.uv + offset.bg); |
|  | return half4(r.r, g.g, b.b, 1.0); |
|  | } |
```

## **关键参数控制**

| 技术参数 | 作用机制 | 物理对应关系 |
| --- | --- | --- |
| `_Intensity` | 控制色散偏移量大小 | 镜头折射率差异程度 |
| `_SpectralTex` | 自定义色散颜色分布 | 镜头镀膜光谱特性 |
| `radius` | 非线性距离衰减系数 | 球面像差修正 |

### **性能优化策略**

* ‌**质量分级**‌

  + Low：仅水平方向偏移（节省1次采样）
  + High：增加垂直方向偏移（4次采样+高斯模糊）
* ‌**边缘遮罩**‌

  通过深度/法线检测限制色差作用区域，避免中心物体失真：

  ```
  |  |  |
  | --- | --- |
  |  | hlsl |
  |  | float edgeMask = 1 - saturate(depth * _EdgeThreshold); |
  |  | offset *= edgeMask; |
  ```

完整实现示例需结合URP的`FullScreenPassRendererFeature`扩展，具体可参考HDRP的通道分离算法移植方案.

# **URP完整实现流程 参数详解**

## 创建Volume对象

```
|  |  |
| --- | --- |
|  | GameObject volumeObj = new GameObject("PostProcessVolume"); |
|  | Volume volume = volumeObj.AddComponent(); |
|  | volume.isGlobal = true; |
```

## 添加色差覆盖

```
|  |  |
| --- | --- |
|  | ChromaticAberration caEffect = volume.profile.Add(); |
|  | caEffect.active = true; |
```

## 参数配置

```
|  |  |
| --- | --- |
|  | caEffect.intensity.overrideState = true; |
|  | caEffect.intensity.value = 0.5f; // 强度值0-1 |
```

| 参数 | 说明 | 典型值 | 用例 |
| --- | --- | --- | --- |
| Intensity | 色散强度 | 0.3-0.8 | 醉酒效果用0.6+ |
| Spectral Lut | 自定义色散颜色纹理 | 空=默认 | 科幻风格用蓝紫色系 |
| Quality | 采样质量等级 | Low/Medium/High | 移动端建议Low |

* ChromaticController.cs

  ```
  |  |  |
  | --- | --- |
  |  | using UnityEngine.Rendering; |
  |  | using UnityEngine.Rendering.Universal; |
  |  |  |
  |  | public class ChromaticController : MonoBehaviour { |
  |  | [Range(0, 1)] public float intensity; |
  |  | private ChromaticAberration _ca; |
  |  |  |
  |  | void Start() { |
  |  | VolumeProfile profile = FindObjectOfType().profile; |
  |  | profile.TryGet(out _ca); |
  |  | } |
  |  |  |
  |  | void Update() { |
  |  | _ca.intensity.value = intensity; |
  |  | } |
  |  | } |
  ```

# **实际应用技巧**

## ‌**动态控制**‌：

* 通过脚本在角色受伤时增强强度值

## ‌**区域限制**‌：

* 结合遮罩纹理只对屏幕边缘应用效果

## ‌**性能优化**‌：

* VR项目中建议关闭或使用Low质量
* 完整项目需确保：
  + URP Asset中启用Post-processing
  + 相机添加Volume组件并设置Layer匹配
  + 在Window > Package Manager中安装Post Processing包

---

> [【从UnityURP开始探索游戏渲染】](https://github.com):[橘子云加速器](https://meibokele.com)**专栏-直达**
> （欢迎*点赞留言*探讨，更多人加入进来能更加完善这个探索的过程，🙏）
