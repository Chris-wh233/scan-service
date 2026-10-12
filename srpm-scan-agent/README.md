# SRPM Scan Agent（规划中）

本目录预留给 `srpm-scan-agent`：一个用 csmock 扫描源码包（SRPM）的 Jenkins
inbound agent。目前只有规范说明，没有可运行代码。

实现时请遵守本文档描述的交互规则与目录/配置约定；同构的参考实现见
`../rpm-scan-agent/`（结构一致，只是扫描工具与产物不同）。

## 职责

1. 作为 Jenkins inbound agent 注册到 Jenkins（label 建议 `srpm-scan`）；
2. 从只读挂载目录读取待扫描的 `.src.rpm`；
3. 执行 csmock 扫描；
4. 把扫描结果归档到 Jenkins；
5. 向 Scan Results Portal 上报 `started` / `completed` / `failed` 事件。

Portal 不执行 csmock，只消费事件和 Jenkins 归档产物。

## 目录与挂载规范

| 宿主目录 | 容器路径 | 权限 | 用途 |
| --- | --- | --- | --- |
| `/home/chris/proj/jenkins/data/inputs/srpm` | `/scan-input` | ro | 待扫描的 `.src.rpm` |
| `/home/chris/proj/jenkins/data/srpm-scan-agent/workspace` | `/home/jenkins/agent` | rw | agent 工作目录（remoting + 每个 job 的工作区） |
| `/home/chris/proj/jenkins/data/srpm-scan-agent/mock` | `/var/lib/mock` | rw | csmock/mock 的 chroot 缓存（按需增加） |

容器路径与 `rpm-scan-agent` 保持一致（都用 `/scan-input`），因为每个 agent 只挂自己那一份输入。

## 配置规范

**不要在本组件内保留 `.env`**。所有配置集中在上一级目录的
`scan-service/.env`，由 compose 通过变量插值注入容器；键名与分区见该目录的
`.env.example`，本组件对应的键为：

| 键 | 说明 |
| --- | --- |
| `SRPM_SCAN_AGENT_NAME` | Jenkins permanent agent 名称，如 `srpm-scan-agent-01` |
| `SRPM_SCAN_AGENT_SECRET` | 该 agent 的 inbound secret |
| `CSMOCK_TOOLS` | csmock 使用的检查器集合（可选，按需） |
| `MOCK_CONFIG` | csmock 使用的 mock 配置（可选，按需） |

以下键在栈内共享，直接复用，不要重复定义：

`JENKINS_URL`（agent 注册）、`JENKINS_AGENT_JAR_URL`（构建镜像时下载
agent.jar）、`SCAN_RESULTS_PORTAL_URL`（回传事件的门户地址，栈内为
`http://scan-results-portal:8080`）。

## 交互规则

### 1. 与 Jenkins

- 在 Jenkins 上创建 permanent agent，名称取 `SRPM_SCAN_AGENT_NAME`，label
  使用 `srpm-scan`；
- 用 `JENKINS_URL` + `SRPM_SCAN_AGENT_SECRET` 通过 JNLP/WebSocket 注册
  （可参考 `../rpm-scan-agent/scripts/jenkins-agent-entrypoint`）；
- Pipeline 侧用 `agent { label 'srpm-scan' }` 指定该 agent。

### 2. 输入

- 只从 `/scan-input`（宿主 `data/inputs/srpm`，只读）读取 `.src.rpm`；
- 由 Pipeline 参数传路径，例如
  `SRPM_PATH=/scan-input/curl-8.4.0-9.ocs23.src.rpm`；不要把待扫描包写进
  镜像或 agent 的工作目录。

### 3. 结果归档（决定 Portal 是否判为 FINISHED）

扫描结果打包成 `.tar.xz` 并归档到 Jenkins。Portal 会在该构建的所有归档产物
中挑选第一个满足条件的 `.tar.xz` 作为扫描结果，条件是：

- 归档内**必须包含**以下 6 个文件：

~~~text
scan.ini
scan-results-summary.txt
scan-results.js
scan-results.err
scan-results.html
scan.log
~~~

- 归档结构：这些文件要么位于归档根目录，要么位于 `<NVR>/` 下一层；
  `<NVR>/debug/` 下的内容会被忽略；
- 归档本身要能正常解压，且单个归档不超过 512 MiB（Portal 的解析上限）；
- Portal 还会要求 `scan.ini` 可解析、`scan-results-summary.txt` 存在，
  否则任务记为 `NOT_AVAILABLE`；
- 可选：额外归档 `.sarif` 文件，Portal 会据此解析 finding 明细。

以上任一不满足，任务都不会被标记为 `FINISHED`。

### 4. 事件上报

向门户上报事件：

~~~text
POST $SCAN_RESULTS_PORTAL_URL/api/v1/srpm/events
Authorization: Bearer $SCAN_SERVICE_INGEST_TOKEN
Content-Type: application/json
~~~

`SCAN_SERVICE_INGEST_TOKEN` 来自 Jenkins 凭据 `scan-service-ingest-token`
（Pipeline 里用 `credentials()` 取，不要写进 .env）。

请求体字段：

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `event` | 是 | `started` / `completed` / `failed` |
| `jenkins_job_full_name` | 是 | 完整 Job 名（支持 folder 路径，如 `packages/curl`） |
| `jenkins_build_number` | 是 | 构建号 |
| `srpm_name` | 否 | SRPM 文件名，建议填写（页面 Label 用它） |
| `created_at` | started 用 | ISO-8601 扫描开始时间 |
| `finished_at` | completed/failed 用 | ISO-8601 扫描结束时间 |
| `csmock_exit_code` | 否 | csmock 退出码 |

发送时机：

- 扫描开始前发 `started`（门户据此建任务，状态 `RUNNING`）；
- csmock 正常结束、并已完成归档后发 `completed`；
- csmock 失败发 `failed`（不发送 `completed`）；
- 同一 `jenkins_job_full_name + jenkins_build_number` 重复发 `started`
  不会重复建任务，可安全重试。

### 5. 状态语义

门户只认这四种状态：`RUNNING` / `FINISHED` / `FAILED` / `NOT_AVAILABLE`。

- `RUNNING`：收到 `started` 之后、尚未结束；
- `FINISHED`：收到 `completed` 且归档校验通过（见第 3 节）；
- `FAILED`：收到 `failed`；
- `NOT_AVAILABLE`：扫描已结束但归档缺失、损坏或缺少预期文件。

## 实现骨架（伪代码）

镜像与 `rpm-scan-agent` 同构：基于 LoongArch64 基础镜像，安装 Java（跑
inbound agent）、csmock、mock、tar/xz 等工具，并预置 `agent.jar`。

~~~dockerfile
# 与 ../rpm-scan-agent/Dockerfile 结构一致，仅替换扫描工具
FROM lcr.loongnix.cn/library/openjdk:21-jdk

ARG JENKINS_AGENT_JAR_URL=http://jenkins:8080/jnlpJars/agent.jar

RUN ...  # 安装 csmock、mock、rpm 工具链、tar/xz
RUN curl --fail --location --output /usr/share/jenkins/agent.jar "$JENKINS_AGENT_JAR_URL"

COPY scripts/run-srpm-scan /usr/local/bin/run-srpm-scan
COPY scripts/send-srpm-event /usr/local/bin/send-srpm-event
COPY scripts/jenkins-agent-entrypoint /usr/local/bin/jenkins-agent-entrypoint

USER jenkins
WORKDIR /home/jenkins/agent
ENTRYPOINT ["jenkins-agent-entrypoint"]
~~~

`jenkins-agent-entrypoint` 与 rpm agent 完全相同（用 `JENKINS_URL` /
`JENKINS_AGENT_NAME` / `JENKINS_SECRET` 启动 `agent.jar`）。

`send-srpm-event` 可照搬 `../rpm-scan-agent/scripts/send-rpm-event`，只把
路径换成 `/api/v1/srpm/events`、字段换成上表所列。

扫描脚本伪代码：

~~~bash
# run-srpm-scan  INPUT_SRPM OUTPUT_DIR
set -euo pipefail
input_srpm=$1
output_dir=$2

test -s "$input_srpm"
mkdir -p "$output_dir"

csmock -r "${MOCK_CONFIG}" -t "${CSMOCK_TOOLS}" \
       --force -o "$output_dir/csmock" "$input_srpm"
csmock_exit=$?

# 结果目录打包成 .tar.xz，且必须包含上面 6 个预期文件
tar -cJf "$output_dir/scan-results.tar.xz" -C "$output_dir/csmock" .

exit "$csmock_exit"
~~~

Pipeline 片段（仅供参考）：

~~~groovy
pipeline {
    agent { label 'srpm-scan' }

    parameters {
        string(name: 'SRPM_PATH',
               defaultValue: '/scan-input/curl-8.4.0-9.ocs23.src.rpm',
               description: 'SRPM file available inside srpm-scan-agent')
    }

    environment {
        // 门户地址由 agent 容器注入，不在 Pipeline 里写死
        SCAN_SERVICE_INGEST_TOKEN = credentials('scan-service-ingest-token')
    }

    stages {
        stage('Prepare SRPM scan') {
            steps {
                script {
                    env.SRPM_NAME = sh(script: 'basename -- "$SRPM_PATH"',
                                       returnStdout: true).trim()
                    sh 'send-srpm-event started --srpm-name "$SRPM_NAME"'
                }
            }
        }
        stage('SRPM scan') {
            steps {
                script {
                    env.SCAN_EXIT = sh(
                        script: 'run-srpm-scan "$SRPM_PATH" srpm-scan-artifacts',
                        returnStatus: true
                    ).toString()
                }
            }
        }
        stage('Archive SRPM scan result') {
            steps {
                archiveArtifacts artifacts: 'srpm-scan-artifacts/**/*.tar.xz',
                                 allowEmptyArchive: false, fingerprint: true
            }
        }
        stage('Publish SRPM scan state') {
            steps {
                script {
                    if (env.SCAN_EXIT == '0') {
                        sh 'send-srpm-event completed --srpm-name "$SRPM_NAME" --csmock-exit-code "$SCAN_EXIT"'
                    } else {
                        sh 'send-srpm-event failed --srpm-name "$SRPM_NAME" --csmock-exit-code "$SCAN_EXIT"'
                        error("SRPM scan failed with exit code ${env.SCAN_EXIT}")
                    }
                }
            }
        }
    }
}
~~~

## 开发步骤

1. 在 `scan-service/.env` 中取消 `srpm-scan-agent` 分区注释并填写；
2. 在 `scan-service/docker-compose.yml` 中取消 `srpm-scan-agent` 服务块注释；
3. 按上面的骨架实现 `Dockerfile` 与 `scripts/`；
4. 在 Jenkins 上创建 permanent agent，并把 secret 写入 `scan-service/.env`；
5. `cd scan-service && docker compose up -d --build srpm-scan-agent`。
