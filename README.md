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

![Uploading image.png…]()

