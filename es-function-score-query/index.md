# Elasticsearch 自定义打分 Function score query


# Elasticsearch 自定义打分 Function score query

Elasticsearch 会为 query 的每个文档计算一个相关度得分 score ，并默认按照 score 从高到低的顺序返回搜索结果。
在很多场景下，我们不仅需要搜索到匹配的结果，还需要能够按照某种方式对搜索结果重新打分排序。例如：
- 搜索具有某个关键词的文档，同时考虑到文档的时效性进行综合排序。
- 搜索某个旅游景点附近的酒店，同时根据距离远近和价格等因素综合排序。
- 搜索标题包含 elasticsearch 的文章，同时根据浏览次数和点赞数进行综合排序。


Function score query 就可以让我们实现对最终 score 的自定义打分。

----
## score 自定义打分过程

为了行文方便，本文把 ES 对 `query` 匹配的文档进行打分得到的 score 记为 `query_score` ，而最终搜索结果的 score 记为 `result_score` ，显然，一般情况下（也就是不使用自定义打分时），`result_score` 就是 `query_score` 。

那么当我们使用了自定义打分之后呢？最终结果的 score 即 `result_score` 的计算过程如下：
1. 跟原来一样执行 `query` 并且得到原来的 `query_score` 。
2. 执行设置的自定义打分函数，并为每个文档得到一个新的分数，本文记为 `func_score` 。
3. 最终结果的分数 `result_score` 等于 `query_score` 与 `func_score` 按某种方式计算的结果（默认是相乘）。


例如，搜索标题包含 elasticsearch 的文档。

不使用自定义打分，则搜索形如：
```
GET /_search
{
  "query": {
    "match": {
      "title": "elasticsearch"
    }
  }
}
```

假设我们最终得到了三个搜索结果，score 分别是 `0.3、0.2、0.1` 。


使用自定义打分，即 `function_score` ，则语法形如：
```
GET /_search
{
  "query": {
    "function_score": {
      "query": {
        "match": {
          "title": "elasticsearch"
        }
      }

      <!-- 设置自定义打分函数，这里先省略，后面再展开讲解 -->

      "boost_mode": "multiply"
    }
  }
}
```

最终搜索结果 score 的计算过程就是：
1. 执行 `query` 得到原始的分数，与上文假设对应，即 `query_score` 分别是 `0.3、0.2、0.1` 。
2. 执行自定义的打分函数，这一步会为每个文档得到一个新的分数，假设新的分数即 `func_score` 分别是 `1、3、5` 。
3. 最终结果的 score 分数即 `result_score` = `query_score` * `func_score` ，对应假设的三个搜索结果最终的 score 分别就是 `0.3 * 1 = 0.3` 、`0.2 * 3 = 0.6`、`0.1 * 5 = 0.5` ，至此我们完成了新的打分过程，而搜索结果也会按照最终的 score 降序排列。


最终的分数 `result_score` 是由 `query_score` 与 `func_score` 进行计算而来，计算方式由参数 `boost_mode` 定义：
- `multiply` : 相乘（默认），`result_score = query_score * function_score`
- `replace` : 替换，`result_score = function_score`
- `sum` : 相加，`result_score = query_score + function_score`
- `avg` : 取两者的平均值，`result_score = Avg(query_score, function_score)`
- `max` : 取两者之中的最大值，`result_score = Max(query_score, function_score)`
- `min` : 取两者之中的最小值，`result_score = Min(query_score, function_score)`


本文读到这，你应该已经对自定义打分的过程有了一个基本印象（`query` 原始分数、自定义函数得分、最终结果 score )。但是我们还有一个关键点没讲，即怎么设置自定义打分函数？


----
## function_score 打分函数

`function_score` 提供了以下几种打分的函数：
- `weight` : 加权。
- `random_score` : 随机打分。
- `field_value_factor` : 使用字段的数值参与计算分数。
- `decay_function` : 衰减函数 gauss, linear, exp 等。
- `script_score` : 自定义脚本。

### `weight`

`weight` 加权，也就是给每个文档一个权重值。

示例：
```
{
  "query": {
    "function_score": {
      "query": { "match": { "message": "elasticsearch" } },
      "weight": 5
    }
  }
}
```
例子中的 weight 是 5 ，即自定义函数得分 `func_score` = 5 ，最终结果的 score 等于 `query_score` * 5 。

当然这个示例将匹配项全部加权并不会改变搜索结果顺序，我们再看一个例子：
```
{
  "query": {
    "function_score": {
      "query": { "match": { "message": "elasticsearch" } },
      "functions": [
        {
          "filter": { "match": { "title": "elasticsearch" } },
          "weight": 5
        }
      ]
    }
  }
}
```

我们可以通过 `filter` 去限制 `weight` 的作用范围，另外我们可以在 `functions` 中同时使用多个打分函数。


### `random_score`

`random_score` 随机打分，生成 [0, 1) 之间均匀分布的随机分数值。

示例：
```
GET /_search
{
  "query": {
    "function_score": {
      "random_score": {}
    }
  }
}
```

虽然是随机值，但是有时候我们需要随机值保持一致，比如所有用户都随机产生搜索结果，但是同一个用户的随机结果前后保持一致，这时只需要为同一个用户指定相同的 `seed` 即可。

示例：
```
{
  "query": {
    "function_score": {
      "random_score": {
        "seed": 10,
        "field": "_seq_no"
      }
    }
  }
}
```

默认情况下，即不设置 `field` 时会使用 Lucene doc ids 作为随机源去生成随机值，但是这会消耗大量内存，官方建议可以设置 `field` 为 `_seq_no` ，主要注意的是，即使指定了相同的 `seed` ，随机值某些情况下也会改变，这是因为一旦字段进行了更新，`_seq_no` 也会更新，进而导致随机源发生变化。


多个函数组合示例：
```
GET /_search
{
  "query": {
    "function_score": {
      "query": { "match_all": {} },
      "boost": "5",
      "functions": [
        {
          "filter": { "match": { "test": "bar" } },
          "random_score": {},
          "weight": 23
        },
        {
          "filter": { "match": { "test": "cat" } },
          "weight": 42
        }
      ],
      "max_boost": 42,
      "score_mode": "max",
      "boost_mode": "multiply",
      "min_score": 42
    }
  }
}
```
上例 `functions` 中设置了两个打分函数：
- 一个是 `random_score` 随机打分，并且 `weight` 是 23
- 另一个只有 `weight` 是 42

假设：
- 第一个函数随机打分得到了 0.1 ，再与 `weight` 相乘就是 2.3
- 第二个函数只有 `weight` ，那么这个函数得到的分数就是 `weight` 的值 42

`score_mode` 设置为了 `max`，意思是取两个打分函数的最大值作为 `func_score`，对应上述假设也就是 2.3 和 42 两者中的最大值，即 `func_score` = 42

`boost_mode` 设置为了 `multiply`，就是把原来的 `query_score` 与 `func_score` 相乘就得到了最终的 score 分数。


参数 `score_mode` 指定多个打分函数如何组合计算出新的分数：
- `multiply` : 分数相乘（默认）
- `sum` : 相加
- `avg` : 加权平均值
- `first` : 使用第一个 filter 函数的分数
- `max` : 取最大值
- `min` : 取最小值


为了避免新的分数的数值过高，可以通过 `max_boost` 参数去设置上限。

需要注意的是：不论我们怎么自定义打分，都不会改变原始 `query` 的匹配行为，我们自定义打分，都是在原始 `query` 查询结束后，对每一个匹配的文档进行重新算分。

为了排除掉一些分数太低的结果，我们可以通过 `min_score` 参数设置最小分数阈值。


### `field_value_factor`

`field_value_factor` 使用字段的数值参与计算分数。

例如使用 `likes` 点赞数字段进行综合搜索：
```
{
  "query": {
    "function_score": {
      "query": { "match": { "message": "elasticsearch" } },
      "field_value_factor": {
        "field": "likes",
        "factor": 1.2,
        "missing": 1,
        "modifier": "log1p"
      }
    }
  }
}
```

说明：
- `field` : 参与计算的字段。
- `factor` : 乘积因子，默认为 1 ，将会与 `field` 的字段值相乘。
- `missing` : 如果 `field` 字段不存在则使用 `missing` 指定的缺省值。
- `modifier` : 计算函数，为了避免分数相差过大，用于平滑分数，可以是以下之一：
    - `none` : 不处理，默认
    - `log` : `log(factor * field_value)`
    - `log1p` : `log(1 + factor * field_value)`
    - `log2p` : `log(2 + factor * field_value)`
    - `ln` : `ln(factor * field_value)`
    - `ln1p` : `ln(1 + factor * field_value)`
    - `ln2p` : `ln(2 + factor * field_value)`
    - `square` : 平方，`(factor * field_value)^2`
    - `sqrt` : 开方，`sqrt(factor * field_value)`
    - `reciprocal` : 求倒数，`1/(factor * field_value)`

假设某个匹配的文档的点赞数是 1000 ，那么例子中其打分函数生成的分数就是 `log(1 + 1.2 * 1000)`，最终的分数是原来的 query 分数与此打分函数分数相差的结果。



### `decay_function`

`decay_function` 衰减函数，例如：
- 以某个数值作为中心点，距离多少的范围之外逐渐衰减（缩小分数）
- 以某个日期作为中心点，距离多久的范围之外逐渐衰减（缩小分数）
- 以某个地理位置点作为中心点，方圆多少距离之外逐渐衰减（缩小分数）

示例：
```
"DECAY_FUNCTION": {
    "FIELD_NAME": {
          "origin": "30, 120",
          "scale": "2km",
          "offset": "0km",
          "decay": 0.33
    }
}
```
上例的意思就是在距中心点方圆 2 公里之外，分数减少到三分之一（乘以 decay 的值 0.33）。


- `DECAY_FUNCTION` 可以是以下任意一种函数：
    - `linear` : 线性函数
    - `exp` : 指数函数
    - `gauss` : 高斯函数
- `origin` : 中心点，只能是数值、日期、geo-point
- `scale` : 定义到中心点的距离
- `offset` : 偏移量，默认 0
- `decay` : 衰减指数，默认是 0.5



示例：
```
GET /_search
{
  "query": {
    "function_score": {
      "gauss": {
        "@timestamp": {
          "origin": "2013-09-17",
          "scale": "10d",
          "offset": "5d",
          "decay": 0.5
        }
      }
    }
  }
}
```

中心点是 2013-09-17 日期，scale 是 10d 意味着日期范围是 2013-09-12 到 2013-09-22 的文档分数权重是 1 ，日期在 scale + offset = 15d 之外的文档权重是 0.5 。


如果参与计算的字段有多个值，默认选择最靠近中心点的值，也就是离中心点的最近距离，可以通过 `multi_value_mode` 设置：
- `min` : 最近距离
- `max` : 最远距离
- `avg` : 平均距离
- `sum` : 所有距离累加

示例：
```
GET /_search
{
  "query": {
    "function_score": {
      "query": {
        "match": {
          "properties": "大阳台"
        }
      },
      "functions": [
        {
          "gauss": {
            "price": {
              "origin": "0",
              "scale": "2000"
            }
          }
        },
        {
          "gauss": {
            "location": {
              "origin": "30, 120",
              "scale": "2km"
            }
          }
        }
      ],
      "score_mode": "multiply"
    }
  }
}
```

假设这是搜索大阳台的房源，上例设置了 price 价格字段的中心点是 0 ，范围 2000 以内，以及 location 地理位置字段的中心点是 "30, 120" ，方圆 2km 之内，在这个范围之外的匹配结果的 score 分数会进行高斯衰减，即打分降低。



### `script_score`

`script_score` 自定义脚本打分，如果上面的打分函数都满足不了你，你还可以直接编写脚本打分。

示例：
```
GET /_search
{
  "query": {
    "function_score": {
      "query": {
        "match": { "message": "elasticsearch" }
      },
      "script_score": {
        "script": {
          "source": "Math.log(2 + doc['my-int'].value)"
        }
      }
    }
  }
}
```

在脚本中通过 `doc['field']` 的形式去引用字段，`doc['field'].value` 就是使用字段值。

你也可以把额外的参数与脚本内容分开：
```
GET /_search
{
  "query": {
    "function_score": {
      "query": {
        "match": { "message": "elasticsearch" }
      },
      "script_score": {
        "script": {
          "params": {
            "a": 5,
            "b": 1.2
          },
          "source": "params.a / Math.pow(params.b, doc['my-int'].value)"
        }
      }
    }
  }
}
```


## 结语

通过了解 Elasticsearch 的自定义打分相信你能更好的完成符合业务的综合性搜索。
