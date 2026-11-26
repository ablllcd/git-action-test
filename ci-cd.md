## GitHub Actions 简介

GitHub Actions 是 GitHub 自带的持续集成 / 持续交付（CI/CD）服务：在仓库里放一个 YAML 文件，声明"什么时候、在什么环境、执行哪些步骤"，GitHub 就会在代码推送、发 PR 等时机自动跑起来。

官方文档：https://docs.github.com/zh/actions
入门教程（中文）：https://docs.github.com/zh/actions/get-started/quickstart

## 核心概念

### 基本元素

| 概念 | 含义 |
| --- | --- |
| workflow（工作流） | 一次自动化流程，对应一个 YAML 文件 |
| event（事件） | 触发条件，如 `push`、`pull_request`、`schedule`、手动触发 |
| job（任务） | workflow 里的一个任务，默认不同 job 并行、跑在不同的机器上 |
| step（步骤） | job 里的一步，按顺序执行，失败即中止该 job |
| action（动作） | 可复用的步骤，用 `uses:` 引用，如 `actions/checkout@v4` |
| runner（运行器） | 执行 job 的机器，`runs-on` 指定，如 `ubuntu-latest` |

几个要点：

* 一个 step 只能是 `run`（跑命令）或 `uses`（用现成 action）之一。
* job 之间默认并行，用 `needs` 声明依赖顺序。
* action 可以被别人引用，引用格式是 `owner/repo@版本`，例如 `actions/setup-node@v4`。

### 最小示例

```yaml
name: Hello CI

# 触发条件：推送到 main、或向 main 发 PR
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: 拉取代码
        uses: actions/checkout@v4

      - name: 打印信息
        run: |
          echo "分支: ${{ github.ref_name }}"
          echo "提交: ${{ github.sha }}"
```

`${{ }}` 里是表达式，`github` 是上下文对象，常用的有 `github.ref_name`（分支名）、`github.sha`、`github.event_name`（事件名）、`github.repository`。

### 触发方式

```yaml
on:
  push:                     # 推送触发
    branches: [main]
    tags: ["v*"]            # 打 v1.0.0 这类标签也触发
  pull_request:             # PR 触发
    branches: [main]
  schedule:                 # 定时触发，cron 用 UTC 时间
    - cron: "0 18 * * *"    # 北京时间每天 02:00
  workflow_dispatch:        # 允许在网页上手动点按钮运行
```

注意：

* `schedule` 的 cron 是 UTC，北京时间要减 8 小时换算。
* 用 `workflow_dispatch` 可以在 Actions 页面手动触发并填参数。
* `pull_request` 默认只对"目标分支是本仓库分支"的 PR 生效，来自 fork 的 PR 拿不到 secrets，这是安全设计。

## 项目架构

### 本地文件位置

workflow 文件放在仓库根目录的 `.github/workflows/` 下，后缀 `.yml` 或 `.yaml`，文件名随意。一个仓库可以有多个 workflow 文件，GitHub 发现就会按各自的事件触发。

```
项目根目录
└── .github
    └── workflows
        ├── ci.yml
        └── deploy.yml
```

### GitHub 查看位置

可在 GitHub 仓库的 Actions 页面查看 workflow 运行情况，点进某次运行可以看到每个 job 的日志。



## 一个 Java / Maven 的完整例子

```yaml
name: Java CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

# 最小权限：默认只读，需要写操作时单独声明
permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      # 安装 JDK，同时用 setup-java 自带的 cache 缓存 ~/.m2
      - name: 安装 JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "17"
          cache: maven

      - name: 编译与单测
        run: mvn -B -ntp verify

      # 把构建产物 jar 上传，供下载或后续 job 使用
      - name: 上传 jar
        uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: target/*.jar
          retention-days: 7
```

几个说明：

* `-B` 是批处理模式，日志不带进度条，适合 CI；`-ntp` 关掉下载进度输出，让日志更干净。
* `cache: maven` 会按 `pom.xml` 的哈希缓存依赖，命中后构建大幅提速。
* 上传的 artifact 默认保留 90 天，可以用 `retention-days` 缩短。

## 需要数据库的测试

job 里可以起 sidecar 容器，测试连 `localhost:3306` 即可：

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      mysql:
        image: mysql:8
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: test
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping -proot"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "17"
          cache: maven
      - run: mvn -B -ntp test
```

`options` 里的健康检查用来等数据库真正就绪，否则测试可能连不上就报错。

## 多版本矩阵

想同时在多个 JDK 版本上验证：

```yaml
jobs:
  build:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false          # 一个组合失败不影响其它组合
      matrix:
        os: [ubuntu-latest, windows-latest]
        java: ["11", "17", "21"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: ${{ matrix.java }}
      - run: mvn -B -ntp verify
```

注意：3 个 JDK × 2 个系统 = 6 个 job 并行，每个都算时长，别滥用。

## 机密信息

密码、Token 一律放仓库的 Settings → Secrets and variables → Actions 里，然后这样用：

```yaml
      - name: 登录镜像仓库
        run: echo "${{ secrets.REGISTRY_PASSWORD }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
```

注意：

* secrets 在日志里会被自动打码，但**不要**把 secret 拼进命令行参数再 echo 出来，容易绕过打码泄露。
* PR 来自 fork 时拿不到 secrets，这是为了防止别人偷你的密钥。
* 环境变量（非机密）放在仓库 Variables 里，用 `vars.XXX` 引用。

## 权限与 GITHUB_TOKEN

每次 workflow 运行都会自动注入一个 `GITHUB_TOKEN`，用来操作仓库（推代码、发 release、评论 PR 等）。默认权限通常只有读，要写就显式声明：

```yaml
permissions:
  contents: write
  pull-requests: write
```

原则是最小权限：哪个 job 需要什么就给什么，不要图省事写 `write-all`。

## 节省时间的小技巧

```yaml
# 同一分支连续推送时，取消上一次还在跑的运行
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

* 用 `actions/cache@v4` 缓存构建目录，但缓存 key 要带上依赖文件哈希，例如 `${{ hashFiles('**/pom.xml') }}`。
* 慢的步骤拆到后面的 job，先用 `needs` 串起来，让"编译 + 快测"先给出结果。
* 只想在特定路径变更时触发，可以用 `paths` 过滤。

## 常见坑

* 引用的 action 用 `@master` 这种浮动版本，上游一改你的流水线就可能挂，**固定到 `@v4` 或 commit SHA**。
* 默认分支现在叫 `main`，网上很多老教程写的 `master` 直接抄会不触发。
* `run` 默认在仓库根目录执行，子目录项目要写 `working-directory: backend`。
* 一个 job 内的 step 之间文件是保留的（同一台机器），但 job 之间不保留，跨 job 传文件要用 artifact。
* `runs-on` 的 runner 是干净环境，每次都要重新 checkout 和装依赖，别指望状态残留。
* 免费额度：公开仓库一般免费，私有仓库按账户计划给一定分钟数，超出按分钟计费，具体以 GitHub 官网当前价格为准。

## 参考

* GitHub Actions 中文文档：https://docs.github.com/zh/actions
* 持续集成（官方）：https://docs.github.com/zh/actions/get-started/continuous-integration
* Action 市场：https://github.com/marketplace?type=actions
* 阮一峰《GitHub Actions 入门教程》：https://www.ruanyifeng.com/blog/2019/09/getting-started-with-github-actions.html （概念讲得清楚，但示例是 2019 年的，`@master` 和 `master` 分支要按现在的写法改）
