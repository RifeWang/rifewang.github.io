+++
draft = false
date = 2019-02-26T14:51:44+08:00
title = "图像处理基础"
description = "图像处理基础"
slug = ""
authors = []
tags = ["Python", "OpenCV"]
categories = ["Uncate"]
externalLink = ""
series = []
disableComments = true
+++

# 图像处理基础

现如今我们每时每刻都在与图像打交道，而图像处理也是我们绕不开的问题，本文将会简述图像处理的基础知识以及对常见的裁剪、画布、水印、平移、旋转、缩放等处理的实现。

在进行图像处理之前，我们必须要先回答这样一个问题：什么是图像？

答案是像素点的集合。

![image-pixel2](/images/pixel2.png)

如上图所示，假设红色圈的部分是一幅图像，其中每一个独立的小方格就是一个像素点（简称像素），像素是最基本的信息单元，而这幅图像的大小就是 11 x 11 px 。

1、二值图像：

图像中的每个像素点只有黑白两种状态，因此每个像素点的信息可以用 0 和 1 来表示。

2、灰度图像：

图像中的每个像素点在黑色和白色之间还有许多级的颜色深度（表现为灰色），通常我们使用 8 个 bit 来表示灰度级别，因此总共有 2 ^ 8 = 256 级灰度，所以可以使用 0 到 255 范围内的数字来对应表示灰度级别。

3、RGB图像：

红（Red）、绿（Green）、蓝（Blue）作为三原色可以调和成任意的颜色，对于 RGB 图像，每个像素点包含 RGB 共三个通道的基本信息，类似的，如果每个通道用 8 bit 表示即 256 级灰度，那么一个像素点可以表示为：

```
([0 ... 255], [0 ... 255], [0 ... 255])
```


图像矩阵：

每个图像都可以很自然的用矩阵来表示，每个像素对应矩阵中的每个元素。

例如：

1、4 x 4 二值图像：

```
0   1   0   1
1   0   0   0
1   1   1   1
0   0   0   0
```


2、4 x 4 灰度图像：

```
156   255   0     14
12    78    94    134
240   55    1     11
0     4     50    100
```

3、4 x 4 RGB 图像：

```
(156, 22, 45)   (255, 0, 0)     (0, 156, 32)    (14, 2, 90)
(12, 251, 88)   (78, 12, 34)    (94, 90, 87)    (134, 0, 2)
(240, 33, 44)   (55, 66, 77)    (1, 28, 167)    (11, 11, 11)
(0, 0, 0)       (4, 4, 4)       (50, 50, 50)    (100, 10, 10)
```


在编程语言中使用哪种数据类型来表示矩阵？答案是多维数组。例如上述 4 x 4 RGB 图像可转换为：

```
[
    [ (156, 22, 45), (255, 0, 0),  (0, 156, 32),  (14, 2, 90)   ],
    [ (12, 251, 88), (78, 12, 34), (94, 90, 87),  (134, 0, 2)   ],
    [ (240, 33, 44), (55, 66, 77), (1, 28, 167),  (11, 11, 11)  ],
    [ (0, 0, 0),     (4, 4, 4),    (50, 50, 50),  (100, 10, 10) ]
]
```

图像处理的本质实际上就是在处理像素矩阵即像素多维数组运算。

## 基本处理实现

对于图像的基本处理，本文示例使用的是 opencv-python 和 numpy 库。

示例：

```
# -*- coding: utf-8 -*-

# 图像处理
import numpy as np
import cv2 as cv

img = cv.imread('../images/cat.jpg')  # 333 x 500
rows, cols, channels = img.shape

# 1、裁剪：切割矩阵
cut = img[100:200, 333:444]  # 选取第100到200行，第333到444列的区间


# 2、画布：填充矩阵
background = np.zeros((600, 600, 3), dtype=np.uint8)  # 创建 600 x 600 黑色画布
background[100:433, 50:550] = img  # 画布指定区域填充图像


# 3、水印：合并矩阵
# addWeighted 参数：src1, alpha, src2, beta, gamma
# dst = src1 * alpha + src2 * beta + gamma;
watermark = cv.imread('../images/node.jpg') # 600 x 800
watermark = watermark[200:533, 200:700];
dst = cv.addWeighted(watermark, 0.3, img, 0.7, 0); # 确保相同的 size 和 channel


# 4、平移
# shift (x, y), 构建平移变换矩阵 M: [[1, 0, tx], [0, 1, ty]], 缺省部分填充黑色
M = np.array([[1, 0, -100], [0, 1, 100]], dtype=np.float32)
shift = cv.warpAffine(img, M, (cols, rows))


# 5、旋转
# getRotationMatrix2D 参数： center 中心点，angle 旋转角度，scale 缩放
M = cv.getRotationMatrix2D(((cols-1)/2.0, (rows-1)/2.0), -60, 1)
rotation = cv.warpAffine(img, M, (cols, rows))


# 6、缩放
# resize 参数：src 输入图像，dsize 输出图片大小，dst 输出图像，fx 水平方向缩放，fy 垂直方向缩放，interpolation 缩放算法
resize = cv.resize(img, None, fx = 2, fy = 2, interpolation = cv.INTER_LINEAR)

```

1. 裁剪：切割矩阵即可。
2. 画布：先构建指定大小的画布背景，再填充图像即可。
3. 水印：矩阵合并运算，使用 cv : addWeighted 方法。
4. 平移：构建平移变换矩阵，使用 cv : warpAffine 方法。
5. 旋转：构建旋转变换矩阵，使用 cv : warpAffine 方法。
6. 缩放：使用 cv : resize 方法。


OpenCV 提供的 resize 缩放算法包括：

![](https://raw.githubusercontent.com/RifeWang/images/master/uncate/opencv-resize.jpeg)

根据官方的文档，缩小图像时建议使用 INTER_AREA 算法，放大图像时建议使用 INTER_CUBIC（较慢）算法或者 INTER_LINEAR（更快效果也不错）算法。

## 结语

本文介绍了图像处理的基础，以及通过 OpenCV 实现了几种常见的图像处理功能。