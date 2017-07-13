---
layout:     post
title:      "Normal Mapping"
date:       2016-10-31
author:     "ChenYong"
header-img: "img/post-bg-normalmap.jpg"
tags:
    - Graphics
    - Shader
    - 翻译
---

法线贴图技术的使用可以使得低面模型具有与高模接近的光照表现。这可极大的提高实时渲染的效率。下面的截图做了很好的对比： 
![这里写图片描述](http://img.blog.csdn.net/20161031215521493)

### How it works
模型的光照表现跟其表面的法线分布密切相关，一般参与逐像素光照计算的法线由模型顶点间的法线光栅化插值得到，可以想象由少量三角面构成的表面插值得到的法线会比较平均。而法线贴图技术就是在不增加模型面数，渲染时用贴图采样得到的法线代替插值得到的法线来参与光照计算。 
![这里写图片描述](http://img.blog.csdn.net/20161031220238480)

### Normal Map Textures
法线贴图是保存高模法线信息的贴图(也有专门的法线贴图转换工具，但它不是正常的美术法线贴图制作流程)。法线贴图有模型空间法线贴图和切线空间法线贴图。模型空间法线贴图存储的是相对于模型的法线朝向，切线空间法线贴图存储的是相对于贴图表面的法线朝向。下图显示靴子的两种法线贴图： 
![这里写图片描述](http://img.blog.csdn.net/20161031220329621)

法线贴图用颜色的rgb分量来分别存储法线的xyz分量，存储前需把法线xyz分量的取值范围[-1,1]转换到颜色rgb分量的取值范围[0,1]，读取的时候从[0,1]转换回[-1,1]。
模型空间的法线贴图颜色看起来五颜六色，因为它存储的法线是跟模型顶点法线一样是空间随机的。而切线空间的法线是相对于贴图表面的，法线的主体方向离开表面朝外，方向在(0,0,1)基础上偏移，转到RGB值范围为(0.5,0.5,1)附近，所以整体呈现亮蓝色。
切线空间的法线贴图独立于模型，一张法线贴图可以用于不同的模型，可以想象一下一个立方体，只用制作一个面的法线贴图就可复用到其他面，节省了贴图量。

### Tangent space
切线空间也叫贴图空间，因为切线空间坐标系的tangent轴和bitangent轴由所处三角形的顶点坐标和UV坐标获得（推导过程见参考文献链接）。切线空间的法线需经过TBN矩阵变换到世界坐标下进行光照计算。
![这里写图片描述](http://img.blog.csdn.net/20161031220711248) 

### Tangent space normal mapping
1.tangent向量存储在模型的顶点数据结构中（tangent的生成算法见参考文献，实际应用中，模型导出工具和引擎模型导入模块都帮我们做了），渲染时提供给顶点着色器；
2.tangent向量和顶点法线N均变换到世界坐标下，插值传递给片段着色器；
3.在片段着色器中cross(N, tangent)得到bitangent向量，构建TBN变换矩阵；
4.从法线贴图读取法线向量，用TBN变换到世界坐标下；
5.光照计算

其中第3步计算bitangent向量前，需要做正交纠正处理 。因为顶点着色器中正交的tangent和法线向量经过光栅化插值处理到达片段着色器可能已不正交。可采用Gram-Schmidt 方法进行纠正。
![这里写图片描述](http://img.blog.csdn.net/20161031221047546) 
用单位向量t减去其在单位向量n上的投影后再标准化，便得到与n正交的t

### Ending
附Unity实现的Normal Mapping开启和关闭对比图(Blinn-Phong光照模型)：
![这里写图片描述](http://img.blog.csdn.net/20161031215341251)

![这里写图片描述](http://img.blog.csdn.net/20161031215314899)
 
附Unity实现的shader代码：

```
*Shader "Custom/Normal Mapping" {
	
	Properties {
		_MainTex ("Base (RGB) Gloss (A)", 2D) = "grey" {}
		_Normal("Normal", 2D) = "bump" {}
	}	
	CGINCLUDE
	#include "UnityCG.cginc"
	#include "AutoLight.cginc"
	sampler2D _MainTex;
	float4 _MainTex_ST;
	sampler2D _Normal;
	float4 _Normal_ST;
	float4 _LightColor0;
	struct VertexInput {
		float4 vertex : POSITION;
		float3 normal : NORMAL;
		float4 tangent : TANGENT;
		float2 texcoord : TEXCOORD0;
	};
	struct VertexOutput {
		float4 pos : SV_POSITION;
		float2 uv : TEXCOORD0;
		float4 posWorld : TEXCOORD1;
		float3 normalDir : TEXCOORD2;
		float3 tangentDir : TEXCOORD3;
	};
	VertexOutput vert(VertexInput v) {
		VertexOutput o;
		o.pos = mul(UNITY_MATRIX_MVP, v.vertex);
		o.uv = TRANSFORM_TEX(v.texcoord.xy, _MainTex);
		o.posWorld = mul(unity_ObjectToWorld, v.vertex);
		o.normalDir = mul(unity_ObjectToWorld, float4(v.normal, 0)).xyz;
		o.tangentDir = normalize(mul(unity_ObjectToWorld, float4(v.tangent.xyz, 0.0)).xyz);
		return o;
	}
	half4 frag(VertexOutput i) :COLOR
	{	
		i.normalDir = normalize(i.normalDir);
		
		//Gram-Schmidt correct
		i.tangentDir = normalize(i.tangentDir - dot(i.tangentDir, i.normalDir)*i.normalDir);		
		float3 binormalDir = normalize(cross(i.normalDir, i.tangentDir));		
		float3x3 tangentTransform = float3x3(i.tangentDir, binormalDir, i.normalDir);
		float3 normalLocal = UnpackNormal(tex2D(_Normal, i.uv));//[0,1]->[-1,1]
		float3 normalDirection = normalize(mul(normalLocal, tangentTransform));
		
		float3 lightDirection = normalize(_WorldSpaceLightPos0.xyz);
		float3 viewDirection = normalize(_WorldSpaceCameraPos.xyz - i.posWorld.xyz);
		float3 halfDirection = normalize(viewDirection + lightDirection);
		////// Lighting:
		float attenuation = LIGHT_ATTENUATION(i);
		float3 attenColor = attenuation * _LightColor0.xyz;
		half4 c = tex2D(_MainTex, i.uv);
		/////// Diffuse:
		float NdotL = max(0, dot(normalDirection, lightDirection));
		float3 directDiffuse = max(0.0, NdotL) * attenColor;
		float3 diffuse = directDiffuse * c.rgb;
		////// Specular:
		float3 directSpecular = attenColor * pow(max(0, dot(halfDirection, normalDirection)), 10);
		float3 specular = directSpecular * 1;
		/// Final Color:
		float3 finalColor = diffuse + specular;
			
		return fixed4(finalColor, 1);
	}
	ENDCG
	
	SubShader
	{
		Tags
		{
			"RenderType" = "Opaque"
		}
		Pass {				
			CGPROGRAM
		
			#pragma vertex vert
			#pragma fragment frag
			#pragma fragmentoption ARB_precision_hint_fastest 
		
			ENDCG
		}
	}
	FallBack "Mobile/Diffuse"
}*

```
附参考文献：

http://learnopengl.com/#!Advanced-Lighting/Normal-Mapping
http://ogldev.atspace.co.uk/www/tutorial26/tutorial26.html
https://en.wikipedia.org/wiki/Normal_mapping