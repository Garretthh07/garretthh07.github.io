---
layout: post
title: "Unity Shader"
description: "Unity Shader"
category: Unity Shader
tags: [Life]
---

{% include JB/setup %}


--------------------------


## Unity Shader

`Shader`, 即着色器，与之关系非常紧密的就是渲染流水线。渲染流水线的最终目的在于生成或渲染一张二维纹理，就是我们电脑上看到的所有效果。它的输入是一个虚拟摄像机，一些光源，一些`Shader`以及纹理等。

`Shader`仅仅是渲染流水线中的一个环节。渲染流水线的工作任务在于由一个三维场景出发，生成(或者说渲染)一张二维图像。换句话说，计算机需要从一系列的顶点数据，纹理等信息出发，把这些信息最终转换成一张人眼可以看到的图像。而这个工作通常是由`CPU`和`GPU`共同完成的。

![3阶段](http://7xpgi9.com1.z0.glb.clouddn.com/UnityShader_1.JPG "3阶段")

`<Real-Time Rendering ,Third Edition>`书中将渲染流程分为`3`个阶段：**应用阶段(Application Stage)**, **几何阶段(Geometry Stage)**, **光栅化阶段(Rasterizer Stage)**。每个阶段本身通常也是一个流水线系统，即包含了子流水线阶段：

* **应用阶段**
由应用主导，通常由`CPU`负责实现，开发者具有这个阶段的绝对控制权。这个阶段开发者由`3`个任务：`1`. 需要准备好场景数据，例如摄像机的位置，视锥体，场景中包含那些模型，使用那些光源等；`2`. 为了提高渲染性能，往往需要做一个粗粒度剔除(`culling`)工作，把那些不可见的物体剔除出去，就不用移交给**几何阶段**处理；`3`. 需要设置每个模型的渲染状态，这些渲染状态包括但不限于它使用的材质(漫反射颜色，高光反射颜色)，使用的纹理，使用的`Shader`等。这一阶段最重要的是输出渲染所需的几何信息，即**渲染图元(rendering primitives)**。通俗来讲，渲染图元可以是**点**，**线**, **三角面**等。这些渲染图元会被传递给下一个阶段-**几何阶段**

* **几何阶段**
**几何阶段**用于处理所有和我们要绘制的几何相关的事情。例如，决定需要绘制的图元是什么，怎样绘制它们，在哪里绘制它们。该阶段通常在`GPU`上进行。**几何阶段**负责和每个渲染图元打交道，进行逐顶点，逐多边形的操作。该阶段可以进一步分成更小的流水线阶段。**几何阶段**的一个重要任务就是把顶点坐标变换到屏幕空间中，再交给光栅器进行处理。通过对输入的渲染图元进行多步处理后，该阶段会输出屏幕空间的二维顶点坐标，每个顶点对应的深度值，着色等相关信息，并传递给下一阶段。

* **光栅化阶段**
该阶段会使用上阶段传递的数据来生成屏幕上的像素，并渲染除最终的图像。该阶段也是在`GPU`上运行的，光栅化的任务主要是决定每个渲染图元中的那些像素应该被绘制在屏幕上。它需要对上一阶段得到的逐顶点数据(例如纹理坐标，顶点颜色等)进行插值，然周再进行像素处理。**光栅化阶段**也可以分成更小的流水线阶段。

> 上面的`3`个流水线阶段和`GPU`流水线阶段不同，这里的流水线都是概念流水线，是我们为来给一个渲染流程进行基本功能划分而提出来的。`GPU`流水线才是硬件真正用于实现上述概念的流水线。

## CPU 和 GPU 之间通信
渲染流水线的起点是`CPU`,即应用阶段。应用阶段大致可分为如下`3`个阶段：

![2](http://7xpgi9.com1.z0.glb.clouddn.com/UnityShader_2.JPG "2")

* **把数据加载到显存中**
所有渲染所需的数据都需要从硬盘(`Hard Disk Drive, HDD`)中加载到系统内存(`Random Access Memory, RAM`)中。然后，网格和纹理等数据又被加载到显卡上的存储空间-显存(`Video Random Access Memory, VRAM`)中。这是因为，显卡对于显存的访问速度更快，而且大多数显卡对于`RAM`没有直接访问权。需要注意的是，真实渲染中需要加载到显存中的数据往往比图中所示复杂的多。例如，顶点的位置信息，法线方向，顶点颜色，纹理坐标等。当把数据加载到显存中后，`RAM`中的数据就可以移除了。但对于一些数据来说，`CPU`仍然需要访问它们(例如，我们希望`CPU`可以访问网格数据来进行碰撞检测)，那么我们可能就不希望这些数据被移除，因为从硬盘加载到`RAM`的过程是十分耗时的。之后，开发者还需要通过`CPU`来设置渲染状态，从而指导`GPU`如何进行渲染工作。

![3](http://7xpgi9.com1.z0.glb.clouddn.com/UnityShader_3.JPG "3")

* **设置渲染状态**
**渲染状态**定义来场景中的网格是怎样被渲染的。例如，使用哪个顶点着色器(`Vertex Shader`)/片元着色器(`Fragment Shader`), 光源属性， 材质等。如果我们没有更改渲染状态，那所有的网格都将使用同一种渲染状态。上图显示了当使用同一种渲染状态时，渲染的`3`个不同网格的结果。准备好所哟工作后，`CPU`就需要调用一个渲染命令（`Draw Call`）来告诉`GPU`。

![4](http://7xpgi9.com1.z0.glb.clouddn.com/UnityShader_4.JPG "4")

* **调用`Draw Call`**
`Draw Call`就是一个命令，它的发起方是`CPU`, 接收方是`GPU`。这个命令仅仅会指向一个需要被渲染的图元(`primitives`)列表，而不会再包含任何材质信息-这是因为我们已经在上一阶段中完成了。当给定一个`Draw Call`时，`GPU`就会根据渲染状态(例如材质，纹理，着色器等)和所有输入的顶点数据来进行计算， 最终输出成屏幕上显示的那些像素。这个过程就是`GPU`流水线。

## GPU 流水线
当`GPU`从`CPU`那里得到渲染命令后，会进行一些列流水线操作，最终把图元渲染到屏幕上。对于概念阶段的后两个阶段(**几何阶段和光栅化阶段**), 开发者无法拥有绝对的控制权，其实现的载体是`GPU`。`GPU`通过实现流水线化，大大加快了渲染速度。虽然我们无法完全控制这两个阶段的实现细节，但`GPU`向开发者开放了很多控制权。

几何阶段和光栅化阶段可以分成若干更小的流水线阶段，这些流水线阶段由`GPU`来实现，每个阶段`GPU`提供来不同的可配置性或可编程性。(1. 顶点数据, 顶点着色器, 曲面细分着色器, 几何着色器, 裁切， 屏幕映射 2.三角形设置, 三角形遍历, 片元着色器, 逐片元操作, 屏幕图像)

![4](http://7xpgi9.com1.z0.glb.clouddn.com/IMG_3805.JPG "4")

图中可以看出，`GPU`的渲染流水线接收顶点数据作为输入。这些顶点数据是由应用阶段加载到显存中，再由`Draw Call`指定，这些数据随后被传递给顶点着色器。

**顶点着色器(Vertex Shader)**是完全可编程的，它通常用于实现顶点的空间变换，顶点着色等功能。**曲面细分着色器(Tessellation Shader)**是一个可选的着色器，它用于细分图元。**几何着色器(Geometry Shader)**同样是一个可选的着色器，可以被用于执行逐图元(Per-Primitive)的着色操作，或被用于生产更多的图元。下一个流水线阶段是**裁切(Clipping)**, 这一阶段目的是将那些不再摄像机视野内的顶点裁剪掉，并剔除某些三角图元的面片，这个阶段是可配置的。例如，我们可以使用自定义的裁剪平面来配置裁剪区域，也可以通过指令控制裁剪三角图元的正面还是反面。几何概念阶段的最后一个流水线阶段是**屏幕映射(Screen Mapping)**。这一阶段是不可配置和编程的，它负责把每个图元的坐标转换到屏幕坐标系中。

**光栅化概念阶段**中的**三角形设置(Triangle Setup)**和**三角形遍历(Triangle Traversal)**阶段也都是固定函数(Fixed-Function)的阶段。接下来的**片元着色器(Fragement Shader)**，则是完全可编程的，它用于实现逐片元(Per-Fragment)的着色操作。最后，**逐片元操作(Per-Fragment Operations)**阶段负责执行很多重要的操作，例如修改颜色，深度缓冲，进行混合等，它不是可编程的，但具有很高的可配置性。 

### 顶点着色器
**顶点着色器(Vertex Shader)**是流水线的第一个阶段，它的输入来自于`CPU`。顶点着色器的处理单位是顶点，也就是说，输入进来的每个顶点都会调用一次顶点着色器。顶点着色器本身不可以创建或者销毁任何顶点，而且无法得到顶点与顶点之间的关系。例如，我们无法得知两个顶点是否属于同一个三角网格。但正是因为这样的相互独立性，`GPU`可以利用本身的特性并行化处理每个顶点，这意味着这一阶段的处理速度会很快。

顶点着色器需要完成的工作主要有: 坐标变换和逐顶点光照。当然，除了这两个主要任务外，顶点着色器还可以输出后续阶段所需的数据。

![5](http://7xpgi9.com1.z0.glb.clouddn.com/IMG_3806.JPG "5")

> 坐标变换就是对顶点的坐标(即位置)进行某种变换。顶点着色器可以在这一步中改变顶点的位置，这在顶点动画中是非常有用的。例如，我们可以通过改变顶点位置来模拟水面，布料等。但需要注意的是，无论我们在顶点着色器中怎样改变顶点的位置，一个最基本的顶点着色器必须完成的一个工作是，**把顶点坐标从模型空间转换到齐次裁剪空间**。我们在顶点着色器中会看到类似下面的代码：

```plain
o.pos = mul(UNITY_MVP, v.position)
```

> 上面这句代码的功能，就是把顶点坐标转换到齐次裁剪坐标系下，接着通常再由硬件做透视除法后，最终得到归一化的设备坐标(**Normalized Device Coordinates, NDC**):

![6](http://7xpgi9.com1.z0.glb.clouddn.com/IMG_3807.JPG "6")

> 需要注意的是，图中给出的坐标范围是`OpenGL`同时也是`Unity`使用的`NDC`,它的`z`分量范围在`[-1, 1]`之间，而在`DirectX`中, `NDC`的`z`分量范围为`[0,1]`。顶点着色器可以有不同的输出方式。最常见的输出路径是经光栅化后交给片元着色器进行处理。而在现代的`Shader Model`中，它还可以把数据发送给曲面细分着色器或几何着色器。

### 裁剪
由于我们的场景可能会很大，而摄像机的视野范围很可能不会覆盖所有的场景物体，一个很自然的想法就是，那些不在摄像机视野范围的物体不需要被处理。而**裁剪(Clipping)**就是为了完成这个目的而被提出来的。

一个图元和摄像机视野的关系有`3`种：**完全在视野中**， **部分在视野中**, **完全在视野外**。完全在视野内的图元就继续传递给下一个流水线阶段，完全在视野外的图元不会继续向下传递, 因为它们不需要被渲染。而那些部分在视野内的图元需要进行一个处理，这就是裁剪。例如，一条线段的一个顶点在视野内，而另一个顶点不在视野内，那么在视野外部的顶点应该使用一个新的顶点来代替，这个新的顶点位于这条线段和视野边界的交点处。由于我们已知的`NDC`下的顶点位置，即顶点位置在一个立方体内，因此裁剪就变得很简单；只需要将图元裁剪到单位立方体内。

![7](http://7xpgi9.com1.z0.glb.clouddn.com/IMG_3808.JPG "7")

和顶点着色器不同，这一步是不可编程的，即我们无法通过编程来控制裁剪的过程，而是硬件上的固定操作，但我们可以自定义一个裁剪操作来对这一步进行配置。

### 屏幕映射
这一步输入的坐标仍然是三维坐标系下的坐标(范围在单位立方体内)。**屏幕映射(Screen Mapping)**的任务就是把每个图元的`x`和`y`坐标转换到**屏幕坐标系(Screen Coordinates)**下。屏幕坐标系是一个二维坐标系，它和我们用于显示画面的分辨率有很大关系。假设，我们需要把场景渲染到一个窗口上，窗口的范围是从最小的窗口坐标`(x1,y1)`到最大的窗口坐标`(x2,y2)`, 其中`x1<x2`且`y1<y2`。由于我们输入的坐标范围在`[-1,1]`，因此可以想象到，这个过程实际是一个缩放的过程:

![8](http://7xpgi9.com1.z0.glb.clouddn.com/IMG_3810.JPG "8")

那么输入的`z坐标`会怎么样呢？屏幕映射不会对输入的`z坐标`做任何处理。实际上，屏幕坐标系和`z坐标`一起构成了一个坐标系，叫做**窗口坐标系(Window Coordinates)**，这些值会一起被传递到光栅化阶段。屏幕映射得到的屏幕坐标决定这个顶点对应屏幕上哪些像素以及距离这个像素有多远。有一个需要引起注意的地方是，屏幕坐标系在`OpenGL`和`DirectX`之间的差异问题。`OpenGL`把屏幕的左下角当作最小的窗口坐标值，而`DirectX`则定义了屏幕的左上角为最小的窗口坐标值。

![9](http://7xpgi9.com1.z0.glb.clouddn.com/IMG_3811.JPG "9")

产生这种差异的原因是，微软的窗口使用了这样的坐标系统，因为这和我们的阅读方式是一致的：从左到右，从上到下，并且很多图像文件也是按照这样的格式进行存储。不管原因如何，差异就这么造成了。留给开发者的就是，要时刻小心这样的差异，如果发现得到的图像是倒转的，很可能是这个原因造成的。

### 三角形设置
由这一步开始就进入光栅化阶段。从上一个阶段输出的信息是屏幕坐标系下的顶点位置以及和他们相关的额外信息，如深度值(z坐标)，法线方向，视角方向等。光栅化阶段有两个最重要的目标：**计算每个图元覆盖那些像素**，以及**为这些像素计算他们的颜色**。

光栅化的第一个流水线阶段是**三角形设置(Triangle Setup)**。这个阶段会计算光栅化一个三角网格所需的信息。具体来说，上一个阶段输出的都是三角网格的顶点，即我们得到的是三角网格每条边的两个端点。但如果要得到整个三角网格对像素的覆盖情况，我们就必须计算每条边上的像素坐标。为了能够计算边界像素的坐标信息，我们就需要得到三角形边界的表示方式。这样一个计算三角网格表示数据的过程就叫做三角形设置。它的输出为了给下一个阶段做准备。

### 三角形遍历
**三角形遍历(Triangle Traversal)**阶段将会检查每个像素是否被一个三角网格所覆盖。如果被覆盖的话，就会生成一个**片元(fragment)**。而这样一个找到那些像素被三角网格覆盖的过程就是三角形遍历，这个阶段也被称为**扫描变换(Scan Conversion)**。三角形遍历阶段会根据上一个阶段的计算结果来判断一个三角网格覆盖了哪些像素，并使用三角网格`3个`顶点的顶点信息对整个覆盖区域的像素进行插值

![10](http://7xpgi9.com1.z0.glb.clouddn.com/IMG_3812.JPG "10")

这一步的输出就是得到一个片元序列。需要注意的是，一个片元并不是真正意义上的像素，而是包含了很多状态的集合，这些状态用于计算每个像素的最终颜色。这些状态包括了(但不限于)它的屏幕坐标，深度信息，以及其他从几何阶段输出的顶点信息，例如法线，纹理坐标等。

