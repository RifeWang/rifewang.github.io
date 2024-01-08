+++
draft = false
date = 2024-01-08T10:57:38+08:00
title = "Redis Stack 不只是缓存之 RedisJSON"
description = "Redis Stack 不只是缓存之 RedisJSON"
slug = ""
authors = []
tags = ["Redis"]
categories = ["Middleware"]
externalLink = ""
series = []
disableComments = true
+++

## Redis Stack

虽然 Redis 作为一个 key-value 数据库早已被广泛应用于各种缓存相关的场景，然而其团队的却并未故步自封，他们希望更进一步为开发者提供一个不只有缓存功能的强大的实时数据平台，用于处理所有实时数据的应用场景。

为此，除了我们所熟知的核心缓存功能之外，Redis 还通过提供 `RedisJSON`、`RediSearch`、`RedisTimeSeries`、`RedisBloom` 等多个模块从而支持 JSON 数据、查询与搜索（包括全文搜索、向量搜索、GEO 地理位置等）、时序数据、概率计算等等扩展功能。

而所谓的 `Redis Stack` 就是这样一个统一了所有上述模块的集大成者（就是除了缓存功能之外，把 `RedisJSON`、`RediSearch`、`RedisTimeSeries`、`RedisBloom` 等模块都打包到了一起）。

## RedisJSON

`RedisJSON` 是 Redis 的一个模块，它用来专门处理 JSON 格式的数据。除了 `string`、`list`、`set`、`hash` ... 等核心数据类型之外，`RedisJSON` 模块将 `JSON` 也作为了一种原生的数据类型。

## JSONPath

为了更方便地访问 JSON 数据中的特定元素，可以使用 path 路径这样一种方式。目前 path 语法有两种：`JSONPath syntax`（JSONPath 语法） 和 `legacy path syntax`（传统 path 语法），本文只讲 [JSONPath](https://redis.io/docs/data-types/json/path/) 这种语法。

语法元素说明：
- `$`：JSON 数据的根路径。
- `.` 或者 `[]`：子元素。
- `..`：递归地遍历 JSON 文档。
- `*`：通配符，返回所有元素。
- `[]`：下标运算符，访问数组元素。
- `[,]`：并集，选择多个元素。
- `[start : end : step]`：数组切片，其中 start、end 是索引，step 是步长。
- `?()`：过滤 JSON 对象或数组。支持比较运算符（`==`、`!=`、`<`、`<=`、`>`、`>=`、`=~`）、逻辑运算符（`&&`、`||`）和括号（`(`, `)`）
- `()`：脚本表达式。
- `@`：当前元素，用于过滤器或脚本表达式。

示例：
```json
{
    "store":{
        "book":[
            {
                "category":"reference",
                "author":"Nigel Rees",
                "title":"Sayings of the Century",
                "price":8.95
            },
            {
                "category":"fiction",
                "author":"Evelyn Waugh",
                "title":"Sword of Honour",
                "price":12.99
            }
        ],
        "bicycle":{
            "color":"red",
            "price":19.95
        }
    }
}
```

- `$` 指向数据的根路径，返回的也就是整个数据。
- `.` 访问子元素，例如 `$.store.bicycle.color`。
- `..` 递归遍历，例如 `$.store.book..title` 获取 book 数组中的所有对象的 title 属性。
- `*`：通配符，返回所有元素，例如 `$.*` 由于第一层级只有 store 一个元素，所以这里等价于 `$.store`；`$.store.*` 则返回 store 下面所有元素的值，也就是 `$.store.book` 的值和 `$.store.bicycle` 的值。
- `[]`：下标运算符，从零开始，访问数组元素。例如 `$.store.book[1]` 返回 book 数组中的第二个元素。
- `[,]`：并集，选择多个元素，例如 `$.store.book[0,1]` 返回 book 数组的前两个元素。
- `[start : end : step]`：数组切片，例如 `$.store.book[0:1]` 返回 book 数组中的第一个元素。
- `@`：当前元素，用于过滤器或脚本表达式。
- `?()`：过滤 JSON 对象或数组，例如 `$.store.book[?(@.price>10)]` 获取 book 数组中 price > 10 的数据。


## JSON command

在 Redis Stack 中支持的 [JSON 命令](https://redis.io/commands/?group=json)：

### 通用类

- `JSON.SET`：设置值
> SET 语法：`JSON.SET key path value [NX | XX]` (NX 不存在则设置，XX 存在则设置)

- `JSON.GET`：获取值
> GET 语法：`JSON.GET key [INDENT indent] [NEWLINE newline] [SPACE space] [path [path ...]]`

- `JSON.MERGE`：合并值
> MERGE 语法：`JSON.MERGE key path value`

- `JSON.FORGET`：同 JSON.DEL
- `JSON.DEL`：删除值
> DEL 语法：`JSON.DEL key [path]`


- `JSON.CLEAR`：清空 array 或 object 类型的值并将 number 类型的值设置为 0
> CLEAR 语法：`JSON.CLEAR key [path]`

- `JSON.TYPE`：返回 JSON 值的类型。类型有：string、number、boolean、object、array、null、integer（integer 有点特殊，它并不是 JSON 标准定义的基本类型，但是给出了校验方式）
> TYPE 语法：`JSON.TYPE key [path]`

示例（以下示例均是通过 redis-cli 进行）：

```shell
> JSON.SET id:1 $ '{"a":2}'
"OK"

> JSON.SET id:1 $.b '3'
"OK"

> JSON.GET id:1 $
"[{\"a\":2,\"b\":3}]"

> JSON.GET id:1 $.a $.b
"{\"$.a\":[2],\"$.b\":[3]}"

> JSON.GET id:1 INDENT "\t" NEWLINE "\n" SPACE " " $
"[\n\t{\n\t\t\"a\": 2,\n\t\t\"b\": 3\n\t}\n]"
```


```shell
> JSON.SET id:2 $ '{"a":2}'
"OK"

> JSON.MERGE id:2 $.c '[4,5]'
"OK"

> JSON.GET id:2 $
"[{\"a\":2,\"c\":[4,5]}]"

> JSON.TYPE id:2 $.a
1) "integer"

> JSON.TYPE id:2 $.c
1) "array"
```

```shell
> JSON.SET id:3 $ '{"a":{"b": [1, 2]}, "c": "c", "d": 123}'
"OK"

> JSON.GET id:3 $
"[{\"a\":{\"b\":[1,2]},\"c\":\"c\",\"d\":123}]"

> JSON.CLEAR id:3 $.*
(integer) 2

> JSON.GET id:3 $
"[{\"a\":{},\"c\":\"c\",\"d\":0}]"

> JSON.DEL id:3 $.a
(integer) 1

> JSON.GET id:3 $
"[{\"d\":0,\"c\":\"c\"}]"

> JSON.DEL id:3
(integer) 1
```

从以上示例中可以看到，通过 JSONPath 可以只操作 JSON 中的部分值，这也意味着用户可以针对特定部分进行原子操作。

### 针对 array 数组类型

- `JSON.ARRAPPEND`：数组尾部增加元素
> ARRAPPEND 语法：`JSON.ARRAPPEND key [path] value [value ...]`

- `JSON.ARRINDEX`：数组中出现指定值的第一个 index
> ARRINDEX 语法：`JSON.ARRINDEX key path value [start [stop]]`

- `JSON.ARRINSERT`：数组指定索引处插入元素
> ARRINSERT 语法：`JSON.ARRINSERT key path index value [value ...]`

- `JSON.ARRLEN`：返回数组的长度
> ARRLEN 语法：`JSON.ARRLEN key [path]`

- `JSON.ARRPOP`：从数组的索引中删除并返回一个元素
> ARRPOP 语法：`JSON.ARRPOP key [path [index]]`

- `JSON.ARRTRIM`：修剪数组，使其仅包含指定范围的元素
> ARRTRIM 语法：`JSON.ARRTRIM key path start stop`

示例：

```shell
> JSON.SET id:4 $ '[1,2,3]'
"OK"

> JSON.ARRAPPEND id:4 $ '4' '5'
1) "5"

> JSON.GET id:4 $
"[[1,2,3,4,5]]"

> JSON.ARRINSERT id:4 $ 2 '2' '3'
1) "7"

> JSON.GET id:4 $
"[[1,2,2,3,3,4,5]]"

> JSON.ARRINDEX id:4 $ '3'
1) "3"

> JSON.ARRPOP id:4
"5"

> JSON.ARRLEN id:4
(integer) 6

> JSON.ARRTRIM id:4 $ 1 3
1) "3"

> JSON.GET id:4 $
"[[2,2,3]]"
```

### 针对 object 对象类型

- `JSON.OBJKEYS`：返回 object 中的 key 数组
- `JSON.OBJLEN`：返回 object 中的 key 的数量

示例：

```shell
> JSON.SET doc $ '{"a":[3], "nested": {"a": {"b":2, "c": 1}}}'
"OK"

> JSON.OBJKEYS doc $..a
1) "null"
2) 1) "b"
   2) "c"

> JSON.OBJLEN doc $
1) "2"
```

### 针对 number 类型

- `JSON.NUMINCRBY`：为 number 类型增加数值

示例：

```shell
> JSON.SET doc $ '{"a": 1, "b": 2}'
"OK"

> JSON.NUMINCRBY doc $.a 10
"[11]"

> JSON.GET doc $
"[{\"a\":11,\"b\":2}]"

> JSON.NUMINCRBY doc $.b -3
"[-1]"

> JSON.GET doc $
"[{\"a\":11,\"b\":-1}]"
```

### 针对 string 类型

- `JSON.STRAPPEND`：追加字符串
- `JSON.STRLEN`：返回字符串的长度

示例：

```shell
> JSON.SET doc $ '{"a":"foo", "nested": {"a": "hello"}, "nested2": {"a": 31}}'
"OK"

> JSON.STRAPPEND doc $..a '"baz"'
1) "6"
2) "8"
3) "null"

> JSON.GET doc $
"[{\"a\":\"foobaz\",\"nested\":{\"a\":\"hellobaz\"},\"nested2\":{\"a\":31}}]"

> JSON.STRLEN doc $.a
1) "6"
```

### 针对 boolean 类型

- `JSON.TOGGLE`：切换布尔值，把 false 与 true 对换

示例：

```shell
> JSON.SET doc $ '{"bool": true}'
"OK"

> JSON.TOGGLE doc $.bool
1) "0"

> JSON.GET doc $
"[{\"bool\":false}]"

> JSON.TOGGLE doc $.bool
1) "1"

> JSON.GET doc $
"[{\"bool\":true}]"
```

### 调试

- `JSON.DEBUG`
- `JSON.DEBUG MEMORY`：返回内存占用大小

示例：

```shell
> JSON.SET doc $ '{"a": 1, "b": 2, "c": {}}'
"OK"

> JSON.DEBUG MEMORY doc
(integer) 147

> JSON.DEBUG MEMORY doc $.c
1) "8"
```

### 批量操作

- `JSON.MGET`：批量 GET 多个 key 的值
> `JSON.MGET key [key ...] path`

示例：

```shell
> JSON.SET doc1 $ '{"a":1, "b": 2, "nested": {"a": 3}, "c": null}'
"OK"

> JSON.SET doc2 $ '{"a":4, "b": 5, "nested": {"a": 6}, "c": null}'
"OK"

> JSON.MGET doc1 doc2 $..a
1) "[1,3]"
2) "[4,6]"
```

- `JSON.MSET`：批量 SET 设置数值，这个操作是原子的，这意味着批量操作要么全都生效，要么全都不生效
> `JSON.MSET key path value [key path value ...]`

示例：

```shell
> JSON.MSET doc1 $ '{"a":1}' doc2 $ '{"f":{"a":2}}' doc3 $ '{"f1":{"a":0},"f2":{"a":0}}'
"OK"

> JSON.MSET doc1 $ '{"a":2}' doc2 $.f.a '3' doc3 $ '{"f1":{"a":1},"f2":{"a":2}}'
"OK"

> JSON.MGET doc1 doc2 doc3 $
1) "[{\"a\":2}]"
2) "[{\"f\":{\"a\":3}}]"
3) "[{\"f1\":{\"a\":1},\"f2\":{\"a\":2}}]"
```

### 已弃用
- `JSON.RESP`
- `JSON.MUMMULTIBY`

## 总结

面对 JSON 数据格式的流行，Redis 也并未落后，通过 `RedisJSON` 模块很好地进行了支持。而基于 `RedisJSON`，Redis Stack 还能作为一个 Document database 文档数据库、一个全文搜索引擎、或者一个向量搜索引擎。

本文姑且介绍了 `RedisJSON` 的基本用法。关注我，等待我的后续文章进一步了解 Redis Stack 的其他功能。

---

参考资料：

- *https://redis.io/docs/about/about-stack/*
- *https://redis.io/docs/data-types/json/*
- *https://redis.io/commands/?group=json*
