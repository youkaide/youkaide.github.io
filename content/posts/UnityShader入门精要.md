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

## 第六章 基础光照

- 辐射度（irradiance）：通过计算垂直于l（l:光源的方向）的单位面积上单位时间内穿过的能量来得到。使用光源方向和表面法线之间的夹角的余弦值来得到。

- 高光反射（specular）：物体表面是如何反射光线的。

- 漫反射（diffuse）：有多少光线会被折射、吸收和散射出表面。

- 出射度（exitance）：出射光线的数量和方向。

- 着色（shading）：根据材质属性、光源信息，使用一个等式去计算沿某个观察方向的出射度的过程。

- 标准光照模型：只管线直接光照

  如果在片元着色器中计算（逐像素光照）：在面片之间对顶点法线之间进行插值，Phong着色

  如果在顶点着色器中计算（逐顶点光照）：在每个顶点上计算光照，然后再渲染图元内部进行线性插值，被称为高洛德着色

  - 自发光（emissive）：使用材质的自发光颜色，这个物体不会被当成一个光源

    ![image-20261030161550316](/images/image-20261030161550316.png)

  - 高光反射

    ![image-20261030161852368](/images/image-20261030161852368.png)

  - 漫反射

    Phong模型

    ![image-20261030161622294](/images/image-20261030161622294.png)

    Blinn Phong模型：引入半程向量h

    ![image-20261030162256563](/images/image-20261030162256563.png)

    无法表现菲涅耳反射等物理现象，并且该模型是各项同性的，当固定视角和光源方向后，旋转表面，反射不会发生变换。

  - 环境光（ambient）：使用环境光来模拟间接光照

    ![image-20261030161600858](/images/image-20261030161600858.png)

    unity中在Window->Lighting->Ambient Source/Amient Color/Ambient Intensity控制

- 实现

  在原书中实现以下模型均使用的是Built-in渲染管线，但是如今这个管线以及被标记为已弃用，所有以下模型我都改为了使用URP渲染管线的Unlit Shader

  - 漫反射光照模型

    `saturate(x)`：可以用于操作矢量或标量，把x截取在[0,1]范围内。
  
    ```shader
    // Upgrade NOTE: replaced '_World2Object' with 'unity_WorldToObject'
    
    Shader "Unlit/Day3/Diffuse Vertex-Level"
    {
    	Properties
    	{
    		_Diffuse("Diffuse Color", Color) = (1, 1, 1, 1)
    	}
    
    	SubShader
    	{
    		Tags
    		{
    			"RenderType" = "Opaque" 
    			//必须声明所属管线，否则URP不会识别
    			"RenderPipeline" = "UniversalPipeline"
    		}
    
    		Pass
    		{
    			Tags{ "LightMode" = "UniversalForward" }
    
    			//URP改用HLSL
    			//CGPROGRAM
    			HLSLPROGRAM
    
    			#pragma target 3.0
    			#pragma vertex vert
    			#pragma fragment frag
    
    			//#include "UnityCG.cginc"
    			//#include "UnityLightingCommon.cginc"
    			// 【URP】Core.hlsl 提供空间变换、Attribute/Varyings 基础结构
                // 【URP】Lighting.hlsl 提供 GetMainLight() 等光照函数
                #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
                #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"
    
    			//fixed4 _Diffuse;
    			 // URP 推荐把材质属性放进 CBUFFER，以兼容 SRP Batcher
                CBUFFER_START(UnityPerMaterial)
                    half4 _Diffuse;
                CBUFFER_END
    
    
    			struct Attributes
    			{
    				float4 positionOS : POSITION;
    				float3 normalOS : NORMAL;
    			};
    
    			struct Varyings
    			{
    				float4 positionHCS : SV_POSITION;
    				half3 color : TEXCOORD0;
    			};
    
    			Varyings vert(Attributes v)
    			{
    				Varyings o;
    				o.positionHCS = TransformObjectToHClip(v.positionOS.xyz);
    
    				// 这里用球谐系数 w 分量取处理后环境光，是 URP 自定义 Shader 中常用的替代方式
                    half3 ambient = half3(unity_SHAr.w, unity_SHAg.w, unity_SHAb.w);
    				half3 worldNormal = normalize(TransformObjectToWorldNormal(v.normalOS));
    
    				Light mainLight = GetMainLight();
    				half3 worldLight = normalize(mainLight.direction);
    
    				half3 diffuse = mainLight.color * _Diffuse.rgb * saturate(dot(worldNormal, worldLight));
    				o.color = ambient + diffuse;
    
    				return o;
    			}
    
    			half4 frag(Varyings i) : SV_Target
    			{
    				return half4(i.color, 1.0);
    			}
    
    			ENDHLSL
    			//ENDCG
    		}
    	}
    
    	Fallback "Universal Render Pipeline/Simple Lit"
    }
    
    ```
  
  - 逐像素光照
  
    ```shader
    Shader "Unlit/URP/Specular Pixel-Level"
    {
        Properties
        {
            _Diffuse("Diffuse Color", Color) = (1, 1, 1, 1)
            _Specular("Specular Color", Color) = (1, 1, 1, 1)
            _Gloss("Gloss", Range(8.0, 256)) = 20
        }
    
        SubShader
        {
            Tags 
            { 
                "RenderType" = "Opaque" 
                // 【URP 必须】声明渲染管线
                "RenderPipeline" = "UniversalPipeline" 
            }
    
            Pass
            {
                // 【原】Tags{ "LightMode" = "ForwardBase" }
                // 【URP】改为 UniversalForward
                Tags { "LightMode" = "UniversalForward" }
    
                // 【原】CGPROGRAM / ENDCG
                // 【URP】改为 HLSLPROGRAM / ENDHLSL
                HLSLPROGRAM
    
                #pragma vertex vert
                #pragma fragment frag
                #pragma target 3.0
    
                // 【原】#include "UnityCG.cginc"
                // 【原】#include "UnityLightingCommon.cginc"
                // 【URP】引入 URP 的核心和光照库
                #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
                #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"
    
                CBUFFER_START(UnityPerMaterial)
                    half4 _Diffuse;
                    half4 _Specular;
                    float _Gloss;
                CBUFFER_END
    
                // 【原】struct a2v
                struct Attributes
                {
                    float4 positionOS : POSITION;
                    float3 normalOS   : NORMAL;
                };
    
                // 【原】struct v2f
                // 【URP】注意这里把颜色去掉，换成了世界空间法线和世界空间坐标
                struct Varyings
                {
                    float4 positionHCS : SV_POSITION;
                    float3 worldNormal : TEXCOORD0;
                    float3 worldPos    : TEXCOORD1;
                };
    
                Varyings vert(Attributes v)
                {
                    Varyings o;
    
                    // 【原】o.pos = mul(UNITY_MATRIX_MVP, v.vertex);
                    // 【URP】使用内置函数转换到裁剪空间
                    o.positionHCS = TransformObjectToHClip(v.positionOS.xyz);
    
                    // 【原】o.worldNormal = mul(v.normal, (float3x3)_World2Object);
                    // 【URP】注意：书里用的是转置矩阵，URP 直接调用法线转换函数更安全
                    o.worldNormal = TransformObjectToWorldNormal(v.normalOS);
    
                    // 【原】o.worldPos = mul(_Object2World, v.vertex).xyz;
                    // 【URP】使用内置函数转换到世界空间
                    o.worldPos = TransformObjectToWorld(v.positionOS.xyz);
    
                    return o;
                }
    
                half4 frag(Varyings i) : SV_Target
                {
                    // 【原】fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
                    // 【URP】使用球谐系数获取环境光
                    half3 ambient = half3(unity_SHAr.w, unity_SHAg.w, unity_SHAb.w);
    
                    // 【原】fixed3 worldNormal = normalize(i.worldNormal);
                    half3 worldNormal = normalize(i.worldNormal);
    
                    // 【原】fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
                    // 【URP】通过 GetMainLight() 获取主光源方向（指向光源）和颜色
                    Light mainLight = GetMainLight();
                    half3 worldLightDir = normalize(mainLight.direction);
    
                    // 【原】fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * saturate(dot(worldNormal, worldLightDir));
                    // 【URP】_LightColor0 替换为 mainLight.color
                    half3 diffuse = mainLight.color * _Diffuse.rgb * saturate(dot(worldNormal, worldLightDir));
    
                    // 【原】fixed3 reflectDir = normalize(reflect(-worldLightDir, worldNormal));
                    // 反射方向计算在 URP 中保持不变
                    half3 reflectDir = normalize(reflect(-worldLightDir, worldNormal));
    
                    // 【原】fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - i.worldPos.xyz);
                    // 【URP】_WorldSpaceCameraPos 依然可用，或者使用内置函数
                    half3 viewDir = normalize(_WorldSpaceCameraPos - i.worldPos);
    
                    // 【原】fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(saturate(dot(reflectDir, viewDir)), _Gloss);
                    // 【URP】同样把 _LightColor0 替换为 mainLight.color
                    half3 specular = mainLight.color * _Specular.rgb * pow(saturate(dot(reflectDir, viewDir)), _Gloss);
    
                    // 【原】return fixed4(ambient + diffuse + specular, 1.0);
                    return half4(ambient + diffuse + specular, 1.0);
                }
    
                ENDHLSL
            }
        }
        
        // 【原】Fallback "Diffuse"
        // 【URP】替换为 URP 的回退着色器
        FallBack "Universal Render Pipeline/Simple Lit"
    }
    ```

## 第七章 基础纹理

顶点的UV坐标的范围通常都会被归一到[0,1]的范围内。

Unity使用的纹理空间符合OpenGL，原点位于纹理左下角。

- 单张纹理

  使用一张纹理来代替物体的漫反射颜色。

  纹理名_ST得到纹理的缩放和偏移，.xy存储缩放值，.zw存放偏移值

- 纹理的属性

  - Alpha from Grayscale ：透明通道的值将会由每个像素的灰度值生成

  - Wrap Mode：当纹理坐标超过[0,1]范围后将会如何被平铺

    - Repeat：超过1的整数部分会被舍弃，直接使用小数部分进行采样
    - Clamp：纹理坐标大于1，将会被截取到1，小于0，会被截取到0

  - Filter Mode：当纹理由于变换而产生拉伸时将会采用哪一种滤波方式，以下模式得到的效果依次增加，但是需要耗费的性能也依次增大

    - Point

      使用最近邻（nearest neighbor）滤波，在放大和缩小的时候，他的采样像素数目通常只有一个，图像偏向像素风

    - Bilinear

      使用线性滤波，对于每个目标像素，会找四个邻近像素，对其进行线性插值混合后得到最终像素，最后结果像是被模糊了

    - Trilinear

      约等于Bilinear，但是在此基础上还使用了mipmap

    纹理缩小比放大更复杂，因为纹理缩小需要处理抗锯齿问题，常使用多级渐远纹理（mipmapping）来解决，该技术是将原纹理提前用滤波处理来得到很多更小的图像，最后形成了一个图像金字塔，每一层都是对上一层降采样的结果。

  - Format ：Unity内部使用哪种格式来存储纹理

- 凹凸映射（bump mapping）：使用一张纹理来修改模型表面的法线

  - 高度纹理（height map）：模拟表面位移最后得到一个修改后的法线值

    使用一个高度图来实现，颜色越浅表示表面越向外凸起，越深表示越向内凹

  - 法线纹理（normal map）：直接存储表面法线

    ![image-20261104173140737](/images/image-20261104173140737.png)

    - 模型空间的法线纹理：将修改后的模型空间中的表面法线存储在一张纹理中

      优点：计算和生成简单，可以提供平滑的边界，在边界通过插值得到的法线可以平滑变换，而切线空间的可能会在边缘处或者尖锐部分造成更多可见的缝合迹象

    - 切线空间的法线纹理：使用模型顶点的切线空间来存储法线

      优点：自由度高，可进行UV动画，可宠用法线纹理，可压缩。

- 渐变纹理：控制漫反射光照的结果

  在漫反射中计算出法线与光线的点积，并且将这个数值映射在[0,1]区间内，作为u去纹理贴图内采样得到颜色，在这种情况下，渐变纹理是一维查表，他只有水平方向上有用，垂直V方向上颜色都一样没用额外信息，所有只需要用u去采样

- 遮罩纹理：保护一些区域使其免于被修改或者混合纹理

  是一张RGBA图，每种通道控制一种效果，片元着色器中只采样一次。

## 第八章 透明效果

实现透明效果，一般是在渲染模型时控制它的透明通道（Alpha）

- 透明度测试，无法得到真正的半透明效果

  只要有一个片元的透明度不满足条件，那么它对应的片元就会被舍弃，产生的效果比较极端，要么完全透明，要么完全不透明

- 透明度混合，会关闭深度写入

  使用当前片元的透明度作为混合因子，与已经存储在颜色缓冲中的颜色值进行混合，得到新的颜色，因为这个方法会关闭深度写入，所有需要注意渲染顺序，但是不会关闭深度测试，因为如果有不透明物体在之前，不需要进行渲染透明物体。

  ```
  半透明物体需要让后面的透明层继续参与混合，而深度写入会让它占住深度缓冲区，从而遮挡后续透明片元。
  深度测试：将当前像素的值与深度缓冲区中的深度值进行比较，根据比较结果选择是否丢弃该片段
  深度写入：深度测试通过后发生，将当前片元的深度值更新到深度缓冲区
  ```

  ![image-20261112164449950](/images/image-20261112164449950.png)

- 渲染顺序

  在透明度混合技术中，应该在不透明物体渲染完之后再渲染半透明物体，然后将半透明物体按它们距离摄像机的远近进行排序，然后按照从后往前的顺序渲染这些半透明物体。

  但是面对有循环重叠的情况，这样渲染出来的结果会有问题

  一种由于关闭深度写入而造成的错误排序的情况的解决办法

  ```
  使用两个Pass来渲染模型，第一个Pass开启深度写入，但是不输出颜色，第二个Pass进行正常的透明度混合
  ```

- 渲染队列

  ![image-20261112164250175](/images/image-20261112164250175.png)

- ShaderLab的混合命令

  ![image-20261112164932510](/images/image-20261112164932510.png)

  ![image-20261112164946869](/images/image-20261112164946869.png)

  ![image-20261112165001274](/images/image-20261112165001274.png)

  ![image-20261112165009534](/images/image-20261112165009534.png)

- 双面渲染

  一个物体如果是透明的，应该可以看到其内部的结构，但是由于默认情况下渲染引擎剔除了物体背面的渲染图元，所以只能观察到像半个一样的状况

  使用Cull指令来控制需要剔除哪个面的渲染图元。

# 中级篇

## 第九章 复杂光照

- 渲染路径 需要为每个pass指定它使用的渲染路径，默认是前向渲染

  - 前向渲染

    每一次完整的前向渲染，需要渲染该对象的渲染图元，并计算两个缓冲区的信息，一个是颜色缓冲区，一个是深度缓冲区。如果深度缓冲中确定一个片元可见就更新颜色缓冲区中的颜色值。

    对于每个逐像素光源，都需要进行上面一次的完整的渲染流程。

    unity中的前向渲染，选择以下哪种处理方式取决于光源的类型和渲染模式

    - 逐像素处理

      最亮的平行光，渲染模式被设置成Important的光源

    - 逐顶点处理

      渲染模型设置成Not Important的光源

    - 球谐函数

      渲染模型设置成Not Important的光源

    如果按上述规则得到的逐像素光源数量小于QulitySetting中的逐像素光源数量，那么就会有更多光源以逐像素的方式进行渲染。

    光照计算的位置，前向渲染有两种Pass

    - Base Pass

      可实现光照纹理，环境光，自发光，阴影（平行光的阴影），可以计算一个逐像素的平行光以及所有逐顶点的SH光源

    - Additional Pass

      默认情况下不支持阴影，但是可以通过编译指令来开启使用，可以计算其他影响该物体的逐像素光源，每一个光源执行一次Pass

      如果开启混合模式，光照结果可以和上一次的进行叠加，但是关闭之后会把上一次计算出来的结果覆盖，通常情况下选择Blend One One

    因为环境光和自发光只希望计算一次，如果在Additional计算这两种会造成叠加多次环境光和自发光。一个Base Pass只会执行一次，但是Additional Pass会根据影响该物体的其他逐像素光源的数目被多次调用。

    在场景中包含大量实时光源的时候，前向渲染的性能会急速下降

    ![image-20261113122004897](/images/image-20261113122004897.png)

    ![image-20261113122017769](/images/image-20261113122017769.png)

  - 延迟渲染

    在前向渲染会使用的基础上，增加了额外的缓冲区，被统称为G缓冲（G-buffer），存储了所关心的表面的其他信息，例如法线，位置和用于光照计算的材质属性等。

    包含两个Pass

    - 第一个Pass

      通过深度缓冲技术判断哪一个片元是可见的，并且将其的相关信息存储到G缓冲区中，对每一个物体来说，这个Pass只会执行一次。

    - 第二个Pass

      利用G缓冲区中各个片元的信息进行光照计算，计算出来的光照颜色存储到帧缓冲中

    延迟渲染到效率不依赖于场景的复杂度，而是与屏幕空间的大小有关。对于场景用光源数目很多、使用前向渲染会造成性能瓶颈的情况下使用。

    每个光源都可以按逐像素的方式处理，但是存在以下缺陷。

    延迟渲染G-Buffer只为每个像素保存最前面的那一个表面的信息。

    - 不支持真正的抗锯齿技术

      MASS会需要G-Buffer为每个子样本存一份表面属性，会导致显存占用和带宽需求成倍增加，在第二个Pass期间需要做超采样，因为他需要读取每个子样本，并分别执行完整的光照计算，效果上已经变成了SSAA

    - 不能处理半透明物体

      因为半透明物体需要混合，但是延迟渲染只保留最前面的颜色

    - 对显卡有要求

  - 顶点照明渲染（已经被抛弃）

    相当于前向渲染中的一个子集，所有在这里可以实现的功能都可以在前向渲染中实现。

  ![image-20261113115930431](/images/image-20261113115930431.png)

- 光源类型

  - 平行光

    几何属性只有方向，而且平行光到场景中所有点的方向都是一样的，光照强度也不会随着距离而发生变化

  - 点光源

    照亮空间有限，是由空间中的一个球体定义，随着物体原理点光源，接收到的光照强度也会逐渐减小，在中心为最强，边界为0。

  - 聚光灯

    由空间中的一块锥形区域定义，表示由一个特定位置出发，向特定方向延伸的光，在锥形顶点处光照强度最强，边界处强度为0。

  - 面光源：只在烘焙时才可发挥作用

- 光照衰减

  unity使用一张纹理作为查找表来在片元计算器中计算逐像素光照的衰减

  - 衰减纹理

    需要进行预处理，并且纹理的大小会影响衰减的精度，一旦存储后无法使用其他的数学公式来计算衰减

  - 数学公式计算衰减

    可以在片元计算器中用数学公式来计算衰减

- 阴影

  使用ShadowMap的技术，阴影映射纹理本质上是一张深度图，记录了从光源位置出发，能看到的场景中距离它最近的表面位置。

  如果按照正常的渲染流程会导致性能浪费，Unity一般使用一个额外的Pass来专门更新光源的ShadowMap——ShadowCaster。

  ![image-20261113125839560](/images/image-20261113125839560.png)

  - 传统的阴影映射纹理实现

    在正常渲染的Pass中把顶点位置变换到光源空间下得到它在光源空间的三维位置信息，然后用xy分量对阴影映射纹理进行采样，从而得到该位置的深度信息。

  - 屏幕空间的阴影映射技术，需要显卡支持MRT

    调用ShadowCaster的Pass来得到可投射阴影光源的shadowmap和摄像机的深度纹理来得到屏幕空间的阴影图

## 第十章 高级纹理

- 立方体纹理

  是环境映射的一种实现方式，如果场景引入新的物体光源，或者物体发生移动时候需要重新生成立方体纹理。可以用一些特殊布局的纹理创建，手动创建一个Cubemap资源，或者用脚本生成。

  用于天空盒

- 反射

  通过入射光线的方向和表面法线的方向来计算反射方向，最后利用反射方向对立方体纹理采样

  菲涅尔反射：当光线照射到物体表面上，一部分发生反射，另一部分进入物体内部，发生折射或散射，被反射的光和入射光之间存在一定的比率关系。

  ![image-20261117151700277](image-20261117151700277.png)

- 折射

  ![image-20261117112227768](image-20261117112227768.png)

- 渲染纹理

  渲染目标纹理（Render Target Texture，RTT）：GPU将三维场景渲染到一个中间缓冲

  多重渲染目标（Multiple Render Target，MRT）：GPU允许将场景同时渲染到多个渲染目标纹理中，而不再需要为每个渲染目标纹理单独渲染完整的场景

  实现方法：

  1.在Project下创建一个渲染纹理，然后把某个相机的渲染目标设置成该渲染纹理

  2.在屏幕后处理时使用GrabPass命令或OnRenderImage函数来获取当前点屏幕图像

  效率好于GrabPass，尤其是在移动设备上，使用渲染纹理可以自定义纹理的大小，可以通过调整摄像机的渲染层来减少二次渲染时的场景大小，而GrabPass获取到的图像分辨率和显示屏幕是一致的，所有可能会在一些高分辨率的设备上造成严重的带宽影响，并且在移动设备上GrabPass不会重新渲染场景，需要CPU直接读取后背缓冲，破坏了GPU和CPU直接的并行性

- 镜子效果

  使用一个渲染纹理作为输入属性，并且把该渲染纹理在水平方向上翻转后直接显示到物体上即可

- 玻璃效果

  一般使用GrabPass来实现，一般使用法线来模拟折射效果，在使用的时候需要额外小心物体的渲染队列设置。

  使用一张法线纹理来修改模型的法线信息，通过一个cubemap来模拟玻璃的反射，在模拟折射的时候，使用了GrabPass获取玻璃后面的屏幕图像，并使用切线空间下的法线对屏幕纹理坐标偏移后，再对屏幕图像进行采样来模拟近似的折射效果。

- 命令缓冲

  扩展Unity的渲染流水线

- 程序纹理

  由计算机生成的图像，使用程序纹理的材质叫程序材质，使用SubstanceDesigner在Unity外部生成的。

