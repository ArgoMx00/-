关于Shader

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

五、表面着色器

表面着色器被定义在SubShader语义块中的CGPROGRAM 和ENDCG之间。原因是，表面着色器不需要开发者
关心使用多少个Pass、每个Pass如何渲染等问题，Unity会在背后为我们做好这些事情，我们要做的只是
告诉他：“嘿，使用这些纹理去填充颜色，使用这个法线纹理去填充法线、使用Lambert光照模型，其他的
别来烦我！”。

CGPROGRAM和ENDCG之间的代码是用CG/HLSL编写的，也就是说，我们需要把CG/HLSL语言嵌套在ShaderLab
语言中。值得注意的是，这里的CG/HLSL是Unity经过封装后提供的，它的语法和标准的CG/HLSL语法几乎
一样，但还是有细微的不同，例如有些原生的函数和用法Unity就并没有提供支持。

六、顶点/片元着色器

在Unity中，我们可以使用CG/HLSL语言来编写顶点/片元着色器。他们更加复杂，但是灵活性也更高。
和表面着色器类似，顶点/片元的着色器也要卸载CGPROGRAM和ENDCG之间，但不同的是，顶点/片元
着色器是写在Pass语义块内，而非SubShader内的。原因是，我们需要自己另一每个Pass需要使用的
Shader代码。虽然我们可能需要编写更多的代码，但带来的好处是灵活性更高。更加重要的是，我们
可以控制渲染的实现细节，同样，这里的CGPROGRAM和ENDCG之间的代码也是使用CG/HLSL编写的。

七、选择哪种Unity Shader形式？

①除非你有非常明确的需求必须要使用固定函数着色器，否则请使用可编程管线的着色器，即表面着色器
或顶点/片元着色器。

②如果我想和各种光源打交道，那么我们可能更喜欢使用表面着色器，但是需要小心它在移动平台的性能
表现。

③如果你需要使用的光照数目非常少，例如只有一个平行光，那么使用顶点/片元着色器是一个更好的选择。

④更加重要的是，如果你有很多自定义的渲染效果，那么请选择使用顶点/片元着色器。

学习Shader的数学基础

一、向量、点、矩阵。

其中矩阵内容较为复杂：

①可逆的==非奇异的、不可逆的==奇异的 M*M(-1)=I

②M*MT=MT*M=I 是正交矩阵

二、空间坐标



初级篇

在基础篇中，我们学习了渲染流水线，并且给出了Unity Shader的基本概况，同时还打下了一定的数学基础
从本章开始，我们将真正的学习如何在Unity中编写Unity Shader

一、顶点/片元着色器的基本结构

一个简单的实例：


```c#
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 5/Simple Shader"
{
	SubShader{
		Pass{
			CGPROGRAM 
			
			//这是在告诉Unity 这个部分的代码块是一个顶点/片元着色器
			#pragma vertex vert
			#pragma fragment frag
	
			float4 vert(float4 v : POSITION) : SV_POSITION{
				return mul(UNITY_MATRIX_MVP,v);
			}
			fixed4 frag():SV_Target{
				return fixed4(1.0,1.0,1.0,1.0);
			}

			ENDCG
		}
	}
}
```

首先，代码的第一行通过Shader语义定义了这个Unity Shader的名字。保持良好的命名习惯有助于我们
在为材质球选择Shader时快速找到自定义的Unity Shader。需要注意的是，在上边的代码中我们并没有
用到Properties语义块。Properties语义块不是必须的，我们可以选择不声明任何材质属性。

然后，我们声明了SubShader和Pass语义块。在本例中，我们不需要进行任何渲染设置和标签设置，因此
SubShader将使用默认的渲染设置和标签设置。在SubShader语义块中，我们定义了一个Pass，在这个Pass
中我们同样没有进行任何自定义的渲染设置和标签设置。

接着，就是由CGPROGRAM 和ENDCG所包围的CG代码片段。这是我们的重点。首先，我们遇到了两行非常重要
的编译指令：

#pragma vertex vert（name）
#pragma fragment farg（name）

它们将告诉Unity，哪个函数包含了顶点着色器的代码，哪个函数包含了片元着色器的代码。相当于在声明
，这一部分代码块，是控制顶点/片元着色器的部分。

接下来，我们具体看一下vert函数的定义：

```Shader
float4 vert(float4 v : POSITION) : SV_POSITION{
	return mul(UNITY_MATRIX_MVP,v);
}
```

这是本例子中使用的顶点着色器代码，它是逐个顶点执行的。vert函数的输入v包含了这个顶点的位子，
这是通过POSITION语义指定的，它的返回值是一个float4类型的变量，它是该顶点在裁剪空间中的位子
，POSITION和SV_POSITION都是CG/HLSL中的语义，他们是不能够省略的，这些语义将告诉系统用户
需要哪些输入值，以及用户的输出是什么。

然后我们再来看一下frag函数：

```Shader
fixed4 frag():SV_Target{
	return fixed4(1.0,1.0,1.0,1.0);
}
```

在这个例子中，frag函数没有任何输入。他的输出是一个fixed4类型的变量，并且使用了SV_Target
语义进行限定。SV_Target也是HLSL中的一个系统语义，它等同于告诉渲染器，把用户的输出颜色保存
到一个渲染目标中，这里将输出到默认的帧缓存中。片元着色器中的代码很简单，返回了一个表示白色
的fixed4类型的变量。片元着色器输出的颜色的每个分量范围在【0,1】，其中（0,0,0）是黑色，相反
就是白色。

接下来，我们想要得到更多的模型数据的话，将内容修改变成结构体即可：

```Shader
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 5/Simple Shader"
{
	SubShader{
		Pass{
			CGPROGRAM 
			
			//这是在告诉Unity 这个部分的代码块是一个顶点/片元着色器
			#pragma vertex vert
			#pragma fragment frag
			struct a2v{
				//POSITION 语义告诉Unity，用模型空间的顶点坐标来填充vertex变量
				//NORMAL 语义告诉Unity，用模型空间的法线方向来填充normal变量
				//TEXCOORD0 语义告诉Unity，用模型的第一套纹理坐标填充texcoord变量
				float4 vertex: POSITION;
				float3 normal:NORMAL;
				float4 texcoord:TEXCOORD0;
			};
			float4 vert(a2v v) : SV_POSITION{
				return mul(UNITY_MATRIX_MVP,v.vertex);
			}
			fixed4 frag():SV_Target{
				return fixed4(0.2,0.3,0.4,0.5);
			}

			ENDCG
		}
	}
}
```

在上边的代码中，我们声明了一个新的结构体a2v，它包含了顶点着色器需要的模型数据。
在a2v的定义中，我们用到了更多Unity支持的语义，如NORMAL和TEXCOORD0,当他们作为
顶点着色器的输入的时候都是有特定的含义的。因为Unity会根据这些语义来填充这个结构体
对于顶点着色器的输出，Unity支持的语义有：POSITION,TANGENT,NORMAL,TEXCOORD0，
TEXCOORD1............COLOR等。

为了创建一个自定义的结构体，我们必须使用如下格式来定义他：

```Shader
struct StructName{
	Type Name : Semantic;
	Type Name : Semantic;
	.......
}
```









