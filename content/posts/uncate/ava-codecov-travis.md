+++
draft = false
date = 2019-12-21T13:36:00+08:00
title = "给你的库加上酷炫的小徽章"
description = "给你的库加上酷炫的小徽章"
slug = ""
authors = []
tags = ["github", "node.js"]
categories = []
externalLink = ""
series = []
disableComments = true
+++

# 给库加上酷炫的小徽章 & ava、codecov、travis 示例

GitHub 很多开源库都会有几个酷炫的小徽章，比如：

![npm (tag)](https://img.shields.io/npm/v/io-memcached/latest) ![Travis (.com)](https://img.shields.io/travis/com/rifewang/io-memcached) ![Codecov](https://img.shields.io/codecov/c/github/rifewang/io-memcached) ![NPM](https://img.shields.io/npm/l/io-memcached) ![GitHub last commit](https://img.shields.io/github/last-commit/rifewang/io-memcached)

这些是怎么加上去的呢？

## Shields.io
首先这些徽章可以直接去 [shields.io](https://shields.io/) 网站自动生成。

比如：

![npm (tag)](https://img.shields.io/npm/v/io-memcached/latest)

就是 `version` 这一类里的一种图标，选择 npm 一栏填入包名，然后复制成 Markdown 内容，就会得到诸如：
```
![npm (tag)](https://img.shields.io/npm/v/io-memcached/latest)
```
直接粘贴在 .md 文件中就可以使用了，最后展现的就是这个图标。

当然还有其他很多徽章都任由你挑选，不过某些徽章是需要额外进行一些配置，比如这里的 ![Travis (.com)](https://img.shields.io/travis/com/rifewang/io-memcached) (自动构建通过) 和 ![Codecov](https://img.shields.io/codecov/c/github/rifewang/io-memcached) (测试覆盖率)。

## AVA
谈到测试覆盖率必须先有单元测试，本文使用 `ava` 作为示例，`ava` 是一个 js 测试库，强烈推荐你使用它。

1、安装
```
npm init ava
```

2、使用示例

编写 `test.js` 文件：
```
import test from 'ava'
import Memcached from '../lib/memcached';


test.before(t => {
	const memcached = new Memcached(['127.0.0.1:11211'], {
        pool: {
            max: 2,
            min: 0
        },
        timeout: 5000
    });
    t.context.memcached = memcached;
});

test('memcached get/set', async t => {
    try {
        t.plan(3);

        const key = 'testkey';
        const testdata = 'testest\r\n\stese';
        const r = await t.context.memcached.set(key, testdata);
        t.is(r, 'STORED');

        const g = await t.context.memcached.get(key, testdata);
        t.is(g, testdata);

        const dr = await t.context.memcached.del(key);
        t.is(dr, 'DELETED');
    } catch (error) {
        t.fail(error.message);
    }
});

test('unit test title', t => {
    t.pass();
});
```
说明：
- `ava` 本身就支持很多 es6 及以上的特性，你不用另外再使用 babel 。

- `test.before` 就是一个钩子，你可以通过 `context` 向后传递变量并使用。

- `test('title', t => {})` 函数构造我们的单元测试，每项测试的名称可以自己定义，使用非常方便，多个 test 之间是并发执行的，如果你需要依次执行则使用 `test.serial()`。
- `t.plan()` 声明了每项测试中应该有几次断言。
- `t.is()` 则是进行断言判断。
- `t.fail()` 声明单项测试不通过。
- `t.pass()` 声明单项测试通过。

当然这里只是展示了很少的几个用法，更多详细的内容看官方文档。

### coverage
单元测试有了，但是还没有测试覆盖率，为此我们还需要 `nyc` 。
```
npm install --save-dev nyc
```
修改 `package.json` 文件:
```
{
	"scripts": {
		"test": "nyc ava"
	}
}
```
获取测试覆盖率时会生成相关的文件，我们在 `.gitignore` 中忽略它们即可:
```
.nyc_output
coverage*
```

当我们再执行 `npm test` 时，其就会执行单元测试，并且获取测试覆盖率，结果类似于:
```
$ npm test

> nyc ava


  4 tests passed

--------------|----------|----------|----------|----------|-------------------|
File          |  % Stmts | % Branch |  % Funcs |  % Lines | Uncovered Line #s |
--------------|----------|----------|----------|----------|-------------------|
All files     |    72.07 |    63.37 |    79.49 |    72.07 |                   |
 memcached.js |    72.59 |    64.37 |    74.19 |    72.59 |... 13,419,428,439 |
 utils.js     |       68 |    57.14 |      100 |       68 |... 70,72,73,75,76 |
--------------|----------|----------|----------|----------|-------------------|
```

## Codecov
测试覆盖率也有了，但这只是本地的，我们还不能生成 ![Codecov](https://img.shields.io/codecov/c/github/rifewang/io-memcached) 这种徽章。

为此，本文选择了 [codecov](https://codecov.io/) 平台，我们需要使用 GitHub 账号登录 [codecov](https://codecov.io/) 并关联我们的 repository 库，同时我们需要生成一个 token 令牌以便后续使用。

安装 `codecov` :
```
npm install --save-dev codecov
```

在 `package.json` 文件中增加一个上报测试覆盖率的脚本:
```
{
	"scripts": {
		"report-coverage": "nyc report --reporter=text-lcov > coverage.lcov && codecov"
	}
}
```
上报测试覆盖率的结果给 [codecov](https://codecov.io/) 是需要权限的，这里的权限需要配置环境变量 `CODECOV_TOKEN=<token>` ，token 就是刚刚在 [codecov](https://codecov.io/) 平台上设置的令牌，然后执行 `npm run report-coverage` 才会成功。

## Travis-ci
本文使用 [travis-ci](https://travis-ci.com/) 来做持续集成，同样的你需要使用 GitHub 账号登录 [travis-ci](https://travis-ci.com/) 并关联我们的 repository 库。

编写 `.travis.yml` 配置文件:
```
language: node_js
node_js:
  - "12"

sudo: required

before_install: sudo apt-get install libevent-dev -y

install:
  - wget -O memcached.tar.gz http://memcached.org/files/memcached-1.5.20.tar.gz
  - tar -zxvf memcached.tar.gz
  - cd memcached-1.5.20
  - ./configure && make && sudo make install
  - memcached -d

script:
    - npm ci && npm test && npm run report-coverage
```

- `language` : 声明语言环境，这里的 `node_js` 还声明了版本。
- `sudo` : 声明在 CI 的虚拟环境中是否需要管理员权限。
- `before_install` : 安装额外的系统依赖。
- `install` : 示例中另外安装了 memcached 并在后台启动，因为本文的测试需要。
- `script` : 声明 CI 执行的脚本命令。

由于我们在 [travis-ci](https://travis-ci.com/) 上执行 `npm run report-coverage` 向 [codecov](https://codecov.io/) 上报测试覆盖率时需要其权限，因此还需要在 [travis-ci](https://travis-ci.com/) 的 `Settings` 中设置环境变量 `CODECOV_TOKEN` 。

最后，当我们向 GitHub 库中提交了新的内容后，就会触发 CI 流程，虚拟化环境、安装依赖、执行命令等等，CI 通过后就可以得到 ![Travis (.com)](https://img.shields.io/travis/com/rifewang/io-memcached) 徽章了。

## 结语
[shields.io](https://shields.io/) 徽章有多种，根据你的需要进行相应的配置即可，本文使用了 [codecov](https://codecov.io/) 和 [travis-ci](https://travis-ci.com/) 作为示例，但是还有很多其他的平台任由你选。