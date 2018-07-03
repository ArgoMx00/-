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
fixe3 worldLightDir=normalize(UnityWoldSpaceLightDir(worldPos));
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


