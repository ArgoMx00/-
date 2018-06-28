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
