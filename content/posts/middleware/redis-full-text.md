+++
draft = false
date = 2024-07-15T22:11:09+08:00
title = "Redis 全文检索及使用示例"
description = "Redis 全文检索及使用示例"
slug = ""
authors = []
tags = ["Redis"]
categories = ["Middleware"]
externalLink = ""
series = []
disableComments = true
+++

## 序言

`Redis` 除了我们所熟知的缓存功能之外，还通过 `RedisJSON`、`RediSearch`、`RedisTimeSeries`、`RedisBloom` 等模块支持了 JSON 数据、查询与搜索（包括全文检索、向量搜索、GEO 地理位置等）、时序数据、概率计算等等扩展功能。这些模块既可以按需导入，也被全部打包到了 `Redis Stack` 中方便我们直接使用。

本文将会简述如何使用 Redis 进行全文检索。

## Redis 全文检索

### 全文检索

全文检索是一种文本检索技术，其根据用户输入的词语或句子，在大量的文档数据中快速找到相关的内容。

全文检索的核心概念包括：
- 分词：将文档（文本内容）拆分为一个个独立的词。
- 倒排索引：一种索引类型，将词与文档进行关联，以便后续查询。
- 相关度评分：对搜索结果的相关性进行评分。

### 使用示例

本文将会使用一个公开的电影数据集，构建一个电影搜索系统。

#### 数据集

数据格式如下图所示：

![](https://raw.githubusercontent.com/RifeWang/images/master/redis/movie-dataset.png)

为了行文方便，本文只会使用以下几个字段：
- _id：唯一标识
- title：电影标题
- directors：导演
- genres：电影类型
- summary：内容摘要
- rating：评分

我们使用 Redis 的 JSON 格式存储数，导入数据使用的是 `JSON.SET` 命令：
```
JSON.SET movieID:1 $ '{"directors":"马丁·里特","genres":["剧情","动作","西部"],"rating":8.0,"title":"野狼 Hombre","summary":"约翰·罗塞尔自幼是老罗塞尔先生从战俘中带回来并抚养他长大的，但是他生性豪放不羁……"}'
```

需要说明的是，Redis 是一个 key-value 数据库，JSON 只是 value 的格式之一，而 key 总是一个字符串，key 在本文中定义为了 movieID:12345 这种固定前缀加 ID 的格式。

使用 Go 批量导入的部分代码如下：
```go
func BuildDataset() {
	movies, _ := ReadMovieJSON()

	rds := getRedisClient()
	ctx := context.Background()
	for _, v := range movies {
		b, _ := json.Marshal(v)
		if r := rds.JSONSet(ctx, "movieID:"+v.ID, "$", b); r.Err() != nil {
			panic(r.Err())
		}
	}
}
```

#### 构建索引

为了进行全文检索，我们必须要使用 `FT.CREATE` 构建索引：

```
FT.CREATE movies ON JSON
PREFIX 1 movieID:
LANGUAGE Chinese
SCHEMA
	$.title as title TEXT WEIGHT 3
	$.directors.*.name as directors TAG
	$.genres.* as genres TAG
	$.summary as summary TEXT
	$.rating.average as rating NUMERIC
```

这个命令的意思是：
- 我们基于 JSON 数据创建了一个名为 movies 的索引
- 该索引作用于前缀为 `movieID:` 的所有 key
- 使用中文分词
- 索引有以下字段：
    - title: 类型为 `TEXT`，权重为 3
    - directors: 类型为 `TAG`
    - genres: 类型为 `TAG`
    - summary: 类型为 `TEXT`
    - rating: 类型为 `NUMERIC`

索引是独立存在的，删除索引不会影响原始 key-value 数据。
在创建完索引之后，新增或修改的文档会同步构建索引，而对于创建索引之前已有的文档则会在后台异步构建索引。

#### 使用全文检索

##### 检索基础

- 全文检索（任何字段包含爱情）:
```sh
FT.SEARCH movies '爱情'
```

- RETURN 返回指定字段:
```sh
FT.SEARCH movies '爱情'
    RETURN 2 title directors
```

- HIGHLIGHT 高亮:
```sh
FT.SEARCH movies '爱情'
    RETURN 2 title directors
    HIGHLIGHT FIELDS 1 title TAGS <span> </span>
```

- SORTBY 指定字段排序:
```sh
FT.SEARCH movies '爱情'
    RETURN 3 title directors rating
    SORTBY rating DESC
```

- LIMIT offset num 分页:
```sh
FT.SEARCH movies '爱情'
    RETURN 3 title directors rating
    SORTBY rating DESC
    LIMIT 0 10
```

- TEXT 指定字段全文检索（电影标题含有爱情）:
```sh
FT.SEARCH movies '@title:爱情'
    RETURN 2 title directors
```

- Tag 字段匹配（导演是马丁·里特）:
```sh
FT.SEARCH movies '@directors:{马丁·里特}'
    RETURN 2 title directors
```

##### 多条件组合

- OR（类型是剧情或者动作）:
```sh
FT.SEARCH movies '@genres:{剧情|动作}'
    RETURN 2 title directors
```

- AND（类型是剧情或者动作且评分大于等于8.0）:
```sh
FT.SEARCH movies '(@genres:{剧情|动作})(@rating:[8.0,+inf])'
    RETURN 3 title directors rating
```

##### 前缀后缀、模糊搜索

```sh
FT.SEARCH movies '@title:爱*' RETURN 1 title
FT.SEARCH movies '@title:*情' RETURN 1 title
FT.SEARCH movies '@title:*命*' RETURN 1 title
FT.SEARCH movies '@title:%人生%' RETURN 1 title
```


##### 自定义分词

与 Elasticsearch 对比，Redis 中的自定义分词这块支持比较有限，主要是：
- 停用词：`FT.CREATE` 命令中的可选参数 `STOPWORDS`，将会影响分词
- 同义词：`FT.SYNUPDATE` 命令
    - 构造同义词：`FT.SYNUPDATE movies group1 爱情 凌虚`
	- 那么 `FT.SEARCH movies '爱情'`
    - 等价于 `FT.SEARCH movies '凌虚'`

##### 自定义打分

Redis 只是提供了可选几种不同的打分算法：
- `TFIDF`（默认使用）
- `TFIDF.DOCNORM`
- `BM25`（Elasticsearch 使用的打分算法）
- `DISMAX`
- `DOCSCORE`
- `HAMMING`

```sh
FT.SEARCH movies '爱情' RETURN 0
    WITHSCORES
    SCORER BM25
```

如果你想要其它的自定义打分，则只能通过编写扩展的方式实现了，扩展必须用 C 语言或者与 C 有接口的编程语言来写。

##### 索引别名

为底层索引创建一个索引别名，在搜索时则使用索引别名，如果数据需要重建索引，那么只需要将索引别名指向新的底层索引即可，这种情况下搜索端不会受到任何影响。

- 创建索引别名：
```sh
FT.ALIASADD aliasName movies
```

- 使用索引别名进行搜索：
```sh
FT.SEARCH aliasName '爱情' RETURN 0
```

- 更新索引别名：
```sh
FT.ALIASUPDATE aliasName anotherIndex
```

- 删除索引别名：
```sh
FT.ALIASDEL aliasName
```

#### Go 示例代码

使用的是 go-redis 库:
```go
func cmdStringToArgs(rdscmd string) (result []interface{}) {
	re := regexp.MustCompile(`\s+`)
	slice := re.Split(rdscmd, -1)
	for _, v := range slice {
		if v != "" && v != " " {
			result = append(result, v)
		}

	}
	return
}

func ExcuteCommand(rdscmd string) {
	rds := getRedisClient()
	ctx := context.Background()

	args := cmdStringToArgs(rdscmd)

	res, err := rds.Do(ctx, args...).Result()
	if err != nil {
		panic(err)
	}
	fmt.Println("RESULT:", res)
}
```

测试代码：
```go
func TestExcuteCommand(t *testing.T) {
	cases := []string{
		// 全文检索（任何字段包含爱情）:
		`FT.SEARCH movies '爱情'`,

		// RETURN 返回指定字段:
		`FT.SEARCH movies '爱情' RETURN 2 title directors`,

		// ......
	}
	for _, v := range cases {
		ExcuteCommand(v)
	}
}
```

## 总结

相较于 `Elasticsearch` 这个全文搜索领域的榜一大哥，`Redis` 支持的功能特性比较少（例如自定义分词和打分），但是基本的全文检索功能也都具备了。

笔者曾见过只有几十万数据却整了三台 `Elasticsearch` 集群的情况，这实在是大炮打蚊子、严重浪费资源。如果数据体量比较小，而且检索的使用场景也比较简单，那么使用 `Redis` 不仅足够，在性能方面还能有更大的优势。

---

参考资料：

- *https://redis.io/docs/latest/develop/interact/search-and-query/*
- *https://redis.io/docs/latest/commands/ft.create/*
- *https://redis.io/docs/latest/commands/ft.search/*