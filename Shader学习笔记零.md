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

















