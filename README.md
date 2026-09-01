前端 vue + element-plus-x


后端 : fastapi 

模型： langchain ，langgraph，langfuse ，postgressalver


system-design : https://roadmap.sh/ai/roadmap-chat/system-design


任务：

1 ci/cd + 智能体  

2 知识库

3 智能体记忆，自我进化

4 任务拆解编排

5 评估模型

6 任务追踪

7 redis key 设计： 精确缓存+ 语义缓存


project:


1 jenkins  , sonarqube ,ai ,ubuntu server


2 测试数据生成

3 测试用例生成

4 浏览器自动化

5 乐乐rag知识库等，四路并行检索，召回率提升    ， 如何评估观测？  judge as llm  智能文档

6 微调？

7 



**********************************************构建智能体******************************************************************

构建成功的自主智能体系统，需要优先考虑五项工程原则：

可扩展性：系统应能承受业务量增长和任务多样化，通过分布式架构、云基础设施、并行处理和资源优化来扩展能力。

例如：一个每分钟只能处理 10 个工单的客服智能体，如果没有自动扩缩容能力，流量骤增到每分钟 1,000 个工单时，可能会卡死或崩溃。
模块化：将智能体拆分为独立、可替换的组件，并用清晰的接口连接。这样可以降低维护难度、增强灵活性，并更快适配新需求或新技术。

例如：若把所有工具都硬编码在智能体服务中，即使只是新增或修改一个小工具，也必须重新部署整个服务。
持续学习：让智能体能够从经验和反馈中改进，例如利用上下文学习、用户反馈闭环和评测数据持续优化行为。

例如：忽略反馈机制的智能体，可能持续重复同样错误，如误分类合同条款，或未能及时升级关键客服问题。
韧性：系统必须能够优雅处理错误、安全威胁、超时和意外情况。应包括错误处理、重试、降级、回退、安全控制与冗余机制。

例如：没有重试或回退逻辑的智能体，可能因一次 API 调用失败而整体崩溃，使用户长时间等待且不知发生了什么。
面向未来：围绕开放标准和可扩展基础设施构建系统，并保持对新技术的试验与适配能力，避免被单一供应商或专有接口锁定。

例如：如果智能体紧密绑定某个供应商的专有提示词格式，未来切换模型会非常痛苦，也会限制模型实验和技术迭代。
核心结论是：只有同时兼顾扩展能力、模块边界、学习反馈、故障恢复和技术演进，组织才能构建在技术变化和业务环境变化下仍然有效、可靠且可持续演进的智能体系统。


*********************************************************************

每一次智能体交互都遵循相同的基本模式：

准备上下文：组合任务、指令、记忆和对话历史。
调用模型：将上下文发送给大语言模型，并获得响应。
处理响应：处理文本回复，或者执行工具调用。
迭代：如果调用了工具，则将工具结果加入上下文，并从第 2 步重新开始。
返回：提供最终响应，并在适用时更新记忆。

# 简化的智能体执行循环伪代码

async def agent_execution_loop(task):
    context = prepare_context(task, instructions, memory, history)

    while not done:
        response = await model_client.create(context)

        if response.has_tool_calls:
            for tool_call in response.tool_calls:
                result = await execute_tool(tool_call)
                context.append(result)
        else:
            done = True

    update_memory(context)  # 可选
    return response




******************************************************************project*******************************************************************************

项目1 ：基于证据链、可控执行、持续学习的 AI 代码质量与安全修复平台<img width="967" height="107" alt="image" src="https://github.com/user-attachments/assets/87940fca-2eb6-4545-b52e-d08832a4209e" />


Jenkins
  │
  ├── 1. 编译与测试
  ├── 2. SonarQube Scan
  ├── 3. 获取 Quality Gate
  └── 4. 调用单 Agent
         │
         ├── SonarQube MCP：查询 Issue、规则和源码
         ├── Hindsight：检索规范与历史修复案例
         ├── LLM：排序、分析、生成修复代码
         ├── 输出 JSON / HTML / PR 评论
         └── 人工确认后写入 Hindsight


<img width="552" height="2524" alt="mermaid-2026-08-18-13-37" src="https://github.com/user-attachments/assets/7609484c-549b-4b86-b72a-4a090838912f" />




一、单 Agent 的输入
Jenkins 调用 Agent 时，只需要传入最小上下文：

{
  "repository": "company/order-service",
  "pull_request_number": 428,
  "commit_sha": "abc123def456",
  "branch": "feature/backup",
  "sonarqube_project_key": "company:order-service",
  "quality_gate_status": "ERROR",
  "build_url": "https://jenkins.example.com/job/order-service/428/"
}

二、Agent 使用的最小工具集
为了保持单 Agent 方案简单，第一期只开放必要工具。
search_sonar_issues_in_projects
get_project_quality_gate_status
show_rule
get_source_code
get_raw_source
get_guidelines


工具用途
工具	用途
search_sonar_issues_in_projects	获取当前项目或当前 PR 的未解决问题
get_project_quality_gate_status	获取质量门状态
show_rule	获取 Sonar 规则的详细说明
get_source_code	获取问题行附近代码
get_raw_source	获取需要修复的完整源文件
get_guidelines	获取 SonarQube 中的编码或安全指南



Hindsight
Hindsight 只承担两个职责：

1. 检索：历史修复方案、编码规范、安全规范
2. 入库：保存经人工确认且 CI 验证通过的修复经验

建议检索内容：

text
- 相同 Sonar Rule Key 的历史修复案例
- 相同语言和框架的修复方式
- 当前项目或模块的编码规范
- 企业安全编码规范
- 经测试和人工审核通过的历史 Patch



三、问题优先级排序规则
不建议仅依据 SonarQube 的严重等级排序。单 Agent 可以使用一个简单、可解释的风险评分模型。

最终优先级分数 =
    Sonar 严重等级分
  + 问题类型分
  + 是否为新增问题
  + 是否位于当前 PR 修改范围
  + 是否存在可复用的高质量修复方案

推荐评分标准
维度	条件	分数
严重等级	BLOCKER	100
严重等级	CRITICAL	80
严重等级	MAJOR	50
严重等级	MINOR	20
问题类型	VULNERABILITY	+30
问题类型	BUG	+20
问题类型	CODE_SMELL	+5
PR 关联	当前 PR 新增	+25
PR 关联	位于本次修改文件中	+15
Hindsight	命中已验证修复方案	+10
代码规范	违反企业安全/架构规范	+20
优先级分层
分数	优先级	处理策略
≥ 100	P0：阻断级	建议修复后再合并
80–99	P1：高优先级	本次 PR 优先修复
50–79	P2：普通问题	建议本次或后续迭代修复
< 50	P3：低优先级	仅汇总，不生成大量评论
四、单 Agent 的核心 Prompt
text
你是企业代码质量修复智能体。

你的职责只有四项：

1. 根据 SonarQube Issue 的严重等级、问题类型和是否属于当前变更，
   对问题进行优先级排序。

2. 读取 SonarQube 提供的规则、问题代码和上下文，
   识别问题根因。

3. 从 Hindsight 检索：
   - 当前项目的代码规范；
   - 企业安全规范；
   - 相同 Sonar 规则的已验证历史修复方案。

4. 输出：
   - 问题优先级；
   - 问题根因；
   - 可执行的修复建议；
   - 可直接参考或应用的修复代码；
   - 建议补充的测试场景；
   - 需要写入 Hindsight 的知识记录草稿。

约束：

- 不得建议关闭、忽略或降低 SonarQube Issue 的严重等级。
- 修复方案必须符合 Hindsight 检索到的代码规范。
- 不得将未验证的 AI 建议直接标记为正式知识。
- 必须将事实、推理、修复建议分开输出。
- 若缺少代码上下文或规范依据，必须明确说明不确定性。
- 对 BLOCKER、CRITICAL 和安全漏洞，标记为“需要人工审核”。
- 输出严格符合 JSON Schema。
五、Agent 输出 JSON
这是整个简化方案的核心产物。HTML、GitHub 评论、Hindsight 入库记录均由该 JSON 渲染或转换。

json
{
  "analysis_id": "aqa-20260218-001",
  "repository": "company/order-service",
  "pull_request_number": 428,
  "quality_gate_status": "ERROR",
  "summary": {
    "total_issues": 4,
    "p0_count": 1,
    "p1_count": 1,
    "p2_count": 2,
    "merge_recommendation": "BLOCK"
  },
  "findings": [
    {
      "issue_key": "AXxx123",
      "priority": "P0",
      "priority_score": 135,
      "severity": "CRITICAL",
      "type": "VULNERABILITY",
      "rule_key": "java:S2076",
      "title": "潜在命令注入风险",
      "location": {
        "file": "src/main/java/com/company/BackupService.java",
        "start_line": 92,
        "end_line": 92
      },
      "facts": [
        "参数 backupPath 来自 HTTP 请求。",
        "参数被直接拼接到命令字符串。",
        "代码通过 Runtime.exec 执行系统命令。"
      ],
      "root_cause": "未校验的外部输入被拼接进入系统命令。",
      "hindsight_context": {
        "guideline_ids": [
          "SEC-CODE-012"
        ],
        "memory_ids": [
          "mem-java-command-injection-001"
        ],
        "memory_summary": "历史项目采用 ProcessBuilder 参数化调用和路径白名单校验修复同类问题。"
      },
      "fix_recommendation": {
        "summary": "使用 ProcessBuilder 参数列表替代 Shell 命令字符串拼接。",
        "steps": [
          "禁止直接拼接外部参数到命令字符串。",
          "对允许执行的操作类型实施枚举白名单。",
          "对文件路径进行规范化与目录边界校验。",
          "使用低权限账户运行备份命令。"
        ]
      },
      "code_patch": {
        "language": "java",
        "file": "src/main/java/com/company/BackupService.java",
        "before": "Runtime.getRuntime().exec(\"backup \" + backupPath);",
        "after": "Path safePath = validateBackupPath(backupPath);\nProcessBuilder processBuilder = new ProcessBuilder(\"backup\", safePath.toString());\nprocessBuilder.redirectErrorStream(true);\nProcess process = processBuilder.start();"
      },
      "test_recommendations": [
        "验证合法路径可以正常执行备份。",
        "验证包含命令分隔符的输入被拒绝。",
        "验证路径穿越输入被拒绝。",
        "验证不在允许目录范围内的路径被拒绝。"
      ],
      "requires_human_approval": true,
      "hindsight_write_candidate": {
        "enabled": true,
        "status": "candidate",
        "write_condition": "人工确认修复，且 Jenkins 构建、测试和 SonarQube Quality Gate 均通过后入库。"
      }
    }
  ]
}
六、修复代码输出原则
Agent 生成“可用代码”时，建议输出以下三部分：

text
1. 修改前代码
2. 修改后代码
3. 必要的辅助方法和测试建议
例如：

java
// 修改前：存在风险
public void executeBackup(String backupPath) throws IOException {
    Runtime.getRuntime().exec("backup " + backupPath);
}
java
// 修改后：参数化执行 + 路径校验
public void executeBackup(String backupPath) throws IOException {
    Path safePath = validateBackupPath(backupPath);

    ProcessBuilder processBuilder = new ProcessBuilder(
        "backup",
        safePath.toString()
    );

    processBuilder.redirectErrorStream(true);
    Process process = processBuilder.start();

    if (process.exitValue() != 0) {
        throw new IOException("备份命令执行失败");
    }
}

private Path validateBackupPath(String backupPath) {
    Path allowedBasePath = Paths.get("/data/backup")
        .toAbsolutePath()
        .normalize();

    Path targetPath = Paths.get(backupPath)
        .toAbsolutePath()
        .normalize();

    if (!targetPath.startsWith(allowedBasePath)) {
        throw new IllegalArgumentException("备份路径不在允许目录范围内");
    }

    return targetPath;
}
需要注意：Agent 生成的代码应被标记为：

text
修复候选代码（Candidate Patch）
只有通过以下验证后，才能视为可采纳方案：

text
编译通过
+ 单元测试通过
+ SonarQube 扫描通过
+ 人工 Code Review 确认
七、Hindsight 入库设计
1. 不直接写入正式知识库
建议采用两阶段入库：

text
Agent 生成修复建议
       │
       ▼
写入 Candidate Memory（候选知识）
       │
       ▼
开发者采纳修复
       │
       ▼
Jenkins 编译 / 测试 / SonarQube 验证通过
       │
       ▼
人工确认
       │
       ▼
升级为 Validated Memory（已验证知识）
2. 推荐的 Hindsight 入库结构
json
{
  "memory_type": "validated_sonar_fix",
  "status": "candidate",
  "repository": "company/order-service",
  "module": "backup",
  "language": "Java",
  "framework": "Spring Boot",
  "sonar_rule_key": "java:S2076",
  "issue_type": "VULNERABILITY",
  "severity": "CRITICAL",
  "problem_pattern": "external_input_to_os_command",
  "root_cause": "外部输入被直接拼接到操作系统命令字符串。",
  "solution_pattern": "ProcessBuilder 参数化调用 + 路径白名单校验",
  "code_guideline_refs": [
    "SEC-CODE-012",
    "JAVA-CODE-023"
  ],
  "test_cases": [
    "合法路径",
    "非法目录",
    "路径穿越",
    "命令分隔符",
    "特殊字符"
  ],
  "validation": {
    "build_passed": false,
    "tests_passed": false,
    "quality_gate_passed": false,
    "human_approved": false
  },
  "source": {
    "pull_request": 428,
    "commit": "abc123def456",
    "sonar_issue_key": "AXxx123"
  }
}
当修复完成并验证后，更新为：

json
{
  "status": "validated",
  "validation": {
    "build_passed": true,
    "tests_passed": true,
    "quality_gate_passed": true,
    "human_approved": true
  }
}
八、简化版 Jenkins Pipeline
groovy
pipeline {
    agent any

    environment {
        SONAR_PROJECT_KEY = 'company:order-service'
        AGENT_API_URL = credentials('ai-code-fix-agent-url')
        AGENT_API_TOKEN = credentials('ai-code-fix-agent-token')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build and Test') {
            steps {
                sh './mvnw clean verify'
            }
        }

        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv('sonarqube-prod') {
                    sh """
                        ./mvnw sonar:sonar \
                          -Dsonar.projectKey=${SONAR_PROJECT_KEY}
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    script {
                        def qualityGate = waitForQualityGate(
                            abortPipeline: false
                        )
                        env.QUALITY_GATE_STATUS = qualityGate.status
                    }
                }
            }
        }

        stage('Single AI Fix Agent') {
            steps {
                sh """
                    curl --fail --request POST "${AGENT_API_URL}/analyze" \
                      --header "Authorization: Bearer ${AGENT_API_TOKEN}" \
                      --header "Content-Type: application/json" \
                      --data '{
                        "repository": "${GIT_URL}",
                        "pull_request_number": "${CHANGE_ID}",
                        "commit_sha": "${GIT_COMMIT}",
                        "branch": "${BRANCH_NAME}",
                        "sonarqube_project_key": "${SONAR_PROJECT_KEY}",
                        "quality_gate_status": "${QUALITY_GATE_STATUS}",
                        "build_url": "${BUILD_URL}"
                      }' \
                      --output ai-quality-report.json
                """
            }
        }

        stage('Archive AI Report') {
            steps {
                archiveArtifacts artifacts: 'ai-quality-report.json', fingerprint: true
            }
        }
    }
}
九、最终展示给用户的内容
针对开发者，建议只展示一份简洁的 PR 评论：

markdown
## 🤖 AI 修复建议

本次 PR 检测到：

| 优先级 | 数量 | 建议 |
|---|---:|---|
| 🔴 P0 | 1 | 合并前必须修复 |
| 🟠 P1 | 1 | 建议本次修复 |
| 🟡 P2 | 2 | 可纳入后续迭代 |

### 🔴 P0：潜在命令注入

- **文件**：`BackupService.java:92`
- **Sonar 规则**：`java:S2076`
- **根因**：外部参数直接拼接进系统命令。
- **建议**：使用 `ProcessBuilder` 参数化调用，并校验路径。

已生成候选修复代码和测试建议，请查看 Jenkins 构建产物：
[AI Quality Report](JENKINS_REPORT_URL)

> 修复方案会在人工确认、测试通过且 SonarQube 质量门通过后，沉淀到 Hindsight 知识库。
最终的简化架构可以概括为：

text
Jenkins
  → SonarQube MCP 获取问题和代码
  → 单 Agent 排序、检索 Hindsight、生成修复建议与代码
  → JSON / PR 评论展示
  → 人工确认 + CI 通过
  → Hindsight 入库
这套方案足够轻量，适合先快速上线；后续再逐步扩展为多 Agent、自动生成 Patch、自动创建修复 PR 和全组织质量治理平台。


架构图：

<img width="2826" height="1837" alt="image" src="https://github.com/user-attachments/assets/72b0610a-9ffa-425a-a239-867e39abda8c" />




最终完全版：

可以。加入 `AsyncPostgresSaver` 后，建议采用下面这套完整架构：

> **AsyncPostgresSaver 管工作流与对话状态；PostgreSQL/审计库管事实与证据；Hindsight 管长期语义经验；SonarQube 管检测与验证；GitHub MCP 管代码与 PR；Jenkins 管构建、测试和隔离扫描。**

---

# 1. 最终总体架构

```text
                         ┌──────────────────────┐
                         │      SonarQube       │
                         │  主项目：gpcs         │
                         │  Web API + Webhook   │
                         └──────────┬───────────┘
                                    │
                           扫描完成 Webhook
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────┐
│                  Agent Orchestrator / LangGraph                      │
│                                                                    │
│  AsyncPostgresSaver                                                 │
│  - 对话状态                                                         │
│  - LangGraph Checkpoint                                             │
│  - 工具调用状态                                                     │
│  - 等待人工审批                                                     │
│  - 等待 Jenkins / SonarQube Webhook                                │
│  - 失败恢复、幂等、断点续跑                                         │
│                                                                    │
│  工作流：                                                           │
│  1. SonarQube 问题筛选                                              │
│  2. Hindsight Recall                                                │
│  3. Agent 给出项目化建议                                            │
│  4. 等待人工确认                                                    │
│  5. 调用 GitHub MCP 修改 AI 分支                                    │
│  6. 调用 Jenkins 执行测试                                           │
│  7. 验证 PR / Diff / SonarQube 结果                                │
│  8. 生成审计证据包                                                  │
│  9. Hindsight Retain                                                │
└──────────┬──────────────────────┬─────────────────────┬───────────┘
           │                      │                     │
           ▼                      ▼                     ▼
┌─────────────────┐   ┌───────────────────┐  ┌──────────────────────┐
│ SonarQube Tool  │   │ GitHub MCP         │  │ Jenkins Tool         │
│                 │   │                   │  │                      │
│ - Quality Gate  │   │ - Commit / Diff   │  │ - Build              │
│ - Metrics       │   │ - Files           │  │ - Unit Test          │
│ - Priority      │   │ - AI Branch       │  │ - Integration Test   │
│ - Issues        │   │ - PR / Review     │  │ - Sonar Scanner      │
│ - Rules         │   │ - Comments        │  │ - Artifact / Logs    │
└─────────────────┘   └───────────────────┘  └──────────────────────┘
           │
           ▼
┌────────────────────────────────────────────────────────────────────┐
│                         Hindsight Memory                            │
│                                                                    │
│ conversation:<tenant>:<user>                                      │
│   - 用户长期偏好、输出要求、审批习惯                                │
│                                                                    │
│ sonarqube:<tenant>:<project>                                      │
│   - 已验证修复经验、模块例外、失败反例、项目规范                    │
│                                                                    │
│ policy:<tenant>                                                    │
│   - 组织安全策略、禁止路径、自动修复白名单、审批规则                │
└────────────────────────────────────────────────────────────────────┘
```

---

# 2. 各组件职责

| 组件 | 职责 | 不能替代什么 |
|---|---|---|
| `AsyncPostgresSaver` | LangGraph Checkpoint、对话状态、任务中断恢复、人工审批恢复 | 不能替代 Hindsight 的语义检索 |
| PostgreSQL 业务表 / 审计库 | 扫描任务映射、修复批次、验证状态、幂等键、审计索引 | 不能替代 Git/PR 的代码事实 |
| GitHub / GitLab | Commit、Diff、PR、Review、代码历史、回滚基础 | 不能判断 SonarQube Issue 是否消失 |
| Jenkins | 编译、测试、扫描、构建日志、Artifact | 不能决定项目经验是否应沉淀 |
| SonarQube | 静态检测、Issue、Rule、Quality Gate、最终扫描验证 | 不能提供项目级处理偏好 |
| Hindsight | 项目经验、用户偏好、例外、反例的语义召回 | 不能作为工作流 Checkpoint 或审计主库 |
| Agent | 综合决策、生成建议、执行受控修复 | 不能绕过主干保护与人工审批 |

---

# 3. 数据存储分层

建议采用四层，而不是把所有数据塞到 Hindsight。

```text
第一层：AsyncPostgresSaver
第二层：业务状态数据库
第三层：审计与证据存储
第四层：Hindsight 长期语义记忆
```

---

## 3.1 第一层：AsyncPostgresSaver

保存 LangGraph 的精确工作状态。

例如：

```text
当前 thread_id
当前对话消息
当前图节点
当前工具调用
当前候选 Issue
当前 PR 编号
当前 Jenkins Build ID
当前 SonarQube taskId
是否等待人工审批
是否等待扫描完成
是否已执行 Hindsight Retain
```

建议的 `thread_id`：

```text
scan:gpcs:<sonar-task-id>
pr:company/gpcs:<pr-number>
conversation:<tenant>:<user>:<session-id>
```

例如：

```text
scan:gpcs:AX0abc123
pr:company/gpcs:1024
conversation:company-a:zhangsan:20260424-001
```

典型状态：

```text
RECEIVED_SONAR_WEBHOOK
→ FETCHING_PRIORITY_ISSUES
→ RECALLING_PROJECT_MEMORY
→ WAITING_FOR_HUMAN_APPROVAL
→ APPLYING_PATCH
→ WAITING_FOR_JENKINS
→ WAITING_FOR_SONAR_VALIDATION
→ GENERATING_AUDIT_PACKAGE
→ RETAINING_HINDSIGHT_MEMORY
→ COMPLETED
```

---

## 3.2 第二层：业务状态数据库

`AsyncPostgresSaver` 用于恢复 LangGraph 状态，但修复系统还需要独立业务表。

建议至少有：

```text
fix_batches
scan_runs
issue_snapshots
audit_events
idempotency_keys
approval_records
```

### `fix_batches`

一个 PR 对应一个修复批次。

```text
repository
pr_number
base_branch
head_branch
base_commit
head_commit
merge_commit
status
created_at
updated_at
```

例如：

```text
company/gpcs
PR #1024
main
ai/sonar-remediation
```

---

### `scan_runs`

建立 Git Commit 与 SonarQube 扫描的关联。

```text
repository
branch
commit_sha
jenkins_build_id
sonar_project_key
sonar_task_id
sonar_analysis_id
status
started_at
completed_at
```

自然唯一键：

```text
repository + commit_sha + sonar_task_id
```

这样即使同一 PR 连续触发 5～6 次 CI，也不会混乱。

---

### `issue_snapshots`

只保存候选文件和候选规则的 Issue 快照，不保存全项目全部 Issue。

```text
snapshot_id
repository
project_key
commit_sha
analysis_id
file_path
rule_key
issue_fingerprint
issue_key_at_time
message
severity
impact_quality
impact_severity
status
```

---

### `idempotency_keys`

防止重复 Webhook、重复 Jenkins 回调、重复 Hindsight 写入。

例如：

```text
sonar-webhook:<task-id>
jenkins-build:<job>:<build-number>
hindsight-retain:<repo>:<merge-sha>:<issue-fingerprint>
audit-package:<repo>:<merge-sha>
```

---

## 3.3 第三层：审计与修复证据库

建议建立独立仓库：

```text
company/ai-remediation-audit
```

目录：

```text
ai-remediation-audit/
└── gpcs/
    └── 2026/
        └── 04/
            └── pr-1024-abc123def456/
                ├── summary.md
                ├── evidence.json
                ├── sonar-before.json
                ├── sonar-after.json
                ├── jenkins-result.json
                ├── pr-diff.patch
                └── hindsight-result.json
```

用途：

```text
人工复盘
审计
事故调查
回滚依据
模型行为追踪
证明修复目的与验证证据
```

---

## 3.4 第四层：Hindsight Memory Bank

建议至少三个 Bank。

```text
conversation:<tenant>:<user>
sonarqube:<tenant>:<project>
policy:<tenant>
```

例如：

```text
conversation:company-a:zhangsan
sonarqube:company-a:gpcs
policy:company-a
```

---

# 4. Hindsight Bank 设计

## 4.1 用户对话记忆

```text
conversation:<tenant>:<user>
```

保存：

```text
用户偏好中文输出；
修复建议必须包含风险、修改文件、测试与回滚方式；
用户不接受 Agent 自动合并 main；
用户希望低风险问题按批次处理；
用户对某个项目的关注重点。
```

不保存：

```text
全部对话原文；
完整代码；
完整日志；
Token；
密码；
业务敏感数据。
```

完整对话和当前工作流状态仍由：

```text
AsyncPostgresSaver
```

保存。

---

## 4.2 项目修复知识

```text
sonarqube:<tenant>:<project>
```

例如：

```text
sonarqube:company-a:gpcs
```

保存：

```text
已验证的修复模式；
项目模块例外；
规则在本项目中的实际处理方法；
失败修复反例；
架构限制；
经过审批的不处理理由。
```

例如：

```text
在 gpcs 的 Spring 配置类中，
java:S1128 的未使用 import 一般可安全删除；
不得顺带调整 Bean 或缓存配置。

在 gpcs 的 EditorConfig 模块中，
replaceAll() 可能依赖正则表达式；
未经确认不得自动替换为 replace()。

legacy-batch 模块的外部协议 DTO，
java:S107 参数过多通常不自动重构；
因为字段和构造函数需保持外部协议兼容。
```

---

## 4.3 组织策略 Bank

```text
policy:<tenant>
```

保存只读规则：

```text
自动修复白名单；
禁止修改路径；
安全问题处理规则；
PR 审批要求；
日志脱敏规则；
Hindsight 写入条件；
模型权限边界。
```

例如：

```text
禁止自动修改：
- Jenkinsfile
- .github/workflows/
- db/migration/
- auth/
- security/
- payment/
- Dockerfile
- helm/
- terraform/

禁止直接提交 main。
禁止自动合并 PR。
新增 Vulnerability 必须人工审批。
```

---

# 5. SonarQube 扫描完成后的分析流程

```text
SonarQube 主干扫描完成
        │
        ▼
Webhook 到达编排服务
        │
        ▼
AsyncPostgresSaver 创建 / 恢复 scan thread
        │
        ▼
校验 taskId、签名、扫描状态、幂等键
        │
        ▼
查询 Quality Gate、项目指标、Issue 聚合
        │
        ▼
按优先级获取有限候选 Issue
        │
        ▼
Hindsight Recall：
policy → 项目经验 → 用户偏好
        │
        ▼
Agent 生成项目化修复建议
        │
        ▼
等待人工接受 / 修改后接受 / 拒绝 / 不处理
```

---

## 5.1 不全量读取 SonarQube Issue

对于 16,876 条 Issue，Agent 不能直接读取全部。

筛选顺序：

```text
1. Quality Gate 的失败原因
2. 新增 Vulnerability / Security 问题
3. 新增 BLOCKER / CRITICAL
4. HIGH 影响的 Maintainability / Reliability 问题
5. quickFixAvailable=true 的低风险问题
6. 历史遗留问题只做规则和模块聚合
```

建议单次上限：

```text
P0 / P1：全部
P2：最多 20 条
P3：最多 30 条
P4：只传统计、Top Rule、Top 文件和少量样本
```

---

# 6. 人工确认与固定 AI 修复分支

人工批准后，Agent 才能修改代码。

```text
main
  └── ai/sonar-remediation
```

规则：

```text
Agent 仅能写入 ai/sonar-remediation。
Agent 不得 push main。
Agent 不得 merge PR。
Agent 不得修改受保护文件。
Agent 不得处理安全和高风险业务逻辑。
```

---

## 6.1 批次策略

一个批次可包含多个低风险问题。

例如：

```text
Commit A：
- 删除 3 个未使用 import；

Commit B：
- 删除 2 个无用变量；

Commit C：
- 修复 4 个低风险冗余代码问题。
```

不需要：

```text
一条 Issue 一个分支；
一条 Issue 一个 PR；
一条 Issue 一个复杂 relationId。
```

批次边界建议：

```text
最多 20～50 个 Issue；
最多 30 个文件；
最多 500 行有效改动；
最多 3 个模块；
最多积累 1～3 天。
```

达到阈值就创建或更新 PR：

```text
ai/sonar-remediation → main
```

---

# 7. Jenkins Pipeline 设计

建议至少有两个 Pipeline。

```text
gpcs-ai-remediation-pipeline
gpcs-main-validation-pipeline
```

可选：

```text
gpcs-revert-ai-remediation-pipeline
```

---

## 7.1 AI 修复分支 Pipeline

```text
Checkout ai/sonar-remediation
      │
      ▼
编译
      │
      ▼
单元测试
      │
      ▼
集成测试
      │
      ▼
依赖 / Secret / 安全检查
      │
      ▼
隔离 SonarQube 扫描
projectKey = gpcs-ai-remediation
      │
      ▼
读取 .report-task.txt
      │
      ▼
上报：
commitSha ↔ Jenkins Build ↔ ceTaskId
```

Jenkins 上报示例：

```json
{
  "repository": "company/gpcs",
  "branch": "ai/sonar-remediation",
  "commitSha": "cccccccc",
  "pullRequest": 1024,
  "jenkinsBuild": 103,
  "sonarProjectKey": "gpcs-ai-remediation",
  "sonarTaskId": "AX0abc123"
}
```

---

## 7.2 主干最终验证 Pipeline

PR 合并后：

```text
Checkout main
      │
      ▼
编译 + 全量测试
      │
      ▼
主项目 SonarQube 扫描
projectKey = gpcs
      │
      ▼
读取 .report-task.txt
      │
      ▼
上报：
mergeCommit ↔ Jenkins Build ↔ ceTaskId
```

只有主干最终扫描才有资格触发：

```text
Hindsight Retain
```

---

# 8. GitHub MCP 在修复验证中的作用

GitHub MCP 主要负责确认：

```text
Agent 是否真的按建议修改了相应代码。
```

核心工具：

```text
pull_request_read
get_commit
get_file_contents
list_commits
create_or_update_file
push_files
create_pull_request
update_pull_request
pull_request_review_write
add_issue_comment
actions_get
get_job_logs
```

验证优先使用：

```text
pull_request_read
```

因为它能够获取：

```text
Base Commit
Head Commit
文件变更列表
Diff
Commit 列表
Check 状态
Review 信息
```

---

## 8.1 Git Diff 与 SonarQube Snapshot 联合验证

例如 SonarQube 修复前存在：

```text
Rule：java:S1128
File：CacheConfig.java
Message：Remove this unused import EnableCaching.
```

Git Diff：

```diff
-import org.springframework.cache.annotation.EnableCaching;
```

主干最终扫描后，该文件不再存在相同规则、相似上下文的 Issue。

则判断：

```text
VERIFIED_RESOLVED
```

---

# 9. Issue Snapshot 与 Issue key 变化处理

不能仅依据：

```text
旧 SonarQube Issue key 是否消失。
```

因为 Issue key、行号、路径可能随重构变化。

应使用问题语义指纹：

```text
projectKey
+ ruleKey
+ 标准化文件路径
+ 标准化问题消息
+ 代码片段或上下文
+ 软件质量影响
```

例如：

```text
gpcs
+ java:S1128
+ src/main/java/jp/co/sws/gpcs/CacheConfig.java
+ Remove this unused import
+ import org.springframework.cache.annotation.EnableCaching;
+ MAINTAINABILITY
```

结果状态：

| 状态 | 含义 |
|---|---|
| `VERIFIED_RESOLVED` | Diff 修改正确，最终扫描中问题消失 |
| `STILL_PRESENT` | 最终扫描中问题仍存在 |
| `AMBIGUOUS` | 文件移动、重构或 Issue 追踪变化，无法自动确认 |
| `RESOLVED_BY_FILE_DELETION` | 文件删除导致 Issue 消失，不可当作通用修复经验 |
| `NEW_HIGH_RISK_ISSUE` | 修复引入新的高风险问题 |
| `TEST_FAILED` | 测试失败，不可写 Hindsight |

---

# 10. 修复历史与审计证据包

每个合并后的 AI 修复 PR，都必须生成证据包。

自然审计标识：

```text
github:company/gpcs:pr:1024:merge:abc123def456
```

至少包含：

```text
修复的问题；
修复目的；
SonarQube 原始证据；
Agent 建议；
Hindsight Recall 的项目经验依据；
修改文件；
Git Diff；
Base / Head / Merge Commit；
Jenkins Build；
测试结果；
SonarQube 最终扫描结果；
Quality Gate 前后状态；
未解决或新增问题；
人工审批人；
回滚方式；
Hindsight 写入结果。
```

---

## 10.1 人类可读报告

Agent 应在 PR 中生成 Markdown 报告。

```markdown
# AI 修复报告

## 修复目的

处理 SonarQube 检测出的低风险可维护性问题。

## 涉及规则

| 规则 | 文件 | 问题 | 处理方式 |
|---|---|---|---|
| java:S1128 | CacheConfig.java | 未使用 import | 删除无用 import |
| java:S5361 | EditorConfig.java | replaceAll 可优化 | 经确认后使用 replace |

## 修改文件

- `CacheConfig.java`
  - 删除未使用的 `EnableCaching` import；
  - 不修改 Spring Bean、缓存配置和业务逻辑。

## 验证

- Jenkins 编译：通过
- 单元测试：通过
- 集成测试：通过
- SonarQube 主干扫描：通过
- Quality Gate：通过
- 新增 Critical / Vulnerability：0

## 回滚

如发现异常，请针对 Merge Commit `abc123def456` 创建 Revert PR。
```

---

# 11. Hindsight Retain 条件

只有满足以下所有条件，才能写入项目记忆：

```text
PR 已合并到 main
AND main Jenkins 成功
AND 测试成功
AND SonarQube 主干扫描成功
AND 修复前候选 Issue 在最终扫描中消失
AND Git Diff 证明目标代码确实被修改
AND 没有新增 P0 / P1 问题
AND 修复审计证据包已生成
```

写入的是经验，不是原始日志。

示例：

```text
项目：gpcs
规则：java:S1128
场景：Spring 配置类中的未使用 import
处理方式：删除无用 import，不重构配置或 Bean 定义。
适用范围：普通配置类。
不适用范围：条件编译、注解处理、生成代码依赖场景。
证据：PR #1024、Merge Commit abc123def456、主干 Sonar 扫描通过。
置信度：HIGH
```

---

# 12. 回滚方案

AI 修复出问题时必须采用标准 Git 回滚流程：

```text
发现问题
    │
    ▼
暂停 AI 自动修复任务
    │
    ▼
从审计证据包取得 Merge Commit
    │
    ▼
创建 Revert PR
    │
    ▼
Jenkins 编译、测试与扫描
    │
    ▼
人工审批
    │
    ▼
合并回滚 PR
    │
    ▼
更新审计记录和 Hindsight 状态
```

禁止：

```text
force push main
git reset --hard main
删除 Commit 历史
Agent 直接修改主干
```

---

# 13. 推荐状态机

```text
SONAR_WEBHOOK_RECEIVED
    ↓
PRIORITY_ISSUES_SELECTED
    ↓
PROJECT_MEMORY_RECALLED
    ↓
RECOMMENDATION_CREATED
    ↓
WAITING_FOR_HUMAN_APPROVAL
    ↓
APPROVED
    ↓
PATCH_APPLIED_TO_AI_BRANCH
    ↓
WAITING_FOR_JENKINS
    ↓
AI_BRANCH_VALIDATED
    ↓
PR_OPENED
    ↓
WAITING_FOR_REVIEW
    ↓
MERGED
    ↓
WAITING_FOR_MAIN_SONAR_VALIDATION
    ↓
FINAL_VERIFICATION
    ├─ VERIFIED_RESOLVED
    ├─ STILL_PRESENT
    ├─ AMBIGUOUS
    ├─ TEST_FAILED
    └─ NEW_HIGH_RISK_ISSUE
    ↓
AUDIT_PACKAGE_CREATED
    ↓
HINDSIGHT_RETAINED
    ↓
COMPLETED
```

每一步都由：

```text
AsyncPostgresSaver
```

保存 Checkpoint，因此即使服务重启、Webhook 延迟、审批跨天，也可以恢复执行。

---

# 14. 最终闭环

```text
SonarQube 主干扫描完成
      │
      ▼
Webhook 触发 LangGraph
      │
      ▼
AsyncPostgresSaver 保存与恢复工作流状态
      │
      ▼
SonarQube Tool 仅获取高优先级候选 Issue
      │
      ▼
Hindsight Recall：
组织策略 → 项目经验 → 用户偏好
      │
      ▼
Agent 输出项目化修复建议
      │
      ▼
人工确认
      │
      ▼
GitHub MCP 修改 ai/sonar-remediation 固定分支
      │
      ▼
Jenkins 构建、测试、隔离扫描
      │
      ▼
GitHub MCP 获取 PR Diff、文件变更与 Check 状态
      │
      ▼
创建 PR，人工 Review
      │
      ▼
PR 合并 main
      │
      ▼
main Jenkins + SonarQube 最终扫描
      │
      ▼
修复前 Snapshot vs 最终 Snapshot
      │
      ▼
生成审计证据包
      │
      ▼
Hindsight Retain 已验证项目经验
      │
      ▼
后续相似问题优先参考项目本地经验
```

这套设计可以保证：

- Agent 不会被大量 Issue 撑爆上下文；
- 不会为每条 Issue 创建分支；
- 多次 CI、多次扫描可以精确关联；
- SonarQube Issue key 变化不会导致验证失效；
- 自动修复不会直接影响主干；
- 修复过程可审计、可复盘、可回滚；
- Hindsight 只积累经过验证的项目经验；
- `AsyncPostgresSaver` 保证长流程、人工审批和异步 Webhook 场景下可恢复、可追踪、可幂等执行。

ps:threa_id按照项目来定义，也就是一个项目一个thread_id,所有人共用一个chat，对于审批，修改文件等写操作采用锁串行，其他读操作（比如读取issues）等串行


*******************************************************project2*****************************************************************************************

内部设计，详细设计 ---》 测试式样书


agent评测： 

历史设计书 + 历史测试式样书
        ↓
标准化、脱敏、切分、人工抽检
        ↓
评测 Dataset（黄金集）
        ↓
Agent 生成测试式样书
        ↓
规则评测 + LLM Judge + 人工抽检
        ↓
Langfuse Dataset / Experiment / Score
        ↓
发现失败模式，补充 Dataset，优化 Prompt / Agent / 模型


langfuse 评测 ：


1. Langfuse
   → POST 调用你的 Webhook

2. FastAPI
   → 收到 dataset 信息、实验信息、config
   → 创建后台任务
   → 立即 return 202 / accepted

3. 后台任务
   → 获取 Dataset Items
   → 对每条 Item 调用 Deep Agent
   → 生成测试式样书 JSON / Excel
   → 执行 Code Evaluator、LLM Judge
   → 通过 Langfuse SDK / API 写入 output 和 scores

4. Langfuse
   → 在 Experiment 页面展示每条运行结果和汇总评分



后台伪代码：


def execute_experiment(payload: dict) -> None:
    # 注意：
    # dataset、experiment、run、config 的真实字段名称，
    # 以 FastAPI 实际收到的 Langfuse payload 为准。
    dataset_info = payload.get("dataset", {})
    experiment_info = payload.get("experiment", {})
    config = payload.get("config", {})

    dataset_id = dataset_info.get("id")
    dataset_name = dataset_info.get("name")
    experiment_id = experiment_info.get("id")

    # 1. 通过 Langfuse SDK / API 获取 Dataset Item
    dataset_items = get_dataset_items(
        dataset_id=dataset_id,
        dataset_name=dataset_name,
    )

    # 2. 逐条执行 Agent
    for item in dataset_items:
        agent_result = generate_unit_test_spec(
            design_input=item["input"],
            agent_version=config.get("agent_version"),
            prompt_version=config.get("prompt_version"),
            model_deployment=config.get("model_deployment"),
        )

        # 3. 规则评测
        rule_scores = run_rule_evaluators(
            expected_output=item["expected_output"],
            agent_output=agent_result,
        )

        # 4. LLM Judge 语义评测
        judge_scores = run_llm_judge(
            design_input=item["input"],
            expected_output=item["expected_output"],
            agent_output=agent_result,
        )

        # 5. Excel 校验
        excel_score = validate_excel(
            excel_path=agent_result.get("excel_path"),
        )

        # 6. 调用 Langfuse SDK / API，
        #    将 Item 的 output、trace、score 关联到本次 experiment。
        post_result_to_langfuse(
            experiment_id=experiment_id,
            dataset_item_id=item["id"],
            input_data=item["input"],
            output_data=agent_result,
            scores=[
                *rule_scores,
                *judge_scores,
                excel_score,
            ],
        )



def execute_experiment(payload: dict) -> None:
    # 注意：
    # dataset、experiment、run、config 的真实字段名称，
    # 以 FastAPI 实际收到的 Langfuse payload 为准。
    dataset_info = payload.get("dataset", {})
    experiment_info = payload.get("experiment", {})
    config = payload.get("config", {})

    dataset_id = dataset_info.get("id")
    dataset_name = dataset_info.get("name")
    experiment_id = experiment_info.get("id")

    # 1. 通过 Langfuse SDK / API 获取 Dataset Item
    dataset_items = get_dataset_items(
        dataset_id=dataset_id,
        dataset_name=dataset_name,
    )

    # 2. 逐条执行 Agent
    for item in dataset_items:
        agent_result = generate_unit_test_spec(
            design_input=item["input"],
            agent_version=config.get("agent_version"),
            prompt_version=config.get("prompt_version"),
            model_deployment=config.get("model_deployment"),
        )

        # 3. 规则评测
        rule_scores = run_rule_evaluators(
            expected_output=item["expected_output"],
            agent_output=agent_result,
        )

        # 4. LLM Judge 语义评测
        judge_scores = run_llm_judge(
            design_input=item["input"],
            expected_output=item["expected_output"],
            agent_output=agent_result,
        )

        # 5. Excel 校验
        excel_score = validate_excel(
            excel_path=agent_result.get("excel_path"),
        )

        # 6. 调用 Langfuse SDK / API，
        #    将 Item 的 output、trace、score 关联到本次 experiment。
        post_result_to_langfuse(
            experiment_id=experiment_id,
            dataset_item_id=item["id"],
            input_data=item["input"],
            output_data=agent_result,
            scores=[
                *rule_scores,
                *judge_scores,
                excel_score,
            ],
        )





***************************************目标***************************************************


智能文档：PaddleOCR-VL，开源免费，可本地部署，包含多个模型,   
模型路由（根据文档类型（pdf，doc，xls），当文档只有文字的时候，默认使用小模型，不适用ocr，当设计ocr时候，在调用ocr，当效果不佳，调用更强大的模型，提升能力）
默认单agent ，文档过长，上下文暴了的时候多agent（最大token量的60%）

共享结构化状态 + 文档片段引用 + 任务消息传递。
Agent 之间传“结论、证据 ID、任务 ID”，不传整份原文。




原始设计书 / Excel / PDF
        ↓
[文档解析层：非 Agent]
        ↓
结构化章节、表格、Sheet、Cell、Chunk
        ↓
[规划 Agent]
        ↓
按功能 / 画面 / API / 批处理划分任务
        ↓
[规则抽取 Agent 群，可并行]
        ↓
Design Facts + Evidence
        ↓
[规则汇总 / 去重 / 冲突检测]
        ↓
按功能形成“规格事实包”
        ↓
[测试场景生成 Agent 群，可并行]
        ↓
Black-box Test Scenarios
        ↓
[测试用例汇总 Agent]
        ↓
去重、覆盖检查、未定义项、输出 JSON
        ↓
[Excel Renderer：非 Agent]
        ↓
测试式样书 Excel
        ↓
Code Evaluator + LLM Judge + 人工抽检



详细设计书（Word / Excel）
  ↓
文本抽取、OCR、章节识别
  ↓
Deep Agent
  ├── 按画面 / API / 批处理 / 功能模块拆分
  ├── 提取输入、处理、输出、DB、异常、权限、状态转换
  ├── 识别遗漏或设计歧义
  └── 生成测试观点评价清单
  ↓
LLM（结构化输出）
  ├── 正常系
  ├── 异常系
  ├── 境界値
  ├── 条件组合
  ├── 权限
  ├── 状态迁移
  └── DB / API / 日志确认项
  ↓
测试用例 JSON
  ↓
xlsxwriter
  ├── 测试式样书 Sheet
  ├── 测试数据 Sheet
  ├── 设计—测试追溯矩阵 Sheet
  └── 未决事项 Sheet
  ↓
Excel：単体テスト仕様書.xlsx






核心目标是建立以下闭环：
```text
历史设计书 + 历史测试式样书
        ↓
标准化、脱敏、切分、人工抽检
        ↓
评测 Dataset（黄金集）
        ↓
Agent 生成测试式样书
        ↓
规则评测 + LLM Judge + 人工抽检
        ↓
Langfuse Dataset / Experiment / Score
        ↓
发现失败模式，补充 Dataset，优化 Prompt / Agent / 模型



//端到端测试 item
## Dataset Item 示例
{
  "input": {
    "feature": "ログイン画面",
    "spec_text": "USERIDまたはPASSWORDが未入力の場合、エラーメッセージを表示する。..."
  },
  "expected_output": {
    "required_scenarios": [    //llm judge 增加语义识别，防止误报
      "正常登录",
      "USERID未输入",
      "PASSWORD未输入",
      "用户不存在",
      "已删除用户登录",
      "密码错误",
      "登录成功后的履历登记"
    ],
    "forbidden_assumptions": [   //llm judge 增加语义识别，防止误报
      "账户连续失败锁定",
      "验证码",
      "设计书未定义的错误码"
    ]
  },
  "metadata": {
    "feature_type": "login",
    "risk_level": "high",
    "dataset_tier": "gold"
  }
}



Langfuse 检测：
通用 ，包括agent开始，tool调用，大模型调用等等


自定义：  doc解析，code生成（？），


