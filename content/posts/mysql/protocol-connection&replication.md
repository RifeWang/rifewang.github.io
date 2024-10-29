+++
draft = false
date = 2020-12-20T00:00:00+08:00
title = "解读 MySQL Client/Server Protocol: Connection & Replication"
description = ""
slug = ""
authors = []
tags = ["Database", "MySQL"]
categories = ["Database"]
externalLink = ""
series = []
disableComments = true
+++

# 解读 MySQL Client/Server Protocol: Connection & Replication

MySQL 客户端与服务器之间的通信基于特定的 TCP 协议，本文将会详解其中的 Connection 和 Replication 部分，这两个部分分别对应的是客户端与服务器建立连接、完成认证鉴权，以及客户端注册成为一个 slave 并获取 master 的 binlog 日志。

## Connetcion Phase

MySQL 客户端想要与服务器进行通信，第一步就是需要成功建立连接，整个过程如下图所示：

![connection-phase](https://raw.githubusercontent.com/RifeWang/images/master/mysql-protocol-connection-phase.png)

1. client 发起一个 TCP 连接。
2. server 响应一个 [`Initial Handshake Packet`](#Initial_Handshake_Packet)（初始化握手包），内容会包含一个默认的认证方式。
3. 这一步是可选的，双方建立 SSL 加密连接。
4. client 回应 [`Handshake Response Packet`](#Handshake_Response_Packet)，内容需要包括用户名和按照指定方式进行加密后的密码数据。
5. server 响应 [`OK_Packet`](#OK_ERR) 确认认证成功，或者 [`ERR_Packet`](#OK_ERR) 表示认证失败并关闭连接。

### Packet

一个 Packet 其实就是一个 TCP 包，所有包都有一个最基本的结构：

![packet](https://raw.githubusercontent.com/RifeWang/images/master/mysql-protocol-packet.png)

如上图所示，所有包都可以看作由 header 和 body 两部分构成：第一部分 header 总共有 4 个字节，3 个字节用来标识 body 即 payload 的大小，1 个字节记录 sequence ID；第二部分 body 就是 payload 实际的负载数据。

由于 payload length 只有 3 个字节来记录，所以一个 packet 的 payload 的大小不能超过 2^24 = 16 MB ，示例：

![packet-example](https://raw.githubusercontent.com/RifeWang/images/master/mysql-protocol-packet-example.png)

Packet :
- 当数据不超过 16 MB 时，准确来说是 payload 的大小不超过 2^24-1 Byte（三个字节所能表示的最大整数 0xFFFFFF），发送一个 packet 就够了。
- 当数据大小超过了 16 MB 时，就需要把数据切分成多个 packet 传输。
- 当数据 payload 的刚好是 2^24-1 Byte 时，一个包虽然足够了，但是为了表示数据传输完毕，仍然会多传一个 payload 为空的 packet 。

<br />

Sequence ID：包的序列号，从 0 开始递增。在一个完整的会话过程中，每个包的序列号依次加一，当开始一个新的会话时，序列号重新从 0 开始。例如：在建立连接的阶段，server 发送 Initial Handshake Packet（ Sequence ID 为 0 ），client 回应 Handshake Response Packet（ Sequence ID 为 1 ），server 再响应 OK_Packet 或者 ERR_Packet（ Sequence ID 为 2 ），然后建立连接的阶段就结束了，再有后续的命令数据，包的 Sequence ID 就重新从 0 开始；在命令阶段（client 向 server 发送增删改查这些都属于命令阶段），一个命令的请求和响应就可以看作一个完整的会话过程，比如 client 先向 server 发送了一个查询请求，然后 server 对这个查询请求进行了响应，那么这一次会话就结束了，下一个命令就是新的会话，Sequence ID 也就重新从 0 开始递增。


### Initial Handshake Packet <span id="Initial_Handshake_Packet"></span>

建立连接时，当客户端发起一个 TCP 连接后，MySQL 服务端就会回应一个 `Initial Handshake Packet` ，这个初始化握手包的数据格式如下图所示：

![handshakeV10](https://raw.githubusercontent.com/RifeWang/images/master/mysql-protocol-handshakeV10.png)

这个图从上往下依次是：
- 1 个字节的整数，表示 handshake protocol 的版本，现在都是 10 。
- 以 NUL（即一个字节 0x00）结尾的字符串，表示 MySQL 服务器的版本，例如 `5.7.18-log` 。
- 4 个字节的整数，表示线程 id，也是这个连接的 id。
- 8 个字节的字符串，`auth-plugin-data-part-1` 后续密码加密需要用到的随机数的前 8 位。
- 1 个字节的填充位。
- 2 个字节的整数，`capability_flags_1` 即 `Capabilities Flags` 的低位 2 位字节。
- 1 个字节的整数，表示服务器默认的字符编码格式，比如 `utf8_general_ci`。
- 2 个字节的整数，服务器的状态标识。
- 2 个字节的整数，`capability_flags_2` 即 `Capabilities Flags` 的高位 2 位字节。
- 1 个字节的整数，如果服务器具有 CLIENT_PLUGIN_AUTH 的能力（其实就是能够进行客户端身份验证，基本都支持），那么传递的是 `auth_plugin_data_len` 即加密随机数的长度，否则传递的是 0x00 。
- 10 个字节的填充位，全部是 0x00 。
- 由 `auth_plugin_data_len` 指定长度的字符串，`auth-plugin-data-part-2` 加密随机数的后 13 位。
- 如果服务器具有 CLIENT_PLUGIN_AUTH 的能力（其实就是能够进行客户端身份验证，基本都支持），那么传递的是 `auth_plugin_name` 即用户认证方式的名称。

对于 MySQL 5.x 版本，默认的用户身份认证方式叫做 `mysql_native_password`（对应上面的 `auth_plugin_name`），这种认证方式的算法是：
```
SHA1( password ) XOR SHA1( "20-bytes random data from server" <concat> SHA1( SHA1( password ) ) )
```
其中加密所需的 20 个字节的随机数就是 `auth-plugin-data-part-1`（ 8 位数）和 `auth-plugin-data-part-2`（ 13 位中的前 12 位数）组成。

注意：MySQL 使用的小端字节序。

看到这，你可能还对 `Capabilities Flags` 感到很困惑。

#### Capabilities Flags

`Capabilities Flags` 其实就是一个功能标志，用来表明服务端和客户端支持并希望使用哪些功能。为什么需要这个功能标志？因为首先 MySQL 有众多版本，每个版本可能支持的功能有区别，所以服务端需要表明它支持哪些功能；其次，对服务端来说，连接它的客户端可以是各种各样的，这些客户端希望使用哪些功能也是需要表明的。

`Capabilities Flags` 一般是 4 个字节的整数：

![capability](https://raw.githubusercontent.com/RifeWang/images/master/mysql-protocol-capability.png)

如上图所示，每个功能都独占一个 bit 位。

`Capabilities Flags` 通常都是多个功能的组合表示，例如要表示 `CLIENT_PROTOCOL_41`、`CLIENT_PLUGIN_AUTH`、`CLIENT_SECURE_CONNECTION` 这三个功能，那么就把他们对应的 `0x00000200`、`0x00080000`、`0x00008000` 进行比特位或运算就能得到最终的值 `0x00088200` 也就是最终的 `Capabilities Flags`。

根据 `Capabilities Flags` 判断是否支持某个功能，例如 `Capabilities Flags` 的值是 `0x00088200`，要判断它是否支持 `CLIENT_SECURE_CONNECTION` 的功能，则直接进行比特位与运算即可，即 `Capabilities Flags` & `CLIENT_SECURE_CONNECTION` == `CLIENT_SECURE_CONNECTION` 。


### Handshake Response Packet <span id="Handshake_Response_Packet"></span>

建立连接的过程中，当客户端收到了服务端的 `Initial Handshake Packet` 后，需要向服务端回应一个 `Handshake Response Packet` ，包的数据格式如下图所示：

![handshake_response_41](https://raw.githubusercontent.com/RifeWang/images/master/mysql-protocol-handshakeResponse41.png)

依次是：
- 4 个字节的整数，`Capabilities Flags`，一定要设置 `CLIENT_PROTOCOL_41`，对于 MySQL 5.x 版本，使用默认的身份认证方式，还需要对应的设置 `CLIENT_PLUGIN_AUTH` 和 `CLIENT_SECURE_CONNECTION`。
- 4 个字节的整数，包大小的最大值，这里指的是命令包的大小，比如一条 SQL 最多能多大。
- 1 个字节的整数，字符编码方式。
- 23 个字节的填充位，全是 0x00。
- 以 NUL（0x00）结尾的字符串，登录的用户名。
- `CLIENT_PLUGIN_AUTH_LENENC_CLIENT_DATA` 一般不使用。
- 1 个字节的整数，`auth_response_length`，密码加密后的长度。
- `auth_response_length` 指定长度的字符串，密码与随机数加密后的数据。
- 如果 `CLIENT_CONNECT_WITH_DB` 直接指定了连接的数据库，则需要传递以 NUL（0x00）结尾的字符串，内容是数据库名。
- `CLIENT_PLUGIN_AUTH` 一般都需要，默认方式需要传递的值就是 `mysql_native_password` 。

可以看到，`Handshake Response Packet` 与 `Initial Handshake Packet` 其实是相对应的。


### `OK_Packet` & `ERR_Packet` <span id="OK_ERR"></span>

`OK_Packet` 和 `ERR_Packet` 是 MySQL 服务端通用的响应包。

![OK_Packet](https://raw.githubusercontent.com/RifeWang/images/master/mysql-protocol-ok_packet.png)

MySQL 5.7.5 版本以后，`OK_Packet` 还包含了 `EOF_Packet`（用来显示警告和状态信息）。区分 `OK_Packet` 和 `EOF_Packet`:
- OK: header = 0x00 and length of packet > 7
- EOF: header = 0xfe and length of packet < 9

MySQL 5.7.5 版本之前，`EOF_Packet` 是一个单独格式的包：

![EOF_Packet](https://raw.githubusercontent.com/RifeWang/images/master/mysql-protocol-EOF_Packet.png)

如果身份认证通过、连接建立成功，返回的 `OK_Packet` 就会是：
```
0x07 0x00 0x00 0x02 0x00 0x00 0x00 0x02 0x00 0x00 0x00
```

 <br />


如果连接失败，或者出现错误则会返回 `ERR_Packet` 格式的包：

![ERR_Packet](https://raw.githubusercontent.com/RifeWang/images/master/mysql-protocol-err_packet.png)


## Replication

想要获取到 master 的 binlog 吗？只要你对接实现 replication 协议即可。

![replication](https://raw.githubusercontent.com/RifeWang/images/master/mysql-protocol-replication.png)

1. client 与 server 之间成功建立连接、完成身份认证，这个过程就是上文所述的 connection phase 。
2. client 向 server 发送 `COM_REGISTER_SLAVE` 包，表明要注册成为一个 slave ，server 响应 `OK_Packet` 或者 `ERR_Packet`，只有成功才能进行后续步骤。
3. client 向 server 发送 `COM_BINLOG_DUMP` 包，表明要开始获取 binlog 的内容。
4. server 响应数据，可能是：
    - `binlog network stream`（ binlog 网络流）。
    - `ERR_Packet`，表示有错误发生。
    - `EOF_Packet`，如果 `COM_BINLOG_DUMP` 中的 flags 设置为了 0x01 ，则在 binlog 没有更多新事件时发送 `EOF_Packet`，而不是阻塞连接继续等待后续 binlog event 。

### `COM_REGISTER_SLAVE`

客户端向 MySQL 发送 `COM_REGISTER_SLAVE` ，表明它要注册成为一个 slave，包格式如下图：
![com_register_slave](https://raw.githubusercontent.com/RifeWang/images/master/mysql-protocol-com_register_slave.png)

除了 1 个字节的固定内容 0x15 和 4 个字节的 server-id ，其他内容通常都是空或者忽略，需要注意的是这里的 user 和 password 并不是登录 MySQL 的用户名和密码，只是 slave 的一种标识而已。


### `COM_BINLOG_DUMP`

注册成为 slave 之后，发送 `COM_BINLOG_DUMP` 就可以开始接受 binlog event 了。

![com_binlog_dump](https://raw.githubusercontent.com/RifeWang/images/master/mysql-protocol-com_binlog_dump.png)

- 1 个字节的整数，固定内容 0x12 。
- 4 个字节的整数，`binlog-pos` 即 binlog 文件开始的位置。
- 2 个字节的整数，`flags`，一般情况下 slave 会一直保持连接等待接受 binlog event，但是当 flags 设置为了 0x01 时，如果当前 binlog 全部接收完了，则服务端会发送 `EOF_Packet` 然后结束整个过程，而不是保持连接继续等待后续 binlog event 。
- 4 个字节的整数，`server-id`，slave 的身份标识，MySQL 可以同时存在多个 slave ，每个 slave 必须拥有不同的 `server-id`。
- 不定长字符串，`binlog-filename`，开始的 binlog 文件名。查看当前的 binlog 文件名和 pos 位置，可以执行 SQL 语句 `show master status` ，查看所有的 binlog 文件，可以执行 SQL 语句 `show binary logs` 。


### Binlog Event

客户端注册 slave 成功，并且发送 `COM_BINLOG_DUMP` 正确，那么 MySQL 就会向客户端发送 binlog network stream 即 binlog 网络流，所谓的 binlog 网络流其实就是源源不断的 binlog event 包（对 MySQL 进行的操作，例如 inset、update、delete 等，在 binlog 中是以一个或多个 binlog event 的形式存在的）。

Replication 的两种方式：
- 异步，默认方式，master 不断地向 slave 发送 binlog event ，无需 slave 进行 ack 确认。
- 半同步，master 向 slave 每发送一个 binlog event 都需要等待 ack 确认回复。

<br />

Binlog 有三种模式：
- `statement` ，binlog 存储的是原始 SQL 语句。
- `row` ，binlog 存储的是每行的实际前后变化。
- `mixed` ，混合模式，binlog 存储的一部分是 SQL 语句，一部分是每行变化。

<br />

Binlog Event 的包格式如下图：

![binlog_event](https://raw.githubusercontent.com/RifeWang/images/master/mysql-protocol-binlog_event.png)

每个 Binlog Event 包都有一个确定的 event header ，根据 event 类型的不同，可能还会有 post header 以及 payload 。

Binlog Event 的类型非常多：
- Binlog Management:
    - `START_EVENT_V3`
    - `FORMAT_DESCRIPTION_EVENT`: MySQL 5.x 及以上版本 binlog 文件中的第一个 event，内容是 binlog 的基本描述信息。
    - `STOP_EVENT`
    - `ROTATE_EVENT`: binlog 文件发生了切换，binlog 文件中的最后一个 event。
    - `SLAVE_EVENT`
    - `INCIDENT_EVENT`
    - `HEARTBEAT_EVENT`: 心跳信息，表明 slave 落后了 master 多少秒（执行 SQL 语句 `SHOW SLAVE STATUS` 输出的 `Seconds_Behind_Master` 字段）。
- Statement Based Replication Events（binlog 为 statement 模式时相关的事件）:
    - `QUERY_EVENT`: 原始 SQL 语句，例如 insert、update ... 。
    - `INTVAR_EVENT`: 基于会话变量的整数，例如把主键设置为了 auto_increment 自增整数，那么进行插入时，这个字段实际写入的值就记录在这个事件中。
    - `RAND_EVENT`: 内部 RAND() 函数的状态。
    - `USER_VAR_EVENT`: 用户变量事件。
    - `XID_EVENT`: 记录事务 ID，事务 commit 提交了才会写入。
- Row Based Replication Events（binlog 为 row 模式时相关的事件）:
    - `TABLE_MAP_EVENT`: 记录了后续事件涉及到的表结构的映射关系。
    - v0 事件对应 MySQL 5.1.0 to 5.1.15 版本
    - `DELETE_ROWS_EVENTv0`: 记录了行数据的删除。
    - `UPDATE_ROWS_EVENTv0`: 记录了行数据的更新。
    - `WRITE_ROWS_EVENTv0`: 记录了行数据的新增。
    - v1 事件对应 MySQL 5.1.15 to 5.6.x 版本
    - `DELETE_ROWS_EVENTv1`: 记录了行数据的删除。
    - `UPDATE_ROWS_EVENTv1`: 记录了行数据的更新。
    - `WRITE_ROWS_EVENTv1`: 记录了行数据的新增。
    - v2 事件对应 MySQL 5.6.x 及其以上版本
    - `DELETE_ROWS_EVENTv2`: 记录了行数据的删除。
    - `UPDATE_ROWS_EVENTv2`: 记录了行数据的更新。
    - `WRITE_ROWS_EVENTv2`: 记录了行数据的新增。
- LOAD INFILE replication（加载文件的特殊场景，本文不做介绍）:
    - `LOAD_EVENT`
    - `CREATE_FILE_EVENT`
    - `APPEND_BLOCK_EVENT`
    - `EXEC_LOAD_EVENT`
    - `DELETE_FILE_EVENT`
    - `NEW_LOAD_EVENT`
    - `BEGIN_LOAD_QUERY_EVENT`
    - `EXECUTE_LOAD_QUERY_EVENT`

想要解析具体某个 binlog event 的内容，只要对照官方文档数据包的格式即可。


## 结语

MySQL Client/Server Protocol 协议其实很简单，就是相互之间按照约定的格式发包，而理解了协议，相信你自己就可以实现一个 lib 去注册成为一个 slave 然后解析 binlog 。

