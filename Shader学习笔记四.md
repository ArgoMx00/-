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

接下来书籍上的菲涅耳反射我们就先不去深入探究了，如有需要，回头参考一下就行了。

三、渲染纹理

在之前的学习中，一个摄像机的渲染结果会输出到颜色缓冲中，并且显示到我们的屏幕上。现代的GPU允许
我们把整个三维场景渲染到一个中间缓冲中，即渲染目标纹理，而不是传统的缓冲或后备缓冲。与之相关的
是多重渲染目标，这种奇数指的是GPU允许我们把场景同时渲染到多个渲染目标纹理中，而不再需要为每个
渲染目标纹理单独渲染完整的场景。延迟渲染就是使用多重渲染目标的一个应用。

Unity为渲染目标纹理定义了一种专门的纹理类型---渲染纹理。在Unity中使用渲染纹理通常有两种方式:
一种方式是在Project目录下创建一个渲染纹理，然后把这个摄像机的渲染目标设置成渲染纹理，这样一来
摄像机的渲染结果就会实时更新到渲染纹理中，而不会显示在屏幕上。另一种方式就是在屏幕后处理时使用
GrabPass命令或者OnRenderImage函数来获取当前屏幕的图像，Unity会把这个屏幕图像放到一张和屏幕
分辨率等同的渲染纹理中。下边我们就自定义Pass来尝试实现各种屏幕特效。

四、镜子效果

我们的镜面效果是基于渲染纹理去处理的，核心思想就是把一个摄像机放置在镜子上，然后让他观察到的效果
渲染到物体上去。那么这里我们就需要渲染纹理Texture了。

我们的工作顺序是：

①创建一个Texture，然后把这个Texture赋给一个摄像机，再将这个Texture赋给Shader（Matrial）中。
然后把这个Shader（Matrial）赋给镜面，最后调整摄像机的观察方向，就能够得到实践效果了。
不过效果放大之后挺一般的。这里学习一下渲染纹理的一种应用即可，不作过多的拓展了；

②实践代码：

```Shader
Shader"Unity Shaders Book/Chapter 10/镜面效果"
{
	Properties{
		_MainTex("Main Tex",2D)="white"{}
	}
	SubShader{
		Tags{"RenderType"="Opaque" "Queue"="Geometry"}
		Pass{
			CGPROGRAM 
			
			#pragma vertex vert
			#pragma fragment frag

			sampler2D _MainTex;

			struct a2v{
				float4 vertex : POSITION;
				float3 texcoord: TEXCOORD0;
			};
			struct v2f{
				float4 pos:SV_POSITION;
				float2 uv:TEXCOORD0;
			};
			
			v2f vert(a2v v){
				v2f o;
				o.pos=UnityObjectToClipPos(v.vertex);
				//处理uv贴图，一开始原封不动的赋过来。
				o.uv=v.texcoord;
				//然后再反转一下x轴，即左右对称一下（因为镜子是左右反对称的。）
				o.uv.x=1-o.uv.x;
				return o;
			}
			fixed4 frag(v2f i):SV_Target{
				return tex2D(_MainTex,i.uv);
			}
			
			ENDCG
		}
	}
}
```

实践效果：

![](https://i.loli.net/2018/07/06/5b3f11f1a4065.png)

四、玻璃效果

在Unity中，我们还可以在Unity Shader中使用一种特殊的Pass来完成获取屏幕图像的目的。这就是GrabPass。
当我们在Shader中定义了一个GrabPass之后，Unity会把当前屏幕的图像绘制在一张纹理中，以便我们在后续的
Pass中访问他，我们通常会使用GrabPass来实现诸如玻璃等透明材质的模拟。与使用简单的混合透明不同，使用
GrabPass可以让我们对该物体后边的图像进行更加复杂的处理，例如实现模拟折射效果。而不再是简单的和原屏
幕的颜色进行混合。

需要注意的是，在使用GrabPass的时候，我们需要额外小心物体的渲染队列的设置。正如之前所说，GrabPass
通常用于渲染透明物体，尽管代码里并不包含混合指令，但是我们往往仍然需要把物体的渲染队列设置为透明
队列才行。这样才能保证渲染不出问题。

在本节中，我们将会使用GrabPass来模拟一个玻璃效果，我们首先使用一张法线纹理来修改模型的法线信息
，然后使用了之前的反射方法，通过一个CubeMap来模拟玻璃的反射，而在模拟折射的时候，则使用了GrabPass
获取玻璃后边的屏幕图像，并使用切线空间下的法线对屏幕纹理坐标偏移后，再对屏幕图像进行模拟来近似构成
折射效果。

我们接下来要用凹凸纹理加上折射效果来渲染出一个玻璃的外框内部有一个实体球的效果图。
既然要渲染出一个凹凸纹理出来，那么我们肯定要选择切线空间转世界空间的方法。
我们选择用矩阵转换的方法来处理这个问题，那么我们在顶点着色器中，进行一系列的换算运算，得到实践代码：

```Shader
v2f vert(a2v v){
	v2f o;

	o.pos=UnityObjectToClipPos(v.vertex);
	//得到对应被抓取的屏幕图像的采样坐标、
	o.scrPos=ComputeGrabScreenPos(o.pos);

	o.uv.xy=TRANSFORM_TEX(v.texcoord,_MainTex);
	o.uv.zw=TRANSFORM_TEX(v.texcoord,_BumpMap);
	float3 worldPos=mul(unity_ObjectToWorld,v.vertex).xyz;
	fixed3 worldNormal=UnityObjectToWorldNormal(v.normal);
	fixed3 worldTangent=UnityObjectToWorldDir(v.tangent.xyz);
	fixed3 worldBinormal=cross(worldNormal,worldTangent)*v.tangent.w;
	o.TtoW0 = float4(worldTangent.x,worldBinormal.x,worldNormal.x,worldPos.x);
	o.TtoW1 = float4(worldTangent.y,worldBinormal.y,worldNormal.y,worldPos.y);
	o.TtoW2 = float4(worldTangent.z,worldBinormal.z,worldNormal.z,worldPos.z);
	
	return o;
}
```

注意到这里边有一个执行语句：o.scrPos=ComputeGrabScreenPos(o.pos);我们通过调用内置的这个函数（ComputeGrabScreenPos）函数来得到对应被抓取的屏幕图像的采样坐标。

之后的任务就变得更加明确了，我们首先通过TtoW0等变量的w分量得到世界空间下的点的坐标，并且用该值得到片元着色器
所对应的视角方向。随后，我们对法线纹理进行采样。得到切线空间下的法线方向。然后接下来我们利用这个值和_Distortion
树型以及_RefractionTex_TexelSize来对屏幕图像的采样坐标进行偏移，模拟折射的效果。_Distortion值越大，偏移量
就越大，背后物体看起来的形变成就越大。（但是实践中感觉这个值没有什么影响效果的成分）。再接下来，我们对srcPos
透视除法得到真正的屏幕坐标，再使用该坐标对抓取的屏幕图像_RefractionTex进行采样，得到模拟的折射颜色。

之后，我们把法线方向从切线空间变换到了世界空间下，并据此得到视角方向相对于法线方向的反射方向，随后使用反射方向
对CubeMap进行采样，并且把结果和主纹理颜色相乘后得到反射颜色。最后，我们使用_RefractAmount树型对反射和折射颜色
进行一个混合，作为最终的输出颜色。

整体的实践代码和效果如下：

```Shader
Shader"Unity Shaders Book/Chapter 10/玻璃效果"
{
	Properties{
		//玻璃的材质纹理，默认为白色纹理
		_MainTex("Main Tex",2D)="white"{}
		//玻璃的法线纹理
		_BumpMap("Normal Map",2D)="bump"{}
		//用于模拟反射的环境纹理
		_Cubemap("Envivironment Cubemap",Cube)="_Skybox"{}
		//折射时图像的扭曲程度
		_Distortion("Distortion",Range(0,100))=10
		//折射效果
		_RefractAmount("Refract Amount",Range(0.0,1.0))=1.0
	}
	SubShader{
		//透明物体的渲染队列,保证在进行当前渲染队列之前，不透明的物体都已经
		//渲染完成了，所以我们才能透过玻璃看到内部物体。
		Tags{"Queue"="Transparent" "RenderTpye"="Opaque"}
		//我们通过关键词GrabPass定义了一个抓取屏幕图像的Pass。在这个Pass中
		//我们定义了一个字符串，该字符串内部的名称决定了抓取得到的屏幕图像
		//会存入到哪个纹理中。实际上，我们可以省了声明该字符串，但是性能会差
		GrabPass{"_RefractionTex"}

		Pass{
			
			CGPROGRAM

			#pragma vertex vert
			#pragma fragment frag

			#include"UnityCG.cginc"

			sampler2D _MainTex;
			float4 _MainTex_ST;
			sampler2D _BumpMap;
			float4 _BumpMap_ST;
			samplerCUBE _Cubemap;
			float _Distortion;
			fixed _RefractAmount;
			sampler2D _RefractionTex;
			// _RefractionTex_TexelSize 可以让我们得到该纹理的素纹大小。
			float4 _RefractionTex_TexelSize;

			struct a2v{
				float4 vertex :POSITION;
				float3 normal:NORMAL;
				float4 tangent:TANGENT;
				float2 texcoord:TEXCOORD0;
			};

			struct v2f{
				float4 pos:SV_POSITION;
				float4 scrPos:TEXCOORD0;
				float4 uv:TEXCOORD1;
				float4 TtoW0:TEXCOORD2;
				float4 TtoW1:TEXCOORD3;
				float4 TtoW2:TEXCOORD4;
			};

			v2f vert(a2v v){
				v2f o;

				o.pos=UnityObjectToClipPos(v.vertex);
				//得到对应被抓取的屏幕图像的采样坐标、
				o.scrPos=ComputeGrabScreenPos(o.pos);

				o.uv.xy=TRANSFORM_TEX(v.texcoord,_MainTex);
				o.uv.zw=TRANSFORM_TEX(v.texcoord,_BumpMap);
				float3 worldPos=mul(unity_ObjectToWorld,v.vertex).xyz;
				fixed3 worldNormal=UnityObjectToWorldNormal(v.normal);
				fixed3 worldTangent=UnityObjectToWorldDir(v.tangent.xyz);
				fixed3 worldBinormal=cross(worldNormal,worldTangent)*v.tangent.w;
				o.TtoW0 = float4(worldTangent.x,worldBinormal.x,worldNormal.x,worldPos.x);
				o.TtoW1 = float4(worldTangent.y,worldBinormal.y,worldNormal.y,worldPos.y);
				o.TtoW2 = float4(worldTangent.z,worldBinormal.z,worldNormal.z,worldPos.z);
	
				return o;
			}

			fixed4 frag(v2f i):SV_Target{
				float3 worldPos=float3(i.TtoW0.w,i.TtoW1.w,i.TtoW2.w);
				fixed3 worldViewDir=normalize(UnityWorldSpaceViewDir(worldPos));

				//对Normal Map进行采样，得到切线空间下的法线方向。
				fixed3 bump=UnpackNormal(tex2D(_BumpMap,i.uv.zw));
				//对其进行偏移，模拟折射效果，_Distortion的值越大，偏移量越大。
				//玻璃背后的物体看起来形变程度就越大。
				float2 offset=bump.xy*_Distortion*_RefractionTex_TexelSize.xy;
				//得到折射的颜色
				fixed3 refrCol=tex2D(_RefractionTex,i.scrPos.xy/i.scrPos.w).rgb;

				
				bump=normalize(half3(dot(i.TtoW0.xyz,bump),dot(i.TtoW1.xyz,bump),dot(i.TtoW2,bump)));

				fixed3 reflDir=reflect(-worldViewDir,bump);
				fixed4 texColor=tex2D(_MainTex,i.uv.xy);
				fixed3 reflCol=texCUBE(_Cubemap,reflDir).rgb*texColor.rgb;

				fixed3 finalColor=reflCol*(1-_RefractAmount)+refrCol*_RefractAmount;
				return fixed4(finalColor,1);

			}

			ENDCG
		}
	}
}
```

实践效果：

![](https://i.loli.net/2018/07/06/5b3f2ca4befeb.png)

五、程序纹理

程序纹理指的是那些由计算机生成的图像，我们通常使用一些特定的算法来创建个性化图案或者非常真实的
自然元素、例如石头、目头等，使用程序纹理的好处在于我们可以使用各种参数来控制纹理的外观，而这些
属性不仅仅是哪些颜色属性，我们首先尝试用算法来实现一个非常简单的程序材质，随后，我们会学习一类
专门使用程序纹理的材质---程序材质。

为了能够在面板上修改数据的同时仍可执行set函数，我们使用了一个开源插件

下载地址：（https://github.com/LMNRY/SetProperty）

使用方法：下载之后直接拖拽到U3d中即可，不需要进行任何操作。

然后我们利用cs脚本画出九个圆出来：

```Shader
using UnityEngine;
using System.Collections;
using System.Collections.Generic;
//为了能让该脚本能够在编辑器模式下运行
[ExecuteInEditMode]
public class Shader_Shape : MonoBehaviour
{
    public Material material = null;
    //#region 没有实际的意义，就是为了组织代码用的。
    #region Material properties

    //为了能够在面板上修改数据的同时仍可执行set函数，我们使用了一个开源插件
    //下载地址：（https://github.com/LMNRY/SetProperty）
    //使用方法：下载之后直接拖拽到U3d中即可，不需要进行任何操作。

    //纹理大小
    [SerializeField, SetProperty("textureWidth")]

    private int m_textureWidth = 512;
    public int textureWidth
    {
        get
        {
            return m_textureWidth;
        }
        set
        {
            m_textureWidth = value;
            _UpdateMaterial();
        }
    }
    //背景颜色
    [SerializeField, SetProperty("backgroundColor")]
    private Color m_backgroundColor = Color.white;
    public Color backgroundColor
    {
        get
        {
            return m_backgroundColor;
        }
        set
        {
            m_backgroundColor = value;
            _UpdateMaterial();
        }
    }
    //圆点的颜色
    [SerializeField, SetProperty("circleColor")]
    private Color m_circleColor = Color.yellow;
    public Color circleColor
    {
        get
        {
            return m_circleColor;
        }
        set
        {
            m_circleColor = value;
            _UpdateMaterial();
        }
    }
    //模糊因子
    [SerializeField, SetProperty("blurFactor")]
    private float m_blurFactor = 2.0f;
    public float blurFactor
    {
        get
        {
            return m_blurFactor;
        }
        set
        {
            m_blurFactor = value;
            _UpdateMaterial();
        }
    }
    #endregion
    //为了保存生成的程序纹理，我们声明一个Texture2D类型的纹理变量
    private Texture2D m_generatedTexture = null;

    private void Start()
    {
        //我们首先检查了material变量是否为空，如果为空，就尝试从使用该脚本
        //所在物体上得到相应的材质。完成后，调用_UpdateMaterial函数来为其生成
        //程序纹理
        if (material == null)
        {
            Renderer renderer = gameObject.GetComponent<Renderer>();
            if (renderer == null)
            {
                Debug.LogWarning("Can not find a renderer.");
                return;
            }
            material = renderer.sharedMaterial;
        }
        _UpdateMaterial();
    }
    private void _UpdateMaterial()
    {
        //当前材质不为空的时候
        if (material != null)
        {
            //我们调用函数来生成一张程序纹理，并且赋值给m_generatedTexture;
            m_generatedTexture = _GenerateProceduralTexture();
            //利用函数将纹理贴给材质。
            material.SetTexture("_MainTex", m_generatedTexture);
        }
    }
    private Color _MixColor(Color color0, Color color1, float mixFactor)
    {
        Color mixColor = Color.white;
        mixColor.r = Mathf.Lerp(color0.r, color1.r, mixFactor);
        mixColor.g = Mathf.Lerp(color0.g, color1.g, mixFactor);
        mixColor.b = Mathf.Lerp(color0.b, color1.b, mixFactor);
        mixColor.a = Mathf.Lerp(color0.a, color1.a, mixFactor);
        return mixColor;
    }
    private Texture2D _GenerateProceduralTexture()
    {
        //首先初始化一张二维纹理
        Texture2D proceduralTexture=new Texture2D(textureWidth, textureWidth);
        float circleInterval = textureWidth / 4.0f;
        float radius = textureWidth / 10.0f;
        float edgeBlur = 1.0f / blurFactor;

        for(int w=0;w<textureWidth;w++)
        {
            for(int h=0;h<textureWidth;h++)
            {
                Color pixel = backgroundColor;
                for(int i=0;i<3;i++)
                {
                    for(int j=0;j<3;j++)
                    {
                        Vector2 circleCenter = new Vector2(circleInterval * (i + 1), circleInterval * (j + 1));
                        float dist = Vector2.Distance(new Vector2(w, h), circleCenter) - radius;

                        Color color = _MixColor(circleColor, new Color(pixel.r, pixel.g, pixel.b, 0.0f), Mathf.SmoothStep(0f, 1.0f, dist * edgeBlur));

                        pixel = _MixColor(pixel, color, color.a);
                    }
                }
                proceduralTexture.SetPixel(w, h, pixel);
            }
        }
        proceduralTexture.Apply();
        return proceduralTexture;
    }

}
```

实践效果：

![](https://i.loli.net/2018/07/06/5b3f4efe3d4d6.png)

至此，本章的学习也算到此结束了，如果需要对这部分的效果内容还需要进行拓展的部分，回头再来看看就行了。
下一次学习，我们就可以让画面动起来了。

<table><tr><td bgcolor=orange>让画面动起来</td></tr></table>

动画效果往往都是把时间添加到一些变量的计算中，以便在时间变化时画面也可以随之变化。

Unity内置的时间变量有：

![](https://i.loli.net/2018/07/10/5b4435c450707.png)

一、序列帧动画

要想实现序列帧动画，我们先要提供一张包含了关键帧图像的图像。我们利用书中资源进行模拟。
播放帧序列动画的精髓在于，我们需要在每个时刻计算该时刻下应该播放的关键帧的位子，并对
该关键帧进行纹理采样。接下来我们对Shader代码进行实践：

```Shader
Shader"Unity Shaders Book/Chapter 11/帧动画播放效果"
{
	Properties{
		_Color("Color Tint",Color)=(1,1,1,1)
		//包含了关键帧图像的纹理
		_MainTex("Image Sequence",2D)="white"{}
		//该图像在水平方向关键帧的个数
		_HorizontalAmount("Horizontal Amount",Float)=8
		//该图像在竖直方向关键帧的个数
		_VerticalAmount("Vertical Amount",Float)=8
		//控制序列帧动画的播放速度
		_Speed("Speed",Range(1,100))=30
	}
	SubShader{
		//因为帧图像通常是透明纹理，所以我们需要设置Pass的相关状态，以来渲染透明的效果：
		Tags{"Queue"="Transparent" "IgnoreProjector"="True" "RenderType"="Transparent"}
		Pass{
			//我们将帧图像处理为半透明的效果，所以要关闭深度写入和开启混合。
			Tags{"LightMode"="ForwardBase"}

			ZWrite Off

			Blend SrcAlpha OneMinusSrcAlpha

			CGPROGRAM

			#pragma vertex vert 
			#pragma fragment frag 

			#include"UnityCG.cginc"

			fixed4 _Color;
			sampler2D _MainTex;
			float4 _MainTex_ST;
			float _HorizontalAmount;
			float _VerticalAmount;
			float _Speed;

			struct a2v{
				float4 vertex :POSITION;
				float2 texcoord:TEXCOORD0;
			};

			struct v2f{
				float4 pos:SV_POSITION;
				float2 uv:TEXCOORD0;
			};

			v2f vert(a2v v){
				v2f o;
				o.pos=UnityObjectToClipPos(v.vertex);
				o.uv=TRANSFORM_TEX(v.texcoord,_MainTex);
				return o;
			}
			fixed4 frag(v2f i):SV_Target{
				float time=floor(_Time.y*_Speed);
				float row=floor(time/_HorizontalAmount);
				float column=time-row*_VerticalAmount;

				half2 uv=i.uv+half2(column,-row);

				uv.x/=_HorizontalAmount;
				uv.y/=_VerticalAmount;

				fixed4 c=tex2D(_MainTex,uv);
				c.rgb*_Color;
				return c;

			}


			ENDCG


		}
	}
}
```

实践效果：

![](https://i.loli.net/2018/07/10/5b443ddbd4e0f.png)

当然他肯定是动态的。并且根据实践行动得知，一开始我们得到的uv贴图的坐标，是左下角的那个位子。

二、滚动的背景

我们继续为了熟悉运动场景的构成，我们接下来尝试让背景滚动起来。

实践代码：

```Shader
Shader"Unity Shaders Book/Chapter 11/背景滚动效果"
{
	Properties{
		//较远纹理贴图
		_MainTex("Base Layer(RGB)",2D)="white"{}
		//较近纹理贴图
		_DetailTex("2nd Layer(RGB)",2D)="white"{}
		//滚动速度
		_ScrollX("Base layer Scroll Speed",Float)=1.0
		_Scroll2X("2nd layer Scrool Speed",Float)=1.0
		//整体亮度 
		_Multiplier("Layer Multiplier",Float)=1
	}
	SubShader{
		Tags{"RenderType"="Opaque" "Queue"="Geometry"}
		Pass{
			Tags{"LightMode"="ForwardBase"}
			CGPROGRAM
	
			#pragma vertex vert
			#pragma fragment frag 

			#include"UnityCG.cginc"

			sampler2D _MainTex;
			sampler2D _DetailTex;
			float4 _MainTex_ST;
			float4 _DetailTex_ST;
			float _ScrollX;
			float _Scroll2X;
			float _Multiplier;

			struct a2v{
				float4 vertex:POSITION;
				float4 texcoord :TEXCOORD0;
			};

			struct v2f{
				float4 pos:SV_POSITION;
				float4 uv:TEXCOORD0;
			};

			v2f vert(a2v v){
				v2f o;
				o.pos=UnityObjectToClipPos(v.vertex);
				o.uv.xy=TRANSFORM_TEX(v.texcoord,_MainTex)+frac(float2(_ScrollX,0.0)*_Time.y);
				o.uv.zw=TRANSFORM_TEX(v.texcoord,_DetailTex)+frac(float2(_Scroll2X,0.0)*_Time.y);
				return o;
			}

			fixed4 frag(v2f i):SV_Target{
				fixed4 firstLayer=tex2D(_MainTex,i.uv.xy);
				fixed4 secondLayer=tex2D(_DetailTex,i.uv.zw);
				fixed4 c=lerp(firstLayer,secondLayer,secondLayer.a);
				c.rgb*_Multiplier;
				return c;
			}
			ENDCG
		}
	}
	Fallback"VertexLit"
}
```

实际效果：

![](https://i.loli.net/2018/07/10/5b4445700b98f.png)

三、顶点动画

在游戏中，我们常常使用顶点动画来模拟飘动的旗帜、小溪等效果。

我们首先实现以下2D河流流动的效果：

```Shader
Shader "Unity Shaders Book/Chapter 11/Water" {
	Properties {
		_MainTex ("Main Tex", 2D) = "white" {}
		_Color ("Color Tint", Color) = (1, 1, 1, 1)
		_Magnitude ("Distortion Magnitude", Float) = 1
 		_Frequency ("Distortion Frequency", Float) = 1
 		_InvWaveLength ("Distortion Inverse Wave Length", Float) = 10
 		_Speed ("Speed", Float) = 0.5
	}
	SubShader {
		// Need to disable batching because of the vertex animation
		Tags {"Queue"="Transparent" "IgnoreProjector"="True" "RenderType"="Transparent" "DisableBatching"="True"}
		
		Pass {
			Tags { "LightMode"="ForwardBase" }
			
			ZWrite Off
			Blend SrcAlpha OneMinusSrcAlpha
			Cull Off
			
			CGPROGRAM  
			#pragma vertex vert 
			#pragma fragment frag
			
			#include "UnityCG.cginc" 
			
			sampler2D _MainTex;
			float4 _MainTex_ST;
			fixed4 _Color;
			float _Magnitude;
			float _Frequency;
			float _InvWaveLength;
			float _Speed;
			
			struct a2v {
				float4 vertex : POSITION;
				float4 texcoord : TEXCOORD0;
			};
			
			struct v2f {
				float4 pos : SV_POSITION;
				float2 uv : TEXCOORD0;
			};
			
			v2f vert(a2v v) {
				v2f o;
				
				float4 offset;
				offset.yzw = float3(0.0, 0.0, 0.0);
				offset.x = sin(_Frequency * _Time.y + v.vertex.x * _InvWaveLength + v.vertex.y * _InvWaveLength + v.vertex.z * _InvWaveLength) * _Magnitude;
				o.pos = mul(UNITY_MATRIX_MVP, v.vertex + offset);
				
				o.uv = TRANSFORM_TEX(v.texcoord, _MainTex);
				o.uv +=  float2(0.0, _Time.y * _Speed);
				
				return o;
			}
			
			fixed4 frag(v2f i) : SV_Target {
				fixed4 c = tex2D(_MainTex, i.uv);
				c.rgb *= _Color.rgb;
				
				return c;
			} 
			
			ENDCG
		}
	}
	FallBack "Transparent/VertexLit"
}
```

![](https://i.loli.net/2018/07/10/5b444d5b5f514.png)




























