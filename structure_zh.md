# IJG JPEG 库：系统架构

Copyright (C) 1991-2013, Thomas G. Lane, Guido Vollbeding.
This file is part of the Independent JPEG Group's software.
For conditions of distribution and use, see the accompanying README file.

​	该文件概述了IJG JPEG软件的体系结构。 即系统中各个模块的功能以及模块之间的接口。 有关任何数据结构或调用约定的更精确的详细信息，请参见源代码中的包含文件和注释。
​    我们假定读者已经有点熟悉JPEG标准。README文件包括用于了解JPEG的参考。  libjpeg.txt文件从使用该库的应用程序程序员的角度描述了该库。 最好先阅读该文件。另外，文件coderules.txt描述了我们使用的编码样式约定。

在本文档中，JPEG专用术语遵循JPEG标准：“组件”表示颜色通道，例如“红色”或“亮度”。
     “样本”是单个分量值（即，图像数据中的一个数字）。
     “系数”是频率系数（DCT变换输出数）。
     “块”是样本或系数的阵列。
     “ MCU”（最小编码单位）是一组交错的块，其大小由采样因子确定，或者是非交错扫描中的单个块。
   我们不会互换使用术语“像素”和“样本”。 当我们说像素时，是指全尺寸图像的元素，而样本是降采样图像的元素。 因此，样本的数量可随组件而变化，而像素的数量则不变化。  （此术语在整个代码中并未严格使用，但在否则会造成混淆的地方使用。）

## 系统特性



## 可移植性的问题



## 系统概览

压缩和解压缩分成两个部分：JPEG 压缩和解压缩专属部分和 前处理或后处理功能。

压缩包含以下的主要元素：

- 前处理
  - 颜色空间转换 （e.g., RGB 转 YCbCr）
  - 边缘扩展和降采样。 这个步骤可以选做简单的平滑处理 --- 通常对低分辨率的原始数据很有用
- JPEG 专属部分
  - MCU 组装， DCT， 量化
  - 熵编码 (sequential 或 progressive, Huffman 或 arithmetic)



## Poor man's object-oriented programming



## Overall control structure



## Compression object structure

以下是 JPEG 压缩库的简略逻辑结构图：

```
                                                 |-- Colorspace conversion
                  |-- Preprocessing controller --|
                  |                              |-- Downsampling
Main controller --|
                  |                            |-- Forward DCT, quantize
                  |-- Coefficient controller --|
                                               |-- Entropy encodin
```

这个草图也描述了典型的图像数据处理的控制（子程序调用）流程。图中的每个组件都是 “object”，每个源文件都有对应的 “object” 的实现。

上图中的 "object" 有：

- Main controller
- Preprocessing controller
- Colorspace conversion
- Downsampling
- Coefficient controller
- Forward DCT 和 quantization
- Entropy encoding

## Decompression object structure



```
                                               |-- Entropy decoding
                  |-- Coefficient controller --|
                  |                            |-- Dequantize, Inverse DCT
Main controller --|
                  |                               |-- Upsampling
                  |-- Postprocessing controller --|   |-- Colorspace conversion
                                                  |-- Color quantization
                                                  |-- Color precision reduction
```



## Decompression input and output separation



## Data formats

pixel sample value的数组用以下的数据结构：

```c
    typedef something JSAMPLE;		a pixel component value, 0..MAXJSAMPLE
    typedef JSAMPLE *JSAMPROW;		ptr to a row of samples
    typedef JSAMPROW *JSAMPARRAY;	ptr to a list of rows
    typedef JSAMPARRAY *JSAMPIMAGE;	ptr to a list of color-component arrays
```

基本元素类型JSAMPLE通常是unsigned char、（signed）char或short之一。如果要支持大于8位的采样（这是一个编译时选项），则将使用Short。否则，如果可能，将使用无符号字符。如果编译器只支持有符号字符，那么在读取时有必要屏蔽该值。因此，JSAMPLE值的所有读取都必须编码为“GETJSAMPLE（value）”，其中宏在有符号字符机器上定义为“（（value）&0xFF）”，在其他机器上定义为“（（int）（value））”。

根据这些约定，JSAMPLE值可以假定为>=0。这有助于简化下采样等过程中的正确的舍入。JPEG标准规定的sample值从-128..127,是通过在DCT步骤中从sample值减去128来适应的。类似地，在解压期间，IDCT步进的输出将立即移回0..255。（注意：使用12位采样时需要不同的值。代码是用MAXJSAMPLE和CENTERJSAMPLE定义的，在8位实现中分别定义为255和128，在12位实现中分别定义为4095和2048。）

我们每行使用一个指针，而不是二维JSAMPLE数组。这种做法只需要占用少量内存，并且有几个好处：

- 代码不需要知道所分配的行宽度。这简化了边缘扩展/压缩，因为我们可以在比逻辑图片宽度更宽的数组中工作。
- 索引不需要乘法；这在许多机器上是一个性能优势。
- 即使在malloc() 无法分配大于64K的块的计算机上，也可以支持元素总数超过64K的数组。
- 构成component数组的行可以在不同的时间分配，而不需要额外的复制。这个技巧能够加速在需要访问上一行和下一行的平滑处理。

请注意，每个颜色组件都存储在一个单独的数组中；我们不使用传统的布局，即一个像素的组件存储在一起。这简化了独立处理每个组件的模块的编码，因为它们不需要知道有多少组件。此外，我们可以独立地将每个组件读写到一个临时文件中，这在处理非交叉JPEG文件时非常有用

一般来说，sample值由 `GETJSAMPLE(image[colorcomponent][row][col]) `之类的代码访问，其中 *col* 是从图像左边缘测量的，而 *row* 是从当前内存中的第一个sample行测量的。前两个索引中的任何一个都可以通过复制相关指针来预计算。

由于大多数图像处理应用程序更喜欢处理像素的components存储在一起的图像，因此传入或传出外围应用程序的数据使用传统约定：单个像素由N个连续的JSAMPLE值表示，图像行是一个由 `` (#颜色分量)*(图像宽度) JSAMPLEs` 组成的数组。在这个方案中，一行或多行数据可以由JSAMPARRAY类型的指针表示。此方案转换为JPEG库中的组件式存储。（要跳过JPEG预处理或后处理的应用程序将不得不与组件级存储相竞争。

## Suspendable processing



## Memory manager services



## Memory manager internal structure



## Implications of DNL marker







