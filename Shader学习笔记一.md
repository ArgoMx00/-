一、什么是Shader?
①GPU流水线上一些可高度编程的阶段，而由着色器编译出来的最终代码是会在GPU上运行的

②有一些特定类型的着色器，如顶点着色器、片元着色器等。

③依靠着色器我们可以控制流水线中的渲染细节，例如用顶点着色器来进行顶点变换以及传递
数据，用片元着色器来进行逐个像素的渲染。

二、Properties语义块支持的属性类型：


```shader
	Properties{
		_Diffuse("DiffuseColor",Color)=(1,1,1,1)
		_Int("Int",Int)=2
		_Float("Float",Float)=1.5
		_Range("Range",Range(0.0,5.0))=3.0
		_Vector("Vector",Vector)=(2,3,6,1)
		_2D("2D",2D)=""{}
		_Cube("Cube",Cube)="white"{}
		_3D("3D",3D)="black"{}
	}
```

注意，Properties中的数据变量的声明和使用都不能加以分号。

为了在Shader中可以访问到这些属性，我们需要在CG代码片中定义和这些属性类型相匹配的
变量，需要说明的是，即使我们不在Properties语义块中声明这些属性，也可以直接在CG代
码片中定义变量。此时我们可以通过脚本向Shader中传递这些属性。因此，Properties语义
块的作用仅仅是为了让这些属性可以出现在材质面板中。

三、SubShader

每一个Unity Shader文件可以包含多个SubShader语义块，但是最少要有一个。当Unity需要
加载这个Unity Shader的时候，会扫描所有的Subshader语义块，然后选择第一个能够在目标
平台上运行的Subshader。如果都不支持的话，Unity就会使用Fallback语义指定的Unity Shader
Unity提供这种语义的原因在于，不同的显卡具有不同的能力。

SubShader中定义了一系列Pass以及可选的状态和标签设置。每个Pass定义了一次完整的渲染流程
，但如果Pass的数目过多，往往会造成渲染性能的下降。因此，我们应该尽量使用最小数目的Pass

对于一个完整的Shader来讲，其结构如下：

```shader
Shader "MyShader"{
    Properties{
    	//所需要的各种属性
    }
    SubShader{
    	//真正意义上的Shader代码会出现在这里
	//表面着色器（Surface Shader）或者
	//顶点/片元着色器（Vertex/Fragment Shader）或者
	//固定函数着色器（Fixed Fuction Shader）
    }
    SubShader{
    	//和上一个SubShader 类似
    }
}
```

四、关于漫反射的一个实例：

```shader
// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unlit/Study_Shader"
{
	//注意属性不能有分号
	Properties{
		_Diffuse("DiffuseColor",Color)=(1,1,1,1)
		_Int("Int",Int)=2
		_Float("Float",Float)=1.5
		_Range("Range",Range(0.0,5.0))=3.0
		_Vector("Vector",Vector)=(2,3,6,1)
		_2D("2D",2D)=""{}
		_Cube("Cube",Cube)="white"{}
		_3D("3D",3D)="black"{}
	}
	SubShader{
		Pass{
			Tags{"LightMode" = "ForwardBase"}
			CGPROGRAM
			#pragma vertex vert
			#pragma fragment frag 

			#include "Lighting.cginc"

			fixed4 _Diffuse;//float 32 half 16 fixed 11 Color【0，1】
			//alply to vertex 应用到点
			struct a2v{
				float4 vertex : POSITION;//POSITION : 语义 GPU在哪里去存取数据
				float3 normal : NORMAL;
			};
			//vertex to fragment 从顶点应用到片段
			struct v2f{
				float4 Pos:SV_POSITION; //SV_POSITION(POSITION) SV_TARGET(COLOR)
				fixed3 color: COLOR;
			};

			v2f vert(a2v v){
				v2f o;
				o.Pos=UnityObjectToClipPos(v.vertex);//从模型空间变化到裁剪空间。
				
			    float3 nDir = normalize(mul(unity_ObjectToWorld,v.normal));////求得顶点法线 世界空间的
			
				float3 lDir=normalize(_WorldSpaceLightPos0.xyz);

				fixed3 diffuse =_LightColor0 * _Diffuse * saturate(dot(nDir,lDir));//把乘积的值框定在0到1之间。
				
				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
				
				o.color= ambient+diffuse;
				
				return o;
			}
			fixed4 frag(v2f IN): SV_TARGET{
				return fixed4(IN.color,1.0);
			}
			ENDCG
		}
	}
}
```









