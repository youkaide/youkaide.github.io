+++
date = '2026-10-14T10:00:00+08:00'
draft = false
title = 'UnityShader入门精要'
description = '一句话摘要'
categories = ['shader']
showTableOfContents = true

+++

{{< katex >}}

# 基础篇

## 第二章 渲染器流水线

- 渲染流程：**应用阶段、几何阶段、光栅化阶段**

  - 应用阶段（CPU进行）

    - 获取场景数据

    - 粗粒度剔除（将不可见物体剔除渲染列表）
    - 设置每个模型的渲染状态
    - 输出**渲染图元**（渲染所需的几何信息）

  - 几何阶段（GPU进行）

    - 对渲染图元进行处理
    - 将顶点坐标变换到屏幕空间中
    - 输出屏幕空间的二维坐标、每个顶点对应的信息

  - 光栅化阶段（GPU进行）

    - 判断像素是否需要被绘制
    - 对顶点数据进行插值和逐像素计算

  但现代渲染有例外：**GPU Driven Rendering**。它可以把一部分原本属于应用阶段的工作，比如视锥剔除、遮挡剔除、LOD 选择、间接绘制参数生成，放到 GPU 的 Compute Shader 里做。这时“应用阶段”的部分职责被 GPU 化了。不过游戏逻辑、物理、场景管理、资源管理、命令提交等，通常仍然由 CPU 主导。

- CPU到GPU

  - CPU（应用阶段）

    - 把数据加载到显存中

      数据从硬盘（HDD）中加载到系统内存（RAM，CPU的内存），然后网格和纹理等数据加载到显存（VRAM，GPU的内存）。因为显卡对显存的访问速度更快，且大多显卡无直接权限访问RAM。移动到显存后的数据可以在RAM中删除（如有需要也可以不删除）

    - 设置渲染状态

      状态定义场景中网格被渲染的方式

    - 调用DrawCall

      DrawCall（渲染命令），CPU调用其告诉GPU可以按当前设置来进行渲染。

      一个DrawCall会指向本次调用需要渲染的图元列表

- GPU流水线（输入由DrawCall指定的顶点数据）

  ![image-20261014203229021](/images/image-20261014203229021.png)

  - 顶点着色器 Vertex Shader（完全可编程）：顶点的空间变换、顶点着色等功能

    从CPU获取顶点，每个顶点调用一次，不知顶点与顶点间的关系，并行化处理每个数据，速度快。对顶点进行坐标变换和逐顶点光照。

    - 坐标变换：MVP，法线变换，切线/副切线变换，纹理坐标变换，顶点动画，自定义变换。法线变换不能用MVP，是因为非均匀缩放会破坏法线，通常用模型矩阵的逆转置。

      | 数据     | 常用变换                    | 目标               |
      | :------- | :-------------------------- | :----------------- |
      | 顶点位置 | Model → View → Projection   | 裁剪空间           |
      | 法线     | 模型矩阵的逆转置            | 世界/观察空间      |
      | 切线     | 模型矩阵 3×3，必要时正交化  | 世界/观察空间      |
      | 副切线   | 法线 × 切线，再乘手性       | 世界/观察空间      |
      | UV       | 直接传，或乘纹理变换矩阵    | UV 空间            |
      | 顶点动画 | 骨骼矩阵、Morph、程序化修改 | 模型空间或世界空间 |
      | 顶点颜色 | 直接传，插值                | 片元               |

    - 逐顶点光照：对每个顶点计算光照，得到该顶点的颜色，然后把这个颜色传给光栅化阶段，由 GPU 在三角形内部插值，最终得到每个片元的颜色。性能好，但是质量低，不适合高光（非线性很强，插值后不准确），大三角形和精细光照，现代渲染常用逐像素光照。

  - 曲面细分着色器 Tessellation Shader（可选）：细分图元，根据距离，曲率等在面片内部生成大量的新顶点，并且把这些顶点连接成许多小三角形。

  - 几何着色器 Geometry Shader（可选）：逐图元着色、产生更多图元

  - 裁剪 Clipping（可配置，不可编程）：将相机外的顶点裁剪掉，并且剔除某些三角图元的面片。

    - 正面/背面（OpenGL中的逆时针为正面）：根据三角形在屏幕空间中的顶点绕序，判断它面向相机还是背对相机。
    - 背面剔除：把背对相机的三角形丢掉，常规实体渲染默认选择。
    - 正面剔除：把面向相机的三角形丢掉，阴影贴图、内部渲染、反射、天空盒内部。
      - 阴影贴图：因为背面深度>正面深度，能减少表面把自己判定成阴影的概率，减少自遮挡（Shadow acne）
      - 反射：反射的时候相机需要根据反射平面做镜像，并且用这个镜像相机来渲染场景，但是镜像相机中的正面应该是之前相机的背面，也可以在渲染平面反射纹理的时候翻转剔除模式。
    - 不剔除：透明物体、树叶、布料、双面材质。
      - 薄片没有厚度，正面和背面几乎共面，剔除可能会导致物体消失，且可能从不同角度看到正面和被骗。

  - 屏幕映射 Screen Mapping（不可配置和编程）：把 NDC 坐标转换成屏幕坐标，屏幕坐标和Z坐标共同构成**窗口坐标系**

    OpenGL和DirectX的屏幕坐标系差异。对于一张512＊512大小的图像，在OpenGL中其（0, 0）点在左下角，而在DirectX中其(0, 0)点在左上角

    ![image-20261014210323836](/images/image-20261014210323836.png)

  - 三角形设置 Triangle Setup（固定函数）：由三角形的顶点计算一个三角网格表示信息。

  - 三角形遍历 Triangle Traversal （固定函数）/扫描变换 Scan Conversion：检测每个像素是否被一个三角网格所覆盖，如果覆盖就生成一个片元，片元的状态由三个顶点的信息进行插值得到。

    ![image-20261014214922946](/images/image-20261014214922946.png)

  - 片元着色器 Fragment Shader （完全可编程）/像素着色器 Pixel Shader：根据上一步插值后的片元信息，来计算该片元的输出颜色。可以完成纹理采样等渲染技术。只能影响单个片元，无法将结果发送给别的片元（例外：访问导数信息，gradient，derivative，硬件以 2×2 片元块为单位执行，允许通过 `dFdx/dFdy/fwidth` 读取相邻片元属性的屏幕空间差分。这是有限的结构化例外，主要用于 mipmap 选择、边缘检测、法线重建等。它不是任意通信，也不能读邻居的最终颜色。）

  - 逐片元操作 Per-Fragment Operations （不可编程，可配置）/输出合并阶段 Output-Merger：修改颜色，深度缓冲，混合。

    - 决定每个片元的可见性
    - 将该片元颜色与颜色缓冲区的颜色合并

    ![image-20261014215615575](/images/image-20261014215615575.png)

    - 模板测试：GPU会首先读取（使用读取掩码）模板缓冲区中该片元位置的模板值，然后将该值和读取（使用读取掩码）到的参考值（reference value）进行比较，然后选择是否丢弃。一般用于限制渲染区域，或者渲染阴影，轮廓渲染等。
    - 深度测试：GPU会把该片元的深度值和已经存在于深度缓冲区中的深度值进行比较。
    - 混合：不透明物体可以用计算到的只覆盖掉颜色缓冲区中的像素值，但是半透明就需要使用混合操作。
    - 以上测试一般在片元着色器之前，有些时候可能会产生冲突，发生冲突需要禁用提前测试。

  - 双重缓冲 Double Buffering：对场景的渲染是在幕后发生的（后置缓冲，Back Buffer），一旦场景渲染到后置缓冲中就与前置缓冲（Front Buffer）交换，从而使图像连续。

- 术语科普

  - OpenGL/DirectX：图像应用编程接口

  - HLSL/GLSL/CG：着色器语言

    - GLSL：跨平台，opengl提供，依赖硬件

    - HLSL：微软提供，与硬件无关，平台有限

    - CG：跨平台，根据平台来编译成不同的中间语言

- Draw Call：CPU调用图像编程接口，如OpenGL中的glDrawElements命令或者DirectX中的DrawIndexedPrimitive命令，以命令GPU进行渲染的操作。

  - 使用命令缓冲区，CPU写命令，GPU读命令，添加与读取相互独立。

    ![image-20261014221549440](/images/image-20261014221549440.png)

  - DrawCall多了影响帧率，因为在每次调用Draw Call之前，CPU需要向GPU发送很多内容，包括数据、状态和命令等，GPU渲染很快，因此渲染速度往往快于CPU提交命令的速度

  - 减少DrawCall

    - 批处理 Batching：把很多小的DrawCall合并成一个大的Draw Call，再发送给GPU，然后在一个Draw Call中渲染它们。但要注意的是，使用批处理合并的网格将会使用同一种渲染状态。一般用于静态物体，因为动态物体每一帧都需要重新合并。
    - 避免使用大量很小的网格

    - 避免使用过多的材质

## 第三章 Unity Shader

- 材质

  - 创建一个材质
  - 创建一个Unity Shader，并把他赋值
  - 将材质赋给渲染对象
  - 在材质属性中调整Unity Shader

- Unity Shader

  - standard Surface Shader

    包含标准光照模型的表面着色器模板

    unity对顶点/片元着色器更高一级的抽象

    ```
            Shader  "Custom/Simple  Surface  Shader"  {
                SubShader  {
                  Tags  {  "RenderType"  =  "Opaque"  }
                  CGPROGRAM
                  #pragma  surface  surf  Lambert
                  struct  Input  {
                      float4  color  :  COLOR;
                  };
                  void  surf  (Input  IN,  inout  SurfaceOutput  o)  {
                      o.Albedo  =  1;
                  }
                  ENDCG
                }
                Fallback  "Diffuse"
            }
    ```

  - Unlit Shader

    不含光照的基本顶点/片元着色器

    ![image-20261019163120018](/images/image-20261019163120018.png)

    顶点/片元着色器写在Pass语义块中

  - Image Effect Shader

    后处理

  - Compute Shader

    利用GPU的并行性来进行一些与常规渲染流水线无关的计算

  - 标准形式

    ```
            Shader  "MyShader"  {
                Properties  {
                  // 所需的各种属性
                }
                SubShader  {
                  // 真正意义上的Shader代码会出现在这里
                  // 表面着色器（Surface Shader）或者
                  // 顶点/片元着色器（Vertex/Fragment Shader）或者
                  // 固定函数着色器（Fixed Function Shader）
                }
                SubShader  {
                  // 和上一个SubShader类似
                }
            }
    ```

- Shader Lab

  Unity为开发者提供的高层级的渲染抽象层。

  基本结构：

  ```ShaderLab
          Shader  "ShaderName"  {
              Properties  {
                // 属性
              }
              SubShader  {
                  // 显卡A使用的子着色器
              }
              SubShader  {
                  // 显卡B使用的子着色器
              }
              Fallback  "VertexLit"
          }
  ```

  - `Shader  "Custom/MyShader"  {      }`这个Unity Shader在材质面板中的位置就是：Shader ->Custom -> MyShader。

    ![image-20261015173602470](/images/image-20261015173602470.png)

  - Properties

    在Unity中，这些属性(Name）的名字通常由一个下划线开始

    ```
            Properties  {
                Name  ("display  name",  PropertyType)  =  DefaultValue
                Name  ("display  name",  PropertyType)  =  DefaultValue
                // 更多属性
            }
    ```

    ![image-20261015174123014](/images/image-20261015174123014.png)

    例子

    ```
            Shader  "Custom/ShaderLabProperties"  {
                Properties  {
                  //  Numbers  and  Sliders
                  _Int  ("Int",  Int)  =  2
                  _Float  ("Float",  Float)  =  1.5
                  _Range("Range",  Range(0.0,  5.0))  =  3.0
                  //  Colors  and  Vectors
                  _Color  ("Color",  Color)  =  (1,1,1,1)
                  _Vector  ("Vector",  Vector)  =  (2,  3,  6,  1)
                  //  Textures
                  _2D  ("2D",  2D)  =  ""  {}
                  _Cube  ("Cube",  Cube)  =  "white"  {}
                  _3D  ("3D",  3D)  =  "black"  {}
                }
    
                FallBack  "Diffuse"
            }
    ```

  - SubShader

    可以有多个，然后选择第一个可以在目标平台上运行的，都不支持的话，使用fullback指定的subshader

    ```
            SubShader  {
                // 可选的
                [Tags]//标签
           	    Tags  {  "TagName1"  =  "Value1"  "TagName2"  =  "Value2"  }
    
                // 可选的
                [RenderSetup]//状态
    
                Pass  {
                [Name]
                Name  "MyPassName"//定义Pass的名称
                [Tags]
                [RenderSetup]
                
                // 可选的
                UsePass  "MyShader/MYPASSNAME"//使用别的UnityShader中的Pass，使用大写名字
                // 可选的
                GrabPass //该Pass负责抓取屏幕并将结果存储在一张纹理中，以用于后续的Pass处理
                //  Other  code
            }
                //  Other  Passes
                //每个pass定义一次完整的渲染流程，数目过多会导致渲染性能下降
            }
    ```

    **常见渲染状态**

    ![image-20261015174712607](/images/image-20261015174712607.png)

    pass外设置全局使用，pass内设置仅pass使用

    **标签类型**

    ![image-20261015174917972](/images/image-20261015174917972.png)

    以上仅在subshader中声明

    **Pass的标签**

    ![image-20261015175351526](/images/image-20261015175351526.png)

    **Fallback**

    ```
            Fallback  "name"
            // 或者
            Fallback  Off
    ```

    所以subshader都不支持的话，使用fullback指定的subshader或者不管

  - 其他语义

    CustomEditor语义来扩展编辑界面。

    Category语义来对Unity Shader中的命令进行分组。

## 第四章 数学

- Unity使用的坐标系

  - 对于模型空间和世界空间，Unity使用左手坐标系
  - 对于观察空间，Unity使用的是右手坐标系
    - 观察空间：以摄像机为原点的坐标系（z轴坐标减少意味着场景深度的增加）

- 向量的点积（dot product, 内积,inner product)

  - 点乘公式，满足交换律，结合律

  $$
  坐标分量法：a ⋅ b =x 1​ x 2​ +y 1​ y 2​ +z 1​ z 2
  $$

  $$
  几何意义：|a ⋅ b| =∣a∣∣b∣cosθ
  $$

  - 任何两个向量的点积ab等于b在a方向上的投影再乘上a的长度。
  - 一个向量和本身进行点积的结果，就是该矢量的模的平方。

- 向量的叉积 (cross product,外积,outer product)

  - 叉乘公式，不满足交换律，结合律
    $$
    坐标分量法：a * b = (ax, ay, az) * (bx, by, bz) = (aybz - azby, azbx - axbz, axby - aybx)
    $$

    $$
    几何意义：| a * b | =∣a∣∣b∣sinθ
    $$

  - 可以由叉积计算出来由ab组成的平行四边形的面积。

- 矩阵（Matrix)

  - 转置矩阵

    ![image-20261028153808945](/images/image-20261028153808945.png)

  - 逆矩阵

    ![image-20261028153927212](/images/image-20261028153927212.png)

    ![image-20261028154027698](/images/image-20261028154027698.png)

  - 正交矩阵

    ![image-20261028154324406](/images/image-20261028154324406.png)

    ![image-20261028154627949](/images/image-20261028154627949.png)

- 矩阵的几何意义：变换

  - 仿射变换：合并线性变换和平移变换的变换类型 。

    ![image-20261028155014615](/images/image-20261028155014615.png)

  - 平移矩阵

    ![image-20261028155336559](/images/image-20261028155336559.png)

  - 缩放矩阵

    ![image-20261028155351239](/images/image-20261028155351239.png)

  - 旋转矩阵

    ![image-20261028155448033](/images/image-20261028155448033.png)

    ![image-20261028155501068](/images/image-20261028155501068.png)

  因为矩阵的乘法不满足交换律，因此不同的乘法顺序得到的结果是不一样的。约定的变换顺序是先缩放，再旋转，最后平移。

  ![image-20261028155940998](/images/image-20261028155940998.png)

  - 正交投影

    ![image-20261028162131943](/images/image-20261028162131943.png)

  - 透视投影

    透视投影矩阵 = 正交投影矩阵 * 压缩矩阵

    ![image-20261028162113956](/images/image-20261028162113956.png)

    ![image-20261028162124221](/images/image-20261028162124221.png)

    以上内容为games101所讲

  - 总结

    ![image-20261028162322935](/images/image-20261028162322935.png)

- 法线变换

  - 切线：通常与纹理空间对齐，并且与法线方向垂直

  - 变换

    切线和法线都是方向矢量，不会受到平移的影响。使用其他用于变换顶点的变换矩阵，对切线无影响，但是法线会发生变换。

    ![image-20261028163209672](/images/image-20261028163209672.png)

    两个列向量的点乘可以写成矩阵相乘的形式，但是需要将前一个列向量转置后变成行向量。

- unity内置变量

  ![image-20261028163405746](/images/image-20261028163405746.png)![image-20261028163417693](/images/image-20261028163417693.png)

# 初级篇

## 第五章 Unity Shader

本书使用Unity 5.2.1 免费版，系统环境为Mac OS X 10.9.5。

- 顶点/片元着色器

  - 创建一个Unity Shader
  - 创建一个材质，将shader赋予这个材质

  ```shader
  Shader "UNity Shaders Book/Chapter 5/Simple Shader"//定义shader的位置和名字
  {
  	Properties
  	{
  		//声明一个Color类型的属性
  		_Color ("Color Tint", Color) = (1.0, 1.0, 1.0, 1.0);
  	}
  	SubShader
  	{
  		pass
  		{
  			CGPROGRAM//和ENDCG已经包围了CG片段
  			
  			//编译指令，告诉Unity哪个函数包含了顶点着色器/片段着色器
  			#pragma vertex vert
  			#pragma fragment frag
  			
  			//如果在Properties声明了属性，需要在CG代码中定义一个与属性名称和类型都匹配的变量
  			fixed4 _Color;
  			
  			//POSITION，SV_POSITION为CG/HLSL中的语义
  			//使用一个结构体来定义顶点着色器的输入
  			//a: application, v: vertex shader, 2 == to
  			struct a2v
  			{
  				//POSITION: 把模型空间的顶点坐标填充到vertex中
  				float4 vertex : POSITION;
  				//NORMAL: 把模型空间的法线方向填充到normal中
  				float3 normal : NORMAL;
  				//TEXCOORDO: 把模型的第一套纹理坐标填充到texcoord中
  				float4 texcoord : TEXCOORDO;
  			}
  			
  			//使用一个结构体来定义顶点着色器的输出
  			struct v2f
  			{
  				//SV_POSITION: pos内包含了裁剪空间中的顶点坐标的位置信息
  				float4 pos : SV_POSITION;
  				//COLOR0: 存储颜色信息
  				fixed3 color : COLOR0;
  			}
  			
  			//顶点着色器，逐顶点执行
  			v2f vert(float4 v : POSITION) : SV_POSITION
  			{
  				v2f o;
  				//mul：矩阵乘法函数
  				//UNITY_MATRIX_MVP: MVP变换矩阵，将v从模型空间换到裁剪空间
  				o.pos = mul(UNITY_MATRIX_MVP, v.vertex);
  				//v.normal: 法线分量，位于[-1, 1]
  				//屏幕/纹理颜色每个分量一般在[0, 1]
  				//把法线当颜色输出，用于调试可视化或写入法线贴图
  				o.color = v.normal * 0.5 + fixed3(0.5, 0.5, 0.5);
  				return o;
  			}
  			
  			//逐片段调用,片段着色器的输入实际上是把顶点着色器的输出镜像插值后得到的结果
  			//fixed: 一种四分量、低精度的数据类型，通常用来表示颜色，.r/.g/.b/.a,分别是红色，绿色，蓝色，透明度（Alpha通道）
  			//SV_Target: 一个系统语义，将输出颜色存储到一个渲染目标。
  			flxed4 frag(v2f i) : SV_Target
  			{
  				//返回颜色，其中(0, 0, 0)为黑色，(1, 1, 1)为白色
  				//将插值后的i.color显示到屏幕上
  				return fixed4(i.color, 1.0);
  			}
  			
  			ENDCG
  		}
  	}
  }
  ```

  填充到POSITION，TANGENT等语言中的数据有材质的Mesh Render所提供。在每一帧调用Draw Call的时候，Mesh Render组件会把它负责渲染到模型数据发给Unity Shader。

  法线贴图默认淡蓝色：因为模型表面大多数法线朝外，在切线空间中为(0, 0, 1)
  $$
  (0, 0, 1) * 0.5 + 0.5 = (0.5, 0.5, 1)
  $$
  Properties语义块用于声明材质属性，不是必须的。

  ![image-20261029114049747](/images/image-20261029114049747.png)

  uniform关键词是CG中修饰变量和参数的一种修饰词，用于提供一些关于该变量初始值是如何制定和存储的相关信息，unity中可以省略。

- Unity提供的内置文件和变量

  内置的包含文件：#include " ",后缀为.cginc

  ```
  CGPROGRAM
  // ...
  include "UnityCG.cginc"
  // ...
  ENDCG
  ```

  ![image-20261029114724865](/images/image-20261029114724865.png)

  ![image-20261029114824861](/images/image-20261029114824861.png)

  ![image-20261029114835167](/images/image-20261029114835167.png)

- Unity提供的CG/HLSL语义

  - 语义

    赋给Shder输入和输出的字符串，让Shader指导从哪里读取数据并且把数据输出到哪里。即使语义的名称一样，如果出现的位置不同含义也不同。

    DirectX 10以后，由SV（system-value) 开头的代表系统数值，用这些语义描述的变量是不可以随便赋值的。

    ![image-20261029115358312](/images/image-20261029115358312.png)

    ![image-20261029115417712](/images/image-20261029115417712.png)

    ![image-20261029115434673](/images/image-20261029115434673.png)

- Debug

  - 使用假彩色图像：把需要调试的变量映射到[0,1]之间，再输出到屏幕上

    ![image-20261029115810194](/images/image-20261029115810194.png)

  - Visual Studio

    使用Graphics Debugger来进行调试

  - Frame Debugger：unity原生帧调试器

    查看该帧进行时的各种渲染事件，DrawCall序列，清空帧缓存等

- 规范Shader

  ![image-20261029120348213](/images/image-20261029120348213.png)

  在大多桌面GPU上所有计算都按最高精度计算，但是在移动端上会有区别。

  Shader Model ：微软提出的规范，它们决定了Shader中各个特性的能力这些特性和能力体现 在Shader能使用的运算指令数目 、 寄存器个数等各个方面 。Shader Model 等级越高 Shader 的能力就越大 。