# 为什么你应该使用 Go module proxy


自从 Go v1.11 版本之后 Go modules 成了官方的包管理方式，与此同时还有一个 `Go module proxy` ，它到底是个什么东西？顾名思义，其实就是个代理，所有的模块和依赖库都可以从这个代理上下载。

`Go module proxy` 到底有何特别之处？我们为什么应该使用它？

使用 Go modules ，如果你添加了新的依赖项或者构建了自己的模块，那么它将会基于 go.mod 文件下载（ go get ）所有的依赖项并且缓存起来。你可以使用 vendor 目录（将依赖项置于此目录下）以绕过缓存，同时通过 -mod=vendor 标记就可以指定使用 vendor 目录下的依赖项进行构建。然而这么做并不好。


## 01

使用 vendor 目录有哪些问题：

- vendor 目录不再是 go 命令的默认项，你必须通过 -mode=vendor 指定。
- vendor 目录占用了太多的空间，克隆时也会花费大量时间，尤其是 CI/CD 的效率很低。
- vendor 更新依赖项很难 review ，而依赖项又常常与业务逻辑紧密关联，我们很难去回顾到底发生了哪些变化。


那么不使用 vendor 目录又会如何呢？这时我们又将面临如下问题：

- go 将尝试从源库下载依赖项，但是源库存在被删除的风险。
- VCS（版本控制系统，如 github.com）可能会挂掉或无法使用，这时你也无法构建你的项目。
- 有些公司的内部网络对外隔离，不使用 vendor 目录对他们来说也不行。
- 依赖库的所有者可能通过推送相同版本的恶意内容进行破坏。要防止这种情况发生，需要将 `go.sum` 和 `go.mod` 文件一起存储。
- 某些依赖项可能会使用与 git 不同的 VCS ，如 hg（Mercurial）、bzr（Bazaar）、svn（Subversion），因此你不得不安装这些其他的工具，很烦。
- `go get` 需要获取 `go.mod` 中每个依赖项的源代码以解决传递依赖，这显著减慢了整个构建过程，因为它必须下载（`git clone`）每个存储库以获取单个文件。


如何解决上述这一系列的问题？答案是使用 `Go module proxy` 。


## 02

默认情况下，go 命令直接从 VCS 下载模块。环境变量 `GOPROXY` 指定使用 `Go module proxy` 以进一步控制下载源。

通过设置 GOPROXY ，你将会解决上述的所有问题：

- Go module proxy 默认缓存并永久存储所有依赖项（不可变存储），你不再需要 vendor 目录。
- 摆脱了 vendor 目录意味着项目不再占用 repository 空间，提高了效率。
- 由于依赖库以不可变的形式存储在代理中，即使源库删除，代理中的库也不会被删除，这保障依赖库的使用者。
- 一旦模块被存储在 Go proxy 中，就无法被覆盖或者删除，换句话说使用相同版本注入恶意代码的行为攻击将不再奏效。
- 你不再需要任何 VCS 工具来下载依赖项，因为你只需要通过 http 与 Go proxy 建立连接。
- 下载和构建将会快很多，官方团队测试的结果是快了三到六倍。
- 你可以轻松管理自己的代理，这可以让你更好的控制构建管道的稳定性。

综上所述，你绝对应该使用 `Go module proxy` 。


## 03

如何使用 `Go module proxy` ？


你需要设置环境变量 `GOPROXY` ：


1、如果 GOPROXY 未设置、为空、或者设置为 direct ，则 go get 将直连 VCS （如 github.com）：

```
GOPROXY=""
GOPROXY=direct
```

如果设置为 off ，则表示不允许使用网络：

```
GOPROXY=off
```

2、你可以使用任意一个公共的代理 :

```
GOPROXY=https://proxy.golang.org # 谷歌官方，大陆地区被墙了
GOPROXY=https://goproxy.io # 个人开源
GOPROXY=https://goproxy.cn # 大陆地区建议使用，七牛云托管
```

3、你可以基于开源方案实现本地部署：

- Athens: https://github.com/gomods/athens
- goproxy: https://github.com/goproxy/goproxy
- THUMBAI: https://thumbai.app/
通过这种方式你可以构建一个公司的内部代理，与外网隔离。


4、你可以购买商业产品：

- Artifactory: https://jfrog.com/artifactory/


5、你可以使用 file:/// URL ，文件系统路径也是可以直接使用的。


## 04

Go v1.13 版本的相关更改：

- `GOPROXY` 可以设置为以逗号分隔的列表，如果某个地址失败将会依次尝试后面的地址。
- `GOPROXY` 默认启动，默认值将会是 https://proxy.golang.org,direct 。direct 之后的地址将会被忽略。
- `GOPRIVATE` 环境变量将会被推出，用于绕过 `GOPROXY` 中的特定路径，尤其是公司中的私有模块。


---

*相关资料：*

- *https://github.com/golang/go/wiki/Modules*
- *https://proxy.golang.org/*
