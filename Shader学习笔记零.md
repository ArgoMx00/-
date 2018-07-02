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






