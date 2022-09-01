---
title: "GitLab CI 工作原理"
date: 2022-08-24T22:24:19+08:00
---

本文介绍，当我们向 GitLab 代码仓库提交代码后（merge request），GitLab Server 如何触发流水线和执行 CI Job。了解 GitLab CI 工作原理可以帮助我们更好实践和优化 CI 流程，提高开发效率。

# 概念

GitLab CI 功能涉及以下组件：

- GitLab Server：GitLab 后端，通过 REST API 提供各种能力，包括 Git 仓库管理、CI/CD 功能；
- GitLab Runner：和 Jenkins 的 agent 是一个概念，负责从 GitLab Server 中不断地拿到待执行 Job，并启动一个 Executor 去执行。Runner 可以是二进制部署，也可以是部署为一个 Docker 容器，或者是 K8s 集群中的 Pod； 
- GitLab Executor：CI Job 执行器，实际处理 CI 任务。GitLab 支持多种 Executor，比如：一个 CI Job 可以在一个本地物理机上执行、或一个每次新建的虚拟机环境中执行、或 Docker 容器以及 K8s Pod（使用时新建，结束时销毁）。

比如，我的开发环境选型是二进制部署 Runner，每台机器上各部署一个，执行器 Executor 采用 Docker。

# GitLab CI 架构

GitLab 官方文档提供了 GitLab CI 架构图如下[^1]：

![](/images/gitlab_ci_architecture.png)

从左到右：

1. 触发 Pipeline

GitLab CI 功能从流水线触发事件开始。流水线触发有多种方式，最常见的是 `git push`、在 UI 上点击 "Run pipeline"按钮、提交 MR 或者调用 Pipeline API 触发等。任何触发事件都会调用 [CreatePipelineService](https://gitlab.com/gitlab-org/gitlab/-/blob/master/app/services/ci/create_pipeline_service.rb) 模块。

GitLab Server 中的 CreatePipelineService 模块负责接收流水线触发事件，并依靠 [YAML Processor 组件](https://gitlab.com/gitlab-org/gitlab/-/blob/master/lib/gitlab/ci/yaml_processor.rb) 创建流水线。YAML Processor 读取并解析用户编写的 .gitlab-ci.yml 文件，并转化成一个包含流水线信息的数据结构（有向无环图）。

2. Pipeline 处理

当流水线创建好后，流水线中的每个 Job 的状态[^2]由 [ProcessPipelineService](https://gitlab.com/gitlab-org/gitlab/-/blob/master/app/services/ci/process_pipeline_service.rb) 模块管理和追踪。最开始，Job 的状态都是 created。ProcessPipelineService 将可以开始执行的 Job 状态标记为 pending。

处于 pending 状态 Job 可以被 GitLab Runner 拿到并执行，进入到 running 状态。最终，执行完的 Job 状态从 Runner 返回给 GitLab Server，并由 ProcessPipelineService 修改状态为 success 或 failed。

3. 分配 Job 到 Runner

CI Job 分配是由 Runner 主动发起。当 GitLab Server 收到 Runner 的请求（如下图[^3]），会通过 [RegisterJobService](https://gitlab.com/gitlab-org/gitlab/-/blob/master/app/services/ci/register_job_service.rb) 模块选择并返回一个 pending Job 给 Runner。当当前所有 Job 全部执行完，GitLab Server 会将下一个阶段（stage）的 CI Job 状态改为 pending 待执行。

![](/images/gitlab_runner_request_job.png)

# GitLab Runner 工作原理

## 安装、启动、注册 Runner

常见 Runner 安装方式有三种：
- 二进制
- Docker：安装、启动和注册的流程同二进制
- K8s：需要 K8s 集群，可用 Helm Chart 一键安装部署 Runner。注册等参数写在 Chart 的配置里

GitLab Runner 安装好后，只有先注册、绑定到 GitLab Server，才能正常和 GitLab Server 通信、工作。安装和注册 Runner 可以通过 gitlab-runner CLI 命令完成[^4]：

```shell
# 指定用户名、工作目录安装 gitlab-runner 服务
gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner

# 启动 gitlab-runner 服务
gitlab-runner start

# 按提示向 GitLab Server 注册自己、以及设置 tag
gitlab-runner register
```

## Executor 选型

Runner 创建多个 Executor 执行不同 CI Job，Executor 是 CI Job 真正执行环境，执行环境互相隔离。GitLab 支持以下几种 Executor 选项[^5]：
- Shell：即 Runner 直接在自己的本地机器上执行 CI Job，因此如果 CI Job 要执行各种指令，例如 make、docker、flake8...，需要提前安装好这些命令行工具；
- VirtualBox：每次要执行，会建立一个干净的 VM，透过 SSH 登录此 VM，并在其中执行 CI Job；
- Docker：同上，但透过 Docker 执行 Job。Runner 连接到  Docker Engine，请求 Docker Engine 创建容器执行 Job。Docker Executor 的相关的参数写在 Runner 的配置文件 config.toml 中的 [ runner.docker ] 下面，包含镜像拉取策略（pull-policy）等参数；
- Kubernetes：同 Docker 一样，也是在容器环境中执行 Job。但 Runner 操控的不再是 Docker Engine，而是 K8s。Runner 连接到 K8s API Server，每当有 CI Job 指派给 Runner 时，Runner 就会透过 K8s 先建立一个干净的 Pod，接着在其中执行 CI Job。

## 时序图[^6]

![](/images/gitlab_ci_runner_workflow.png)

Runner 的工作流程如下：

1. 每个空闲的 Runner 会持续、周期性地调用 POST /api/v4/jobs/request API（默认每 3 秒），向 GitLab Server 请求排队中待执行的 CI Job。大多数时候，Server 返回 204（No Content）。

当有 commit 提交时，GitLab Server 检查 commit 中的 .gitlab-ci.yaml 文件触发 CI/CD 流水线，产生 pipeline 和 CI Job。Runner 再请求 API 时，会返回 201（Created），以及 Job 信息。具体返回哪个 Job 给 Runner，由 Server 根据一定的策略决定，如：谁先请求，先分配给谁。

2. Runner 收到分配的 Job 后，调用 PUT /api/v4/jobs/{job_id} 确认，并创建 Executor 去实际执行 Job。比如使用的 Executor 是 Docker，Job 就以 docker-in-docker 的方式执行。

3. Executor 执行过程中的中间结果和信息，会返回给 Runner。Runner 则调用 PATCH /api/v4/jobs/{job_id}/trace API 将中间状态信息、日志写回 Server。当最终执行完成，调用 PUT /api/v4/jobs/{job_id} 更新 Job 结果。

![](/images/gitlab_ci_runner_api_1.png)

![](/images/gitlab_ci_runner_api_2.png)

[^1]: [CI Architecture overview](https://docs.gitlab.com/ee/development/cicd/) 
[^2]: [CI Job Status](https://docs.gitlab.com/ee/ci/jobs/#the-order-of-jobs-in-a-pipeline)
[^3]: [⼜拍云 CI/CD 实践](https://opentalk-blog.b0.upaiyun.com/prod/2017-10-27/59c1c82f33c56c23422250ca45f52d31.pdf)
[^4]: [GitLab Runner commands](https://docs.gitlab.com/runner/commands/)
[^5]: [GitLab CI 之 Runner 的 Executor 該如何選擇？](https://chengweichen.com/2021/03/gitlab-ci-executor.html)
[^6]: [Runner execution flow](https://docs.gitlab.com/runner/#runner-execution-flow)