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








