<table><tr><td bgcolor=orange>基础纹理</td></tr></table>

纹理最初的目的就是使用一张图片来控制模型的外观。使用纹理映射技术，我们可以把一张图“黏”在模型表面
逐纹素（texel）地控制模型的颜色。

在美术人员建模的时候，通常会在建模软件中利用纹理展开技术把纹理映射坐标存储在每个顶点上。纹理坐标
定义了该顶点在纹理中对应的2D坐标。通常，这些坐标使用一个二维变量（u，v）来表示，其中u是横向坐标，
v是纵向坐标。因此，纹理映射坐标也被称为uv坐标。

uv坐标的范围通常都被归一化到【0,1】范围内。需要注意的是，纹理采样使用的纹理坐标不一定是在【0，1】
范围内。

接下来我们尝试贴图的同时再处理高光反射：

```Shader
// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader"Unity Shaders Book/Chapter 7/单张纹理"
{
	Properties{
		_Color("Color Tint", Color)=(1,1,1,1)
		_MainTex("Main Tex",2D)="white"{}
		_Specular("Specular",Color)=(1,1,1,1)
		_Gloss("Gloss",Range(8,256))=20
	}
	SubShader{
		Pass{
			Tags{"LightMode"="ForwardBase"}
			CGPROGRAM

			#pragma vertex vert
			#pragma fragment frag

			#include"Lighting.cginc"

			//我们需要在CG代码片中生命和上述树型类型相匹配的变量
			//以便和材质面板中的树型建立联系
			//其中，_MainTex_ST的名字不是任意起的，在Unity中我们需要使用
			//纹理名_ST的方式来生命某个纹理的属性
			fixed4 _Color;
			sampler2D _MainTex;
			float4 _MainTex_ST;
			fixed4 _Specular;
			float _Gloss;

			struct a2v{
				float4 vertex : POSITION;
				float3 normal :NORMAL;
				//Unity会将模型的第一组纹理坐标存储到该变量中
				float4 texcoord : TEXCOORD0;
			};

			struct v2f{
				float4 pos: SV_POSITION;
				float3 worldNormal : TEXCOORD0;
				float3 worldPos : TEXCOORD1;
				// 用于存储纹理坐标的变量uv，以便在片元着色器中使用该坐标进行纹理采样
				float2 uv : TEXCOORD2;
			};

			v2f vert(a2v v){
				v2f o;
				//将模型空间中的点的信息，改变至裁剪空间中。
				o.pos=UnityObjectToClipPos(v.vertex);
				//将模型空间中的法线信息，改变至世界空间中。
				o.worldNormal=UnityObjectToWorldNormal(v.normal);
				//将模型空间中的点的信息，改变至世界空间中。
				o.worldPos=mul(unity_ObjectToWorld,v.vertex).xyz;
				//首先使用缩放属性进行缩放，然后将其整体偏移一下，得到uv坐标
				o.uv=v.texcoord.xy * _MainTex_ST.xy + _MainTex_ST.zw;
				//o.uv=TRANSFORM_TEX(v.texcoord,_MainTex);
				return o;
			}
			fixed4 frag(v2f i): SV_Target{
				//计算单位世界空间中的法线
				fixed3 worldNormal=normalize(i.worldNormal);
				//计算单位时间空间中的光线
				fixed3 worldLightDir=normalize(UnityWorldSpaceLightDir(i.worldPos));
				//tex2D函数对纹理进行采样，它的第一个参数是需要被采样的纹理，第二个参数是一个float2类型的纹理坐标
				//它将返回值计算得到的纹素值。

				//我们采样结果和颜色属性_Color的乘积来作为材质的反射率
				fixed3 albedo=tex2D(_MainTex,i.uv).rgb*_Color.rgb;
				//并且把他和环境光照相乘得到环境光部分。
				fixed3 ambient=UNITY_LIGHTMODEL_AMBIENT.xyz*albedo;

				//接下来我们计算漫反射的值。
				fixed3 diffuse=_LightColor0.rgb*albedo*max(0,dot(worldNormal,worldLightDir));
				
				fixed3 viewDir=normalize(UnityWorldSpaceViewDir(i.worldPos));
				fixed3 halfDir=normalize(worldLightDir+viewDir);
				//计算高光反射的值
				fixed3 specular=_LightColor0.rgb*_Specular.rgb*pow(max(0,dot(worldNormal,halfDir)),_Gloss);
				//然后混合所有光的值即可。
				return fixed4(ambient + diffuse + specular ,1.0);
			}

			ENDCG

		}
	}
}


```

实际运行效果：

![](https://i.loli.net/2018/06/30/5b3764021914f.png)

二、纹理的属性

Unity的纹理映射描述起来其实很简单-----声明一个纹理变量，再使用tex2D函数采样即可。

当我们上传一张图片纹理到Unity中之后，他会产生若干的属性，其中最重要的属性是Wrap Mode
。它决定了当纹理坐标超过【0,1】之后将会如何取舍值的变化。比如Repeat，在这种模式下，
如果纹理坐标超过了1，那么它的整数部分将被舍弃，直接用小数进行采样。这样的结果是纹理将
不断重复。

如果我们想要使用2D贴图的话，我们首先必须得到它的uv坐标：

```Shader
o.uv=v.texcoord.xy * _MainTex_ST.xy + _MainTex_ST.zw;
o.uv= TRANSFORM_TEX(v.texcoord, _MainTex)
```

三、凹凸映射

有两种主要的方法来进行凹凸映射；一种方法是使用一张高度纹理来模拟表面的位移
然后得到一个修改后的法线值，这种方法也被称为高度映射；另一种方法则是使用一张
法线纹理来直接存储表面法线，这种方法又被称为法线映射。尽管我们常常将凹凸映射和
法线映射当成是相同的技术，但是我们应该知道他们的不同之处。

其实我们凹凸映射的本质就是在逐像素（或者点）的求法线的时候，方式变换了一下，从
直接求法线变成了对uv贴图等切线方式进行求值。



在切线空间中，我们实践结果先进行一次记录：

```Shader
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader"Unity Shaders Book/Chapter 7/凹凸纹理"
{
	Properties{
		_Color("Color Tint", Color)=(1,1,1,1)
		_MainTex("Main Tex",2D)="white"{}
		_BumpMap("Normal Map",2D)="bump"{}
		_BumpScale("Bump Scale",Float)=1.0
		_SpecularScale("Specular",Color)=(1,1,1,1)
		_Gloss("Gloss",Range(8.0,256))=20
	}
	SubShader{
		Pass{
			Tags{"LightMode"="ForwardBase"}
			CGPROGRAM
			
			#pragma vertex vert
			#pragma fragment frag
			
			#include"Lighting.cginc"
			fixed4 _Color;
			sampler2D _MainTex;
			float4 _MainTex_ST;
			sampler2D _BumpMap;
			float4 _BumpMap_ST;
			float _BumpScale;
			fixed4 _Specular;
			float _Gloss;

			struct a2v{
				float4 vertex :POSITION;
				float3 normal :NORMAL;
				//顶点的切线,切线是float4的
				float4 tangent : TANGENT;
				//2D贴图
				float4 texcoord :TEXCOORD0;
			};
			struct v2f{
				float4 pos:SV_POSITION;
				float4 uv:TEXCOORD0;
				float3 lightDir:TEXCOORD1;
				float3 viewDir: TEXCOORD2;
			};

			v2f vert(a2v v){
				v2f o;
				o.pos=UnityObjectToClipPos(v.vertex);
				o.uv.xy=v.texcoord.xy*_MainTex_ST.xy+_MainTex_ST.zw;
				o.uv.zw=v.texcoord.xy*_BumpMap_ST.xy+_BumpMap_ST.zw;
				//得到从模型空间到切线空间的变换矩阵rotation
				TANGENT_SPACE_ROTATION;
				
				o.lightDir=mul(rotation,ObjSpaceLightDir(v.vertex)).xyz;
				o.viewDir=mul(rotation,ObjSpaceViewDir(v.vertex)).xyz;
				return o;
			}
			fixed4 frag(v2f i) :SV_Target{
				//得到单位切线空间的视角方向和光线方向
				fixed3 tangentLightDir=normalize(i.lightDir);
				fixed3 tangentViewDir=normalize(i.viewDir);

				fixed4 packedNormal=tex2D(_BumpMap,i.uv.zw);
				
				fixed3 tangentNormal;
				tangentNormal=UnpackNormal(packedNormal);
				tangentNormal.xy*=_BumpScale;
				tangentNormal.z=sqrt(1.0-saturate(dot(tangentNormal.xy,tangentNormal.xy)));
   


				//具有贴图的高光反射、漫反射、环境光的混合。
				fixed3 albedo = tex2D(_MainTex, i.uv).rgb*_Color.rgb;
				fixed3 ambient= UNITY_LIGHTMODEL_AMBIENT.xyz*albedo;
				fixed3 diffuse=_LightColor0.rgb*albedo*max(0,dot(tangentNormal,tangentLightDir));
				fixed3 halfDir=normalize(tangentLightDir+tangentViewDir);
				fixed3 specular=_LightColor0.rgb*_Specular.rgb*pow(max(0,dot(tangentNormal,halfDir)),_Gloss);
				return fixed4(ambient + diffuse + specular ,1.0);

			}
			ENDCG
		}
	}
	fallback"Diffuse"
}
```

实际效果：

![](https://i.loli.net/2018/07/02/5b39db478b476.png)

![](https://i.loli.net/2018/07/02/5b39dbcc4e86b.png)

接下来 ，我们来实现第二种凹凸纹理设定的方式，即在世界空间下计算光照模型。我们需要在片元着色器中
把法线方向从切线空间变换到世界空间下。这种方法的基本思想是：在顶点着色器中计算从切线空间到世界
空间的变换矩阵，并且把它传递给片元着色器。变换矩阵的计算可以由顶点的切线、副切线和法线在世界空间
下的表示来得到最后，我们只需要在片元着色器中把发现纹理的法线方向从切线空间变换到世界空间下即可。
尽管这种方法需要更多的计算，但在需要使用CubeMap进行环境映射的情况下，我们就需要使用这种方法。

实践代码：

```Shader
// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'

Shader"Unity Shaders Book/Chapter 7/凹凸纹理世界空间处理"
{
	Properties{
		_Color("Color Tint", Color)=(1,1,1,1)
		_MainTex("Main Tex",2D)="white"{}
		_BumpMap("Normal Map",2D)="bump"{}
		_BumpScale("Bump Scale",Float)=1.0
		_SpecularScale("Specular",Color)=(1,1,1,1)
		_Gloss("Gloss",Range(8.0,256))=20
	}
	SubShader{
		Pass{
			Tags{"LightMode"="ForwardBase"}
			CGPROGRAM
			
			#pragma vertex vert
			#pragma fragment frag
			
			#include"Lighting.cginc"
			fixed4 _Color;
			sampler2D _MainTex;
			float4 _MainTex_ST;
			sampler2D _BumpMap;
			float4 _BumpMap_ST;
			float _BumpScale;
			fixed4 _Specular;
			float _Gloss;

			struct a2v{
				float4 vertex :POSITION;
				float3 normal :NORMAL;
				//顶点的切线,切线是float4的
				float4 tangent : TANGENT;
				//2D贴图
				float4 texcoord :TEXCOORD0;
			};
			struct v2f{
				float4 pos:SV_POSITION;
				float4 uv:TEXCOORD0;
				float4 TtoW0: TEXCOORD1;
				float4 TtoW1: TEXCOORD2;
				float4 TtoW2: TEXCOORD3;
			};

			v2f vert(a2v v){
				v2f o;
				o.pos=UnityObjectToClipPos(v.vertex);
				o.uv.xy=v.texcoord.xy*_MainTex_ST.xy+_MainTex_ST.zw;
				o.uv.zw=v.texcoord.xy*_BumpMap_ST.xy+_BumpMap_ST.zw;
				
				//得到切线空间转换到世界空间的转换矩阵：

				float3 worldPos=mul(unity_ObjectToWorld,v.vertex).xyz;
				fixed3 worldNormal=UnityObjectToWorldNormal(v.normal);
				fixed3 worldTangent=UnityObjectToWorldDir(v.tangent.xyz);
				fixed3 worldBinormal=cross(worldNormal,worldTangent)*v.tangent.w;
				o.TtoW0 = float4(worldTangent.x,worldBinormal.x,worldNormal.x,worldPos.x);
				o.TtoW1 = float4(worldTangent.y,worldBinormal.y,worldNormal.y,worldPos.y);
				o.TtoW2 = float4(worldTangent.z,worldBinormal.z,worldNormal.z,worldPos.z);
				return o;
			}
			fixed4 frag(v2f i) :SV_Target{
				
				float3 worldPos = float3(i.TtoW0.w, i.TtoW1.w, i.TtoW2.w);
				//得到这个点的观察向量和光源向量
				fixed3 lightDir=normalize(UnityWorldSpaceLightDir(worldPos));
				fixed3 viewDir=normalize(UnityWorldSpaceViewDir(worldPos));

				//求出凹凸纹理的法向量
				fixed3 bump=UnpackNormal(tex2D(_BumpMap,i.uv.zw));
				bump.xy*=_BumpScale;
				bump.z=sqrt(1.0-saturate(dot(bump.xy,bump.xy)));
				bump=normalize(half3(dot(i.TtoW0.xyz,bump),dot(i.TtoW1.xyz,bump),dot(i.TtoW2,bump)));



				//具有贴图的高光反射、漫反射、环境光的混合。
				fixed3 albedo = tex2D(_MainTex, i.uv).rgb*_Color.rgb;
				fixed3 ambient= UNITY_LIGHTMODEL_AMBIENT.xyz*albedo;
				fixed3 diffuse=_LightColor0.rgb*albedo*max(0,dot(bump,lightDir));
				fixed3 halfDir=normalize(lightDir+viewDir);
				fixed3 specular=_LightColor0.rgb*_Specular.rgb*pow(max(0,dot(bump,halfDir)),_Gloss);

				return fixed4(ambient+diffuse+specular,1.0);
			}
			ENDCG
		}
	}
	fallback"Diffuse"
}
```

四、渐变纹理

尽管在一开始，我们在渲染使用纹理是为了定义一个五题的颜色，后来人们法线，纹理其实可以用于存储任何表面
属性。一种常见的用法就是使用渐变纹理来控制漫反射光照的结果。在之前计算漫反射光照的时候，我们都是用
表面法线和光照方向的点积结果与材质的反射率相乘来得到漫反射的光照。单有的时候，我们需要更加灵活的控制
光照结果。这种技术在Valve公司提出来的，他们使用这种技术来渲染游戏中具有插画风格的角色。

实践代码：

```
// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'

Shader"Unity Shaders Book/Chapter 7/渐变纹理"
{
	Properties{
		_Color("Color Tint",Color)=(1,1,1,1)
		_RampTex("Ramp Tex", 2D)="white"{}
		_Specular("Specular",Color)=(1,1,1,1)
		_Gloss("Gloss",Range(8.0,256))=20
	}
	SubShader{
		Pass{
			Tags{"LightMode"="ForwardBase"}
			CGPROGRAM

			#pragma vertex vert
			#pragma fragment frag

			#include"Lighting.cginc"

			fixed4 _Color;
			sampler2D _RampTex;
			float4 _RampTex_ST;
			fixed4 _Specular;
			float _Gloss;

			struct a2v{
				float4 vertex:POSITION;
				float3 normal:NORMAL;
				float4 texcoord:TEXCOORD0;
			};
			struct v2f{
				float4 pos:SV_POSITION;
				float3 worldNormal:TEXCOORD0;
				float3 worldPos:TEXCOORD1;
				float2 uv:TEXCOORD2;
			};

			v2f vert(a2v v){
				v2f o;
				o.pos=UnityObjectToClipPos(v.vertex);
				o.worldNormal=UnityObjectToWorldNormal(v.normal);
				o.worldPos=mul(unity_ObjectToWorld,v.vertex).xyz;
				o.uv=TRANSFORM_TEX(v.texcoord,_RampTex);
				return o;
			
			}
			fixed4 frag(v2f i):SV_Target{
				fixed3 worldNormal=normalize(i.worldNormal);
				fixed3 worldLightDir=normalize(UnityWorldSpaceLightDir(i.worldPos));
				fixed3 ambient =UNITY_LIGHTMODEL_AMBIENT.xyz;
				//计算半兰伯特模型
				fixed halfLambert = 0.5 *dot(worldNormal,worldLightDir)+0.5;
				
				//我们使用半兰伯特模型构建uv坐标。
				fixed3 diffuseColor = tex2D(_RampTex, fixed2(halfLambert, halfLambert)).rgb * _Color.rgb;
				//计算漫反射
				fixed3 diffuse=_LightColor0.rgb*diffuseColor;
				fixed3 viewDir=normalize(UnityWorldSpaceViewDir(i.worldPos));
				fixed3 halfDir=normalize(worldLightDir+viewDir);
				//再计算一个高光反射
				fixed3 specular=_LightColor0.rgb*_Specular.rgb*pow(max(0,dot(worldNormal,halfDir)),_Gloss);
				return fixed4(ambient+diffuse+specular,1.0);
				
			}
			ENDCG
		}
	}
	Fallback"Diffuse"
}
```

实际效果：

![](https://i.loli.net/2018/07/02/5b3a047a59d3d.png)

五、遮罩纹理

遮罩纹理，是本章介绍的最后一个纹理，它非常有用，那么什么是遮罩呢？简单来讲，遮罩允许我们可以
保护一些区域，使得他们免于某些修改，例如，在之前的实现中，我们都是把高光反射应用到模型表面的
所有地方，即所有的像素都使用同样的大小的高光强度和高光指数，但是有的时候 ，我们希望模型表面
某些区域的反光强度大一些，有一些地方弱一些，为了得到更加细腻的结果，我们就可以使用一张遮罩
纹理来控制光照。表现裸露土地等纹理。

使用遮罩纹理的流程一般是：通过采样得到遮罩纹理的纹素值，然后使用其中某个通道的值，来与某种表面
属性进行相乘，这样，当该通道值为0的时候，可以保护表面不受该属性的影响。总而言之，使用遮罩纹理
可以让美术人员更加精准地控制模型表面的各种性质。

实践代码：

```Shader
Shader"Unity Shaders Book/Chapter 7/遮罩纹理"
{
	Properties{
		//光照颜色
		_Color("Color Tint",Color)=(1,1,1,1)
		//2D 贴图
		_MainTex("Main Tex",2D)="white"{}
		//Normal Map贴图
		_BumpMap("Normal Map",2D)="bump"{}
		//凹凸纹理属性值
		_BumpScale("Bump Scale",Float)=1.0
		//遮罩处理
		_SpecularMask("Specular Mask",2D)="white"{}
		//遮罩影响度的系数
		_SpecularScale("Specular Scale",Float)=1.0
		//高光反射颜色
		_Specular("Specular",Color)=(1,1,1,1)
		//高光反射系数值
		_Gloss("Gloss",Range(8.0,256))=20
	}
	SubShader{
		Pass{
			Tags{"LightMode"="ForwardBase"}
			CGPROGRAM

			#pragma vertex vert
			#pragma fragment frag

			#include"Lighting.cginc"

			//主纹理_MainTex，法线纹理_BumpMap和遮罩纹理_SpecularMask
			//在这里，三个纹理贴图的属性值都由_MainTex_ST来控制
			fixed4 _Color;
			sampler2D _MainTex;
			float4 _MainTex_ST;
			sampler2D _BumpMap;
			float _BumpScale;
			sampler2D _SpecularMask;
			float _SpecularScale;
			float4 _Specular;
			float _Gloss;

			struct a2v{ 
				float4 vertex:POSITION;
				float3 normal:NORMAL;
				float4 tangent:TANGENT;
				float4 texcoord:TEXCOORD0;
			};

			struct v2f{
				float4 pos:SV_POSITION;
				float2 uv:TEXCOORD0;
				float3 lightDir:TEXCOORD1;
				float3 viewDir:TEXCOORD2;
			};

			v2f vert(a2v v){
				v2f o;

				o.pos=UnityObjectToClipPos(v.vertex);
				o.uv=TRANSFORM_TEX(v.texcoord,_MainTex);

				//使用切线空间来求法向量的变化，造成凹凸效果。
				TANGENT_SPACE_ROTATION;

				o.lightDir=mul(rotation,ObjSpaceLightDir(v.vertex)).xyz;
				o.viewDir=mul(rotation,ObjSpaceViewDir(v.vertex)).xyz;

				return o;
			}

			fixed4 frag (v2f i) :SV_Target{
				//得到切线空间下的单位矢量
				fixed3 tangentLightDir=normalize(i.lightDir);
				fixed3 tangentViewDir=normalize(i.viewDir);
				
				//计算逐像素的法线
				fixed3 tangentNormal;
				tangentNormal=UnpackNormal(tex2D(_BumpMap,i.uv));
				tangentNormal.xy*=_BumpScale;
				tangentNormal.z=sqrt(1.0-saturate(dot(tangentNormal.xy,tangentNormal.xy)));

				//计算漫反射、高光反射、环境光的总值
				fixed3 albedo=tex2D(_MainTex,i.uv).rgb*_Color.rgb;
				fixed3 ambient=UNITY_LIGHTMODEL_AMBIENT.xyz*albedo;
				fixed3 diffuse=_LightColor0.rgb*albedo*max(0,dot(tangentNormal,tangentLightDir));
				fixed3 halfDir=normalize(tangentLightDir+tangentViewDir);
				
				//遮罩纹理是改变高光反射值来改变效果的。
				//计算Mask Value
				fixed specularMask=tex2D(_SpecularMask,i.uv).r*_SpecularScale;

				fixed3 specular = _LightColor0.rgb* _Specular.rgb*pow(max(0,dot(tangentNormal,halfDir)),_Gloss)*specularMask;

				return fixed4(ambient +diffuse +specular ,1.0);

			}



			ENDCG
		}
	}
}
```

实践效果：

![](https://i.loli.net/2018/07/02/5b3a140847575.png)

总结：

对于纹理效果的总结，学习了单张纹理的处理方式、学会了凹凸纹理的效果（虽然在直接做的时候渲染效果不明显（没有了光照
的明亮感）），但是在之后的遮罩纹理处理方法中，很好的解决了这个问题，该暗的地方就是暗的、该亮的地方就是亮的。
同时了解了什么是渐变纹理。之后有时间更加深入的回来探究一下数学部分的内容。
那么这一部分纹理的学习就到此结束了。

<table><tr><td bgcolor=orange>透明效果</td></tr></table>


一、了解什么是透明度测试和透明度混合以及透明度渲染顺序

透明是游戏中经常要使用的一种效果。在实时渲染中要实现透明效果，通常会在渲染模型时控制他的透明管道
当开启透明混合之后，当一个物体被渲染到屏幕上的时候，每个片元除了颜色值和深度值之外，他还有一个另
一哥属性---透明度。当透明度为1的时候，表示该像素是完全不透明的，而当其为0的时候，就是完全透明的。
在Unity中，我们通常使用两种方法来实现透明效果：第一种是使用透明度测试，这种方法其实无法得到真正的
半透明效果，另一种是透明度混合。

深度缓冲是用于解决可见性问题的，它可以决定哪个物体的哪些部分会被渲染在前边，而哪些部分会被其他物体
遮挡。他的基本思想是:根据深度缓存中的值来判断该片元距离摄像机的距离，当渲染一个片元的时候，需要把
它的深度值和已经存在于深度缓存中的值进行比较，如果它的值距离摄像机更远，那么说明这个片元不应该被
渲染到屏幕上；否则，这个片元应该覆盖掉此时颜色缓冲中的像素值，并且把它的深度之更新到缓冲中（默认
开启了深度写入）。

但是如果想要实现透明的效果，事情就没有那么简单了，那是因为，当使用透明度混合的时候，我们关闭了深度
写入。简单来说，透明度相关原理基本如下：

①透明度测试：它采用一种“霸道极端”的机制，不需要关闭深度写入，它产生的效果很极端，要么完全透明，看
不到，要么完全不透明。

②透明度混合：这种方法可以得到真正半透明的效果，它会使用当前片元的透明度作为混合因子，与已经存储
在颜色缓冲中的颜色值进行混合，得到新的颜色。但是，透明度混合需要关闭深度写入。这使得我们要非常
小心物体的渲染顺序。需要注意的是，透明度混合只关闭了深度写入，但是没有关闭深度测试，这意味着，当
使用透明度混合渲染一个片元的时候，还是会比较它的深度之与当前缓冲区中的深度值，如果它的深度值距离
摄像机更远，那么就不会再进行混合操作，这一点决定了，当一个不透明物体出现在一个透明物体的前边，而
我们先渲染了不透明物体，它仍然可以正常地遮挡住透明物体。也就是说，对于透明度混合来说，深度缓冲是
只读的。

那么根据我们对深度写入和相关内容的了解我们知道，如果关闭了深度写入，渲染的顺序就变得尤为重要了。


![](https://i.loli.net/2018/07/03/5b3b0849232ac.png)
![](https://i.loli.net/2018/07/03/5b3b07fc57265.png)

基于书上给出的两个样例来看，渲染引擎一般都会先对物体进行排序再渲染，常用的方法是：

首先我们渲染所有不透明的物体，并且卡其他们的深度测试和深度写入。然后再把不透明的物体按照他们距离
摄像机的远近进行排序，然后按照从后往前的顺序渲染不透明的物体，并且开启他们的深度测试，但是关闭深度
写入。

即使我们知道这种方法是足以完成渲染工作和内容的，但是这里存在一个关键的问题：如何排序？我们知道这是
游戏世界，一个物体和另外一个物体没有相对覆盖的准则，有可能只有一半覆盖上了，但是我们却做以不正确的
渲染操作了，这种问题解决方法通常是分割网络。如果我们不想分割网络，可以试着让透明通道更加柔和，使
穿插看起来不是那么明显，我们也可以使用开启了深度写入的半透明效果来近似模拟物体的半透明。
接下来，我们就学习一下Unity是如何解决排序问题的。

二、Unity Shader的渲染顺序

Unity为了解决渲染顺序的问题，提供了渲染队列（Render Queue）这一个解决方法。我们可以使用
SubShader中的Queue标签来决定我们的模型将归于哪个渲染队列。Unity在内部使用一系列整数索引
来表示每个渲染队列，且索引号越小表示越早被渲染。在Unity 5中，Unity提前定义了五个渲染队列，
当然在每个队列中间我们都可以使用其他队列，那么给出这五个提前定义的渲染队列以及描述：

![](https://i.loli.net/2018/07/03/5b3b09ac82270.png)

因此，如果我们想要通过透明度测试来实现透明效果，代码中应该包含类似下边的代码：

```Shader
SubShader{
	Tags{"Queue"="AlphaTest"}
	Pass{
		...
	}
}
SubShader{
	Tags{"Queue"="Transparent"}
	Pass{
		ZWrite Off
		...
	}
}
```

其中的ZWrite Off操作，我们即使进行猜测也能够知道，这是在关闭深度写入。这意味在当前SubShader
下的所有Pass都会关闭深度写入。

三、透明度测试

透明度测试：只要一个片元的透明度不满足条件，那么它对应的片元就会被舍弃。被舍弃的片元将不会进行
任何处理，也不会对颜色缓冲产生任何影响，否则，就会按照普通的不透明物体的处理方式来处理他。
通常，我们会在片元着色器中使用clip函数来进行透明度测试。clip是CG终点额一个函数。对于他的定义如下：

![](https://i.loli.net/2018/07/03/5b3b0b202dac8.png)

接下来，我们要对透明度测试进行一个实践,实践代码如下：

```Shader

// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'

Shader"Unity Shaders Book/Chapter 8 /透明度测试Alpha Test"
{
	Properties{
		_Color("Main Tint",Color)=(1,1,1,1)
		_MainTex("Main Tex",2D)="white"{}
		//为了在材质面板中控制透明度测试时使用的阈值，我们在属性
		//中声明定义一个范围在【0，1】之间的属性_Cutoff
		_Cutoff("Alpha Cutoff",Range(0,1))=0.5
	}
	SubShader{
		//如果进行透明度测试，将该渲染队列设置为AlphaTest
		//并且将忽略投影器的影响设置为真（1）
		//RenderType标签通常被用于着色器替换功能，标签就是指明该Shader
		//归入到提前定义的组（这里就是TransparentCutout组）中，
		//指明该Shader是一个使用了透明度测试的Shader
		Tags{"Queue"="AlphaTest" "IgnoreProjector"="True" "RenderType"="TransparentCutout"}
		Pass{
			Tags{"LightMode"="ForwardBase"}

			CGPROGRAM

			#pragma vertex vert
			#pragma fragment frag 
			#include"Lighting.cginc"

			fixed4 _Color;
			sampler2D _MainTex;
			fixed4 _MainTex_ST;
			fixed _Cutoff;


			struct a2v{
				float4 vertex:POSITION;
				float3 normal:NORMAL;
				float4 texcoord:TEXCOORD0;
			};

			struct v2f{
				float4 pos:SV_POSITION;
				float3 worldNormal:TEXCOORD0;
				float3 worldPos:TEXCOORD1;
				float2 uv:TEXCOORD2;
			};

			v2f vert(a2v v){
				v2f o;
				o.pos=UnityObjectToClipPos(v.vertex);
				o.worldNormal=UnityObjectToWorldNormal(v.normal);
				o.worldPos=mul(unity_ObjectToWorld,v.vertex).xyz;
				o.uv=TRANSFORM_TEX(v.texcoord,_MainTex);
				return o;
			}

			fixed4 frag(v2f i): SV_Target{
				
				fixed3 worldNormal=normalize(i.worldNormal);
				fixed3 worldLightDir=normalize(UnityWorldSpaceLightDir(i.worldPos));

				//2D纹理采样得到素纹值
				fixed4 texColor =tex2D(_MainTex,i.uv);
				//透明度测试Alpha Test
				//如果当前片元的素纹值小于阈值，那么当前片元就不进行染色渲染
				//相当于直接抛弃了当前片元
				clip(texColor.a-_Cutoff);
				//if(texColor.a-Cutoff<0.0){
					//discard;
				//}
				//计算2D贴图的颜色值：2D贴图素纹值*材质颜色
				fixed3 albedo=texColor.rgb*_Color.rgb;
				//计算环境光的值:环境光*贴图颜色值
				fixed3 ambient=UNITY_LIGHTMODEL_AMBIENT.xyz*albedo;
				//计算漫反射
				fixed3 diffuse=_LightColor0.rgb*albedo*max(0,dot(worldNormal,worldLightDir));
				return fixed4 (diffuse+ambient,1.0);

			}

			ENDCG
		}
	} 
	Fallback"Transparent/Cutout/VertexLit"
}
```

实践效果（随着阈值的变大，我们透明度测试的结果就变得越来越明显。）：

![](https://i.loli.net/2018/07/03/5b3b1485e49cd.png)

![](https://i.loli.net/2018/07/03/5b3b14b96e90b.png)

由测试结果可以看出，透明测试得到的透明效果很“极端”---要么完全透明，要么完全不透明。
它的效果往往像在一个不透明物体上挖了一个空洞，而且，得到的透明效果在边缘往往参差不齐
有锯齿，这是因为在边界纹理的透明度变化的精度问题，为了得到更加柔滑的透明效果，就可以
使用透明度混合。

四、透明度混合(关闭深度写入)

透明度混合的实现要比透明度测试稍微复杂一些，这是因为我们在处理透明度测试的时候，实际上
跟对待普通的不透明物体几乎是一样的，只是在片元着色器中增加了对透明度判断并且裁剪片元的
代码。而想要实现透明度混合就没有那么简单了。

首先我们回顾一下透明度混合是什么：这种方法可以得到真正的半透明效果。他会使用当前片元的
透明度作为混合因子，与已经存储在颜色缓冲区中的颜色值进行混合，得到新的颜色。但是，透明
度混合需要关闭深度写入，这使得我们要非常小心物体的渲染顺序。

为了进行混合，我们需要使用Unity提供的混合命令：Blend，Blend是Unity提供的设置混合模式的指令。
想要实现半透明的效果就需要把当前自身的颜色和已经存在于颜色缓冲中的颜色值进行混合。混合时
使用的函数就是由该指令决定的。下标表给出了Blend命令的语义：

![](https://i.loli.net/2018/07/03/5b3b16807a726.png)

Blend SrcFactor DstFactor：开启混合，并且设置混合因子。源颜色（该片元产生的颜色）会乘以SrcFacotr
而目标颜色（已经存在于颜色缓存中的颜色）会乘以DstFacotr ，然后把两种颜色混合后再加入颜色缓冲中。

在本节中的实例，我们会使用第二种语义，即Blend SrcFactor DstFactor来进行混合，需要注意的是
这个命令在设置混合因子的同时也开启了混合模式。我们会把源颜色的混合因子SrcFactor设定为SrcAlpha
而目标颜色的混合因子DstFactor定义为OneMinusScrAlpha。

那么，经过混合后的新的颜色是：DstColor（New）=SrcAlpha * SrcColor+ （1-SrcAlpha）*DstColor（Old）；

了解了大概的实现原理，接下来我们依旧进行实践：

写了一遍代码之后，值得注意的点就是要写好标签，关闭深度写入，并且开启透明度混合，设置两个参数值。

代码如下：

```
Shader"Unity Shaders Book/Chapter 8/透明度混合"
{
	Properties{
		_Color("Main Tint",Color)=(1,1,1,1)
		_MainTex("Main Tex",2D)="white"{}
		//控制整体的透明度
		_AlphaScale("Alpha Scale", Range(0,1))=1
	}
	SubShader{
		//第一个标签使用透明度混合的渲染队列是名为Transparent的队列，之前在学习过程中也
		//知道，Unity预定义了五个渲染队列
		//第二个标签是表示，我们忽略了投影器的影响，不会受到投影器的影响。
		//最后一个标签RenderType通常被用于着色器替换功能。
		Tags{"Queue"="Transparent" "IgnoreProjector"="True" "RenderType"="Transparent"}
		Pass{
			Tags{"LightMode"="ForwardBase"}
			//透明度混合我们需要关闭深度写入
			ZWrite Off
			//然后设定Blend命令，当前就是说以
			//DstColor（New）=SrcAlpha * SrcColor+ （1-SrcAlpha）*DstColor（Old）；
			//的计算方式进行颜色混合。
			Blend SrcAlpha OneMinusSrcAlpha

			CGPROGRAM

			#pragma vertex vert
			#pragma fragment frag 
			#include"Lighting.cginc"

			fixed4 _Color;
			sampler2D _MainTex;
			fixed4 _MainTex_ST;
			fixed _AlphaScale;


			struct a2v{
				float4 vertex:POSITION;
				float3 normal:NORMAL;
				float4 texcoord:TEXCOORD0;
			};

			struct v2f{
				float4 pos:SV_POSITION;
				float3 worldNormal:TEXCOORD0;
				float3 worldPos:TEXCOORD1;
				float2 uv:TEXCOORD2;
			};

			v2f vert(a2v v){
				v2f o;
				o.pos=UnityObjectToClipPos(v.vertex);
				o.worldNormal=UnityObjectToWorldNormal(v.normal);
				o.worldPos=mul(unity_ObjectToWorld,v.vertex).xyz;
				o.uv=TRANSFORM_TEX(v.texcoord,_MainTex);
				return o;
			}

			fixed4 frag(v2f i): SV_Target{
				fixed3 worldNormal=normalize(i.worldNormal);
				fixed3 worldLightDir=normalize(UnityWorldSpaceLightDir(i.worldPos));
				fixed4 texColor=tex2D(_MainTex,i.uv);
				fixed3 albedo=texColor.rgb*_Color.rgb;
				fixed3 ambient=UNITY_LIGHTMODEL_AMBIENT.xyz*albedo;
				fixed3 diffuse=_LightColor0.rgb*albedo*max(0,dot(worldNormal,worldLightDir));
				//实现半透明，使用贴图的透明度*我们实际的透明度控制数值
				return fixed4(ambient+diffuse,texColor.a*_AlphaScale);

			}

			ENDCG
		}
	}
	Fallback"Diffuse"
}

```

实现过程就是，将得到的新的tex2D纹素值，是混合之后的值。改变透明度直接在片元着色器最后的部分，返回一个乘积值
作为透明度即可。

实践效果：

![](https://i.loli.net/2018/07/03/5b3b2869685ec.png)

五、开启深度写入的半透明效果

在书上8.4节的最后，我们给出了一种由于关闭深度写入而造成的错误排序的情况，如下图：

![](https://i.loli.net/2018/07/03/5b3b29318ddda.png)

一种解决的方法就是使用两个Pass来渲染模型，第一个Pass开启深度写入,但是不进行着色渲染（就是不
输出颜色），它的目的仅仅是为了把该模型的深度值写入深度缓冲中，第二个Pass进行正常的透明度混合
由于上一个Pass已经得到了逐像素的正确深度信息，该Pass就可以按照像素级别的深度排序结果进行透明度
渲染，。但是这种方法的缺点在于，多使用一个Pass会对性能造成一定的影响，但是不影响我们改变屏幕效果

接下来我们依旧进行一波实践操作（代码如下）：

```Shader
Shader"Unity Shaders Book/Chapter 8/使用两个Pass来实现半透明效果"
{
	Properties{
		_Color("Main Tint",Color)=(1,1,1,1)
		_MainTex("Main Tex",2D)="white"{}
		//控制整体的透明度
		_AlphaScale("Alpha Scale", Range(0,1))=1
	}
	SubShader{
		//第一个标签使用透明度混合的渲染队列是名为Transparent的队列，之前在学习过程中也
		//知道，Unity预定义了五个渲染队列
		//第二个标签是表示，我们忽略了投影器的影响，不会受到投影器的影响。
		//最后一个标签RenderType通常被用于着色器替换功能。
		Tags{"Queue"="Transparent" "IgnoreProjector"="True" "RenderType"="Transparent"}
		Pass{
			//这个Pass的作用就是开启深度写入，得到每个像素真正的深度值之后
			//在下一个Pass中，修正排序不正确的问题
			ZWrite On
			//ColorMask用户设置颜色通道的写掩码（write mask）。他的语义如下：
			//Color Mask RGB|A|0|其他任何R、G、B、A的组合。
			//当Color Mask 设置为0的时候，意味着该Pass不写入任何颜色通道，即
			//不会输出任何颜色。这正是我们需要的---该Pass只需要写入深度缓存即可。
			ColorMask 0
		}
		Pass{
			Tags{"LightMode"="ForwardBase"}
			//透明度混合我们需要关闭深度写入
			ZWrite Off
			//然后设定Blend命令，当前就是说以
			//DstColor（New）=SrcAlpha * SrcColor+ （1-SrcAlpha）*DstColor（Old）；
			//的计算方式进行颜色混合。
			Blend SrcAlpha OneMinusSrcAlpha

			CGPROGRAM

			#pragma vertex vert
			#pragma fragment frag 
			#include"Lighting.cginc"

			fixed4 _Color;
			sampler2D _MainTex;
			fixed4 _MainTex_ST;
			fixed _AlphaScale;


			struct a2v{
				float4 vertex:POSITION;
				float3 normal:NORMAL;
				float4 texcoord:TEXCOORD0;
			};

			struct v2f{
				float4 pos:SV_POSITION;
				float3 worldNormal:TEXCOORD0;
				float3 worldPos:TEXCOORD1;
				float2 uv:TEXCOORD2;
			};

			v2f vert(a2v v){
				v2f o;
				o.pos=UnityObjectToClipPos(v.vertex);
				o.worldNormal=UnityObjectToWorldNormal(v.normal);
				o.worldPos=mul(unity_ObjectToWorld,v.vertex).xyz;
				o.uv=TRANSFORM_TEX(v.texcoord,_MainTex);
				return o;
			}

			fixed4 frag(v2f i): SV_Target{
				fixed3 worldNormal=normalize(i.worldNormal);
				fixed3 worldLightDir=normalize(UnityWorldSpaceLightDir(i.worldPos));
				fixed4 texColor=tex2D(_MainTex,i.uv);
				fixed3 albedo=texColor.rgb*_Color.rgb;
				fixed3 ambient=UNITY_LIGHTMODEL_AMBIENT.xyz*albedo;
				fixed3 diffuse=_LightColor0.rgb*albedo*max(0,dot(worldNormal,worldLightDir));
				//实现半透明，使用贴图的透明度*我们实际的透明度控制数值
				return fixed4(ambient+diffuse,texColor.a*_AlphaScale);

			}

			ENDCG
		}
	}
	Fallback"Diffuse"
}

```

实践效果对比：

![](https://i.loli.net/2018/07/03/5b3b2c187fe7d.png)

![](https://i.loli.net/2018/07/03/5b3b2c3925182.png)

上边那张效果图是只实现了透明度混合的效果图，下边的效果图就是用了两个Pass得到的效果图。

六、ShaderLab的混合指令

在之前的学习中，我们已经看到如何利用Blend命令进行混合颜色。实际上，混合还有其他用处，不仅仅是用于透明度的混合
在本节中。我们将更加详细地了解混合中的细节问题。

我们首先来看一下混合是如何实现的。当片元着色器产生一个颜色的时候，可以选择与颜色缓存中的颜色进行混合。
这样一来，混合就和两个操作数有关：源颜色和目标颜色。源原色用S表示，指的是片元着色器产生的颜色值。
目标颜色，我们用D来表示，指的是从颜色缓冲中读到的颜色值，当我们谈及混合中的源颜色、目标颜色和输出颜色
的时候，他们都包含了RGBA四个通道的值，而并非仅仅RGB通道。

想要使用混合，我们必须首先开启他，在Unity中，当我们使用Blend（Blend Off命令除外）命令的时候，除了设置
混合状态外也开启了混合。但是，在其他图形API中我们是需要手动开启的。

在之前的学习中我们了解过，混合是一个逐片元的操作，而且它是不可编程的，但却是高度可配置的。

现在，我们已经知道两个操作数：S,D，想要得到输出颜色O就必须要使用一个等式来计算，我们把这个等式称为混合
等式。当进行混合的时候，我们需要使用两个混合等式：一个用于混合RGB通道，一个用于混合A通道。当设置混合状态的
时候，我们实际上设置的就是混合等式中的操作和因子。在默认情况下，混合等式使用的操作都是加法操作、我们只需要
设置一下混合因子即可，，由于需要两个等式，每个等式有两个因子，因此就一共需要四个因子，下表给出了ShaderLab
中设置混合因子的命令。

![](https://i.loli.net/2018/07/03/5b3b3201c100d.png)

可以发现，第一个命令只提供了两个因子，这意味着将使用同样的混合因子来混合RGB通道和A通道，即此时SrcFactorA
将等同于SrcFactor，DstFactor将等同于DstFactor。下边就是使用这些因子所混合使用的公式：

![](https://i.loli.net/2018/07/03/5b3b3267400c7.png)

我们之前的代码填入的因子的方法也很简单：

```Shader
Blend SrcAlpha OneMinusSrcAlpha
//Blend 因子A 因子B
```

这里混合因子的参数对应的描述有一个大表来对应：

![](https://i.loli.net/2018/07/03/5b3b332d6a6c4.png)

对于Blend默认的混合操作做法是加法，那么肯定混合操作是多种多样的啊。所以我们了解一下什么是Blend混合操作：

在上边涉及的混合等式中，当把源颜色和目标颜色与他们对应的混合因子相乘后，我们都是把他们的结果加起来作为
输出颜色。那么我们要是想要不用默认的加法去混合怎么办呢？我们可以使用ShaderLab的BlendOp BlendOperation
命令，下表给出ShaderLab中支持的混合操作：

![](https://i.loli.net/2018/07/03/5b3b34b688142.png)

混合操作命令通常是与混合因子命令一起工作的。但需要注意的是，当使用Min或者Max混合操作的时候，混合
因子通常实际上是不起任何作用的，他们只会判断原始的颜色和目标颜色之间的比较结果。

通过混合操作和混合因子命令的组合，我们可以得到一些类似PhotoShop混合模式中的混合效果：

```Shader
//正常，即透明度混合
Blend SrcAlpha oneMinusSrcAlpha
//柔和相加
Blend oneMinusDstColor One
//正片垫底，即相乘
Blend DstColor Zero
//两倍相乘
Blend DstColor SrcColor
//变暗
BlendOp Min
Blend One One
//变亮
BlendOp Max
Blend One One
//滤色
Blend OneMinusDstColor One
//等同于
Blend One OneMinusSrcColor
//线性减淡
Blend One One

```

上述几种常见效果图：

![](https://i.loli.net/2018/07/03/5b3b36215e339.png)

七、双面渲染的透明效果

默认情况下渲染引擎剔除了物体背面的渲染效果，所以我们之前的透明度测试的实践都只能像是只有半个物体一样去
渲染的，只渲染了物体的正面，如果我们想要得到双面渲染的效果，那么可以使用Cull指令来控制需要剔除哪个面的
渲染图元。在Unity中，Cull指令的语法如下：

```Shader
//如果设置为Back，那么那些背对着摄像机的渲染图元就不会被渲染，这也是默认情况下的剔除状态，如果设置为
//Front，那么面朝摄像机的渲染图元就不会被渲染，如果设置为Off，就会关闭剔除功能，那么所有的渲染图元都
//会被渲染，但由于这时需要渲染的图元数目会成倍增加，因此除非是用于特殊效果，通常情况是不会关闭剔除功能的
Cull Back|Front|Off

```

那么我们接下来开始进行实践、代码基本上是重新利用了一次，首先我们实现以下透明度测试的双面渲染：

```Shader
// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'

Shader"Unity Shaders Book/Chapter 8/透明度测试双面渲染"
{
	Properties{
		_Color("Main Tint",Color)=(1,1,1,1)
		_MainTex("Main Tex",2D)="white"{}
		//为了在材质面板中控制透明度测试时使用的阈值，我们在属性
		//中声明定义一个范围在【0，1】之间的属性_Cutoff
		_Cutoff("Alpha Cutoff",Range(0,1))=0.5
	}
	SubShader{
		//如果进行透明度测试，将该渲染队列设置为AlphaTest
		//并且将忽略投影器的影响设置为真（1）
		//RenderType标签通常被用于着色器替换功能，标签就是指明该Shader
		//归入到提前定义的组（这里就是TransparentCutout组）中，
		//指明该Shader是一个使用了透明度测试的Shader
		Tags{"Queue"="AlphaTest" "IgnoreProjector"="True" "RenderType"="TransparentCutout"}
		Pass{
			Tags{"LightMode"="ForwardBase"}
			Cull Off
			CGPROGRAM

			#pragma vertex vert
			#pragma fragment frag 
			#include"Lighting.cginc"

			fixed4 _Color;
			sampler2D _MainTex;
			fixed4 _MainTex_ST;
			fixed _Cutoff;


			struct a2v{
				float4 vertex:POSITION;
				float3 normal:NORMAL;
				float4 texcoord:TEXCOORD0;
			};

			struct v2f{
				float4 pos:SV_POSITION;
				float3 worldNormal:TEXCOORD0;
				float3 worldPos:TEXCOORD1;
				float2 uv:TEXCOORD2;
			};

			v2f vert(a2v v){
				v2f o;
				o.pos=UnityObjectToClipPos(v.vertex);
				o.worldNormal=UnityObjectToWorldNormal(v.normal);
				o.worldPos=mul(unity_ObjectToWorld,v.vertex).xyz;
				o.uv=TRANSFORM_TEX(v.texcoord,_MainTex);
				return o;
			}

			fixed4 frag(v2f i): SV_Target{
				
				fixed3 worldNormal=normalize(i.worldNormal);
				fixed3 worldLightDir=normalize(UnityWorldSpaceLightDir(i.worldPos));

				//2D纹理采样得到素纹值
				fixed4 texColor =tex2D(_MainTex,i.uv);
				//透明度测试Alpha Test
				//如果当前片元的素纹值小于阈值，那么当前片元就不进行染色渲染
				//相当于直接抛弃了当前片元
				clip(texColor.a-_Cutoff);
				//if(texColor.a-Cutoff<0.0){
					//discard;
				//}
				//计算2D贴图的颜色值：2D贴图素纹值*材质颜色
				fixed3 albedo=texColor.rgb*_Color.rgb;
				//计算环境光的值:环境光*贴图颜色值
				fixed3 ambient=UNITY_LIGHTMODEL_AMBIENT.xyz*albedo;
				//计算漫反射
				fixed3 diffuse=_LightColor0.rgb*albedo*max(0,dot(worldNormal,worldLightDir));
				return fixed4 (diffuse+ambient,1.0);

			}

			ENDCG
		}
	} 
	Fallback"Transparent/Cutout/VertexLit"
}
```

效果对比(第一张图是之前学习的透明度测试，第二张图是双面渲染的透明度测试结果)：

（虽然对面的效果图不是特别好，需要重新计算光线法线之类的映射效果，但是我们这里为了表现出来这是双面渲染，这里就先不进行拓展
学习尝试了）
![](https://i.loli.net/2018/07/03/5b3b387d494da.png)

![](https://i.loli.net/2018/07/03/5b3b386d237fa.png)

明显更加真实了。

最后我们再实现以下透明度混合的双面渲染：

还是那个要强调的问题，由于我们之后要关闭深度写入，所以排序的顺序就很重要，我们要小心的控制渲染
顺序来得到正确的深度关系。如果我们仍然采用之前的做法，直接Cull Off就关闭了剔除功能的话，那么我
门就无法保证同一个物体的正面和背面图元的渲染顺序，就有可能得到错误的半透明效果。

为此 ，我们选择使用两个Pass来进行工作：第一个Pass只渲染背面，第二个Pass只渲染正面
这样由于Unity会按照Pass的先后顺序渲染，那么我们就保证了这个半透明的物体的背面永远是先被渲染并且
先一步渲染的，从而可以保证正确的先后顺序。

那么我们得到一个很简单的实践代码：

```Shader
Shader"Unity Shaders Book/Chapter 8/使用两个Pass双面渲染"
{
	Properties{
		_Color("Main Tint",Color)=(1,1,1,1)
		_MainTex("Main Tex",2D)="white"{}
		//控制整体的透明度
		_AlphaScale("Alpha Scale", Range(0,1))=1
	}
	SubShader{
		//第一个标签使用透明度混合的渲染队列是名为Transparent的队列，之前在学习过程中也
		//知道，Unity预定义了五个渲染队列
		//第二个标签是表示，我们忽略了投影器的影响，不会受到投影器的影响。
		//最后一个标签RenderType通常被用于着色器替换功能。
		Tags{"Queue"="Transparent" "IgnoreProjector"="True" "RenderType"="Transparent"}
		Pass{
			Cull Front
			//透明度混合我们需要关闭深度写入
			ZWrite Off
			//然后设定Blend命令，当前就是说以
			//DstColor（New）=SrcAlpha * SrcColor+ （1-SrcAlpha）*DstColor（Old）；
			//的计算方式进行颜色混合。
			Blend SrcAlpha OneMinusSrcAlpha

			CGPROGRAM

			#pragma vertex vert
			#pragma fragment frag 
			#include"Lighting.cginc"

			fixed4 _Color;
			sampler2D _MainTex;
			fixed4 _MainTex_ST;
			fixed _AlphaScale;


			struct a2v{
				float4 vertex:POSITION;
				float3 normal:NORMAL;
				float4 texcoord:TEXCOORD0;
			};

			struct v2f{
				float4 pos:SV_POSITION;
				float3 worldNormal:TEXCOORD0;
				float3 worldPos:TEXCOORD1;
				float2 uv:TEXCOORD2;
			};

			v2f vert(a2v v){
				v2f o;
				o.pos=UnityObjectToClipPos(v.vertex);
				o.worldNormal=UnityObjectToWorldNormal(v.normal);
				o.worldPos=mul(unity_ObjectToWorld,v.vertex).xyz;
				o.uv=TRANSFORM_TEX(v.texcoord,_MainTex);
				return o;
			}

			fixed4 frag(v2f i): SV_Target{
				fixed3 worldNormal=normalize(i.worldNormal);
				fixed3 worldLightDir=normalize(UnityWorldSpaceLightDir(i.worldPos));
				fixed4 texColor=tex2D(_MainTex,i.uv);
				fixed3 albedo=texColor.rgb*_Color.rgb;
				fixed3 ambient=UNITY_LIGHTMODEL_AMBIENT.xyz*albedo;
				fixed3 diffuse=_LightColor0.rgb*albedo*max(0,dot(worldNormal,worldLightDir));
				//实现半透明，使用贴图的透明度*我们实际的透明度控制数值
				return fixed4(ambient+diffuse,texColor.a*_AlphaScale);

			}

			ENDCG
		}
		Pass{
			Cull Back
			//透明度混合我们需要关闭深度写入
			ZWrite Off
			//然后设定Blend命令，当前就是说以
			//DstColor（New）=SrcAlpha * SrcColor+ （1-SrcAlpha）*DstColor（Old）；
			//的计算方式进行颜色混合。
			Blend SrcAlpha OneMinusSrcAlpha

			CGPROGRAM

			#pragma vertex vert
			#pragma fragment frag 
			#include"Lighting.cginc"

			fixed4 _Color;
			sampler2D _MainTex;
			fixed4 _MainTex_ST;
			fixed _AlphaScale;


			struct a2v{
				float4 vertex:POSITION;
				float3 normal:NORMAL;
				float4 texcoord:TEXCOORD0;
			};

			struct v2f{
				float4 pos:SV_POSITION;
				float3 worldNormal:TEXCOORD0;
				float3 worldPos:TEXCOORD1;
				float2 uv:TEXCOORD2;
			};

			v2f vert(a2v v){
				v2f o;
				o.pos=UnityObjectToClipPos(v.vertex);
				o.worldNormal=UnityObjectToWorldNormal(v.normal);
				o.worldPos=mul(unity_ObjectToWorld,v.vertex).xyz;
				o.uv=TRANSFORM_TEX(v.texcoord,_MainTex);
				return o;
			}

			fixed4 frag(v2f i): SV_Target{
				fixed3 worldNormal=normalize(i.worldNormal);
				fixed3 worldLightDir=normalize(UnityWorldSpaceLightDir(i.worldPos));
				fixed4 texColor=tex2D(_MainTex,i.uv);
				fixed3 albedo=texColor.rgb*_Color.rgb;
				fixed3 ambient=UNITY_LIGHTMODEL_AMBIENT.xyz*albedo;
				fixed3 diffuse=_LightColor0.rgb*albedo*max(0,dot(worldNormal,worldLightDir));
				//实现半透明，使用贴图的透明度*我们实际的透明度控制数值
				return fixed4(ambient+diffuse,texColor.a*_AlphaScale);

			}

			ENDCG
		}
	}
	Fallback"Diffuse"
}
```

实践效果图：

![](https://i.loli.net/2018/07/03/5b3b3d539a05f.png)

总结：

我们在这一部分中，学习了如何使用一些透明度的东西，以及学会了一些Blend命令、Cull剔除命令，混合命令等。

加强了基础的学习，自己也感觉算是半只脚迈进了Shader的大门。。

之后的学习会再另开一个文档继续记录。
这一部分的学习就先告一段落了。






