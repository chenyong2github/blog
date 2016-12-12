---
layout:     post
title:      "Shadow mapping"
date:       2016-03-12
author:     "ChenYong"
header-img: "img/post-bg-shadowmap.jpg"
tags:
    - OpenGL
    - 图形学
    - 翻译
---

Shadow maps是当前（2016）主流的动态阴影技术。该技术优点是比较容易实现，缺点是很难实现得很完美。

在这篇教程中，我们先介绍Shadow maps的基础算法，了解它的不足之处，然后采用一些技术来得到更好的效果。因为在目前本文写的时候（2012），Shadow maps仍然是一个重要的研究课题，我们将给你们一些技术方向，这样你们可以根据需求来进一步改进你们自己的Shadow map。

## Shadow map基础
Shadow map算法由两个绘制过程构成。第一个过程，在光源处设置相机绘制场景，只计算每个片段的深度值，输出离光源最近片段的深度值生成阴影图。第二个过程，正常绘制场景，但额外测试当前片段是否在阴影中。
“是否在阴影中”测试实际上很简单。即判断当前片段离光源的距离是否比阴影图中存放的距离要远，如果是的话，表示场景中有个物体离光源比当前片段更近，那么当前片段在阴影中。
下图帮你理解它的原理：
 ![这里写图片描述](/img/in-post/shadowmap/20160312104733115.jpg)
 
## 绘制阴影图
这篇教程中，我们将只考虑方向光——光源在足够远可以被认为是平行光。这样的话，就是用正交矩阵绘制阴影图。正交矩阵跟透视投影矩阵很像，但正交矩阵不考虑透视——物体在正交摄像机下没有近大远小。

### 设置rendertarget和MVP矩阵
这里我们使用1024x1024 16位的深度贴图来保存阴影图。16位保存深度信息一般情况下精度是足够的。

```c++
 1 // The framebuffer, which regroups 0, 1, or more textures, and 0 or 1 depth buffer.
 2  GLuint FramebufferName = 0;
 3  glGenFramebuffers(1, &FramebufferName);
 4  glBindFramebuffer(GL_FRAMEBUFFER, FramebufferName);
 5 
 6  // Depth texture. Slower than a depth buffer, but you can sample it later in your shader
 7  GLuint depthTexture;
 8  glGenTextures(1, &depthTexture);
 9  glBindTexture(GL_TEXTURE_2D, depthTexture);
10  glTexImage2D(GL_TEXTURE_2D, 0,GL_DEPTH_COMPONENT16, 1024, 1024, 0,GL_DEPTH_COMPONENT, GL_FLOAT, 0);
11  glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
12  glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
13  glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
14  glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
15 
16  glFramebufferTexture(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, depthTexture, 0);
17 
18  glDrawBuffer(GL_NONE); // No color buffer is drawn to.
19 
20  // Always check that our framebuffer is ok
21  if(glCheckFramebufferStatus(GL_FRAMEBUFFER) != GL_FRAMEBUFFER_COMPLETE)
22  return false;
```

构造光源MVP矩阵：
•	投影矩阵P：正交矩阵，空间范围是(-10,10),(-10,10),(-10,20)的立方体盒子。保证场景中所有物体都被包括其中。
•	视图矩阵V：光源处摄像机的朝向为-z
•	模型矩阵M：随意

```c++
 1 glm::vec3 lightInvDir = glm::vec3(0.5f,2,2);
 2 
 3  // Compute the MVP matrix from the light's point of view
 4  glm::mat4 depthProjectionMatrix = glm::ortho<float>(-10,10,-10,10,-10,20);
 5  glm::mat4 depthViewMatrix = glm::lookAt(lightInvDir, glm::vec3(0,0,0), glm::vec3(0,1,0));
 6  glm::mat4 depthModelMatrix = glm::mat4(1.0);
 7  glm::mat4 depthMVP = depthProjectionMatrix * depthViewMatrix * depthModelMatrix;
 8 
 9  // Send our transformation to the currently bound shader,
10  // in the "MVP" uniform
11  glUniformMatrix4fv(depthMatrixID, 1, GL_FALSE, &depthMVP[0][0])
```

### 着色器
绘制过程的shaders非常简单，顶点shader输出顶点经过光源MVP变换后的齐次坐标。

```glsl
1 #version 330 core
 2 
 3 // Input vertex data, different for all executions of this shader.
 4 layout(location = 0) in vec3 vertexPosition_modelspace;
 5 
 6 // Values that stay constant for the whole mesh.
 7 uniform mat4 depthMVP;
 8 
 9 void main(){
10  gl_Position =  depthMVP * vec4(vertexPosition_modelspace,1);
11 }
```

片段shader把每个片段的深度（z值）输出到location 0。

```glsl
1 #version 330 core
2 
3 // Ouput data
4 layout(location = 0) out float fragmentdepth;
5 
6 void main(){
7     // Not really needed, OpenGL does it anyway
8     fragmentdepth = gl_FragCoord.z;
9 }
```

绘制阴影图通常比正常渲染快2倍多，因为只有低精度的深度值（16位）被写入，而不是深度和颜色。因为内存带宽一般是GPU性能瓶颈。

### 结果
这个绘制过程得到的结果贴图像这样：
 ![这里写图片描述](/img/in-post/shadowmap/20160312104821092.jpg)
图片中越黑的颜色代表着越小的z值，意味着墙模型的右上角离摄像机（也就是光源）更近。相反白颜色代表z值为1（齐次坐标下，除以w），意味着无限远（缺省值白色）

## 使用阴影图
### Shader基础
现在回到我们正常绘制的shader。在每个片段的计算中，我们将比对片段的z值与阴影图的z值前后关系。
这样的话，我们需要获取当前片段在光源MVP下的z值，所以我们需要做两次顶点变换，一次正常绘制MVP矩阵变换，一次为取光源MVP对应z值做的depthMVP矩阵变换。
这里有个小问题：经过depthMVP矩阵变换得到的齐次坐标的分量范围在[-1,1]，但是贴图采样的uv值范围必须在[0,1]。
举例说明，屏幕中间的片段齐次坐标x,y分量为(0, 0)，但是贴图中间的UVs为(0.5, 0.5)，所以MVP变换得到的齐次坐标x,y分量不能直接用于贴图采样。
这个问题可以通过在片段shader调整变换后的坐标来解决，也可以在MVP矩阵基础上左乘一个偏移矩阵解决。本质上是对MVP后的齐次坐标再做一个[-1,1]->[0,1]的变换。

```glsl
1 glm::mat4 biasMatrix(
2 0.5, 0.0, 0.0, 0.0,
3 0.0, 0.5, 0.0, 0.0,
4 0.0, 0.0, 0.5, 0.0,
5 0.5, 0.5, 0.5, 1.0
6 );
7 glm::mat4 depthBiasMVP = biasMatrix*depthMVP;
```

我们现在写顶点shader，跟之前的差不多，但是这里我们输出2个值
•	gl_Position ：当前摄像机MVP变换得到的齐次坐标
•	ShadowCoord：光源处摄像机DepthBiasMVP变换得到的齐次坐标

```
1 // Output position of the vertex, in clip space : MVP * position
2 gl_Position =  MVP * vec4(vertexPosition_modelspace,1);
3 
4 // Same, but with the light's view matrix
5 ShadowCoord = DepthBiasMVP * vec4(vertexPosition_modelspace,1);
```

片段shader也很简单：
•	texture( shadowMap, ShadowCoord.xy ).z值是光源与最近遮挡物的距离
•	ShadowCoord.z 是当前片段与光源的距离

```
1 float visibility = 1.0;
2 if ( texture( shadowMap, ShadowCoord.xy ).z  <  ShadowCoord.z){
3     visibility = 0.5;
4 }
```

所以如果ShadowCoord.z的值比texture( shadowMap, ShadowCoord.xy ).z的值大，说明当前片段是在阴影中（遮挡物挡在它与光源之间）

我们将根据比值结果来修改我们的着色，不过，环境光是不受比值结果影响的，因为环境光本来就是模拟间接光的（如果环境光也受遮挡影响的话，我们现实世界的影子将是纯黑的）

```
1 color =
2  // Ambient : simulates indirect lighting
3  MaterialAmbientColor +
4  // Diffuse : "color" of the object
5  visibility * MaterialDiffuseColor * LightColor * LightPower * cosTheta+
6  // Specular : reflective highlight, like a mirror
7  visibility * MaterialSpecularColor * LightColor * LightPower * pow(cosAlpha,5);`
```

### 结果
这是我们当前代码运行得到的结果。很明显，大概的阴影结果是对的，但是阴影质量太差了。
 ![这里写图片描述](/img/in-post/shadowmap/20160312104908998.jpg)
 
## 问题
### Shadow acne
 ![这里写图片描述](/img/in-post/shadowmap/20160312104941180.jpg)

图示的现象可以用下图简单解释：
![这里写图片描述](/img/in-post/shadowmap/20160312104953977.jpg) 

一般“解决”办法是加上一个容错偏移：我们给深度值（光源MVP变换后的齐次坐标z分量）加上一个偏移：

```
1 float bias = 0.005;
2 float visibility = 1.0;
3 if ( texture( shadowMap, ShadowCoord.xy ).z  <  ShadowCoord.z-bias){
4     visibility = 0.5;
5 }
```

结果好多了：
![这里写图片描述](/img/in-post/shadowmap/20160312105005681.jpg) 
然而，我们注意到这个固定偏移量，并不能解决场景所有位置的阴影瑕疵，一些瑕疵在环面和球面上仍然存在。
合理的做法是根据斜度（表面法线和光源方向夹角）确定一个可变偏移量：

```
1 float bias = 0.005*tan(acos(cosTheta)); // cosTheta is dot( n,l ), clamped between 0 and 1
2 bias = clamp(bias, 0,0.01);
```

曲面上Shadow acne 也没有了。
![这里写图片描述](/img/in-post/shadowmap/20160312105025306.jpg) 

另外有个技巧就是生成阴影图的时候裁剪方式设置为正面剔除（glCullFace(GL_FRONT)，背面作为遮挡进行计算，这样的话，只要你的模型有厚度的，shadow acne现象在面向光源是不存在的（背阴面还存在），见下图：
![这里写图片描述](/img/in-post/shadowmap/20160312105044437.jpg) 

### 得潘Peter Panning
shadow acne问题不存在了，但是这种通过增加偏移量解决shadow acne方式，使得投影物与阴影是分开的，图中模型墙浮在其阴影上（因此术语叫“Peter Panning”，彼得潘——小飞侠）。

“Peter Panning”的问题相对好解决：
•	首先，避免使用薄片几何体，只要几何体厚度比偏移量大就行
•	其次，用正面剔除绘制阴影图，轻松解决面向光源的shadow acne的问题

上述解决方式缺点是2倍的渲染量（以前一个薄片就够了的物体，因为阴影得加厚或者至少是双面绘制，这是个问题）
![这里写图片描述](/img/in-post/shadowmap/20160312105111813.jpg)
 
### 锯齿
解决掉shadow acne和 Peter Panning，我们还注意到阴影的边界有锯齿，锯齿可以这样理解：比如当前像素是白的，旁边的像素是黑的，相邻像素之间的颜色没有平滑过渡。
![这里写图片描述](/img/in-post/shadowmap/20160312105153183.jpg) 
#### PCF
最简单的改进方式是更改阴影图的采样方式为sampler2DShadow：当取一个位置的纹理值时，GPU会同时采样它周围的纹理值，经过双线性过滤得到一个[0,1]平均值。
比如，值0.5意味着2个采样点在阴影内，2个采样点在阴影外。
![这里写图片描述](/img/in-post/shadowmap/20160312105214949.jpg) 
如你所见，阴影边界变平滑了，但是阴影的像素颗粒感仍然比较明显。

#### Poisson Sampling
比较容易的做法是采样阴影图N次，配合PCF，可以得到比较好的效果，哪怕是N很小。这里的代码是4次采样（PCF下，每次采样GPU采样了4~5次，总采样次数是16~20次）：

```
1 for (int i=0;i<4;i++){
2   if ( texture( shadowMap, ShadowCoord.xy + poissonDisk[i]/700.0 ).z  <  ShadowCoord.z-bias ){
3     visibility-=0.2;
4   }
5 }
```
poissonDisk 是一个2维向量数组 ：

```
1 vec2 poissonDisk[4] = vec2[](
2   vec2( -0.94201624, -0.39906216 ),
3   vec2( 0.94558609, -0.76890725 ),
4   vec2( -0.094184101, -0.92938870 ),
5   vec2( 0.34495938, 0.29387760 )
6 );
```

采样次数N会影响到最后生成的片段阴影或明或暗一些：
![这里写图片描述](/img/in-post/shadowmap/20160312105228657.jpg) 


## 更进一步
除了上面介绍的方式之外，还有很多其它方式的来改进你的阴影效果，这里介绍的只是最常用的（原文还有些内容，读者有兴趣可去原文查看）

## 结论
如你所见，shadowmaps是一个复杂的课题。每一年都有其相关的变化和改进，而且直到今天（2012），没有哪个实现方式是完美的，不过幸运的是，目前大部分实现方式可被综合使用来实现接近完美的效果。
作为一个结论：建议对静态物体使用预烘焙的lightmaps，对动态物体使用shadowmaps。这两种阴影质量好坏都很重要，静态阴影很完美，但动态阴影很挫给人的感觉整个阴影质量还是很挫，反之亦然。

翻译自：<http://www.opengl-tutorial.org/intermediate-tutorials/tutorial-16-shadow-mapping/> 

代码上面链接有下载
