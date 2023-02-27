# Elasticsearch 入门指南


### 引言

Elasticsearch 是什么？一个开源的可扩展、高可用、分布式的全文搜索引擎。

你为什么需要它？《人生一串》中有这样一段话：

> 没了烟火气，人生就是一段孤独的旅程。

而我们如何通过烟火气、人生或者旅程等这样的关键词来搜索出这部纪录片呢？显然无论是传统的关系型数据库，还是 NOSQL 数据库都无法实现这样的需求，而这里 Elasticsearch 就派上了用场。

再来理解全文搜索是什么？举例来说，就是将上面那段话按照语义拆分成不同的词组并记录其出现的频率（专业术语叫构建倒排索引），这样当你输入一个简单的关键词就能将其搜索出来。

总而言之，Elasticsearch 就是为搜索而生。

## 一、基本概念

1. Near Realtime（近实时）

Elasticsearch 是一个近实时的搜索平台。为什么是近实时？在传统的数据库中一旦我们插入了某条数据，则立刻可以搜索到它，这就是实时。反之在 Elasticsearch 中为某条数据构建了索引（插入数据的意思）之后，并不能立刻就搜索到，因为它在底层需要进行构建倒排索引、将数据同步到副本等等一系列操作，所以是近实时（通常一秒以内，无需过于担心）。


2. Cluster（集群）& Node（节点）

每一个单一的 Elasticsearch 服务器称之为一个 Node 节点，而一个或多个 Node 节点则组成了 Cluster 集群。Cluster 和 Node 一定是同时存在的，换句话说我们至少拥有一个由单一节点构成的集群，而在实际对外提供索引和搜索服务时，我们应该将 Cluster 集群视为一个基本单元。

Cluster 集群默认的名称就是 elasticsearch ，而 Node 节点默认的名称是一个随机的 UUID ，我们只要将不同 Node 节点的 cluster name 设置为同一个名称便构成了一个集群（不论这些节点是否在同一台服务器上，只要网络有效可达，Elasticsearch 本身会自己去搜索并发现这些节点并构成集群）。

3. Index（索引）& Type（类型）& Document（文档）

Document（文档）是最基本的数据单元，我们可以将其理解为 mysql 中的具体的某一行数据。

Type（类型）在 6.0 版本之后被移除，它是一个逻辑分类，我们可以将其理解为 mysql 中的某一张表。

Index（索引）是具有类似特征的 Document 文档的集合，我们可以将其理解为 mysql 中的某一个数据库。


4. Shards（分片）& Replicas（副本）

为了更有效的存储庞大体量的数据，Elasticsearch 有了 shard 分片的存在，在对数据进行存储便会将其分散到不同的 shard 分片中，这就如同在使用 mysql 时，如果一张表的数据量过于庞大时，我们将其水平拆分为多张表一样的道理。然而 shard 的分布方式以及如何将不同分片的文档聚合回搜索请求都是由 Elasticsearch 本身来完成，这些对用户而言是无感的。同时分片的数量一旦设置则在索引创建后便无法修改，默认为五个分片。

对于副本，则是为了防止数据丢失、实现高可用，同时副本也是可以进行查询的，所以也有助于提高吞吐量。副本与分片一一对应，副本的数量可以随时调整，默认设置为每一个主分片有一个副本分片。副本分片和主分片一定不会被分配在同一个节点中，所以对于单节点集群而言，副本分片是无效的。

---
## 二、Mapping

Mapping （映射）在 ES 中的作用至关重要，数据结构、存储和索引规则等等都是通过 mapping 来进行设置的。

### Dynamic Mapping（动态映射)

在使用传统关系型数据库如 mysql 时，如果不事先明确定义数据结构是无法进行数据操作的，但是在 ES 中不需要这样，因为 ES 本身会自己去检测数据并给出其数据类型然后进行索引或存储。所以称之为动态映射。

数据类型的判断及定义规则如下：

![](/images/es/dynamic-mapping.jpeg)

然而，仅仅依赖于 ES 自身去判断并定义数据类型显然是比较受限的，我们仍然需要对数据类型进行密切关注。

需要注意的是，虽然 mapping 映射是动态的，但这并不意味着我们可以随意的修改它，对于已经存在的 field mapping（字段映射）是无法直接修改的，只能重新索引（reindex），所以我们需要对 mapping 有一个深入的了解。

### Field datatypes（字段数据类型）

ES 中的 filed（字段）如同 mysql 表中的列一样，其数据类型也有很多种：

![](/images/es/field-datatypes.jpeg)


### Meta-fields（元字段）

每一个 document 都有一些与之关联的元数据：

![](/images/es/meta-fields.jpeg)


### Mapping parameters（映射参数）

设置 mapping 时的各种参数及其含义：

![](/images/es/mapping-parameters-1.jpeg)
![](/images/es/mapping-parameters-2.jpeg)


### Dynamic templates（动态模板）

应用于动态添加字段时设置自定义 mapping 映射，通过在模板中设置匹配及映射的规则，匹配命中则会被设置为对应的 mapping ，匹配参数设置如下：

![](/images/es/dynamic-templates.jpeg)

Mapping 的设置其实是一个不断循环改进的过程，同时其与具体业务又有着密切的联系。理解了 Mapping 更有助于理解数据在 ES 中的搜索行为表现。


---

在 ES 中，全文搜索与 Analysis 部分密不可分。我们为什么能够通过一个简单的词条就搜索到整个文本？因为 Analyzer 分析器的存在，其作用简而言之就是把整个文本按照某个规则拆分成一个一个独立的字或词，然后基于此建立倒排索引。

---
## 三、Analyzer

Analyzer（分析器）的作用前文已经说过了：拆分文本。

每一个 Analyzer 都由三个基础等级的构建块组成：
- Character filters
- Tokenizer
- Token filters

1、Character filters ：接受原始输入文本，将其转换为字符流并按照指定规则基于字符进行增删改操作，最后输出字符流。

2、Tokenizer ：接受字符流作为输入，将其按照指定规则拆分为单独的     tokens（ 这里的 token 就是我们通常理解的字或者词 ），最后输出 tokens 流。

3、Token filters ：接受 tokens 流作为输入，按照指定规则基于 token 进行增删改操作，最后的输出也是 tokens 流。

一个完整的包含以上三个部分的分析流程如下图所示：

![](/images/es/analyzer.jpeg)

注意：并不是每一个 Analyzer 分析器都需要同时具备以上三种基础构建块。

一个 Analyzer 分析器的组成有：
- 零个或多个 Character filters
- 必须且只能有一个 Tokenizer
- 零个或多个 Token filters

### Character filter

Character filter 的作用就是对字符进行处理，比如移除 HTML 中的元素如 <b> ，指定某个具体的字符进行替换 abc => 123 ，或者使用正则的方式替换掉匹配的部分。

ES 内置了以下三种 Character filters ：

![](/images/es/character-filter.jpeg)


### Tokenizer

Tokenizer  的作用就是按照某个规则将文本字符流拆分成独立的 token（字词）。

word、letter、character 的区别：
```
word：我们通常理解的字或者词。
letter：指英语里的那 26 个字母。
character：指 letter 加上其它各种标点符号。
```


token 和 term 的区别（参考Lucene）：
```
token：在文本分词的过程中产生的对象，其不仅包含了分词对象的词语内容，还包含了其在文本中的开始和结束位置，以及这个词语的类型（是关键词还是停用词之类的）。
term：指文本中的某一个词语内容，以及其所在的 field 域。
```

然而，在某些语境下，其实 token 和 term 更关注的仅仅只是词语内容本身。


ES 内置了十五种 Tokenizer ，并划分为三类：

1、面向字词：

![](/images/es/tokenizer-1.jpeg)

2、以字词的某部分为粒度：

![](/images/es/tokenizer-2.jpeg)

3、结构化文本：

![](/images/es/tokenizer-3.jpeg)


### Token Filter

Token Filter 的作用就是把 Tokenizer 处理完生成的 token 流进行增删改再处理。

ES 内置的 token filter 数量多达四五十种：

![](/images/es/token-filter.jpeg)

上图只是简单罗列说明，此处不进行展开说明，更多细节还是查阅官方文档好了。


### Analyzer

ES 内置了以下 Analyzer ：

![](/images/es/analyzer-built-in.jpeg)

可以看到每一个 Analyzer 都紧紧围绕 Character filters 、Tokenizer、Token filters 三个部分。

同样，只要选择并组合自己需要的以上这三个基本部分就可以简单的进行自定义 Analyzer 分析器了。

本节简单介绍了与全文搜索密切相关的【分析】这一重要部分，而如何进行实际的分析器设置则与 Mapping 相关联，另外除了 ES 内置的之外，还有很多开源的分析器同样值得使用，比如中文分词，使用较多的就是 IK 分词。


---

## 四、Query DSL

对于 ES，当我们了解了 mapping 和 analysis 的相关内容之后，使用者更关心的问题往往是如何构建查询语句从而搜索到自己想要的数据。因此，本节将会介绍 Query DSL 的相关内容。

Query DSL 是什么？

Query Domain Specific Language ，特定领域查询语言。首先它的作用是查询，其次其语法格式只能作用于 ES 中，所以就成了所谓的特定领域。


Query DSL 可分为两种类型：
1. Leaf query clauses
       简单查询子句，查询特定 field 字段中的特定值。

2. Compound query clauses
       复合查询子句，由多个简单查询子句或复合查询子句以逻辑方式组合而成。


### Query and filter context

查询语句的行为依赖于其上下文环境是 query context 还是 filter context 。

1. Filter context :
       某个 document 文档是否匹配查询语句，答案只有是和否。对于 filter 查询 ES 会自动进行缓存处理，因此查询效率非常高，应尽可能多的使用。

2. Query context :
       除了文档是否匹配之外，还会计算其匹配程度，以 _score 表示。例如某个文档被 analyzer 解析成了十个 terms，而查询语句匹配了其中的七个 terms，那么匹配程度 _score 就是 0.7 。

### Match All Query

最简单的查询。
- match_all : 匹配所有文档。
- match_none : 不匹配任何文档。


### Full text queries

全文查询，在执行之前会先分析进行查询的字符串，而查询的行为也与 analyzer 息息相关。


位于这一组内的查询包括：

01. match

全文查询中的标准查询，包括模糊匹配和短语或邻近查询。

02. match_phrase

类似于 match ，但用于匹配精确短语或单词邻近匹配。

03. match_phrase_prefix

类似于 match_phrase，但是进行单词尾部通配符搜索。

04. multi_match

match 的 multi-fields 多字段版本。

05. common terms

优先考虑不常见单词的更专业的查询。例如英文中的 the 是一个常见的高频单词，若直接查询会匹配到大量文档且浪费性能，但是某些时候又无法直接将其忽略，这时候就用到了 common terms query ，其原理是先匹配低频单词，然后在此匹配结果上再去匹配 the 这种高频单词。

06. query_string

支持 Lucene 查询字符串语法，对 Lucene 比较熟悉的可以玩玩，但一般不需要用到。

07. simple_query_string

query_string 的简易版本。


### Term level queries

term 是倒排索引中的基本单元，term-level 级别的查询也是直接操作精确的存储在倒排索引上的 terms 。通常用于结构化数据查询，如数字、日期、枚举，而不是全文字段。

查询包括：

01. term

精确匹配某个 term 。

02. terms

匹配多个 terms 中的任意一个。

03. terms_set

版本 6.1 才加入的查询。匹配一个或多个 terms，minimum should match 指定至少需要匹配的个数。

04. range

范围查询。

05. exists

存在与否。判断依据是 non-null 非空值。若要查询不存在，则可以使用 must_not 加 exists 。

06. prefix

字段头部确定，尾部模糊匹配。

07. wildcard

通配符模糊匹配。符号 ？匹配一个字符，符号 * 匹配任意字符。

08. regexp

正则匹配。

09. fuzzy

模糊相似。模糊度是以 Levenshtein edit distance 来衡量，可以理解为为了使两个字符串相等需要更改的字符的数量。

10. type

指定 type 。

11. ids

指定 type 和文档 ids 。


### Compound queries

复合查询由其它复合查询或简单查询组成，其要么组合他们的查询结果和匹配分数，更改查询行为，要么从 query 切换到 filter context 。

查询包括：

01. constant_score

包裹 query 查询，但在 filter context 中执行，所有匹配到的文档都被给与一个相同的 _score 分数。

02. bool

组合多个查询，如 must 、should、must_not、filter 语句。must 和 should 有 scores 分数的整合，越匹配分数越高，must_not 和 filter 子句执行于 filter context 中。

03. dis_max

匹配多个查询子句中的任意一个，与 bool 从所有匹配的查询中整合匹配分数不同的是，dis_max 只会选取一个最匹配的查询中的分数。

04. function_score

使用特定函数修改主查询返回的匹配分数。

05. boosting

匹配正相关的查询，同时降低负相关查询的匹配分数。


### Joining queries

在 ES 这种分布式系统中执行完整 SQL 风格的 join 连接的代价是非常昂贵的，而作为替代并有利于水平扩展 ES 提供了以下两种方式：

01. nested

针对包含有 nested 类型的 fields 字段的文档，这些 nested 字段被用于索引对象数组，而其中的每个对象都可以被当做一个独立的文档以供查询。

02. has_child、has_parent

join 连接关系可能存在于同一个索引中不同 document 文档之间。
has_child 查询返回 child 子文档匹配的 parent 父文档。
has_parent 查询返回 parent 父文档匹配的 child 子文档。

03. parent Id

直接指定父文档的 ID 进行查询。


### Geo queries

ES 提供了两种类型的 geo 地理数据：
- geo_point：lat / lon 纬度/经度对。
- geo_shape：地理区间，包括 points 点组、lines 线、circles 圆形区域、polygons 多边形区域、multi-polygons 复合多边形区域。

查询包括：

01. geo_shape

查询指定的地理区间。要么相交、要么包含、要么不相交。查的是 geo_shape 。

02. geo_bounding_box

查询指定矩形地理区间内的坐标点。查的是 geo_points 。

03. geo_distance

查询距离某个中心点指定范围内点，也就是一个圆形区间。查的是 geo_points 。

04. geo_polygon

查询指定多边形区间内的点。查的是 geo_points 。


### Specialized queries

未包含于其它查询组内的查询：

01. more_like_this

相似于指定的 text 文本、document 文档、或 documents 文档集。

这里的相似，不仅仅是指 term 的具体内容，同时也要考量其位置因素。查询字段必须先存储 term_vector 也就是 term 向量。

02. script

接受一个 script 作为一个 filter 。

03. percolate

通常情况下，我们通过 query 语句去查询具体的文档，但是 percolate 正好相反，它是通过文档去查询 query 语句（ query 必须先注册到 percolate 中）。

percolate 一般常用于数据分类、数据路由、事件监控和预警。

04. wrapper

接受 json 或 yaml 字符串进行查询，需要 base64 编码。


### Span queries

更加底层的查询，对 term 的顺序和接近度有更加严格的要求，常用于法律或专利文件等。

除了 span_multi 之外，其它的 span 查询不能与非 span 查询混合使用。

此类所有查询在 Lucene 中都有对应的查询。

01. span_term

与 term query 相同，但用于其它 span queries 中，因为不能混合使用的原因才有的这个 span 环境特定的查询。

02. span_multi

包裹 term、range、prefix、wildcard、regexp、fuzzy 查询，以在 span 环境下使用。对应于 Lucene 中的 SpanTermQuery 。

03. span_first

相对于起始位置的偏移距离。对应于 Lucene 中的 SpanFirstQuery 。

04. span_near

匹配必须在多个 span_term 的指定距离内，通常用于检索某些相邻的单词。对应于 Lucene 中的 SpanNearQuery 。

05. span_or

匹配多个 span queries 中的任意一个。对应于 Lucene 中的 SpanOrQuery 。

06. span_not

不匹配，也就是排除。对应于 Lucene 中的 SpanNotQuery 。

07. span_containing

指定多个 span queries 中的匹配优先级。对应于 Lucene 中的 SpanContainingQuery 。

08. span_within

与 span_containing 类似，但对应于 Lucene 中的 SpanWithinQuery 。

09. field_masking_span

对不同的 fields 字段执行 span-near 或 span-or 查询。

Query DSL 部分的内容大概就是这么多，本文只是让你对于查询部分有一个整体的大概的印象，至于某个具体查询的详细细节还请查阅官方文档。

---

## 五、优化

ES 的默认配置已经提供了良好的开箱即用的体验，但是仍有一些优化手段去继续提升它的使用性能。


### General recommendations

通用建议。

01. Don't return large result sets

不要返回大量的结果集。ES 是一个搜索引擎，擅长于返回匹配度较高的几个文档（默认 10 个，取决于 size 参数），而不擅长于数据库领域的工作，例如返回一个查询条件匹配的所有文档，如果你一定要实现这个功能，建议使用 scroll API。

这个问题其实是与深度分页相关联的，ES 中的配置项 index.max_result_window 默认是 10000 ，这就是说最多只支持返回前一万条数据，如果想返回更多的数据，一方面可以增大此配置项，另一方面就是使用 scroll API ，scroll API 的原理就是记录上一次的结果标记，基于此标记再继续往下查询。


02. Avoid large documents

避免大文档。配置项 http.max_content_length 默认是 100 MB，ES 将会拒绝索引超过此大小的文档，你也可以提高这项配置，但是最大不得超过 2 GB，因为 Lucene 的限制为 2 GB。

大文档会给网络、内存、磁盘、文件系统缓存等带来更大的压力。

为了解决这个问题，我们需要重新考虑信息的基本单元，例如想要去索引一本书的内容，这并不意味着我们要把整本书都塞进一个文档中去，按照章节或者段落去划分文档显然是更好的选择。


### Recipes

解决一些常见问题的方式。

01. Mixing exact search with stemming

精确搜索混合词干搜索。

在英文场景下，词干搜索如 skiing 将会匹配包含有 ski 或 skis 的文档，但是如果用户想要实现 skiing 的精确匹配呢？最典型的解决方法就是将同样的内容索引为 multi-field 多个不同的字段，这样就能在不同的字段上分别使用词干搜索和精确搜索了。

除此之外，query_string 和 simple_query_string 的 quote_field_suffix 也可以解决这种问题。


02. Getting consistent scoring

1、Scores are not reproducible

即使同样的查询同时执行两次，文档的匹配分数也并不一致。这是因为副本存在的原因，副本的配置项是 index.number_of_replicas ，ES 进行查询时会以 round-robin 的方式轮询到不同的 shard 分片，而删除或更新文档时（在 ES 中，更新分为两步，第一步标记旧文档为删除，第二步写入新文档），旧文档并不会立刻被删除，而是等待下一个 refresh 周期此文档从属的 segment （shard 分片会被分割为多个 segment）被合并，有时候主分片刚刚完成合并操作并移除了大量标记为删除的文档，而从分片还未来得及同步此项操作，这就导致了主从索引统计信息的不同，也就影响到了匹配分数的不同。

解决方法是在查询时使用 preference 参数，此参数决定了将查询路由到哪个分片中去执行，只要 preference 一致则一定会使用相同的分片。例如你可以使用用户ID 或者 session id 作为 preference ，这样就能保证同一个用户或者同一个会话查询的一致性。


2、Relevancy looks wrong

如果你注意到两个相同内容文档的分数不同或者精确匹配的未排序在第一位，这也可能与分片有关。默认情况下，每个分片各自评分，文档也会被均匀的路由到不同的分片中，分片中的索引统计信息也会是相似的，评分将按照预期工作，但是如果你进行了下列操作之一，那么很有可能搜索请求涉及到的分片没有类似的索引统计信息，相关性可能很差：

- use routing at index time (索引时自定义路由规则导致分片不均匀)
- query multiple indices (查询跨越了多个索引)
- have too little data in your index (数据量少得可怜)

如果你的数据集很小，那么最简单的方法就是只使用一个分片（ index.number_of_shards : 1 ）。

其余情况建议的方式是使用 dfs_query_then_fetch 搜索类型，这种方式将会查询所有关联分片的索引统计信息然后合并，这样评分时使用的就是全局的索引统计信息而不是某个分片的，显然这样增加了额外的成本，然而大多数情况下，这些额外成本是很低廉的，但是如果查询中包含有大量的 fields/terms 或 fuzzy 模糊查询，增加的额外成本可能并不低。


### Tune for indexing speed

加速构建索引。

01. Use bulk requests

尽量使用 bulk 请求。

02. Use multiple workers/threads to send data to ES

其实就是提高客户端的并发数。

03. Increase the refresh interval

配置项 index.refresh_interval 默认是 1s ，这个时间指的是创建新 segment 合并旧 segment 的周期，增加此间隔时间有助于减轻 segment 合并的压力。

04. Disable refresh and replicas for initial loads

禁用 refresh 和备份可以提升不少的索引构建速度，但是正常情况下 refresh 和备份都是必须的，所以一般只在初始化导入数据如重建索引等特殊情况才使用。配置项为 index.refresh_interval : -1 和 index_number_of_repicas : 0 。

05. Disable swapping

禁用宿主机操作系统的 swap 。

06. Give memory to the filesystem cache

将宿主机至少一半的内存分配给 filesystem cache 文件系统缓存。

07. Use auto-generated ids

使用用户自定义的文档 id ，ES 将会检查其是否冲突，而使用 ES 自动生成的 id 则会跳过此步骤。

08. Use faster hardware

使用更好的硬件。

09. Indexing buffer size

确保 indices.memory.index_buffer_size 足够大，能为每个分片提供最大 512 MB 的索引缓冲区，超过这个值也不会有更高的性能。默认是 10%，即 JVM 有 10 GB 内存，那么 1 GB 将会用于索引缓存。

10. Disable _field_names

在 mapping 设置中禁用 _field_names ，但会导致 exists 查询无法使用。

11. Additional optimizations

其余一些额外的优化项与下文中的 Tune for disk usage 优化磁盘使用相关联。


### Tune for search speed

加速搜索。

01. Give memory to the filesystem cache

给 filesystem cache 分配更多内存。

02. Use faster hardware

使用更好的硬件。

03. Document modeling

文档模块化，避免 join 操作，nested 和 parent-child 关联查询都会比较慢。

04. Search as few fields as possible

在 query_string 和 multi-match 查询中，fields 越多查询越慢。你可以新增一个联合字段，在 mapping 中设置 copy_to 将多个 fields 字段自动复制到这个联合 field 字段中，这样就能把多字段查询变为单字段查询。

05. Pre-index data

预索引数据。在进行 range aggregation 范围聚合查询时，我们可以新增一个字段以在索引时标记其范围，这样 range aggregation 就变成了 term aggregation 。例如，要查询 price 在 10-100 范围内的文档数据，那么可以在构建索引时新增一个 price_range 字段标记此文档为 10-100 ，这样就可以直接根据 price_range 进行查询了。

06. Consider mapping identifiers as keyword

数字不一定要映射为数字类型字段，也可以是 keyword ，索引数字类型对于 range 查询进行了优化，而 keyword 在 term 查询时更有利。

07. Avoid scripts

避免使用 scripts，如果一定要用，优先使用 painless 和 expressions 引擎。

08. Search rounded dates

放宽日期类型的精度，由于 now 是实时变动的，因此无法缓存，而如果使用诸如 now-1h/m ，这是可以进行缓存的，相应的精度也就成了一分钟。

09. Force-merge read-only indices

强制合并只读索引为单一的 segment 更有利于搜索。使用场景常常是例如基于时间的索引，历史日期的数据不再改变，因此是只读的，而对于存在写入操作的索引不得进行此项操作。

10. Warm up global ordinals

Global ordinals 是一种数据结构，用于 keyword 字段上进行 terms aggregations，可以在 mapping 中设置 eager_global_ordinals : true 提前告诉 ES 这个字段将会用于聚合查询。

11. Warm up the filesystem cache

ES 重启后，filesystem cache 是空的，可以通过 index.store.preload 提前导入指定文件到内存中进行预热，但是如果文件系统缓存不够大，将会导致所有数据被 hold 住，一定要小心使用。

12. Use index sorting to speed up conjunctions

使用 index sorting 索引排序可以使连接更快（组织 Lucene 文档 id，使连接如 a AND b AND ... 更高效），但代价是构建索引将变慢。

13. Use preference to optimize cache utilization

缓存包括 filesystem cache、request cache、query cache 等都是基于 node 节点的，使用 preference 更够将同样的请求路由到同样的分片也就是同一个节点上，这样能够更好的利用缓存。

14. replicas might help with throughput, but not always

备份也会参与查询，这有助于提高吞吐量，但并非总是如此。

如何设置备份的数量？假设集群中有 num_nodes 个节点，num_primaries 个主分片，一次最多允许 max_failures 个节点故障，那么备份的数量应该设置为

max( max_failures, ceil( num_nodes/num_primaries ) - 1 )


15. Turn on adaptive replica selection

开启动态副本选择，ES 将会基于副本的状态动态选择以处理请求。

```
PUT /_cluster/settings
{
    "transient": {
        "cluster.routing.use_adaptive_replica_selection": true
    }
}
```

### Tune for disk usage

优化磁盘使用。

01. Disable the features you do not need

不需要构建倒排索引的字段不建索引，index: false。

text 类型字段不需要评分的可以不写入 norms，norms: false （norms 是评分因子）。text 类型字段默认也会存储频率和位置信息，频率计算分数，位置用于短语查询，不需要短语查询可以不存储位置信息，index_options: freqs ，不关心评分可以设置 index_options: freqs 的同时设置 norms: false 。


02. Don't use default dynamic string mappings

默认的动态字符串映射会将 string 字段同时索引为 text 和 keyword ，这造成了空间的浪费，明确使用其中一个即可。

03. Watch your shard size

shard 分片越大，则存储数据越高效，缺点就是恢复需要的时间更久。

04. Disable _all

禁用 _all ，此字段索引了所有的字段， v6.0.0 版本已经将其移除。

05. Disable _source

禁用 _source ，此字段存储了原始的 json 文档数据，虽然禁用可以节省磁盘空间，但是我个人并不建议这么做，因为禁用后将无法获取到此字段的内容，如 update 和 reindex 等 API 都将无法使用。

06. Use best_compression

通过 index.codec 设置压缩方式为 best_compression 。

07. Force merge

每个 shard 分片有多个 segments，segment 越大存储数据越高效。可以通过 _forcemerge API 减少每个分片的 segments 数量，通过 max_num_segments = 1 即可设置每个分片一个 segment 。

08. Shrink index

可以通过 shrink API 减少 shard 分片的数量，可以与 _forcemerge API 一起使用。

09. Use the smallest numeric type that is sufficient

使用合适的数字类型，数字类型越小占用磁盘空间越少。

10. Use index sorting to colocate similar documents

默认情况下，文档按照添加到索引的顺序进行压缩，如果启用了 index sorting 则按照索引排序顺序进行压缩，对具有相似结构、字段和值的文档进行排序可以提高压缩效率。

11. Put fields in the same order in documents

压缩是将多个文档压缩成块，如果字段始终以相同的顺序出现，则更有可能在这些 _source 文档中找到更长的重复字符串，从而压缩效率更高。

其实从实际情况来看，磁盘的成本往往是比较低廉的，我们常常更关注的是搜索和索引性能的提升。了解优化相关的部分内容有助于我们更好的理解和使用 ES。
