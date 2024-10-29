+++
draft = false
date = 2018-04-17T10:50:21+08:00
title = "Docker 入门教程"
description = "Docker 入门教程"
slug = ""
authors = []
tags = ["Docker", "Kubernetes"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

# 一

程序明明在我本地跑得好好的，怎么部署上去就出问题了？如果要在同一台物理机上同时部署多个 node 版本并独立运行互不影响，这又该怎么做？如何更快速的将服务部署到多个物理机上？

“Build once , run anywhere” ，既可以保证环境的一致性，同时又能更方便的将各个环境相互隔离，还能更快速的部署各种服务，这就是 docker 的能力。

## 基本概念

一张图慢慢讲：

![](https://raw.githubusercontent.com/RifeWang/images/master/docker/docker1-1.jpeg)

1、本地开发写好了 code ，首先我们需要通过 build 命令构建 image 镜像，而构建的规则呢，就需要写在这个 dockerfile 文件里。

2、image 镜像是什么？静态的、只读的文件（先不着急，有个基本印象，后面再慢慢讲）。如何更方便的区分不同的镜像呢，通过 tag 命令给镜像打上标签就行了。

3、image 镜像存在哪里？通过 push 命令推送到 repository 镜像仓库，每个仓库可以存放多个镜像。

4、registry 是啥？仓库服务器，所有 repository 仓库都必须依赖于一个 registry 才能提供镜像存储的服务。我们在自己的物理机上安装一个 registry ，这样可以构建自己私有的镜像仓库了。

5、镜像光存到仓库里可没用，还要能部署并运行起来。

6、首先通过 pull 命令将仓库里的镜像拉到服务器上，然后通过 run 命令即可将这个镜像构建成一个 container 容器，容器又是什么？是镜像的运行时，读取镜像里的各种配置文件并如同一个小而独立的服务器一样运行你的各种服务。到这里，你的一个服务就算是部署并运行起来了。

7、数据怎么办？通过 volume 数据卷可以将容器使用的数据挂在到物理机本地，而各个容器之间相互传递处理数据呢，统一通过另一个 volume container 数据卷容器提供数据的服务，数据卷容器也只是一个普通的容器。

8、image 镜像怎么导入导出到本地？通过 save 命令即可导出成压缩包到物理机本地磁盘上，通过 load 命令就可以导入成 docker 环境下的镜像。

9、container 容器的导入导出呢？通过 export 命令同样可以导出到物理机本地磁盘，但是与镜像导出不同的是，这样导出的只是一个容器的快照文件，这就是说它会丢弃所有的历史记录和元数据信息，只记录了当前容器的状态。导入则是 import 命令，但是只能导入为另一个 image 镜像，而不能直接就导入成容器，容器只是一个运行时。

---

# 二

## 镜像

Docker 中的镜像到底是什么？它是一个可供执行的文件系统包，里面包含了运行一个应用程序所需要的代码、库、环境变量和配置文件等等所有内容。

![](https://raw.githubusercontent.com/RifeWang/images/master/docker/docker2-1.jpeg)

镜像是分层的。它是由一个或多个文件系统叠加而成，最底层是 bootfs 即引导文件系统，我们几乎永远不会与这个东西有什么交互，而且当容器启动时 bootfs 会被卸载掉。第二层是 rootfs ，通常是一个操作系统，其包含了程序运行所需的最基本环境，也称之为基础镜像。第三、第四、第N层，是由我们自己指定的其它资源文件。

镜像层层叠加，向下引用依赖，而 docker 使用了联合加载技术同时加载多层文件系统，使我们可以一起看到所有的文件及其资源，仿佛其并没有被分层，而是一个文件系统一样。

镜像是只读的，也就意味着其无法被更改，这正是保证环境一致性的关键原因。

容器则是镜像的运行时，会在镜像最外层加载一层读写层，这样便能进行文件的读写，但其不会对下层镜像的内容进行修改，应用程序只有通过容器才能启动并对外提供服务。

## 构建镜像

首先需要准备我们的项目代码：

```node.js
const express = require('express');

const app = express();
const PORT = 8888;

app.get('/', async (req, res) => {
    res.end(` NODE_NEV : ${process.env.NODE_ENV} \n Contanier port : ${PORT}`);
});

app.listen(PORT);

/*
    构建镜像：docker build --build-arg NODE_ENV=develop -t docker-demo .
    启动容器：docker run --name demo -it -p 9999:8888 docker-demo
*/
```

编写 Dockerfile 文件：

```dockerfile
# 指定基础镜像
FROM node:8.11.1

# MAINTAINER 新版本已经被废弃，用来声明作者信息，使用 LABEL 代替

# LABEL 通过自定义键值对的形式声明此镜像的相关信息，声明后通过 docker inspect 可以看到
LABEL maintainer="rife"
LABEL version="1.0.0"
LABEL description="This is a test image."

# WORKDIR 指定工作目录，若不存在则自动创建，其他指令均会以此作为路径。
WORKDIR /work/myapp/

# ADD <src> <dest>
# 将源文件资源添加到镜像的指定目录中，若是压缩文件会自动在镜像中解压，可以通过 url 指定远程的文件
ADD 'https://github.com/nodejscn/node-api-cn/blob/master/README.md' ./test/

# COPY <src> <dest>
# 同样是复制文件资源，但无法解压，无法通过 url 指定远程文件
# 示例：将本地的当前目录所有文件复制到镜像中 WORKDIR 指定的当前目录
COPY ./ ./

# RUN 构建镜像时执行的命令
RUN npm install

# ARG 指定构建镜像时可传递的参数，与 ENV 配合使用
# 示例：通过 docker build --build-arg NODE_ENV=develop 可灵活指定环境变量
ARG NODE_ENV

# ENV 设置容器运行的环境变量
ENV NODE_ENV=$NODE_ENV

# EXPOSE 暴露容器端口，需要在启动时指定其与宿主机端口的映射
EXPOSE 8888

# CMD 容器启动后执行的命令，只执行最后声明的那条命令，会被 docker run 命令覆盖
CMD ["npm", "start"]

# ENTRYPOINT 容器启动后执行的命令，只执行最后声明的那条命令，不会被覆盖掉
# 任何 docker run 设置的指令参数或 CMD 指令，都将作为参数追加到 ENTRYPOINT 指令的命令之后。
```

在 Dockerfile 中，我们指定了基础镜像、声明了镜像的基础信息，指定了镜像的工作目录，把项目文件添加到了镜像中，指定了环境变量，暴露了容器端口，指定了容器启动后执行的命令。

在复制文件时，我们可以通过 .dockerignore 指定忽略复制到镜像中的文件，用法与 .gitignore 类似。

读者可以仔细阅读上图 Dockerfile 中的注释。

输入指令：

```
docker build --build-arg NODE_ENV=develop -t docker-demo .
```

通过 -t 指定了镜像的标签，--build-arg 指定了 Dockerfile 中的 ARG 声明的变量，也就是 ENV 环境变量，至此我们就成功的构建了自己的镜像。由于网络原因拉取镜像可能会很慢，读者可以使用 DaoCloud 提供的加速地址（其官网的加速器就是）。

输入命令：

```
docker run --name demo -it -p 9999:8888 docker-demo
```

通过 --name 指定容器的别名，-p 指定宿主机与容器之间端口的映射，至此我们基于刚刚构建的镜像启动了一个容器，而容器就是镜像的运行时，最后我们在自己的宿主机上访问 localhost:9999 就能连接到 docker 容器内的 web 示例服务了。

镜像是只读的、分层的文件系统，容器是镜像的运行时。重点关注通过 Dockerfile 构建镜像。


---

# 三

现在有了 docker，如果要频繁的更改和测试程序时怎么办，每次都重新打一个新的镜像然后启动容器？

容器只是一个运行时，一旦被杀死，其内部的数据都会被清除，但是我们想要数据被持久化，又该怎么办？

不同的容器之间常常需要共享某些数据，这又该解决呢？

## volume

Volume 翻译为卷，因为基本上用于挂载数据，所以也常常直接称之为数据卷。

所谓的挂载数据卷，实际上就是把宿主机本地的目录文件映射到 docker 容器内部的目录下。也就是说实际的目录文件是存放在本地磁盘上的，docker 容器通过挂载的方式可以直接使用本地磁盘上的文件。

![](https://raw.githubusercontent.com/RifeWang/images/master/docker/docker3-1.jpeg)

如上图所示：

1、Data Volume 数据卷是存放在本地磁盘上，所以数据是持久化的，即使容器被杀死也不会影响数据卷中的数据。
2、不同的容器挂载同一个数据卷就实现了数据的共享。
3、容器对数据卷中操作都是即时的，一个容器改变了数据，那么另一个容器就会即时看到这种改变。

总而言之，挂载数据卷其实就是间接的操作本地磁盘上的数据，所谓间接是因为容器操作的是其内部映射的目录，而不是宿主机本地目录。

## 数据卷容器

如果有多个容器都需要挂载数据卷，难道需要每一个容器都挂载一遍到本地？当然不是。

![](https://raw.githubusercontent.com/RifeWang/images/master/docker/docker3-2.jpeg)

如上图所示，这里引入了数据卷容器（图中的 Data Container ），其实就是一个普通的容器，我们只需要通过数据卷容器挂载（ -v ）一次数据卷，其他需要挂载的容器直接连接（ --volumes-from ）这个数据卷容器就行了，而再不需要知道实际的宿主机本地目录。

数据卷容器是否存在单点故障？也就是说数据卷容器挂了，其它的容器还能挂载并使用数据吗？答案是仍然能正常使用数据，因为数据卷容器本身只是一个数据卷挂载的配置传递的作用，只要其它容器挂载上就会一直有效，不会因为数据卷容器挂了而产生单点故障。

本节简单讲述了数据卷的相关概念，实际操作只需要通过 docker run 命令启动容器时使用 -v（挂载到本地目录）和 --volumes-from（连接到数据卷容器）参数即可。

---

# 四

场景：假设我们有一个 web 应用，需要显示总共连接的次数，同时我们使用另一个 redis 服务去记录这个数值，显然 web 是需要连接到 redis 上的，而在 docker 容器中，每个容器都默认有自己独立的虚拟网络，那么容器之间应该如何连接？

## link

首先我们启动一个 redis 容器，并通过 --name 指定容器名也叫 redis ：

```
docker run --name redis redis
```

然后，启动 web 容器，通过 --link 指定连接的容器并指定这个连接的名称（注意以下指令都是在 docker run 后面添加的部分）：

```
--link  redis:redis_connection
```

而我们的 web 程序中直接使用上面定义的连接名 redis_connetion 即可：

```node.js
const express = require('express');
const Redis = require('ioredis');
const redis = new Redis({
    port: 6379,
    host: 'redis'  // --net 自定义网络，使用别名
    // host: 'redis_connection'  // --link [container]:[alias]
    // host: 'localhost'  // --net=container:[container-name] 使用指定容器的网络
});
const app = express();
const PORT = 8888;

app.get('/', async (req, res) => {
    try {
        const r = await redis.incr('count');
        res.end(` count: ${r} \r\n`);
    } catch (error) {
        console.log(error);
    }
});

app.listen(PORT);
```

这样 web 容器便可以连接上 redis 容器了，如下图所示：

![](https://raw.githubusercontent.com/RifeWang/images/master/docker/docker4-1.jpeg)

使用 link 方法，其会在容器启动时（容器每次启动都会默认配置不同的虚拟网络）找到连接的目标容器并在本容器内部设置环境变量并修改 /etc/hosts 文件，这也是我们可以直接使用连接别名而不用指定具体 IP 地址的原因。

但是，不建议使用这种方式，同时这种方式也将会在未来被移除。

## net

1、net 方式一，如下图所示：

![](https://raw.githubusercontent.com/RifeWang/images/master/docker/docker4-2.jpeg)

我们先将 redis 容器的端口暴露到本地宿主机，然后在 web 中指定本地宿主机具体的 IP 地址，这样也可以实现连接，但是需要注意的是，在 web 中不能直接使用 localhost ，因为前面已经提到了，每个容器都有自己独立的虚拟网络，使用 localhost 将会指向的是这个容器内部，而不是宿主机。

这种方式，我们也可以看到，很麻烦，一方面 redis 需要暴露端口，另一方面还必须知道宿主机具体的 IP 地址。


2、net 方式二，如下图所示：

![](https://raw.githubusercontent.com/RifeWang/images/master/docker/docker4-3.jpeg)

这里与前一种方式不同的是，我们直接通过 --net host 指定容器直接使用宿主机网络，这样在 web 中就可以直接通过 localhost 连接到 redis 了，不用知道宿主机具体的 IP 地址，对比上一种方式看似有一点小的改进。

但是这种方式的问题在于，对于 MacOS 系统无法使用，因为在 MacOS 上 Docker 仍然是跑在一层虚拟机中的，这种方式目前还无法穿透这层虚拟机直接将 localhost 映射到宿主机本地，同时，直接使用宿主机网络，容器其实会全部暴露出来，存在安全隐患，因此也不建议使用这种方式。

3、net 方式三，如下图所示：

![](https://raw.githubusercontent.com/RifeWang/images/master/docker/docker4-4.jpeg)

这里通过 --net container 的方式直接指定 web 使用与 redis 相同的网络，这样既避免了无谓的端口暴露，同时又能保持容器与宿主机之间的隔离，这种方式是建议使用的。

但是存在需要注意的地方，那就是 --net container 指定容器网络与 -p 暴露端口不能同时使用，换句话说，本来我们的 web 容器是需要 -p 暴露端口到宿主机，这样我们才能在本地访问到 web 服务，但是因为我们已经使用了 --net container 指定其使用与 redis 相同的网络，所以不能再使用 -p 了，那怎么办？可以在另一个 redis 容器上使用 -p ，将本来应该由 web 直接暴露的端口间接的由 redis 暴露，毕竟此时我们的 web 和 redis 容器都已经使用了同一个网络，所以这样做也是没问题的，但还是有点别扭的。


## 自定义网络

官方在宣告 link 方式将会被移除的同时，推荐的替代方式就是自定义网络。

创建一个简单的自定义网络：

```
docker network create -d bridge my-network
```

将 web 和 redis 容器连接到同一个自定义的网络中，并直接在 web 中的 redis host 指向 redis 容器的别名，即可完成连接，如下图所示：

![](https://raw.githubusercontent.com/RifeWang/images/master/docker/docker4-5.jpeg)

对于自定义网络，我们不仅能够在容器启动时通过 --net 直接指定，还能够在容器已经启动完成后通过：

```
docker network connect [network-name] [container]
```

后续添加进去，这也就意味着我们可以方便快速的完成容器网络的切换与迁移。

通过自定义网络，我们还能够定义更加复杂的网络规则，比如网关、子网、IP 地址范围等等，当然更多的细节还请查阅官方文档。

---

# 五

假设我们现在需要启动多个容器，这些容器又需要进行不同的数据挂载，容器之间也需要相互连接，显然，如果按照传统的方法通过 docker run 指令启动他们将会是非法麻烦的，这里我们就需要用到 docker-compose 进行容器编排。

## docker-compose

这里我们使用一个简单的示例：一个 web 服务，一个 redis 数据库，web 服务挂载本地的数据方便调试，同时也需要连接上 redis 进行操作。

首先在进行编排时，我们将一个大的项目称之为 project ，默认名称为项目文件夹的名称，可以通过设置环境变量 COMPOSE_PROJECT_NAME 改变。在 project 之下，会有多个 service 服务，这是编排的基本单位，比如示例中的 web 和 redis 就是两个不同的 service 。

docker-compose.yml：

```yaml
version: '3'  # 指定 compose 的版本
services:
  web:  # 定义 service

    # build:  # 重新构建镜像
    #   context: .  # 构建镜像的上下文(本地相对路径)
    #   dockerfile: Dockerfile   # 指定 dockerfile 文件
    #   args:  # 构建镜像时使用的环境变量
    #     - NODE_ENV=develop

    container_name: web-container   # 容器名称
    image: docker-demo  # 使用已存在的镜像
    ports:  # 端口映射
      - "9999:8888"
    networks:  # 网络
      - my-network
    depends_on:  # service 之间的依赖
      - redis
    volumes:  # 挂载数据
      - "./:/work/myapp/"
    restart: always   # 重启设置
    env_file:   # 环境变量配置文件, key=value
      - ./docker.env
    environment:   # 设置环境变量, 会覆盖 env 中相同的环境变量
      NODE_ENV: abc
    command: npm run test # 容器启动后执行的指令

  redis:
    container_name: redis-container
    image: redis:latest
    networks:
      - my-network

networks:   # 自定义网络
  my-network:
```

如上所示，首先需要指定 compose 的版本，不同版本之间存在一定的差异，具体的需要查阅官方文档。

然后，在 services 这个 top-level 下面指明各个具体的 service 比如 web 和 redis ，在具体的 service 下面再进行详细的配置：

- build：通过 dockerfile 重新构建镜像
- container_name：指定容器的名称
- image：直接使用已存在的镜像
- ports：设置端口映射
- networks：设置容器所在的网络
- depends_on：设置依赖关系
- volumes：设置数据挂载
- restart：设置重启
- env_file：设置环境变量的集中配置文件
- environment：同样是设置环境变量
- command：容器启动后执行的指令


在具体 service 下指定的 networks 必须对应存在于 top-level 的 networks 中，名称可以随意取，所有具有相同 networks 的 service 也就可以进行相互连接，这样就是一个定义网络。

通过 depends_on 设置的依赖关系会决定容器启动的先后顺序，在示例中，由于我们指定了 web 是依赖于 redis 的，所以会启动 redis 之后再启动 web ，但是这里的判断标准是容器运行了就继续启动下一个，如果你想更好的控制启动顺序，可以使用 wait-for-it 或者 dockerize 等官方推荐的第三方开源工具。

至于 volumes ，你可以使用传统挂载设置（示例中就是的），也可以通过自命名的方法，但是如果使用了自命名，其与 networks 类似，必须对应存在于 top-level 的 volumes 之中。

对于环境变量，既可以通过 environment 单独设置，也可以将所有的环境变量集中配置到 env 文件中，然后通过 env_file 引用。

以上就是一些简单且常用的配置。配置完成之后，通过：

```
docker-compose up
```

就可以一次启动所有容器了，启动完成后同样可以通过 compose 的其他指令诸如：pause、unpause、start、stop、restart、kill、down 等等进行其他操作。

写到这里，其实我们已经完成了从构建镜像到容器编排整个流程，这里先告一段落。但是目前我们所基于的却一直是单主机环境，而对于多主机等更复杂的环境下如何快速方便的满足生产上的各种需求，我们就不得不提到 swarm 和 kubernetes（简称 k8s ），从目前来看，k8s 已然成为了主流，后续有机会将会围绕 k8s 写一写系列文章。