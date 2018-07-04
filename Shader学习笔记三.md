<table><tr><td bgcolor=orange>更复杂的光照</td></tr></table>

一、Unity的渲染路径

在Unity中，渲染路径决定了光照是如何应用到Unity Shader中的。因此，如果想要和光源打交道，我们需要为
每个Pass指定他的渲染路径。我们只有为Shader正确的选择和设置了需要的渲染路径，该Shader才能正确计算
光照的情况。

Unity支持多种类型的渲染路径，在Unity 5.0版本之前，主要有三种：前向渲染路径、延迟渲染路径和定点照明
渲染路径。大多数情况下，一个项目只使用一种渲染路径。不同类型的渲染路径可能会包含多种标签的设置。

之前我们写过类似的标签：

```Shader
Pass{
  Tags{"LightMode"="ForwardBase"}
}
```

上边的代码就是在告诉Unity，该Pass使用前向渲染路径中的ForwardBase路径。而前向渲染路径还有一种路径被
叫做ForwardAdd。下表给出了Pass中LightMode标签所支持的渲染路径的选项：

![](https://i.loli.net/2018/07/04/5b3c4a7d7e91b.png)

接下来，我们就来稍微探讨一下关于渲染路径的内容：

①前向渲染路径的原理：每进行一次完整的前向渲染，我们需要渲染该对象的渲染图元，并计算两个缓冲区的信息：
一个是颜色缓冲区，一个是深度缓冲区。我们利用深度缓冲来决定一个片元是否可见，如果可见就更新颜色缓冲区
中的颜色值。我们可以用下边的伪代码来描述前向渲染路径的大致流程：

```Shader
Pass{
  for(each fragment covered by this primitive){
    if(failed in depth test){
      //如果没有进行深度测试，说明该片元是不可见的。
      discard;
    }
    else {
      //如果该片元可见
      //就进行光照计算
      float4 color=Shading(materialInfo,pos,normal,lightDir,viewDir);
      //更新帧缓冲
      writeFramBuffer(fragment,color);
    }
  }
}
```

根据上边代码我们大概能够了解到，对于逐个像素光源，我们都需要进行上面一次完整的渲染路程。如果一个物体
在多个逐像素光源的影响区域内，那么该物体就需要执行多个Pass，每个Pass计算一个逐像素光源的光照结果，然后
在帧缓冲中把这些光照的结果混合起来的得到最终的颜色值。假设，场景中有N个物体，每个物体受到M个光源的影响，
那么渲染整个场景一共需要N*M个Pass。

②Unity中的前向渲染

在前向渲染中，当我们渲染一个物体的时候，Unity会根据场景中各个光源的设置以及这些光源对物体的影响程度
对这些光源进行一个重要度的排序，其中，一定数目的光源会按照逐像素的方式处理，然后最多有4个光源按照
逐顶点的方式处理，剩下的光源可以按照SH方式进行处理、Unity使用的判断规则大概有：

场景中最亮的平行光总是按照逐像素进行处理的。
渲染模式被设置成Not Important的光源，会按照逐顶点或者SH进行处理。
渲染模式被设置成Important的光源，会按照逐像素处理。

③内置的光照变量和函数。

前边说过，根据我们使用额渲染路径，Unity会把不同的光照变量传递给Shader。

在untiy中，对于前向渲染来说，下表给出了我们可以在Shader中直接访问到的光照变量：

![](https://i.loli.net/2018/07/04/5b3c4dc936678.png)

同时，再列出一个表，列出前向渲染中可以直接使用的内置光照函数：

![](https://i.loli.net/2018/07/04/5b3c4e48ebdb9.png)

④顶点照明渲染路径

顶点照明渲染路径是对硬件配置要求最少、运算性能最高、但同时也是得到的效果最差的一种类型。
这里同时也有能够直接访问的光照变量和函数，就不再贴出了，因为使用的情况很少，需要的话查阅
书籍即可

⑤延迟渲染路径

前向渲染的问题是：当场景中包含大量的实时光照的时候，前向渲染的性能就会急剧下降。例如，如果
我们在场景的某一块区域设置了多个光照，这些光照互相重叠，那么为了得到最终的光照效果，我们就
需要对该区域内的每个物体执行多个Pass来计算不同光源对物体的光照结果，然后还需要把结果混合起
来。然而，每执行一个Pass我们都需要重新渲染一遍物体，单很多计算实际上是重复的。接下来我们讨
论一下其中的原理：

```Shader
//第一个Pass不进行真正的光照计算
//仅仅把光照计算需要的信息存储到G缓冲中
Pass 1{
  for(each primitvie in this model){
    for(each fragment covered by this primitie){
      if(failed in depth test){
        discard;
      }
      else{
        //如果通过了深度测试，表明该片元可见，那么就把需要的信息存储到G缓冲中。
        writeGBuffer(materialInfo,pos,normal,lightDir,viewDir);
      }
    }
  }
}
Pass 2{
  for(each pixel in the Screen){
    //如果该像素是有效的，那么读取他在G缓冲中的信息
    readGBuffer(pixel,materialInfo,pos,normal,lightDir,viewDir);
    
    //根据读取到的信息进行光照计算
    float4 color=Shading(materialInfo,pos,normal,lightDir,viewDir);
    writeFramBuffer(pixel,color);
  }
}
```

可以看出，延迟渲染使用的Pass数目通常就是两个，这跟场景中包含的光源数目是没有关系的
换句话说，延迟渲染的效率不依赖场景的复杂度，而是和我们使用的屏幕空间的大小有关。这是
因为，我们需要的信息都存储在缓冲区中，而这些缓冲区可以理解成是一张张2D图像，我们的计
算实际上就是在这些图像空间中进行的。

可以访问的内置变量和函数：

![](https://i.loli.net/2018/07/04/5b3c528d22ae0.png)

因为更多的应用是前向渲染路径，所以本次学习大多还是专研前向渲染路径方面的问题。

二、Unity的光源类型

Unity中一共支持4种光源类型：平行光、点光源、聚光灯以及面光源。面光源仅在烘焙时才会发挥作用
因此我们不进行深入谈论。由于每种光源的几何定义不同，因此它们对应的光源属性也就各不相同，这就
要求我们要区别对待它们，幸运的是，Unity提供了很多内置函数来帮助我们处理光源。

光源类型有什么影响？最常用德光源属性有光源的位子、方向、颜色、强度以及衰减这五种属性。而这些
属性和它们的几何定义有相关性。接下来我们对于几种光源来讨论一下:

①平行光：它通常是作为太阳这样的角色在场景中出现的。平行光有唯一的一个位子，也就是说，它可以被放在
场景中的任意位子，他的集合属性只有方向，我们可以调整平行光的Transform中的Rotation属性来改变
他的光源方向。光照强度不会随着距离而改变。

②点光源：点光源的照亮空间是有限的；点光源可以表示由一个点发出的，向所有方向延伸的光、点光源是有
位子属性的，它是由属性中的Position定义的。对于方向属性，我们需要用点光源的位子减去某点的位子得到相对
向量方向。而点光源的颜色和强度可以在Light组件面板中进行调整。同时，点光源也是会衰减的。随着物体
远离点光源，它接受到的光源强度也会逐渐减小，点光源求新出的光照强度最强，边缘最弱。其中的衰减值可以
由函数定义，稍后我们会对其进行深入探究。

③聚光灯：聚光灯是最复杂的光源，他照亮空间同样是有限的，但不再是简单的球体。而是由空间中的一块锥形
区域定义的。他的位子属性和光照方向和点光源的定义相同。衰减也是随着物体的距离决定的。探究其衰减值的
函数稍微复杂一些，因为聚光灯的光线是锥形的，我们还需要判断一个点是否在锥形范围内等。

三、在前向渲染中处理不同的光源类型

接下来我们进行实践 ：

我们首先处理第一个Pass-BasePass，对于他来说，它处理的逐像素光照类型一定是平行光。
那么这部分的代码，我们直接高光反射+漫反射+环境光叠加一下即可。又因为是平行光，所以衰减率我们定义为1（即
没有衰减）。

然后我们定义第二个Pass-Additional Pass，这部分的渲染路径我们定义为ForwardAdd（用于前向渲染，该Pass会
计算额外的逐像素光源，每个Pass对应一个光源），对于这部分的Pass，往往要去掉Base Pass中的环境光、自发光、
逐顶点光照。Additional Pass处理的光源类型可能是任何光源类型，因此在计算光源的五个属性的时候，颜色和
强度我们仍然可以使用_LightColor0来得到。但对于位子方向和衰减属性，我们就要根据光源的类型进行分门别类
的计算了。首先，我们来看如何计算不同光源的方向：

```Shader
#ifdef USING_DIRECTIONAL_LIGHT
  fixed3 worldLightDir=normalize(_WorldSpaceLightPos0.xyz);
#else 
  fixed3 worldLightDir=normalize(_WolrdSpaceLightPos0.xyz-i.worldPosition.xyz);
#endif
```

在上边的代码中，我们首先判断了当前处理的逐像素光源的类型，这是通过使用#ifdef指令来判断是否定义了
USING_DIRECTIONAL_LIGHT来得到的额。如果当前前向渲染Pass处理的光源类型是平行光，那么Unity的底层
渲染引擎就会定义USING_DIRECTIONAL_LIGHT。如果判断得知是平行光的话，光源方向我们直接由_WorldSpace
LightPos0.xyz得到。如果判断是点光源或者是聚光灯的话，那么_WorldSpaceLightPos0.xyz表示的是世界下
的光源位子，而想要得到光源方向的话，我们就需要用这个位子减去世界空间下的顶点位子来得到。

最后，我们还需要处理不同光源的衰减问题：

```Shader
#ifdef USING_DIRECTIONAL_LIGHT
  fixed atten=1.0;
#else 
  float3 lightCoord=mul(_LightMatrix0,float4(i.worldPosition,1)).xyz;
  fixed atten=tex2D(_LightTexture0,dot(lightCoord,lightCoord).rr).UNITY_ATTEN_CHANNEL
#endif
```

我们同样是通过判断是否定义了USING_DIRECTIONAL_LIGHT来决定当前处理的光源类型。如果是平行光的话
衰减值为1.0，如果是其他光源类型，那么处理就更加复杂一些。尽管我们可以使用数学表达式来计算顶点相
对于光源和聚光灯的衰减，但是这些计算往往涉及到各种操作，计算相对较大的操作，所以Unity选择了一张
纹理作为查找表（Lookup Table，LUT），以在片元着色器中得到光源的衰减。我们首先得到光源空间下的
坐标，然后使用该坐标对衰减纹理进行采样得到衰减值。关于Unity中衰减纹理的细节会在后序学习中说明。
本节主要是为了讲解对于不同光源的处理方式，并不能作为真正的光源处理方式进行直接使用，后续学习会
给出真正的Unity Shader。

走到当前步骤，我们写出了对应的Shader：

```Shader
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

// Upgrade NOTE: replaced '_LightMatrix0' with 'unity_WorldToLight'

// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'

Shader"Unity Shaders Book/Chapter 9/前向渲染的几种光照(Test Not Use for game)"
{
	Properties{
		_Diffuse("Diffuse",Color)=(1,1,1,1)
		_Specular("Specular",Color)=(1,1,1,1)
		_Gloss("Gloss",Range(8.0,256))=20
	}
	SubShader{
		//渲染队列的归组
		Tags{"RenderType"="Opaque"}
		//这个Pass处理了平行光，如果场景中包含多个平行光，在这个例子中，
		//Unity会选择最亮的平行光传递给这个Base Pass进行逐像素的处理。
		//其他平行光会按照逐顶点在Additionl Pass中按照逐像素的处理。
		Pass{
			Tags{"LightMode"="ForwardBase"}
			CGPROGRAM

			//这个pragma编译指令，可以保证我们在Shader中使用光照衰减等
			//光照变量可以被正确赋值。这是必不可少的。
			#pragma multi_compile_fwdbase

			#pragma vertex vert
			#pragma fragment frag

			#include"Lighting.cginc"

			fixed4 _Diffuse;
			fixed4 _Specular;
			float _Gloss;

			struct a2v{
				float4 vertex:POSITION;
				float3 normal:NORMAL;
			};

			struct v2f{
				float4 pos:SV_POSITION;
				float3 worldNormal:TEXCOORD0;
				float3 worldPos:TEXCOORD1;
			};

			v2f vert(a2v v)
			{
				v2f o;
				o.pos=UnityObjectToClipPos(v.vertex);
				o.worldNormal=UnityObjectToWorldNormal(v.normal);
				o.worldPos=mul(unity_ObjectToWorld,v.vertex).xyz;
				return o;
			}
			fixed4 frag(v2f i):SV_Target{
			
				fixed3 worldNormal=normalize(i.worldNormal);
				fixed3 worldLightDir=normalize(UnityWorldSpaceLightDir(i.pos));
				
				fixed3 ambient=UNITY_LIGHTMODEL_AMBIENT.xyz;
				fixed3 diffuse=_LightColor0.rgb*_Diffuse.rgb*max(0,dot(worldNormal,worldLightDir));

				fixed3 viewDir=normalize(_WorldSpaceCameraPos.xyz-i.worldPos.xyz);
				fixed3 halfDir=normalize(worldLightDir+viewDir);
				fixed3 specular=_LightColor0.rgb*_Specular.rgb*pow(max(0,dot(worldNormal,halfDir)),_Gloss);
				//由于平行光是没有衰减的，所以我们直接设定衰减值为1.0即可。
				fixed atten=1.0;
				return fixed4(ambient+(diffuse+specular)*atten,1.0);

			
			}


			ENDCG
		}
		//接下来，我们定义一个Additional Pass
		Pass{
			//设置渲染路径
			Tags{"LightMode"="ForwardAdd"}
			//在这里，我们还开启了混合模式。这是因为，我们希望这部分Pass
			//计算得到的光照结果可以在帧缓存中叠加效果。
			Blend One One
			CGPROGRAM
			//给出这个指令，能够保证我们在当前Pass中访问到正确的光照变量
			#pragma multi_compile_fwdadd
			
			#pragma vertex vert 
			#pragma fragment frag
			#include"Lighting.cginc"
			#include"AutoLight.cginc"

			fixed4 _Diffuse;
			fixed4 _Specular;
			float _Gloss;
			struct a2v{
				float4 vertex:POSITION;
				float3 normal:NORMAL;
			};

			struct v2f{
				float4 pos:SV_POSITION;
				float3 worldNormal:TEXCOORD0;
				float3 worldPos:TEXCOORD1;
			};
			v2f vert(a2v v)
			{
				v2f o;
				o.pos=UnityObjectToClipPos(v.vertex);
				o.worldNormal=UnityObjectToWorldNormal(v.normal);
				o.worldPos=mul(unity_ObjectToWorld,v.vertex).xyz;
				return o;
			}

			fixed4 frag(v2f i):SV_Target{
				fixed3 worldNormal=normalize(i.worldNormal);
				//判断光源是否定义了USING_DIRECTIONAL_LIGHT，如果定义了，那么说明该光源类型是品星光
				//否则就是点光源或者是聚光灯，两种光的求光源方向的方式不同，所以要进行区别。
				#ifdef USING_DIRECTIONAL_LIGHT
					fixed3 worldLightDir=normalize(_WorldSpaceLightPos0.xyz);
				#else 
					fixed3 worldLightDir=normalize(_WorldSpaceLightPos0.xyz-i.worldPos.xyz);
				#endif

				fixed3 diffuse=_LightColor0.rgb*_Diffuse.rgb*max(0,dot(worldNormal,worldLightDir));
				fixed3 viewDir=normalize(_WorldSpaceCameraPos.xyz-i.worldPos.xyz);
				fixed3 halfDir=normalize(viewDir+worldLightDir);				
				fixed3 specular=_LightColor0.rgb*_Specular.rgb*pow(max(0,dot(worldNormal,halfDir)),_Gloss);

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
			}

			ENDCG


		}
	}
}

```

多光源的实践效果：

![](https://i.loli.net/2018/07/04/5b3c6f1b85834.png)








