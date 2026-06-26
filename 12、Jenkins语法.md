# 十二、Jenkins语法.md

## 1. Jenkins Pipeline 简介

Jenkins Pipeline 是一种插件，允许以**代码的方式**定义持续集成和持续交付（CI/CD）流程。它基于 Jenkins 的 Groovy DSL（领域特定语言），提供了一种强大的方式来描述和自动化构建、测试和部署软件。

| 维度     | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| 核心功能 | 以代码方式定义 CI/CD 流程，基于 Groovy DSL 实现自动化描述    |
| 使用场景 | 适用于需要将软件交付流程标准化的项目，通过代码形式管理流水线配置 |
| 学习特点 | 语法相对简单直观，即使没有开发经验也能理解其逻辑结构         |

------

### 1.1. DSL 简介

- **全称**：Domain Specific Language（领域特定语言）
- **专用性**：专为 Jenkins 流水线设计，语法仅适用于 Jenkins 环境
- **优势**：相比通用语言更聚焦于 CI/CD 场景，提供简洁的流程描述方式

------

### 1.2. DSL 与 GPL 的区别

| 对比项   | DSL                             | GPL                        |
| -------- | ------------------------------- | -------------------------- |
| 全称     | Domain Specific Language        | General Purpose Language   |
| 典型代表 | Jenkins Pipeline 语法           | Java、Python、C 等         |
| 适用范围 | 专门解决 Jenkins 流水线编排问题 | 适用于各种通用软件开发场景 |

------

### 1.3. Pipeline 基本概述

#### 1)  阶段（Stage）

Pipeline 由多个 `stages`（阶段）构成完整流水线，每个阶段对应特定任务。

```
构建（Build）→ 测试（Test）→ 部署（Deploy）
```

- **执行方式**：默认顺序执行，也支持并行执行以优化效率
- **任务划分**：每个阶段承担单一职责，如构建、测试、部署

------

#### 2) 步骤（Step）

每个 `stage` 包含多个 `steps`（步骤），是流水线的**原子操作单元**。

| 操作类型 | 示例命令      |
| -------- | ------------- |
| 代码构建 | `mvn package` |
| 测试执行 | `npm test`    |
| 制品推送 | `docker push` |

> 支持使用内置步骤或自定义步骤满足特殊需求。

------

#### 3)  代理（Agent）

通过 `agent` 指定 Pipeline 的**执行环境**。

| 节点类型    | 说明                                       |
| ----------- | ------------------------------------------ |
| master 节点 | 主控节点，负责调度                         |
| agent 节点  | 工作节点，负责实际执行                     |
| K8s Pod     | 在 Kubernetes 环境中动态创建，作为执行环境 |

> 分布式支持：允许不同阶段在不同节点上执行。

------

#### 4)  变量（Variable）

变量用于存储版本号、环境配置等**动态值**，实现配置与代码分离。

```groovy
environment {
    APP_VERSION = 'v1.0.0'
    DEPLOY_ENV  = 'PROD'
}
```

- **典型应用**：定义构建版本、配置环境参数（DEV / PROD）
- **管理优势**：提高可维护性，避免硬编码

------

#### 5) 控制流（Control Flow）

控制流决定 Pipeline 的**代码执行路径**，类比编程中的条件判断与循环结构。

| 控制类型 | 说明                       | 类比          |
| -------- | -------------------------- | ------------- |
| 条件语句 | 根据条件决定是否执行某阶段 | `if / else`   |
| 循环语句 | 重复执行某段逻辑           | `for / while` |
| 错误处理 | 捕获并处理执行异常         | `try / catch` |



## 2. Jenkins Pipeline 入门

### 2.1 Groovy 语言简介

Pipeline 脚本基于 **Groovy 语言**编写，但无需专门学习 Groovy。

------

#### 1)  Pipeline 两种语法

| 语法类型                  | 说明                                   |
| ------------------------- | -------------------------------------- |
| **声明式（Declarative）** | 主推语法，结构清晰，适合大多数场景     |
| **脚本式（Scripted）**    | 灵活度更高，适合复杂逻辑，但可读性较低 |

------

#### 2)  声明式语法核心结构

声明式 Pipeline 由以下 6 个核心关键字组成：

| 关键字     | 作用                                                   |
| ---------- | ------------------------------------------------------ |
| `pipeline` | 声明 Pipeline 脚本的**起始点**，标志这是一个声明式脚本 |
| `agent`    | 指定 Pipeline **运行的节点**（master 或 slave 节点）   |
| `stages`   | 包含**所有阶段的集合**，如打包、部署等                 |
| `stage`    | 定义**单个阶段**，一个 `stages` 内可包含多个 `stage`   |
| `steps`    | 定义每个阶段内的**最小执行单元**                       |
| `post`     | 定义**构建后操作**，根据构建结果执行对应逻辑           |

整体层级关系如下：

```
pipeline
├── agent
├── stages
│   └── stage("阶段名")
│       └── steps
│           └── 具体步骤
└── post
    └── always / success / failure ...
```

------

#### 3) 入门示例

```groovy
pipeline {
    agent any                              // 在任意可用节点上运行

    stages {
        stage("This is the first stage") { // 定义第一个阶段
            steps {
                echo "I am xuegod"         // 执行步骤：打印输出
            }
        }
    }

    post {
        always {
            echo "The process is ending"   // 无论成功或失败，始终执行
        }
    }
}
```

**代码说明：**

- `agent any`：表示在任意可用的节点上执行，无需指定特定机器
- `stage("...")`：括号内为阶段名称，会显示在 Jenkins 构建视图中
- `echo`：内置步骤，用于在控制台输出文本信息
- `post { always { } }`：无论构建结果如何，`always` 块中的步骤都会被执行

> **`post` 常用条件块：**
>
> - `always`：始终执行
> - `success`：仅构建成功时执行
> - `failure`：仅构建失败时执行
> - `unstable`：构建结果不稳定时执行

### 2.2 Jenkins Pipeline 声明式语法

#### 1) Environment（环境变量）

**字段概述**

| 属性     | 说明                                          |
| -------- | --------------------------------------------- |
| 是否必需 | 否，可省略不写                                |
| 作用     | 在 Pipeline 执行过程中定义和配置环境变量      |
| 定义位置 | 可在 `pipeline` 顶层或单个 `stage` 内定义     |
| 参数配置 | 无需额外参数，直接以 `KEY = 'VALUE'` 形式声明 |

**示例代码**

```groovy
pipeline {
    agent any
    environment {
        CC = 'clang'       // 定义环境变量 CC，值为 clang
    }
    stages {
        stage('Example') {
            steps {
                sh 'printenv'  // 打印所有环境变量，验证 CC=clang 是否生效
            }
        }
    }
}
```

**代码说明**

| 元素            | 说明                                                |
| --------------- | --------------------------------------------------- |
| `agent any`     | 在任意可用节点运行；K8s 环境下在 Jenkins Pod 内执行 |
| `CC = 'clang'`  | 在 `environment` 块中定义键值对形式的环境变量       |
| `sh 'printenv'` | 执行 Shell 命令，打印所有环境变量以验证配置         |

> **验证方法**：执行后在控制台日志中搜索 `CC=clang`，出现即表示环境变量设置成功。

------

#### 2) Options（行为配置）

**字段概述**

| 属性     | 说明                                                   |
| -------- | ------------------------------------------------------ |
| 是否必需 | 否，可省略不写                                         |
| 定义位置 | 仅允许在 `pipeline` 顶层块内定义，不可用于单个 `stage` |
| 作用     | 配置 Pipeline 的全局行为策略                           |

**常用配置项**

| 配置项                    | 说明                                         |
| ------------------------- | -------------------------------------------- |
| `buildDiscarder`          | 构建丢弃策略，控制历史构建记录的保留数量     |
| `disableConcurrentBuilds` | 禁止并发执行，同一时间只允许一个构建运行     |
| `timeout`                 | 设置 Pipeline 执行的超时时间，超时后自动中止 |
| `retry`                   | Pipeline 执行失败时的自动重试次数            |
| `timestamps`              | 在控制台日志中为每行输出添加时间戳           |
| `checkoutToSubdirectory`  | 将源代码检出到工作区的指定子目录中           |

**示例代码**

```groovy
pipeline {
    agent any
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))  // 仅保留最近 10 次构建记录
        disableConcurrentBuilds()                        // 禁止并发构建
        timeout(time: 1, unit: 'HOURS')                 // 超过 1 小时自动终止
        retry(3)                                         // 失败时最多重试 3 次
        timestamps()                                     // 日志输出添加时间戳
    }
    stages {
        stage('Example') {
            steps {
                echo 'Hello'
            }
        }
    }
}
```

> **注意**：`options` 中的配置项作用于整个 Pipeline，如需对单个阶段设置超时，应在 `stage` 内使用 `options { timeout(...) }` 单独配置。

#### 3)  Parameters（参数化构建）

**字段概述**

| 属性     | 说明                                     |
| -------- | ---------------------------------------- |
| 是否必需 | 否，可省略不写                           |
| 定义位置 | 只能在 `pipeline` 最外层定义一次         |
| 作用范围 | 对所有阶段和步骤全局生效                 |
| 调用方式 | 通过 `params` 对象在 Pipeline 各阶段引用 |
| 核心功能 | 允许用户在触发构建时传入不同的参数值     |

**常用参数类型**

| 参数类型       | 说明                               |
| -------------- | ---------------------------------- |
| `booleanParam` | 布尔类型，仅接受 `true` 或 `false` |
| `string`       | 字符串类型，接受任意文本输入       |
| `choice`       | 下拉选项，从预设列表中选择一个值   |
| `password`     | 密码类型，输入内容在日志中脱敏显示 |

**示例代码**

```groovy
pipeline {
    agent any
    parameters {
        booleanParam(name: 'DEBUG', defaultValue: false, description: '是否开启调试模式')
        string(name: 'VERSION', defaultValue: 'v1.0.0', description: '构建版本号')
    }
    stages {
        stage('Example') {
            steps {
                echo "版本号：${params.VERSION}"
                echo "调试模式：${params.DEBUG}"
            }
        }
    }
}
```

> **调用方式**：通过 `params.参数名` 引用，例如 `params.VERSION`。

------

#### 4) Triggers（自动触发）

**字段概述**

| 属性     | 说明                                     |
| -------- | ---------------------------------------- |
| 是否必需 | 否，可省略不写                           |
| 定义位置 | 仅在 `pipeline` 最外层定义，只能出现一次 |
| 核心功能 | 指定 Pipeline 的自动化触发方式           |

**触发类型**

| 类型       | 说明                                        | 使用频率 |
| ---------- | ------------------------------------------- | -------- |
| `cron`     | 按 Cron 表达式定时触发，类似 Linux 计划任务 | ⭐⭐⭐ 高   |
| `pollSCM`  | 定时轮询代码仓库，检测到变更则触发构建      | ⭐⭐ 中    |
| `upstream` | 当上游 Pipeline 构建完成后触发              | ⭐ 低     |

**Cron 表达式说明**

```
# 格式：分钟  小时  日  月  星期
#        *     *    *   *    *

H 4 * * 1-5     # 周一至周五，每天 04:xx 执行一次（H 用于分散触发时间，避免多任务同时启动）
0 0 * * *       # 每天凌晨 00:00 执行
0 8 * * 1-5     # 周一至周五，每天上午 08:00 执行
```

> **`H` 符号说明**：Jenkins 使用哈希值将触发时间分散在指定区间内，避免多个任务在同一时刻并发启动，提高系统稳定性。

**示例代码**

```groovy
pipeline {
    agent any
    triggers {
        cron('H 4 * * 1-5')   // 周一至周五每天 04:xx 自动触发
    }
    stages {
        stage('Example') {
            steps {
                echo '定时构建触发'
            }
        }
    }
}
```

------

#### 5)  Input（人工审批）

**字段概述**

| 属性     | 说明                                            |
| -------- | ----------------------------------------------- |
| 是否必需 | 否，可省略不写                                  |
| 核心功能 | 暂停 Pipeline 执行，等待用户手动输入确认后继续  |
| 必填参数 | `message`，用于向审批人展示提示信息             |
| 权限控制 | 通过 `submitter` 参数指定允许审批的用户或用户组 |

**典型应用场景**

```
开发环境  →  自动发布（无需确认）
测试环境  →  人工确认后发布
生产环境  →  人工确认后发布（高风险操作必须审批）
```

**示例代码**

```groovy
pipeline {
    agent any
    stages {
        stage('Deploy to Production') {
            input {
                message "确认发布到生产环境？"          // 展示给审批人的提示信息
                submitter "admin,ops-team"              // 仅允许 admin 和 ops-team 审批
                parameters {
                    string(name: 'person', defaultValue: 'Mr Jenkins', description: '审批人姓名')
                }
            }
            steps {
                echo "Hello ${person}, 已确认发布！"   // 引用 input 中定义的参数
            }
        }
    }
}
```

> **执行流程**：Pipeline 运行到 `input` 步骤时自动暂停 → 指定审批人收到通知 → 审批人确认或中止 → 根据结果继续或终止后续步骤。

------

#### 6)  Parallel（并行执行）

**字段概述**

| 属性     | 说明                                                   |
| -------- | ------------------------------------------------------ |
| 是否必需 | 否，可省略不写                                         |
| 定义位置 | 在 `stages` 下的 `stage` 内定义 `parallel` 块          |
| 核心功能 | 允许同一阶段内的多个子任务同时并行执行，缩短总构建时长 |

**示例代码**

```groovy
pipeline {
    agent any
    stages {
        stage('Parallel Tests') {
            parallel {
                stage('Unit Test') {          // 并行任务一：单元测试
                    steps {
                        echo '执行单元测试'
                    }
                }
                stage('Integration Test') {   // 并行任务二：集成测试
                    steps {
                        echo '执行集成测试'
                    }
                }
                stage('Code Scan') {          // 并行任务三：代码扫描
                    steps {
                        echo '执行代码质量扫描'
                    }
                }
            }
        }
    }
}
```

------

#### 7) . 声明式指令汇总

| 指令          | 是否必需 | 核心作用         |
| ------------- | -------- | ---------------- |
| `agent`       | ✅ 必需   | 指定执行节点     |
| `stages`      | ✅ 必需   | 包含所有阶段     |
| `environment` | 否       | 定义环境变量     |
| `options`     | 否       | 配置全局行为策略 |
| `parameters`  | 否       | 参数化构建输入   |
| `triggers`    | 否       | 自动化触发方式   |
| `input`       | 否       | 人工审批确认     |
| `parallel`    | 否       | 并行执行多个任务 |

> **学习建议**：先理解各指令的功能与用途，再通过实际项目练习。检验标准：能否不参考文档，独立写出一个包含环境变量、参数化、定时触发和并行阶段的完整 Pipeline 脚本。