<table><tr><td bgcolor=orange>关于Shader</td></tr></table>

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

<table><tr><td bgcolor=orange>学习Shader的数学基础</td></tr></table>

一、向量、点、矩阵。

其中矩阵内容较为复杂：

①可逆的==非奇异的、不可逆的==奇异的 M*M(-1)=I

②M*MT=MT*M=I 是正交矩阵

二、空间坐标



<table><tr><td bgcolor=orange>初级篇</td></tr></table>

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

这里边，语义是不可以进行省略的。
填充到POSITION,TANGENT,NORMAL这些语义中的数据究竟是从哪里来的呢？在Unity中，它们
是由使用该材质的Mesh Render组件提供的。在每帧调用Draw Call的时候，Mesh Render组件
会把他负责渲染的模型数据发送给Unity Shader。我们知道，一个模型通常包含了一组三角面片，
每个三角面片由三个顶点组成，而每个顶点又包含了一些数据，例如位子、法线、切线、纹理坐标、
顶点颜色等，通过上面的方法自定义结构体，我们就可以在顶点着色器中访问这些顶点的模型数据。

二、顶点着色器和片元着色器之间的通信

a2v---->应用于顶点着色器---->通常作为顶点着色器的读入

v2f---->应用于片元着色器---->通常作为顶点着色器的输出，片元着色器的读入

这样，用两个结构体就能够将顶点着色器和片元着色器相连通了。

那么我们看这样一个实例：

```Shader

// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 5/Simple Shader" {
	//注意属性后边不能有分号
	Properties{
		_Color("Color Tint",Color)=(1.0,1.0,1.0,1.0)
	}
	SubShader{
		Pass{
			CGPROGRAM
			
			//告诉Unity 这部分代码块中的内容是顶点/片元着色器
			#pragma vertex vert
			#pragma fragment frag

			//在CG代码中，我们必须定义一个与属性名称和类型都匹配的变量
			uniform fixed4 _Color;

			//a2v 表示的是应用到顶点着色器，一般用于顶点着色器的输入
			struct a2v{
				//POSITION 语义告诉Unity，用模型空间的顶点坐标来填充vertex变量
				//NORMAL 语义告诉Unity，用模型空间的法线方向来填充normal变量
				//TEXCOORD0 语义告诉Unity，用模型的第一套纹理坐标填充texcoord变量
				float4 vertex : POSITION;
				float3 normal : NORMAL;
				float4 texcoord :TEXCOORD0;
			};
			//v2f 表示的是应用到片元着色器，是顶点着色器的输出，片元着色器的输入
			struct v2f{
				//SV_POSITION 语义告诉Unity，pos里包含了顶点在裁剪空间中的位子信息
				//COLOR0语义可以用于存储颜色信息
			    float4 pos : SV_POSITION;
				fixed3 color :COLOR0;
			};

			v2f vert(a2v v) {
				//声明输出结构
				v2f o;
				o.pos=UnityObjectToClipPos(v.vertex);//o.pos=mul(UNITY_MATRIX_MVP,v.vertex);
				//v.normal 包含了顶点的法线方向，其分量在【-1，1】之间。
				//下边的代码把分量的范围映射到了【0，1】；
				//存储到o.color中传递给片元着色器。
				o.color= v.normal * 0.5 +fixed3(0.5,0.5,0.5);
				return o;
			}

			fixed4 frag (v2f i): SV_Target{
				fixed3 c=i.color;
				//使用Color属性来控制输出的颜色。
				c*=_Color.rgb;
				return fixed4(c,1.0);
			}

			ENDCG
		}
	}
}
```

在实践中，我们往往希望从顶点着色器输出一些数据之后，传递给片元着色器，实现两者之间的通信，
那么我们按照上述过程来控制shader的颜色，就可以实现出来。

同时我们再引入属性，利用属性来控制着色器显示的颜色。在上边的代码中，我们首先添加了Properties
语义块中，并且在其中声明了一个属性_Color，它的类型是Color，初始值是一个白色，为了在CG代码中
访问到他，我们还需要在CG代码片段中提前定义一个新的变量，注意这个变量的名称和类型必须和属性语义块
中的属性定义相匹配。

整个代码的含义其实就是在实现对应顶点法线方向所带来的值的颜色。

我们此时shader赋予给一个球体的显示效果如下：

![](https://i.loli.net/2018/06/28/5b34bfd9029ef.png)

其中一些相关内容需要了解一下：

![](https://i.loli.net/2018/06/28/5b34c03286b31.png)

有时，读者可能会发现在CG变量前会有一个uniform关键字，例如：

uniform fixed4 _Color;

uniform关键字是CG中修饰变量和参数的一种修饰词，在Unity Shader中，uniform关键词
是可以省略的。
</br></br></br></br>
三、强大的援手：Unity提供的内置文件和变量

Unity中一些常用的包含文件：

![](https://i.loli.net/2018/06/28/5b34c41164d7a.png)

可以看出，有一些文件即使我们没有使用#include指令，他们也是会被自动包含进来的，
例如UnityShaderVariables.cginc。因此，在前边的例子中，我们直接能够使用模型-
观察-投影矩阵的方式来将模型空间中的点变化为裁剪视图空间中的点。

包含文件是类似于c++中头文件的一种文件。在Unity中，他们的文件后缀是.cginc。
在编写shader的时候，我们可以用#include指令把这些文件包含进来。这样我们就可以
直接使用一些非常有帮助的函数了，例如：

```Shader
CGPROGRAM
//-------
#include"UnityCG.cginc"
//-------
ENDCG
```

四、Unity提供的CG/HLSL语义。

Unity为了方便对模型数据的传输，对一些语义进行了特别的含义规定。例如，在顶点
着色器的输入结构体a2f用TEXCOORD0来描述texcoord，unity会自动识别TEXCOORD0
语义，以把模型的第一组纹理坐标填充到texcoord中。需要注意的是，即使语义的名称
一样，如果出现的位子不同，那么含义也不同。例如，TEXCOORD0既可以用于描述顶点
着色器的输入结构体a2f，也可以用描述输出结构体v2f。但是在输入种，有特别的含义
就是把模型的第一组纹理坐标存储在当前变量中，而在输出结构体v2f中，TEXCOORD0修
饰的变量含义就可以由我们来决定。

DirectX10之后，有了一种新的语义类型，就是系统数值语义，这类语义是以SV开头的。
比如SV_POSITION语义去修饰顶点着色器的输出变量pos，那么就表示pos包含了可用于
光栅化的变换后的顶点坐标（即齐次裁剪空间中的坐标）。用这些语义描述的变量是不能
随便赋值的，因为流水线需要使用它们来完成特定的目的。例如渲染引擎会把用SV_POSITION
修饰的变量经过光栅化后显示在屏幕上。在绝大数平台上，SV_POSITION和POSITION是
等价的。因此，为了让我们的Shader有更好的跨平台性，对于这些有特殊含义的变量
我们最好用以SV开头进行修饰。

![](https://i.loli.net/2018/06/28/5b34c9036d7fe.png)

![](https://i.loli.net/2018/06/28/5b34c933726b0.png)

通常，我们如果需要把一些自定义的数据从顶点着色器传递给片元着色器，一般选用TEXCOORD0等。

![](https://i.loli.net/2018/06/28/5b34c98c804de.png)

五、写Shader的几个注意

①注意慎用分支和循环语句。

随着GPU的发展，我们现在已经可以使用if-else、for、while这些流程控制指令了。但是，他们在
GPU上的实现和CPU上实现有很大的不同。如果我们在Shader中使用了大量流程控制语句，那么这个
Shader的性能可能会成倍的下降。当然，有时我们不可避免的要使用分支语句进行运算，那么一些
建议有：

分支判断语句中使用的条件变量最好是常数，即使在Shader运行过程中不会发生变化；

每个分支中包含的操作指令数尽可能的要少。

分支嵌套层数尽可能的要少

②不要除以0

③避免不必要的计算。

</br></br></br></br>
<table><tr><td bgcolor=orange>基础光照</td></tr></table>

计算机图形学的第一定律：如果它看起来是对的，那么他就是对的。

一、计算机图形学中的几种常见光：


自发光（Cmissive）：光线也可以直接由光照源头直接发射进入摄像机，而不需要任何物体的反射、
标准光照模型使用自发光来计算这个部分的贡献度。他的计算方式和很简单，就是直接使用了该材质的
自发光颜色：Cmissive=Mmissive

高光反射（Cspecular）：

![](https://i.loli.net/2018/06/30/5b3748fe4acda.png)
![](https://i.loli.net/2018/06/30/5b3744385702c.png)

漫反射（Cdiffuse）：漫反射光照是对于那些被物体表面随机散射到各个方向的辐射度进行建模的。
在漫反射中，视角的位子是不重要的，因为反射是完全随机的。因此可以认为在任何反射方向的分步
都是一样的。但是，入射光纤角度很重要。漫反射光照符合兰伯特定律：反射光线的强度与表面法线
和光源方向之间夹角的余弦值成正比。因此，漫反射的计算方式如下：

Cdiffuse=（Clight*Mdiffuse）*max（0，n*I）

其中，n是表面法线，I是指向光源的单位矢量，Mdiffuse是材质的漫反射颜色，Clight是光源颜色。
需要注意的是，我们需要防止法线和光源方向点乘的结果为负值，位次，我们用取最大值为0的函数来将其
截取到最小为0.这可以用来防止物体被从后边来的光源照亮。


环境光（Cambient）:虽然标准光照模型的重点在于描述直接光照，但是在真实的世界中，物体
也可以被间接光照所照亮。在标准光照模型中，我们使用了一种被称为环境光的部分来近似模拟
间接光照。环境光的计算非常简单，它通常是一个全局变量，即场景中的所有物体都使用这个环
境光。Cambient=gambient

二、Unity中的环境光和自发光

在Unity场景中，环境光可以在Window中setting中进行控制。在Shader中，我们只需要通过Unity
内置变量UNITY_LIGHTMODEL_AMBIENT就可以得到环境光的颜色和强度信息。

三、在Unity Shader中实现漫反射光照模型。

漫反射的计算公式：Cdiffuse=（Clight*Mdiffuse）*max（0，n*I）

接下来我们进行实践,学习逐个点进行漫反射的效果：

```Shader
// Upgrade NOTE: replaced '_World2Object' with 'unity_WorldToObject'
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 6/Diffuse Vertex-Level"
{
	Properties{
		_Diffuse("Diffuse" , Color)=(1,1,1,1)
	}
	SubShader{
		Pass{
			//LightMode标签是Pass标签中的一种，在第九章我们会更加详细的解释他
			//在这里我们只要知道，只有定义了正确的LightMode我们才能得到一些Unity
			//中的内置光照变量，例如下边要讲到的_LightColor
			Tags{"LightMode"="ForwardBase"}
			//接下来我们写CG代码片
			CGPROGRAM

			//告诉Unity这一部分代码是要在顶点/片元着色器上工作的。
			#pragma vertex vert
			#pragma fragment frag

			//为了使用Unity中的内置一些变量，我们还需要包含内置文件Lighting.cginc
			#include "Lighting.cginc"

			//为了要在这一部分代码块中使用颜色，我们必须重新定义一个新的该属性的东西。
			fixed4 _Diffuse;

			//接下来我们要实现顶点/片元着色器之间的通信，那么一定要有输入和输出的结构体。
			struct a2v{
				float4 vertex : POSITION;
				float3 normal : NORMAL;
			};

			struct v2f{
				float4 pos : SV_POSITION;
				fixed3 color: COLOR;
			};

			//下一步我们需要在顶点着色器中，计算每个点的漫反射情况

			v2f vert (a2v v){
				//首先我们定义一个返回值o
				v2f o;
				//顶点着色器最基本的任务就是将顶点位子从模型空间转换到裁剪空间中。
				o.pos=UnityObjectToClipPos(v.vertex);

				//得到环境光的坐标信息。
				fixed3 ambient =UNITY_LIGHTMODEL_AMBIENT.xyz;
				
				//如果光源方向的计算不具有通用性是不能进行计算的
				//只有两者处于同一坐标空间下，他们的dot才有意义，
				//所以我们将其都转换到世界空间中。
				fixed3 worldNormal=normalize(mul(unity_ObjectToWorld,v.normal));
				fixed3 worldLight=normalize(_WorldSpaceLightPos0.xyz);
				
				//计算漫反射
				fixed3 diffuse =_LightColor0.rgb*_Diffuse.rgb*saturate(dot(worldNormal,worldLight));
				
				//将环境光和漫发射的光相加，得到最终光照的结果。
				o.color=diffuse+ambient;
				
				return o;
			}

			//对于我们的片元着色器，我们直接将其颜色进行输出即可。
			fixed4 frag(v2f i) : SV_Target{
				return fixed4(i.color,1.0);
			}


			ENDCG
		}
	}
	Fallback"Diffuse"
}

```

实际运行效果：

![](https://i.loli.net/2018/06/30/5b3729ae39940.png)


接下来我们尝试一下用片元着色器，逐个像素的将其渲染。

代码内容及其相近，只是我们在片元着色器中，同时计算和渲染管线，使得结果更加平滑了：

```Shader
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

// Upgrade NOTE: replaced '_World2Object' with 'unity_WorldToObject'
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 6/Diffuse Vertex-Level"
{
	Properties{
		_Diffuse("Diffuse" , Color)=(1,1,1,1)
	}
	SubShader{
		Pass{
			//LightMode标签是Pass标签中的一种，在第九章我们会更加详细的解释他
			//在这里我们只要知道，只有定义了正确的LightMode我们才能得到一些Unity
			//中的内置光照变量，例如下边要讲到的_LightColor
			Tags{"LightMode"="ForwardBase"}
			//接下来我们写CG代码片
			CGPROGRAM

			//告诉Unity这一部分代码是要在顶点/片元着色器上工作的。
			#pragma vertex vert
			#pragma fragment frag

			//为了使用Unity中的内置一些变量，我们还需要包含内置文件Lighting.cginc
			#include "Lighting.cginc"

			//为了要在这一部分代码块中使用颜色，我们必须重新定义一个新的该属性的东西。
			fixed4 _Diffuse;

			//接下来我们要实现顶点/片元着色器之间的通信，那么一定要有输入和输出的结构体。
			struct a2v{
				float4 vertex : POSITION;
				float3 normal : NORMAL;
			};

			struct v2f{
				float4 pos : SV_POSITION;
				fixed3 worldNormal :TEXCOORD0;
			};


			v2f vert (a2v v){
				//首先我们定义一个返回值o
				v2f o;

				//顶点着色器不需要计算光照模型，只需要把世界空间下的法线传递给片元着色器即可。
				o.pos=UnityObjectToClipPos(v.vertex);
				o.worldNormal=mul((float3x3)unity_ObjectToWorld,v.normal);
				return o;
			}

			
			fixed4 frag(v2f i) : SV_Target{
				fixed3 ambient =UNITY_LIGHTMODEL_AMBIENT.xyz;
				
				fixed3 worldNormal=normalize(i.worldNormal);
				fixed3 worldLightDir=normalize(_WorldSpaceLightPos0.xyz);
				
				//计算漫反射
				fixed3 diffuse =_LightColor0.rgb*_Diffuse.rgb*saturate(dot(worldNormal,worldLightDir));
				
				//将环境光和漫发射的光相加，得到最终光照的结果。
				fixed3 color=diffuse+ambient;
				
				return fixed4 (color,1.0);
			}


			ENDCG
		}
	}
	Fallback"Diffuse"
}
```

实际效果如下：

![](https://i.loli.net/2018/06/30/5b372c78e220e.png)

我们在U3d中确实能够观察到，片元着色器处理出来的结果更加平滑一些（逐个像素去处理）。

四、高光反射

我们根据之前介绍的高光反射计算方法，来尝试逐顶点光照：

```Shader
// Upgrade NOTE: replaced '_World2Object' with 'unity_WorldToObject'

// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader"Unity Shaders Book/Chapter 6/高光反射"
{
	Properties{
		//再次强调，属性的定义后边不能有分号
		//定义用户自定义面板修改颜色属性
		//分别有：光源颜色、材质颜色、材质光泽度。
		_Diffuse("Diffuse",Color)=(1,1,1,1)
		_Specular("Specular",Color)=(1,1,1,1)
		_Gloss("Gloss",Range(8.0,256))=20
	}
	SubShader{
		Pass{
			Tags{"LightMode"="ForwardBase"}
			CGPROGRAM

			//告诉Unity这一部分代码是要在顶点/片元着色器上工作的。
			#pragma vertex vert
			#pragma fragment frag

			//为了使用Unity中的内置一些变量，我们还需要包含内置文件Lighting.cginc
			#include "Lighting.cginc"

			//为了要在这一部分代码块中使用颜色，我们必须重新定义一个新的该属性的东西。
			fixed4 _Diffuse;
			fixed4 _Specular;
			float _Gloss;
			//接下来我们要实现顶点/片元着色器之间的通信，那么一定要有输入和输出的结构体。
			struct a2v{
				float4 vertex : POSITION;
				float3 normal : NORMAL;
			};

			struct v2f{
				float4 pos : SV_POSITION;
				fixed3 color : COLOR;
			};
			//利用那个Phong方法来计算高光反射
			v2f vert(a2v v){
				v2f o;
				o.pos=UnityObjectToClipPos(v.vertex);
				//得到环境光
				fixed3 ambient =UNITY_LIGHTMODEL_AMBIENT.xyz;

				fixed3 worldNormal=normalize(mul(v.normal,(float3x3)unity_WorldToObject));
				fixed3 worldLightDir=normalize(_WorldSpaceLightPos0.xyz);
				//计算漫反射光
				fixed3 diffuse=_LightColor0.rgb*_Diffuse.rgb*saturate(dot(worldNormal,worldLightDir));


				//实际反射方向
				fixed3 reflectDir=normalize(reflect(-worldLightDir,worldNormal));
				//视角方向
				fixed3 viewDir=normalize(_WorldSpaceCameraPos.xyz-mul(unity_ObjectToWorld,v.vertex).xyz);
				
				//计算高光反射光
				fixed3 specular=_LightColor0.rgb*_Specular.rgb*pow(saturate(dot(reflectDir,viewDir)),_Gloss);
				o.color=ambient+diffuse+specular;
				return o;
			}
			fixed4 frag(v2f i): SV_Target{
				return fixed4(i.color,1.0);
			}
			ENDCG
		}
	}
	Fallback"Diffuse"
}
```

在这里，我们增添了几个用户自定义的元素，在逐点计算过程中，我们首先计算出实际反射方向、再得到视角方向再
按照公式计算即可。这里值得注意的是，我们通过_WorldSpaceCameraPos得到了实际空间中的摄像机位子，再把顶
点位子从模型空间变换到世界空间下，再通过和_WorldSpaceCameraPos相减即可得到世界空间下的视角方向。


实际效果图：

![](https://i.loli.net/2018/06/30/5b374ad4f35bd.png)

接下来，我们将代码进行稍微的修改，将其变成逐像素进行的操作：

```Shader
// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'

// Upgrade NOTE: replaced '_World2Object' with 'unity_WorldToObject'

// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader"Unity Shaders Book/Chapter 6/高光反射逐像素Phong方法"
{
	Properties{
		//再次强调，属性的定义后边不能有分号
		//定义用户自定义面板修改颜色属性
		//分别有：光源颜色、材质颜色、材质光泽度。
		_Diffuse("Diffuse",Color)=(1,1,1,1)
		_Specular("Specular",Color)=(1,1,1,1)
		_Gloss("Gloss",Range(8.0,256))=20
	}
	SubShader{
		Pass{
			Tags{"LightMode"="ForwardBase"}
			CGPROGRAM

			//告诉Unity这一部分代码是要在顶点/片元着色器上工作的。
			#pragma vertex vert
			#pragma fragment frag

			//为了使用Unity中的内置一些变量，我们还需要包含内置文件Lighting.cginc
			#include "Lighting.cginc"

			//为了要在这一部分代码块中使用颜色，我们必须重新定义一个新的该属性的东西。
			fixed4 _Diffuse;
			fixed4 _Specular;
			float _Gloss;
			//接下来我们要实现顶点/片元着色器之间的通信，那么一定要有输入和输出的结构体。
			struct a2v{
				float4 vertex : POSITION;
				float3 normal : NORMAL;
			};

			struct v2f{
				float4 pos : SV_POSITION;
				float3 worldNormal : TEXCOORD0;
				float3 worldPos : TEXCOORD1;	
			};
			//利用那个Phong方法来计算高光反射
			v2f vert(a2v v){
				v2f o;
				o.pos=UnityObjectToClipPos(v.vertex);
				o.worldNormal=mul(v.normal,(float3x3)unity_WorldToObject);
				o.worldPos=mul(unity_ObjectToWorld,v.vertex).xyz;
				return o;
			}
			fixed4 frag(v2f i): SV_Target{
				//得到环境光
				fixed3 ambient =UNITY_LIGHTMODEL_AMBIENT.xyz;

				fixed3 worldNormal=normalize(i.worldNormal);
				fixed3 worldLightDir=normalize(_WorldSpaceLightPos0.xyz);
				//计算漫反射光
				fixed3 diffuse=_LightColor0.rgb*_Diffuse.rgb*saturate(dot(worldNormal,worldLightDir));


				//实际反射方向
				fixed3 reflectDir=normalize(reflect(-worldLightDir,worldNormal));
				//视角方向
				fixed3 viewDir=normalize(_WorldSpaceCameraPos.xyz-i.worldPos.xyz);
				
				//计算高光反射光
				fixed3 specular=_LightColor0.rgb*_Specular.rgb*pow(saturate(dot(reflectDir,viewDir)),_Gloss);
				return fixed4(ambient+diffuse+specular,1.0);
			}
			ENDCG
		}
	}
	Fallback"Diffuse"
}
```

实际效果图：

![](https://i.loli.net/2018/06/30/5b374e3eb7ad8.png)

可以看出，的确逐像素进行处理渲染的结果更加圆滑一些。

在这里，我们就不多浪费时间去写Blinn-Phong模型的结果了。这里我们对光照的Shader部分告一段落。




