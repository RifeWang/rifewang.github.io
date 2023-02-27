# Elasticsearch Search Template


# Elasticsearch Search Template

所谓 search template 搜索模板其实就是：
1. 预先定义好查询语句 DSL 的结构并预留参数
2. 搜索的时再传入参数值
3. 渲染出完整的 DSL ，最后进行搜索

使用搜索模板可以将 DSL 从应用程序中解耦出来，并且可以更加灵活的更改查询语句。

例如：
```
GET _search/template
{
  "source" : {
    "query": {
      "match" : {
        "{{my_field}}" : "{{my_value}}"
      }
    }
  },
  "params" : {
    "my_field" : "message",
    "my_value" : "foo"
  }
}
```

构造出来的 DSL 就是：
```
{
  "query": {
    "match": {
      "message": "foo"
    }
  }
}
```

在模板中通过 `{{ }}` 的方式预留参数，然后查询时再指定对应的参数值，最后填充成具体的查询语句进行搜索。


----
## 搜索模板 API


为了实现搜索模板和查询分离，我们首先需要单独保存和管理搜索模板。


#### 保存搜索模板

使用 scripts API 保存搜索模板（不存在则创建，存在则覆盖）。示例：
```
POST _scripts/<templateid>
{
  "script": {
    "lang": "mustache",
    "source": {
      "query": {
        "match": {
          "title": "{{query_string}}"
        }
      }
    }
  }
}
```

#### 查询搜索模板

```
GET _scripts/<templateid>
```

#### 删除搜索模板

```
DELETE _scripts/<templateid>
```

#### 使用搜索模板

示例：
```
GET _search/template
{
  "id": "<templateid>",
  "params": {
    "query_string": "search words"
  }
}
```

`params` 中的参数与搜索模板中定义的一致，上文保存搜索模板的示例是 `{{query_string}}`，所以这里进行搜索时对应的参数就是 `query_string` 。


#### 检验搜索模板

有时候我们想看看搜索模板输入了参数之后渲染成的 DSL 到底长啥样。

示例：
```
GET _render/template
{
  "source": "{ \"query\": { \"terms\": {{#toJson}}statuses{{/toJson}} }}",
  "params": {
    "statuses" : {
        "status": [ "pending", "published" ]
    }
  }
}
```

返回的结果就是：
```
{
  "template_output": {
    "query": {
      "terms": {
        "status": [
          "pending",
          "published"
        ]
      }
    }
  }
}
```

`{{#toJson}} {{/toJson}}` 就是转换成 json 格式。

</br>

已经保存的搜索模板可以通过以下方式查看渲染结果：
```
GET _render/template/<template_name>
{
  "params": {
    "..."
  }
}
```


#### 使用 `explain` 和 `profile` 参数

示例：
```
GET _search/template
{
  "id": "my_template",
  "params": {
    "status": [ "pending", "published" ]
  },
  "explain": true
}
```

```
GET _search/template
{
  "id": "my_template",
  "params": {
    "status": [ "pending", "published" ]
  },
  "profile": true
}
```

## 模板渲染

#### 填充简单值

```
GET _search/template
{
  "source": {
    "query": {
      "term": {
        "message": "{{query_string}}"
      }
    }
  },
  "params": {
    "query_string": "search words"
  }
}
```

渲染出来的 DSL 就是：
```
{
  "query": {
    "term": {
      "message": "search words"
    }
  }
}
```


#### 将参数转换为 JSON

使用 `{{#toJson}}parameter{{/toJson}}` 会将参数转换为 JSON。

```
GET _search/template
{
  "source": "{ \"query\": { \"terms\": {{#toJson}}statuses{{/toJson}} }}",
  "params": {
    "statuses" : {
        "status": [ "pending", "published" ]
    }
  }
}
```

渲染出来的 DSL 就是：
```
{
  "query": {
    "terms": {
      "status": [
        "pending",
        "published"
      ]
    }
  }
}
```

</br>

对象数组的渲染示例：
```
GET _search/template
{
  "source": "{\"query\":{\"bool\":{\"must\": {{#toJson}}clauses{{/toJson}} }}}",
  "params": {
    "clauses": [
      { "term": { "user" : "foo" } },
      { "term": { "user" : "bar" } }
    ]
  }
}
```

渲染结果就是：
```
{
  "query": {
    "bool": {
      "must": [
        { "term": { "user" : "foo" } },
        { "term": { "user" : "bar" } }
      ]
    }
  }
}
```


#### 将数组 join 成字符串

使用 `{{#join}}array{{/join}}` 可以将数组 join 成字符串。

示例：
```
GET _search/template
{
  "source": {
    "query": {
      "match": {
        "emails": "{{#join}}emails{{/join}}"
      }
    }
  },
  "params": {
    "emails": [ "aaa", "bbb" ]
  }
}
```

渲染结果：
```
{
  "query" : {
    "match" : {
      "emails" : "aaa,bbb"
    }
  }
}
```

</br>

除了默认以 `,` 分隔外，还可以自定义分隔符，示例：
```
{
  "source": {
    "query": {
      "range": {
        "born": {
          "gte": "{{date.min}}",
          "lte": "{{date.max}}",
          "format": "{{#join delimiter='||'}}date.formats{{/join delimiter='||'}}"
        }
      }
    }
  },
  "params": {
    "date": {
      "min": "2016",
      "max": "31/12/2017",
      "formats": [ "dd/MM/yyyy", "yyyy" ]
    }
  }
}
```

例子中的 `{{#join delimiter='||'}} {{/join delimiter='||'}}` 意思就是进行 join 操作，分隔符设置为 `||` ，渲染结果就是：
```
{
  "query": {
    "range": {
      "born": {
        "gte": "2016",
        "lte": "31/12/2017",
        "format": "dd/MM/yyyy||yyyy"
      }
    }
  }
}
```


#### 默认值

使用 `{{var}}{{^var}}default{{/var}}` 的方式设置默认值。

示例：
```
{
  "source": {
    "query": {
      "range": {
        "line_no": {
          "gte": "{{start}}",
          "lte": "{{end}}{{^end}}20{{/end}}"
        }
      }
    }
  },
  "params": { ... }
}
```

`{{end}}{{^end}}20{{/end}}` 就是给 `end` 设置了默认值为 20 。

当 `params` 是 `{ "start": 10, "end": 15 }` 时，渲染结果是：
```
{
  "range": {
    "line_no": {
      "gte": "10",
      "lte": "15"
    }
  }
}
```

当 `params` 是 `{ "start": 10 }` 时，end 就会使用默认值，渲染结果就是：
```
{
  "range": {
    "line_no": {
      "gte": "10",
      "lte": "20"
    }
  }
}
```


#### 条件子句

有时候我们的参数是可选的，这时候就可以使用 `{{#key}}  {{/key}}`的语法。



示例，假设参数 `line_no`, `start`, `end` 都是可选的，使用 `{{#key}}  {{/key}}` 形如：
```
{
  "query": {
    "bool": {
      "must": {
        "match": {
          "line": "{{text}}"
        }
      },
      "filter": {
        {{#line_no}}
          "range": {
            "line_no": {
              {{#start}}
                "gte": "{{start}}"
                {{#end}},{{/end}}
              {{/start}}
              {{#end}}
                "lte": "{{end}}"
              {{/end}}
            }
          }
        {{/line_no}}
      }
    }
  }
}
```

1、 当参数为：
```
{
  "params": {
    "text": "words to search for",
    "line_no": {
      "start": 10,
      "end": 20
    }
  }
}
```

渲染结果是：
```
{
  "query": {
    "bool": {
      "must": {
        "match": {
          "line": "words to search for"
        }
      },
      "filter": {
        "range": {
          "line_no": {
            "gte": "10",
            "lte": "20"
          }
        }
      }
    }
  }
}
```

2、 当参数为：
```
{
  "params": {
    "text": "words to search for"
  }
}
```

渲染结果为：
```
{
  "query": {
    "bool": {
      "must": {
        "match": {
          "line": "words to search for"
        }
      },
      "filter": {}
    }
  }
}
```

3、当参数为：
```
{
  "params": {
    "text": "words to search for",
    "line_no": {
      "start": 10
    }
  }
}
```

渲染结果为：
```
{
  "query": {
    "bool": {
      "must": {
        "match": {
          "line": "words to search for"
        }
      },
      "filter": {
        "range": {
          "line_no": {
            "gte": 10
          }
        }
      }
    }
  }
}
```

4、当参数为：
```
{
  "params": {
    "text": "words to search for",
    "line_no": {
      "end": 20
    }
  }
}
```

渲染结果为：
```
{
  "query": {
    "bool": {
      "must": {
        "match": {
          "line": "words to search for"
        }
      },
      "filter": {
        "range": {
          "line_no": {
            "lte": 20
          }
        }
      }
    }
  }
}
```

需要注意的是在 JSON 对象中，
```
{
  "filter": {
    {{#line_no}}
    ...
    {{/line_no}}
  }
}
```

这样直接写 `{{#line_no}}` 肯定是非法的JSON格式，你必须转换为 JSON 字符串。


#### URLs 编码

使用 `{{#url}}value{{/url}}` 的方式可以进行 HTML 编码转义。

示例：
```
GET _render/template
{
  "source": {
    "query": {
      "term": {
        "http_access_log": "{{#url}}{{host}}/{{page}}{{/url}}"
      }
    }
  },
  "params": {
    "host": "https://www.elastic.co/",
    "page": "learn"
  }
}
```

渲染结果：
```
{
  "template_output": {
    "query": {
      "term": {
        "http_access_log": "https%3A%2F%2Fwww.elastic.co%2F%2Flearn"
      }
    }
  }
}
```


----
## Mustache 基本语法

上文中的 `{{ }}` 语法其实就是 [mustache language](https://mustache.github.io/mustache.5.html) ，补充介绍下基本的语法规则。


#### 使用 `{{key}}`

模板：`Hello {{name}}`

输入：
```
{
    "name": "Chris"
}
```

输出：`Hello Chris`


#### 使用 `{{{key}}}` 避免转义

所有变量都会默认进行 HTML 转义。

模板：`{{company}}`

输入：
```
{
    "company": "<b>GitHub</b>"
}
```

输出：`&lt;b&gt;GitHub&lt;/b&gt;`

使用 `{{{ }}}` 避免转义。

模板：`{{{company}}}`

输入：
```
{
    "company": "<b>GitHub</b>"
}
```

输出：`<b>GitHub</b>`


#### 使用 `{{#key}}  {{/key}}` 构造区块


1、 当 key 是 false 或者空列表将会忽略
模板：
```
    Shown.
    {{#person}}
        Never shown!
    {{/person}}
```

输入：
```
{
    "person": false
}
```

输出：
```
    Shown.
```

</br>

2、 当 key 非空值则渲染填充
模板：
```
    {{#repo}}
        <b>{{name}}</b>
    {{/repo}}
```

输入：
```
{
    "repo": [
        { "name": "resque" },
        { "name": "hub" },
        { "name": "rip" }
    ]
}
```

输出：
```
    <b>resque</b>
    <b>hub</b>
    <b>rip</b>
```


</br>

3、当 key 是函数则调用后渲染
模板：
```
    {{#wrapped}}
        {{name}} is awesome.
    {{/wrapped}}
```

输入：
```
{
    "name": "Willy",
    "wrapped": function() {
        return function(text, render) {
            return "<b>" + render(text) + "</b>"
        }
    }
}
```

输出：
```
    <b>Willy is awesome.</b>
```

</br>

4、当 key 是非 false 且非列表
模板：
```
    {{#person?}}
        Hi {{name}}!
    {{/person?}}
```

输入：
```
{
    "person?": { "name": "Jon" }
}
```

输出：
```
    Hi Jon!
```


#### 使用 `{{^key}}  {{/key}}` 构造反区块

`{{^key}}  {{/key}}` 的语法与 `{{#key}}  {{/key}}` 类似，不同的是，当 key 不存在，或者是 false ，又或者是空列表时才渲染输出区块内容。

模板：
```
    {{#repo}}
        <b>{{name}}</b>
    {{/repo}}
    {{^repo}}
        No repos :(
    {{/repo}}
```

输入：
```
{
    "repo": []
}
```

输出：
```
    No repos :(
```


#### 使用 `{{! }}` 添加注释

`{{! }}` 注释内容将会被忽略。

模板：
```
<h1>Today{{! ignore me }}.</h1>
```

输出：
```
<h1>Today.</h1>
```


#### 使用 `{{> }}` 子模块

模板：
```
base.mustache:
<h2>Names</h2>
{{#names}}
    {{> user}}
{{/names}}

user.mustache:
<strong>{{name}}</strong>
```

其实也就等价于：
```
<h2>Names</h2>
{{#names}}
    <strong>{{name}}</strong>
{{/names}}
```


#### 使用 `{{=  =}}` 自定义定界符

有时候我们需要改变默认的定界符 `{{ }}` ，那么就可以使用 `{{=  =}}` 的方式自定义定界符。

例如：
```
{{=<% %>=}}
```

定界符被定义为了 `<% %>`，这样原先 `{{key}}` 的使用方式就变成了 `<%key%>`。
再使用：
```
<%={{ }}=%>
```
就重新把定界符改回了 `{{ }}`。


更多语法详情请查阅官方文档 [mustache language](https://mustache.github.io/mustache.5.html) 。



## 结语

使用 search template 可以对搜索进行有效的解耦，即应用程序只需要关注搜索参数与返回结果，而不用关注具体使用的 DSL 查询语句，到底使用哪种 DSL 则由搜索模板进行单独管理。
