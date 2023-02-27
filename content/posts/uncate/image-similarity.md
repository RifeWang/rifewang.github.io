+++
draft = false
date = 2019-03-14T15:15:10+08:00
title = "图像相似性：哈希和特征"
description = "图像相似性：哈希和特征"
slug = ""
authors = []
tags = ["Python", "OpenCV"]
categories = ["Uncate"]
externalLink = ""
series = []
disableComments = true
+++

## 引言

如何判断图像的相似性？

直接比较图像内容的 md5 值肯定是不行的，md5 的方式只能判断像素级别完全一致。图像的基本单元是像素，如果两张图像完全相同，那么图像内容的 md5 值一定相同，然而一旦小部分像素发生变化，比如经过缩放、水印、噪声等处理，那么它们的 md5 值就会天差地别。

本文将会介绍图像相似性的两大有关概念：图像哈希、图像特征。

## 图像哈希

图像通过一系列的变换和处理最终得到的一组哈希值称之为图像的哈希值，而中间的变换和处理过程则称之为哈希算法。

下面以 Average Hash 算法为例描述这一基本过程：

![](/images/uncate/image-ahash.jpeg)

1、Reduce size : 将原图压缩到 8 x 8 即 64 像素大小，忽略细节。

2、Reduce color : 灰度处理得到 64 级灰度图像。

3、Average the colors : 计算 64 级灰度均值。

4、Compute the bits : 二值化处理，将每个像素与上一步均值比较并分别记为 0 或者 1 。

5、Construct the hash : 根据上一步结果矩阵构成一个 64 bit 整数，比如按照从左到右、从上到下的顺序。最后得到的就是图像的均值哈希值。


参考：http://www.hackerfactor.com/blog/?/archives/432-Looks-Like-It.html


如果你稍加留意，就会发现 Average Hash 均值哈希算法的处理过程相当简单，优点就是计算速度快，缺点就是局限性比较明显。

当然计算机视觉领域发展到现在已经有了多种图像哈希算法，OpenCV 支持的图像哈希算法包括：

- AverageHash : 也叫 Different Hash.
- PHash : Perceptual Hash.
- MarrHildrethHash : Marr-Hildreth Operator Based Hash.
- RadialVarianceHash : Image hash based on Radon transform.
- BlockMeanHash : Image hash based on block mean.
- ColorMomentHash : Image hash based on color moments.


这些哈希算法的具体实现过程不在本文的讲述范围内，我们重点关注的是他们的实际表现。

![](/images/uncate/image-hash/jpeg)

如上图所示，左下角标明了如水印、椒盐噪声、旋转、缩放、jpeg压缩、高斯噪声、高斯模糊、对比度等对抗影响，右下角则是各种哈希算法，圆锥体的高度则代表哈希算法对各种影响的抗性，高度越高说明抗性越高、越能成功匹配。

值得注意的是，不同的哈希算法输出的哈希值是不同的（在 OpenCV 中），这里是指数据类型和位数并不完全相同，结果越复杂需要的计算成本也就越高。

下面运用这些哈希算法对某张图分别计算其哈希值，观察他们的输出结果：

![](/images/uncate/image-hash-result.jpeg)

从上图中可以看到，ColorMomentHash 比较特别，输出的是浮点数，它也是唯一一个能够对抗旋转的哈希算法，但是也局限于 -90 ~ 90 度。

图像的哈希值提取出来了，那么下一个问题来了，如何比较两张图片的相似性？


## Hamming distance

Hamming distance 汉明距离，指的是两个等长字符串对应位置不同字符的个数。

例如：

```
1  0  1  1  1  0  1
1  0  0  1  0  0  1
```

汉明距离为 2 。

两张图片之间的相似性可以通过他们的哈希值之间的汉明距离来判断，汉明距离越小则说明图片越相似，ColorMomentHash 除外。

如果我们的图片在百万以上量级，那么我们如何在实际工程应用中快速找到相似的图片？难点在于提取了所有图片构建哈希数据集后如何存储，其次如何进行百万次比较也就是计算汉明距离。

答案是构建倒排索引，例如 Elasticsearch 可以轻松实现。但是 ES 并不直接支持计算汉明距离，妄图利用模糊查询你会死的很惨，这里必须变通处理。再回到汉明距离的定义上，假设我们的图片哈希值是 64 bit 位的数据，如果按照定义则需要比较 64 次，但是我们完全可以将哈希值拆分，64 = 8 x 8，每 8 bit 构成一个比较单元，这样我们就只需要比较 8 次即可。为什么能拆分？因为我们认为相似图片即使经过拆分后比较仍然具有较好的匹配性。

显然哈希值越复杂则比较的成本越高，所以在实际应用中我们需要综合业务需求来考量具体采用哪种哈希算法。

图像哈希的方式其实可以理解为图像整体上的相似性。既然有整体，那么就有局部。


## 图像特征

「一双丹凤双角眼，两弯柳叶吊梢眉」，人脸可以有特征，那么图像呢？当然也有，只要图像具有类似的特征，那么就可以认为他们是相似的，这也就是局部相似性：

![](/images/uncate/image3-1.jpeg)

例如上面左右两张图，特征匹配，局部相似。

什么是特征？特征一定是图片的低频部分。

![](/images/uncate/image3-2.png)

上图三个部分，显然蓝色圈能匹配更多，黑色圈次之，红色圈最不易匹配，如果要选择一个作为特征，当然就是红色圈。

Corner Detection : 图像特征提取的基础算法，目的在于提取图像中的 corner ，这里的 corner 可并不是四个边框角，而是图像中的具有突变特征的点，例如：

![](/images/uncate/image3-3.jpeg)

Corner detectors 最大的缺点在于无法应对伸缩情况，为了解决这个问题 SIFT 特征提取算法问世，SIFT 的全称即 Scale Invariant Feature Transform 。

Keypoint 和 Descriptor ：keypoint 也就是图像的特征点，descriptor 则是对应特征点的描述因子，在 OpenCV 中，keypoint 也一组浮点数矩阵，这并不利于计算，于是可以将其转换为了整形值也就是 descriptor ，每一个特征点的 descriptor 描述因子就是一个多维向量。

SIFT 提取特征点示例：

![](/images/uncate/image-sift.jpeg)

需要注意的是一张图像的特征点是有多个的。

SIFT 算法的缺点在于计算速度太慢，SIFT 每个特征点的 descriptor 有 128 维。为此 SURF（ Speeded-Up Robust Features ）算法对其进行了加速优化，SURF 特征点可以是 64 维，也可以转换为 128 维。

SIFT 和 SURF 算法都是有专利的，这意味着你有责任和义务向其付费，然而 OpenCV 团队经过自己的研究提出了一个更快速优秀且免费的 ORB （ Oriented FAST and Rotated BRIEF ）算法，每个特征点更只有 32 维，减少了更多计算成本。

特征点提取出来了，怎么通过特征点去比较图像的相似性？两个特征点之间的汉明距离小于一定程度，则我们认为这两个特征点是匹配的，每张图像可以提取出多个特征点，匹配的特征点的个数达到我们设定的阈值，则我们就可以认为这两张图片是相似的。

## 结语

相同图像像素级别完全相同，相似图片则分为两级，图像哈希对应整体相似，图像特征对应局部相似。
