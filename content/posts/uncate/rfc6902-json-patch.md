+++
draft = false
date = 2023-04-07T11:27:16+08:00
title = "什么？修改 JSON 内容居然还有个 JSON PATCH 标准"
description = "什么？修改 JSON 内容居然还有个 JSON PATCH 标准"
slug = ""
authors = []
tags = ["RFC 标准"]
categories = ["Uncate"]
externalLink = ""
series = []
disableComments = true
+++

## 引言

你一定知道 JSON 吧，那专门用于修改 JSON 内容的 JSON PATCH 标准你是否知道呢？

[RFC 6902](https://www.rfc-editor.org/rfc/rfc6902) 就定义了这么一种 JSON PATCH 标准，本文将对其进行介绍。

## JSON PATCH

JSON Patch 本身也是一种 JSON 文档结构，用于表示要应用于 JSON 文档的操作序列；它适用于 HTTP `PATCH` 方法，其 MIME 媒体类型为 `"application/json-patch+json"`。

这句话也许不太好理解，我们先看一个例子：

```
PATCH /my/data HTTP/1.1
Host: example.org
Content-Length: 326
Content-Type: application/json-patch+json
If-Match: "abc123"

[
    { "op": "test", "path": "/a/b/c", "value": "foo" },
    { "op": "remove", "path": "/a/b/c" },
    { "op": "add", "path": "/a/b/c", "value": [ "foo", "bar" ] },
    { "op": "replace", "path": "/a/b/c", "value": 42 },
    { "op": "move", "from": "/a/b/c", "path": "/a/b/d" },
    { "op": "copy", "from": "/a/b/d", "path": "/a/b/e" }
]
```

这个 HTTP 请求的 body 也是 JSON 格式（JSON PATCH 本身也是一种 JSON 结构），但是这个 JSON 格式是有具体规范的（只能按照标准去定义要应用于 JSON 文档的操作序列）。

具体而言，JSON Patch 的数据结构就是一个 JSON 对象数组，其中每个对象必须声明 `op` 去定义将要执行的操作，根据 `op` 操作的不同，需要对应另外声明 `path`、`value` 或 `from` 字段。

再例如，

原始 JSON ：

```json
{
    "a": "aaa",
    "b": "bbb"
}
```

应用如下 JSON PATCH ：

```json
[
    { "op": "replace", "path": "/a", "value": "111" },
    { "op": "remove", "path": "/b" }
]
```

得到的结果为：

```json
{
    "a": "111"
}
```

需要注意的是：

- patch 对象中的属性没有顺序要求，比如 `{ "op": "remove", "path": "/b" }` 与 `{ "path": "/b", "op": "remove" }` 是完全等价的。
- patch 对象的执行是按照数组顺序执行的，比如上例中先执行了 replace，然后再执行 remove。
- patch 操作是原子的，即使我们声明了多个操作，但最终的结果是要么全部成功，要么保持原数据不变，不存在局部变更。也就是说如果多个操作中的某个操作异常失败了，那么原数据就不变。

### op

`op` 只能是以下操作之一：

- `add`
- `remove`
- `replace`
- `move`
- `copy`
- `test`

这些操作我相信不用做任何说明你就能理解其具体的含义，唯一要说明的可能就是 `test`，`test` 操作其实就是检查 `path` 位置的值与 `value` 的值“相等”。

#### add

`add` 操作会根据 `path` 定义去执行不同的操作：

- 如果 `path` 是一个数组 index ，那么新的 `value` 值会被插入到执行位置。
- 如果 `path` 是一个不存在的对象成员，那么新的对象成员会被添加到该对象中。
- 如果 `path` 是一个已经存在的对象成员，那么该对象成员的值会被 `value` 所替换。

`add` 操作必须另外声明 `path` 和 `value`。

`path` 目标位置必须是以下之一：

- 目标文档的根 - 如果 `path` 指向的是根，那么 `value` 值就将是整个文档的内容。
- 一个已存在对象的成员 - 应用后 `value` 将会被添加到指定位置，如果成员已存在则其值会被替换。
- 一个已存在数组的元素 - 应用后 `value` 值会被添加到数组中的指定位置，任何在指定索引位置或之上的元素都会向右移动一个位置。指定的索引不能大于数组中元素的数量。可以使用 `-` 字符来索引数组的末尾。

由于此操作旨在添加到现有对象和数组中，因此其目标位置通常不存在。尽管指针的错误处理算法将被调用，但本规范定义了 add 指针的错误处理行为，以忽略该错误并按照指定方式添加值。

然而，对象本身或包含它的数组确实需要存在，并且如果不是这种情况，则仍然会出错。

例如，对数据 `{ "a": { "foo": 1 } }` 执行 add 操作，path 为 "/a/b" 时不是错误。但如果对数据 `{ "q": { "bar": 2 } }` 执行同样的操作则是一种错误，因为 "a" 不存在。

---

示例：

1. `add` 一个对象成员
    ```yaml
    # 源数据：
    { "foo": "bar"}

    # JSON Patch:
    [
        { "op": "add", "path": "/baz", "value": "qux" }
    ]

    # 结果：
    {
        "baz": "qux",
        "foo": "bar"
    }
    ```

1. `add` 一个数组元素
    ```yaml
    # 源数据：
    { "foo": [ "bar", "baz" ] }

    # JSON Patch:
    [
        { "op": "add", "path": "/foo/1", "value": "qux" }
    ]

    # 结果：
    { "foo": [ "bar", "qux", "baz" ] }
    ```

1. `add` 一个嵌套成员对象
    ```yaml
    # 源数据：
    { "foo": "bar" }

    # JSON Patch:
    [
        { "op": "add", "path": "/child", "value": { "grandchild": { } } }
    ]

    # 结果：
    {
        "foo": "bar",
        "child": {
            "grandchild": {}
        }
    }
    ```

1. 忽略未识别的元素
    ```yaml
    # 源数据：
    { "foo": "bar" }

    # JSON Patch:
    [
        { "op": "add", "path": "/baz", "value": "qux", "xyz": 123 }
    ]

    # 结果：
    {
        "foo": "bar",
        "baz": "qux"
    }
    ```

1. `add` 到一个不存在的目标失败
    ```yaml
    # 源数据：
    { "foo": "bar" }

    # JSON Patch:
    [
        { "op": "add", "path": "/baz/bat", "value": "qux" }
    ]

    # 失败，因为操作的目标位置既不引用文档根，也不引用现有对象的成员，也不引用现有数组的成员。
    ```

1. `add` 一个数组
    ```yaml
    # 源数据：
    { "foo": ["bar"] }

    # JSON Patch:
    [
        { "op": "add", "path": "/foo/-", "value": ["abc", "def"] }
    ]

    # 结果：
    { "foo": ["bar", ["abc", "def"]] }
    ```

#### remove

`remove` 将会删除 path 目标位置上的值，如果 path 指向的是一个数组 index ，那么右侧其余值都将左移。

示例：

1. `remove` 一个对象成员
    ```yaml
    # 源数据：
    {
        "baz": "qux",
        "foo": "bar"
    }

    # JSON Patch:
    [
        { "op": "remove", "path": "/baz" }
    ]

    # 结果：
    { "foo": "bar" }
    ```

1. `remove` 一个数组元素
    ```yaml
    # 源数据：
    { "foo": [ "bar", "qux", "baz" ] }

    # JSON Patch:
    [
        { "op": "remove", "path": "/foo/1" }
    ]

    # 结果：
    { "foo": [ "bar", "baz" ] }
    ```

#### replace

`replace` 操作会将 `path` 目标位置上的值替换为 `value`。此操作与 `remove` 后 `add` 同样的 `path` 在功能上是相同的。

示例：

1. `replace` 某个值
    ```yaml
    # 源数据：
    {
        "baz": "qux",
        "foo": "bar"
    }

    # JSON Patch:
    [
        { "op": "replace", "path": "/baz", "value": "boo" }
    ]

    # 结果：
    {
        "baz": "boo",
        "foo": "bar"
    }
    ```

#### move

`move` 操作将 `from` 位置的值移动到 `path` 位置。`from` 位置不能是 `path` 位置的前缀，也就是说，一个位置不能被移动到它的子级中。

示例：

1. `move` 某个值
    ```yaml
    # 源数据：
    {
        "foo": {
            "bar": "baz",
            "waldo": "fred"
        },
        "qux": {
            "corge": "grault"
        }
    }

    # JSON Patch:
    [
        { "op": "move", "from": "/foo/waldo", "path": "/qux/thud" }
    ]

    # 结果：
    {
        "foo": {
            "bar": "baz"
        },
        "qux": {
            "corge": "grault",
            "thud": "fred"
        }
    }
    ```

1. `move` 一个数组元素
    ```yaml
    # 源数据：
    { "foo": [ "all", "grass", "cows", "eat" ] }

    # JSON Patch:
    [
        { "op": "move", "from": "/foo/1", "path": "/foo/3" }
    ]

    # 结果：
    { "foo": [ "all", "cows", "eat", "grass" ] }
    ```

#### copy

`copy` 操作将 `from` 位置的值复制到 `path` 位置。

#### test

`test` 操作会检查 `path` 位置的值是否与 `value` “相等”。

这里，“相等”意味着 `path` 位置的值和 `value` 的值是相同的JSON类型，并且它们遵循以下规则：

- 字符串：如果它们包含相同数量的 Unicode 字符并且它们的码点是逐字节相等，则被视为相等。
- 数字：如果它们的值在数值上是相等的，则被视为相等。
- 数组：如果它们包含相同数量的值，并且每个值可以使用此类型特定规则将其视为与另一个数组中对应位置处的值相等，则被视为相等。
- 对象：如果它们包含相同数量​​的成员，并且每个成员可以通过比较其键（作为字符串）和其值（使用此类型特定规则）来认为与其他对象中的成员相等，则被视为相等 。
- 文本（false，true 和 null）：如果它们完全一样，则被视为相等。

请注意，所进行的比较是逻辑比较；例如，数组成员之间的空格不重要。


示例：

1. `test` 某个值成功
    ```yaml
    # 源数据：
    {
        "baz": "qux",
        "foo": [ "a", 2, "c" ]
    }

    # JSON Patch:
    [
        { "op": "test", "path": "/baz", "value": "qux" },
        { "op": "test", "path": "/foo/1", "value": 2 }
    ]
    ```

1. `test` 某个值错误
    ```yaml
    # 源数据：
    { "baz": "qux" }

    # JSON Patch:
    [
        { "op": "test", "path": "/baz", "value": "bar" }
    ]
    ```

1. `~` 符号转义

    `~` 字符是 JSON 指针中的关键字。因此，我们需要将其编码为 `〜0`

    ```yaml
    # 源数据：
    {
        "/": 9,
        "~1": 10
    }

    # JSON Patch:
    [
        {"op": "test", "path": "/~01", "value": 10}
    ]

    # 结果：
    {
        "/": 9,
        "~1": 10
    }
    ```

1. 比较字符串和数字
    ```yaml
    # 源数据：
    {
        "/": 9,
        "~1": 10
    }

    # JSON Patch:
    [
        {"op": "test", "path": "/~01", "value": "10"}
    ]

    # 失败，因为不遵循上述相等的规则。
    ```

## 结语

使用 JSON PATCH 的原因之一其实是为了避免在只需要修改某一部分内容的时候重新发送整个文档。JSON PATCH 也早已应用在了 Kubernetes 等许多项目中。