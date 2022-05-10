---
title: "GitLab CI/CD"
date: 2022-05-09T22:59:10+08:00
toc: true
---

## 概念

|术语|说明|
|---|---|
|pipeline|流水线|
|stage|阶段|
|job|任务|
|.gitlab-ci.yml|CI/CD配置文件|
|GitLab Runner|负责运行job的Agent。使用前需要先安装注册|
|Executor|注册Runner时，需要选择一个Executor作为运行环境，如 Docker、K8s[^1] [^2]|
|预置变量|CI/CD预置变量，每个流水线都有这些变量|

## Pipeline 类型

- Basic pipelines：基本流水线，同一 Stage 内并发执行，不同 Stage 顺序执行。

![gitlab-cicd-1](/images/gitlab-cicd-1.png)

- DAG pipelines：有向无环流水线，使用 `needs` 指明 Job 间的依赖关系，优化并发流程。

![gitlab-cicd-2](/images/gitlab-cicd-2.png)

- Merge request pipelines：又称 branch pipeline，提交 MR 的时候触发，需要配合 `only` 或 `rules` 使用。

```yaml
default:
  image: ubuntu:latest

build-job:
  only:
    - merge_requests
  script:
    - echo 'build-job'

test-job:
  script:
    - echo 'test-job'
```

如图，提交的 MR 只触发了 build-job：

![gitlab-cicd-3](/images/gitlab-cicd-3.png)

- Merged results pipelines：代码合并时触发的流水线。可以在 Projects > Settings > General 中配。

- Scheduled pipelines：周期执行的流水线。

## Job

Job 被 Runner 执行，Job 之间是相互独立的。Runner 越多，能支持并发执行的 Job 数越大。

Job 类型：

- Hide Job：名字以（.）开头的 Job 不会被 GitLab CI/CD 执行

```yaml
.hidden_job:
  script:
    - run test
```

- Deployment Job：包含 `environment`、`action: start` 的 Job 是用来将代码部署到环境

```yaml
deploy-me:
  script:
    - deploy-to-cats.sh
  environment:
    name: production
    url: https://cats.example.com
    action: start
```

[^1]: [Runner executors](https://docs.gitlab.com/runner/executors/)
[^2]: [A Brief Guide to GitLab CI Runners and Executors](https://medium.com/devops-with-valentine/a-brief-guide-to-gitlab-ci-runners-and-executors-a81b9b8bf24e)