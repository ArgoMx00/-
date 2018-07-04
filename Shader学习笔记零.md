1.将模型空间中的点的信息，转换到裁剪空间中：
```
o.pos：SV_POSITION
v.vertex：POSITION
o.pos=UnityObjectToClipPos(v.vertex);
```

2.将模型中的法线信息，转换到世界空间中:

```
v.normal:NORMAL
o.worldNormal:TEXCOORD0
o.worldNormal=UnityObjectToWorldNormal(v.normal);
```

3.将模型中的点的信息，转换到世界空间中：

```
v.vertex:POSITION
o.worldPos:TEXCOORD0
o.worldPos=mul(unity_ObjectTo World,v.vertex).xyz;
```

4.得到uv贴图坐标并存储：

```
v.texcoord:REXCOORD0
o.uv:TEXCOORD0
o.uv=v.texcoord.xy * _MainTex_ST.xy + _MainTex_ST.zw;
//o.uv=TRANSFORM_TEX(v.texcoord,_MainTex);
```

5.计算单位向量（矢量）

```
fixed3 worldNormal=normalize();
```

6.计算世界空间中的某点（单位）光照法线

```
worldPos:世界空间中的点
fixed3 worldLightDir=normalize(UnityWoldSpaceLightDir(worldPos));
fixed3 worldLightDir=normalize(_WorldSpaceLightPos0.xyz);
```

7.计算世界空间中的某点（单位）观察向量（矢量）

```
worldPos:世界空间中的点
fixed3 viewDir=normalize(UnityWorldSpaceViewDir(worldPos));
```

8.计算带有2D贴图的点的材质颜色：

```
_MainTex：2D贴图
uv:2D贴图的uv坐标
_Color:我们定义的材质体颜色
fixed3 albedo=tex2D(_MainTex,uv).rgb*_Color.rgb
```

9.提取环境光信息：

```
fixed3 ambient=UNITY_LIGHTMODEL_AMBIENT;
//如果带有2D贴图，我们计算的时候：
albedo:带有2D贴图的点的点的材质颜色
fixed3 ambient=UNITY_LIGHTMODEL_AMBIENT.xyz*albedo;
```

10.(凹凸纹理切线空间)得到切线空间中的光线、视角矢量

```
//得到模型空间到切线空间的变换矩阵rotation
TANGENT_SPACE_ROTATION;

o.lightDir=mul(rotation,ObjSpaceLightDir(v.vertex)).xyz;
o.viewDir=mul(rotation,ObjSpaceViewDir(v.vertex)).xyz;
```

11.（凹凸纹理切线空间）求出凹凸纹理法向量并且处理求值问题：

```Shader
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
```


12.（凹凸纹理世界空间）将切线空间转换到世界空间的转换矩阵：

```Shader
float3 worldPos=mul(unity_ObjectToWorld,v.vertex).xyz;
fixed3 worldNormal=UnityObjectToWorldNormal(v.normal);
fixed3 worldTangent=UnityObjectToWorldDir(v.tangent.xyz);
fixed3 worldBinormal=cross(worldNormal,worldTangent)*v.tangent.w;
o.TtoW0 = float4(worldTangent.x,worldBinormal.x,worldNormal.x,worldPos.x);
o.TtoW1 = float4(worldTangent.y,worldBinormal.y,worldNormal.y,worldPos.y);
o.TtoW2 = float4(worldTangent.z,worldBinormal.z,worldNormal.z,worldPos.z);
```

13.(凹凸纹理世界空间) 求出凹凸纹理的法向量并且处理求值问题：

```Shader
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
```

14._LightColor0.rgb =光照颜色。

15.遮罩纹理处理方法（改变高光反射值来处理）：

```Shader

//遮罩纹理是改变高光反射值来改变效果的。
//计算Mask Value
fixed specularMask=tex2D(_SpecularMask,i.uv).r*_SpecularScale;
//计算高光反射的值
fixed3 specular = _LightColor0.rgb* _Specular.rgb*pow(max(0,dot(tangentNormal,halfDir)),_Gloss)*specularMask;
```

16.透明度测试标签

![](https://i.loli.net/2018/07/03/5b3b09ac82270.png)

```
//如果进行透明度测试，将该渲染队列设置为AlphaTest
//并且将忽略投影器的影响设置为真（1）
//RenderType标签通常被用于着色器替换功能，标签就是指明该Shader
//归入到提前定义的组（这里就是TransparentCutout组）中，
//指明该Shader是一个使用了透明度测试的Shader
Tags{"Queue"="AlphaTest" "IgnoreProjector"="True" "RenderType"="TransparentCutout"}
```

17.Tex2D采样得到素纹值（颜色值）

```Shader
//返回值是一个rgba值
fixed4 texColor =tex2D(_MainTex,i.uv);
```

18.透明度测试：

```Shader
//透明度测试Alpha Test
//如果当前片元的素纹值小于阈值，那么当前片元就不进行染色渲染
//相当于直接抛弃了当前片元
clip(texColor.a-_Cutoff);
//if(texColor.a-Cutoff<0.0){
	//discard;
//}
```

19.透明度混合标签和必备格式：

```
Shader"Unity Shaders Book/Chapter 8/透明度混合"
{
	Properties{
		...
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
		}
	}
}
```

20.设定一个Pass只用来深度读入。并不输出颜色：

```Shader

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
```

21.一些混合操作的效果：

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

22.准确得到光照衰减的pragma编译指令

```

//这个pragma编译指令，可以保证我们在Shader中使用光照衰减等
//光照变量可以被正确赋值。这是必不可少的。
#pragma multi_compile_fwdbase
```

23.区别光源类型（并且给出了平行光和非平行光的求光线方向的方式）

```Shader
//判断光源是否定义了USING_DIRECTIONAL_LIGHT，如果定义了，那么说明该光源类型是品星光
//否则就是点光源或者是聚光灯，两种光的求光源方向的方式不同，所以要进行区别。
#ifdef USING_DIRECTIONAL_LIGHT
	fixed3 worldLightDir=normalize(_WorldSpaceLightPos0.xyz);
#else 
	fixed3 worldLightDir=normalize(_WorldSpACElIGHTpOS0.xyz-i.worldPos.xyz);
#endif

```

24.世界坐标下的摄像机位子：_WorldSpaceCameraPos.xyz

25.定义不同光源的衰减值：

```Shader
#ifdef USING_DIRECTIONAL_LIGHT
	fixed atten=1.0;
#else 
	#if defined(POINT)
		float3 lightCoord=mul(unity_WorldToLight,float4(i.worldPos,1)).xyz;
		fixed atten=tex2D(_LightTexture0,dot(lightCoord,lightCoord).rr).UNITY_ATTEN_CHANNEL;
	#elif defined(SPOT)
		float4 lightCoord=mul(unity_WorldToLight,float4(i.worldPos,1)).xyz;
		fixed atten=(lightCoord.z>0)*tex2D(_LightTexture0,lightCoord.xy/lightCoord.w+0.5).w*tex2D(_LightTextureB0,dot(_LightCoord,lightCoord).rr).UNITY_ATTEN_CHANNEL;
	#else
		fixed atten=1.0;
	#endif
#endif
return fixed4((diffuse+specular)*atten,1.0); 

```

26.定义不同光源的衰减值（简化版）：

```Shader
//判断是否是平行光来断定衰减值
#ifdef USING_DIRECTIONAL_LIGHT
	fixed atten=1.0;
#else
	float3 lightCoord=mul(_LightMatrix0,float4(i.worldPos,1)).xyz;
	fixed atten=tex2D(_LightTexture0,dot(lightCoord,lightCoord).rr).UNITY.ATTEN_CHANNEL;
#endif
```

27.将世界空间中的点转换到光源空间中，再利用Unity内部的2D贴图进行计算衰减值：

```Shader
//得到点在光源空间中的位子
float3 lightCoord=mul(unity_WorldToLight,float4(i.worldPos,1)).xyz;
//然后我们可以使用这个坐标的模的平方对衰减纹理进行采样，得到衰减值：
fixed atten=tex2D(_LightTexture0,dot(lightCoord,lightCoord).rr).UNITY_ATTEN_CHANNEL;
```

28.求阴影值三剑客：

SHADOW_COORDS(2):用于定义在顶点着色器输出结构体中
TRANSFER_SHADOW(O):写在顶点着色器中。
SHADOW_ATTENUATION（i）：用于求阴影值。

29.求阴影值和衰减值的乘积值：
```Shader
UNITY_LIGHT_ATTENUATION(atten,i,i.worldPos);
```



