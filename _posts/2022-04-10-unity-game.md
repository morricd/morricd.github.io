---
title: Unity Shader 初次探索
author: morric
date: 2022-04-10 23:10:00 +0800
categories: [开发记录, Unity]
tags: [unity, games]
---

我最早进入编程这行，起点是游戏——Flash 时代的 AS3.0。后来苹果放弃 Flash，Adobe 也随之停止维护，Flash 游戏开发逐渐走向历史。但对游戏编程的热情从未消退，这些年一直断断续续地学习各类游戏框架。Unity 是目前移动端最主流的游戏引擎，学习 Shader 是深入 Unity 渲染体系绕不开的一步。

本文记录第一次系统学习 Unity Shader 的过程，包括顶点着色器、片元着色器、Surface Shader 的基础用法，以及用 Shader 实现溶解效果的实践。

---

## 一、Shader 是什么

Shader（着色器）是运行在 GPU 上的程序，决定了物体表面每个像素最终显示的颜色。Unity 中的 Shader 用 ShaderLab 语法编写，内部可以嵌入 HLSL/CG 代码。

Unity Shader 主要有三种类型：

| 类型            | 特点                             | 适用场景               |
| --------------- | -------------------------------- | ---------------------- |
| 顶点/片元着色器 | 完全手动控制渲染流程，灵活度最高 | 自定义特效、底层渲染   |
| Surface Shader  | Unity 封装了光照计算，编写简单   | 需要响应光照的物体表面 |
| 固定管线 Shader | 已废弃，不建议使用               | —                      |

---

## 二、顶点着色器与片元着色器

顶点着色器和片元着色器是 GPU 渲染管线中两个核心阶段：

- **顶点着色器（Vertex Shader）**：处理每个顶点的位置、法线、UV 等数据，将模型空间坐标转换到裁剪空间；
- **片元着色器（Fragment Shader）**：处理每个像素的颜色，决定最终输出到屏幕上的颜色值。

### 2.1 最简单的顶点/片元 Shader

下面是一个输出纯色的基础 Shader：

```hlsl
Shader "Custom/SimpleColor"
{
    Properties
    {
        _Color ("Color", Color) = (1, 1, 1, 1)
    }

    SubShader
    {
        Tags { "RenderType"="Opaque" }

        Pass
        {
            CGPROGRAM
            #pragma vertex vert      // 指定顶点着色器函数
            #pragma fragment frag    // 指定片元着色器函数

            #include "UnityCG.cginc"

            fixed4 _Color;

            // 顶点着色器输入
            struct appdata
            {
                float4 vertex : POSITION;
            };

            // 顶点着色器输出（传给片元着色器）
            struct v2f
            {
                float4 pos : SV_POSITION;
            };

            // 顶点着色器：将模型顶点转换到裁剪空间
            v2f vert(appdata v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                return o;
            }

            // 片元着色器：输出颜色
            fixed4 frag(v2f i) : SV_Target
            {
                return _Color;
            }

            ENDCG
        }
    }
}
```

### 2.2 带 UV 纹理采样的 Shader

在顶点着色器中传递 UV 坐标，片元着色器中进行纹理采样：

```hlsl
Shader "Custom/TextureSample"
{
    Properties
    {
        _MainTex ("Main Texture", 2D) = "white" {}
    }

    SubShader
    {
        Tags { "RenderType"="Opaque" }

        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"

            sampler2D _MainTex;
            float4 _MainTex_ST; // 纹理的 Tiling 和 Offset

            struct appdata
            {
                float4 vertex : POSITION;
                float2 uv     : TEXCOORD0;
            };

            struct v2f
            {
                float4 pos : SV_POSITION;
                float2 uv  : TEXCOORD0;
            };

            v2f vert(appdata v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                // TRANSFORM_TEX 处理 Tiling 和 Offset
                o.uv = TRANSFORM_TEX(v.uv, _MainTex);
                return o;
            }

            fixed4 frag(v2f i) : SV_Target
            {
                return tex2D(_MainTex, i.uv);
            }

            ENDCG
        }
    }
}
```

---

## 三、Surface Shader

Surface Shader 是 Unity 对光照计算的封装，不需要手动处理顶点变换和光照模型，编写起来比顶点/片元 Shader 简单很多，适合大多数需要响应场景光照的材质。

### 3.1 基础 Surface Shader

```hlsl
Shader "Custom/BasicSurface"
{
    Properties
    {
        _MainTex ("Albedo (RGB)", 2D) = "white" {}
        _Color   ("Color", Color) = (1, 1, 1, 1)
        _Glossiness ("Smoothness", Range(0,1)) = 0.5
        _Metallic   ("Metallic", Range(0,1)) = 0.0
    }

    SubShader
    {
        Tags { "RenderType"="Opaque" }

        CGPROGRAM
        // 使用 Standard 光照模型，支持全局光照
        #pragma surface surf Standard fullforwardshadows
        #pragma target 3.0

        sampler2D _MainTex;
        fixed4    _Color;
        half      _Glossiness;
        half      _Metallic;

        struct Input
        {
            float2 uv_MainTex;
        };

        void surf(Input IN, inout SurfaceOutputStandard o)
        {
            fixed4 c = tex2D(_MainTex, IN.uv_MainTex) * _Color;
            o.Albedo    = c.rgb;
            o.Metallic  = _Metallic;
            o.Smoothness = _Glossiness;
            o.Alpha     = c.a;
        }
        ENDCG
    }

    FallBack "Diffuse"
}
```

与顶点/片元 Shader 的区别在于：只需要实现 `surf` 函数，填写表面属性（颜色、金属度、光滑度等），Unity 会自动处理光照计算、阴影接收等细节。

---

## 四、溶解效果实现

溶解效果（Dissolve）是游戏中常见的特效，常用于角色死亡、传送、技能触发等场景。原理是：通过一张噪声贴图，根据噪声值与一个可控阈值进行比较，低于阈值的像素直接丢弃（`clip`），同时在溶解边缘叠加发光颜色。

### 4.1 实现思路

1. 采样噪声贴图，获取当前像素的噪声值（0~1）；
2. 将噪声值与溶解进度 `_DissolveAmount` 比较，低于阈值的像素执行 `clip` 丢弃；
3. 在阈值边缘附近（`_EdgeWidth` 控制宽度）叠加边缘发光颜色 `_EdgeColor`。

### 4.2 完整 Shader 代码

```hlsl
Shader "Custom/Dissolve"
{
    Properties
    {
        _MainTex       ("Main Texture", 2D)    = "white" {}
        _NoiseTex      ("Noise Texture", 2D)   = "white" {}
        _DissolveAmount("Dissolve Amount", Range(0, 1)) = 0
        _EdgeWidth     ("Edge Width", Range(0, 0.2))    = 0.05
        _EdgeColor     ("Edge Color", Color)   = (1, 0.5, 0, 1)
    }

    SubShader
    {
        Tags { "RenderType"="Transparent" "Queue"="Transparent" }

        Pass
        {
            Cull Off // 双面渲染，溶解时能看到背面

            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"

            sampler2D _MainTex;
            sampler2D _NoiseTex;
            float4    _MainTex_ST;
            float     _DissolveAmount;
            float     _EdgeWidth;
            fixed4    _EdgeColor;

            struct appdata
            {
                float4 vertex : POSITION;
                float2 uv     : TEXCOORD0;
            };

            struct v2f
            {
                float4 pos : SV_POSITION;
                float2 uv  : TEXCOORD0;
            };

            v2f vert(appdata v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.uv  = TRANSFORM_TEX(v.uv, _MainTex);
                return o;
            }

            fixed4 frag(v2f i) : SV_Target
            {
                // 采样噪声贴图，取 r 通道作为溶解值
                float noise = tex2D(_NoiseTex, i.uv).r;

                // 低于阈值的像素直接丢弃（溶解消失）
                clip(noise - _DissolveAmount);

                // 在边缘范围内叠加发光颜色
                fixed4 col = tex2D(_MainTex, i.uv);
                if (noise - _DissolveAmount < _EdgeWidth)
                {
                    // 越接近边缘，发光越强
                    float edgeFactor = 1 - (noise - _DissolveAmount) / _EdgeWidth;
                    col.rgb = lerp(col.rgb, _EdgeColor.rgb, edgeFactor);
                }

                return col;
            }

            ENDCG
        }
    }
}
```

### 4.3 在脚本中控制溶解进度

通过脚本动态修改 `_DissolveAmount` 参数，实现随时间溶解的动画效果：

```csharp
using UnityEngine;

public class DissolveEffect : MonoBehaviour
{
    public float dissolveSpeed = 0.5f;

    private Material _material;
    private float _dissolveAmount = 0f;
    private bool _isDissolving = false;

    void Start()
    {
        _material = GetComponent<Renderer>().material;
    }

    // 调用此方法开始溶解
    public void StartDissolve()
    {
        _isDissolving = true;
    }

    void Update()
    {
        if (!_isDissolving) return;

        _dissolveAmount += Time.deltaTime * dissolveSpeed;
        _dissolveAmount = Mathf.Clamp01(_dissolveAmount);
        _material.SetFloat("_DissolveAmount", _dissolveAmount);

        if (_dissolveAmount >= 1f)
        {
            _isDissolving = false;
            gameObject.SetActive(false); // 完全溶解后隐藏物体
        }
    }
}
```

---

## 五、顶点/片元 Shader 与 Surface Shader 的选择

|            | 顶点/片元 Shader          | Surface Shader         |
| ---------- | ------------------------- | ---------------------- |
| 编写复杂度 | 高，需要手动处理所有阶段  | 低，Unity 自动处理光照 |
| 灵活性     | 极高，完全可控            | 受限于 Unity 光照模型  |
| 光照支持   | 需要手动实现              | 自动支持               |
| 适用场景   | 自定义特效、后处理        | 标准材质、受光物体     |
| 溶解效果   | ✅ 推荐（`clip` 控制精确） | 也可实现，但略复杂     |

---

## 总结

第一次系统学习 Shader，最大的收获是理解了 GPU 渲染管线的基本流程：顶点着色器负责坐标变换，片元着色器负责颜色输出，两者之间通过插值传递数据。Surface Shader 是 Unity 对这套流程的封装，降低了光照材质的编写门槛。

溶解效果是 Shader 入门的经典练习，核心是 `clip` 函数——它让我第一次真正理解了"丢弃像素"这个概念，也让我意识到 Shader 能做的事远不止改改颜色那么简单。后续计划继续深入学习法线贴图、折射效果和后处理 Shader。