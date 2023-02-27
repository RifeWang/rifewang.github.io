+++
draft = false
date = 2019-12-15T13:39:47+08:00
title = "使用 Makefile 构建指令集"
description = "使用 Makefile 构建指令集"
slug = ""
authors = []
tags = []
categories = ["Uncate"]
externalLink = ""
series = []
disableComments = true
+++

# 使用 Makefile 构建指令集

`make` 是一个历史悠久的构建工具，通过配置 `Makefile` 文件就可以很方便的使用你自己自定义的各种指令集，且与具体的编程语言无关。
例如配置如下的 `Makefile` :
```
run dev:
	NODE_ENV=development nodemon server.js
```

这样当你在命令行执行 `make run dev` 时其实就会执行 `NODE_ENV=development nodemon server.js` 指令。

使用 `Makefile` 构建指令集可以很大的提升工作效率。

## Makefile 基本语法
```
<target>: <prerequisites>
    <commands>
```

`target` 其实就是执行的目标，`prerequisites` 是执行这条指令的前置条件，`commands` 就是具体的指令内容。

示例：
```
build: clean
	go build -o myapp main.go

clean:
	rm -rf myapp
```
这里的 `build` 有一个前置条件 `clean` ，意思就是当你执行 `make build` 时，会先执行 `clean` 的指令内容 `rm -rf myapp` ，然后再执行 `build` 的内容 `go build -o myapp main.go` 。

## 变量
自定义变量，示例：
```
APP=myapp

build: clean
	go build -o ${APP} main.go

clean:
	rm -rf ${APP}
```

## PHONY
上例中的定义了 `target` 目标有 `build` 和 `clean` ，如果当前目录中正好有一个文件叫做 `build` 或 `clean`，那么其指令内容不会执行，这是因为 make 会把 `target` 视为文件，只有当文件不存在或发生改变时才会去执行命令。

为了解决这个问题，我们需要使用 `PHONY` 声明 `target` 其实是伪目标：
```
APP=myapp

.PHONY: build
build: clean
	go build -o ${APP} main.go

.PHONY: clean
clean:
	rm -rf ${APP}
```

多个 PHONY 也可以统一声明在一行中:
```
.PHONY: build clean
```

## 递归的目标
假设我们的工程目录结构如下：
```
~/project

├── main.go
├── Makefile
└── mymodule/
      ├── main.go
      └── Makefile
```
文件根目录下还有一个文件夹 `mymodule`，它可能是一个单独的模块，也需要打包构建，并且定义有自己的 `Makefile` :

```
# ~/project/mymodule/Makefile

APP=module

build:
	go build -o ${APP} main.go
```

现在当你处于项目的根目录时，如何去执行 mymodule 子目录下定义的 `Makefile` 呢？

使用 `cd` 命令也可以，不过我们有其它的方式去解决这个问题：使用 `-C` 标志和特定的 `${MAKE}` 变量。

修改项目根目录中的 `Makefile` 为：
```
APP=myapp

.PHONY: build
build: clean
	go build -o ${APP} main.go

.PHONY: clean
clean:
	rm -rf ${APP}


.PHONY: build-mymodule
build-mymodule:
	${MAKE} -C mymodule build
```
这样，当你执行 `make build-mymodule` 时，其将会自动切换到 `mymodule` 目录，并且执行 `mymodule` 目录下的 `Makefile` 中定义的 `build` 指令。

## shell 输出作为变量
我们可以把 shell 中执行的指令的输出作为变量：
```
V=$(shell go version)

gv:
	echo ${V}
```
这里执行 `make gv` 就会先执行 `go version` 指令然后把输出的内容赋值给变量 V 。

## 判断语句
假设我们的指令依赖于环境变量 `ENV` ，我们可以使用一个前置条件去检查是否忘了输入 `ENV` ：
```
.PHONY: run
run: check-env
	echo ${ENV}

check-env:
ifndef ENV
    $(error ENV not set, allowed values - `staging` or `production`)
endif
```
这里当我们执行 `make run` 时，因为有前置条件 `check-env` 会先执行前置条件中的内容，指令内容是一个判断语句，判断 `ENV` 是否未定义，如果未定义，则会抛出一个错误，错误提示就是 `error` 后面的内容。

## 帮助提示
添加 `help` 帮助提示：
```
.PHONY: build
## build: build the application
build: clean
    @echo "Building..."
    @go build -o ${APP} main.go

.PHONY: run
## run: runs go run main.go
run:
	go run -race main.go

.PHONY: clean
## clean: cleans the binary
clean:
    @echo "Cleaning"
    @rm -rf ${APP}

.PHONY: setup
## setup: setup go modules
setup:
	@go mod init \
		&& go mod tidy \
		&& go mod vendor

.PHONY: help
## help: prints this help message
help:
	@echo "Usage: \n"
	@sed -n 's/^##//p' ${MAKEFILE_LIST} | column -t -s ':' |  sed -e 's/^/ /'
```

这样当你执行 `make help` 时，就是打印如下的提示内容：
```
Usage:

  build   build the application
  run     runs go run main.go
  clean   cleans the binary
  setup   setup go modules
  help    prints this help message
```

## 参考资料
- [https://danishpraka.sh/2019/12/07/using-makefiles-for-go.html](https://danishpraka.sh/2019/12/07/using-makefiles-for-go.html)
- [http://www.ruanyifeng.com/blog/2015/02/make.html](http://www.ruanyifeng.com/blog/2015/02/make.html)
- [https://www.gnu.org/software/make/manual/make.html](https://www.gnu.org/software/make/manual/make.html)
