# 数据管道 Logstash 入门


# Logstash 入门

----
## Logstash 是什么

Logstash 就是一个开源的数据流工具，它会做三件事：
1. 从数据源拉取数据
2. 对数据进行过滤、转换等处理
3. 将处理后的数据写入目标地

例如：
- 监听某个目录下的日志文件，读取文件内容，处理数据，写入 influxdb 。
- 从 kafka 中消费消息，处理数据，写入 elasticsearch 。


### 为什么要用 Logstash ？

方便省事。

假设你需要从 kafka 中消费数据，然后写入 elasticsearch ，如果自己编码，你得去对接 kafka 和 elasticsearch 的 API 吧，如果你用 Logstash ，这部分就不用自己去实现了，因为 Logstash 已经为你封装了对应的 `plugin` 插件，你只需要写一个配置文件形如：
```
input {
    kafka {
        # kafka consumer 配置
    }
}

filter {
    # 数据处理配置
}

output {
    elasticsearch {
        # elasticsearch 输出配置
    }
}
```
然后运行 logstash 就可以了。


Logstash 提供了两百多个封装好的 `plugin` 插件，这些插件被分为三类：
- `input plugin` : 从哪里拉取数据
- `filter plugin` : 数据如何处理
- `output plugin` : 数据写入何处

使用 logstash 你只要编写一个配置文件，在配置文件中挑选组合这些 `plugin` 插件，就可以轻松实现数据从输入源到输出源的实时流动。


----
## 安装 logstash

请参数：[官方文档](https://www.elastic.co/guide/en/logstash/current/installing-logstash.html)


----
## 第一个示例

假设你已经安装好了 logstash ，并且可执行文件的路径已经加入到了 PATH 环境变量中。

下面开始我们的第一个示例，编写 `pipeline.conf` 文件，内容为：
```
input {
    stdin {

    }
}

filter {

}

output {
    stdout {

    }
}
```

这个配置文件的含义是：
- `input` 输入为 `stdin`（标准输入）
- `filter` 为空（也就是不进行数据的处理）
- `output` 输出为 `stdout`（标准输出）


执行命令：
```
logstash -f pipeline.conf
```

等待 logstash 启动完毕，输入 hello world 然后回车, 你就会看到以下输出内容:
```
{
       "message" => "hello world",
      "@version" => "1",
    "@timestamp" => 2020-11-01T08:25:10.987Z,
          "host" => "local"
}
```
我们输入的内容已经存在于 `message` 字段中了。

当你输入其他内容后也会看到类似的输出。

至此，我们的第一个示例已经完成，正如配置文件中所定义的，Logstash 从 stdin 标准输入读取数据，不对源数据做任何处理，然后输出到 stdout 标准输出。


### 特定名词和字段

- `event` : 数据在 logstash 中被包装成 `event` 事件的形式从 input 到 filter 再到 output 流转。
- `@timestamp` : 特殊字段，标记 event 发生的时间。
- `@version` : 特殊字段，标记 event 的版本号。
- `message` : 源数据内容。
- `@metadata` : 元数据，key/value 的形式，是否有数据得看具体插件，例如 kafka 的 input 插件会在 `@metadata` 里记录 topic、consumer_group、partition、offset 等一些元数据。
- `tags` : 记录 tag 的字符串数组。

### 字段引用

在配置文件中，可以通过 `[field]` 的形式引用字段内容，如果在字符串中，则可以通过 `%{[field]}` 的方式进行引用。

示例：
```
input {
    kafka {
        # kafka 配置
    }
}

filter {
    # 引用 log_level 字段的内容进行判断
    if [log_level] == "debug" {

    }
}

output {
  elasticsearch {
    # %{+yyyy.MM.dd} 来源于 @timestamp
    index => "log-%{+yyyy.MM.dd}"
    document_type => "_doc"
    document_id => "%{[@metadata][kafka][key]}"
    hosts => ["127.0.0.1:9200"]
  }
}
```

----
## Plugin 插件一览

用好 Logstash 的第一步就是熟悉 plugin 插件，只有熟悉了这些插件你才能快速高效的建立数据管道。

### `Input plugin`

`Input` 插件定义了数据源，即 logstash 从哪里拉取数据。


- `beats` : 从 [Elastic Beats](https://www.elastic.co/cn/beats/) 框架中接收数据。

示例：
```
input {
  beats {
    port => 5044
  }
}
```


- `dead_letter_queue` : 从 Logstash 自己的 [dead letter queue](https://www.elastic.co/guide/en/logstash/current/dead-letter-queues.html) 中拉取数据，目前 dead letter queue 只支持记录 output 为 elasticsearch 时写入 400 或 404 的数据。

示例：
```
input {
  dead_letter_queue {
    path => "/var/logstash/data/dead_letter_queue"
    start_timestamp => "2017-04-04T23:40:37"
  }
}
```


- `elasticsearch` : 从 elasticsearch 中读取 search query 的结果。

示例：
```
input {
  elasticsearch {
    hosts => "localhost"
    query => '{ "query": { "match": { "statuscode": 200 } } }'
  }
}
```


- `exec` : 定期执行一个 shell 命令，然后捕获其输出。

示例：
```
input {
  exec {
    command => "ls"
    interval => 30
  }
}
```

- `file` : 从文件中流式读取内容。

示例：
```
input {
  file {
    path => ["/var/log/*.log", "/var/log/message"]
    start_position => "beginning"
  }
}
```

- `generator` : 生成随机数据。

示例：
```
input {
  generator {
    count => 3
    lines => [
      "line 1",
      "line 2",
      "line 3"
    ]
  }
}
```

- `github` : 从 github webhooks 中读取数据。
- `graphite` : 接受 graphite 的 metrics 指标数据。
- `heartbeat` : 生成心跳信息。这样做的一般目的是测试 Logstash 的性能和可用性。
- `http` : Logstash 接受 http 请求作为数据。
- `http_poller` : Logstash 发起 http 请求，读取响应数据。

示例：
````
input {
  http_poller {
    urls => {
      test1 => "http://localhost:9200"
      test2 => {
        method => get
        user => "AzureDiamond"
        password => "hunter2"
        url => "http://localhost:9200/_cluster/health"
        headers => {
          Accept => "application/json"
        }
     }
    }
    request_timeout => 60
    schedule => { cron => "* * * * * UTC"}
    codec => "json"
    metadata_target => "http_poller_metadata"
  }
}
````


- `imap` : 从 IMAP 服务器读取邮件。
- `jdbc` : 通过 JDBC 接口导入数据库中的数据。

示例：
```
input {
  jdbc {
    jdbc_driver_library => "mysql-connector-java-5.1.36-bin.jar"
    jdbc_driver_class => "com.mysql.jdbc.Driver"
    jdbc_connection_string => "jdbc:mysql://localhost:3306/mydb"
    jdbc_user => "mysql"
    parameters => { "favorite_artist" => "Beethoven" }
    schedule => "* * * * *"
    statement => "SELECT * from songs where artist = :favorite_artist"
  }
}
```


- `kafka` : 消费 kafka 中的消息。

示例：
```
input {
  kafka {
    bootstrap_servers => "127.0.0.1:9092"
    group_id => "consumer_group"
    topics => ["kafka_topic"]
    enable_auto_commit => true
    auto_commit_interval_ms => 5000
    auto_offset_reset => "latest"
    decorate_events => true
    isolation_level => "read_uncommitted"
    max_poll_records => 1000
  }
}
```

- `rabbitmq` : 从 RabbitMQ 队列中拉取数据。
- `redis` : 从 redis 中读取数据。
- `stdin` : 从标准输入读取数据。
- `syslog` : 读取 syslog 数据。
- `tcp` : 通过 TCP socket 读取数据。
- `udp` : 通过 udp 读取数据。
- `unix` : 通过 UNIX socket 读取数据。
- `websocket` : 通过 websocket 协议 读取数据。


### `Output plugin`

`Output` 插件定义了数据的输出地，即 logstash 将数据写入何处。

- `csv` : 将数据写入 csv 文件。
- `elasticsearch` : 写入 Elasticsearch 。
- `email` : 发送 email 邮件。
- `exec` : 执行命令。
- `file` : 写入磁盘文件。
- `graphite` : 写入 Graphite 。
- `http` : 发送 http 请求。
- `influxdb` : 写入 InfluxDB 。
- `kafka` : 写入 Kafka 。
- `mongodb` : 写入 MongoDB 。
- `opentsdb` : 写入 OpenTSDB 。
- `rabbitmq` : 写入 RabbitMQ 。
- `redis` : 使用 RPUSH 的方式写入到 Redis 队列。
- `sink` : 将数据丢弃，不写入任何地方。
- `syslog` : 将数据发送到 syslog 服务端。
- `tcp` : 发送 TCP socket。
- `udp` : 发送 UDP 。
- `webhdfs` : 通过 webhdfs REST API 写入 HDFS 。
- `websocket` : 推送 websocket 消息 。


### `Filter plugin`

`Filter` 插件定义对数据进行如何处理。

- `aggregate` : 聚合数据。
- `alter` : 修改数据。
- `bytes` : 将存储大小如 "123 MB" 或 "5.6gb" 的字符串表示形式解析为以字节为单位的数值。
- `cidr` : 检查 IP 地址是否在指定范围内。

示例：
```
filter {
  cidr {
    add_tag => [ "testnet" ]
    address => [ "%{src_ip}", "%{dst_ip}" ]
    network => [ "192.0.2.0/24" ]
  }
}
```
- `cipher` : 对数据进行加密或解密。
- `clone` : 复制 event 事件。
- `csv` : 解析 CSV 格式的数据。
- `date` : 解析字段中的日期数据。

示例，匹配输入的 timestamp 字段，然后替换 @timestamp ：
```
filter {
  date {
    match => ["timestamp", "dd/MMM/yyyy:HH:mm:ss ZZ"]
    target => "@timestamp"
  }
}
```
- `dissect` : 使用 `%{}` 的形式拆分字符串并提取出特定内容，比较常用，具体语法见 [dissect 文档](https://www.elastic.co/guide/en/logstash/current/plugins-filters-dissect.html)。
- `drop` : 丢弃这个 event 。

示例：
```
filter {
  if [loglevel] == "debug" {
    drop { }
  }
}
```

- `elapsed` : 通过记录开始和结束时间跟踪 event 的耗时。
- `elasticsearch` : 在 elasticsearch 中进行搜索，并将数据复制到当前 event 中。
- `environment` : 将环境变量中的数据存储到 @metadata 字段中。
- `extractnumbers` : 提取字符串中找到的所有数字。
- `fingerprint` : 根据一个或多个字段的内容创建哈希值，并存储到新的字段中。
- `geoip` : 使用绑定的 [GeoLite2](https://dev.maxmind.com/geoip/geoip2/geolite2/) 数据库添加有关 IP 地址的地理位置的信息，这个插件非常有用，你可以根据 IP 地址得到对应的国家、省份、城市、经纬度等地理位置数据。

示例，通过 clent_ip 字段获取对应的地理位置信息：
```
filter {
  geoip {
    cache_size => 1000
    default_database_type => "City"
    source => "clent_ip"
    target => "geo"
    tag_on_failure => ["_geoip_city_fail"]
    add_field => {
      "geo_country_name" => "%{[geo][country_name]}"
      "geo_region_name" => "%{[geo][region_name]}"
      "geo_city_name" => "%{[geo][city_name]}"
      "geo_location" => "%{[geo][latitude]},%{[geo][longitude]}"
    }
    remove_field => ["geo"]
  }
}
```

- `grok` : 通过正则表达式去处理字符串，比较常用，具体语法见 [grok 文档](https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html)。
- `http` : 与外部 web services/REST APIs 集成。
- `i18n` : 从字段中删除特殊字符。
- `java_uuid` : 生成 UUID 。
- `jdbc_static` : 从远程数据库中读取数据，然后丰富 event 。
- `jdbc_streaming` : 执行 SQL 查询然后将结果存储到指定字段。
- `json` : 解析 json 字符串，生成 field 和 value。

示例：
```
filter {
  json {
    skip_on_invalid_json => true
    source => "message"
  }
}
```
如果输入的 message 字段是 json 字符串如 `"{"a": 1, "b": 2}"`, 那么解析后就会增加两个字段，字段名分别是 a 和 b 。

- `kv` : 解析 key=value 形式的数据。
- `memcached` : 与外部 memcached 集成。
- `metrics` : logstash 在内存中去聚合指标数据。
- `mutate` : 对字段进行一些常规更改。

示例：
```
filter {
  mutate {
    split => ["hostname", "."]
    add_field => { "shortHostname" => "%{hostname[0]}" }
  }

  mutate {
    rename => ["shortHostname", "hostname"]
  }
}
```

- `prune` : 通过黑白名单的方式删除多余的字段。

示例：
```
filter {
  prune {
    blacklist_names => [ "method", "(referrer|status)", "${some}_field" ]
  }
}
```

- `ruby` : 执行 ruby 代码。

示例，解析 `http://example.com/abc?q=haha` 形式字符串中的 query 参数 q 的值 ：
```
filter {
  ruby {
    code => "
      require 'cgi'

      req = event.get('request_uri').split('?')
      query = ''
      if req.length > 1
        query = req[1]

        qh = CGI::parse(query)
        event.set('search_q', qh['q'][0])
      end
    "
  }
}
```

在 ruby 代码中，字段的获取和设置通过 `event.get()` 和 `event.set()` 方法进行操作。

- `sleep` : 休眠指定时间。
- `split` : 拆分字段。
- `throttle` : 限流，限制 event 数量。
- `translate` : 根据指定的字典文件将数据进行对应转换。

示例：
```
filter {
  translate {
    field => "[http_status]"
    destination => "[http_status_description]"
    dictionary => {
      "100" => "Continue"
      "101" => "Switching Protocols"
      "200" => "OK"
      "500" => "Server Error"
    }
    fallback => "I'm a teapot"
  }
}
```

- `truncate` : 将字段内容超出长度的部分裁剪掉。
- `urldecode` : 对 urlencoded 的内容进行解码。
- `useragent` : 解析 user-agent 的内容得到诸如设备、操作系统、版本等信息。

示例：
```
filter {
  # ua_device : 设备
  # ua_name : 浏览器
  # ua_os : 操作系统
  useragent {
    lru_cache_size => 1000
    source => "user_agent"
    target => "ua"
    add_field => {
      "ua_device" => "%{[ua][device]}"
      "ua_name" => "%{[ua][name]}"
      "ua_os" => "%{[ua][os_name]}"
    }
    remove_field => ["ua"]
  }
}
```

- `uuid` : 生成 UUID 。
- `xml` : 解析 XML 格式的数据。


----
## 结语

Logstash 的插件除了本文提到的这些之外还有很多，想要详细的了解每个插件如何使用还是要去查阅官方文档。

得益于 Logstash 的插件体系，你只需要编写一个配置文件，声明使用哪些插件，就可以很轻松的构建数据管道。
