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




















