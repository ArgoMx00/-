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





