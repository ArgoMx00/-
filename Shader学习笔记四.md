<table><tr><td bgcolor=orange>高级纹理</td></tr></table>

我们之前学习了关于基础纹理的内容，这些内容包括法线纹理、渐变纹理和遮罩纹理等。
这些纹理尽管不同，但是他们都属于低维纹理。我们接下来将接触如何使用立方体纹理实现
环境映射。然后，我们还会接触如何使用渲染纹理、以及了解什么是程序纹理等。

使用立方体纹理的好处就是简单快速，而且得到的效果非常好。但是也有一些缺点，例如
当场景中引入了新的物体、光源、或者物体发生移动时，我们就需要重新生成立方体纹理。
立方体纹理在实时渲染中有很多应用，最常见的就是用于天空盒子以及环境映射。

一、天空盒子。

我们设定一个材质，然后选择默认Shader为6 Sided，之后将六张图片贴进去，得到一个环境天空盒 。

我们来看一下天空盒的效果：

![](https://i.loli.net/2018/07/05/5b3da4e62fca7.png)

在Unity中，天空盒是在所有不透明物体之后渲染的，而其背后使用的额王哥是一个立方体或者一个细分后的球体。

在Unity5中，创建用于环境映射的立方体纹理的方法有三种：第一种方法是直接由一些特殊布局的纹理创建。第二种
方法是手动创建一个CubeMap资源，再把6张图赋给他；第三种方法是由脚本生成。

接下来我们再讨论一下反射。使用了反射效果的物体通常看起来就像镀了层金属。想要模拟反射的效果很简单，我们只
需要通过入射光线的方向和表面的法线来计算反射方向即可。再利用反射方向对立方体纹理采样即可。

在进入实践环节之前，我们要知道如何进行环境渲染。我们首先建一个空的Gameobject，然后记录位子。然后新建一个cubmap
，再之后点击gameOjbect->Render into cubemap，把空的gameobject和cubemap拖入进去。点击Render之后，再建立新的
物体，将Cubemap设置进去，并且设置好其Shader即可。

如果Unity中没有自带环境渲染的按键(得到环境映射的立方体纹理)，那么我们可以手动导入一个脚本来进行操作：

```c#
using UnityEngine;
using UnityEditor;
using System.Collections;

public class RenderCubemapWizard : ScriptableWizard
{

    public Transform renderFromPosition;
    public Cubemap cubemap;

    void OnWizardUpdate()
    {
        helpString = "Select transform to render from and cubemap to render into";
        isValid = (renderFromPosition != null) && (cubemap != null);
    }

    void OnWizardCreate()
    {
        // create temporary camera for rendering
        GameObject go = new GameObject("CubemapCamera");
        go.AddComponent<Camera>();
        // place it on the object
        go.transform.position = renderFromPosition.position;
        go.transform.rotation = Quaternion.identity;
        // render into cubemap		
        go.GetComponent<Camera>().RenderToCubemap(cubemap);

        // destroy temporary camera
        DestroyImmediate(go);
    }

    [MenuItem("GameObject/Render into Cubemap")]
    static void RenderCubemap()
    {
        ScriptableWizard.DisplayWizard<RenderCubemapWizard>(
            "Render cubemap", "Render!");
    }
}

```

那么我们接下来进行反射的实践：


```Shader

// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'

Shader"Unity Shaders Book/Chapter 10/反射效果"
{
	Properties{
		_Color("Color Tint",Color)=(1,1,1,1)
		_ReflectColor("Reflection Color",Color)=(1,1,1,1)
		_ReflectAmount("Reflect Amount",Range(0,1))=1
		_Cubemap("Refect Cubemap",Cube)="_Skybox"{}
	}
	SubShader{
		//默认的渲染队列Geometry，大多数物体都使用这个队列，不透明物体也使用这个队列
		Tags{"RenderType"="Opaque" "Queue"="Geometry"}
		Pass{
			Tags{"LightMode"="ForwardBase"}
			CGPROGRAM

			//为了得到更加准确的光照值
			#pragma multi_compile_fwdbase

			#pragma vertex vert
			#pragma fragment frag

			#include"Lighting.cginc"
			#include"AutoLight.cginc"

			fixed4 _Color;
			fixed4 _ReflectColor;
			fixed _ReflectAmount;
			samplerCUBE _Cubemap;

			struct a2v{
				float4 vertex:POSITION;
				float3 normal:NORMAL;
			};

			struct v2f{
				float4 pos:SV_POSITION;
				float3 worldPos:TEXCOORD0;
				float3 worldNormal:TEXCOORD1;
				float3 worldViewDir:TEXCOORD2;
				float3 worldRefl:TEXCOORD3;
				SHADOW_COORDS(4)
			};

			v2f vert(a2v v){
				v2f o;
				o.pos=UnityObjectToClipPos(v.vertex);
				o.worldPos=mul(unity_ObjectToWorld,v.vertex).xyz;
				o.worldNormal=UnityObjectToWorldNormal(v.normal);
				o.worldViewDir=UnityWorldSpaceViewDir(o.worldPos);

				//通过观察方向和法线方向得到反射方向
				//物体反射到摄像机中的光线方向，可以由光路可逆的原则来反向求得。
				//也就是说，我们可以计算视角方向关于顶点法线发的反射方向来求得入射方向
				o.worldRefl=reflect(-o.worldViewDir,o.worldNormal);

				TRANSFER_SHADOW(o);

				return o;
			}

			fixed4 frag(v2f i):SV_Target{
				fixed3 worldNormal=normalize(i.worldNormal);
				fixed3 worldLightDir=normalize(UnityWorldSpaceLightDir(i.worldPos));
				fixed3 worldViewDir=normalize(i.worldViewDir);
				fixed3 ambient=UNITY_LIGHTMODEL_AMBIENT.xyz;
				fixed3 diffuse=_LightColor0.rgb*_Color.rgb*max(0,dot(worldNormal,worldLightDir));

				//对立方体纹理的采样需要使用CG的texCUBE函数。注意到，在计算中
				//我们并没有对worldRefl归一化。这是因为，，用于采样的参数仅仅是
				//作为方向变量传递给函数的因此我们没有必要进行归一化操作。
				fixed3 reflection=texCUBE(_Cubemap,i.worldRefl).rgb*_ReflectColor.rgb;
			
				UNITY_LIGHT_ATTENUATION(atten,i,i.worldPos);
				//lerp函数用于混合颜色，插值计算渐变颜色
				fixed3 color=ambient+lerp(diffuse,reflection,_ReflectAmount)*atten;
				return fixed4(color,1.0);
			}
			ENDCG
		}
	}
	Fallback"Diffuse"
	//FallBack "Reflective/VertexLit"
}
```

在这里，我们对于反射的操作步骤稍微进行一个总结：

首先创建一个cubemap，再创建一个空的gameobject，将这个cubemap通过render into cubemap操作按钮
，将cubemap的六个面进行环境渲染。得到一个可用的cubemap，然后我们创建一个物体gameobject，调整
位子和空gameobject位子和方向一样，然后将cubemap拖拽给物体的shader中即可。

实践效果：

![](https://i.loli.net/2018/07/06/5b3efd071d3b5.png)

二、折射效果

图形学的第一准则：看起来是对的，那么结果就是对的。

我们这里使用斯涅尔定律来计算反射角。当光从介质1沿着表面法线夹角为α的方向斜射入介质2时，我们可以
使用如下公式计算折射光线与法线的夹角α2：

![](https://i.loli.net/2018/07/06/5b3f004744392.png)

通常来说，当得到折射方向后我们就会直接使用它来对立方体纹理进行采样，单这是不符合物理规律的，对于
一个透明物体来说，一种更加精确的模拟方法需要进行计算两次折射：一次是当光线进入内部的时候，另一次
是当它从内部射出的时候。但是，想要在实时渲染中模拟计算两次折射，其实是比较复杂的，而只模拟一次又
看起来“像那么回事”，所以我们就只进行模拟一次了。

![](https://i.loli.net/2018/07/06/5b3f01469761b.png)

实践代码：

```Shader
Shader"Unity Shaders Book/Chapter 10/折射效果"
{
	Properties{
		//材质颜色、折射颜色、折射程度、透射比、cubemap
		_Color("Color Tint",Color)=(1,1,1,1)
		_RefractColor("Refraction Color",Color)=(1,1,1,1)
		_RefractAmount("Refraction Amount",Range(0,1))=1
		_RefractRatio("Refraction Ratio",Range(0.1,1))=0.5
		_Cubemap ("Refraction Cubemap", Cube) = "_Skybox" {}
	}
	SubShader{
		Tags{"RenderType"="Opaque" "Queue"="Geometry"}
		Pass{
			Tags{"LightMode"="ForwardBase"}
			CGPROGRAM

			#pragma multi_compile_fwdbase	
			
			#pragma vertex vert
			#pragma fragment frag
			
			#include "Lighting.cginc"
			#include "AutoLight.cginc"
			
			fixed4 _Color;
			fixed4 _RefractColor;
			float _RefractAmount;
			fixed _RefractRatio;
			samplerCUBE _Cubemap;

			struct a2v {
				float4 vertex : POSITION;
				float3 normal : NORMAL;
			};
			
			struct v2f {
				float4 pos : SV_POSITION;
				float3 worldPos : TEXCOORD0;
				fixed3 worldNormal : TEXCOORD1;
				fixed3 worldViewDir : TEXCOORD2;
				fixed3 worldRefr : TEXCOORD3;
				SHADOW_COORDS(4)
			};

			v2f vert(a2v v){
				v2f o;
				o.pos=UnityObjectToClipPos(v.vertex);
				o.worldPos=mul(unity_ObjectToWorld,v.vertex).xyz;
				o.worldNormal=UnityObjectToWorldNormal(v.normal);
				o.worldViewDir=UnityWorldSpaceViewDir(o.worldPos);


				//我们使用了CG的refract函数来计算折射方向，他的第一个参数即为入射光线
				//的方向，它必须是归一化后的矢量，第二个参数是表面的法线，也是需要归一化的
				//第三个参数是入射光线所在介质的折射率和折射光线所在介质的折射率之间的比值
				//例如光是从空气射入到玻璃表面，那么这个比值就是1/1.5。这个函数的返回值就是
				//计算所得到的折射方向。
				o.worldRefr=refract(-normalize(o.worldViewDir),normalize(o.worldNormal),_RefractRatio);				

				TRANSFER_SHADOW(o);

				return o;
			}

			fixed4 frag(v2f i):SV_Target{
				fixed3 worldNormal=normalize(i.worldNormal);
				fixed3 worldLightDir=normalize(UnityWorldSpaceLightDir(i.worldPos));
				fixed3 worldViewDir=normalize(i.worldViewDir);
				fixed3 ambient=UNITY_LIGHTMODEL_AMBIENT.xyz;
				fixed3 diffuse=_LightColor0.rgb*_Color.rgb*max(0,dot(worldNormal,worldLightDir));
				
				fixed3 refraction=texCUBE(_Cubemap,i.worldRefr).rgb*_RefractColor.rgb;
				UNITY_LIGHT_ATTENUATION(atten,i,i.worldPos);

				fixed3 color=ambient+lerp(diffuse,refraction,_RefractAmount)*atten;

				return fixed4(color,1.0);

			}


			ENDCG
		}
	}
	Fallback"Diffuse"
}
```

实践效果：

![](https://i.loli.net/2018/07/06/5b3f06216bf39.png)








