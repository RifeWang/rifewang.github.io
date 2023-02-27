+++
draft = false
date = 2019-12-23T00:00:00+08:00
title = "GitHub Actions 指南"
description = "GitHub Actions 指南"
slug = ""
authors = []
tags = ["Github", "CICD"]
categories = ["Uncate"]
externalLink = ""
series = []
disableComments = true
+++

# GitHub Actions 指南
GitHub Actions 使你可以直接在你的 GitHub 库中创建自定义的工作流，工作流指的就是自动化的流程，比如构建、测试、打包、发布、部署等等，也就是说你可以直接进行 CI（持续集成）和 CD （持续部署）。

---
## 基本概念
- workflow : 一个 workflow 工作流就是一个完整的过程，每个 workflow 包含一组 jobs 任务。
- job : jobs 任务包含一个或多个 job ，每个 job 包含一系列的 steps 步骤。
- step : 每个 step 步骤可以执行指令或者使用一个 action 动作。
- action : 每个 action 动作就是一个通用的基本单元。

---
## 配置 workflow
workflow 必须存储在你的项目库根路径下的 `.github/workflows` 目录中，每一个 workflow 对应一个具体的 `.yml` 文件（或者 `.yaml`）。

workflow 示例：
```
name: Greet Everyone
# This workflow is triggered on pushes to the repository.
on: [push]

jobs:
  your_job_id:
    # Job name is Greeting
    name: Greeting
    # This job runs on Linux
    runs-on: ubuntu-latest
    steps:
      # This step uses GitHub's hello-world-javascript-action: https://github.com/actions/hello-world-javascript-action
      - name: Hello world
        uses: actions/hello-world-javascript-action@v1
        with:
          who-to-greet: 'Mona the Octocat'
        id: hello
      # This step prints an output (time) from the previous step's action.
      - name: Echo the greeting's time
        run: echo 'The time was ${{ steps.hello.outputs.time }}.'
```

说明：
- 最外层的 `name` 指定了 workflow 的名称。
- `on` 声明了一旦发生了 push 操作就会触发这个 workflow 。
- `jobs` 定义了任务集，其中可以有一个或多个 job 任务，示例中只有一个。
- `runs-on` 声明了运行的环境。
- `steps` 定义需要执行哪些步骤。
- 每个 `step` 可以定义自己的 `name` 和 `id` ，通过 `uses` 可以声明使用一个具体的 `action` ，通过 `run` 声明需要执行哪些指令。
- `${{ }}` 可以使用上下文参数。

上述示例可以抽象为：
```
name: <workflow name>

on: <events that trigger workflows>

jobs:
  <job_id>:
    name: <job_name>
    runs-on: <runner>
    steps:
      - name: <step_name>
        uses: <action>
        with:
          <parameter_name>: <parameter_value>
        id: <step_id>

      - name: <step_name>
        run: <commands>
```

---
## on
`on` 声明了何时触发 workflow ，它可以是：
- 一个或多个 GitHub 事件，比如 push 了一个 commit、创建了一个 issue、产生了一次 pull request 等等，示例：
```
on: [push, pull_request]
```

- 预定的时间，示例（每天零点零分触发）：
```
on:
  schedule:
    - cron:  '0 0 * * *'
```

- 某个外部事件。所谓外部事件触发，简而言之就是你可以通过 REST API 向 GitHub 发送请求去触发，具体请查阅官方文档: [repository-dispatch-event](https://developer.github.com/v3/repos/#create-a-repository-dispatch-event)


配置多个事件，示例：
```
on:
  # Trigger the workflow on push or pull request,
  # but only for the master branch
  push:
    branches:
      - master
  pull_request:
    branches:
      - master
  # Also trigger on page_build, as well as release created events
  page_build:
  release:
    types: # This configuration does not affect the page_build event above
      - created
```

详细文档请参考: [触发事件](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/events-that-trigger-workflows)

---
## jobs
jobs 可以包含一个或多个 job ，如:
```
jobs:
  my_first_job:
    name: My first job
  my_second_job:
    name: My second job
```

如果多个 job 之间存在依赖关系，那么你可能需要使用 `needs` :
```
jobs:
  job1:
  job2:
    needs: job1
  job3:
    needs: [job1, job2]
```
这里的 `needs` 声明了 job2 必须等待 job1 成功完成，job3 必须等待 job1 和 job2 依次成功完成。

每个任务默认超时时间最长为 360 分钟，你可以通过 `timeout-minutes` 进行配置:
```
jobs:
  job1:
    timeout-minutes:
```

---
## runs-on & strategy
`runs-on` 指定了任务的 runner 即执行环境，runner 分两种：GitHub-hosted runner 和 self-hosted runner 。

所谓的 self-hosted runner 就是用你自己的机器，但是需要 GitHub 能进行访问并给与其所需的机器权限，这个不在本文描述范围内，有兴趣可参考 [self-hosted runner](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/using-self-hosted-runners-in-a-workflow) 。

GitHub-hosted runner 其实就是 GitHub 提供的虚拟环境，目前有以下四种:
- `windows-latest` : Windows Server 2019
- `ubuntu-latest` 或 `ubuntu-18.04` : Ubuntu 18.04
- `ubuntu-16.04` : Ubuntu 16.04
- `macos-latest` : macOS Catalina 10.15

比较常见的:
```
runs-on: ubuntu-latest
```

### runs-on 多环境
有时候我们常常需要对多个操作系统、多个平台、多个编程语言版本进行测试，为此我们可以配置一个构建矩阵。

例如：
```
runs-on: ${{ matrix.os }}
strategy:
  matrix:
    os: [ubuntu-16.04, ubuntu-18.04]
    node: [6, 8, 10]
```
示例中配置了两种 os 操作系统和三种 node 版本即总共六种情况的构建矩阵，`${{ matrix.os }}` 是一个上下文参数。


`strategy` 策略，包括：
- `matrix` : 构建矩阵。
- `fail-fast` : 默认为 true ，即一旦某个矩阵任务失败则立即取消所有还在进行中的任务。
- `max-paraller` : 可同时执行的最大并发数，默认情况下 GitHub 会动态调整。

示例：
```
runs-on: ${{ matrix.os }}
strategy:
  matrix:
    os: [macos-latest, windows-latest, ubuntu-18.04]
    node: [4, 6, 8, 10]
    include:
      # includes a new variable of npm with a value of 2 for the matrix leg matching the os and version
      - os: windows-latest
        node: 4
        npm: 2
```
`include` 声明了 os 为 windows-latest 时，增加一个 node 和 npm 分别使用特定的版本的矩阵环境。

与 `include` 相反的就是 `exclude` ：
```
runs-on: ${{ matrix.os }}
strategy:
  matrix:
    os: [macos-latest, windows-latest, ubuntu-18.04]
    node: [4, 6, 8, 10]
    exclude:
      # excludes node 4 on macOS
      - os: macos-latest
        node: 4
```
`exclude` 用来删除特定的配置项，比如这里当 os 为 macos-latest ，将 node 为 4 的版本从构建矩阵中移除。


---
## steps
steps 的通用格式类似于：
```
steps:
  - name: <step_name>
    uses: <action>
    with:
      <parameter_name>: <parameter_value>
    id: <step_id>
    continue-on-error: true

  - name: <step_name>
    timeout-minutes:
    run: <commands>
```
每个 step 步骤可以有:
- `id` : 每个步骤的唯一标识符
- `name` : 步骤的名称
- `uses` : 使用哪个 action
- `run` : 执行哪些指令
- `with` : 指定某个 action 可能需要输入的参数
- `continue-on-error` : 设置为 true 允许此步骤失败 job 仍然通过
- `timeout-minutes` : step 的超时时间

---
## action
action 动作通常是可以通用的，这意味着你可以直接使用别人定义好的 action 。

### checkout action
checkout action 是一个标准动作，当以下情况时必须且需要率先使用:
- workflow 需要项目库的代码副本，比如构建、测试、或持续集成这些操作。
- workflow 中至少有一个 action 是在同一个项目库下定义的。

使用示例：
```
- uses: actions/checkout@v1
```

如果你只想浅克隆你的库，或者只复制最新的版本，你可以在 `with` 中使用 `fetch-depth` 声明，例如:
```
- uses: actions/checkout@v1
  with:
    fetch-depth: 1
```

### 引用 action
- 官方 action 标准库: [github.com/actions](https://github.com/actions)
- 社区库: [marketplace](https://github.com/marketplace?type=actions)

#### 1、引用公有库中的 action

引用 action 的格式是 `{owner}/{repo}@{ref}` 或 `{owner}/{repo}/{path}@{ref}` ，例如上例的中 `actions/checkout@v1` ，你还可以使用标准库中的其它 action ，如设置 node 版本:
```
jobs:
  my_first_job:
    name: My Job Name
      steps:
        - uses: actions/setup-node@v1
          with:
            node-version: 10.x
```

#### 2、引用同一个库中的 action
引用格式：`{owner}/{repo}@{ref}` 或 `./path/to/dir` 。

例如项目文件结构为：
```
|-- hello-world (repository)
|   |__ .github
|       └── workflows
|           └── my-first-workflow.yml
|       └── actions
|           |__ hello-world-action
|               └── action.yml
```
当你想要在 workflow 中引用自己的 action 时可以：
```
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      # This step checks out a copy of your repository.
      - uses: actions/checkout@v1
      # This step references the directory that contains the action.
      - uses: ./.github/actions/hello-world-action
```

#### 3、引用 Docker Hub 上的 container
如果某个 action 定义在了一个 docker container image 中且推送到了 Docker Hub 上，你也可以引入它，格式是 `docker://{image}:{tag}` ，示例：
```
jobs:
  my_first_job:
    steps:
      - name: My first step
        uses: docker://alpine:3.8
```

更多信息参考: [Docker-image.yml workflow](https://github.com/actions/starter-workflows/blob/master/ci/docker-image.yml) 和 [Creating a Docker container action](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/creating-a-docker-container-action) 。


### 构建 actions
请参考：[building-actions](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/building-actions)

---
## env
环境变量可以配置在以下地方:
- `env`
- `jobs.<job_id>.env`
- `jobs.<job_id>.steps.env`

示例：
```
env:
  NODE_ENV: dev

jobs:
  job1:
    env:
      NODE_ENV: test

    steps:
      - name:
        env:
          NODE_ENV: prod
```
如果重复，优先使用最近的那个。

---
## if & context
你可以在 job 和 step 中使用 `if` 条件语句，只有满足条件时才执行具体的 job 或 step :
- `jobs.<job_id>.if`
- `jobs.<job_id>.steps.if`

任务状态检查函数:
- `success()` : 当上一步执行成功时返回 true
- `always()` : 总是返回 true
- `cancelled()` : 当 workflow 被取消时返回 true
- `failure()` : 当上一步执行失败时返回 true


例如：
```
steps:
  - name: step1
    if: always()

  - name: step2
    if: success()

  - name: step3
    if: failure()
```
意思就是 step1 总是执行，step2 需要上一步执行成功才执行，step3 只有当上一步执行失败才执行。

### `${{ <expression> }}`
上下文和表达式: `${{ <expression> }}` 。

有时候我们需要与第三方平台进行交互，这时候通常需要配置一个 token ，但是显然这个 token 不可能明文使用，这种个情况下我们要做的就是：
1. 在具体 repository 库 `Settings` 的 `Secrets` 中添加一个密钥，如 `SOMEONE_TOKEN`
2. 然后在 workflow 中就可以通过 `${{ secrets.SOMEONE_TOKEN }}` 将 token 安全地传递给环境变量。
```
steps:
  - name: My first action
    env:
      SOMEONE_TOKEN: ${{ secrets.SOMEONE_TOKEN }}
```

这里的 `secrets` 就是一个上下文，除此之外还有很多，比如：
- `github.event_name` : 触发 workflow 的事件名称
- `job.status` : 当前 job 的状态，如 success, failure, or cancelled
- `steps.<step id>.outputs` : 某个 action 的输出
- `runner.os` : runner 的操作系统如 Linux, Windows, or macOS

这里只列举了少数几个。

另外在 `if` 中使用时不需要 `${{ }}` 符号，比如：
```
steps:
 - name: My first step
   if: github.event_name == 'pull_request' && github.event.action == 'unassigned'
   run: echo This event is a pull request that had an assignee removed.
```


上下文和表达式详细信息请参考： [contexts-and-expression](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/contexts-and-expression-syntax-for-github-actions)


---
## 结语
最后给个自己写的示例，仅供参考：
```
name: GitHub Actions CI

on: [push]

jobs:
  build-test-deploy:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [8.x, 10.x, 12.x]

    steps:
    - uses: actions/checkout@v1

    - name: install linux packages
      run: sudo apt-get install -y --no-install-recommends libevent-dev

    - name: install memcached
      if: success()
      run: |
        wget -O memcached.tar.gz http://memcached.org/files/memcached-1.5.20.tar.gz
        tar -zxvf memcached.tar.gz
        cd memcached-1.5.20
        ./configure && make && sudo make install
        memcached -d

    - name: Use Node.js ${{ matrix.node-version }}
      uses: actions/setup-node@v1
      if: success()
      with:
        node-version: ${{ matrix.node-version }}

    - name: npm install, build, and test
      env:
        CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}
      if: success()
      run: |
        npm ci
        npm test
        npm run report-coverage

```