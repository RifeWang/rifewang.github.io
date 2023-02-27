+++
draft = false
date = 2020-04-11T00:00:00+08:00
title = "以图搜图系统工程实践"
description = "以图搜图系统工程实践"
slug = ""
authors = []
tags = ["Engineering"]
categories = ["Engineering"]
externalLink = ""
series = []
disableComments = true
+++

# 以图搜图系统工程实践

之前写过一篇概述: [以图搜图系统概述](/posts/engineering/image-search-system/) 。

以图搜图系统需要解决的主要问题是：
- 提取图像特征向量（用特征向量去表示一幅图像）
- 特征向量的相似度计算（寻找内容相似的图像）

对应的工程实践，具体为：
- 卷积神经网络 `CNN` 提取图像特征
- 向量搜索引擎 `Milvus`

## CNN

使用卷积神经网路 CNN 去提取图像特征是一种主流的方案，具体的模型则可以使用 `VGG16` ，技术实现上则使用 `Keras` + `TensorFlow` ，参考 [Keras 官方示例](https://keras.io/applications/#extract-features-with-vgg16)：

```
from keras.applications.vgg16 import VGG16
from keras.preprocessing import image
from keras.applications.vgg16 import preprocess_input
import numpy as np

model = VGG16(weights='imagenet', include_top=False)

img_path = 'elephant.jpg'
img = image.load_img(img_path, target_size=(224, 224))
x = image.img_to_array(img)
x = np.expand_dims(x, axis=0)
x = preprocess_input(x)

features = model.predict(x)
```

这里提取出来的 `feature` 就是特性向量。

### 1、归一化

为了方便后续操作，我们常常会将 `feature` 进行归一化的处理：

```
from numpy import linalg as LA

norm_feat = feat[0]/LA.norm(feat[0])
```

后续实际使用的也是归一化后的 `norm_feat` 。


### 2、Image 说明

这里加载图像使用的是 `keras.preprocessing` 的 `image.load_img` 方法即：

```
from keras.preprocessing import image

img_path = 'elephant.jpg'
img = image.load_img(img_path, target_size=(224, 224))
```

实际上是 `Keras` 调用的 `TensorFlow` 的方法，详情见 [ TensorFlow 官方文档](https://www.tensorflow.org/api_docs/python/tf/keras/preprocessing/image/load_img) ，而最后得到的 `image` 对象其实是一个 [PIL Image](https://pillow.readthedocs.io/en/stable/reference/Image.html) 实例（ `TensorFlow` 使用的 `PIL` ）。


### 3、Bytes 转换

实际工程中图像内容常常是通过网络进行传输的，因此相比于从 path 路径加载图片，我们更希望直接将 bytes 数据转换为 image 对象即 [PIL Image](https://pillow.readthedocs.io/en/stable/reference/Image.html) ：

```
import io
from PIL import Image

# img_bytes: 图片内容 bytes
img = Image.open(io.BytesIO(img_bytes))
img = img.convert('RGB')

img = img.resize((224, 224), Image.NEAREST)
```

以上 `img` 与前文中的 `image.load_img` 得到的结果相同，这里需要注意的是：
 - 必须进行 `RGB` 转换
 - 必须进行 `resize` （ `load_img` 方法的第二个参数也就是 resize ）


### 4、黑边处理
有时候图像会有比较多的黑边部分（例如截屏），而这些黑边的部分即没有实际价值，又会产生比较大的干扰，因此去除黑边也是一项常见的操作。

所谓黑边，本质上就是一行或一列的像素点全部都是 `(0, 0, 0)` ( RGB 图像)，去除黑边就是找到这些行或列，然后删除，实际是一个 numpy 的 3-D Matrix 操作。

移除横向黑边示例：

```
# -*- coding: utf-8 -*-

import numpy as np
from keras.preprocessing import image


def RemoveBlackEdge(img):
    """移除图片横向黑边

    Args:
        img: PIL image 实例

    Returns:
        PIL image 实例
    """
    width = img.width
    img = image.img_to_array(img)
    img_without_black = img[~np.all(img == np.zeros((1, width, 3), np.uint8), axis=(1, 2))]
    img = image.array_to_img(img_without_black)
    return img

```

CNN 提取图像特征以及图像的其它相关处理先写这么多，我们再看向量搜索引擎。

---------

## 向量搜索引擎 Milvus

只有图像的特征向量是远远不够的，我们还需要对这些特征向量进行动态的管理（增删改），以及计算向量的相似度并返回最邻近范围内的向量数据，而开源的向量搜索引擎 [Milvus](https://milvus.io/cn/) 则很好的完成这些工作。

下文将会讲述具体的实践，以及要注意的地方。

### 1、对 CPU 有要求

想要使用 `Milvus` ，首先必须要求你的 CPU 支持 `avx2` 指令集，如何查看你的 CPU 支持哪些指令集呢？对于 Linux 系统，输入指令
```
cat /proc/cpuinfo | grep flags
```
你将会看到形如以下的内容：
```
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid dca sse4_1 sse4_2 x2apic movbe popcnt aes xsave avx f16c rdrand lahf_lm abm cpuid_fault epb invpcid_single pti intel_ppin tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust bmi1 avx2 smep bmi2 erms invpcid cqm xsaveopt cqm_llc cqm_occup_llc dtherm ida arat pln pts
```
`flags` 后面的这一大堆就是你的 CPU 支持的全部指令集，当然内容太多了，我只想看是否支持具体的某个指令集，比如 `avx2` ， 再加一个 grep 过滤一下即可：
```
cat /proc/cpuinfo | grep flags | grep avx2
```

如果执行结果没有内容输出，就是不支持这个指令集，你只能换一台满足要求的机器。


### 2、容量规划

系统设计时，容量规划是需要首先考虑的地方，我们需要存储多少数据，这些数据需要多少内存以及多大的磁盘空间？

速算，上文中特征向量的每一个维度都是 `float32` 的数据类型，一个 `float32` 需要占用 `4 byte`，那么一个 512 维的向量就需要 `2 KB` ，依次类推：
- 一千个 512 维向量需要 2 MB
- 一百万 512 维向量需要 2 GB
- 一千万 512 维向量需要 20 GB
- 一个亿 512 维向量需要 200 GB
- 十个亿 512 维向量需要 2 TB

如果我们希望能将数据全部存在内存中，那么系统就至少需要对应大小的内存容量。

这里推荐你使用官方的大小计算工具: [milvus tools](https://milvus.io/tools/sizing)

实际上我们的内存可能并没有那么大（内存不够没关系，milvus 会将数据自动刷写到磁盘上），另外除了这些原始的向量数据之外，还会有一些其他的数据例如日志等的存储也是我们需要考虑的地方。


### 3、系统配置

关于系统配置，官方文档有比较详细的说明：
- [Milvus 服务端配置](https://www.milvus.io/cn/docs/v0.7.1/reference/milvus_config.md)
- [如何设置系统配置项](https://www.milvus.io/cn/blogs/2020-2-19-milvus-config.md)
- [配置 Milvus 用于生产环境](https://www.milvus.io/cn/docs/v0.7.1/reference/performance_tuning.md)


### 4、数据库设计

#### collection & partition

在 Milvus 中，数据会按照 `collection` 和 `partition` 进行划分：
- `collection` 就是我们理解的表。
- `partition` 则是 `collection` 的分区，也就是某个表内部的分区。

`partition` 分区在底层实现上其实与 `collection` 集合是一致的，只是前者从属于后者，但是有了分区之后，数据的组织方式变得更加灵活，我们也可以指定集合中某个特定分区进行查询，从而达到一个更高的查询性能，更多内容参考 [分区表详细说明](https://www.milvus.io/cn/blogs/2019-12-16-table-partition.md) 。

我们可以使用多少个 `collection` 和 `partition` ？
由于 `collection` 和 `partition` 的基本信息都属于元数据，而 milvus 内部进行元数据管理需要使用 `SQLite`（ milvus 内部集成）或者 `MySQL` (需要外部连接) 其中之一，如果你使用默认的 `SQLite` 去管理元数据的话，当集合和分区的数量过多时，性能损耗会很严重，因此集合和分区总数不要超过 `50000` ( 0.8.0 版本将会限制为 `4096` ) ，需要设置更多的数量则建议使用外接 `MySQL` 的方式。

Milvus 的 `collection` 和 `partition` 内部支持的数据结构非常简单，只支持 `ID + vector` ，换句话说，表只有两列，一列是 ID ，一列是向量数据。

注意：
- ID 目前只支持整数类型
- 我们需要保证 ID 在 `collection` 的层面是唯一的，而不是 `partition` 。

#### 条件过滤
我们使用一些传统的数据库时，往往可以指定字段进行条件过滤，但是 Milvus 并不能直接支持这项功能，然而我们是可以通过集合和分区的设计去实现简单的条件过滤，例如，我们有很多图片数据，但是这些图片数据都明确的属于具体的用户，那么我们就可以按照用户去划分 `partition` ，这样查询的时候以用户作为过滤条件其实就是指定 `partition` 即可。


#### 结构化数据与向量的映射

由于 milvus 只支持 `ID + vector` 的数据结构，而实际业务上我们最终需要的往往是具有业务意义的结构化数据，也就是说，我们需要通过 `vector` 向量最终找到结构化数据，因此我们需要通过 `ID` 去维护结构化数据与向量之间的映射关系:
```
结构化数据 ID  <-->  映射表  <-->  Milvus ID
```

#### 索引类型选择
请参考以下文档:
- [索引类型](https://www.milvus.io/cn/docs/v0.7.1/guides/index.md)
- [如何选择索引类型](https://www.milvus.io/cn/blogs/2019-12-03-select-index.md)


### 5、搜索结果处理
Milvus 的搜索结果是 `ID + distance` 的集合:
- `ID` : `collection` 中的 `ID` 。
- `distance` : 0 ~ 1 的距离值，表示相似性程度，越小越相似。

#### 过滤 ID 为 -1 的数据
当数据集过少的时候，搜索结果可能会包含 ID 为 -1 的数据，我们需要自己去过滤掉。

#### 翻页
向量的搜索比较特别，查询的结果是按照相似性顺序，从最相似开始往后选取 `topK` 个数据（ `topK` 需要搜索时由用户指定）。

Milvus 的搜索不支持翻页，如果我们希望在业务上实现这个功能，那么只能由我们自己去处理，比如，我想要每页 10 条数据，只显示第 3 页的数据，那么我们需要去取 `topK = 30` 的数据，然后只返回最后 10 条。

#### 业务上的相似性阈值
两张图片的特征向量的距离 `distance` 范围是 0 ~ 1 ，有些时候我们需要在业务上去判定两张图片是否相似，这时就需要我们自己去设置一个距离的阈值，当 `distance` 小于阈值时就可以判定为相似，大于阈值时判定为不相似，这个也是需要根据具体的业务自己去处理。


## 结语

本文讲述了以图搜图系统进行工程实践时比较常见的内容，最后强烈推荐一下 [Milvus](https://milvus.io/cn/) 。