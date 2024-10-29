+++
draft = false
date = 2019-06-27T16:26:45+08:00
title = "Let's Encrypt 配置 HTTPS 免费泛域名证书"
description = "Let's Encrypt 配置 HTTPS 免费泛域名证书"
slug = ""
authors = []
tags = []
categories = ["Uncate"]
externalLink = ""
series = []
disableComments = true
+++

想要使用 HTTPS ，你必须先拥有权威 CA（证书签发机构）签发的证书（对于自签名的证书，浏览器是不认账的）。Let's Encrypt 就是一家权威的 CA 证书签发机构，你可以向他申请免费的证书（一般商业证书的价格比较贵）。

推荐使用 `acme.sh` 这个工具，申请泛域名证书示例：

注意：以下示例中，我的二级域名是 rifewang.club （一般你向云服务商购买的都是二级域名），泛域名是 *.x.rifewang.club 。

1、在系统上安装 acme.sh ，默认安装位置是 `~/.acme.sh` :

```
curl https://get.acme.sh | sh
```

安装要求系统必须已经安装了 cron , crontab , crontabs , vivie-cron 其中任意一个工具，不然会提示你安装失败，没有的话先安装一个即可。

注意：以下操作使用的是 DNS manual mode 的方式。

2、发起 issue 申请获取域名 DNS TXT 记录：

```
acme.sh --issue --force --dns -d <二级域名> -d <泛域名> \
   --yes-I-know-dns-manual-mode-enough-go-ahead-please
```

注意：你必须先将 acme.sh 这个可执行文件的路径添加到系统的环境变量 PATH 中，或者直接在可执行文件目录下执行，否则肯定会提示你 acme.sh command not found 。

--force 强制 issue ，某些情况下你的域名已经验证成功了就会跳过验证，不会生成新的 TXT 记录，所以这里强制执行一下。

--yes-I-know... 这一堆冗长的东西是必须加的，这里就是想提示你 DNS manual mode 的方式不支持自动续签。

issue 之后的结果如图所示：

![](https://raw.githubusercontent.com/RifeWang/images/master/uncate/lets-encrypt1.jpeg)

按照说明你需要分别添加 _acme-challenge.<二级域名> 和  _acme-challenge.<泛域名>  这两个域名的 TXT 类型的域名解析：

![](https://raw.githubusercontent.com/RifeWang/images/master/uncate/lets-encrypt2.jpeg)

之所以要添加域名解析是为了验证你对此域名的所有权。

3、等待 DNS TXT 解析生效，同一条解析重复更新需要避免 DNS 缓存的问题。

4、发起 renew 申请签发并下载证书：

```
acme.sh --renew --force --dns -d <二级域名> -d <泛域名> \   --yes-I-know-dns-manual-mode-enough-go-ahead-please
```

示例结果如图所示：

![](https://raw.githubusercontent.com/RifeWang/images/master/uncate/lets-encrypt3.jpeg)

输出结果除了会告诉你证书签发成功之外，还会在最后说明证书的存放位置，默认是 `~/.acme.sh/<二级域名>/` 这个目录。


5、配置你的证书和密钥，对应的就是 `fullchain.cer` 和 `<二级域名>.key` 这两个文件的内容。不同的情况下，配置的操作是不同的：比如你是在自己的服务器上直接操作 nginx ，那么将配置路径指向正确的证书和密钥地址即可，而如果你使用的是云服务，那么你可能需要做的是上传证书和密钥文件内容。总之，你已经成功获取了 HTTPS 证书。


Let's Encrypt 的泛域名证书有效期是三个月，acme.sh 的 DNS manual mode 方式不支持自动续签，你想要续签就必须重新 issue 然后 renew 操作一遍，我之所以这么做是因为权限受限，当然写个定时脚本任务就行了，也不用我手动操作。

acme.sh 不只一种 mode 方式，其它的方式是有支持自动续签的，并且也接入了主流的云服务商（你只需要配置 apikey 即可），更多内容请参考官网。