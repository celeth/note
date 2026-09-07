

## 内容总结

这段文本介绍了 **Langfuse 中与实验（Experiments）相关对象的数据模型**：

- 本页面说明实验相关对象的数据模型。
- 如需了解这些对象如何协同工作，应查看 **Concepts（概念）** 页面。
- 关于 **Score（评分）** 和 **Score Config（评分配置）** 对象，应查看 **Scores data model（评分数据模型）**。
- 详细接口文档可参考：
  - Python SDK 参考文档
  - JS/TS SDK 参考文档
  - API 参考文档
- 文中开始介绍 **Datasets（数据集）**：
  - 数据集是一组输入，以及可选的预期输出。
  - 它们可以用于执行数据集运行（Dataset runs）。
  - 一个数据集由多个 **DatasetItem（数据集条目）** 组成。

---

## 中文翻译

# 实验数据模型

本页面介绍 Langfuse 中与实验相关对象的数据模型。有关这些对象如何协同工作的概述，请参阅 **Concepts（概念）** 页面。有关 **评分（score）** 和 **评分配置（score config）** 对象，请参阅 **评分数据模型（Scores data model）**。

如需查看详细参考资料，请参阅：

- Python SDK 参考文档
- JS/TS SDK 参考文档
- API 参考文档

## 对象

### 数据集（Datasets）

数据集是一组输入，以及可选的预期输出，可在数据集运行（Dataset runs）期间使用。

数据集由多个数据集条目（DatasetItems）组成。



---

**User** _(2026-08-24 09:52:46)_

rag评估一般用什么，langfuse可以用于rag评估吗，和ragas的区别

---

**Bot** _(2026-08-24 09:53:12)_

RAG 评估通常不是只看一个指标，而是分为 **检索质量、生成质量、端到端效果、线上运行质量** 四个层面。**Langfuse 可以用于 RAG 评估**；它与 **Ragas 并不是完全替代关系**，更常见、也更实用的方式是把二者结合起来使用。

---

## 1. RAG 评估一般评估什么？

### A. 检索（Retrieval）质量

核心是：系统检索到的文档是否真的有用、是否覆盖了回答问题所需的信息。

常见指标：

| 指标 | 含义 |
|---|---|
| **Context Precision / 上下文精确率** | 检索出的内容中，有多少是与问题真正相关的；用于衡量“噪声多不多”。 |
| **Context Recall / 上下文召回率** | 回答问题需要的关键信息，有多少被成功检索出来；用于衡量“漏召回多不多”。 |
| **Hit Rate / 命中率** | Top-K 检索结果中是否至少包含一个正确或相关文档。 |
| **MRR** | 第一个相关文档的排名是否靠前。 |
| **NDCG@K** | 同时考虑检索结果的相关性和排序位置。 |

如果数据集有人工标注的“标准相关文档”，可以使用 Hit Rate、MRR、NDCG 等传统信息检索指标。  
如果没有文档相关性标注，则常通过 LLM-as-a-Judge 或 Ragas 一类工具估计上下文质量。

### B. 回答（Generation）质量

核心是：模型最终答案是否正确、相关、完整、易读。

常见指标：

| 指标 | 含义 |
|---|---|
| **Answer Correctness / 回答正确性** | 答案是否与标准答案或事实一致。 |
| **Answer Relevancy / 回答相关性** | 答案是否真正回应了用户问题，有没有跑题或冗余。 |
| **Completeness / 完整性** | 是否覆盖了问题要求的关键点。 |
| **语义相似度** | 生成答案与参考答案在语义上是否接近。 |
| **人工偏好评分** | 人工或裁判模型对多个答案进行比较、排序。 |

### C. 忠实性与引用质量

这是 RAG 中特别关键的一类评估：模型是否只依据检索到的上下文回答，而不是“看似合理地编造”。

| 指标 | 含义 |
|---|---|
| **Faithfulness / 忠实性** | 回答中的陈述能否从检索上下文得到支持。 |
| **Groundedness / 有据性** | 与忠实性相近，强调回答是否基于证据。 |
| **Citation Correctness / 引用正确性** | 引用的文档是否确实支持对应结论。 |
| **Citation Completeness / 引用完整性** | 需要引用支持的关键主张是否都提供了引用。 |

### D. 线上工程与业务指标

离线分数高不代表产品体验一定好，因此还应关注：

- **延迟**：检索、重排、生成分别耗时多少，P50/P95/P99 延迟；
- **成本**：每次请求的 Token、模型调用费用、检索成本；
- **错误率**：超时、空检索、解析失败、工具调用失败；
- **用户反馈**：点赞/点踩、满意度、追问率、人工转接率；
- **业务结果**：问题解决率、转化率、工单减少率等。

---

## 2. Langfuse 可以用于 RAG 评估吗？

**可以。** Langfuse 很适合用于构建 RAG 的可观测性、测试集实验和评估闭环。

它通常可以记录一条 RAG 请求的完整链路，例如：

```text
用户问题
  → Query Rewrite（可选）
  → Embedding
  → Vector Search / BM25 Search
  → Rerank
  → Prompt Assembly
  → LLM Generation
  → 最终答案与引用
  → 评分与用户反馈
```

在 Langfuse 中，可以将这些环节记录为 trace、span / observation、generation 等对象（具体对象名称取决于所使用的 SDK 和版本），并保存：

- 输入问题；
- 检索 Query；
- 候选文档、Top-K 文档及其分数；
- 重排前后的结果；
- Prompt 和模型参数；
- 生成的最终答案；
- Token 用量、成本、延迟；
- 参考答案、人工标签；
- 自动评估分数与评估理由。

### Langfuse 对 RAG 评估特别有价值的场景

1. **线上可观测性**
   - 某个回答为什么错误？
   - 是没有检索到资料、召回结果噪声过多、重排失败，还是模型没有遵循上下文？
   - 哪个模型、提示词、知识库版本导致了退化？

2. **离线数据集实验**
   - 建立一组问题、参考答案和期望上下文；
   - 对比不同的 embedding 模型、分块策略、Top-K、reranker、Prompt、生成模型；
   - 将各版本结果放在同一实验中横向比较。

3. **自动与人工评分汇总**
   - 接入基于规则的评分、LLM Judge、人工审核或用户反馈；
   - 按知识库版本、模型版本、问题类型、租户、语言等维度分析分数。

4. **生产回归监控**
   - 新版上线后，持续比较忠实性、正确性、延迟、成本等指标；
   - 发现某类问题、某一文档集合或某个版本的性能下降。

---

## 3. Langfuse 和 Ragas 的区别

一句话概括：

> **Ragas 更像是“RAG 评估指标与计算框架”；Langfuse 更像是“可观测性、数据集、实验、评分与运营平台”。**

| 维度 | Langfuse | Ragas |
|---|---|---|
| 核心定位 | LLM 应用的可观测性与评估平台 | 面向 RAG 的评估框架/指标库 |
| 主要能力 | Trace、调用链、成本、延迟、数据集、实验、评分、线上监控 | 计算 RAG 质量指标，如忠实性、回答相关性、上下文精确率等 |
| 是否专注 RAG | 不仅限于 RAG，也适用于 Agent、聊天机器人、工作流等 | 主要面向 RAG 评估 |
| 检索链路调试 | 强：可检查每次检索、重排、Prompt、模型输出 | 不是核心重点，主要输出评估结果 |
| 线上观测 | 强，适合追踪真实生产请求 | 通常更偏离线批量评估 |
| 数据集与实验管理 | 支持数据集、运行、实验结果对比 | 可以加载数据并计算指标，但不是完整实验运营平台 |
| 评估指标 | 可记录任意自定义/自动/人工评分 | 内置并聚焦许多 RAG 专用指标 |
| 可视化与协作 | 强：UI、筛选、追踪、比较、团队协作 | 主要通过 Python 代码、Notebook 或自行保存结果 |
| 部署形态 | 云服务或自托管平台 | Python 库，通常在你的计算环境运行 |
| 适合人群 | 需要将评估接入开发、测试和生产运行全流程的团队 | 需要快速、系统化计算 RAG 指标的开发者/研究人员 |

---

## 4. 如何选择？

### 只用 Ragas 的情况

如果你当前目标是：

- 已经有 CSV、JSON 或 DataFrame 格式的测试集；
- 想快速计算忠实性、上下文相关性、答案相关性等 RAG 指标；
- 主要在 Notebook、Python 脚本或 CI 中做离线评估；
- 暂时不需要线上 trace、成本监控和团队协作界面；

那么 **Ragas 单独使用就足够合适**。

### 只用 Langfuse 的情况

如果你更关心：

- 生产环境中每次 RAG 调用的完整过程；
- 排查失败案例；
- 管理不同 Prompt、模型、知识库和检索配置的实验；
- 记录人工审核、用户反馈和自定义评分；
- 追踪成本、Token 和延迟；
- 将评估结果与实际线上请求对应起来；

那么可以以 **Langfuse 为主**，用自定义 evaluator 或 LLM Judge 完成评估。

### 推荐：Langfuse + Ragas

对于多数生产级 RAG 项目，推荐组合方式：

```text
Langfuse：记录、管理、对比、监控
        +
Ragas：计算 RAG 专项自动化指标
```

典型流程：

1. 在 Langfuse 创建数据集：保存问题、参考答案、期望文档或人工标签。
2. 运行不同 RAG 版本：例如不同分块大小、embedding、Top-K、重排模型、Prompt。
3. 对每条运行结果用 Ragas 计算：
   - Context Precision；
   - Context Recall；
   - Faithfulness；
   - Answer Relevancy；
   - Answer Correctness。
4. 将 Ragas 计算结果作为 score 回写或关联到 Langfuse 的实验记录。
5. 在 Langfuse 中按模型、检索策略、知识库版本、问题类别查看趋势和失败样本。
6. 对低分样本进行人工复核，形成新的测试集和改进闭环。

---

## 5. 一个实用的最小指标组合

如果刚开始做 RAG 评估，不建议一次引入过多指标。可以先从以下组合开始：

| 目标 | 建议指标 |
|---|---|
| 检索是否找对资料 | Context Precision、Context Recall 或 Hit Rate@K |
| 回答是否有依据 | Faithfulness / Groundedness |
| 回答是否解决问题 | Answer Correctness、Answer Relevancy |
| 系统能否上线 | P95 延迟、单请求成本、错误率 |
| 用户是否满意 | 点踩率、人工抽检分数、追问率 |

对于有标准答案、标准文档标注的高价值场景，例如客服、合规、医疗辅助或企业知识问答，应优先加入：

- 人工标注的文档相关性；
- 事实正确性人工审核；
- 引用正确性和完整性；
- 高风险问题的拒答/升级策略评估。

---

## 结论

- **Ragas** 擅长回答：**“这套 RAG 的检索和回答质量是多少分？”**
- **Langfuse** 擅长回答：**“这次回答为什么得了这个分、问题在哪个链路、哪个版本更好、线上是否正在退化？”**
- 对生产级 RAG 而言，二者结合通常优于二选一：  
  **Ragas 提供评估指标能力，Langfuse 提供可观测、实验管理和持续优化闭环。**



---

**User** _(2026-08-24 09:58:16)_

也就是说  langfuse可以用来观测召回的文档对不对，召回率等等，而ragas只能用来做端到端测试

---

**Bot** _(2026-08-24 09:58:33)_

不完全对，关键区别在于：**“观测与记录”** 和 **“计算评估指标”** 是两件事。

## 更准确的理解

| 能力 | Langfuse | Ragas |
|---|---|---|
| 查看每次请求召回了哪些文档 | **可以，且很擅长** | 可以作为评估输入保存/查看，但不是它的核心 UI 能力 |
| 追踪文档排名、相似度分数、Top-K、重排前后结果 | **可以**，前提是你的代码把这些数据记录到 trace/span 中 | 可用于计算相关指标，但通常不负责生产调用链追踪 |
| 计算召回率（Recall）、MRR、NDCG | **可以做**，但需要你有“标准相关文档”并配置/写评估逻辑 | **可以做**，这是其 RAG 专项评估能力的一部分 |
| 计算 Context Recall / Context Precision | 可通过自定义 evaluator、LLM Judge 或集成评估方式实现 | **可以**，是典型能力 |
| Faithfulness、Answer Relevancy 等 | 可以记录任何评分，也可调用评估器计算 | **可以**，是典型能力 |
| 端到端批量回归测试 | 可以通过 Dataset / Experiment 进行 | **可以**，常见用法 |
| 生产环境问题排查与成本/延迟监控 | **强项** | 不是重点 |

---

## 1. Langfuse：可以观测“召回了什么”，但不会自动知道“召回得对不对”

例如一次 RAG 请求中，你可以在 Langfuse 里记录：

```json
{
  "question": "公司的差旅报销标准是什么？",
  "retrieval_query": "公司差旅报销标准",
  "top_k": 5,
  "retrieved_documents": [
    {
      "document_id": "travel-policy-2025",
      "chunk_id": "chunk-12",
      "score": 0.89,
      "content": "机票、住宿及交通费用的报销规定……"
    },
    {
      "document_id": "it-security",
      "chunk_id": "chunk-7",
      "score": 0.72,
      "content": "员工使用公司设备时应遵守安全规定……"
    }
  ],
  "final_answer": "……"
}
```

这样你就能在 Langfuse 中检查：

- 这次到底召回了哪些文档和文本块；
- Top-K 设置是否合理；
- 相似度分数是否异常；
- 重排是否把正确文档排到了后面；
- 回答错误是因为**没有召回正确资料**，还是模型**没有正确使用已经召回的资料**；
- 哪个知识库版本、embedding 模型或 reranker 导致效果变差。

但注意：

> Langfuse 只看到“系统实际召回了哪些文档”，并不能天然知道这些文档是不是正确答案所需的文档。

要计算传统的**召回率（Recall@K）**，你必须有 ground truth，例如：

```text
问题：公司的差旅报销标准是什么？
标准相关文档：travel-policy-2025
实际 Top-5 召回：it-security、travel-policy-2025、...
```

此时才可以计算：

```text
Recall@5 = 已召回的相关文档数 / 全部相关文档数
```

你可以把这个计算逻辑写成 Langfuse 的自定义评分器，并把结果记录为一个 score。

---

## 2. Ragas 不只是做端到端测试

**Ragas 并非只能做端到端测试。** 它也能评估检索阶段，只是它一般以“离线数据集批量评估”的方式工作。

Ragas 常见的检索相关指标包括：

- **Context Precision**：召回上下文中有多少是有用的，衡量噪声；
- **Context Recall**：回答所需的信息是否被上下文覆盖，衡量遗漏；
- **Context Relevancy**：召回上下文与问题的相关程度；
- 在你提供文档级 ground truth 的情况下，也可以结合传统的检索指标逻辑评估召回效果。

同时，它还可评估生成阶段：

- **Faithfulness**：答案是否能由召回上下文支撑；
- **Answer Relevancy**：答案是否回应问题；
- **Answer Correctness**：答案是否与参考答案一致或接近。

所以 Ragas 的典型输入是一条完整记录：

```python
{
    "question": "公司的差旅报销标准是什么？",
    "contexts": [
        "机票、住宿及交通费用的报销规定……",
        "员工使用公司设备时应遵守安全规定……"
    ],
    "answer": "员工可报销符合标准的机票、住宿和交通费用……",
    "ground_truth": "按照公司差旅政策，符合规定的机票、住宿和市内交通费用可报销。"
}
```

它会根据 `question`、`contexts`、`answer`、`ground_truth` 的不同组合，计算检索相关和回答相关的指标。

---

## 3. 最简洁的分工方式

可以这样记：

> **Langfuse：这次线上请求“发生了什么”？**  
> **Ragas：这一批 RAG 结果“质量怎么样”？**

### Langfuse 更适合

- 记录每一次真实请求中的召回文档；
- 查看检索分数、重排结果、Prompt、回答、Token、成本和延迟；
- 定位线上失败样本；
- 管理数据集与实验；
- 汇总、筛选和对比评分结果；
- 监控上线后版本是否退化。

### Ragas 更适合

- 对一批测试问题自动计算 RAG 质量；
- 衡量检索上下文的精确性、覆盖性和相关性；
- 衡量回答的忠实性、相关性和正确性；
- 在 CI/CD 或离线实验中做批量回归测试。

---

## 4. 推荐组合

最常见的做法不是二选一：

```text
业务请求
  ↓
Langfuse 记录：问题、召回文档、相似度、重排、Prompt、答案、耗时、成本
  ↓
Ragas 批量评分：Context Precision、Context Recall、Faithfulness、Answer Correctness
  ↓
将评分结果写回 Langfuse
  ↓
在 Langfuse 中定位低分案例，比较不同检索/模型/Prompt 版本
```

因此，修正后的结论是：

- **Langfuse 可以观测召回的文档、排序、分数和完整 RAG 链路；也可以保存并展示召回率等分数，但前提是你提供标准答案/标准文档或接入评估器。**
- **Ragas 不只做端到端测试，也能评估检索质量；它更侧重于计算 RAG 专用质量指标，而不是生产环境的调用链可观测性。**



---

**User** _(2026-08-24 10:02:23)_

rag检索中的意图树是什么，有没有第三方框架

---

**Bot** _(2026-08-24 10:03:03)_

RAG 检索中的**意图树（Intent Tree）**，通常是指将用户问题按“业务领域 → 子领域 → 具体任务”的层级进行分类，并据此决定**去哪里检索、用什么方式检索、是否需要拆分问题**的一种路由结构。

它不是 RAG 的必备组件，而是当知识库规模变大、领域变多、检索策略变复杂时，用来提升召回准确性和可控性的设计模式。

---

## 1. 什么是意图树？

例如，一个企业内部知识助手可能有如下意图树：

```text
企业知识问答
├── 人力资源
│   ├── 假期与考勤
│   ├── 薪酬与福利
│   └── 招聘与入职
├── 财务与报销
│   ├── 差旅报销
│   ├── 采购付款
│   └── 发票与税务
├── IT 服务
│   ├── 账号与权限
│   ├── 设备与网络
│   └── 信息安全
└── 产品与技术
    ├── 产品文档
    ├── API 使用
    └── 故障排查
```

用户提问：

> “去上海出差住酒店每天最多报销多少？”

系统可先识别：

```text
企业知识问答
└── 财务与报销
    └── 差旅报销
        └── 酒店住宿标准
```

然后不再全库检索，而是优先或仅在以下范围内检索：

- `finance/travel/` 对应的向量索引；
- 差旅制度、报销标准、城市等级表；
- 带有 `department=finance`、`topic=travel` 等元数据的文档块。

---

## 2. 它在 RAG 链路中放在哪里？

典型链路如下：

```text
用户问题
  ↓
意图识别 / 路由
  ↓
确定知识库、索引、过滤条件、检索策略
  ↓
向量检索 / 关键词检索 / 混合检索
  ↓
重排（Rerank）
  ↓
组装上下文
  ↓
LLM 生成答案
```

更具体一些：

```text
问题：
“VPN 连不上，如何排查？”

意图识别：
IT 服务 → 设备与网络 → VPN 故障排查

检索配置：
- 指定知识库：IT 运维知识库
- metadata filter：category = "network" AND topic = "vpn"
- 使用 Hybrid Search
- Top-K = 20
- Rerank 后取 Top-5

生成：
仅基于最终检索到的 VPN 文档回答
```

---

## 3. 意图树解决什么问题？

### 3.1 降低跨领域误召回

如果将 HR、财务、产品、技术文档全部放进同一个索引，问题较短或存在歧义时，容易召回错误领域内容。

例如：

> “怎么申请？”

它可能是：

- 申请年假；
- 申请采购；
- 申请服务器权限；
- 申请 API Key；
- 申请发票。

意图树可先询问澄清问题，或根据会话上下文路由到正确知识域。

### 3.2 支持不同领域采用不同检索策略

不同意图不一定使用同一套检索参数：

| 意图 | 更合适的检索策略 |
|---|---|
| 制度、合规、报销标准 | 精确关键词 + 元数据过滤 + 引用原文 |
| 产品/API 文档 | 向量检索 + 代码/标题关键词检索 |
| 故障排查 | 多轮检索、查询改写、按步骤匹配 |
| 数据查询 | 不一定检索文档，可能路由到 SQL、BI 或工具调用 |
| 实时订单/库存/天气 | 路由到 API，而非静态 RAG 知识库 |

因此，意图树的本质不只是“文档分类”，而是一个**检索与工具调用的决策层**。

### 3.3 提升评估和问题定位能力

有意图树后，可以按意图节点统计：

- `财务 → 差旅报销` 的 Recall@K；
- `IT → VPN` 的回答正确率；
- 哪个节点最容易误路由；
- 哪个节点的知识库内容不足；
- 哪类问题应增加澄清步骤。

这也很适合结合 Langfuse：记录预测意图、实际检索库、过滤条件、召回文档和最终评估分数。

---

## 4. 意图树不等于知识图谱，也不等于目录树

这几个概念容易混淆：

| 概念 | 核心作用 | 例子 |
|---|---|---|
| **意图树** | 决定如何理解问题、去哪里查、调用什么能力 | “报销问题 → 差旅报销知识库” |
| **文档目录/标签体系** | 组织文档和元数据 | `department=finance`、`topic=travel` |
| **向量索引** | 根据语义相似度找相关文本块 | 找到“上海住宿标准”对应 chunk |
| **知识图谱** | 表示实体与关系，并支持关系推理 | “员工 A 属于部门 B；制度 C 适用于 B” |
| **Query Plan / 查询计划** | 将复杂问题拆解为多个检索或工具步骤 | “先查政策，再查城市等级，再计算上限” |

实际项目中常常组合使用：

```text
意图树负责路由
  ↓
元数据过滤缩小范围
  ↓
混合检索召回候选文档
  ↓
Reranker 排序
  ↓
必要时从知识图谱或数据库补充结构化事实
```

---

## 5. 有哪些第三方框架或方案？

截至 **2026 年 2 月**，严格意义上专门提供“意图树”这一完整产品概念的通用 RAG 框架不算特别多；更多框架提供的是实现意图树所需的**路由、分类、元数据过滤、多索引检索和工作流编排能力**。

### 5.1 LangChain / LangGraph

适合：需要自定义路由逻辑、多步骤工作流和 Agent 编排的场景。

相关能力：

- Router Chain / 路由模式；
- Structured Output：将 LLM 分类结果限制为预定义意图；
- Runnable Branch：按分类结果分支执行；
- MultiQueryRetriever：从多个改写问题检索；
- Self-Query Retriever：从问题中提取元数据过滤条件；
- LangGraph：把意图树的每个节点建成状态机节点或工作流分支。

概念示例：

```text
分类器
  ├── HR → HR Retriever
  ├── Finance → Finance Retriever
  ├── IT → IT Retriever
  └── Product → Product Retriever
```

优点是灵活；缺点是树结构、分类评估、回退策略和可观测性通常需要自行设计。

---

### 5.2 LlamaIndex

适合：一个应用对接多个知识库、多个索引、多个数据源，并需要查询路由的场景。

相关能力包括：

- **Router Query Engine**：根据问题选择一个或多个 Query Engine；
- **ToolRetrieverRouterQueryEngine**：把不同数据源或查询引擎作为工具，根据语义选择；
- 多索引、多数据源查询；
- metadata filtering；
- Query Transformation、SubQuestion Query Engine；
- 文档节点与层级索引组织。

可以将不同意图节点映射为不同的 Query Engine：

```text
Finance Intent → 财务政策向量库 + BM25
IT Intent      → IT 知识库 + 工单系统
Product Intent → 产品文档库 + API 文档库
```

如果你的“意图树”主要目标是**多知识库/多索引路由**，LlamaIndex 往往是较直接的选择。

---

### 5.3 Haystack

适合：偏工程化、Pipeline 明确、希望每个组件可替换的 RAG 系统。

相关能力：

- Pipeline 编排；
- Router / ConditionalRouter；
- Document Joiner；
- 多 Retriever、多 Document Store；
- Metadata Filter；
- Hybrid Retrieval；
- Reranker；
- 评估组件和离线测试流程。

可以将意图分类器输出作为 Pipeline 中的路由变量：

```text
intent = "travel_expense"
  ↓
选择 Finance Document Store
  ↓
过滤 metadata.topic = "travel"
  ↓
Hybrid Retriever
  ↓
Reranker
```

---

### 5.4 Semantic Router

适合：意图集合相对稳定，想用低延迟方式完成路由，而不希望每次都调用大模型分类。

它的基本思路是：

1. 为每个路由定义若干典型 utterance；
2. 对用户输入进行 embedding；
3. 根据语义相似度选择路由；
4. 低于阈值时进入默认分支或让 LLM 做二次判断。

示意：

```text
route: finance_travel
examples:
- 出差酒店怎么报销
- 住宿费用上限是多少
- 差旅补贴标准

route: it_vpn
examples:
- VPN 无法连接
- 公司内网打不开
- VPN 登录失败
```

它比较适合意图树的**第一层或前几层粗分类**。  
对于层级很深、需要复杂业务规则的树，通常仍需搭配规则、LLM 分类器或工作流框架。

---

### 5.5 Rasa

适合：客服机器人、流程型对话、需要显式维护 Intent、Entity、Slot 和对话状态的业务。

Rasa 原本就是面向意图识别和对话管理的框架。它可以用于：

- 定义业务意图；
- 训练或配置意图分类；
- 提取实体，例如城市、日期、报销类型；
- 根据意图进入不同 RAG 检索或业务流程；
- 在低置信度时触发澄清或人工转接。

但它本身不是专门的 RAG 框架。更合理的组合是：

```text
Rasa：意图识别、槽位、对话流
  +
RAG 框架：文档检索与生成
  +
Langfuse：调用链追踪、评估与监控
```

---

### 5.6 向量数据库的 Metadata Filter

这不算完整的意图树框架，但常常是落地意图树最关键的一环。

例如在 Milvus、Qdrant、Weaviate、Pinecone、Elasticsearch / OpenSearch 等系统中，可以为文档或 chunk 添加元数据：

```json
{
  "department": "finance",
  "domain": "expense",
  "topic": "travel",
  "city_level": "tier_1",
  "effective_date": "2025-01-01",
  "access_role": ["employee", "finance"]
}
```

意图路由得到：

```json
{
  "department": "finance",
  "topic": "travel"
}
```

然后检索时加过滤条件：

```text
department = "finance" AND topic = "travel"
```

这往往比“为树中每个叶子节点单独建立一个向量库”更容易维护。

---

## 6. 一个推荐的落地架构

对大部分企业 RAG，不建议一开始构建特别深的硬编码意图树。更实用的是“三层路由 + 回退”的设计：

```text
用户问题
  ↓
第一层：粗粒度领域分类
  ├── HR
  ├── Finance
  ├── IT
  ├── Product
  ├── Data / SQL
  └── General
  ↓
第二层：抽取结构化检索条件
  ├── topic
  ├── document_type
  ├── product
  ├── region
  ├── time / policy version
  └── user permission
  ↓
第三层：选择执行策略
  ├── Vector Search
  ├── Hybrid Search
  ├── Rerank
  ├── SQL / API Tool
  ├── Web Search
  └── Clarification Question
```

以“上海出差酒店报销标准”为例：

```json
{
  "domain": "finance",
  "intent": "travel_reimbursement",
  "entities": {
    "expense_type": "hotel",
    "city": "上海"
  },
  "retrieval_strategy": "hybrid_with_rerank",
  "filters": {
    "department": "finance",
    "topic": "travel",
    "document_status": "active"
  },
  "confidence": 0.93
}
```

如果置信度不足，例如用户只问“怎么申请”，则不要强行路由到某个叶子节点，而应：

- 结合会话上下文；
- 执行全局检索；
- 或反问用户：“您是想申请年假、采购、报销还是系统权限？”

---

## 7. 实施时最重要的几个注意点

1. **不要把意图树设计得太细太深**  
   节点过多会导致数据不足、分类不稳定、维护成本高。通常先从 5～15 个一级领域开始。

2. **路由错误会直接破坏召回率**  
   即使向量检索和 reranker 很强，只要一开始被路由到错误知识库，正确文档就完全没有机会被召回。应单独评估：
   - Intent Accuracy；
   - Route Accuracy；
   - 每个意图节点的 Recall@K；
   - 低置信度问题的澄清率与误路由率。

3. **优先采用“共享索引 + 元数据过滤”**  
   除非数据隔离、权限或规模需求很强，否则不建议每个意图节点都维护独立向量库。

4. **保留全局回退检索**  
   局部检索结果质量低、无结果或分类置信度低时，应回退到全局混合检索，避免“路由错即无答案”。

5. **意图不是唯一信号**  
   除了意图，还要使用用户身份权限、会话上下文、时间有效性、产品版本、地域等条件。

6. **在 Langfuse 中记录完整路由信息**  
   建议至少记录：
   - 预测的领域和意图；
   - 置信度；
   - 提取的实体与 metadata filter；
   - 使用的检索器、索引和 Top-K；
   - 原始候选文档与重排后文档；
   - 最终答案；
   - 用户反馈及 Ragas / 人工评估分数。

---

## 结论

- **意图树**是 RAG 检索前的分层路由机制：先识别用户要解决的业务问题，再确定检索范围、过滤条件和检索策略。
- 它尤其适合多部门、多知识库、多工具调用的企业 RAG。
- 没有必须使用的单一“意图树框架”；常见组合是：
  - **LangChain / LangGraph**：灵活的路由与工作流；
  - **LlamaIndex**：多数据源、多索引的查询路由；
  - **Haystack**：工程化 Pipeline 和条件路由；
  - **Semantic Router**：轻量、高速的语义路由；
  - **Rasa**：显式意图与对话流程；
  - **向量数据库 Metadata Filter**：实现检索范围约束的基础能力。
- 推荐做法是：**轻量意图分类 + 元数据过滤 + 混合检索 + 重排 + 全局回退**，而非一开始建立庞大、僵硬的深层树。



---

**User** _(2026-08-24 10:05:46)_

rag flow

---

**Bot** _(2026-08-24 10:06:05)_

## 内容概述

“RAG Flow”通常指两种可能含义：

1. **RAG 的流程（Retrieval-Augmented Generation Flow）**：即检索增强生成系统从用户提问到最终回答的处理链路；
2. **RAGFlow**：一个开源的 RAG 应用/知识库引擎项目，常用于构建带文档解析、检索、引用和对话能力的企业知识库问答系统。

结合你刚才在问“意图树和第三方框架”，你大概率指的是第二种：**RAGFlow 框架**。

---

## 中文翻译

**RAG Flow**：RAG 流程；也可能指 **RAGFlow** 这一 RAG 框架/平台名称。

---

## RAGFlow 是什么？

**RAGFlow** 是一个面向 RAG 应用构建的开源项目，目标是帮助开发者搭建“基于企业文档的问答系统”。它将许多 RAG 工程组件集成在一起，而不只是提供一个 Python 检索库。

一般可以把它理解为：

```text
文档上传
  ↓
文档解析与切分
  ↓
向量化、索引与检索
  ↓
可选的重排与混合检索
  ↓
将检索结果作为上下文交给大模型
  ↓
生成答案，并尽量附带来源引用
```

它特别强调文档解析和“可追溯引用”，适用于 PDF、Word、PPT、网页、表格等企业知识文档较多的场景。

---

## 它和“意图树”的关系

RAGFlow 本身更偏向于提供完整的 RAG 基础设施，例如：

- 知识库管理；
- 文档上传、解析、切块；
- 检索与召回；
- 对话应用；
- 大模型配置；
- 来源引用；
- API 与界面。

而**意图树**是你可以部署在其检索前的一层路由策略：

```text
用户问题
  ↓
意图识别
  ├── 财务 → 财务知识库
  ├── HR → HR 知识库
  ├── IT → IT 知识库
  └── 产品 → 产品文档知识库
  ↓
RAGFlow 执行对应知识库的检索与生成
  ↓
最终回答 + 引用来源
```

也就是说：

> **RAGFlow 可以作为意图树下游的“知识库检索和问答执行引擎”。**  
> 但复杂的企业意图树、跨库路由、低置信度澄清、SQL/API 工具选择等逻辑，通常仍需要在应用层、工作流层或 Agent 层自行补充。

---

## RAGFlow 的典型优点

| 能力 | 对 RAG 的价值 |
|---|---|
| 文档解析 | 对复杂 PDF、扫描件、表格、章节结构等文档更友好。 |
| 可视化知识库管理 | 非研发人员也可以上传、更新和管理知识库。 |
| 检索与生成一体化 | 不需要从零拼接向量库、文档处理、检索和聊天 UI。 |
| 来源引用 | 有助于用户查看答案依据，也有利于排查幻觉。 |
| 多种模型/嵌入模型配置 | 可根据成本、语言、效果选择模型。 |
| API 化能力 | 可以集成到现有客服、内部助手或业务系统。 |
| 私有化部署 | 对企业内部资料、权限与合规要求较高的场景更合适。 |

---

## 局限与注意点

RAGFlow 并不能自动保证 RAG 质量。即使用了它，仍然要解决：

- 文档是否解析正确；
- Chunk 是否按语义合理切分；
- 元数据是否完整；
- Top-K、混合检索、重排参数是否合理；
- 是否正确做了权限过滤；
- 问题是否应该先经过意图路由；
- 回答是否真正基于召回文档；
- 召回率、忠实性、回答正确性是否达标。

因此，较完整的生产架构常是：

```text
Langfuse
  └── 记录调用链、文档召回、成本、延迟、评分与实验结果

意图路由层
  └── 判断问题属于哪个部门、知识库、工具或检索策略

RAGFlow
  └── 文档解析、知识库管理、召回、上下文构建、回答生成

Ragas / 自定义评估器
  └── 评估 Context Recall、Faithfulness、Answer Correctness 等
```

---

## 和 LangChain、LlamaIndex、Haystack 的区别

| 工具 | 更像什么 | 适用情况 |
|---|---|---|
| **RAGFlow** | 一体化 RAG 平台/知识库引擎 | 想快速搭建可用的文档问答、知识库产品和管理界面。 |
| **LangChain / LangGraph** | 应用开发与 Agent 工作流框架 | 需要复杂路由、多工具调用、多步骤流程。 |
| **LlamaIndex** | 数据连接、索引和查询框架 | 有多个数据源、索引和知识库，需要灵活查询路由。 |
| **Haystack** | 工程化 RAG Pipeline 框架 | 希望用明确、可替换的组件搭建可控检索流水线。 |
| **Langfuse** | 可观测性与评估平台 | 需要追踪、调试、实验比较、成本和质量监控。 |
| **Ragas** | RAG 评估指标框架 | 需要批量评估检索上下文和最终答案质量。 |

如果目标是**快速做企业内部知识库问答**，RAGFlow 很合适。  
如果目标是**高度定制的意图树、多系统查询、Agent 工作流**，通常需要将 RAGFlow 与 LangGraph/LlamaIndex 等路由编排能力结合，或直接使用这些框架自行实现检索层。



---

**User** _(2026-08-24 10:21:40)_

意图树是不是可以用agent做替代，用agent做语义路由

---

**Bot** _(2026-08-24 10:22:25)_

可以，但更准确地说：

> **Agent 可以实现或替代“意图树中的路由决策层”，但通常不应无差别地替代整个意图树。**

意图树是相对确定、可维护的路由结构；Agent 语义路由则让 LLM 根据问题、上下文和工具描述，动态决定应该检索哪个知识库、调用哪个工具或先追问什么。

---

## 两种方式的区别

| 维度 | 意图树 | Agent 语义路由 |
|---|---|---|
| 决策方式 | 预定义分类层级、规则、分类器 | LLM 基于语义和上下文动态决策 |
| 可控性 | 高 | 相对较低 |
| 灵活性 | 较低，新增意图通常要改配置/训练 | 高，新增工具或知识库后可通过描述接入 |
| 延迟与成本 | 通常低 | 通常更高，需要一次或多次 LLM 调用 |
| 可解释性 | 清晰：命中哪个节点、为什么过滤 | 需要记录 Agent 的 reasoning / tool selection 才易排查 |
| 适用任务 | 高并发、稳定、领域明确的知识问答 | 复杂、多域、问题类型变化大的任务 |
| 风险 | 分类错后可能检错库 | 可能选错工具、重复检索、过度调用或路径不稳定 |

---

## 用 Agent 做语义路由的典型形式

将不同知识库、检索器或外部系统封装为“工具”：

```text
Tools
├── search_hr_policy
│   └── 查询假期、薪酬、福利、入职等 HR 制度
├── search_finance_policy
│   └── 查询报销、预算、发票、采购政策
├── search_it_knowledge
│   └── 查询 VPN、账号、权限、设备、网络问题
├── search_product_docs
│   └── 查询产品功能、使用手册、API 文档
├── query_order_system
│   └── 查询实时订单、物流、库存
└── ask_clarification
    └── 信息不足时向用户提出澄清问题
```

用户提问：

> “我去上海出差住酒店，能报多少钱？”

Agent 可能输出类似的结构化路由决策：

```json
{
  "action": "search_finance_policy",
  "arguments": {
    "query": "上海出差 酒店住宿 报销上限",
    "filters": {
      "topic": "travel_expense",
      "status": "active"
    }
  }
}
```

然后由财务知识库的 RAG 检索器执行搜索。

---

## Agent 不等于一定比意图树好

例如以下问题：

> “VPN 无法登录怎么办？”

若系统长期稳定地只需路由到 IT 知识库，那么使用一次 LLM Agent 来选择工具，可能是“杀鸡用牛刀”：

```text
固定语义路由 / 小模型分类器 / 向量路由
  → IT 知识库
```

通常会比 Agent 更快、更便宜、更稳定。

但对于下面这种问题，Agent 更有价值：

> “我下周去上海出差，住宿上限是多少？如果超标，应该由谁审批？顺便帮我看看我的预算还剩多少。”

它可能需要：

1. 检索差旅报销政策；
2. 检索超标审批规则；
3. 调用预算或报销系统 API；
4. 合并多个来源；
5. 如果缺少员工级别、出差日期等必要信息，则追问用户。

这种动态多步骤任务很适合 Agent。

---

## 推荐架构：不要二选一，而是“确定性路由 + Agent 回退”

生产环境中较稳妥的设计通常是：

```text
用户问题
  ↓
第一层：轻量、确定性的语义路由
  ├── 高置信度 HR       → HR RAG
  ├── 高置信度 Finance  → Finance RAG
  ├── 高置信度 IT       → IT RAG
  └── 低置信度 / 跨领域 / 复杂任务
            ↓
第二层：Agent 路由和任务规划
  ├── 选择多个知识库
  ├── 选择检索、SQL、API 等工具
  ├── 分解复杂问题
  └── 必要时追问澄清问题
            ↓
第三层：检索、重排、生成与引用
```

这样有几个好处：

- 简单问题走快速路径；
- 降低 LLM 路由成本和延迟；
- 高风险领域可保留明确规则；
- 复杂、歧义、跨域问题交给 Agent；
- 避免 Agent 对每一个简单问题都“思考并规划”。

---

## 三种常见的语义路由实现

### 1. Embedding / 向量语义路由

为每个意图或工具准备多个示例语句，将用户问题与示例进行向量相似度匹配。

```text
“酒店报销上限是多少？”
  ↓
与 finance_travel 路由示例最相似
  ↓
路由到 Finance RAG
```

优点：

- 快；
- 成本低；
- 输出较稳定；
- 适合高频、意图固定的问题。

缺点：

- 对跨意图、复杂约束和长上下文理解能力有限；
- 需要设计阈值和兜底策略。

可考虑使用 Semantic Router，或自行用 embedding 模型和向量数据库实现。

### 2. LLM 结构化分类路由

让 LLM 只输出受约束的 JSON，而不是自由地执行完整 Agent 循环。

```json
{
  "route": "finance_rag",
  "confidence": 0.91,
  "needs_clarification": false,
  "filters": {
    "topic": "travel_expense"
  }
}
```

优点：

- 比向量路由更能理解上下文、否定条件和复杂表述；
- 比完整 Agent 更容易控制、评估和审计；
- 很适合作为意图树的“智能分类器”。

缺点：

- 有模型调用成本与延迟；
- 仍须校验 JSON、枚举值和权限条件。

对多数企业 RAG 来说，这是非常实用的折中方案。

### 3. Tool-calling Agent 路由

让模型根据工具说明，自主选择一个或多个检索器、API 或数据库。

优点：

- 最灵活；
- 支持跨库检索、复杂任务拆解和多轮调用；
- 可融合实时业务系统。

缺点：

- 成本、延迟、不可预测性更高；
- 需要严格限制可用工具、最大调用次数、权限和参数；
- 更难做稳定的离线回归评估。

---

## 实施时的关键约束

如果用 Agent 做语义路由，建议至少做到以下几点：

1. **工具白名单**  
   不允许模型随意构造 URL、SQL、数据库过滤条件或系统操作；只能选择预定义工具。

2. **参数结构化与服务端校验**  
   Agent 输出应通过 JSON Schema / Pydantic 等机制约束；服务端仍应验证字段、枚举、权限和过滤条件。

3. **设置最大工具调用次数**  
   防止循环检索、反复调用或成本失控。

4. **加入置信度和澄清分支**  
   对“怎么申请”“这个怎么处理”等模糊问题，优先追问，而不是猜测路由。

5. **权限过滤不能交给 Agent 决定**  
   用户能访问哪些文档、哪些业务数据，必须由后端身份认证和访问控制强制执行。

6. **保留全局回退检索**  
   当局部库无结果或召回质量低时，回退到全局检索或触发人工/澄清流程。

7. **记录完整链路**  
   在 Langfuse 中记录：
   - 输入问题和会话上下文；
   - 路由候选、最终工具选择和置信度；
   - 调用的知识库及 metadata filter；
   - 候选文档、重排结果；
   - 工具调用次数、延迟、成本；
   - 最终回答、引用和用户反馈。

---

## 如何评估 Agent 路由是否优于意图树？

不要只评估最终答案，还要单独评估路由层。

| 指标 | 含义 |
|---|---|
| Route Accuracy | 是否路由到正确知识库或工具 |
| Top-N Route Recall | 正确路由是否出现在前 N 个候选中 |
| Wrong-route Rate | 被明确路由到错误领域的比例 |
| Clarification Precision | 该追问时是否追问，不该追问时是否避免追问 |
| Retrieval Recall@K | 在选定路由后，正确文档是否被召回 |
| End-to-End Correctness | 最终答案是否正确 |
| Latency / Cost | 路由和工具调用是否带来可接受的时延与成本 |

---

## 结论

- **可以用 Agent 做语义路由**，尤其适用于跨领域、跨知识库、需调用 API/SQL、需要任务分解的复杂问题。
- **不建议完全取消意图树或确定性路由**：对于高频、简单、稳定或高风险的业务问题，固定路由、向量路由或结构化分类通常更快、更便宜、更可靠。
- 最推荐的生产方案是：

```text
轻量意图/语义路由处理简单且高置信度的问题
+
Agent 处理低置信度、跨域、多步骤和工具调用型问题
+
Langfuse 观测与评估整个路由—检索—生成链路
```



---

**User** _(2026-08-24 10:26:33)_

ragflow 和LlamaIndex

---

**Bot** _(2026-08-24 10:26:57)_

## 内容概述

你在询问 **RAGFlow 和 LlamaIndex 的区别、适用场景，以及如何选择**。两者都能用于构建 RAG 系统，但定位不同：

- **RAGFlow** 更偏向开箱即用的 RAG 平台和知识库产品；
- **LlamaIndex** 更偏向开发者使用的数据连接、索引、检索与 Agent/RAG 编排框架。

---

## 中文翻译

**RAGFlow 和 LlamaIndex**

---

## 核心区别

> **RAGFlow：更像可部署的“企业知识库问答系统底座”。**  
> **LlamaIndex：更像用于定制 RAG 与 Agent 应用的“开发框架”。**

| 维度 | RAGFlow | LlamaIndex |
|---|---|---|
| 核心定位 | 一体化 RAG 引擎/平台 | RAG、数据索引和 Agent 开发框架 |
| 主要使用方式 | Web UI、API、Docker/私有化部署 | Python / TypeScript 代码开发 |
| 上手速度 | 快，适合快速搭建知识库问答 | 需要编程和架构设计 |
| 文档解析 | 通常是其重点能力，面向复杂企业文档 | 支持多种 Reader 和解析器，但需自行组合和调优 |
| 知识库管理 UI | 通常内置 | 默认不提供完整业务管理后台 |
| 检索流程定制 | 有一定配置能力，但受平台抽象限制 | 很强，可自由定制 Retriever、Reranker、Router、Query Engine |
| Agent / 工具调用 | 可支持，但通常不是最灵活的核心优势 | 强项之一，适合多工具、多数据源、多步骤任务 |
| 多知识库路由 | 可实现，但复杂意图路由往往需要外围逻辑 | 有 Router Query Engine、Tool Routing 等能力，适合代码化实现 |
| 可观测性 | 需要结合 Langfuse 等工具完善 | 同样建议结合 Langfuse 等工具 |
| 适合对象 | 想快速交付知识库产品、偏低代码/平台化团队 | 有研发能力、需要高度定制的 AI 应用团队 |

---

## RAGFlow：适合“快速搭建企业知识库”

RAGFlow 通常更适合这样的需求：

```text
上传 PDF / Word / PPT / Excel / 网页
  ↓
自动解析、切分和建立知识库
  ↓
配置模型和检索参数
  ↓
创建聊天助手
  ↓
通过 UI 或 API 给员工、客户使用
```

它的优势通常在于：

- 有较完整的知识库管理体验；
- 文档解析和引用溯源能力较受重视；
- 更容易让非算法/非后端人员参与文档维护；
- 更容易快速部署出一个可演示、可使用的内部问答产品；
- 对“企业有很多 PDF、制度、手册、报告”的场景很友好。

### 适合的例子

- 公司制度、HR、财务、法务知识库；
- 产品手册和售后问答；
- 内部 SOP、运维手册、项目文档问答；
- 快速做 PoC 或内部试点；
- 希望使用已有知识库管理界面，而非从零做前端和后台。

### 可能的限制

如果你需要以下能力，RAGFlow 可能需要额外开发或外围服务：

- 非常复杂的意图树；
- 多个知识库、SQL、CRM、订单系统、搜索引擎之间的动态路由；
- 多 Agent 协作；
- 高度定制的检索策略和回退策略；
- 完全自主的权限模型、业务流程和用户界面；
- 将 RAG 深度嵌入已有业务系统。

---

## LlamaIndex：适合“深度定制 RAG 与 Agent”

LlamaIndex 更像是一套开发库。你通过代码来决定：

- 文档如何读取、清洗、切分；
- 如何建立索引；
- 使用什么向量数据库；
- 是否采用 BM25、向量检索、混合检索；
- 是否重排；
- 不同问题去哪个知识库；
- 哪些问题调用 SQL、API 或 Web Search；
- 如何进行多步任务拆分；
- 如何处理会话记忆和权限过滤。

一个典型的 LlamaIndex 架构可以是：

```text
用户问题
  ↓
Router / Agent
  ├── HR Query Engine
  ├── 财务 Query Engine
  ├── 产品文档 Query Engine
  ├── SQL 查询工具
  ├── CRM / 订单 API 工具
  └── 澄清问题工具
  ↓
选择一个或多个工具
  ↓
检索、重排、生成与引用
```

### 适合的例子

- 多部门、多知识库的企业助手；
- 结合 RAG、SQL、API、搜索和工作流的业务 Agent；
- 需要意图树或 Agent 语义路由；
- 需要实现复杂 Query Planning、多跳检索、子问题拆解；
- 想精细控制 Chunk、索引、检索、Rerank、Prompt 和输出格式；
- 需要把能力嵌入自己的 Web、App、客服或业务系统。

### 代价

- 你需要自行完成更多工程工作；
- 一般需要自行建设管理后台、文件上传流程、权限控制、任务队列等；
- 文档解析质量、检索效果和评估体系要自己负责；
- 框架灵活也意味着架构选择更多，项目复杂度更高。

---

## 在“意图树 / Agent 语义路由”场景下怎么选？

这是二者差距最明显的地方。

### 使用 RAGFlow 的模式

```text
外部意图路由器 / Agent
  ↓
选择 RAGFlow 中的某个知识库
  ↓
调用对应 RAGFlow API 检索与生成
```

例如：

```text
Finance Intent → finance_kb
HR Intent      → hr_kb
IT Intent      → it_kb
```

这种方式适合：

- 路由规则较清楚；
- 知识库边界明确；
- 主要目标是文档问答；
- 希望充分利用 RAGFlow 的文档解析和管理能力。

### 使用 LlamaIndex 的模式

```text
LlamaIndex Router / Agent
  ├── 财务知识库检索器
  ├── HR 知识库检索器
  ├── IT 知识库检索器
  ├── SQL 工具
  └── API 工具
```

这种方式适合：

- 问题可能跨多个知识域；
- 需要多个检索器并行或串行；
- 检索之外还需调用实时业务系统；
- 希望 Agent 动态规划执行步骤；
- 意图树会频繁变化或不断扩展。

---

## 是否可以组合使用？

**可以，而且是合理的架构。**

一种常见分工是：

```text
RAGFlow
  └── 负责文档上传、解析、知识库管理、基础检索与引用

LlamaIndex / LangGraph
  └── 负责意图识别、多知识库路由、Agent 编排、SQL/API 工具调用

Langfuse
  └── 负责观测：路由、召回文档、Prompt、模型输出、延迟、Token、成本

Ragas / 自定义评估器
  └── 负责离线评估：Context Recall、Faithfulness、Answer Correctness
```

不过，组合意味着系统复杂度上升。若你只有单一知识库问答需求，没必要同时引入两者。

---

## 选择建议

### 选 RAGFlow，如果你希望：

- 尽快上线一个可用的企业文档问答系统；
- 有大量复杂文档，尤其是 PDF、表格、扫描件、制度文件；
- 需要知识库管理界面；
- 团队希望少写底层 RAG 工程代码；
- 问题大多是“在一个知识库中查资料并回答”。

### 选 LlamaIndex，如果你希望：

- 用代码构建并完全控制 RAG 链路；
- 实现复杂意图树、Agent 路由和多知识库协同；
- 接入数据库、CRM、ERP、订单、工单等实时系统；
- 构建多步骤任务 Agent，而非单纯文档问答；
- 深度定制检索、重排、权限和业务逻辑。

### 实际落地建议

如果是从零开始，建议按需求复杂度选择：

```text
单知识库、文档问答、快速上线
  → 优先 RAGFlow

多个知识库、意图路由、复杂业务集成
  → 优先 LlamaIndex（可配合 LangGraph）

既要企业文档管理，又要复杂 Agent 编排
  → RAGFlow 作为知识库层
  + LlamaIndex / LangGraph 作为路由与编排层
```

对于你前面提到的“**意图树能否由 Agent 语义路由替代**”：

- 如果主要是 HR、财务、IT 等固定知识库之间路由，**RAGFlow + 轻量分类器/Embedding 路由**通常足够；
- 如果还要同时判断“查知识库、查订单、跑 SQL、调用审批 API、追问用户”，则 **LlamaIndex 或 LangGraph 的 Agent 路由模式更合适**。



---

**User** _(2026-08-24 10:29:17)_

mysql 可以存储树形结构吗

---

**Bot** _(2026-08-24 10:29:39)_

可以。MySQL 可以存储树形结构，常见实现有多种；选哪种取决于树的深度、读写频率，以及是否经常查询“某节点的全部子孙”或“完整路径”。

对于你前面说的 **RAG 意图树**，最常用、最推荐的通常是 **邻接表模型（parent_id）**，必要时再配合路径字段或闭包表优化查询。

---

## 1. 邻接表模型：最常用

每个节点保存父节点 ID。

```sql
CREATE TABLE intent_node (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    parent_id BIGINT UNSIGNED NULL,
    name VARCHAR(100) NOT NULL,
    code VARCHAR(100) NOT NULL,
    description TEXT NULL,
    route_type VARCHAR(30) NOT NULL DEFAULT 'rag',
    target_resource VARCHAR(255) NULL,
    sort_order INT NOT NULL DEFAULT 0,
    status TINYINT NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    UNIQUE KEY uk_intent_code (code),
    KEY idx_parent_sort (parent_id, sort_order),
    CONSTRAINT fk_intent_parent
        FOREIGN KEY (parent_id) REFERENCES intent_node(id)
        ON DELETE RESTRICT
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

示例数据：

| id | parent_id | name | code |
|---:|---:|---|---|
| 1 | `NULL` | 企业知识问答 | enterprise_qa |
| 2 | 1 | 财务与报销 | finance |
| 3 | 2 | 差旅报销 | travel_expense |
| 4 | 3 | 酒店住宿标准 | hotel_policy |
| 5 | 1 | IT 服务 | it |
| 6 | 5 | VPN 故障排查 | vpn_troubleshooting |

树形关系：

```text
企业知识问答
├── 财务与报销
│   └── 差旅报销
│       └── 酒店住宿标准
└── IT 服务
    └── VPN 故障排查
```

### 查询直接子节点

```sql
SELECT *
FROM intent_node
WHERE parent_id = 2
  AND status = 1
ORDER BY sort_order, id;
```

### MySQL 8+：递归查询全部子节点

查询“财务与报销”节点及其所有后代：

```sql
WITH RECURSIVE intent_tree AS (
    SELECT
        id,
        parent_id,
        name,
        code,
        0 AS depth,
        CAST(name AS CHAR(1000)) AS path
    FROM intent_node
    WHERE id = 2

    UNION ALL

    SELECT
        child.id,
        child.parent_id,
        child.name,
        child.code,
        parent.depth + 1,
        CONCAT(parent.path, ' / ', child.name)
    FROM intent_node AS child
    INNER JOIN intent_tree AS parent
        ON child.parent_id = parent.id
    WHERE child.status = 1
)
SELECT *
FROM intent_tree
ORDER BY path;
```

---

## 2. 物化路径模型：方便按路径查子树

在节点中额外保存完整路径，例如：

```text
/1/
/1/2/
/1/2/3/
/1/2/3/4/
```

表结构示例：

```sql
CREATE TABLE intent_node_path (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    parent_id BIGINT UNSIGNED NULL,
    name VARCHAR(100) NOT NULL,
    path VARCHAR(1000) NOT NULL,
    depth INT NOT NULL DEFAULT 0,
    PRIMARY KEY (id),
    KEY idx_path (path(255)),
    KEY idx_parent_id (parent_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

查询节点 `2` 的全部后代：

```sql
SELECT *
FROM intent_node_path
WHERE path LIKE '/1/2/%';
```

### 优缺点

- 优点：查询整棵子树简单、直观；
- 缺点：节点移动时，所有后代节点的路径都要更新；
- 注意：`LIKE '/1/2/%'` 在数据量大时索引利用和性能需实际验证。

这适合树不经常调整、但经常读取整个子树的情况。

---

## 3. 闭包表模型：适合复杂祖先/后代查询

闭包表单独保存所有“祖先—后代”关系。

```sql
CREATE TABLE intent_node (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    parent_id BIGINT UNSIGNED NULL,
    PRIMARY KEY (id),
    KEY idx_parent_id (parent_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE intent_node_closure (
    ancestor_id BIGINT UNSIGNED NOT NULL,
    descendant_id BIGINT UNSIGNED NOT NULL,
    depth INT NOT NULL,
    PRIMARY KEY (ancestor_id, descendant_id),
    KEY idx_descendant_depth (descendant_id, depth),
    CONSTRAINT fk_closure_ancestor
        FOREIGN KEY (ancestor_id) REFERENCES intent_node(id)
        ON DELETE CASCADE,
    CONSTRAINT fk_closure_descendant
        FOREIGN KEY (descendant_id) REFERENCES intent_node(id)
        ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

其中：

- `depth = 0`：节点自己；
- `depth = 1`：直接父子关系；
- `depth = 2+`：更深层的祖先/后代关系。

查询节点 `2` 的全部后代：

```sql
SELECT
    n.id,
    n.name,
    c.depth
FROM intent_node_closure AS c
JOIN intent_node AS n
    ON n.id = c.descendant_id
WHERE c.ancestor_id = 2
  AND c.depth > 0
ORDER BY c.depth, n.id;
```

查询某叶子节点的完整祖先路径：

```sql
SELECT
    n.id,
    n.name,
    c.depth
FROM intent_node_closure AS c
JOIN intent_node AS n
    ON n.id = c.ancestor_id
WHERE c.descendant_id = 4
ORDER BY c.depth DESC;
```

### 优缺点

- 优点：查祖先、子孙、路径非常高效，SQL 清晰；
- 缺点：插入、删除、移动节点时维护逻辑复杂；关系表会增加存储量。

适合树较大、查询“所有祖先/子孙”很频繁、权限继承复杂的系统。

---

## 4. Nested Set（左右值）模型：适合读多写少

每个节点存 `lft`、`rgt` 值。一个节点的全部后代落在其左右值区间中。

```text
企业知识问答：lft=1, rgt=12
财务与报销：  lft=2, rgt=7
差旅报销：    lft=3, rgt=6
酒店标准：    lft=4, rgt=5
```

查询子树很快：

```sql
SELECT child.*
FROM intent_node AS parent
JOIN intent_node AS child
  ON child.lft BETWEEN parent.lft AND parent.rgt
WHERE parent.id = 2
ORDER BY child.lft;
```

但插入、移动节点通常要批量更新很多 `lft/rgt` 数据，因此对经常变化的意图树不太友好。

---

## 5. RAG 意图树推荐哪一种？

### 大部分场景：`parent_id` 邻接表

建议先使用：

```text
parent_id + MySQL 8 的 WITH RECURSIVE
```

原因：

- 模型简单；
- 新增、删除、移动意图节点方便；
- 意图树通常深度不大；
- 能自然保存节点的路由配置；
- MySQL 8+ 支持递归 CTE；
- 对多数企业 RAG 的管理后台足够。

可以给节点增加这些字段：

```sql
ALTER TABLE intent_node
ADD COLUMN kb_id BIGINT UNSIGNED NULL COMMENT '关联知识库',
ADD COLUMN retriever_config JSON NULL COMMENT '检索策略，如 top_k、rerank、filter',
ADD COLUMN tool_name VARCHAR(100) NULL COMMENT 'Agent 可调用的工具名',
ADD COLUMN route_mode ENUM('rag', 'tool', 'agent', 'clarify') NOT NULL DEFAULT 'rag',
ADD COLUMN confidence_threshold DECIMAL(4,3) NOT NULL DEFAULT 0.800;
```

例如：

```json
{
  "top_k": 20,
  "rerank_top_n": 5,
  "search_mode": "hybrid",
  "metadata_filter": {
    "department": "finance",
    "topic": "travel"
  }
}
```

这使得意图树节点不仅是分类目录，也是**路由策略配置**。

---

## 6. 重要提醒：数据库树不等于 Agent 路由能力

MySQL 中保存的树一般是：

```text
“允许的领域、意图、知识库和策略”定义
```

而 Agent 或分类器负责：

```text
“当前用户问题应该命中树中的哪个节点”
```

推荐的职责划分：

```text
MySQL
  └── 保存意图树、节点配置、知识库绑定、权限与版本

语义路由器 / LLM 分类器 / Agent
  └── 根据用户问题选择节点

RAG 检索层
  └── 按节点配置执行 metadata filter、召回、重排

Langfuse
  └── 记录路由路径、召回文档、最终答案、耗时和评估分数
```

---

## 结论

MySQL 完全可以存储 RAG 的意图树。

- **简单且通用**：使用 `parent_id` 邻接表；
- **经常查整棵子树**：加入物化路径；
- **经常查上下级关系、权限继承或复杂树查询**：使用闭包表；
- **读极多、树几乎不变**：考虑 Nested Set。

对于大多数 RAG 意图路由项目，建议先从 **`parent_id + 递归 CTE + JSON 路由配置`** 开始。



---

**User** _(2026-08-24 10:40:56)_

邻接表在插入的时候要知道父节点吗，还要自己维护父节点？

---

**Bot** _(2026-08-24 10:41:11)_

是的，**邻接表模型插入子节点时，需要指定它的父节点 `parent_id`**。

但不需要在父节点记录中再额外维护一个“子节点列表”。父子关系只由**子节点这一行的 `parent_id`** 保存。

---

## 基本规则

```text
父节点：不保存 children 字段
子节点：保存 parent_id = 父节点.id
根节点：parent_id = NULL
```

例如：

| id | parent_id | name |
|---:|---:|---|
| 1 | `NULL` | 企业知识问答 |
| 2 | 1 | 财务与报销 |
| 3 | 2 | 差旅报销 |
| 4 | 3 | 酒店住宿标准 |

这里：

- `财务与报销` 的 `parent_id = 1`；
- `差旅报销` 的 `parent_id = 2`；
- `酒店住宿标准` 的 `parent_id = 3`；
- 父节点不需要存“我有哪些 children”。

---

## 插入示例

### 插入根节点

```sql
INSERT INTO intent_node (
    parent_id,
    name,
    code
) VALUES (
    NULL,
    '企业知识问答',
    'enterprise_qa'
);
```

假设插入后根节点的 `id = 1`。

### 在根节点下插入“财务与报销”

```sql
INSERT INTO intent_node (
    parent_id,
    name,
    code
) VALUES (
    1,
    '财务与报销',
    'finance'
);
```

### 在“财务与报销”下插入“差旅报销”

假设财务节点 ID 是 `2`：

```sql
INSERT INTO intent_node (
    parent_id,
    name,
    code
) VALUES (
    2,
    '差旅报销',
    'travel_expense'
);
```

---

## “要自己维护父节点”是什么意思？

分两种情况。

### 不需要维护的部分

不需要在父节点上手动更新这类数据：

```json
{
  "id": 2,
  "name": "财务与报销",
  "children": [3, 4, 5]
}
```

因为 MySQL 可以直接通过查询得到子节点：

```sql
SELECT *
FROM intent_node
WHERE parent_id = 2
ORDER BY sort_order, id;
```

也就是说，**子节点关系由查询反向推导出来**。

### 需要维护的部分

需要在新增、移动节点时正确写入子节点的 `parent_id`。

例如，把“酒店住宿标准”从“差旅报销”移动到“财务与报销”下：

```sql
UPDATE intent_node
SET parent_id = 2
WHERE id = 4;
```

只更新被移动节点的 `parent_id` 即可；它下面的子孙节点不用逐个更新。

---

## 后端接口一般怎么设计？

管理后台或接口不必让管理员直接输入数据库 ID；可以让用户在树形界面选择父节点，前端传 `parentId`。

```json
POST /api/intents
{
  "name": "酒店住宿标准",
  "code": "hotel_policy",
  "parentId": 3,
  "routeMode": "rag",
  "knowledgeBaseId": 12
}
```

后端做以下检查后写入数据库：

1. `parentId` 是否存在；
2. 当前用户是否有权限在该父节点下创建；
3. 父节点是否启用；
4. 同一父节点下是否允许重复名称或重复 `code`；
5. 层级是否超过最大深度；
6. 若是移动节点，目标父节点是否是当前节点自身或其子孙节点，避免形成循环。

---

## 建议加外键约束

这样可以避免插入一个不存在的父节点 ID：

```sql
ALTER TABLE intent_node
ADD CONSTRAINT fk_intent_parent
FOREIGN KEY (parent_id)
REFERENCES intent_node(id)
ON DELETE RESTRICT;
```

含义是：

- `parent_id` 必须引用一个真实存在的 `id`；
- 父节点仍有子节点时，不允许直接删除父节点；
- 应先移动或删除其子节点。

如果业务上希望删除父节点时自动删除全部子节点，可以使用 `ON DELETE CASCADE`，但对意图树和知识库路由配置而言通常风险较高，不建议默认使用。

---

## 一句话结论

**邻接表插入时要知道或选择父节点，并将它写入子节点的 `parent_id`；但无需在父节点中同步维护子节点列表。**



---

**User** _(2026-08-24 10:41:52)_

删除节点需要维护父节点的指向吗

---

**Bot** _(2026-08-24 10:42:05)_

通常**不需要维护父节点的指向**。

在邻接表模型中，关系是由子节点保存的：

```text
子节点.parent_id → 父节点.id
```

父节点本身没有保存“我有哪些子节点”的指针或列表。因此删除节点时，重点是处理**该节点的子节点**，而不是更新它的父节点。

---

## 1. 删除叶子节点：直接删除即可

假设删除“酒店住宿标准”：

```text
企业知识问答
└── 财务与报销
    └── 差旅报销
        └── 酒店住宿标准  ← 删除
```

如果它没有子节点，直接删除：

```sql
DELETE FROM intent_node
WHERE id = 4;
```

不需要更新“差旅报销”节点。

---

## 2. 删除有子节点的节点：必须先确定策略

假设删除“差旅报销”：

```text
企业知识问答
└── 财务与报销
    └── 差旅报销         ← 删除
        ├── 酒店住宿标准
        └── 交通报销标准
```

此时不能直接随意删除。常见有三种策略。

### 策略 A：禁止删除有子节点的节点（推荐默认）

使用外键：

```sql
FOREIGN KEY (parent_id)
REFERENCES intent_node(id)
ON DELETE RESTRICT
```

此时直接删除父节点会失败。你需要先：

- 删除子节点；
- 或将子节点移动到其他节点；
- 或将该节点标记为停用。

适合意图树、权限树、菜单树等业务配置，安全性最高。

---

### 策略 B：删除节点，同时删除整个子树

使用：

```sql
FOREIGN KEY (parent_id)
REFERENCES intent_node(id)
ON DELETE CASCADE
```

删除“差旅报销”后，其所有子孙节点会一起被删除。

```sql
DELETE FROM intent_node
WHERE id = 3;
```

风险是误操作可能删掉大量意图配置、知识库绑定和路由策略，因此生产中建议：

- 使用软删除；
- 先展示受影响节点数量；
- 要求二次确认；
- 记录审计日志；
- 或仅允许管理员执行。

---

### 策略 C：删除当前节点，将直接子节点上移到其父节点

删除“差旅报销”时，将其子节点改挂到“财务与报销”下：

```text
删除前：

财务与报销
└── 差旅报销
    ├── 酒店住宿标准
    └── 交通报销标准

删除后：

财务与报销
├── 酒店住宿标准
└── 交通报销标准
```

需要在**事务**内执行：

```sql
START TRANSACTION;

UPDATE intent_node
SET parent_id = (
    SELECT parent_id
    FROM (
        SELECT parent_id
        FROM intent_node
        WHERE id = 3
    ) AS target
)
WHERE parent_id = 3;

DELETE FROM intent_node
WHERE id = 3;

COMMIT;
```

这里虽然更新了子节点的 `parent_id`，但仍然不是维护“父节点指向”；而是在重新指定**子节点的新父节点**。

> 注意：MySQL 中不能在同一层查询里一边更新表、一边直接从同一个表子查询读取，因此示例中多包了一层派生表。

应用层也可以先查出待删节点的 `parent_id`，再用参数化 SQL 执行，逻辑通常更清晰。

---

## 3. 根节点不能用“上移子节点”策略

根节点的 `parent_id` 是 `NULL`。

如果删除根节点并将其子节点上移，子节点都会变成新的根节点：

```sql
UPDATE intent_node
SET parent_id = NULL
WHERE parent_id = 1;
```

这是否合理取决于业务设计。对意图树而言，通常建议保留一个固定的虚拟根节点，或禁止删除根节点。

---

## 4. 推荐：软删除而非物理删除

对 RAG 意图树，节点往往绑定：

- 知识库；
- 检索参数；
- 路由策略；
- 权限；
- 历史路由记录；
- Langfuse 中的调用与评估数据。

因此，通常建议增加软删除字段：

```sql
ALTER TABLE intent_node
ADD COLUMN status ENUM('active', 'disabled', 'deleted')
    NOT NULL DEFAULT 'active',
ADD COLUMN deleted_at DATETIME NULL,
ADD COLUMN deleted_by BIGINT UNSIGNED NULL;
```

执行“删除”时：

```sql
UPDATE intent_node
SET
    status = 'deleted',
    deleted_at = NOW()
WHERE id = 3;
```

实际检索和树展示时过滤：

```sql
WHERE status = 'active'
```

这样既能避免误删，也保留历史路由配置，便于审计与回滚。

---

## 结论

- **删除叶子节点**：不需要更新父节点。
- **删除非叶子节点**：不需要更新父节点，但必须处理直接子节点：
  - 禁止删除；
  - 级联删除整棵子树；
  - 或将子节点重新挂到被删节点的父节点。
- 对 RAG 意图树，默认推荐：  
  **`ON DELETE RESTRICT` + 软删除 + 管理员确认 + 审计日志**。



---

**User** _(2026-08-24 10:43:54)_

邻接表实际上还是存储的普通表，只不过多加了个字段parentid ，在查询的时候根据这个一起查询所有的树形结构，然后缓存到redis，是这样吗

---

**Bot** _(2026-08-24 10:44:11)_

对，你的理解基本正确。

**邻接表本质上就是一张普通关系型表**，只是在每一行增加一个 `parent_id` 字段，用它表达“当前节点属于哪个父节点”。

```text
普通表 + parent_id
= 邻接表树形结构
```

例如：

| id | parent_id | name |
|---:|---:|---|
| 1 | `NULL` | 企业知识问答 |
| 2 | 1 | 财务与报销 |
| 3 | 2 | 差旅报销 |
| 4 | 3 | 酒店住宿标准 |
| 5 | 1 | IT 服务 |

数据库中它依旧是平铺的数据：

```text
id=1, parent_id=NULL
id=2, parent_id=1
id=3, parent_id=2
...
```

只有在应用查询和组装时，才会呈现为树：

```text
企业知识问答
├── 财务与报销
│   └── 差旅报销
│       └── 酒店住宿标准
└── IT 服务
```

---

## 常见查询方式

### 方式一：一次查出所有节点，再在应用层组树

对于意图树、菜单树、组织架构树这类通常节点数量不算特别大的配置数据，最常用的是一次查询全部有效节点：

```sql
SELECT
    id,
    parent_id,
    name,
    code,
    route_mode,
    kb_id,
    retriever_config,
    sort_order
FROM intent_node
WHERE status = 'active'
ORDER BY sort_order, id;
```

然后在 Java、Python、Go、Node.js 等应用层中，根据 `id` 和 `parent_id` 组装树结构。

伪代码如下：

```python
nodes_by_id = {node["id"]: {**node, "children": []} for node in rows}
roots = []

for node in nodes_by_id.values():
    if node["parent_id"] is None:
        roots.append(node)
    else:
        parent = nodes_by_id.get(node["parent_id"])
        if parent:
            parent["children"].append(node)
        else:
            # 父节点不存在：记录数据异常或作为根节点处理
            roots.append(node)
```

时间复杂度通常是 **O(n)**，性能很好。

---

## 再缓存到 Redis：是一个常见方案

如果意图树读取频繁、修改不频繁，可以采用：

```text
MySQL
  └── 作为真实数据源（Source of Truth）

Redis
  └── 缓存已经组装好的意图树 / 路由配置

应用服务
  └── 优先从 Redis 读取；未命中时从 MySQL 加载、组树并写回 Redis
```

示意：

```text
请求进入
  ↓
GET rag:intent-tree:v1
  ↓
命中 Redis？
  ├── 是 → 直接得到树及节点路由配置
  └── 否 → 查询 MySQL 全部有效节点
              ↓
            应用层构建树
              ↓
            写入 Redis
              ↓
            返回树
```

对于 RAG 意图路由，Redis 中也可以不仅缓存树，还缓存适合运行时判断的信息，例如：

```json
{
  "version": 12,
  "updatedAt": "2026-02-15T10:00:00Z",
  "nodes": {
    "finance": {
      "parentCode": "enterprise_qa",
      "routeMode": "rag",
      "knowledgeBaseId": 20,
      "filters": {
        "department": "finance"
      },
      "topK": 20,
      "rerankTopN": 5
    },
    "it_vpn": {
      "parentCode": "it",
      "routeMode": "rag",
      "knowledgeBaseId": 31,
      "filters": {
        "topic": "vpn"
      },
      "topK": 10,
      "rerankTopN": 3
    }
  }
}
```

---

## 缓存更新比“缓存多久”更重要

意图树是配置数据，通常应优先做**主动失效或版本更新**，而不只是设置很长 TTL。

当新增、修改、移动、停用或删除节点时：

```text
数据库事务提交成功
  ↓
删除旧 Redis Key，或发布配置变更事件
  ↓
下一次请求自动从 MySQL 重建缓存
```

例如：

```text
rag:intent-tree:v12
```

更新配置后可以：

```text
DEL rag:intent-tree:v12
```

或者使用版本号：

```text
rag:intent-tree:current  →  13
rag:intent-tree:v13      →  新配置内容
```

这样可以避免多个服务实例长期使用旧配置。

---

## 是否每次都要查询“所有树”？

不一定，取决于用途。

| 场景 | 建议做法 |
|---|---|
| 管理后台展示树 | 查全部节点，在应用层组树 |
| Agent / 意图路由初始化 | 加载全部启用节点，缓存到 Redis 或本地内存 |
| 根据节点 ID 找其直接子节点 | `WHERE parent_id = ?` |
| 查某节点所有后代 | MySQL 8+ 用递归 CTE，或从缓存树遍历 |
| 路由到叶子节点后读取配置 | 直接按 `id` / `code` 从 Redis Hash 或本地 Map 获取 |
| 多租户、权限隔离 | 缓存 Key 加租户和版本，如 `rag:intent-tree:{tenantId}:{version}` |

对于运行时路由，往往不需要每个请求都把整个树从 Redis 读出来。可以把树拆为更适合读取的结构：

```text
rag:intent:node:{tenantId}:{nodeCode}
rag:intent:children:{tenantId}:{parentCode}
rag:intent:version:{tenantId}
```

或者服务启动时加载到进程内存，并通过 Redis Pub/Sub、消息队列或配置版本号通知各实例刷新。

---

## 一个重要补充：树结构和语义路由索引可以分开缓存

如果你用 Agent 或向量语义路由，运行时不一定是在树上逐层遍历：

```text
用户问题
  ↓
Embedding / LLM 分类
  ↓
直接得到 intent_code = "travel_expense"
  ↓
Redis / 本地 Map 根据 code 读取节点配置
  ↓
执行对应 RAG 检索
```

此时：

- MySQL 的邻接表用于**管理树、展示层级、配置继承和权限规则**；
- Redis 缓存用于**快速读取节点配置**；
- 向量库或 embedding 索引用于**把自然语言问题匹配到某个意图节点**。

---

## 结论

你的理解可以总结为：

```text
MySQL 普通表
  + parent_id 表示父子关系
  + 应用层或递归 SQL 查询并组装成树
  + Redis 缓存树和路由配置
```

这是非常常见且合理的实现方式。

但建议：

- **MySQL 是唯一真实数据源**；
- Redis 只做缓存，不做唯一存储；
- 配置变更后主动删除/刷新 Redis 缓存；
- 运行时最好按节点 `code/id` 快速读取配置，而不是每次遍历整棵树。



---

**User** _(2026-08-24 11:30:38)_

对日软件开发中   ，内部设计或者详细设计  转 测试式样书

---

**Bot** _(2026-08-24 11:31:08)_

在对日软件开发中，**内部设计书（内部設計書）/详细设计书（詳細設計書）转换为测试式样书（テスト仕様書）**，本质上是把“系统应如何实现”的设计描述，转换为“需要验证什么、如何操作、预期结果是什么”的可执行测试用例。

按照 V 模型的常见对应关系：

```text
详细设计／内部设计
        ↓
单体测试式样书（単体テスト仕様書）
        ↓
单体测试（単体テスト）
```

> 不同日本客户对“内部设计”和“详细设计”的定义可能略有差异，但它们通常对应**单体测试**。  
> 基本设计／外部设计通常更多对应结合测试、系统测试或验收测试。

---

## 1. 文档对应关系

| 设计书内容 | 转换后的测试式样书内容 |
|---|---|
| 功能处理流程 | 正常流程、异常流程的测试用例 |
| 输入项目定义 | 输入条件、边界值、必填校验、格式校验 |
| 输出项目定义 | 输出结果、画面显示、返回值、消息验证 |
| 业务规则/判断条件 | 条件覆盖、分支覆盖、规则组合测试 |
| 数据库表定义 | 新增、更新、删除、查询及数据一致性验证 |
| 接口定义 | 请求参数、响应结果、错误码、超时和异常处理 |
| 错误处理设计 | 各类错误条件及预期错误消息 |
| 权限设计 | 不同角色的可见范围、可操作范围 |
| 批处理设计 | 输入文件、处理结果、日志、错误文件、重跑验证 |

---

## 2. 测试式样书一般包含什么？

日式测试式样书常见字段如下：

| 字段 | 日文 | 含义 |
|---|---|---|
| 测试项目编号 | テスト項目ID | 测试用例唯一编号 |
| 测试对象 | テスト対象 | 画面、API、类、方法、批处理等 |
| 测试观点评价点 | テスト観点 | 要验证的角度，例如必填、权限、边界值 |
| 前提条件 | 事前条件 | 测试前需准备的账号、数据、状态 |
| 输入值/操作 | 入力値・操作手順 | 具体输入数据和操作步骤 |
| 预期结果 | 期待結果 | 页面、DB、接口返回、日志等预期 |
| 确认方法 | 確認方法 | 如何确认，例如画面、SQL、日志、API 返回 |
| 实际结果 | 実績結果 | 执行后填写 |
| 判定 | 判定 | OK / NG |
| 备注 | 備考 | 关联设计书章节、缺陷编号等 |

---

## 3. 从设计到测试的转换步骤

### 第一步：拆分设计书中的“可验证项”

不要直接按设计书章节机械复制，而是识别每个可验证的行为。

例如详细设计中写：

```text
用户登录时：
1. 输入用户 ID 和密码；
2. 用户 ID 或密码为空时，显示错误消息；
3. 用户不存在或密码不正确时，显示认证失败消息；
4. 连续认证失败 5 次时，锁定账号；
5. 认证成功后跳转到首页。
```

应拆成至少以下测试观点评价点：

```text
- 用户 ID 必填
- 密码必填
- 用户 ID 不存在
- 密码错误
- 第 1～4 次密码错误
- 第 5 次密码错误后账号锁定
- 锁定账号再次登录
- 正常登录
- 登录成功后的跳转及 Session 创建
```

---

### 第二步：按测试观点评价点设计测试数据

常用设计方法：

| 方法 | 日文 | 适用场景 |
|---|---|---|
| 等价类划分 | 同値分割 | 合法/非法输入分类 |
| 边界值分析 | 境界値分析 | 长度、金额、日期、数量、次数 |
| 条件组合 | デシジョンテーブル | 多个条件共同决定结果 |
| 状态迁移 | 状態遷移テスト | 锁定、审批、订单、登录状态 |
| 分支覆盖 | 分岐網羅 | 单体测试中的 `if/else`、`switch` |
| 异常测试 | 異常系テスト | DB 错误、接口超时、数据缺失 |
| 权限测试 | 権限テスト | 管理员、普通用户、无权限用户 |

---

### 第三步：明确每个用例的预期结果

日式测试中，预期结果不能只写：

```text
正常显示。
```

应明确写出可确认的结果，例如：

```text
1. 画面上显示错误消息「ユーザーIDを入力してください。」。
2. 登录处理不执行。
3. 不创建 Session。
4. 数据库的 login_history 表中不新增记录。
```

如果涉及数据库或接口，建议同时写清：

- 页面显示内容；
- 接口状态码、返回字段；
- 数据库记录变化；
- 日志内容；
- 文件输出内容；
- 后续状态变化。

---

## 4. 示例：详细设计转测试式样书

### 详细设计内容

```text
功能：用户注册

输入项目：
- 用户名：必填，1～20 位
- 邮箱：必填，符合邮箱格式，系统内不可重复
- 密码：必填，8～32 位，必须包含英文字母和数字

处理：
1. 校验输入内容；
2. 校验邮箱是否重复；
3. 密码加密后保存到 users 表；
4. 注册成功后显示“注册完成”。
```

### 转换后的测试式样书示例

| 测试ID | 测试观点评价点 | 前提条件 | 输入/操作 | 预期结果 |
|---|---|---|---|---|
| UT-REG-001 | 正常注册 | 邮箱未注册 | 输入合法用户名、邮箱、密码后点击注册 | 显示“登録が完了しました。”；`users` 表新增 1 条记录；密码以加密值保存 |
| UT-REG-002 | 用户名必填 | 无 | 用户名为空，其他输入合法，点击注册 | 显示“ユーザー名を入力してください。”；不新增数据 |
| UT-REG-003 | 用户名最小边界 | 无 | 用户名输入 1 位字符 | 校验通过 |
| UT-REG-004 | 用户名最大边界 | 无 | 用户名输入 20 位字符 | 校验通过 |
| UT-REG-005 | 用户名超长 | 无 | 用户名输入 21 位字符 | 显示长度错误消息；不新增数据 |
| UT-REG-006 | 邮箱格式错误 | 无 | 输入 `abc@` | 显示邮箱格式错误消息；不新增数据 |
| UT-REG-007 | 邮箱重复 | 已存在 `test@example.com` 用户 | 输入已注册邮箱 | 显示“このメールアドレスは既に登録されています。”；不新增数据 |
| UT-REG-008 | 密码最小边界 | 无 | 输入 8 位且含字母和数字的密码 | 校验通过 |
| UT-REG-009 | 密码规则违反 | 无 | 输入只有数字的 8 位密码 | 显示密码规则错误消息；不新增数据 |
| UT-REG-010 | DB 保存异常 | 模拟 DB 写入失败 | 输入合法数据后点击注册 | 显示系统错误消息；事务回滚；不产生部分数据；输出错误日志 |

---

## 5. 单体测试和结合测试的区别

从详细设计生成测试式样书时，要注意测试层次，避免把其他层的内容混进来。

| 测试层次 | 主要依据文档 | 主要验证内容 |
|---|---|---|
| 单体测试（単体テスト） | 详细设计、内部设计、源代码 | 方法、类、模块、分支、输入校验、异常处理 |
| 结合测试（結合テスト） | 基本设计、接口设计、外部设计 | 模块/系统间接口、数据传递、事务、调用顺序 |
| 系统测试（システムテスト） | 需求定义、业务流程、非功能需求 | 端到端业务流程、性能、安全、可用性 |
| 验收测试（受入テスト） | 用户需求、验收标准 | 是否满足客户业务需求 |

例如：

```text
“密码为空时返回校验错误”
→ 详细设计/单体测试范围

“注册成功后自动调用邮件服务发送欢迎邮件”
→ 结合测试范围

“高峰期每秒 1,000 个注册请求系统仍可用”
→ 性能测试/系统测试范围
```

---

## 6. 推荐的转换方法：建立可追溯矩阵

建议维护一份“设计—测试对应表（トレーサビリティマトリクス）”。

| 设计书章节/ID | 设计功能 | 测试ID | 覆盖类型 | 状态 |
|---|---|---|---|---|
| DD-REG-01 | 用户名必填校验 | UT-REG-002 | 异常系 | 已覆盖 |
| DD-REG-02 | 用户名长度 1～20 位 | UT-REG-003～005 | 境界値 | 已覆盖 |
| DD-REG-03 | 邮箱重复校验 | UT-REG-007 | 异常系 | 已覆盖 |
| DD-REG-04 | 密码加密保存 | UT-REG-001 | 正常系/DB确认 | 已覆盖 |
| DD-REG-05 | DB 异常时回滚 | UT-REG-010 | 異常系 | 已覆盖 |

这样可以避免：

- 设计书某个分支没有测试；
- 修改设计后遗漏更新测试；
- 测试用例没有对应设计依据；
- 客户评审时无法说明测试覆盖范围。

---

## 7. 对日项目中特别要注意的点

1. **正常系和异常系必须分开且完整覆盖**  
   日本客户通常非常重视输入异常、系统异常、边界值和错误消息。

2. **测试前提数据要写具体**  
   不要只写“准备测试数据”，应明确账号、数据库记录、权限、日期、文件内容等。

3. **预期结果要可验证**  
   明确页面、接口、DB、日志、文件、状态变化；避免模糊表述。

4. **错误消息需与设计书完全一致**  
   包括日文文案、标点、项目名、消息 ID、显示位置和显示条件。

5. **覆盖设计书中的所有条件分支**  
   特别是 `if/else`、状态判断、权限判断、日期判断、空值判断和例外处理。

6. **保持编号与版本管理**  
   测试用例应关联设计书版本、程序版本及需求/缺陷编号。

---

## 总结

从内部设计或详细设计转测试式样书，可以概括为：

```text
设计中的输入、处理、输出、条件、异常、DB/接口规则
                    ↓
拆分为测试观点评价点
                    ↓
设计正常值、异常值、边界值、状态和权限测试数据
                    ↓
编写操作步骤与明确、可验证的预期结果
                    ↓
建立“设计项 ↔ 测试用例”的可追溯关系
```

对于对日项目，详细设计通常对应的交付物是：

```text
詳細設計書／内部設計書
        ↓
単体テスト仕様書
        ↓
単体テスト結果報告書
```



---

**User** _(2026-08-24 11:33:28)_

利用langchain deepagent + llm，自动转换，生成excel表格 

---

**Bot** _(2026-08-24 11:34:21)_

可以。比较稳妥的方案是：

> **Deep Agent 负责读取/拆分详细设计、提取规则和检查覆盖；LLM 负责生成结构化测试用例；Python 使用 `xlsxwriter` 把结构化结果确定性地写成 Excel 测试式样书。**

不要让 LLM 直接生成 Excel 二进制文件，也不要只让它输出自由文本表格。核心是先约束为 JSON/Pydantic 结构，再生成 `.xlsx`。

---

## 推荐整体流程

```text
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
```

---

## 适合生成的日式测试式样书列

建议至少生成以下字段：

| 列名 | 日文名称 | 说明 |
|---|---|---|
| 测试 ID | テスト項目ID | 如 `UT-LOGIN-001` |
| 设计书 ID | 設計書ID | 对应详细设计章节、功能 ID |
| 测试对象 | テスト対象 | 画面、API、批处理、方法等 |
| 测试分类 | テスト区分 | 正常系、異常系、境界値、権限、状態遷移等 |
| 测试观点评价点 | テスト観点 | 要验证的规则 |
| 前提条件 | 事前条件 | 账号、数据库初始状态、外部依赖等 |
| 测试数据 | テストデータ | 输入值或准备数据 |
| 操作步骤 | 操作手順 | 可执行的操作步骤 |
| 预期结果 | 期待結果 | 画面、响应、DB、日志等 |
| 确认方法 | 確認方法 | UI、SQL、API、日志、文件等 |
| 优先级 | 優先度 | High / Medium / Low |
| 实际结果 | 実績結果 | 测试执行时填写 |
| 判定 | 判定 | `未実施` / `OK` / `NG` |
| 备注 | 備考 | 缺陷号、补充说明等 |

---

## 关键原则

### 1. 先生成“测试观点评价点”，再生成测试用例

不要让模型看到设计书后直接吐出大量 Case。应分两步：

```text
详细设计
  ↓
测试观点评价点
  ↓
测试用例
```

例如设计规则：

```text
用户名为必填，长度为 1～20 字符；
邮箱必须符合格式且不可重复；
密码长度为 8～32 位，且必须包含字母和数字。
```

先抽取测试观点评价点：

```text
- 用户名必填
- 用户名长度下限：1
- 用户名长度上限：20
- 用户名超长：21
- 邮箱格式合法性
- 邮箱重复性
- 密码最小长度：8
- 密码最大长度：32
- 密码仅数字
- 密码仅字母
- 密码含字母和数字
```

然后才生成详细的测试步骤和预期结果。

这样可以显著降低遗漏边界值、异常流程和 DB 验证项的概率。

---

### 2. LLM 必须输出结构化数据

推荐使用 Pydantic Schema 约束模型输出，而不是解析 Markdown 表格。

```python
from typing import Literal
from pydantic import BaseModel, Field


class TestCase(BaseModel):
    test_id: str = Field(description="唯一测试用例编号，例如 UT-LOGIN-001")
    design_ids: list[str] = Field(description="关联的详细设计章节或设计项ID")
    target: str = Field(description="测试对象，例如 登录画面、POST /api/login")
    category: Literal[
        "正常系",
        "異常系",
        "境界値",
        "条件組合せ",
        "権限",
        "状態遷移",
        "DB確認",
        "外部連携",
    ]
    viewpoint: str = Field(description="测试观点评价点")
    preconditions: list[str] = Field(description="测试前置条件")
    test_data: list[str] = Field(description="测试输入数据或准备数据")
    steps: list[str] = Field(description="按顺序执行的测试步骤")
    expected_results: list[str] = Field(
        description="可验证的预期结果，需包含画面、API、DB、日志等必要确认项"
    )
    verification_method: list[str] = Field(description="确认方法，例如画面、SQL、日志")
    priority: Literal["High", "Medium", "Low"] = "Medium"
    notes: str = ""


class OpenIssue(BaseModel):
    design_ids: list[str]
    description: str
    reason: str
    recommendation: str


class TestSpecification(BaseModel):
    system_name: str
    function_name: str
    source_document_version: str
    test_cases: list[TestCase]
    open_issues: list[OpenIssue]
```

---

## Deep Agent 的职责设计

Deep Agent 更适合处理长文档、多章节和多功能模块，而不是只调用一次 LLM。

推荐将其拆为以下子任务：

```text
设计书分析 Agent
├── 功能拆分 Agent
│   └── 按画面、API、批处理、模块拆分
├── 规则提取 Agent
│   └── 提取输入、校验、处理、输出、DB、异常、权限
├── 测试设计 Agent
│   └── 生成正常、异常、边界、条件组合、状态迁移测试点
├── 覆盖率审查 Agent
│   └── 检查是否遗漏设计项、异常处理、错误消息和数据更新
└── 汇总 Agent
    └── 合并、去重、编号并输出结构化测试式样书
```

对于较大的详细设计书，不建议把全文一次性塞入模型上下文。应按功能或设计章节切分，例如：

```text
詳細設計書
├── 1. 登录功能
├── 2. 用户注册功能
├── 3. 密码重置功能
├── 4. 用户信息维护功能
└── 5. 批量用户导入功能
```

每个功能单独生成测试用例，最后合并并全局去重。

---

## 推荐 Prompt

系统 Prompt 应明确限制模型“不得臆造”。

```text
你是一名熟悉日本软件开发流程的高级测试设计工程师。

任务：
根据输入的内部设计书或详细设计书，生成单体测试式样书数据。

必须遵守：
1. 仅根据设计书中明确描述的内容生成测试用例，不得虚构字段、接口、错误消息或业务规则。
2. 每个测试用例必须关联至少一个设计书 ID 或章节号。
3. 覆盖正常系、异常系、边界值、条件组合、状态迁移、权限、数据库更新和外部接口异常；但仅在设计书存在对应内容时生成。
4. 对输入长度、数量、金额、日期、次数等明确边界，生成下限、上限、下限-1、上限+1 测试。
5. 预期结果必须可验证。不能只写“正常显示”或“处理成功”。
6. 涉及数据库更新时，必须说明新增、更新、删除、回滚或不更新的预期。
7. 设计书存在歧义、缺少预期结果或无法判定测试结果时，不得猜测；写入 open_issues。
8. 输出必须符合指定的结构化 Schema。
9. 测试步骤使用中文描述；日文错误消息、字段名、画面名应保留原文。
```

---

## Python 实现骨架

下面示例体现了三个职责：

1. 使用 Deep Agent 分析详细设计；
2. 用结构化输出生成测试式样数据；
3. 用 `xlsxwriter` 创建 Excel。

> `Deep Agent` 的具体初始化参数会随所选模型供应商及 `deepagents` 版本略有不同；但“Agent 分析 + 结构化 LLM 输出 + 确定性生成 Excel”的分层方式不变。

```python
import json
import os
from typing import Literal

import xlsxwriter
from pydantic import BaseModel, Field
from langchain_openai import ChatOpenAI

# 如使用 LangChain Deep Agents：
# from deepagents import create_deep_agent


class TestCase(BaseModel):
    test_id: str
    design_ids: list[str]
    target: str
    category: Literal[
        "正常系",
        "異常系",
        "境界値",
        "条件組合せ",
        "権限",
        "状態遷移",
        "DB確認",
        "外部連携",
    ]
    viewpoint: str
    preconditions: list[str]
    test_data: list[str]
    steps: list[str]
    expected_results: list[str]
    verification_method: list[str]
    priority: Literal["High", "Medium", "Low"] = "Medium"
    notes: str = ""


class OpenIssue(BaseModel):
    design_ids: list[str]
    description: str
    reason: str
    recommendation: str


class TestSpecification(BaseModel):
    system_name: str
    function_name: str
    source_document_version: str
    test_cases: list[TestCase]
    open_issues: list[OpenIssue]


SYSTEM_PROMPT = """
你是一名熟悉日本软件开发流程的高级测试设计工程师。

请基于内部设计书或详细设计书，生成“单体测试式样书”数据。

规则：
- 仅使用设计书明确提供的信息，不得虚构业务规则。
- 所有测试用例必须关联设计书章节号或设计项 ID。
- 对明确存在的字段规则，覆盖正常系、异常系、边界值和条件组合。
- 对明确存在的状态、权限、DB 更新、外部接口和异常处理，生成对应测试。
- 预期结果必须可验证，说明画面、响应、DB、日志或文件结果。
- 设计不明确时，写入 open_issues，不要猜测。
- 输出必须严格符合指定 Schema。
"""


def load_design_text(file_path: str) -> str:
    """
    示例：此函数只读取 UTF-8 文本或 Markdown。
    实际项目中可扩展为：
    - DOCX：python-docx
    - XLSX：openpyxl（只读）
    - PDF：PyMuPDF / pdfplumber
    - 扫描 PDF：OCR
    """
    with open(file_path, "r", encoding="utf-8") as f:
        return f.read()


def analyze_design_with_agent(design_text: str) -> str:
    """
    这里是 Deep Agent 分析步骤的占位实现。

    生产实现可让 Deep Agent：
    1. 按功能章节拆分；
    2. 识别输入、处理、输出、异常、DB 和接口；
    3. 输出中间结构化分析结果；
    4. 对长文档按章节并行处理后汇总。
    """

    # 示例伪代码：
    #
    # agent = create_deep_agent(
    #     model="openai:gpt-4.1",
    #     tools=[],
    #     system_prompt="""
    #     请分析详细设计书，按功能提取：
    #     设计ID、输入项、校验规则、业务规则、输出、
    #     DB操作、接口调用、错误处理、权限和状态变化。
    #     不生成最终测试用例，只输出结构化分析结论。
    #     """,
    # )
    #
    # result = agent.invoke(
    #     {"messages": [{"role": "user", "content": design_text}]}
    # )
    # return result["messages"][-1].content

    return design_text


def generate_test_spec(design_analysis: str) -> TestSpecification:
    llm = ChatOpenAI(
        model="gpt-4.1",
        temperature=0,
    )

    structured_llm = llm.with_structured_output(TestSpecification)

    prompt = f"""
{SYSTEM_PROMPT}

以下是详细设计书或经 Agent 整理后的设计分析内容：

--- 设计内容开始 ---
{design_analysis}
--- 设计内容结束 ---

请生成单体测试式样书。
"""

    return structured_llm.invoke(prompt)


def join_lines(values: list[str]) -> str:
    return "\n".join(f"{index + 1}. {value}" for index, value in enumerate(values))


def write_excel(spec: TestSpecification, output_path: str) -> None:
    workbook = xlsxwriter.Workbook(output_path)

    title_format = workbook.add_format({
        "bold": True,
        "font_size": 14,
        "align": "center",
        "valign": "vcenter",
        "bg_color": "#1F4E78",
        "font_color": "#FFFFFF",
    })

    header_format = workbook.add_format({
        "bold": True,
        "align": "center",
        "valign": "vcenter",
        "text_wrap": True,
        "bg_color": "#D9EAF7",
        "border": 1,
    })

    cell_format = workbook.add_format({
        "valign": "top",
        "text_wrap": True,
        "border": 1,
    })

    center_cell_format = workbook.add_format({
        "align": "center",
        "valign": "top",
        "text_wrap": True,
        "border": 1,
    })

    pending_format = workbook.add_format({
        "align": "center",
        "valign": "vcenter",
        "text_wrap": True,
        "border": 1,
        "bg_color": "#FFF2CC",
    })

    # Sheet 1：测试式样书
    ws = workbook.add_worksheet("単体テスト仕様書")
    ws.freeze_panes(3, 0)
    ws.merge_range("A1:N1", f"単体テスト仕様書 - {spec.function_name}", title_format)

    ws.write("A2", "系统名称", header_format)
    ws.write("B2", spec.system_name, cell_format)
    ws.write("D2", "设计书版本", header_format)
    ws.write("E2", spec.source_document_version, cell_format)

    headers = [
        "テスト項目ID",
        "設計書ID",
        "テスト対象",
        "テスト区分",
        "テスト観点",
        "事前条件",
        "テストデータ",
        "操作手順",
        "期待結果",
        "確認方法",
        "優先度",
        "実績結果",
        "判定",
        "備考",
    ]

    for col, header in enumerate(headers):
        ws.write(2, col, header, header_format)

    widths = [16, 18, 22, 12, 30, 30, 30, 38, 42, 25, 10, 30, 10, 24]
    for col, width in enumerate(widths):
        ws.set_column(col, col, width)

    for row, case in enumerate(spec.test_cases, start=3):
        ws.write(row, 0, case.test_id, center_cell_format)
        ws.write(row, 1, "\n".join(case.design_ids), cell_format)
        ws.write(row, 2, case.target, cell_format)
        ws.write(row, 3, case.category, center_cell_format)
        ws.write(row, 4, case.viewpoint, cell_format)
        ws.write(row, 5, join_lines(case.preconditions), cell_format)
        ws.write(row, 6, join_lines(case.test_data), cell_format)
        ws.write(row, 7, join_lines(case.steps), cell_format)
        ws.write(row, 8, join_lines(case.expected_results), cell_format)
        ws.write(row, 9, join_lines(case.verification_method), cell_format)
        ws.write(row, 10, case.priority, center_cell_format)
        ws.write(row, 11, "", cell_format)
        ws.write(row, 12, "未実施", pending_format)
        ws.write(row, 13, case.notes, cell_format)
        ws.set_row(row, 110)

    last_row = max(3, len(spec.test_cases) + 2)

    ws.data_validation(
        3,
        12,
        last_row,
        12,
        {
            "validate": "list",
            "source": ["未実施", "OK", "NG", "保留"],
        },
    )

    ws.autofilter(2, 0, last_row, len(headers) - 1)

    # Sheet 2：设计—测试追溯矩阵
    trace_ws = workbook.add_worksheet("トレーサビリティ")
    trace_ws.freeze_panes(1, 0)

    trace_headers = [
        "設計書ID",
        "テスト項目ID",
        "テスト対象",
        "テスト観点",
        "テスト区分",
        "優先度",
    ]

    for col, header in enumerate(trace_headers):
        trace_ws.write(0, col, header, header_format)

    trace_ws.set_column("A:A", 20)
    trace_ws.set_column("B:B", 18)
    trace_ws.set_column("C:C", 24)
    trace_ws.set_column("D:D", 40)
    trace_ws.set_column("E:E", 15)
    trace_ws.set_column("F:F", 12)

    trace_row = 1
    for case in spec.test_cases:
        for design_id in case.design_ids:
            trace_ws.write(trace_row, 0, design_id, cell_format)
            trace_ws.write(trace_row, 1, case.test_id, cell_format)
            trace_ws.write(trace_row, 2, case.target, cell_format)
            trace_ws.write(trace_row, 3, case.viewpoint, cell_format)
            trace_ws.write(trace_row, 4, case.category, center_cell_format)
            trace_ws.write(trace_row, 5, case.priority, center_cell_format)
            trace_row += 1

    trace_ws.autofilter(0, 0, max(1, trace_row - 1), len(trace_headers) - 1)

    # Sheet 3：未决事项
    issue_ws = workbook.add_worksheet("未決事項")
    issue_ws.freeze_panes(1, 0)

    issue_headers = [
        "No.",
        "関連設計書ID",
        "未決事項",
        "理由",
        "推奨対応",
        "対応状況",
    ]

    for col, header in enumerate(issue_headers):
        issue_ws.write(0, col, header, header_format)

    issue_ws.set_column("A:A", 8)
    issue_ws.set_column("B:B", 22)
    issue_ws.set_column("C:C", 45)
    issue_ws.set_column("D:D", 35)
    issue_ws.set_column("E:E", 45)
    issue_ws.set_column("F:F", 14)

    for row, issue in enumerate(spec.open_issues, start=1):
        issue_ws.write(row, 0, row, center_cell_format)
        issue_ws.write(row, 1, "\n".join(issue.design_ids), cell_format)
        issue_ws.write(row, 2, issue.description, cell_format)
        issue_ws.write(row, 3, issue.reason, cell_format)
        issue_ws.write(row, 4, issue.recommendation, cell_format)
        issue_ws.write(row, 5, "未対応", pending_format)
        issue_ws.set_row(row, 70)

    issue_ws.data_validation(
        1,
        5,
        max(1, len(spec.open_issues)),
        5,
        {
            "validate": "list",
            "source": ["未対応", "確認中", "対応済み", "対象外"],
        },
    )

    workbook.close()


def main():
    input_file = "detailed_design.md"
    output_file = "単体テスト仕様書.xlsx"

    design_text = load_design_text(input_file)

    # 长文档时，这一步应改为：分章节调用 Deep Agent，最后汇总。
    design_analysis = analyze_design_with_agent(design_text)

    test_spec = generate_test_spec(design_analysis)

    write_excel(test_spec, output_file)

    print(f"已生成：{os.path.abspath(output_file)}")


if __name__ == "__main__":
    main()
```

---

## 建议增加的质量控制

自动生成后，不建议直接作为最终交付物，而应增加以下检查。

### 设计覆盖检查

检查每个设计 ID 是否至少对应一个测试用例：

```text
详细设计项总数：120
已覆盖设计项：116
未覆盖设计项：4
```

### 测试分类检查

例如要求每个有输入校验的功能至少包含：

```text
正常系
+ 必填异常
+ 格式异常
+ 边界值
+ 数据重复或冲突
```

### 幻觉检查

针对每个 Case 反向让 LLM 或规则校验：

```text
该测试用例中的字段、错误消息、接口、DB 表、规则，
是否都能在设计书对应章节中找到依据？
```

无法找到依据的用例应标记为：

```text
要确认（要設計確認）
```

而不是直接写进正式测试式样书。

### 人工 Review

最终仍建议由测试负责人或设计担当确认：

- 是否遗漏业务规则；
- 日文术语、画面名称、错误消息是否准确；
- 设计书本身的歧义是否已作为“未决事项”列出；
- 测试层级是否正确，避免把结合测试或系统测试内容误放到单体测试中。

---

## 与 Langfuse 的结合

如果项目已经使用 Langfuse，可以记录每次转换任务：

```text
Trace
├── 输入：设计书文件、章节、版本号
├── Agent：功能拆分结果
├── LLM：测试观点评价点
├── LLM：结构化测试用例 JSON
├── 输出：Excel 文件地址、用例数量
├── Score：设计覆盖率
├── Score：人工 Review 通过率
└── Score：虚构项数量 / 未决事项数量
```

这样可以比较不同 Prompt、模型、设计书模板下的：

- 测试用例生成数量；
- 设计项覆盖率；
- 评审后的修改率；
- 未决事项识别率；
- 单份设计书的生成时间和模型成本。

---

## 推荐的最小可行版本

第一阶段不要直接处理所有 Word、PDF、Excel 格式，也不要一开始上多 Agent。可以先做：

```text
详细设计书 Markdown / 已抽取文本
  ↓
单次 LLM 结构化输出
  ↓
xlsxwriter 输出标准 Excel
  ↓
人工 Review
```

验证测试用例质量后，再逐步增加：

```text
文档解析
→ Deep Agent 章节拆分
→ 多 Agent 覆盖审查
→ Langfuse 可观测性
→ 自动生成追溯矩阵
→ CI/CD 中的回归检查
```

最重要的产物不是 Excel 本身，而是中间的、可验证的 **结构化测试用例 JSON**。Excel 只是面向对日项目交付和人工评审的最终展示格式。



---

**User** _(2026-08-24 12:03:23)_

现在有以往的内部设计和对应的测试式样书，如何根据这个东西，做一些dataset用于评测agent的效果，做到可量化，可观测，可演进

---

**Bot** _(2026-08-24 12:04:35)_

可以把历史的“**内部设计书 → 测试式样书**”沉淀为一个面向 Agent 的评测集，并把评测拆成多个可量化任务，而不是只比较“生成的 Excel 和历史 Excel 像不像”。

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
```

---

# 1. 不要把“整份 Excel”直接作为唯一黄金答案

历史测试式样书很有价值，但直接拿新旧 Excel 做文本 diff，评测意义有限。

原因是同一个设计，可能存在多个同样正确的测试用例写法：

```text
历史用例：
用户名为空时，显示“ユーザー名を入力してください”。

Agent 生成：
用户名未输入时，显示必填错误消息“ユーザー名を入力してください”。
```

两者文字不完全一致，但测试意图和预期结果是一致的。

因此，应将历史文档转换为**结构化的、分层的黄金数据**：

```text
设计书
  ↓
设计需求 / 规则（Requirement / Rule）
  ↓
测试观点评价点（Test Viewpoint）
  ↓
测试用例（Test Case）
  ↓
步骤、数据、预期结果、追溯关系
```

评测时主要比较：

- 规则是否被识别；
- 规则是否被覆盖；
- 测试用例是否正确；
- 预期结果是否可验证；
- 是否虚构了设计书没有的内容；
- 是否存在重复、冲突或缺失；
- 是否生成了符合模板的 Excel。

---

# 2. 建议建立的 Dataset 数据模型

建议不要只保存原始文件，而是将每个“功能单元”作为一条 Dataset Item。

一个功能单元可以是：

- 一个画面；
- 一个 API；
- 一个批处理；
- 一个类/模块；
- 一个业务功能，例如“用户注册”“订单取消”“差旅报销申请”。

## Dataset Item 示例

```json
{
  "dataset_item_id": "DD-LOGIN-001",
  "project_id": "project_a",
  "system_name": "员工门户系统",
  "module": "认证模块",
  "function_id": "LOGIN",
  "function_name": "用户登录",
  "test_level": "unit",
  "source_design": {
    "document_id": "DD-AUTH-V1.2",
    "version": "1.2",
    "section_ids": ["3.1", "3.1.1", "3.1.2"],
    "content": "……详细设计文本……"
  },
  "ground_truth": {
    "requirements": [],
    "test_viewpoints": [],
    "test_cases": [],
    "open_issues": []
  },
  "metadata": {
    "language": "ja",
    "domain": "internal_system",
    "has_db_update": true,
    "has_api_call": false,
    "has_validation": true,
    "has_state_transition": true,
    "difficulty": "medium",
    "review_status": "approved"
  }
}
```

---

## 黄金测试用例建议采用的结构

```json
{
  "test_id": "UT-LOGIN-003",
  "design_ids": ["DD-LOGIN-01"],
  "target": "ログイン画面",
  "category": "異常系",
  "viewpoint": "ユーザーID必須チェック",
  "preconditions": [
    "ログイン画面を表示していること"
  ],
  "test_data": [
    "ユーザーID：空白",
    "パスワード：ValidPass123"
  ],
  "steps": [
    "ユーザーIDを空白のまま入力する",
    "パスワードに「ValidPass123」を入力する",
    "ログインボタンを押下する"
  ],
  "expected_results": [
    "「ユーザーIDを入力してください。」が表示される",
    "認証処理が実行されない",
    "セッションが作成されない",
    "login_historyテーブルにレコードが追加されない"
  ],
  "verification_method": [
    "画面メッセージを確認する",
    "ログを確認する",
    "DBをSQLで確認する"
  ],
  "priority": "High"
}
```

---

# 3. 数据集不要只做一个，而应按任务拆分

建议至少建设四类 Dataset。

| Dataset | 输入 | 输出/黄金标准 | 用途 |
|---|---|---|---|
| 设计规则提取集 | 设计书章节 | 输入项、业务规则、异常、DB 操作、接口定义 | 评估 Agent 是否读懂设计 |
| 测试观点评价集 | 设计书/规则 | 测试观点评价点 | 评估是否覆盖正常、异常、边界、状态等 |
| 测试用例生成集 | 设计书 + 测试观点 | 测试用例、步骤、预期结果 | 评估生成质量 |
| Excel 交付质量集 | 结构化测试用例 | Excel 模板与字段规则 | 评估格式、公式、Sheet、下拉框、追溯关系 |

这样可以定位问题出在哪一层：

```text
最终测试用例漏了边界值
  ↓
是设计规则没提取到？
还是测试观点评价点没生成？
还是生成用例时遗漏？
还是 Excel 渲染过程丢失？
```

---

# 4. 历史数据处理流程

## 第一步：收集并配对

建立“设计书—测试式样书”的配对关系。

```text
项目 A
├── DD-LOGIN-V1.0.docx
├── UT-LOGIN-V1.0.xlsx
├── DD-ORDER-V2.1.docx
└── UT-ORDER-V2.1.xlsx
```

建议建立元数据表：

| 字段 | 说明 |
|---|---|
| project_id | 项目编号 |
| system_name | 系统名称 |
| design_document_id | 设计书编号 |
| design_version | 设计书版本 |
| test_document_id | 测试式样书编号 |
| test_version | 测试式样书版本 |
| function_id | 功能 ID |
| function_name | 功能名 |
| source_file_path | 原文件路径或对象存储地址 |
| approval_status | 是否已评审通过 |
| created_at | 创建时间 |
| reviewer | 审核人 |

> 优先选用“已执行、已评审、缺陷修正后稳定”的测试式样书。  
> 不建议把草稿、未评审文档直接当黄金标准。

---

## 第二步：解析和标准化

不同项目、不同供应商的文档格式通常不统一。要先将其标准化。

### 设计书应抽取的内容

```text
- 功能 ID、功能名称
- 画面/接口/批处理名称
- 输入项目
- 输入规则
- 处理流程
- 业务判断条件
- 输出项目
- 错误消息
- DB 读写规则
- 外部接口规则
- 权限规则
- 状态转换规则
- 设计书章节号
```

### 测试式样书应抽取的内容

```text
- 测试 ID
- 关联设计书 ID
- 测试对象
- 测试区分
- 测试观点评价点
- 前提条件
- 测试数据
- 操作步骤
- 预期结果
- 确认方法
- 实际结果
- 判定
- 备注
```

标准化时特别注意日文同义字段，例如：

```text
テスト観点
確認観点
テストポイント
確認内容
```

应统一映射到：

```text
viewpoint
```

---

## 第三步：建立“设计项—测试用例”追溯关系

这是最重要的人工治理环节。

理想关系如下：

```text
详细设计规则 R-001
  ├── 测试观点评价点 V-001
  │   ├── 测试用例 TC-001：正常输入
  │   ├── 测试用例 TC-002：空值
  │   ├── 测试用例 TC-003：最小边界
  │   └── 测试用例 TC-004：最大边界
  └── 测试观点评价点 V-002
      └── 测试用例 TC-005：错误消息确认
```

示例：

| 设计规则 ID | 设计规则 | 测试 ID | 覆盖类型 |
|---|---|---|---|
| R-LOGIN-001 | 用户 ID 必填 | UT-LOGIN-002 | 异常系 |
| R-LOGIN-002 | 密码必填 | UT-LOGIN-003 | 异常系 |
| R-LOGIN-003 | 连续失败 5 次锁定 | UT-LOGIN-010 | 状态迁移 |
| R-LOGIN-004 | 登录成功创建 Session | UT-LOGIN-001 | 正常系 / DB确认 |
| R-LOGIN-005 | 锁定后禁止认证 | UT-LOGIN-011 | 异常系 |

如果历史文档本身没有明确关联，可先用 LLM 辅助建立候选关联，再由人工确认。

---

# 5. 可量化的核心指标

不要只看“测试用例数量”。测试用例越多不一定越好，可能只是重复或虚构。

建议从以下六类指标评估。

## 5.1 设计规则提取质量

衡量 Agent 是否正确理解详细设计。

```text
规则提取 Precision = 正确提取的规则数 / Agent 提取规则总数

规则提取 Recall = 正确提取的规则数 / 黄金规则总数

规则提取 F1 = 2 × Precision × Recall / (Precision + Recall)
```

例如：

```text
黄金设计规则：20 条
Agent 提取规则：22 条
其中正确：18 条

Precision = 18 / 22 = 81.8%
Recall    = 18 / 20 = 90.0%
F1        ≈ 85.7%
```

---

## 5.2 设计覆盖率

衡量设计书中的规则是否都至少有一个有效测试用例覆盖。

```text
设计覆盖率 =
被 Agent 测试用例覆盖的设计规则数
/
黄金设计规则总数
```

例如：

```text
黄金规则：20
被覆盖规则：17

Coverage = 17 / 20 = 85%
```

对于详细设计转单体测试式样书，这通常是最重要的指标之一。

可以按规则类型分别统计：

| 规则类型 | 覆盖率 |
|---|---:|
| 正常业务流程 | 96% |
| 输入必填 | 100% |
| 格式校验 | 92% |
| 边界值 | 76% |
| 异常处理 | 68% |
| DB 更新 | 89% |
| 状态迁移 | 71% |
| 权限控制 | 83% |
| 外部接口异常 | 54% |

这样能明确 Agent 的薄弱点。

---

## 5.3 测试观点评价点覆盖率

衡量 Agent 是否考虑到应该测试的角度。

```text
测试观点 Recall =
Agent 命中的黄金测试观点数
/
黄金测试观点总数
```

例如，对于长度规则 `1～20`，理想观点一般包括：

```text
- 最小合法值：1
- 最大合法值：20
- 下界外：0
- 上界外：21
```

如果 Agent 只生成“长度超过 20”的异常测试，而没有 0、1、20 的情况，说明边界覆盖不完整。

---

## 5.4 测试用例有效性与正确性

测试用例不是按 ID 一一匹配，而是按“语义等价”做匹配。

可定义：

```text
Test Case Precision =
语义正确且可执行的 Agent 测试用例数
/
Agent 生成测试用例总数

Test Case Recall =
被 Agent 覆盖的黄金测试意图数
/
黄金测试意图总数
```

这里建议使用“规则 + LLM Judge”的混合判断：

- 规则判断：是否含设计 ID、是否有前提条件、步骤、预期结果；
- LLM Judge：是否和黄金测试意图语义等价；
- 人工抽检：校准 Judge 的可靠性。

---

## 5.5 幻觉率 / 虚构率

这是自动生成测试式样书必须重点监控的指标。

```text
Hallucination Rate =
设计书中找不到依据的测试用例数
/
Agent 生成测试用例总数
```

虚构通常包括：

- 设计书中没有的字段；
- 不存在的 API、DB 表或错误消息；
- 没有规定的权限规则；
- 没有依据的性能指标；
- 设计书未说明却强行假设的业务处理。

建议再区分严重程度：

| 类型 | 示例 | 严重度 |
|---|---|---|
| 字段虚构 | 设计中无“部门编码”，测试中出现该字段 | 高 |
| 错误消息虚构 | 自行生成日文错误文案 | 高 |
| DB 行为臆测 | 自行假设某表新增/更新 | 高 |
| 测试数据补充 | 合理补充普通测试账号 | 低/中 |
| 测试步骤表达差异 | 描述不同但不改变含义 | 低 |

---

## 5.6 可执行性和交付格式质量

这部分可通过确定性规则校验，不需要完全依赖 LLM。

### 用例完整性

```text
完整用例数 / 总用例数
```

每条用例至少应具备：

```text
测试 ID
+ 测试对象
+ 测试观点评价点
+ 前提条件
+ 操作步骤
+ 预期结果
+ 确认方法
+ 设计书关联
```

### Excel 合规率

检查：

- 是否存在必需 Sheet；
- 列名、顺序、样式是否符合模板；
- 测试 ID 是否唯一；
- 判定列是否有有效下拉项；
- 合并单元格、冻结窗格、过滤器是否符合要求；
- 是否存在超出模板范围的数据；
- 是否有空白必填字段；
- 是否存在损坏的公式、无效链接或乱码。

---

# 6. 评测不应只依赖历史测试式样书

历史测试式样书不一定完整，也可能存在缺陷。

因此建议黄金标准分为三层：

```text
Level 1：历史测试式样书原始内容
Level 2：历史设计书 + 历史测试式样书自动结构化结果
Level 3：人工 Review 后的标准测试观点评价点与追溯关系
```

其中：

- **Level 1**：用于快速构建初始 Dataset；
- **Level 2**：用于批量评测和检索；
- **Level 3**：用于高可信 Benchmark 和 Prompt/模型发布门禁。

建议先挑选 30～50 个典型功能，人工整理成高质量黄金集，而不是一开始整理几千份文档。

---

# 7. Dataset 切分策略：防止“记住模板”而不是学会测试设计

必须避免训练/优化集与测试集泄漏。

## 推荐划分

```text
训练/提示词设计集：60%
验证集：20%
固定测试集：20%
```

更关键的是，不能随机按文件切分，而要按**项目、功能族或版本**切分。

### 不推荐

```text
同一个“用户注册”功能的 V1 进入训练集，
V1.1 进入测试集。
```

这样设计和测试用例高度相似，分数会虚高。

### 推荐

```text
训练集：
- 项目 A：HR、财务模块
- 项目 B：订单模块

验证集：
- 项目 C：采购模块

固定测试集：
- 项目 D：审批模块
- 或项目 A 中从未出现过的功能域
```

还可以按难度分层：

| 难度 | 特征 |
|---|---|
| Easy | 单画面、简单输入校验、无 DB 或外部接口 |
| Medium | 多字段校验、DB 读写、状态变化 |
| Hard | 多模块、权限、事务、异步、外部接口、批处理、复杂条件组合 |

---

# 8. 在 Langfuse 中如何组织

可以在 Langfuse 中按以下方式使用。

## Dataset

每个 Dataset Item：

```text
input：
  - detailed_design_text
  - function_metadata
  - output_excel_template_version
  - test_level

expected_output：
  - 黄金规则
  - 黄金测试观点
  - 黄金测试用例
  - 追溯矩阵
  - 未决事项
```

示例：

```json
{
  "input": {
    "function_id": "LOGIN",
    "design_text": "……",
    "test_level": "unit",
    "template_version": "UT_TEMPLATE_V3"
  },
  "expected_output": {
    "requirements": ["R-001", "R-002", "R-003"],
    "viewpoints": ["必填", "边界值", "账号锁定"],
    "test_intents": ["TC-LOGIN-001", "TC-LOGIN-002"]
  }
}
```

## Trace

一次 Agent 运行记录为一个 Trace：

```text
Trace: generate_test_specification
├── 输入：设计书内容、版本、功能 ID
├── Agent：设计书解析
├── Agent：规则提取
├── Agent：测试观点生成
├── Agent：测试用例生成
├── Agent：覆盖审查
├── LLM：最终结构化 JSON
├── Tool：Excel 文件生成
└── 输出：测试用例数、未决事项数、Excel 地址
```

## Score

建议至少写入这些 Score：

| Score 名称 | 类型 | 例子 |
|---|---|---|
| `requirement_recall` | 数值 | `0.92` |
| `requirement_precision` | 数值 | `0.86` |
| `design_coverage` | 数值 | `0.88` |
| `viewpoint_recall` | 数值 | `0.84` |
| `test_case_precision` | 数值 | `0.81` |
| `hallucination_rate` | 数值，越低越好 | `0.04` |
| `case_completeness_rate` | 数值 | `0.98` |
| `excel_validation_pass` | 布尔值 | `true` |
| `human_review_score` | 数值 | `4.2 / 5` |
| `human_review_pass` | 布尔值 | `true` |
| `cost` | 数值 | 调用成本 |
| `latency_seconds` | 数值 | 总耗时 |

然后按以下维度筛选和分析：

```text
- 项目
- 设计书模板版本
- 功能类型
- 复杂度
- 是否含 DB
- 是否含状态迁移
- 是否含外部接口
- 使用的模型
- Prompt 版本
- Agent 版本
- Excel 模板版本
```

---

# 9. 建议设置“发布门禁”

每次修改 Prompt、模型、Agent 工作流或 Excel 模板时，都在固定 Benchmark 上运行 Experiment。

例如设定上线门槛：

| 指标 | 最低要求 |
|---|---:|
| 设计规则 Recall | ≥ 90% |
| 设计覆盖率 | ≥ 90% |
| 测试观点 Recall | ≥ 85% |
| 测试用例 Precision | ≥ 85% |
| 幻觉率 | ≤ 3% |
| 必填字段完整率 | ≥ 99% |
| Excel 格式校验通过率 | 100% |
| 人工 Review 平均分 | ≥ 4.0 / 5 |

同时设“不可退化”约束：

```text
新版本不得使固定测试集的：
- 设计覆盖率下降超过 2%
- 幻觉率上升超过 1%
- 人工 Review 通过率下降超过 3%
```

---

# 10. 让系统可观测：记录每一层的中间产物

如果只记录最终 Excel，就很难定位问题。

建议在每次生成任务中保存以下中间结果：

```text
1. 原始设计书文本 / 章节
2. 文档解析结果
3. 功能切分结果
4. 提取出的设计规则
5. 提取出的字段与边界条件
6. 生成的测试观点评价点
7. 初版测试用例
8. 覆盖审查结果
9. 幻觉审查结果
10. 最终 JSON
11. 最终 Excel
12. 人工修改后的 Excel 或修改意见
```

这样当某个结果不好时，可以判断：

```text
设计书解析错误？
    ↓
规则提取遗漏？
    ↓
测试观点生成不全？
    ↓
测试用例生成质量差？
    ↓
覆盖审查没有发现问题？
    ↓
Excel 渲染错误？
```

---

# 11. 人工 Review 结果要回流为 Dataset

这是“可演进”的关键。

人工 Review 时，不要只改 Excel；还应结构化记录修改类型。

| 修改类型 | 示例 | 应如何使用 |
|---|---|---|
| 漏测 | 缺少最大长度边界测试 | 新增到黄金测试观点 |
| 误测 | 设计中没有该字段却生成测试 | 作为幻觉负例 |
| 预期结果不完整 | 缺少 DB 回滚确认 | 补充黄金预期结果 |
| 测试层级错误 | 把接口联调写进单体测试 | 增加测试层级规则 |
| 日文术语错误 | 画面名、消息文案不一致 | 增加术语表和校验规则 |
| 重复用例 | 多个 Case 验证同一规则 | 增加去重策略 |
| 设计书歧义未识别 | 本应提出确认但 Agent 自行假设 | 新增“未决事项”样本 |

可以建立反馈表：

```json
{
  "dataset_item_id": "DD-LOGIN-001",
  "test_case_id": "UT-LOGIN-010",
  "review_action": "modify",
  "issue_type": "missing_boundary_case",
  "review_comment": "缺少密码长度为 33 位的上界外测试。",
  "corrected_case": {
    "viewpoint": "パスワードの最大桁数超過",
    "test_data": ["パスワード：33桁の英数字"],
    "expected_results": ["桁数エラーメッセージが表示される", "DB更新されない"]
  }
}
```

这些数据可以持续补充到：

- 回归测试集；
- 幻觉检测集；
- 高难度样本集；
- Prompt few-shot 示例库；
- 审查 Agent 的评测集。

---

# 12. 一个务实的分阶段落地方案

## 第一阶段：建立最小 Benchmark

目标：先验证系统是否真的有价值。

建议范围：

```text
30～50 个功能
每个功能 10～30 条高质量测试用例
覆盖登录、CRUD、审批、批处理、接口、权限等典型类型
```

产物：

- 标准化 JSON Dataset；
- 设计—测试追溯矩阵；
- 基础指标；
- Excel 结构校验器；
- 人工评审表。

---

## 第二阶段：实现自动评测

加入：

```text
- 设计规则提取 Recall / Precision
- 测试观点覆盖率
- 设计覆盖率
- 幻觉率
- 用例完整率
- Excel 格式校验
- LLM Judge
```

此阶段目标是：能比较不同 Prompt、模型、Agent 版本。

---

## 第三阶段：在 Langfuse 上做实验与回归

每次版本变更后：

```text
Agent v1.0
Prompt v1.2
Model A
        vs
Agent v1.1
Prompt v1.3
Model B
```

对固定 Dataset 批量运行，记录 Experiment 和 Score，确认是否满足发布门禁。

---

## 第四阶段：从“生成”升级为“协同编写”

成熟后可采用以下模式：

```text
Agent 自动生成初版测试式样书
  ↓
测试工程师 Review 和修改
  ↓
修改意见结构化回流
  ↓
更新 Dataset、术语库、规则库和评测集
  ↓
下一版本 Agent 改进
```

最终不是追求完全无人化，而是提高测试设计人员的产出和一致性。

---

# 13. 最终建议

对于这个场景，建议优先关注以下四个指标：

```text
1. 设计覆盖率（是否漏测）
2. 幻觉率（是否乱写）
3. 测试用例有效性（是否真的可执行、可验证）
4. 人工 Review 修改率（是否节省了实际工作量）
```

可以将质量目标简化为：

```text
生成的测试用例：
- 不漏关键设计规则；
- 不虚构设计书未定义的内容；
- 预期结果可验证；
- 能正确写入日式测试式样书模板；
- 经人工 Review 后只需少量修改。
```

这比“生成的 Excel 是否和历史 Excel 完全一致”更可靠，也更适合持续迭代。



---

**User** _(2026-08-24 14:10:48)_

**(BTG00000)BTS_内部仕様_ログイン_ログイン.xlsx**

- [(BTG00000)BTS_内部仕様_ログイン_ログイン.xlsx](https://sws-gpt-cn-api.sys.sws.com/api/files/chat-attachments/file/h6a61fTs5mp_CKcBwQOgixSulC9fPrgw_k3h1iaaXk0/01M0S2M7Q9A37RA7B4DAQWRT3T.xlsx)

**プログラム仕様書：BTG00000_LOGIN画面.xlsx**

- [プログラム仕様書：BTG00000_LOGIN画面.xlsx](https://sws-gpt-cn-api.sys.sws.com/api/files/chat-attachments/file/h6a61fTs5mp_CKcBwQOgixSulC9fPrgw_k3h1iaaXk0/01M0S2SNVXBW2EVFV2CJV7FDEZ.xlsx)

比如下面这两个已经做好的内部式样和单体测试式样，如何建立评测集，详细一点（其中プログラム仕様書：BTG00000_LOGIN画面.xlsx 是单体测试式样）

---

**Bot** _(2026-08-24 14:13:07)_

可以，但先要澄清一个关键点：

我检查了你提供的两个工作簿。根据**文件实际内容**，它们目前更像是：

1. **`(BTG00000)BTS_内部仕様_ログイン_ログイン.xlsx`**  
   登录画面的上游设计资料，包含：
   - 画面定义；
   - 画面项目编辑定义；
   - 输出编辑定义；
   - 输入检查规格；
   - LDAP 认证规格。

2. **`プログラム仕様書：BTG00000_LOGIN画面.xlsx`**  
   标题为“プログラム仕様書”，内容包含：
   - 处理概要；
   - 画面程序流程；
   - DB 表；
   - 字段列表；
   - 初期显示 SQL；
   - 登录按钮的详细处理；
   - 登录履历登记规则。

第二个文件中没有看到通常“单体测试式样书”应有的字段，例如：

```text
テスト項目ID
テスト観点
事前条件
テストデータ
操作手順
期待結果
確認方法
実績結果
判定
```

因此，**这两个文件当前最适合先构建的是：**

```text
BTS 内部规格 / 画面规格
        ↓
程序详细规格书（プログラム仕様書）
```

的评测集。

如果你的最终目标是评测：

```text
内部设计 / 详细设计
        ↓
单体测试式样书
```

则还需要提供真正的 `単体テスト仕様書` 文件；或者将第二个文件作为“详细设计输入”，再由 Agent 生成单体测试式样书。

---

我已根据这两个文件建立了一个可编辑的**评测集种子模板**：

[下载：RAG / Agent 评测集种子模板（Excel）](https://sws-gpt-cn-api.sys.sws.com/api/files/generated-file/file/assistants/01M0S2ZGK8XDP6HRGHPGVD8Y5Q.xlsx)

其中已预填了 BTG00000 登录功能的来源单元格、规范化事实、追溯关系和待人工确认的问题。

---

# 1. 不要以“Excel 文本完全一致”作为评测标准

如果直接让 Agent 生成 Excel，再与历史 Excel 做 Diff，会有很多误判。

例如，以下两种测试描述语义等价：

```text
用户名为空时，显示必填错误消息。
```

```text
USERID 未输入时，显示“ユーザIDを入力してください。”。
```

但 Excel 文本不同。

因此，应该将历史文档转换成中间的**结构化评测对象**：

```text
设计书单元格
    ↓
原子设计事实（Design Facts）
    ↓
测试观点评价点（Test Viewpoints）
    ↓
测试意图（Test Intents）
    ↓
测试用例（Test Cases）
    ↓
Excel 渲染结果
```

其中，真正用于评测的核心不是 Excel，而是：

- 设计规则是否被正确识别；
- 规则是否被测试覆盖；
- 测试是否有依据；
- 是否虚构字段、DB 表、消息或规则；
- 测试步骤和预期结果是否可执行、可验证；
- 最后生成的 Excel 是否符合交付模板。

---

# 2. 基于 BTG00000 可拆出的首批 Dataset Item

不要把整个登录画面作为一条 Dataset。应按**功能行为单元**拆分。

当前文件可先拆为以下 5 条评测数据。

| Dataset Item | 输入范围 | 应验证的目标 |
|---|---|---|
| `BTG00000-INIT-001` | 初期显示时 INFORMATION 取得规则 | 是否生成正确的信息取得逻辑、有效期和删除标志条件 |
| `BTG00000-LOGIN-VALIDATION-001` | 登录按钮的 4 类错误校验 | 是否覆盖未输入、用户不存在、用户已删除、密码错误 |
| `BTG00000-LOGIN-HISTORY-001` | 登录履历输出定义 | 是否在成功登录时正确登记 4 个字段 |
| `BTG00000-SCREEN-FIELDS-001` | 画面项目定义 | 是否保留 USERID、PASSWORD、按钮、长度、必填和掩码属性 |
| `BTG00000-EXIT-001` | EXIT 按钮定义 | 是否生成关闭画面的处理说明或测试用例 |

例如，登录校验的 Dataset 输入可以不是完整 Excel，而是已抽取并带证据定位的结构化内容：

```json
{
  "dataset_item_id": "BTG00000-LOGIN-VALIDATION-001",
  "input": {
    "function_id": "BTG00000",
    "function_name": "ログイン画面",
    "source_evidence": [
      {
        "sheet": "ログイン【チェック仕様】",
        "cells": "D9:S12",
        "content": "USERIDまたはPASSWORDが入力されていない場合、エラーメッセージを表示する。..."
      }
    ]
  },
  "expected_output": {
    "required_facts": [
      "USERID または PASSWORD が未入力の場合、エラー表示",
      "USERID が存在しない場合、エラー表示",
      "削除フラグ=1 のユーザーの場合、エラー表示",
      "PASSWORD が不正の場合、エラー表示"
    ]
  }
}
```

---

# 3. 评测集的核心：建立“原子事实（Gold Facts）”

所谓“原子事实”，是指不能再继续拆分、且可以明确判断对错的设计要求。

例如，“登录成功后写入登录履历”太粗，应拆成以下事实：

| Gold Fact ID | 原子事实 |
|---|---|
| `GF-HIST-001` | 成功登录时写入登录履历 |
| `GF-HIST-002` | 登录 ID 从 `INTLOGINID_SEQ` 取得 |
| `GF-HIST-003` | 用户 ID 取自画面 USERID |
| `GF-HIST-004` | 登录日期时间取系统日期 |
| `GF-HIST-005` | 终端 IP 地址取登录终端 IP |

这样 Agent 即使生成的表述与历史文档不同，也可以判断其是否真正覆盖了规则。

---

## 当前 BTG00000 中已识别的关键事实

### A. 初期显示：INFORMATION 取得

来源：

```text
ログイン【画面編集定義書】!B9:S9
```

可建立的事实包括：

```text
GF-INIT-001：初期显示时取得 INFORMATION。
GF-INIT-002：只取得有效消息。
GF-INIT-003：系统日期在有效期 From～To 内。
GF-INIT-004：取得的信息显示在登录画面。
GF-INIT-005：按更新时间降序排列。
```

其中 `GF-INIT-005` 是目标程序规格书 SQL 中存在、但上游 BTS 文档中没有明确写出的排序规则。它不应直接算作“Agent 必须生成”的需求，而应标记为：

```text
target_only / allowed_enhancement / needs_review
```

---

### B. 登录校验

来源：

```text
ログイン【チェック仕様】!D9:S12
```

可建立的核心事实：

```text
GF-LOGIN-001：USERID 或 PASSWORD 未输入时显示错误。
GF-LOGIN-002：未输入错误消息文本正确。
GF-LOGIN-003：USERID 不存在时显示错误。
GF-LOGIN-004：用户不存在错误消息文本正确。
GF-LOGIN-005：删除标志=1 时显示错误。
GF-LOGIN-006：删除用户错误消息文本正确。
GF-LOGIN-007：PASSWORD 不匹配时显示错误。
GF-LOGIN-008：密码错误消息文本正确。
GF-LOGIN-009：点击错误消息 OK 后返回登录画面。
GF-LOGIN-010：任何校验错误时，不得登记登录履历。
```

---

### C. 画面字段

来源：

```text
ログイン【画面編集定義書】!B9:S13
```

可建立的事实：

```text
GF-FIELD-001：INFORMATION 为全角，最大 1280 字符。
GF-FIELD-002：USERID 为半角英数，最大 10 位，必填。
GF-FIELD-003：PASSWORD 为半角英数，最大 20 位，必填，掩码显示。
GF-FIELD-004：LOGIN 是按钮。
GF-FIELD-005：EXIT 是按钮，点击后关闭画面。
```

---

# 4. 当前文件中发现的“差异”不能直接当作错误

历史文档本身未必完全一致。因此评测集必须有一个**差异审查队列（Review Queue）**。

我已在模板的 `Review_Queue` Sheet 中预填了以下问题。

| 编号 | 发现的问题 | 为什么重要 |
|---|---|---|
| `RQ-001` | 来源检查规格的消息代码为 `E00001～E00004`，目标程序规格书为 `E00011～E00013` | 可能是版本变更，也可能是文档不一致 |
| `RQ-002` | 密码错误在来源中为 `E00004`，目标中却是 `E00011` | `E00011` 同时用于“未输入”，需要重点确认 |
| `RQ-003` | 来源定义 USERID 最大 10 位、PASSWORD 最大 20 位；目标字段列表未填写最大桁数 | 可能是目标规格遗漏 |
| `RQ-004` | 目标目录存在“4-3 EXITボタン押下時”，但正文有效内容中未看到其详细规格 | 可能是截断、版本差异或遗漏 |
| `RQ-005` | 来源要求删除标志=`0`，目标 SQL 使用 `<> '1'`；目标还增加了排序规则 | 需确认语义等价性及是否允许增强 |

---

## 很重要：将事实分为不同断言等级

不能把所有历史目标文档内容都视为正确答案。建议为每个事实设置 `assertion_level`。

| 类型 | 说明 | 评测处理 |
|---|---|---|
| `must` | 来源设计明确规定，必须覆盖 | 未生成则算漏测 |
| `should` | 最佳实践或项目规则要求 | 单独统计，不一定阻断 |
| `target_only` | 历史目标中存在，但来源没有明确依据 | 不要求 Agent 必须生成 |
| `legacy_omission` | 来源明确有，但历史目标遗漏 | 新 Agent 生成反而应视为改进 |
| `version_delta` | 来源与目标冲突，疑似版本差异 | 人工确认前不参与硬性评分 |
| `ambiguous` | 设计书无法确定 | Agent 应提出未决事项，而不是猜测 |

例如，USERID 长度 `10`、PASSWORD 长度 `20`：

```text
来源 BTS：明确规定。
目标程序规格书：未体现。
```

如果你的目标是“复现历史程序规格书”，它可能被标记为“历史输出未覆盖”。  
但如果目标是“生成高质量测试式样书”，则 Agent 应该基于来源设计生成边界值测试，这反而是正确行为。

---

# 5. 如果最终目标是生成“单体测试式样书”，黄金数据应如何设计？

当拿到实际的单体测试式样书后，每条 Test Intent 至少需要以下结构。

```json
{
  "test_intent_id": "TCI-LOGIN-REQUIRED",
  "dataset_item_id": "BTG00000-LOGIN-VALIDATION-001",
  "design_fact_ids": [
    "GF-LOGIN-001",
    "GF-LOGIN-002",
    "GF-LOGIN-010"
  ],
  "test_category": "異常系",
  "test_viewpoint": "USERID/PASSWORD 必填校验",
  "preconditions": [
    "登录画面已显示"
  ],
  "test_data": [
    "USERID：空",
    "PASSWORD：有效值"
  ],
  "steps": [
    "输入测试数据",
    "点击 LOGIN 按钮"
  ],
  "expected_results": [
    "显示「ユーザID、パスワードを入力してください。」",
    "不执行 LDAP 认证",
    "不新增登录履历"
  ],
  "verification_method": [
    "画面确认",
    "应用日志确认",
    "登录履历表 SQL 确认"
  ]
}
```

注意：一个设计规则不一定只对应一个测试用例。

例如 USERID 最大长度为 10，至少可产生：

```text
长度 9：正常边界内
长度 10：最大合法值
长度 11：超过最大值
全角字符：字符种类异常
空值：必填异常
```

因此，评测的最小单位最好是“**测试意图**”，而不是历史 Excel 中的一整行。

---

# 6. 建议拆成四层评测

## 第一层：设计事实提取评测

输入：

```text
BTS / 内部设计书内容
```

输出：

```text
输入项目、长度、格式、必填、DB 操作、异常条件、消息、权限、状态等事实。
```

指标：

```text
Fact Precision = 正确提取事实数 / Agent 提取事实数
Fact Recall = 正确提取事实数 / 黄金事实数
Fact F1 = 2 × Precision × Recall / (Precision + Recall)
```

---

## 第二层：追溯关系评测

输入：

```text
来源设计事实 + Agent 生成的程序规格/测试用例
```

输出：

```text
设计事实 → 目标章节 / 测试用例 的关联关系。
```

指标：

```text
Traceability Recall =
已建立正确追溯关系的 must 事实数
/
全部 must 事实数
```

例如：

```text
GF-HIST-003
“登录履历.用户ID 取画面 USERID”
        ↓
程序规格书 A165
        或
测试用例 TCI-LOGIN-SUCCESS-HISTORY
```

---

## 第三层：测试覆盖评测

这是“设计转测试式样书”最重要的指标。

```text
Design Coverage =
至少被一个有效测试意图覆盖的 must 设计事实数
/
must 设计事实总数
```

可以按类别拆分：

| 指标 | 说明 |
|---|---|
| `validation_coverage` | 输入校验是否覆盖 |
| `boundary_coverage` | 长度、数量、日期等边界是否覆盖 |
| `db_coverage` | DB 新增、更新、删除、回滚是否覆盖 |
| `error_message_coverage` | 错误消息是否覆盖 |
| `state_coverage` | 状态变化是否覆盖 |
| `negative_effect_coverage` | 异常时“不应发生的 DB/API 操作”是否覆盖 |

---

## 第四层：幻觉与矛盾评测

Agent 自动生成测试式样书时，最需要控制的是“编造”。

```text
Hallucination Rate =
无法在来源设计书中找到依据的生成事实数
/
Agent 生成事实总数
```

例如以下情况应算高风险幻觉：

- 设计书中没有“验证码”，Agent 却生成验证码测试；
- 设计书中没有“账户连续失败 5 次锁定”，Agent 却生成锁定测试；
- 设计书未定义某 DB 表，Agent 却写入该表；
- Agent 自行编造错误消息编号和日文消息；
- 设计书没有说明性能要求，却生成具体 TPS 或响应时间门槛。

对于每个 Agent 生成的测试用例，建议要求它输出引用依据：

```json
{
  "test_id": "UT-LOGIN-007",
  "design_evidence": [
    {
      "sheet": "ログイン【チェック仕様】",
      "cells": "D12:S12"
    }
  ]
}
```

没有证据的测试用例进入人工审查或降低可信度。

---

# 7. Langfuse 中的 Dataset、Trace 与 Score 设计

## Dataset Item 输入

建议将 Excel 预先抽取为 JSON/Markdown，而不是每次直接丢整个二进制 Excel。

```json
{
  "function_id": "BTG00000",
  "function_name": "ログイン画面",
  "task": "generate_unit_test_spec",
  "design_sections": [
    {
      "source_file": "(BTG00000)BTS_内部仕様_ログイン_ログイン.xlsx",
      "sheet": "ログイン【チェック仕様】",
      "cells": "D9:S12",
      "text": "..."
    }
  ],
  "template_version": "UT_SPEC_V1"
}
```

## Expected Output

```json
{
  "gold_facts": ["GF-LOGIN-001", "GF-LOGIN-002"],
  "required_test_intents": [
    "TCI-LOGIN-REQUIRED",
    "TCI-LOGIN-USER-NOT-FOUND",
    "TCI-LOGIN-USER-DELETED",
    "TCI-LOGIN-PASSWORD-WRONG"
  ],
  "forbidden_or_unverified_facts": [
    "账户锁定",
    "验证码"
  ]
}
```

## Trace 建议分段记录

```text
Trace: generate_unit_test_spec

├── Observation: extract_excel_content
├── Observation: normalize_design_facts
├── Generation: classify_test_viewpoints
├── Generation: generate_test_intents
├── Generation: generate_test_cases
├── Generation: self_review_coverage
├── Tool: render_excel
├── Tool: validate_excel
└── Generation: final_summary
```

## 推荐 Score

| Score | 含义 | 目标趋势 |
|---|---|---|
| `fact_precision` | 事实提取准确率 | 越高越好 |
| `fact_recall` | 事实提取召回率 | 越高越好 |
| `design_coverage` | 设计规则覆盖率 | 越高越好 |
| `test_intent_recall` | 黄金测试意图覆盖率 | 越高越好 |
| `hallucination_rate` | 无依据生成比例 | 越低越好 |
| `contradiction_rate` | 与设计冲突比例 | 越低越好 |
| `traceability_rate` | 用例能回溯到设计证据的比例 | 越高越好 |
| `case_completeness_rate` | 测试用例字段完整率 | 越高越好 |
| `excel_validation_pass` | Excel 模板校验是否通过 | 必须为真 |
| `human_review_score` | 人工评审评分 | 越高越好 |
| `review_edit_rate` | 人工修改的用例比例 | 越低越好 |
| `latency_seconds` | 生成耗时 | 越低越好 |
| `cost` | 单次生成成本 | 越低越好 |

---

# 8. 建议的质量门禁

例如，初期可设置以下目标：

| 指标 | 初期门槛 | 稳定后门槛 |
|---|---:|---:|
| 设计事实 Recall | ≥ 85% | ≥ 92% |
| 设计覆盖率 | ≥ 85% | ≥ 92% |
| 测试意图 Recall | ≥ 80% | ≥ 90% |
| 幻觉率 | ≤ 8% | ≤ 3% |
| 用例字段完整率 | ≥ 95% | ≥ 99% |
| 证据可追溯率 | ≥ 90% | ≥ 98% |
| Excel 模板校验 | 100% | 100% |
| 人工评审通过率 | ≥ 70% | ≥ 85% |

每次修改以下内容，都必须在固定 Benchmark 上运行 Langfuse Experiment：

```text
- LLM 模型
- Prompt
- Agent 工作流
- 文档切分策略
- 测试观点评价规则
- Excel 模板
- 术语表
- Reranker / RAG 知识库
```

并确保新版本没有明显退化：

```text
设计覆盖率不得下降超过 2%
幻觉率不得增加超过 1%
人工评审通过率不得下降超过 3%
```

---

# 9. 这一个样本目前能做什么、不能做什么？

当前这对登录文档很适合做：

```text
Smoke Test / Golden Example / 回归样本
```

即用于验证 Agent 是否能处理：

- 画面字段；
- 输入校验；
- 错误消息；
- 初期显示 SQL；
- DB 插入；
- LDAP 认证相关逻辑；
- EXIT 处理；
- 设计间追溯。

但**单一功能不能作为正式 Benchmark**。因为 Agent 可能只是“记住登录模板”，而不是掌握设计转测试的能力。

建议后续至少收集：

```text
30～50 个功能单元
```

并覆盖：

| 功能类型 | 建议样本 |
|---|---:|
| 简单查询画面 | 5～10 |
| CRUD 维护画面 | 10～15 |
| 输入校验密集型画面 | 5～10 |
| 登录/权限/认证 | 3～5 |
| 状态迁移、审批流程 | 5～10 |
| 批处理、文件导入导出 | 5～10 |
| 外部接口/DB 事务 | 5～10 |

切分训练、验证、固定回归集时，不要把同一功能的不同版本分别放到训练集和测试集。例如：

```text
登录 V1 放训练集
登录 V1.1 放测试集
```

这会造成严重泄漏。应按项目、模块或功能族进行切分。

---

# 10. 下一步建议

建议按以下顺序推进：

1. **先人工确认模板中 `Review_Queue` 的 5 个差异。**
2. 明确第二个文件是否确实被项目定义为“单体测试式样书”；若不是，请补充真实的测试式样书。
3. 基于本模板建立统一的 JSON Schema：
   - `DesignFact`
   - `TestIntent`
   - `TestCase`
   - `TraceLink`
   - `ReviewIssue`
4. 选取 30～50 个历史功能，按同样方式抽取和人工标注。
5. 将人工确认后的 `must` 事实和测试意图导入 Langfuse Dataset。
6. 为 Agent 输出增加“设计证据单元格引用”字段。
7. 实现规则评测、LLM Judge 和人工抽检三层评测。
8. 每次 Prompt/Agent 改造均在固定 Dataset 上做 Langfuse Experiment。

最关键的原则是：

> **历史测试式样书是高价值参考，但不是天然正确的黄金答案。**  
> 应把它与来源设计书对照、审查、结构化，形成“设计事实 + 测试意图 + 追溯关系”的可演进黄金集。



---

**User** _(2026-08-24 14:23:08)_

这个评测集是用llm来生成加人工抽检的形式吧，要不然式样书太多了，没法所有都人工生成那么多数据，还有等价的描述也需要llm语义理解来判断吧

---

**Bot** _(2026-08-24 14:23:57)_

对，正确的方向不是“人工手工制作全部评测数据”，而是：

> **LLM 批量生成/结构化标注 + 规则校验 + 人工分层抽检 + 人工反馈回流。**

并且，对于“等价描述”“同一测试意图的不同写法”“同一规则被不同用例组合覆盖”等问题，确实需要引入 **LLM 语义判定（LLM-as-a-Judge）**，不能只做字符串比较。

但要注意：LLM 自动标注的数据不能直接全部视为黄金标准。更合理的是建立**分级可信度的数据集**。

---

# 1. 推荐的数据集分层

```text
原始历史文档
    ↓
LLM 自动抽取、匹配和标注
    ↓
Silver Dataset（银标数据）
    ↓
人工分层抽检、修正、确认
    ↓
Gold Dataset（金标数据）
    ↓
固定回归集 / 发布门禁集
```

| 层级 | 名称 | 来源 | 用途 | 是否可直接作为发布门禁 |
|---|---|---|---|---|
| L0 | Raw Dataset | 原始设计书、原始测试式样书 | 保留证据、可回溯 | 否 |
| L1 | Silver Dataset | LLM 自动结构化和语义匹配 | 大规模趋势分析、候选评测 | 不建议单独使用 |
| L2 | Gold Dataset | 人工抽检、修正、确认过的样本 | 模型/Prompt/Agent 回归评测 | 是 |
| L3 | Critical Gold Set | 关键功能、严重缺陷案例、人工复审样本 | 上线门禁、不可退化检查 | 是，优先级最高 |

这里的“人工”不是人工从零写出一万个测试用例，而是：

```text
LLM 先完成 80%～95% 的抽取、对齐和初版标注
        ↓
人工只审核高风险、低置信度、冲突和抽样数据
        ↓
人工确认后的结论沉淀为金标数据
```

---

# 2. 哪些工作可以交给 LLM 自动完成？

针对历史的：

```text
内部设计书 / 详细设计书
        +
单体测试式样书
```

LLM 可以批量完成以下任务。

| 工作 | 是否适合 LLM 批量做 | 说明 |
|---|---|---|
| Excel 内容抽取与结构化 | 适合，但先用程序读取单元格 | LLM 用于理解表格含义和合并单元格上下文 |
| 设计规则提取 | 很适合 | 提取输入、校验、异常、DB、接口、权限等 |
| 测试用例字段标准化 | 很适合 | 将不同项目模板映射为统一 Schema |
| 设计规则与测试用例关联 | 很适合 | 建立 Design Fact → Test Case 追溯关系 |
| 识别测试类别 | 很适合 | 正常系、异常系、边界值、权限、状态迁移等 |
| 判断两个测试用例是否语义等价 | 很适合 | 不能只依赖文本 Diff |
| 发现来源与历史测试式样书不一致 | 适合 | 作为“待人工确认”项，而非直接判错 |
| 生成初版测试意图 | 很适合 | 尤其适合补全历史测试式样书的缺失项 |
| Excel 格式、必填列、ID 唯一性检查 | 不需要 LLM | 应采用确定性规则 |
| 关键业务规则最终确认 | 不宜只依赖 LLM | 必须人工确认或有权威规则来源 |
| 高风险业务的验收标准 | 不宜只依赖 LLM | 合规、金融、医疗、权限等尤其如此 |

---

# 3. “等价描述”应该如何评测？

你说得对，测试用例不可能要求文本完全一致。

例如历史测试式样书写的是：

```text
ユーザIDが未入力の場合、エラーメッセージを表示する。
```

Agent 生成的是：

```text
USERID を空欄にして LOGIN ボタンを押下した場合、
「ユーザID、パスワードを入力してください。」を表示し、
認証処理およびログイン履歴登録を行わない。
```

第二种描述不仅语义等价，而且信息更完整。

如果只做字符串比较，会得到错误结论：

```text
字符串不一致
→ 误判为错误
```

正确做法是判断其是否满足同一个**测试意图（Test Intent）**。

---

## 测试意图层的语义判断

不直接比较两段自然语言，而是先把测试用例归一化为：

```json
{
  "target": "登录画面",
  "condition": "USERID 为空",
  "action": "点击 LOGIN",
  "expected_behavior": [
    "显示必填错误消息",
    "不执行认证",
    "不登记登录履历"
  ],
  "test_category": "异常系",
  "covered_design_facts": [
    "GF-LOGIN-001",
    "GF-LOGIN-002",
    "GF-LOGIN-010"
  ]
}
```

之后用 LLM Judge 判断：

```text
历史测试用例与 Agent 生成用例是否验证同一个业务规则？
Agent 用例是否覆盖了历史用例的必要预期？
Agent 是否引入了设计书中不存在的假设？
Agent 是否遗漏了必要结果？
```

Judge 输出应是结构化结果，而不是一句“看起来差不多”。

```json
{
  "semantic_relation": "covers",
  "equivalence_score": 0.92,
  "historical_case_covered": true,
  "generated_case_supported_by_design": true,
  "missing_requirements": [],
  "unsupported_claims": [],
  "reason": "两者均验证 USERID 未输入时的登录拒绝。Agent 额外验证了不进行认证和不写入登录履历，且该行为可由设计规则支持。"
}
```

推荐的关系类型：

| `semantic_relation` | 含义 |
|---|---|
| `equivalent` | 两者验证相同测试意图，覆盖范围相当 |
| `covers` | Agent 用例覆盖历史用例，且更完整 |
| `partially_covers` | Agent 覆盖了部分要求，但遗漏关键条件或结果 |
| `different` | 测试目标不同，不能互相替代 |
| `contradicts` | Agent 结果与设计或历史确认结果冲突 |
| `unsupported` | Agent 引入无设计依据的内容 |
| `ambiguous` | 设计书不足以判断，需要人工确认 |

---

# 4. LLM 自动建集的推荐流水线

## 步骤 1：先用程序抽取 Excel，不要直接让 LLM“看 Excel”

Excel 的读取、合并单元格处理、Sheet 名称、行列坐标、格式校验应由程序完成。

抽取后保留每段内容的来源位置：

```json
{
  "source_file": "(BTG00000)BTS_内部仕様_ログイン_ログイン.xlsx",
  "sheet": "ログイン【チェック仕様】",
  "cells": "D9:S12",
  "text": "USERIDまたはPASSWORDが入力されていない場合……"
}
```

这样后续每个 LLM 结论都有证据可追溯。

---

## 步骤 2：LLM 从设计书提取原子事实

让 LLM 将设计书转为 `DesignFact`。

```json
{
  "fact_id": "GF-LOGIN-001",
  "fact_type": "validation",
  "assertion_level": "must",
  "condition": "USERID または PASSWORD が未入力",
  "expected_behavior": "エラーメッセージを表示し、ログイン処理を行わない",
  "evidence": {
    "file": "(BTG00000)BTS_内部仕様_ログイン_ログイン.xlsx",
    "sheet": "ログイン【チェック仕様】",
    "cells": "D9:S9"
  },
  "confidence": 0.96
}
```

此时结果属于：

```text
Silver Label
```

不是最终 Gold Label。

---

## 步骤 3：LLM 从历史测试式样书提取测试意图

让 LLM 将每条历史用例、每个测试步骤或每个测试行，转换为统一结构。

```json
{
  "test_intent_id": "TCI-LOGIN-REQUIRED",
  "category": "異常系",
  "condition": "USERID または PASSWORD が未入力",
  "action": "LOGINボタン押下",
  "expected_results": [
    "エラーメッセージ表示",
    "認証を実行しない",
    "ログイン履歴を登録しない"
  ],
  "evidence": {
    "file": "単体テスト仕様書.xlsx",
    "sheet": "単体テスト",
    "cells": "A25:N25"
  },
  "confidence": 0.94
}
```

---

## 步骤 4：LLM 建立“设计事实 → 测试意图”关联

这是数据集最有价值的一层。

```json
{
  "trace_id": "TRACE-LOGIN-001",
  "design_fact_id": "GF-LOGIN-001",
  "test_intent_id": "TCI-LOGIN-REQUIRED",
  "relation": "covered",
  "coverage_level": "full",
  "confidence": 0.93,
  "reason": "测试条件包含 USERID/PASSWORD 未输入，预期结果包含错误消息显示。"
}
```

对于不确定的情况，不能让 LLM 强行关联：

```json
{
  "relation": "needs_review",
  "confidence": 0.51,
  "reason": "测试用例只描述认证失败，无法判断是否覆盖删除用户条件。"
}
```

---

## 步骤 5：做规则校验和交叉校验

LLM 语义判断之外，要加入确定性规则。

例如：

```text
- 测试 ID 是否唯一；
- 每条测试用例是否有测试对象；
- 是否有前提条件、操作步骤、预期结果；
- 预期结果是否为空；
- 设计书 ID 是否存在；
- 引用的 Sheet / 单元格是否存在；
- 错误消息编号是否在来源文档中出现；
- DB 表名、字段名是否存在于设计书；
- Excel 模板列是否符合要求。
```

这些不应该浪费 LLM Token，也不能只依赖 LLM 判断。

---

# 5. 人工不必全检，而要“分层抽检”

## 需要人工优先审核的样本

以下样本应进入人工审核队列：

| 条件 | 原因 |
|---|---|
| LLM 置信度低 | 语义关系不清楚 |
| 来源与目标文档冲突 | 可能是版本差异、设计遗漏或实现缺陷 |
| Agent 生成内容无来源证据 | 高风险幻觉 |
| 涉及 DB 更新/删除/事务回滚 | 错误影响较大 |
| 涉及权限、认证、个人信息 | 高风险业务 |
| 涉及外部接口、异步、批处理 | 语义与执行顺序复杂 |
| 边界值、状态迁移、复杂条件组合 | 最容易漏测 |
| 新模板、新项目、新业务领域 | 缺乏历史可靠性 |
| Agent 新版本首次出现的失败模式 | 用于建立新 Gold Case |
| LLM Judge 与规则评测结论冲突 | 需要裁决 |

---

## 对高置信度样本做统计抽样

例如对如下高置信度 Silver 数据：

```text
LLM confidence ≥ 0.90
且规则校验全部通过
且没有来源/目标冲突
且不属于高风险业务
```

可以只人工抽检一部分。

一个实用的初始比例：

| 数据类型 | 建议人工审核比例 |
|---|---:|
| 高风险、冲突、低置信度数据 | 100% |
| 新项目 / 新模板首批数据 | 30%～50% |
| 常规中置信度数据 | 10%～20% |
| 高置信度且长期稳定的数据 | 3%～10% |
| 固定回归 Benchmark | 100% |

重点不是固定比例，而是持续计算：

```text
LLM 自动标注准确率
= 人工确认正确的抽检项 / 抽检项总数
```

如果某个文档类型、某种功能、某个 Agent 版本的抽检准确率下降，就自动提高该分组的人工抽检比例。

---

# 6. 不要用“同一个 LLM”同时生成和裁判

这是非常重要的风险点。

如果：

```text
模型 A 生成测试用例
        ↓
模型 A 自己判定自己生成得好不好
```

容易产生偏差，模型会偏向自己的表达和推理模式。

推荐至少采用三层判断：

```text
第一层：确定性规则
    - Schema、字段、ID、Excel 格式、来源引用、DB 表名等

第二层：语义 Judge
    - 判断等价、覆盖、矛盾、幻觉
    - 尽可能使用不同模型、不同 Prompt 或不同版本

第三层：人工抽检
    - 校准 Judge
    - 处理冲突、低置信度和高风险案例
```

更具体的组合可以是：

```text
生成 Agent：
    模型 A

语义 Judge：
    模型 B
    或同一模型但不同版本 / 独立 Prompt / 无生成过程上下文

人工：
    对高风险和抽样样本做最终裁定
```

---

# 7. 语义评测不要“全量两两比较”

如果有：

```text
黄金测试意图：1,000 条
Agent 生成测试意图：1,200 条
```

直接做 1,000 × 1,200 次 LLM 判断，成本很高，也没有必要。

推荐两阶段匹配：

```text
第一阶段：Embedding 候选召回
    对每条 Agent 用例，从黄金测试意图中找 Top-K（如 K=3 或 5）

第二阶段：LLM Judge 精判
    只判断最相关候选是否：
    equivalent / covers / partially_covers / different
```

流程如下：

```text
Agent Test Intent
    ↓
Embedding 检索 Top-5 黄金 Test Intent
    ↓
LLM Judge 语义匹配
    ↓
选择最佳匹配关系
    ↓
计算 Precision / Recall / Coverage
```

这样既能处理等价表达，也能控制调用成本。

---

# 8. 建议的评测指标组合

## 设计理解层

```text
Fact Precision
Fact Recall
Fact F1
```

## 测试覆盖层

```text
Design Coverage
Test Intent Recall
Boundary Coverage
Exception Coverage
DB Coverage
Permission Coverage
State Transition Coverage
```

## 生成质量层

```text
Test Case Precision
Case Completeness Rate
Traceability Rate
Hallucination Rate
Contradiction Rate
Duplicate Case Rate
```

## 交付质量层

```text
Excel Validation Pass Rate
Template Compliance Rate
Test ID Uniqueness Rate
Required Field Fill Rate
```

## 人工效果层

```text
Human Review Pass Rate
Human Review Edit Rate
Average Review Score
Average Review Time per Case
```

其中，最能体现实际业务价值的通常是：

```text
人工 Review 修改率下降
+
关键设计规则覆盖率上升
+
幻觉率下降
+
人工处理时间减少
```

---

# 9. Langfuse 中建议保存的评测证据

每次生成和评测都应保留：

```text
- 原始设计书文件 ID、版本和 Sheet/单元格位置；
- 原始测试式样书文件 ID、版本和 Sheet/单元格位置；
- 文档抽取结果；
- LLM 抽取的 Design Facts；
- LLM 抽取的 Test Intents；
- Design Fact → Test Intent 追溯关系；
- 语义 Judge 的输入、输出、理由、置信度；
- 规则校验结果；
- 人工审核结果和修改内容；
- Agent、Prompt、模型、Excel 模板版本；
- 最终生成的 Excel 文件地址。
```

每个 Score 最好带 Metadata，例如：

```json
{
  "score_name": "semantic_coverage",
  "value": 0.91,
  "metadata": {
    "judge_model": "judge-model-v2",
    "judge_prompt_version": "semantic-match-v3",
    "dataset_tier": "silver",
    "document_type": "login_screen",
    "risk_level": "medium"
  }
}
```

这样以后看到某个分数下降，才能追溯：

```text
是新 Prompt 导致？
是新模型导致？
是某类 Excel 模板解析失败？
是状态迁移类测试变差？
还是某项目的历史文档质量本身较差？
```

---

# 10. 最终建议的落地模式

建议采用下面这个模式：

```text
历史设计书 + 历史测试式样书
        ↓
程序抽取 Excel 内容与坐标
        ↓
LLM 自动提取事实、测试意图、追溯关系
        ↓
规则校验
        ↓
LLM Judge 判断语义等价、覆盖、冲突和幻觉
        ↓
形成 Silver Dataset
        ↓
按风险、置信度、冲突情况进行人工抽检
        ↓
人工确认样本升级为 Gold Dataset
        ↓
固定 Gold Dataset 用于 Langfuse 回归评测
        ↓
人工修正结果回流，持续扩充 Gold Dataset
```

结论是：

- **是的，大规模建集必须依赖 LLM 自动生成和自动语义标注。**
- **是的，等价描述、包含关系、部分覆盖关系，适合使用 LLM Judge 判断。**
- 但不能完全放弃人工：人工的角色应从“手工造所有数据”，转为“抽检、裁决冲突、校准模型、维护关键金标集”。  
- 最终应同时拥有：
  - 大规模、低成本的 **Silver Dataset**；
  - 小规模、高可信的 **Gold Dataset**；
  - 专门用于上线门禁的 **Critical Gold Set**。



---

**User** _(2026-08-24 14:32:22)_

**image.png**

![image.png](https://sws-gpt-cn-api.sys.sws.com/api/files/chat-attachments/file/h6a61fTs5mp_CKcBwQOgixSulC9fPrgw_k3h1iaaXk0/01M0S429CRA3T5B0JJZNTQA819.png)

下面是langfuse的构造数据界面，如何使用

---

**Bot** _(2026-08-24 14:33:13)_

图 1（1枚目）是 Langfuse Dataset `test` 的空数据集页面。这里提供了四种加入 Dataset Item 的方式：

1. **Upload CSV**：批量导入，适合先把历史式样书自动抽取成评测数据后一次导入；
2. **Add Manually**：手工新增单条数据，适合先验证 Schema、做少量 Gold Case；
3. **Add via Code**：通过 SDK 自动写入，适合后续“LLM 自动标注 + 人工抽检”的批处理流水线；
4. **Select Observations**：从线上 Trace/Observation 中挑选真实失败案例，加入回归集。

对于你现在的“**内部设计/详细设计 → 单体测试式样书**”Agent，推荐按下面方式使用。

---

# 一、先确定 Dataset Item 的单位

不要将一整个 Excel 文件作为一条 Item。

建议把一个功能拆成多个独立的评测单元，例如 BTG00000 登录画面可以拆成：

| Dataset Item ID | 测试对象/设计对象 | 主要评测内容 |
|---|---|---|
| `BTG00000-INIT-001` | 初期显示 | INFORMATION 的取得条件、有效日期、删除标志、排序 |
| `BTG00000-LOGIN-001` | 登录输入校验 | USERID/PASSWORD 必填 |
| `BTG00000-LOGIN-002` | 登录认证 | 用户不存在、已删除、密码错误 |
| `BTG00000-LOGIN-003` | 登录成功 | LDAP 认证成功后的处理、画面迁移 |
| `BTG00000-HISTORY-001` | 登录履历 | 成功时的履历写入、字段映射 |
| `BTG00000-EXIT-001` | EXIT 按钮 | 画面关闭处理 |
| `BTG00000-FIELD-001` | 画面字段 | USERID 长度、PASSWORD 长度、必填、掩码显示 |

这样能精确知道 Agent 在哪里失败：

```text
是没有理解输入校验？
还是漏掉了 DB 更新？
还是没有生成边界值测试？
还是生成了没有设计依据的错误消息？
```

---

# 二、推荐在 Langfuse 中保存的 Item 结构

Langfuse 的 Dataset Item 可理解为：

```text
Input
    → Agent 的输入，即设计书内容

Expected Output
    → 评测依据，即标准化后的黄金规则、测试意图或目标结果

Metadata
    → 项目、功能、文件、版本、数据可信度、风险等级等辅助信息
```

对于“设计书转测试式样书”的场景，建议不要把 `expected_output` 仅设置为一份完整 Excel 的文本，而应放入**结构化黄金信息**。

---

## 1. Input：设计书片段

例如 `BTG00000-LOGIN-001`：

```json
{
  "task": "generate_unit_test_spec",
  "function_id": "BTG00000",
  "function_name": "ログイン画面",
  "test_level": "unit",
  "design_sections": [
    {
      "source_file": "(BTG00000)BTS_内部仕様_ログイン_ログイン.xlsx",
      "sheet": "ログイン【チェック仕様】",
      "cells": "D9:S9",
      "content": "USERIDまたはPASSWORDが入力されていない場合、エラーメッセージを表示する。"
    }
  ],
  "constraints": {
    "language": "ja",
    "output_format": "unit_test_spec_json",
    "require_design_evidence": true
  }
}
```

这里保留：

- 来源文件；
- Sheet 名；
- 单元格范围；
- 原始设计文本；
- 测试层级；
- 输出格式要求。

这样未来出现问题时，可以追溯到原始式样书的具体位置。

---

## 2. Expected Output：不是“唯一答案”，而是“必须满足的语义要求”

对于必填校验，建议写成：

```json
{
  "required_design_facts": [
    {
      "fact_id": "GF-LOGIN-001",
      "type": "validation",
      "assertion_level": "must",
      "condition": "USERID または PASSWORD が未入力",
      "expected_behavior": "エラーメッセージを表示し、ログイン処理を実行しない"
    },
    {
      "fact_id": "GF-LOGIN-002",
      "type": "message",
      "assertion_level": "must",
      "expected_message": "ユーザID、パスワードを入力してください。"
    },
    {
      "fact_id": "GF-LOGIN-003",
      "type": "negative_side_effect",
      "assertion_level": "must",
      "expected_behavior": "ログイン履歴を登録しない"
    }
  ],
  "required_test_intents": [
    {
      "intent_id": "TCI-LOGIN-REQUIRED-USERID",
      "category": "異常系",
      "condition": "USERID が未入力、PASSWORD は有効値",
      "action": "LOGINボタン押下",
      "expected_results": [
        "必須エラーメッセージが表示される",
        "LDAP認証を実行しない",
        "ログイン履歴を登録しない"
      ]
    },
    {
      "intent_id": "TCI-LOGIN-REQUIRED-PASSWORD",
      "category": "異常系",
      "condition": "PASSWORD が未入力、USERID は有効値",
      "action": "LOGINボタン押下",
      "expected_results": [
        "必須エラーメッセージが表示される",
        "LDAP認証を実行しない",
        "ログイン履歴を登録しない"
      ]
    }
  ],
  "prohibited_assumptions": [
    "アカウントロック機能",
    "CAPTCHA認証",
    "設計書に存在しないDBテーブルまたはエラーメッセージ"
  ]
}
```

这种设计支持语义评测：

```text
历史文档的措辞 ≠ Agent 的措辞
但只要覆盖相同设计事实和测试意图
→ 可以判定为正确或更完整
```

---

## 3. Metadata：用于后续筛选、分组和统计

```json
{
  "case_id": "BTG00000-LOGIN-001",
  "project": "BTG",
  "system": "BTS",
  "module": "認証",
  "feature": "ログイン",
  "risk_level": "high",
  "difficulty": "medium",
  "has_db_update": true,
  "has_external_auth": true,
  "has_validation": true,
  "has_state_transition": false,
  "source_design_version": "v1",
  "dataset_tier": "silver",
  "review_status": "pending",
  "template_version": "UT_SPEC_V1"
}
```

后续在 Langfuse 里就可以按如下维度分析：

```text
- 登录/认证类功能的覆盖率是否下降？
- 包含 DB 更新的测试用例是否容易漏？
- 包含 LDAP、API 等外部调用时，幻觉率是否升高？
- 高风险功能的人工 Review 通过率是多少？
- Silver 数据和 Gold 数据上的分数是否存在差异？
```

---

# 三、图 1（1枚目）中四种方式分别怎么用？

## 方式 1：先用 `Add Manually` 创建 3～5 条样例

建议第一步点击图 1（1枚目）中的 **Add Manually**。

目的不是手工录入所有数据，而是验证：

- Input JSON 格式是否适合 Agent；
- Expected Output 的 Gold Fact 是否足够；
- Metadata 是否满足后续筛选；
- 你的评测程序是否能正确读取 Dataset Item；
- 语义 Judge 是否能正确评分。

建议先创建以下 5 条：

```text
BTG00000-LOGIN-001：必填校验
BTG00000-LOGIN-002：用户不存在
BTG00000-LOGIN-003：删除用户
BTG00000-LOGIN-004：密码错误
BTG00000-HISTORY-001：登录成功写入履历
```

这 5 条可作为最小 Smoke Test Set。

---

## 方式 2：用 `Upload CSV` 导入批量 Silver Dataset

当你已用程序 + LLM 从历史 Excel 中抽取了大量数据后，点击图 1（1枚目）的 **Upload CSV**。

CSV 建议使用 UTF-8 with BOM 编码。常见列可以设计为：

```text
input,expected_output,metadata
```

示例：

```csv
input,expected_output,metadata
"{""task"":""generate_unit_test_spec"",""function_id"":""BTG00000"",""design_sections"":[{""sheet"":""ログイン【チェック仕様】"",""cells"":""D9:S9"",""content"":""USERIDまたはPASSWORDが入力されていない場合、エラーメッセージを表示する。""}]}","{""required_design_facts"":[{""fact_id"":""GF-LOGIN-001"",""condition"":""USERID または PASSWORD が未入力"",""expected_behavior"":""エラー表示し、ログイン処理を実行しない""}],""required_test_intents"":[{""intent_id"":""TCI-LOGIN-REQUIRED"",""category"":""異常系""}]}","{""case_id"":""BTG00000-LOGIN-001"",""project"":""BTG"",""dataset_tier"":""silver"",""review_status"":""pending"",""risk_level"":""high""}"
```

上传时重点确认：

1. `input` 被识别为输入；
2. `expected_output` 被识别为预期输出；
3. `metadata` 被识别为元数据；
4. JSON 中的双引号没有因 CSV 转义而损坏；
5. 不要在 CSV 中保存数据库密码、个人信息、真实账号等敏感数据；
6. 每一行是一个功能单元或测试意图，不是一整份式样书。

> 如果导入界面对列名有固定要求，以该界面的字段映射预览为准；不同 Langfuse 版本的 CSV 映射细节可能略有不同。

---

## 方式 3：生产化后用 `Add via Code`

这是最适合你“LLM 自动生成 + 人工抽检”的主方式。

典型流程：

```text
历史内部设计 Excel
    ↓
Python 读取 Sheet、单元格、合并单元格
    ↓
LLM 抽取 Design Fact
    ↓
LLM 抽取历史测试用例 / 测试意图
    ↓
LLM 建立语义追溯关系
    ↓
规则校验
    ↓
写入 Langfuse Dataset
```

概念代码如下：

```python
from langfuse import Langfuse

langfuse = Langfuse()

dataset_name = "design-to-unit-test-v1"

langfuse.create_dataset_item(
    dataset_name=dataset_name,
    input={
        "task": "generate_unit_test_spec",
        "function_id": "BTG00000",
        "function_name": "ログイン画面",
        "design_sections": [
            {
                "source_file": "(BTG00000)BTS_内部仕様_ログイン_ログイン.xlsx",
                "sheet": "ログイン【チェック仕様】",
                "cells": "D9:S9",
                "content": "USERIDまたはPASSWORDが入力されていない場合、エラーメッセージを表示する。"
            }
        ]
    },
    expected_output={
        "required_design_facts": [
            {
                "fact_id": "GF-LOGIN-001",
                "assertion_level": "must",
                "condition": "USERID または PASSWORD が未入力",
                "expected_behavior": "エラー表示し、ログイン処理を実行しない"
            }
        ],
        "required_test_intents": [
            {
                "intent_id": "TCI-LOGIN-REQUIRED",
                "category": "異常系",
                "viewpoint": "必須チェック"
            }
        ]
    },
    metadata={
        "case_id": "BTG00000-LOGIN-001",
        "project": "BTG",
        "dataset_tier": "silver",
        "review_status": "pending",
        "risk_level": "high"
    }
)
```

> SDK 的方法参数会随你当前安装的 Langfuse SDK 版本变化；实施时以该版本 Python SDK 的 Dataset Item API 为准。核心结构是 `dataset_name + input + expected_output + metadata`。

---

## 方式 4：用 `Select Observations` 把线上失败案例加入回归集

这是 Dataset 最有价值的长期使用方式。

当 Agent 上线后，Langfuse Trace 中会保存：

```text
用户输入设计书
→ Agent 提取规则
→ Agent 生成测试观点
→ Agent 生成测试用例
→ Excel 输出
→ 人工审核反馈
```

如果发现以下情况：

```text
- 漏掉 PASSWORD 最大长度 20 的边界值；
- 将 USERID 最大长度 10 错写为 20；
- 错误消息编号不一致；
- 密码错误时仍生成“写入登录履历”；
- 编造设计书中不存在的账户锁定逻辑；
```

可从 Observation 列表中选中这条真实失败记录，再使用 **Select Observations** 加入 Dataset。

然后为它补充或更新：

```text
- expected_output；
- 失败原因；
- 正确的设计证据；
- 人工确认结果；
- 风险等级；
- 回归标签。
```

这会形成：

```text
线上失败案例
    ↓
回归 Dataset
    ↓
下一个 Agent 版本必须通过
```

这就是评测集“可演进”的关键。

---

# 四、建议你在 Langfuse 中建立三套 Dataset

不要将所有数据放入一个 Dataset。

## 1. `design-to-test-silver`

用于大规模自动导入。

```text
来源：LLM 自动抽取 + 规则校验
规模：可以数千到数万条
用途：观察趋势、发现薄弱领域、初步比较模型
人工审核：抽检
```

Metadata：

```json
{
  "dataset_tier": "silver",
  "review_status": "pending"
}
```

---

## 2. `design-to-test-gold`

用于人工确认后的高可信数据。

```text
来源：人工抽检、修改、确认
规模：几十到数百个高质量功能单元
用途：主要离线评测集
人工审核：已完成
```

Metadata：

```json
{
  "dataset_tier": "gold",
  "review_status": "approved"
}
```

---

## 3. `design-to-test-critical-regression`

用于发布门禁。

```text
来源：
- 历史严重缺陷；
- 高风险认证、权限、审批、DB 更新功能；
- 线上真实失败案例；
- 人工发现的典型幻觉案例。

规模：20～100 个重点样本
用途：每次改 Prompt、模型、Agent 前必须运行
人工审核：100%
```

例如 BTG00000 登录功能很适合进入这一类：

```text
- 未输入账号或密码；
- 用户不存在；
- 删除用户；
- 密码错误；
- LDAP 失败；
- 登录成功；
- 登录履历写入；
- 登录失败不写入履历；
- EXIT；
- USERID / PASSWORD 长度边界；
- 密码掩码显示。
```

---

# 五、Dataset 导入后如何真正运行评测？

Dataset 只保存“题目”和“参考标准”，本身不会自动评测 Agent。

后续应在 **Experiments** 中执行：

```text
Dataset Item
    ↓
调用你的设计书 → 测试式样书 Agent
    ↓
得到 Agent Output
    ↓
规则评测
    + LLM 语义 Judge
    + Excel 模板校验
    ↓
将 Score 写入 Langfuse
```

一个 Item 的评测逻辑大致如下：

```text
1. Agent 是否提取了所有 required_design_facts？
2. Agent 是否生成 required_test_intents？
3. Agent 测试是否能引用设计书中的 Sheet/单元格证据？
4. Agent 是否生成了设计书中没有的字段、表、消息或规则？
5. Agent 输出是否包含测试 ID、前提条件、步骤、预期结果、确认方法？
6. 输出 Excel 是否符合模板？
7. Agent 表述与黄金用例是否语义等价、覆盖或部分覆盖？
```

建议写入以下 Score：

```text
design_fact_recall
test_intent_recall
design_coverage
semantic_coverage
hallucination_rate
contradiction_rate
traceability_rate
case_completeness_rate
excel_validation_pass
human_review_score
```

---

# 六、你现在最适合执行的顺序

```text
第 1 步：
在图 1（1枚目）点击 Add Manually，
先手工录入 BTG00000 的 3～5 条 Smoke Test Item。

第 2 步：
让 Agent 跑一次，确认 Input / Output Schema 是否合理。

第 3 步：
写评测程序，先实现：
- 设计事实覆盖；
- 测试意图覆盖；
- 来源证据校验；
- Excel 格式校验。

第 4 步：
加入 LLM Judge，处理“等价表述”“包含关系”“部分覆盖”等语义问题。

第 5 步：
用历史式样书批量抽取 Silver Dataset，
通过 Upload CSV 或 Add via Code 导入。

第 6 步：
对低置信度、冲突、高风险样本进行人工审核，
升级为 Gold Dataset。

第 7 步：
将线上失败 Observation 选择并加入 Critical Regression Dataset。
```

最开始不要追求一次导入几千份式样书。先用 BTG00000 登录功能建立 5～10 条高质量样例，跑通：

```text
数据格式
→ Agent 输出
→ 规则评分
→ LLM Judge
→ Langfuse Experiment
→ 人工 Review
```

这个链路验证正确后，再进行历史文档的批量自动建集。



---

**User** _(2026-08-24 14:35:58)_

**image.png**

![image.png](https://sws-gpt-cn-api.sys.sws.com/api/files/chat-attachments/file/h6a61fTs5mp_CKcBwQOgixSulC9fPrgw_k3h1iaaXk0/01M0S496MEES13Y2FH92SBPZZM.png)

 这个不支持代理吗

---

**Bot** _(2026-08-24 14:36:28)_

图 1（1枚目）这个页面是 **Langfuse 的 LLM Connection 配置**，用于让 Langfuse 自身调用 Azure OpenAI，例如运行：

- LLM-as-a-Judge 评测；
- 托管 Evaluator；
- Dataset / Experiment 中需要由 Langfuse 发起的模型调用。

它**不是 Agent 的网络代理或运行时代理配置页面**。

---

## 如果你说的是“网络代理（HTTP/HTTPS Proxy）”

这个界面**没有按单个 LLM Connection 配置代理地址的字段**。通常需要在部署 Langfuse 的服务端/容器层配置出站代理：

```text
Langfuse Server
   ↓ 通过 HTTPS_PROXY
企业 HTTP/HTTPS 代理
   ↓
Azure OpenAI
```

例如 Docker Compose 中可配置：

```yaml
services:
  langfuse-web:
    environment:
      HTTPS_PROXY: "http://proxy.company.local:8080"
      HTTP_PROXY: "http://proxy.company.local:8080"
      NO_PROXY: "localhost,127.0.0.1,postgres,redis"

  langfuse-worker:
    environment:
      HTTPS_PROXY: "http://proxy.company.local:8080"
      HTTP_PROXY: "http://proxy.company.local:8080"
      NO_PROXY: "localhost,127.0.0.1,postgres,redis"
```

建议同时配置大小写两套变量，以兼容不同运行环境：

```yaml
HTTPS_PROXY: "http://proxy.company.local:8080"
HTTP_PROXY: "http://proxy.company.local:8080"
NO_PROXY: "localhost,127.0.0.1,postgres,redis"

https_proxy: "http://proxy.company.local:8080"
http_proxy: "http://proxy.company.local:8080"
no_proxy: "localhost,127.0.0.1,postgres,redis"
```

### 注意

- 如果 Langfuse 是 Docker 部署，通常至少要给负责实际模型请求的 Web/Worker 服务配置代理。
- `NO_PROXY` 要包含 Langfuse 内部依赖，例如 PostgreSQL、Redis、ClickHouse、MinIO/S3 等内部服务名；否则内部服务通信也可能被错误地送往企业代理。
- 配置后重启对应容器。
- 代理认证信息不要直接硬编码到 Git 仓库中的 `docker-compose.yml`；应使用 Secret、环境变量文件或部署平台的密钥管理功能。

---

## 如果你说的是“Azure API Management / 企业 AI 网关代理”

这种场景可以尝试把网关地址填到图 1（1枚目）的 **API Base URL**，前提是该网关对 Langfuse 暴露了与 Azure OpenAI 兼容的接口。

页面提示的标准 Azure 格式是：

```text
https://{instanceName}.openai.azure.com/openai/deployments
```

如果你们使用 Azure API Management（APIM）作为网关，可能类似：

```text
https://{apim-name}.azure-api.net/openai/deployments
```

但它是否可用取决于 APIM 的策略是否正确转发：

```text
Langfuse 请求路径
  ↓
APIM 网关路径
  ↓
Azure OpenAI 的 deployments / chat completions 路径
```

如果 APIM 使用订阅密钥，还可以在图 1（1枚目）底部的 **Extra Headers** 增加网关要求的 Header，例如：

```text
Header name:  Ocp-Apim-Subscription-Key
Header value: <你的 APIM Subscription Key>
```

不过具体 Header 名称取决于你们 APIM 的安全策略；有些网关会要求：

```text
Authorization: Bearer ...
```

有些则要求：

```text
Ocp-Apim-Subscription-Key: ...
```

而 Azure OpenAI 原生通常使用：

```text
api-key: <Azure OpenAI API Key>
```

---

## 图 1（1枚目）各字段建议怎么填写

| 字段 | 建议填写内容 |
|---|---|
| Provider | `azure` |
| Provider name | 自定义连接名，例如 `azure-openai-prod`、`azure-openai-eval` |
| API Key | Azure OpenAI Key，或网关要求的 Key |
| API Base URL | Azure OpenAI 或 APIM 的兼容 Base URL |
| Custom models | **Azure Deployment Name**，不是模型原始名称 |
| Extra Headers | 只有企业网关/APIM要求额外认证 Header 时才填写 |

例如：

```text
Provider name:
azure-openai-eval

API Base URL:
https://my-aoai.openai.azure.com/openai/deployments

Custom model:
gpt-4o-eval-prod
```

这里的 `gpt-4o-eval-prod` 应是 Azure Portal 中创建的 **Deployment name**，而不一定是模型名 `gpt-4o`。

---

## 如果你说的是“Agent 代理”

若你说的“代理”是 **Agent**，答案是：

> 这个页面不负责运行、配置或编排 Agent；它只负责配置 Langfuse 可调用的 LLM 连接。

你的 LangChain Deep Agent 仍然运行在自己的应用服务中：

```text
你的 Python / Java 服务
  └── LangChain Deep Agent
       └── Azure OpenAI / 企业网关
```

Langfuse 则承担：

```text
Langfuse
  ├── Trace：记录 Agent 推理、工具调用、模型调用
  ├── Dataset：保存设计书输入与期望输出
  ├── Experiment：批量运行评测
  ├── Score：记录覆盖率、幻觉率、人工评分
  └── LLM Connection：供 Langfuse 的 Judge/Evaluator 调用 Azure 模型
```

因此，这个 Connection 不会自动让你的 Deep Agent “走代理”或“改用 Azure”。你的 Agent 代码中的 `AzureChatOpenAI`、企业 HTTP 代理或 APIM 地址，仍需在 Agent 服务自身的配置中设置。

---

## 推荐的部署分工

```text
Agent 服务
  ├── LangChain / Deep Agent
  ├── AzureChatOpenAI
  ├── HTTPS_PROXY 或 APIM 网关
  └── Langfuse SDK：上报 Trace、Observation、Score

Langfuse 服务
  ├── Dataset
  ├── Experiment
  ├── LLM Connection（Azure）
  ├── LLM Judge / Evaluator
  └── HTTPS_PROXY 或 APIM 网关
```

也就是说：

- **Agent 调模型的网络代理**：在 Agent 服务配置；
- **Langfuse Judge 调模型的网络代理**：在 Langfuse 部署环境配置；
- **Azure APIM 网关地址**：通常可以尝试配置在图 1（1枚目）的 `API Base URL`；
- **这个页面本身没有单独的 HTTP Proxy 输入框。**



---

**User** _(2026-08-24 14:37:53)_

**image.png**

![image.png](https://sws-gpt-cn-api.sys.sws.com/api/files/chat-attachments/file/h6a61fTs5mp_CKcBwQOgixSulC9fPrgw_k3h1iaaXk0/01M0S4C9009EG10JNSZHCJ5JC0.png)

 这个如何设置，是设置你的agent的接口，用来调用生成结果的吗

---

**Bot** _(2026-08-24 14:38:31)_

是的，但更准确地说：

> 图 1（1枚目）右侧的 **via Webhook**，配置的是一个“实验执行器（Experiment Runner）接口”，不是直接填写 Azure OpenAI 或单纯 Agent Chat 接口。

对于你的场景——`LangChain Deep Agent + LLM` 根据设计书生成测试式样书——应选择 **via Webhook**。

Langfuse 会在你点击运行 Experiment 时，调用你提供的 Webhook；你的服务收到 Dataset 和实验配置后，执行 Agent，再把每个 Dataset Item 的结果和评分回传到 Langfuse。

---

## 两种方式的区别

| 方式 | 图 1（1枚目）左侧：via User Interface | 图 1（1枚目）右侧：via Webhook |
|---|---|---|
| 面向对象 | Prompt / 模型配置 | 自定义 Agent、业务服务、复杂工作流 |
| 是否需要编码 | 不需要或很少 | 需要 |
| 能否运行 LangChain Deep Agent | 通常不适合 | **适合** |
| 能否处理 Excel、RAG、数据库、工具调用 | 能力有限 | **可以** |
| 能否执行你的测试式样书生成逻辑 | 不适合复杂流程 | **可以** |
| 结果如何进入 Langfuse | Langfuse 自动执行 | 你的服务将结果回传 Langfuse |
| 推荐用途 | 比较 Prompt、模型参数 | 比较 Agent 版本、RAG、工具、评测逻辑 |

---

# 你的场景应选 `via Webhook`

整体调用关系如下：

```text
Langfuse Dataset
    ↓
在 Langfuse UI 中点击 Run Experiment
    ↓
Langfuse 调用你的 Webhook
    ↓
你的 Experiment Runner 服务
    ├── 读取 Dataset Item
    ├── 调用 LangChain Deep Agent
    ├── 生成单体测试式样书 JSON
    ├── 生成 Excel
    ├── 执行规则评测
    ├── 调用 LLM Judge 做语义评测
    └── 将结果、Score、Trace 回传 Langfuse
    ↓
Langfuse Experiment 页面展示每条运行结果和汇总分数
```

因此，答案是：

> **可以理解为：Langfuse 通过 Webhook 调用你的 Agent 服务，让你的 Agent 对 Dataset 中每一条数据生成结果。**  
> 但 Webhook 更适合作为“实验任务入口”，内部再调用你已有的 Agent 接口或 Agent 代码。

---

# 不建议直接暴露 Agent 主接口

假设你已有一个普通 Agent 接口：

```text
POST /api/generate-unit-test-spec
```

输入：

```json
{
  "design_text": "...",
  "function_id": "BTG00000"
}
```

输出：

```json
{
  "test_cases": [],
  "excel_url": "..."
}
```

不要简单把这个接口直接当 Langfuse Webhook。

更推荐加一个专门的实验接口：

```text
POST /api/langfuse/experiment/run
```

结构如下：

```text
Langfuse Webhook
    ↓
/api/langfuse/experiment/run
    ↓
Experiment Runner
    ↓
/api/generate-unit-test-spec
    ↓
LangChain Deep Agent
```

这样可以将“正常业务调用”和“批量实验调用”隔离。

| 接口 | 主要职责 |
|---|---|
| `/api/generate-unit-test-spec` | 正常业务：对一份设计书生成测试式样书 |
| `/api/langfuse/experiment/run` | 实验编排：读取 Dataset、循环调用 Agent、评测并回传结果 |
| `/api/langfuse/experiment/callback` | 可选：异步完成后向 Langfuse 回传结果 |

---

# Webhook 服务实际要做什么？

Langfuse 调用 Webhook 后，你的服务需要完成以下任务。

```text
1. 接收实验 ID、Dataset 信息、运行配置
2. 获取本次实验包含的 Dataset Item
3. 对每个 Item 调用 Agent
4. 得到结构化测试用例 JSON
5. 生成 Excel（如需要）
6. 执行确定性评测
7. 执行 LLM 语义 Judge
8. 记录 Trace、Generation、Score
9. 将实验结果关联回 Langfuse
```

---

# 针对你的“设计书 → 测试式样书”Agent 的处理流程

假设 Dataset Item 是：

```json
{
  "input": {
    "task": "generate_unit_test_spec",
    "function_id": "BTG00000",
    "function_name": "ログイン画面",
    "design_sections": [
      {
        "sheet": "ログイン【チェック仕様】",
        "cells": "D9:S12",
        "content": "USERIDまたはPASSWORDが入力されていない場合..."
      }
    ]
  },
  "expected_output": {
    "required_design_facts": [
      "USERID または PASSWORD の必須チェック",
      "ユーザー不存在チェック",
      "削除済みユーザー拒否",
      "パスワード不一致チェック"
    ],
    "required_test_intents": [
      "必須チェック異常系",
      "ユーザー不存在異常系",
      "削除済み異常系",
      "パスワード不一致異常系"
    ]
  }
}
```

你的 Webhook 内部处理：

```text
Dataset Item
    ↓
Deep Agent：
设计规则提取
    ↓
Deep Agent：
生成测试观点评价点
    ↓
Deep Agent：
生成测试用例 JSON
    ↓
Excel Renderer：
生成单体测试式样书
    ↓
Evaluator：
评测覆盖率、幻觉率、完整性、语义等价性
    ↓
写入 Langfuse Score
```

---

# 一个推荐的 Webhook 请求处理结构

以下是概念代码，重点是职责划分；具体 Webhook 请求字段以你当前 Langfuse UI 在配置时展示的 Schema 为准。

```python
from fastapi import FastAPI, Header, HTTPException
from pydantic import BaseModel

app = FastAPI()


class ExperimentWebhookRequest(BaseModel):
    experiment_id: str
    dataset_name: str
    dataset_run_id: str | None = None
    run_config: dict = {}
    dataset_items: list[dict] = []


@app.post("/api/langfuse/experiment/run")
async def run_langfuse_experiment(
    request: ExperimentWebhookRequest,
    x_experiment_secret: str | None = Header(default=None),
):
    # 1. 校验 Langfuse Webhook 的认证信息
    if x_experiment_secret != "your-secret":
        raise HTTPException(status_code=401, detail="Unauthorized")

    # 2. 尽快返回任务已接收状态。
    #    真正的批量运行建议交给 Celery、RQ、Sidekiq、Kafka 消费者等异步任务系统。
    create_async_experiment_job(
        experiment_id=request.experiment_id,
        dataset_name=request.dataset_name,
        dataset_run_id=request.dataset_run_id,
        dataset_items=request.dataset_items,
        run_config=request.run_config,
    )

    return {
        "accepted": True,
        "experiment_id": request.experiment_id
    }
```

后台任务的逻辑：

```python
def execute_experiment_for_item(dataset_item: dict, run_config: dict):
    # 1. 调用你的 Deep Agent
    agent_result = generate_unit_test_spec(
        design_input=dataset_item["input"],
        agent_version=run_config.get("agent_version", "v1")
    )

    # 2. 规则评测
    deterministic_scores = evaluate_with_rules(
        expected_output=dataset_item["expected_output"],
        agent_output=agent_result
    )

    # 3. LLM Judge 语义评测
    semantic_scores = evaluate_with_llm_judge(
        design_input=dataset_item["input"],
        expected_output=dataset_item["expected_output"],
        agent_output=agent_result
    )

    # 4. Excel 校验
    excel_scores = validate_generated_excel(
        excel_path=agent_result["excel_path"]
    )

    # 5. 写入 Langfuse Trace / Score / Experiment Result
    publish_result_to_langfuse(
        dataset_item=dataset_item,
        agent_output=agent_result,
        scores={
            **deterministic_scores,
            **semantic_scores,
            **excel_scores
        }
    )
```

---

# 回传给 Langfuse 的结果应包含什么？

对于每一条 Dataset Item，建议至少记录：

```json
{
  "agent_version": "deepagent-v0.3",
  "prompt_version": "test-spec-prompt-v5",
  "model": "azure-gpt-4o-prod",
  "output": {
    "test_cases": [],
    "open_issues": [],
    "excel_url": "..."
  },
  "scores": {
    "design_coverage": 0.92,
    "test_intent_recall": 0.89,
    "hallucination_rate": 0.03,
    "traceability_rate": 0.98,
    "case_completeness_rate": 1.0,
    "excel_validation_pass": true,
    "semantic_judge_score": 0.90
  }
}
```

对于你这个场景，尤其建议记录：

| Score | 说明 |
|---|---|
| `design_coverage` | 设计规则是否被测试覆盖 |
| `test_intent_recall` | 是否覆盖黄金测试意图 |
| `hallucination_rate` | 是否生成设计书中没有的字段、规则、消息或 DB 行为 |
| `traceability_rate` | 测试用例能否回溯到设计书证据 |
| `case_completeness_rate` | 是否有前提、数据、步骤、预期、确认方法 |
| `semantic_judge_score` | 与黄金测试意图的语义覆盖程度 |
| `excel_validation_pass` | Excel 格式是否符合测试式样书模板 |
| `human_review_score` | 人工抽检评分，可后续补写 |

---

# Webhook 配置时应注意的安全问题

## 1. Webhook 必须能被 Langfuse 访问

如果 Langfuse 部署在云端，而你的 Agent 在内网：

```text
Langfuse Cloud
    ✕ 无法直接访问
企业内网 Agent API
```

你需要以下之一：

- 将 Webhook 服务部署到可公开访问的 HTTPS 地址；
- 使用企业 VPN、专线、私有网络互联；
- 在企业网关/API Gateway 上暴露受控入口；
- 使用反向隧道方案；
- 自托管 Langfuse 到同一内网或 VPC。

---

## 2. 必须做认证

建议至少使用：

```text
Authorization: Bearer <Webhook Secret>
```

或自定义 Header：

```text
X-Langfuse-Experiment-Secret: <Secret>
```

服务端应验证：

- Secret 是否正确；
- 请求来源是否可信；
- 实验 ID、Dataset 名称是否允许；
- 是否允许调用当前 Agent 版本；
- 是否限制最大并发、最大 Dataset Item 数量。

---

## 3. 必须异步化

不要让 Webhook HTTP 请求一直等待几十分钟。

推荐：

```text
Langfuse 调用 Webhook
    ↓
Webhook 验证请求
    ↓
写入任务队列
    ↓
立即返回 200 / accepted
    ↓
后台 Worker 执行 Deep Agent 批量实验
    ↓
逐条或批量回传实验结果
```

因为设计书解析、Deep Agent、多次 LLM 调用、Excel 生成、LLM Judge 都可能耗时较长。

---

# 什么时候选择左侧 `via User Interface`？

仅在以下情况优先使用左侧：

```text
- 你只是比较 Prompt A 与 Prompt B；
- 输入是简单文本；
- 调用链只有一次 LLM 调用；
- 不依赖 Excel 解析；
- 不依赖 RAG；
- 不依赖 Deep Agent；
- 不需要调用内部工具、数据库、文件系统或业务 API。
```

例如：

```text
“将一段需求文本总结为测试观点”
```

可以用 UI 直接对比不同 Prompt。

但你的完整目标是：

```text
内部设计 Excel
    ↓
规则提取
    ↓
多步骤 Agent
    ↓
测试用例 JSON
    ↓
Excel 测试式样书
    ↓
规则评测 + LLM Judge
```

所以应使用：

```text
via Webhook
```

---

## 最终结论

是的，右侧 **via Webhook** 可以理解为配置“调用你的 Agent 的实验入口”。

但推荐架构不是：

```text
Langfuse → 直接调用 Agent Chat API
```

而是：

```text
Langfuse → Experiment Webhook
         → 你的实验执行服务
         → 你的 Deep Agent
         → 评测服务
         → 回传结果到 Langfuse
```

这样才能支持 Dataset 批量运行、多个 Agent/Prompt/模型版本对比、LLM Judge、人工抽检以及可持续回归评测。



---

**User** _(2026-08-24 14:51:00)_

**image.png**

![image.png](https://sws-gpt-cn-api.sys.sws.com/api/files/chat-attachments/file/h6a61fTs5mp_CKcBwQOgixSulC9fPrgw_k3h1iaaXk0/01M0S54F452W5W48RF3SSDNCJP.png)

 地址是我本地用fastapi建的，为什么会报这个错误

---

**Bot** _(2026-08-24 14:52:01)_

图 1（1枚目）里显示的橙色文字：

```text
This endpoint uses plain HTTP...
Use HTTPS for production endpoints.
```

这是**安全警告，不一定是实际报错**。它表示你现在配置的是：

```text
http://172.19.71.207:8000/eval/unit_test_eval
```

即明文 HTTP；如果开启签名或使用自定义认证 Header，相关内容会以明文方式传输。

如果点击 **Set up** 或后续运行 Experiment 时失败，最常见原因不是 FastAPI 代码本身，而是：

> **Langfuse 服务端无法访问你本地的 `172.19.71.207:8000`。**

---

## 先理解调用方向

不是浏览器调用 FastAPI，而是：

```text
Langfuse 后端服务 / Worker
        ↓ HTTP POST
你的 FastAPI 地址
http://172.19.71.207:8000/eval/unit_test_eval
```

所以即使你在本机浏览器中能访问：

```text
http://172.19.71.207:8000/docs
```

也不代表 Langfuse 容器、Langfuse 云服务或远程服务器能够访问这个地址。

---

# 最常见的 4 个原因

## 1. FastAPI 只监听了 `127.0.0.1`

如果你使用：

```bash
uvicorn main:app --port 8000
```

FastAPI 可能只监听本机回环地址：

```text
127.0.0.1:8000
```

这种情况下：

- 本机可访问；
- 其他机器、Docker 容器、Langfuse 服务无法访问。

应改为：

```bash
uvicorn main:app --host 0.0.0.0 --port 8000
```

然后确认监听地址：

```bash
ss -lntp | grep 8000
```

期望看到类似：

```text
0.0.0.0:8000
```

而不是：

```text
127.0.0.1:8000
```

---

## 2. Langfuse 部署在 Docker 中，容器无法访问宿主机地址

如果：

```text
FastAPI：运行在宿主机
Langfuse：运行在 Docker 容器
```

那么 Langfuse 容器不一定能访问宿主机的 `172.19.71.207`。

应进入实际负责调用 Webhook 的 Langfuse 容器中测试：

```bash
docker ps
```

找到 Langfuse 的 Web / Worker 容器后执行：

```bash
docker exec -it <langfuse-container-name> sh
```

在容器内执行：

```bash
curl -v \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{}' \
  http://172.19.71.207:8000/eval/unit_test_eval
```

判断方法：

| 结果 | 含义 |
|---|---|
| `Connection refused` | FastAPI 没有监听外部地址，或端口没有开放 |
| `Connection timed out` | 网络路由、防火墙、Docker 网络不可达 |
| `404 Not Found` | 网络可达，但 URL 路径不对 |
| `405 Method Not Allowed` | 网络可达，但 FastAPI 没有定义 POST 方法 |
| `422 Unprocessable Entity` | 网络可达，FastAPI 收到了请求，但请求体不符合接口 Schema |
| `200 OK` / `202 Accepted` | 网络和接口基本正常 |

> 即使返回 `422`，也说明“Langfuse 到 FastAPI 的网络连接”已经没问题；此时只需调整请求体 Schema。

---

## 3. Langfuse 是云端部署，无法访问内网 IP

如果你使用的是 Langfuse Cloud，或者 Langfuse 部署在另一台服务器/VPC 中，那么：

```text
172.19.71.207
```

通常是内网地址，云端 Langfuse 无法直接访问。

调用路径会失败：

```text
Langfuse Cloud
    ✕
172.19.71.207:8000
```

此时可选方案是：

### 方案 A：Langfuse 和 FastAPI 部署到同一个内网/VPC

推荐用于企业内部系统：

```text
企业内网 / 同一个 VPC
├── Langfuse
└── FastAPI Experiment Runner
```

### 方案 B：通过 API Gateway / Nginx 暴露 HTTPS 地址

例如：

```text
https://agent-eval.company.example/eval/unit_test_eval
```

并限制访问来源、增加认证和签名校验。

### 方案 C：开发环境使用 Tunnel

例如通过企业允许的反向代理、VPN Tunnel、Cloudflare Tunnel、ngrok 等方式暴露临时 HTTPS 地址。

```text
https://xxxxx.example-tunnel.com/eval/unit_test_eval
        ↓
http://172.19.71.207:8000/eval/unit_test_eval
```

生产环境不建议依赖临时 Tunnel。

---

## 4. 防火墙或安全组未开放 8000 端口

即使 FastAPI 使用了：

```bash
uvicorn main:app --host 0.0.0.0 --port 8000
```

也可能被以下规则阻断：

- Windows Defender Firewall；
- Linux `firewalld` / `ufw` / `iptables`；
- 云服务器 Security Group；
- 企业网络 ACL；
- Kubernetes NetworkPolicy；
- Docker 网络规则。

建议先在运行 FastAPI 的机器上测试：

```bash
curl -v http://127.0.0.1:8000/docs
curl -v http://172.19.71.207:8000/docs
```

然后从 Langfuse 所在服务器或容器测试：

```bash
curl -v http://172.19.71.207:8000/docs
```

---

# FastAPI 最小接口应满足的条件

你的路由必须接受 **POST**，而不是只定义 `GET`。

```python
from typing import Any
from fastapi import FastAPI, Request

app = FastAPI()


@app.post("/eval/unit_test_eval")
async def unit_test_eval(request: Request) -> dict[str, Any]:
    payload = await request.json()

    # 先记录实际收到的 Langfuse 请求结构，便于确认字段
    print(payload)

    # 后续应改为：
    # 1. 校验请求签名或认证 Header
    # 2. 创建异步实验任务
    # 3. 调用 Deep Agent
    # 4. 回传实验结果到 Langfuse

    return {
        "accepted": True,
        "message": "Experiment trigger received"
    }
```

启动：

```bash
uvicorn main:app --host 0.0.0.0 --port 8000
```

先不要一开始就定义复杂的 Pydantic 请求模型。因为你需要先通过日志确认 Langfuse 实际发送的 payload 字段结构；否则很容易直接得到 `422`。

---

# 图 1（1枚目）中当前配置的注意点

## URL

你当前填写：

```text
http://172.19.71.207:8000/eval/unit_test_eval
```

确认你的 FastAPI 中存在完全一致的路由：

```python
@app.post("/eval/unit_test_eval")
```

以下任何细微差异都会导致 `404`：

```text
/eval/unit_test_eval/
/eval/unit-test-eval
/eval/unit_test
/eval/unit_test_eval/{id}
```

---

## Default config

当前是：

```json
{}
```

可以保留为空。

后续可写入 Agent 实验配置，让每次 Experiment 可切换模型、Prompt 和版本，例如：

```json
{
  "agent_version": "deepagent-v1",
  "prompt_version": "unit-test-spec-v3",
  "model_deployment": "gpt-4o-eval",
  "enable_llm_judge": true,
  "max_concurrency": 3
}
```

你的 FastAPI 接口收到该配置后，根据这些参数运行不同版本的 Agent。

---

## Sign requests

当前是关闭状态：

```text
Requests will be sent without an x-langfuse-signature header.
```

开发调试阶段可以先关闭。

但生产环境建议：

1. 使用 HTTPS；
2. 开启 **Sign requests**；
3. 在 FastAPI 中验证 `x-langfuse-signature`；
4. 或者在 Advanced Options 中增加自定义认证 Header。

不要在 HTTP 明文环境下传递生产密钥、Bearer Token 或签名 Secret。

---

# 建议你按这个顺序排查

```text
1. FastAPI 是否已启动？
   → http://127.0.0.1:8000/docs

2. FastAPI 是否监听 0.0.0.0？
   → uvicorn main:app --host 0.0.0.0 --port 8000

3. 宿主机是否可通过 172.19.71.207 访问？
   → curl http://172.19.71.207:8000/docs

4. Langfuse 所在容器/服务器是否可访问？
   → docker exec ... curl http://172.19.71.207:8000/docs

5. POST 路由是否存在？
   → curl -X POST http://172.19.71.207:8000/eval/unit_test_eval -d '{}'

6. 查看 FastAPI 日志中是否收到了请求。

7. 网络联通后，再处理 Langfuse 实际 Payload、签名校验和结果回传。
```

最可能的情况是：

```text
FastAPI 只绑定到 127.0.0.1
```

或者：

```text
Langfuse 运行在 Docker/云端，无法访问 172.19.71.207 这个内网地址。
```



---

**User** _(2026-08-24 15:43:46)_

How to Create Scores
There are five ways to add scores:

LLM-as-a-Judge: Set up automated evaluators that score traces based on custom criteria (e.g. hallucination, tone, relevance). These can return numeric or categorical scores plus reasoning, and can run on live production traces or on experiment results.
Code evaluators: Run custom Python or TypeScript evaluators in Langfuse for deterministic checks such as exact match, JSON validation, or custom business rules.
Scores via UI: Team members manually score traces, observations, or sessions directly in the Langfuse UI. Requires a score config to be set up first.
Annotation Queues: Set up structured review workflows where reviewers work through batches of traces.
Scores via API/SDK: Programmatically add scores from your application code. This is the way to go for user feedback (thumbs up/down, star ratings), guardrail results, or custom evaluation pipelines. 这几个如何使用，langfuse

---

**Bot** _(2026-08-24 15:45:00)_

## 内容总结

这段说明介绍了 Langfuse 中新增评分（Scores）的五种方式：

1. **LLM-as-a-Judge**：用大模型按自定义标准自动评分，例如幻觉、语气、相关性；可输出数值/分类评分及理由，适用于线上 Trace 或 Experiment 结果。
2. **Code evaluators**：运行 Python 或 TypeScript 评测代码，适合精确匹配、JSON 校验、业务规则等确定性检查。
3. **Scores via UI**：团队成员在 Langfuse UI 中直接对 Trace、Observation 或 Session 人工打分；需先配置 Score Config。
4. **Annotation Queues**：建立批量人工审核工作流，让审核人员按队列处理 Trace。
5. **Scores via API/SDK**：业务应用通过 API/SDK 写入评分，适合用户点赞/点踩、星级、护栏检测结果和自定义评测流水线。

---

## 中文翻译

# 如何创建评分

添加评分有五种方式：

- **LLM 作为评委（LLM-as-a-Judge）**：设置自动评测器，根据自定义标准对 Trace 进行评分，例如幻觉、语气和相关性。这些评测器可以返回数值或分类评分及其理由，并且可以运行在线上生产 Trace 或实验结果上。
- **代码评测器（Code evaluators）**：在 Langfuse 中运行自定义 Python 或 TypeScript 评测器，执行精确匹配、JSON 校验或自定义业务规则等确定性检查。
- **通过 UI 评分（Scores via UI）**：团队成员直接在 Langfuse UI 中对 Trace、Observation 或 Session 进行人工评分。需要先设置 Score Config。
- **标注队列（Annotation Queues）**：建立结构化人工审核工作流，让审核人员批量审核 Trace。
- **通过 API/SDK 评分（Scores via API/SDK）**：从应用程序代码中以编程方式添加评分。适用于用户点赞/点踩、星级评分、护栏结果或自定义评测流水线。

---

# 你的场景应该如何使用？

你的目标是：

```text
内部设计书 / 详细设计书
        ↓
LangChain Deep Agent
        ↓
生成单体测试式样书 JSON / Excel
        ↓
评测覆盖率、幻觉、格式、语义质量
        ↓
Langfuse Dataset / Experiment / Score
```

因此，不建议只使用其中某一种评分方式，而是建议采用组合方案：

```text
Code Evaluator
    + LLM-as-a-Judge
    + Annotation Queue
    + API / SDK 写入业务评分
```

各方式的推荐分工如下。

| 评分方式 | 是否推荐 | 主要用途 |
|---|---:|---|
| LLM-as-a-Judge | 强烈推荐 | 语义等价、测试覆盖、幻觉、预期结果质量 |
| Code Evaluators | 强烈推荐 | JSON/Excel 校验、字段完整性、ID 唯一性、规则匹配 |
| Scores via UI | 推荐 | 少量人工复核、快速确认 |
| Annotation Queues | 强烈推荐 | 大量历史式样书的人工抽检与 Gold 数据建设 |
| Scores via API/SDK | 强烈推荐 | 从 Deep Agent、Excel 校验器、业务系统回传评分 |

---

# 一、先定义 Score Config

无论使用 UI、Annotation Queue、LLM Judge 还是 SDK，建议先统一定义评分名称和类型。

对于“设计书 → 单体测试式样书”Agent，可以创建以下 Score Config。

## 数值型 Score

| Score 名称 | 取值范围 | 含义 | 趋势 |
|---|---:|---|---|
| `design_coverage` | 0～1 | 设计规则被测试用例覆盖的比例 | 越高越好 |
| `test_intent_recall` | 0～1 | 黄金测试意图被覆盖的比例 | 越高越好 |
| `semantic_coverage` | 0～1 | Agent 输出和黄金测试意图的语义覆盖程度 | 越高越好 |
| `traceability_rate` | 0～1 | 测试用例可追溯到设计书证据的比例 | 越高越好 |
| `case_completeness_rate` | 0～1 | 用例是否具备前提、数据、步骤、预期、确认方法 | 越高越好 |
| `hallucination_rate` | 0～1 | 无设计依据的字段、规则、消息、DB 行为比例 | 越低越好 |
| `contradiction_rate` | 0～1 | 与设计书明确冲突的比例 | 越低越好 |
| `duplicate_case_rate` | 0～1 | 重复测试用例比例 | 越低越好 |
| `human_review_score` | 1～5 | 人工总体质量评分 | 越高越好 |
| `review_edit_rate` | 0～1 | 被人工修改的用例比例 | 越低越好 |

## 分类型 Score

| Score 名称 | 可选值 | 用途 |
|---|---|---|
| `excel_validation` | `pass` / `fail` | Excel 模板、Sheet、字段和下拉框校验 |
| `human_review_status` | `approved` / `needs_fix` / `rejected` | 人工审核结论 |
| `semantic_relation` | `equivalent` / `covers` / `partial` / `different` / `contradicts` | Agent 用例和黄金用例的语义关系 |
| `risk_level` | `low` / `medium` / `high` / `critical` | 数据集和审核优先级 |
| `review_action` | `accept` / `modify` / `add_case` / `remove_case` | 审核人员采取的动作 |

---

# 二、LLM-as-a-Judge：处理“语义等价”和“是否有依据”

这是你最需要的一种自动评分方式。

因为历史测试式样书和 Agent 生成的测试式样书，不会逐字一致。例如：

```text
历史用例：
ユーザIDが未入力の場合、エラーメッセージを表示する。
```

```text
Agent 输出：
USERID を空欄にして LOGIN ボタンを押下した場合、
「ユーザID、パスワードを入力してください。」を表示し、
LDAP認証およびログイン履歴登録を実施しない。
```

虽然文字不同，但第二种可能覆盖更完整。

---

## LLM Judge 应判断什么？

建议为 Agent 输出建立以下 Judge。

### 1. 设计依据 Judge

判断 Agent 生成内容是否有设计书依据。

```text
输入：
- 原始设计书片段
- Agent 生成的测试用例
- Agent 声明的来源证据

输出：
- 是否有设计依据
- 是否包含无依据的内容
- 是否与设计矛盾
- 理由
```

输出示例：

```json
{
  "supported_by_design": true,
  "unsupported_claims": [],
  "contradictions": [],
  "score": 1.0,
  "reason": "设计书明确规定 USERID 或 PASSWORD 未输入时应显示错误消息；Agent 用例与该规则一致。"
}
```

对应 Score：

```text
traceability_rate
hallucination_rate
contradiction_rate
```

---

### 2. 测试覆盖 Judge

判断生成的测试用例是否覆盖黄金测试意图。

```text
输入：
- 黄金 Test Intent
- Agent 生成的 Test Case
- 来源设计事实

输出：
- equivalent
- covers
- partially_covers
- different
- contradicts
```

示例：

```json
{
  "semantic_relation": "covers",
  "coverage_score": 0.95,
  "missing_requirements": [],
  "reason": "Agent 覆盖了必填校验、错误消息、不执行认证和不写入登录履历；覆盖范围高于黄金用例。"
}
```

对应 Score：

```text
semantic_coverage
test_intent_recall
semantic_relation
```

---

### 3. 预期结果质量 Judge

判断测试用例是否可执行、可验证。

重点检查：

```text
- 是否有明确前提条件；
- 是否有可执行操作步骤；
- 是否有具体测试数据；
- 是否明确期望消息；
- 是否明确 DB、日志、接口或画面确认方式；
- 是否只写了“正常显示”“正常结束”这类模糊结果。
```

输出：

```json
{
  "score": 0.86,
  "missing_fields": [
    "未明确指定登录履历的确认 SQL"
  ],
  "reason": "测试步骤和画面预期明确，但数据库验证方法不够具体。"
}
```

对应 Score：

```text
case_completeness_rate
```

---

## LLM Judge 的使用建议

你已经在配置 Azure LLM Connection，因此可让 Langfuse 使用该 Azure Deployment 运行 Judge。

推荐原则：

```text
Agent 生成模型
        ≠
Judge 模型
```

例如：

```text
生成：
azure-gpt-4o-agent

Judge：
azure-gpt-4o-eval
```

即使底层模型相同，也建议使用：

- 独立的 Prompt；
- 独立的模型 Deployment；
- 不向 Judge 暴露 Agent 的内部思考过程；
- 要求 Judge 给出设计证据、理由和结构化输出。

不要让 Agent 自己给自己打分。

---

# 三、Code Evaluators：处理确定性校验

Code Evaluator 适合不需要“语义理解”的项目。

例如：

```text
- 测试 ID 是否重复；
- 是否存在必填列；
- JSON 是否符合 Schema；
- Excel 是否生成成功；
- Sheet 是否齐全；
- Excel 是否有“単体テスト仕様書”“トレーサビリティ”“未決事項”；
- 每条测试用例是否有设计书 ID；
- 每条测试用例是否有前提条件、操作步骤、预期结果；
- 引用的设计书 Sheet / 单元格是否存在；
- 错误消息编号是否存在于设计书；
- DB 表名和字段名是否存在于设计书；
- 输出文件是否损坏；
- 是否有空白测试 ID；
- 是否有重复测试用例。
```

---

## 推荐的 Code Evaluator 示例

### 1. 测试用例完整性评测

```python
def evaluate_case_completeness(test_cases: list[dict]) -> dict:
    required_fields = [
        "test_id",
        "design_ids",
        "target",
        "category",
        "viewpoint",
        "preconditions",
        "test_data",
        "steps",
        "expected_results",
        "verification_method",
    ]

    complete_count = 0
    incomplete_cases = []

    for case in test_cases:
        missing = []

        for field in required_fields:
            value = case.get(field)

            if value is None or value == "" or value == []:
                missing.append(field)

        if missing:
            incomplete_cases.append({
                "test_id": case.get("test_id"),
                "missing_fields": missing,
            })
        else:
            complete_count += 1

    total = len(test_cases)
    score = complete_count / total if total > 0 else 0.0

    return {
        "name": "case_completeness_rate",
        "value": score,
        "comment": {
            "total_cases": total,
            "complete_cases": complete_count,
            "incomplete_cases": incomplete_cases,
        },
    }
```

---

### 2. Excel 输出校验

```python
import openpyxl


def validate_excel_spec(file_path: str) -> dict:
    required_sheets = {
        "単体テスト仕様書",
        "トレーサビリティ",
        "未決事項",
    }

    workbook = openpyxl.load_workbook(
        file_path,
        read_only=True,
        data_only=False,
    )

    actual_sheets = set(workbook.sheetnames)
    missing_sheets = sorted(required_sheets - actual_sheets)

    return {
        "name": "excel_validation",
        "value": "pass" if not missing_sheets else "fail",
        "comment": {
            "actual_sheets": sorted(actual_sheets),
            "missing_sheets": missing_sheets,
        },
    }
```

---

### 3. 设计证据校验

例如 Agent 声称其测试用例引用了：

```text
ログイン【チェック仕様】!D9:S9
```

代码可检查：

```text
- Sheet 是否存在；
- 单元格区域是否存在；
- 引用的消息 ID、字段名或规则文本是否确实存在；
- 用例中的 DB 表名是否出现在设计资料中。
```

这类校验可以帮助发现“编造证据”的情况。

---

# 四、Scores via UI：少量人工快速审核

UI 手工打分适合：

```text
- 对单条 Agent 输出快速确认；
- 演示阶段；
- 少量严重问题；
- 项目负责人快速查看；
- 人工复核 LLM Judge 的结论；
- 给典型失败案例打标签。
```

例如，在某条 Trace 中，你发现 Agent 漏掉了：

```text
PASSWORD 最大长度 20
```

可以直接在 UI 中设置：

```text
Score Config:
human_review_status

Value:
needs_fix
```

再添加：

```text
Score Config:
human_review_score

Value:
2
```

并在评论中写：

```text
遗漏 PASSWORD 长度 20 的上界及上界外测试；
未验证异常时不写入登录履历。
```

缺点是无法高效审核几千份历史式样书，因此不应作为主建集方法。

---

# 五、Annotation Queues：大规模人工抽检的核心

对于你的场景，Annotation Queue 很重要。

它可以让人工审核人员按队列处理“需要审核的 Trace”，而不是手动在大量 Trace 中找问题。

建议建立多个审核队列。

| Annotation Queue | 入队规则 | 审核目标 |
|---|---|---|
| `high-risk-auth-review` | 登录、权限、LDAP、密码、个人信息 | 认证和权限相关高风险逻辑 |
| `low-confidence-review` | LLM Judge 置信度低于 0.75 | 语义难判、可能误标 |
| `hallucination-review` | `hallucination_rate > 0` | 无设计依据内容 |
| `coverage-gap-review` | `design_coverage < 0.85` | 设计规则漏测 |
| `excel-failure-review` | `excel_validation = fail` | Excel 模板或输出问题 |
| `new-agent-regression-review` | 新 Prompt/模型首次运行结果 | 新版本质量确认 |
| `production-failure-review` | 线上用户点踩、人工发现失败 | 将真实失败沉淀为回归集 |

---

## 审核人员应如何打分？

审核人不需要重新写测试式样书，只需完成结构化判断。

建议人工审核表单字段如下：

| 字段 | 类型 | 示例 |
|---|---|---|
| `human_review_status` | 分类 | `approved` / `needs_fix` / `rejected` |
| `human_review_score` | 数值 | `1～5` |
| `missing_design_fact` | 多选/文本 | `PASSWORD最大长度` |
| `issue_type` | 分类 | `missing_case` / `hallucination` / `wrong_expectation` |
| `review_action` | 分类 | `accept` / `modify` / `add_case` / `remove_case` |
| `review_comment` | 文本 | “缺少密码长度为 21 的异常测试。” |

人工结果应回流到 Dataset：

```text
人工发现漏测
    ↓
补充 required_design_facts / required_test_intents
    ↓
该样本升级为 Gold Dataset
    ↓
后续版本必须通过
```

---

# 六、Scores via API/SDK：由你的 Agent 服务主动写入评分

这是你的 Deep Agent 实验流程中最实用的方式。

你的服务已经知道：

```text
- 生成了多少条测试用例；
- Excel 是否生成成功；
- Schema 是否有效；
- 规则覆盖率是多少；
- 哪些测试用例没有设计依据；
- LLM Judge 分数是多少；
- 生成耗时、Token、成本是多少。
```

这些结果可以通过 SDK 写回 Langfuse。

概念代码如下：

```python
from langfuse import Langfuse

langfuse = Langfuse()

trace_id = "当前Agent运行对应的Trace ID"

langfuse.score(
    trace_id=trace_id,
    name="design_coverage",
    value=0.92,
    comment="23 个 must 设计事实中，21 个已被有效测试用例覆盖。",
)

langfuse.score(
    trace_id=trace_id,
    name="hallucination_rate",
    value=0.03,
    comment="共生成 34 条测试用例；其中 1 条引用了设计书中未定义的规则。",
)

langfuse.score(
    trace_id=trace_id,
    name="excel_validation",
    value="pass",
    comment="Sheet、列、下拉框和测试 ID 校验通过。",
)
```

> 不同 Langfuse Python SDK 版本的 Score API 方法名称可能略有差异。实施时应以你当前安装 SDK 的 `score` / `score.create` 相关接口为准。

---

## 推荐：按层级写 Score

你的 Agent 应有一条总 Trace，例如：

```text
Trace: generate_unit_test_spec
```

下面包含多个 Observation：

```text
Trace: generate_unit_test_spec
├── Observation: excel_extract
├── Observation: design_fact_extract
├── Observation: test_viewpoint_generate
├── Observation: test_case_generate
├── Observation: excel_render
├── Observation: deterministic_evaluation
└── Observation: llm_judge
```

建议评分层级如下：

| 评分对象 | 建议 Score |
|---|---|
| `design_fact_extract` Observation | `fact_precision`、`fact_recall` |
| `test_case_generate` Observation | `semantic_coverage`、`hallucination_rate` |
| `excel_render` Observation | `excel_validation` |
| 整个 Trace | `design_coverage`、`human_review_score`、`human_review_status` |
| Experiment Item Run | `test_intent_recall`、`traceability_rate`、`case_completeness_rate` |

这样可以回答：

```text
总体分数低，是因为：
- Excel 解析有问题？
- 设计规则提取漏了？
- Deep Agent 没有生成边界值测试？
- LLM Judge 发现幻觉？
- 还是 Excel 模板输出不合规？
```

---

# 七、推荐给你的完整组合

```text
历史内部设计书 + 历史测试式样书
        ↓
LLM 自动抽取 Design Fact / Test Intent
        ↓
Code Evaluator
├── JSON Schema
├── Excel 模板
├── ID 唯一性
├── 字段完整性
└── 来源证据存在性
        ↓
LLM-as-a-Judge
├── 语义等价
├── 覆盖关系
├── 幻觉判断
├── 设计矛盾判断
└── 预期结果质量
        ↓
API / SDK 写入 Score
        ↓
低分、冲突、高风险样本
        ↓
Annotation Queue 人工抽检
        ↓
人工确认后升级为 Gold Dataset
        ↓
固定 Gold Dataset 用于后续 Experiment 回归测试
```

---

# 八、最小可行落地顺序

建议先做以下四项，而不是一次配置全部能力：

```text
第 1 步：
创建 Score Config：
design_coverage、hallucination_rate、
case_completeness_rate、excel_validation、
human_review_status。

第 2 步：
实现 Code Evaluator：
JSON、Excel、必填字段、测试 ID、来源引用校验。

第 3 步：
配置一个 LLM Judge：
判断“生成测试用例是否覆盖设计事实，是否存在无依据内容”。

第 4 步：
将低分、冲突和认证类高风险样本放入 Annotation Queue，
由测试人员进行人工确认。
```

之后再逐步加入：

```text
- 多个 Judge；
- 生产用户反馈；
- 自动回归门禁；
- 历史文档 Silver Dataset 批量导入；
- Gold Dataset 建设；
- Prompt / Agent / Model A-B 实验对比。
```

最关键的分工可以记为：

```text
Code Evaluator：结构、格式、确定性规则
LLM Judge：语义等价、覆盖、幻觉、质量
Annotation Queue：人工裁决与金标数据建设
API/SDK：将你自己的 Agent 评测结果写回 Langfuse
UI Score：少量人工快速标注和复核
```



---

**User** _(2026-08-24 15:56:02)_

**image.png**

![image.png](https://sws-gpt-cn-api.sys.sws.com/api/files/chat-attachments/file/h6a61fTs5mp_CKcBwQOgixSulC9fPrgw_k3h1iaaXk0/01M0S8VG26CC18XG5MWEQY2RES.png)

  default config 是用于传参给这个url地址的？

---

**Bot** _(2026-08-24 15:56:38)_

对。图 1（1枚目）中的 **Default config** 就是：

> 每次你在 Langfuse UI 触发这个 Remote Experiment 时，Langfuse 会把这段 JSON 配置一并发送给你填写的 Webhook URL。

它相当于给你的 Experiment Runner / FastAPI 接口传递“本次实验如何运行”的参数。

---

## 请求关系

假设你配置：

```text
URL:
https://agent-eval.company.example/eval/unit_test_eval
```

Default config 填写：

```json
{
  "agent_version": "deepagent-v1",
  "prompt_version": "unit-test-spec-v3",
  "model_deployment": "gpt-4o-eval",
  "enable_llm_judge": true,
  "max_concurrency": 3
}
```

当你从 Langfuse Dataset 页面运行 Experiment 时，Langfuse 会向你的 URL 发送：

```text
POST https://agent-eval.company.example/eval/unit_test_eval
```

请求中会包含两部分信息：

```text
Langfuse 自动附带的信息
    +
Default config 或本次运行时修改后的 config
```

概念上类似：

```json
{
  "dataset": {
    "id": "dataset-id",
    "name": "test"
  },
  "experiment": {
    "id": "experiment-id",
    "name": "unit-test-spec-eval"
  },
  "config": {
    "agent_version": "deepagent-v1",
    "prompt_version": "unit-test-spec-v3",
    "model_deployment": "gpt-4o-eval",
    "enable_llm_judge": true,
    "max_concurrency": 3
  }
}
```

实际字段名和层级以你当前 Langfuse 版本的 Webhook 文档及实际收到的请求为准，但语义就是：

```text
dataset 信息 + experiment 信息 + config
```

---

## Default config 的作用

它适合存放“同一份 Dataset 下，如何运行不同实验版本”的参数。

例如你要比较：

```text
Deep Agent v1
vs
Deep Agent v2
```

可以创建两次 Experiment。

### 实验 A

```json
{
  "agent_version": "deepagent-v1",
  "prompt_version": "unit-test-spec-v3",
  "model_deployment": "gpt-4o-agent-v1",
  "enable_llm_judge": true
}
```

### 实验 B

```json
{
  "agent_version": "deepagent-v2",
  "prompt_version": "unit-test-spec-v4",
  "model_deployment": "gpt-4o-agent-v2",
  "enable_llm_judge": true
}
```

你的 FastAPI 服务根据 `config` 选择对应的 Agent、Prompt 和模型。

---

## 对你的项目推荐填写什么？

针对“内部设计书 → 单体测试式样书”Agent，建议先填以下最小配置：

```json
{
  "agent_version": "deepagent-v1",
  "prompt_version": "unit-test-spec-v1",
  "model_deployment": "azure-gpt-4o",
  "output_format": "json",
  "generate_excel": true,
  "enable_rule_evaluator": true,
  "enable_llm_judge": true,
  "max_concurrency": 1
}
```

字段含义：

| 字段 | 含义 |
|---|---|
| `agent_version` | 当前运行的 Deep Agent 版本 |
| `prompt_version` | 当前测试式样书生成 Prompt 版本 |
| `model_deployment` | Azure OpenAI Deployment Name |
| `output_format` | 先输出结构化 JSON，还是直接输出 Excel |
| `generate_excel` | 是否同时生成 `.xlsx` 文件 |
| `enable_rule_evaluator` | 是否运行 JSON、字段、Excel、ID 等确定性校验 |
| `enable_llm_judge` | 是否调用 Judge 做语义覆盖和幻觉评测 |
| `max_concurrency` | 并发处理 Dataset Item 的数量；初期建议为 `1` |

---

## 不应该放进 Default config 的内容

不要放入：

```text
- Azure OpenAI API Key
- Langfuse Secret Key
- 数据库密码
- Bearer Token
- 企业代理账号密码
- 用户个人信息
- 固定设计书正文
- 固定测试用例正文
```

原因：

- Default config 是实验运行参数，不是密钥仓库；
- 在 HTTP 场景下尤其危险；
- 即使使用 HTTPS，也应将敏感配置放在服务端环境变量、Vault 或 Secret Manager 中。

正确做法：

```text
Default config:
model_deployment = "azure-gpt-4o"

服务端环境变量:
AZURE_OPENAI_API_KEY = "..."
AZURE_OPENAI_ENDPOINT = "..."
```

---

## FastAPI 中如何读取？

概念上，你的接口收到请求后读取 `config`：

```python
from typing import Any
from fastapi import FastAPI, Request

app = FastAPI()


@app.post("/eval/unit_test_eval")
async def run_unit_test_evaluation(request: Request) -> dict[str, Any]:
    payload = await request.json()

    # 不同 Langfuse 版本的实际字段层级可能不同。
    # 调试阶段先打印 payload，确认真实结构。
    print(payload)

    config = payload.get("config", {})

    agent_version = config.get("agent_version", "deepagent-v1")
    prompt_version = config.get("prompt_version", "unit-test-spec-v1")
    model_deployment = config.get("model_deployment", "azure-gpt-4o")
    enable_llm_judge = config.get("enable_llm_judge", True)

    # 后续将任务提交到异步队列：
    # create_experiment_job(
    #     payload=payload,
    #     agent_version=agent_version,
    #     prompt_version=prompt_version,
    #     model_deployment=model_deployment,
    #     enable_llm_judge=enable_llm_judge,
    # )

    return {
        "accepted": True,
        "agent_version": agent_version,
        "prompt_version": prompt_version,
        "model_deployment": model_deployment,
    }
```

---

## Default config 和 Dataset Item 的区别

这两个很容易混淆。

| 内容 | 应放在哪里 | 示例 |
|---|---|---|
| 某个功能的设计书内容 | Dataset Item `input` | 登录画面的检查规格 |
| 某功能必须覆盖的测试意图 | Dataset Item `expected_output` | 必填、用户不存在、密码错误 |
| 当前使用哪个 Agent | Default config | `deepagent-v2` |
| 当前使用哪个 Prompt | Default config | `unit-test-spec-v4` |
| 是否开启 LLM Judge | Default config | `true` |
| 当前 Dataset 名称、ID | Langfuse 自动发送 | `test`、`dataset-id` |
| Azure API Key | 服务端 Secret | 不应放入 Default config |
| Excel 模板版本 | Default config 或 Dataset metadata | `UT_TEMPLATE_V3` |

可以理解为：

```text
Dataset Item
    = 考题 + 参考标准

Default config
    = 本次考试使用哪位考生、哪套答题策略、哪些评测开关

Webhook URL
    = 执行考试的服务入口
```

---

## 一个实用的版本对比配置

例如以后要比较两套策略：

```json
{
  "agent_version": "deepagent-v2",
  "prompt_version": "unit-test-spec-v5",
  "model_deployment": "gpt-4o-agent-prod",
  "retrieval_strategy": "hybrid_rerank",
  "enable_requirement_extraction": true,
  "enable_test_viewpoint_review": true,
  "enable_rule_evaluator": true,
  "enable_llm_judge": true,
  "judge_model_deployment": "gpt-4o-judge",
  "generate_excel": true,
  "max_concurrency": 2
}
```

这样 Langfuse Experiment 的结果会带着这份配置，后续比较时可以明确知道：

```text
这个分数提升，
到底是 Agent 版本变了、
Prompt 变了、
模型变了、
检索策略变了，
还是 Judge 配置变了。
```



---

**User** _(2026-08-24 15:57:29)_

"dataset": {
    "id": "dataset-id",
    "name": "test"
  }, 这个不用配置自带的吗

---

**Bot** _(2026-08-24 15:57:45)_

对，**不用配置**。

我前面示例中的：

```json
"dataset": {
  "id": "dataset-id",
  "name": "test"
}
```

不是让你写进图 1（1枚目）的 **Default config**，而是 Langfuse 在触发 Remote Experiment 时，**自动发送给你的 Webhook 的上下文信息**。

---

## 你的 Default config 只需要写自定义运行参数

例如：

```json
{
  "agent_version": "deepagent-v1",
  "prompt_version": "unit-test-spec-v1",
  "model_deployment": "azure-gpt-4o",
  "generate_excel": true,
  "enable_rule_evaluator": true,
  "enable_llm_judge": true,
  "max_concurrency": 1
}
```

不需要写：

```json
{
  "dataset": {
    "id": "dataset-id",
    "name": "test"
  }
}
```

也不需要重复传：

```text
- Dataset 名称
- Dataset ID
- Experiment ID
- 当前运行 ID
```

这些应由 Langfuse 自动提供。

---

## 实际概念结构

当你在 Dataset `test` 页面启动实验时，Langfuse 会向 Webhook 发送类似信息：

```json
{
  "dataset": {
    "id": "Langfuse 自动生成的 Dataset ID",
    "name": "test"
  },
  "experiment": {
    "id": "Langfuse 自动生成的 Experiment ID"
  },
  "config": {
    "agent_version": "deepagent-v1",
    "prompt_version": "unit-test-spec-v1",
    "model_deployment": "azure-gpt-4o",
    "generate_excel": true,
    "enable_rule_evaluator": true,
    "enable_llm_judge": true,
    "max_concurrency": 1
  }
}
```

其中：

| 内容 | 来源 |
|---|---|
| `dataset.id` | Langfuse 自动生成并发送 |
| `dataset.name` | Langfuse 自动读取当前 Dataset，例如 `test` |
| `experiment.id` | Langfuse 自动生成并发送 |
| `config` | 你在 Default config 中填写，或运行实验时临时修改 |
| Dataset Item 的具体内容 | 你的服务依据 Dataset ID 从 Langfuse 查询，或按 Webhook 实际 payload 处理 |

---

## FastAPI 中应如何理解？

你的服务收到请求后：

```python
payload = await request.json()

dataset_info = payload.get("dataset", {})
dataset_id = dataset_info.get("id")
dataset_name = dataset_info.get("name")

config = payload.get("config", {})

agent_version = config.get("agent_version")
prompt_version = config.get("prompt_version")
```

然后：

```text
根据 dataset_id / dataset_name
    ↓
从 Langfuse 获取本次 Dataset Item
    ↓
根据 config 选择 Agent、Prompt、模型
    ↓
执行实验
    ↓
将结果和 Score 回传 Langfuse
```

---

## 重要提醒：以实际收到的 Payload 为准

我前面给出的 JSON 是**概念结构**，不是要保证每个字段名都和你当前 Langfuse 版本完全一致。

首次联通时，建议先让 FastAPI 打印原始请求：

```python
from typing import Any
from fastapi import FastAPI, Request

app = FastAPI()


@app.post("/eval/unit_test_eval")
async def unit_test_eval(request: Request) -> dict[str, Any]:
    payload = await request.json()

    print("Langfuse payload:")
    print(payload)

    return {
        "accepted": True
    }
```

然后在 Langfuse 中真正启动一次 Remote Experiment，查看 FastAPI 日志。你会看到当前版本实际传入的：

```text
- dataset ID
- dataset name
- experiment / run ID
- config
- 回调地址或结果写入所需信息（如有）
```

之后再根据真实字段写 Pydantic Schema 和实验执行逻辑即可。



---

**User** _(2026-08-24 16:00:18)_

如何回传，直接return就行呗

---

**Bot** _(2026-08-24 16:01:49)_

不是。对于图 1（1枚目）的 **Remote Experiment Trigger**：

> `return` 主要用于告诉 Langfuse“Webhook 已接收/已启动”。  
> **实验结果、Agent 输出和 Score 需要再通过 Langfuse API / SDK 写回。**

页面文案已经说明了这一点：

```text
Your service receives the selected dataset and run config,
executes the experiment,
and posts results back to Langfuse.
```

---

## 正确的调用方式

```text
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
```

---

# `return` 应返回什么？

Webhook 接口的同步返回建议只表达“任务接收成功”。

```python
from typing import Any
from fastapi import BackgroundTasks, FastAPI, Request

app = FastAPI()


@app.post("/eval/unit_test_eval")
async def start_remote_experiment(
    request: Request,
    background_tasks: BackgroundTasks,
) -> dict[str, Any]:
    payload = await request.json()

    # 保存 payload 中的 dataset / experiment / config 信息，
    # 后台任务执行时需要用它把结果关联回当前实验。
    background_tasks.add_task(
        execute_experiment,
        payload,
    )

    # 这里只是确认 Webhook 已收到任务。
    return {
        "accepted": True,
        "message": "Remote experiment has been queued."
    }
```

也可以明确返回 HTTP `202 Accepted`：

```python
from fastapi.responses import JSONResponse


@app.post("/eval/unit_test_eval")
async def start_remote_experiment(
    request: Request,
    background_tasks: BackgroundTasks,
) -> JSONResponse:
    payload = await request.json()

    background_tasks.add_task(execute_experiment, payload)

    return JSONResponse(
        status_code=202,
        content={
            "accepted": True,
            "message": "Experiment queued."
        },
    )
```

---

# 为什么不能直接 `return` 完整结果？

因为一轮 Dataset Experiment 通常包含多条 Item：

```text
Dataset
├── BTG00000-LOGIN-001
├── BTG00000-LOGIN-002
├── BTG00000-LOGIN-003
├── BTG00000-HISTORY-001
└── ...
```

每条 Item 又可能需要：

```text
设计书解析
→ Deep Agent
→ 多次 LLM 调用
→ Excel 生成
→ 规则校验
→ LLM Judge
```

可能执行几十秒到几十分钟。

如果直接在 Webhook 请求中等待：

```text
Langfuse HTTP 请求
    ↓
一直等待所有 Dataset Item 完成
    ↓
等待超时 / 连接中断 / Worker 被杀掉
```

而且即使你返回：

```json
{
  "output": "...",
  "scores": {
    "design_coverage": 0.92
  }
}
```

Langfuse 也不会自动将它识别为“每个 Dataset Item 的 Experiment Result”。

---

# 真正的“回传”是什么？

真正回传的是：你的后台任务通过 Langfuse SDK/API，给对应的 Dataset Item Run 写入：

```text
- Agent 输出；
- 测试用例 JSON；
- Excel 文件 URL；
- Trace；
- Generation；
- Score；
- 错误信息；
- Agent / Prompt / 模型版本；
- 运行耗时和成本。
```

概念结构如下：

```text
Dataset Item
    ↓
Experiment Item Run
    ├── input
    ├── output
    ├── metadata
    ├── trace
    └── scores
         ├── design_coverage
         ├── hallucination_rate
         ├── traceability_rate
         ├── semantic_coverage
         └── excel_validation
```

---

# 后台任务的伪代码

```python
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
```

其中最重要的是：

```python
post_result_to_langfuse(...)
```

它不是 HTTP Webhook 的 `return`，而是你的服务主动调用 Langfuse API / SDK。

---

# 回传的数据示例

假设 Agent 对登录必填校验生成了结果：

```json
{
  "test_cases": [
    {
      "test_id": "UT-LOGIN-001",
      "category": "異常系",
      "viewpoint": "USERID 必填检查",
      "expected_results": [
        "显示必填错误消息",
        "不执行 LDAP 认证",
        "不登记登录履历"
      ]
    }
  ],
  "excel_url": "https://storage.company.example/results/UT_BTG00000.xlsx",
  "open_issues": []
}
```

回传 Langfuse 时，建议附带评分：

```json
{
  "scores": {
    "design_coverage": 0.92,
    "test_intent_recall": 0.89,
    "semantic_coverage": 0.91,
    "traceability_rate": 1.0,
    "hallucination_rate": 0.0,
    "case_completeness_rate": 1.0,
    "excel_validation": "pass"
  }
}
```

这样 Langfuse 的 Experiment 页面才能展示：

```text
BTG00000-LOGIN-001
├── Agent 输出：测试用例 JSON
├── Excel：下载链接
├── design_coverage：0.92
├── hallucination_rate：0.00
├── semantic_coverage：0.91
└── excel_validation：pass
```

---

# 需要区分两个“返回”

| 动作 | 目的 | 方式 |
|---|---|---|
| Webhook 的 HTTP `return` | 告诉 Langfuse“请求已收到” | `return {"accepted": true}` |
| 写入 Experiment Result | 告诉 Langfuse“这条 Dataset Item 的执行结果和评分是什么” | Langfuse API / SDK |
| 上传 Excel | 保存生成的 `.xlsx` 文件 | 对象存储、文件服务或附件能力 |
| 写入 Score | 保存覆盖率、幻觉率、人工评分等 | Langfuse API / SDK |

---

# 开发阶段可以同步执行吗？

可以，但只适合很小的 Smoke Test，例如 1～3 条 Dataset Item。

```python
@app.post("/eval/unit_test_eval")
async def run_sync_experiment(request: Request) -> dict:
    payload = await request.json()

    result = execute_experiment_sync(payload)

    # 注意：
    # 即使这里返回 result，
    # 仍应在 execute_experiment_sync 内部调用 Langfuse API / SDK 回传。
    return {
        "accepted": True,
        "result_summary": result,
    }
```

即使同步跑完，也不要假设 Langfuse 会自动解析：

```python
return {
    "test_cases": [...],
    "scores": {...}
}
```

作为 Experiment Item Result。

---

# 推荐实现方式

生产环境建议：

```text
Webhook
  ↓
FastAPI
  ↓
任务队列
  ├── Celery
  ├── RQ
  ├── Dramatiq
  ├── Kafka / RabbitMQ Consumer
  └── 云任务服务
  ↓
Deep Agent Worker
  ↓
Langfuse SDK / API 回传结果
```

最小开发版本可以先使用：

```text
FastAPI BackgroundTasks
```

但批量 Dataset、大量 LLM 调用、Excel 生成或长任务，建议最终改为独立任务队列，避免服务重启后任务丢失。



---

**User** _(2026-08-24 16:19:43)_

post_result_to_langfuse 这个如何做，能给出具体代码示例吗

---

**Bot** _(2026-08-24 16:20:44)_

可以。对于 Remote Experiment，推荐不要自己拼一个不确定的 `post_result_to_langfuse()` REST 请求，而是使用 Langfuse Python SDK 的 **Dataset Run** 能力：

```text
获取 Dataset
    ↓
遍历 Dataset Item
    ↓
item.run(...)
    ↓
run.update(output=...)
run.score(...)
    ↓
结果自动关联到该 Dataset 的 Experiment Run
```

也就是说，`item.run()` 就是你“回传结果到 Langfuse”的关键入口。

---

# 1. 前置条件

环境变量：

```bash
export LANGFUSE_PUBLIC_KEY="pk-lf-..."
export LANGFUSE_SECRET_KEY="sk-lf-..."
export LANGFUSE_HOST="http://your-langfuse-host:3000"
```

如果是 Langfuse Cloud，则 `LANGFUSE_HOST` 通常不必设置；如果是自托管 Langfuse，则必须指向你的 Langfuse 地址。

安装依赖：

```bash
pip install "langfuse>=3,<4" fastapi uvicorn
```

> 以下代码按 Langfuse Python SDK 的 Dataset Run 风格编写。若你项目中使用的是较旧 SDK，`get_client()`、`item.run()`、`run.score()` 等方法名称可能略有不同；建议优先升级或以当前 SDK 文档为准。

---

# 2. 最核心的回传代码

下面这段就是你问的 `post_result_to_langfuse` 的核心实现。

```python
from typing import Any

from langfuse import get_client


langfuse = get_client()


def post_result_to_langfuse(
    dataset_item: Any,
    run_name: str,
    run_description: str,
    run_metadata: dict[str, Any],
    agent_output: dict[str, Any],
    scores: dict[str, float],
) -> None:
    """
    将某一个 Dataset Item 的 Agent 输出和评分写回 Langfuse。

    dataset_item:
        通过 langfuse.get_dataset(...).items 获取的 Dataset Item。

    run_name:
        本次 Experiment Run 的名称。
        同一轮实验的所有 Dataset Item 必须使用同一个 run_name。
    """

    with dataset_item.run(
        run_name=run_name,
        run_description=run_description,
        metadata=run_metadata,
    ) as run:
        # 1. 写入 Agent 的最终输出。
        # Langfuse 会将其显示为该 Dataset Item 的 Experiment Output。
        run.update(
            output=agent_output,
        )

        # 2. 写入数值型评分。
        # 建议预先在 Langfuse 中建立对应的 Score Config。
        for score_name, score_value in scores.items():
            run.score(
                name=score_name,
                value=score_value,
            )
```

例如：

```python
post_result_to_langfuse(
    dataset_item=item,
    run_name="unit-test-spec-deepagent-v1",
    run_description="BTG detailed design to unit test specification",
    run_metadata={
        "agent_version": "deepagent-v1",
        "prompt_version": "unit-test-spec-v1",
        "model_deployment": "azure-gpt-4o",
    },
    agent_output={
        "test_cases": [
            {
                "test_id": "UT-LOGIN-001",
                "category": "異常系",
                "viewpoint": "USERID 必填检查",
                "expected_results": [
                    "显示必填错误消息",
                    "不执行 LDAP 认证",
                    "不写入登录履历"
                ]
            }
        ],
        "excel_url": "https://storage.example.com/UT_BTG00000.xlsx",
        "open_issues": []
    },
    scores={
        "design_coverage": 0.92,
        "test_intent_recall": 0.89,
        "semantic_coverage": 0.91,
        "traceability_rate": 1.0,
        "hallucination_rate": 0.0,
        "case_completeness_rate": 1.0,
        "excel_validation_pass": 1.0,
    },
)
```

---

# 3. 可运行的 FastAPI 示例

下面是一个较完整的结构。

它做以下事情：

```text
Langfuse Remote Experiment
    ↓
调用 POST /eval/unit_test_eval
    ↓
FastAPI 接收 Dataset 名称、Run 名称、Config
    ↓
后台任务读取 Dataset
    ↓
循环执行 Deep Agent
    ↓
通过 dataset_item.run() 写回输出和 Score
```

```python
import json
import logging
from typing import Any

from fastapi import BackgroundTasks, FastAPI, HTTPException, Request
from fastapi.responses import JSONResponse
from langfuse import get_client

app = FastAPI()
logger = logging.getLogger(__name__)

langfuse = get_client()


def get_payload_value(
    payload: dict[str, Any],
    *keys: str,
    default: Any = None,
) -> Any:
    """
    兼容不同 Langfuse / Webhook 版本可能使用的字段名。

    例如：
    datasetName / dataset_name
    runName / run_name
    """
    for key in keys:
        value = payload.get(key)

        if value is not None:
            return value

    return default


def invoke_unit_test_deep_agent(
    design_input: dict[str, Any],
    config: dict[str, Any],
) -> dict[str, Any]:
    """
    在这里替换为你的 LangChain Deep Agent 调用逻辑。

    输入：
    - Dataset Item 的 input
    - Langfuse Default config

    输出：
    - 结构化测试用例
    - Excel 地址
    - 未决事项
    """

    # 示例：实际应替换为：
    #
    # result = deep_agent.invoke(...)
    # generated_excel_url = render_excel(result)
    # return {
    #     "test_cases": result["test_cases"],
    #     "excel_url": generated_excel_url,
    #     "open_issues": result["open_issues"],
    # }

    return {
        "test_cases": [
            {
                "test_id": "UT-LOGIN-001",
                "design_ids": ["GF-LOGIN-001"],
                "target": "ログイン画面",
                "category": "異常系",
                "viewpoint": "USERID/PASSWORD 必填检查",
                "preconditions": [
                    "ログイン画面が表示されていること"
                ],
                "test_data": [
                    "USERID：空白",
                    "PASSWORD：ValidPass123"
                ],
                "steps": [
                    "USERID を空白のままにする",
                    "PASSWORD を入力する",
                    "LOGIN ボタンを押下する"
                ],
                "expected_results": [
                    "必須エラーメッセージが表示される",
                    "LDAP 認証を実行しない",
                    "ログイン履歴を登録しない"
                ],
                "verification_method": [
                    "画面確認",
                    "ログ確認",
                    "DB確認"
                ]
            }
        ],
        "excel_url": None,
        "open_issues": [],
        "agent_config": {
            "agent_version": config.get("agent_version"),
            "prompt_version": config.get("prompt_version"),
            "model_deployment": config.get("model_deployment"),
        }
    }


def run_rule_evaluators(
    expected_output: dict[str, Any] | None,
    agent_output: dict[str, Any],
) -> dict[str, float]:
    """
    这里替换为你的确定性规则评测器。

    示例指标：
    - 测试用例完整性
    - 测试 ID 唯一性
    - Excel 是否生成
    - 是否包含设计书 ID
    - 是否覆盖要求的 Test Intent
    """

    test_cases = agent_output.get("test_cases", [])

    required_fields = [
        "test_id",
        "design_ids",
        "target",
        "category",
        "viewpoint",
        "preconditions",
        "test_data",
        "steps",
        "expected_results",
        "verification_method",
    ]

    complete_cases = 0

    for case in test_cases:
        is_complete = all(
            case.get(field) not in (None, "", [])
            for field in required_fields
        )

        if is_complete:
            complete_cases += 1

    case_completeness_rate = (
        complete_cases / len(test_cases)
        if test_cases
        else 0.0
    )

    return {
        "case_completeness_rate": case_completeness_rate,
        "excel_validation_pass": 1.0,
    }


def run_llm_judge(
    design_input: dict[str, Any],
    expected_output: dict[str, Any] | None,
    agent_output: dict[str, Any],
    config: dict[str, Any],
) -> dict[str, float]:
    """
    这里替换为 LLM-as-a-Judge 调用。

    Judge 推荐评估：
    - design_coverage
    - semantic_coverage
    - hallucination_rate
    - traceability_rate
    """

    # 示例固定返回值。
    # 实际应调用独立 Judge LLM，并要求 JSON Schema 输出。
    return {
        "design_coverage": 0.92,
        "test_intent_recall": 0.89,
        "semantic_coverage": 0.91,
        "traceability_rate": 1.0,
        "hallucination_rate": 0.0,
    }


def execute_remote_experiment(payload: dict[str, Any]) -> None:
    """
    后台执行整个 Dataset Experiment。
    """

    logger.info(
        "Received remote experiment payload:\n%s",
        json.dumps(payload, ensure_ascii=False, indent=2),
    )

    # 注意：
    # 不同 Langfuse 版本的 Webhook payload 字段名称可能存在差异。
    # 第一次联调时，应先查看上面的原始日志。

    dataset_name = get_payload_value(
        payload,
        "datasetName",
        "dataset_name",
    )

    run_name = get_payload_value(
        payload,
        "runName",
        "run_name",
        default="unit-test-spec-remote-run",
    )

    run_description = get_payload_value(
        payload,
        "runDescription",
        "run_description",
        default="Remote experiment: design to unit-test specification",
    )

    run_metadata = get_payload_value(
        payload,
        "runMetadata",
        "run_metadata",
        default={},
    )

    config = payload.get("config", {})

    if not dataset_name:
        raise ValueError(
            "Webhook payload does not contain datasetName. "
            "Check the actual Langfuse webhook payload in logs."
        )

    # 1. 获取 Langfuse Dataset。
    dataset = langfuse.get_dataset(
        name=dataset_name,
    )

    # 2. 逐条执行 Dataset Item。
    for item in dataset.items:
        try:
            # 3. 调用你的 Deep Agent。
            agent_output = invoke_unit_test_deep_agent(
                design_input=item.input,
                config=config,
            )

            # 4. 执行确定性规则评测。
            rule_scores = run_rule_evaluators(
                expected_output=item.expected_output,
                agent_output=agent_output,
            )

            # 5. 执行 LLM Judge。
            judge_scores = run_llm_judge(
                design_input=item.input,
                expected_output=item.expected_output,
                agent_output=agent_output,
                config=config,
            )

            scores = {
                **rule_scores,
                **judge_scores,
            }

            # 6. 关键步骤：
            #    item.run() 会将 output、score 关联至 Dataset Run / Experiment。
            with item.run(
                run_name=run_name,
                run_description=run_description,
                metadata={
                    **run_metadata,
                    **config,
                    "dataset_item_id": item.id,
                },
            ) as run:
                run.update(
                    output=agent_output,
                )

                for score_name, score_value in scores.items():
                    run.score(
                        name=score_name,
                        value=score_value,
                    )

            logger.info(
                "Experiment item completed: item_id=%s, run_name=%s",
                item.id,
                run_name,
            )

        except Exception as exc:
            logger.exception(
                "Experiment item failed: item_id=%s",
                item.id,
            )

            # 失败也写回 Langfuse，避免 UI 中没有任何记录。
            with item.run(
                run_name=run_name,
                run_description=run_description,
                metadata={
                    **run_metadata,
                    **config,
                    "dataset_item_id": item.id,
                    "execution_status": "failed",
                },
            ) as run:
                run.update(
                    output={
                        "error": str(exc),
                        "status": "failed",
                    },
                )

                run.score(
                    name="execution_success",
                    value=0.0,
                )

    # 7. 强制将 SDK 中缓冲的事件发送到 Langfuse。
    langfuse.flush()


@app.post("/eval/unit_test_eval")
async def trigger_remote_experiment(
    request: Request,
    background_tasks: BackgroundTasks,
) -> JSONResponse:
    """
    Langfuse Remote Experiment Trigger 的 Webhook 入口。
    """

    payload = await request.json()

    # 开发阶段可打印 payload，确认真实字段名。
    logger.info(
        "Webhook accepted:\n%s",
        json.dumps(payload, ensure_ascii=False, indent=2),
    )

    background_tasks.add_task(
        execute_remote_experiment,
        payload,
    )

    return JSONResponse(
        status_code=202,
        content={
            "accepted": True,
            "message": "Remote experiment queued.",
        },
    )
```

启动：

```bash
uvicorn main:app --host 0.0.0.0 --port 8000
```

---

# 4. 关键点：`run_name` 必须一致

同一轮实验中的所有 Item，必须使用相同的：

```python
run_name
```

例如：

```text
unit-test-spec-deepagent-v1
```

Langfuse 才能把它们归类为同一轮 Dataset Run / Experiment。

之后你可以运行另一套参数：

```text
unit-test-spec-deepagent-v2
```

并在 Langfuse Experiment 页面比较：

```text
v1 vs v2
```

例如：

| Run Name | design_coverage | hallucination_rate | excel_validation_pass |
|---|---:|---:|---:|
| `unit-test-spec-deepagent-v1` | 0.86 | 0.06 | 0.98 |
| `unit-test-spec-deepagent-v2` | 0.92 | 0.02 | 1.00 |

---

# 5. 先创建 Score Config

建议在 Langfuse UI 中先创建以下数值型 Score Config，避免团队成员看到不统一的分数含义。

```text
design_coverage
test_intent_recall
semantic_coverage
traceability_rate
hallucination_rate
case_completeness_rate
excel_validation_pass
execution_success
```

推荐取值范围：

```text
0.0 ～ 1.0
```

其中：

```text
design_coverage：越高越好
hallucination_rate：越低越好
excel_validation_pass：1=通过，0=失败
execution_success：1=成功，0=失败
```

---

# 6. 远程触发 Payload 首次联调时怎么确认？

你之前问过 Dataset 信息是否自动附带，答案是会附带；但具体字段名应以你实际收到的请求为准。

第一次调试时，让 FastAPI 暂时只打印请求：

```python
@app.post("/eval/unit_test_eval")
async def debug_webhook(request: Request) -> dict:
    payload = await request.json()

    print(
        json.dumps(
            payload,
            ensure_ascii=False,
            indent=2,
        )
    )

    return {
        "accepted": True
    }
```

然后在 Langfuse 中执行一次 Experiment。

重点确认日志中是否存在类似字段：

```text
datasetName
datasetId
runName
runDescription
runMetadata
config
```

若字段不是这些名字，例如结构为：

```json
{
  "dataset": {
    "name": "test"
  }
}
```

则只需要调整：

```python
dataset_name = ...
```

的读取逻辑。

---

# 7. 一个重要限制

上面示例中的：

```python
BackgroundTasks
```

适合开发和小规模测试。

正式运行大量 Dataset Item 时，建议换成任务队列：

```text
FastAPI
    ↓
Redis / RabbitMQ / Kafka
    ↓
Celery / RQ / Dramatiq Worker
    ↓
Deep Agent
    ↓
Langfuse SDK 写回 Experiment Result
```

原因是：

- Web 进程重启时 `BackgroundTasks` 可能丢失；
- Deep Agent + Excel + LLM Judge 耗时长；
- 需要控制并发；
- 需要失败重试；
- 需要任务状态管理；
- 需要避免同一 Experiment 被重复触发。



---

**User** _(2026-08-24 16:43:03)_

---
title: Code evaluators (self-hosted)
description: Configure dispatchers and workers for self-hosted code evaluator execution.
label: "Version: v4"
sidebarTitle: "Code evaluators"
---

# Code evaluators

This page covers self-hosted infrastructure setup. For an introduction to the feature, see [Code evaluators](/docs/evaluation/evaluation-methods/code-evaluators).

Self-hosted code evaluators are picked up by the Langfuse worker and executed through a dispatcher backend. The dispatcher currently invokes either AWS Lambda or the local insecure runner synchronously. Production self-hosted deployments need a dispatcher for execution.

Code evaluator availability is derived from `LANGFUSE_CODE_EVAL_DISPATCHER`. If this variable is unset, code evaluators are disabled for self-hosted deployments. Set it on the web service so the UI can show the supported code evaluator templates, and on the worker service so executions can be dispatched.

## Available dispatchers

Set `LANGFUSE_CODE_EVAL_DISPATCHER` to one of the available dispatchers:

| Dispatcher     | Value            | Use for                                   | Runtime support                  |
| -------------- | ---------------- | ----------------------------------------- | -------------------------------- |
| AWS Lambda     | `aws-lambda`     | Production self-hosted deployments        | TypeScript/JavaScript and Python |
| Local insecure | `insecure-local` | Local development and trusted deployments | TypeScript/JavaScript only       |

  If you do not run Langfuse on AWS, use the `insecure-local` dispatcher only
  when all evaluator code is fully trusted. For sandboxed execution on other
  platforms, let us know what support you need in [GitHub
  Discussions](https://github.com/orgs/langfuse/discussions).

## AWS Lambda dispatcher

Use the AWS Lambda dispatcher for production self-hosted deployments:

```bash
LANGFUSE_CODE_EVAL_DISPATCHER=aws-lambda
QUEUE_CONSUMER_CODE_EVAL_EXECUTION_QUEUE_IS_ENABLED=true
```

The AWS Lambda dispatcher enables TypeScript/JavaScript and Python code evaluators.

By default, Langfuse dispatches TypeScript/JavaScript and Python evaluators to these Lambda functions:

```bash
LANGFUSE_CODE_EVAL_AWS_LAMBDA_NODE_FUNCTION_NAME=code-based-eval-executor-node
LANGFUSE_CODE_EVAL_AWS_LAMBDA_PYTHON_FUNCTION_NAME=code-based-eval-executor-python
```

Set these variables if your Lambda functions use different names.

### Lambda runners

Create one Lambda runner per supported runtime. The runner source code is available on `main`:

| Runtime    | Default function name             | AWS runtime  | Handler         | Bundle file | Runner source                                                                                                                                                               |
| ---------- | --------------------------------- | ------------ | --------------- | ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Python     | `code-based-eval-executor-python` | `python3.14` | `index.handler` | `index.py`  | [`scripts/code-eval-runners/python/code_based_eval_handler.py`](https://github.com/langfuse/langfuse/blob/main/scripts/code-eval-runners/python/code_based_eval_handler.py) |
| JavaScript | `code-based-eval-executor-node`   | `nodejs24.x` | `index.handler` | `index.mjs` | [`scripts/code-eval-runners/node/code-based-eval-handler.mjs`](https://github.com/langfuse/langfuse/blob/main/scripts/code-eval-runners/node/code-based-eval-handler.mjs)   |

Configure each Lambda with [tenant isolation](https://docs.aws.amazon.com/lambda/latest/dg/tenant-isolation.html) enabled:

```hcl
tenancy_config {
  tenant_isolation_mode = "PER_TENANT"
}
```

Tenant isolation mode is supported in all commercial Regions except Asia Pacific (New Zealand).

Use short timeouts and bounded concurrency so evaluator execution cannot consume unbounded Lambda capacity. Langfuse Cloud currently uses 128 MB memory, a 2 second timeout, and reserved concurrency per runtime.

### IAM permissions

Grant the Langfuse worker permission to invoke only the code evaluator runner functions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "lambda:InvokeFunction",
      "Resource": [
        "arn:aws:lambda:<region>:<account-id>:function:code-based-eval-executor-node",
        "arn:aws:lambda:<region>:<account-id>:function:code-based-eval-executor-python"
      ]
    }
  ]
}
```

The Lambda execution role should be minimal: trust `lambda.amazonaws.com`, allow writing to the runner CloudWatch log groups, and avoid application, database, secret, or storage permissions. User-provided evaluator code runs inside these functions, so the runner role should not have access to Langfuse infrastructure beyond what the Lambda platform itself requires.

For local AWS-compatible endpoints, set a custom Lambda endpoint:

```bash
LANGFUSE_CODE_EVAL_AWS_LAMBDA_ENDPOINT=http://localhost:4566
```

  Self-hosters deploying Lambda runners are responsible for isolating evaluator
  execution, including no-network-egress isolation.

## Local insecure dispatcher

For local development and trusted production environments, you can run code evaluators in the worker process:

```bash
LANGFUSE_CODE_EVAL_DISPATCHER=insecure-local
QUEUE_CONSUMER_CODE_EVAL_EXECUTION_QUEUE_IS_ENABLED=true
```

`insecure-local` runs user-provided code inside the Langfuse worker process. It is only appropriate when all evaluator code is trusted, including in production. It is not a sandbox boundary for untrusted code. It currently supports TypeScript/JavaScript only.

## Queue and runtime knobs

These optional variables tune execution behavior:

| Variable                                          | Description                                                   |
| ------------------------------------------------- | ------------------------------------------------------------- |
| `LANGFUSE_CODE_EVAL_EXECUTION_QUEUE_SHARD_COUNT`  | Number of shards for the code evaluator execution queue.      |
| `LANGFUSE_CODE_EVAL_EXECUTION_WORKER_CONCURRENCY` | Worker concurrency for code evaluator execution.              |
| `LANGFUSE_CODE_EVAL_LOCAL_TIMEOUT_MS`             | Timeout in milliseconds for locally executed code evaluators. |

If you experience any issues when self-hosting Langfuse, please:

1. Check out [Troubleshooting & FAQ](/self-hosting/troubleshooting-and-faq) page.
2. Use [Ask AI](/ask-ai) to get instant answers to your questions.
3. Ask the maintainers on [GitHub Discussions](/gh-support).
4. Create a bug report or feature request on [GitHub](/issues).

  Enterprise-grade support is available when self-hosting Langfuse. Learn more on
  our [pricing page](/pricing-self-host).

<!-- agent-instructions -->

---

## Agent Instructions

This page is part of the [Langfuse](https://langfuse.com) documentation, published as plain Markdown for AI agents. Every page is available as Markdown by appending `.md` to its URL, or by sending an `Accept: text/markdown` header. This page: `https://langfuse.com/self-hosting/configuration/code-evaluators.md`.

### Querying these docs

If the answer is not on this page, query the documentation instead of guessing:

- **Semantic search** across all Langfuse docs, returning an answer with the relevant pages and excerpts. Ask a specific, self-contained question:

  ```bash
  curl -sG "https://langfuse.com/api/search-docs" --data-urlencode "query=How do I trace a LangGraph agent?"
  ```

- **Index of every page**: <https://langfuse.com/llms.txt>, with per-section indexes [llms-docs.txt](https://langfuse.com/llms-docs.txt), [llms-integrations.txt](https://langfuse.com/llms-integrations.txt), and [llms-self-hosting.txt](https://langfuse.com/llms-self-hosting.txt).

### Before writing Langfuse code

- **Install the [Langfuse Agent Skill](https://langfuse.com/docs/api-and-data-platform/features/agent-skill).** It encodes Langfuse's own best practices for instrumentation, prompt management, and evaluation, and materially improves results.
- **Read [What does a good trace look like?](https://langfuse.com/docs/observability/best-practices.md)** before instrumenting an application.
- **Verify endpoints, parameters, and response fields** against the [API reference](https://api.reference.langfuse.com) instead of inferring them from code examples.
- **Use the [Langfuse CLI](https://langfuse.com/docs/api-and-data-platform/features/cli)** (`npx langfuse-cli api <resource> <action>`) to read or write traces, prompts, datasets, and scores from the terminal.

Found an error in these docs? Please open an issue at <https://github.com/langfuse/langfuse-docs/issues>.
如何设置Code evaluators

---

**Bot** _(2026-08-24 16:44:03)_

## 内容总结

你贴出的文档说明了 Langfuse **自托管 Code Evaluators（代码评测器）** 的基础设施配置方式：

- 自托管环境默认不启用 Code Evaluators；
- 必须设置 `LANGFUSE_CODE_EVAL_DISPATCHER`；
- 配置需要同时作用于 Langfuse 的 **Web 服务** 和 **Worker 服务**；
- 可选执行器有两种：
  - `aws-lambda`：生产环境推荐，支持 Python 和 TypeScript/JavaScript；
  - `insecure-local`：仅适合本地开发或完全可信环境，只支持 TypeScript/JavaScript；
- 还需开启 Code Evaluator 执行队列；
- AWS Lambda 方案需部署 Runner、配置 Worker 的 IAM 调用权限；
- 本地模式会直接在 Langfuse Worker 内执行用户代码，不是安全沙箱。

---

## 中文翻译

# 代码评测器

本页面说明自托管 Langfuse 的代码评测器基础设施配置。有关功能介绍，请参阅“代码评测器”文档。

自托管代码评测器由 Langfuse Worker 获取，并通过 Dispatcher 后端执行。目前 Dispatcher 可以同步调用 AWS Lambda 或本地不安全 Runner。生产环境的自托管部署需要配置 Dispatcher。

代码评测器是否可用由 `LANGFUSE_CODE_EVAL_DISPATCHER` 决定：

- 若未设置该变量，自托管部署中的代码评测器功能将被禁用；
- 应在 Web 服务中设置该变量，使 UI 显示支持的代码评测器模板；
- 应在 Worker 服务中设置该变量，使 Worker 能分发评测执行任务。

可用 Dispatcher：

| Dispatcher | 配置值 | 使用场景 | 支持的运行时 |
|---|---|---|---|
| AWS Lambda | `aws-lambda` | 生产环境自托管 | TypeScript/JavaScript、Python |
| 本地不安全 Runner | `insecure-local` | 本地开发、完全可信部署 | TypeScript/JavaScript |

若不是 AWS 环境，只应在所有评测代码均可信的情况下使用 `insecure-local`。它不是安全隔离边界。

AWS Lambda 模式需要设置：

```bash
LANGFUSE_CODE_EVAL_DISPATCHER=aws-lambda
QUEUE_CONSUMER_CODE_EVAL_EXECUTION_QUEUE_IS_ENABLED=true
```

默认 Lambda 函数名：

```bash
LANGFUSE_CODE_EVAL_AWS_LAMBDA_NODE_FUNCTION_NAME=code-based-eval-executor-node
LANGFUSE_CODE_EVAL_AWS_LAMBDA_PYTHON_FUNCTION_NAME=code-based-eval-executor-python
```

Langfuse Worker 需要具备调用这些 Lambda 的最小 IAM 权限。Lambda Runner 不应拥有应用数据库、业务系统、密钥或对象存储的额外权限。

本地不安全 Runner 配置：

```bash
LANGFUSE_CODE_EVAL_DISPATCHER=insecure-local
QUEUE_CONSUMER_CODE_EVAL_EXECUTION_QUEUE_IS_ENABLED=true
```

本地模式会在 Langfuse Worker 进程中直接执行用户代码，仅支持 TypeScript/JavaScript。

---

# 你的场景应该选哪个？

你当前是：

```text
Langfuse 自托管
+ FastAPI
+ LangChain Deep Agent
+ Python
+ 内部设计书 → 单体测试式样书
```

建议这样选择。

| 场景 | 推荐 Dispatcher | 原因 |
|---|---|---|
| 本地 PoC、开发验证、所有代码都由自己维护 | `insecure-local` | 配置最简单，但只能写 JS/TS Evaluator |
| 生产环境，需要 Python Code Evaluator | `aws-lambda` | 支持 Python，隔离性更好 |
| 生产环境，但不在 AWS | 不建议直接使用 `insecure-local` | 需要自行提供隔离执行环境，或继续用自己的 FastAPI Evaluator |
| Excel 深度解析、`openpyxl`、复杂 DB 检查、RAG、Deep Agent | 自己的 FastAPI / Worker | 不适合放进短时 Code Evaluator |
| JSON 完整性、字段检查、ID 重复、简单规则 | Code Evaluator | 很适合 |

---

# 一、推荐的职责划分

不要把所有评测都塞进 Langfuse Code Evaluator。

对于你的“测试式样书生成 Agent”，推荐：

```text
Remote Experiment Webhook
    ↓
你的 FastAPI / Deep Agent
    ├── 读取 Excel
    ├── 调用 RAG / Deep Agent
    ├── 生成测试用例 JSON
    ├── 生成 Excel
    ├── 复杂规则评测
    ├── LLM Judge
    └── 通过 SDK 写回结果和 Score
    ↓
Langfuse
    ├── Dataset
    ├── Experiment
    ├── Trace
    ├── Score
    └── Code Evaluator
         ├── JSON Schema 校验
         ├── 测试 ID 重复检查
         ├── 必填字段检查
         └── 简单确定性规则
```

即：

| 任务 | 放在哪里执行 |
|---|---|
| Deep Agent 运行 | 你的 FastAPI / Worker |
| Excel 读取、生成、复杂校验 | 你的 FastAPI / Worker |
| LLM-as-a-Judge | Langfuse LLM Evaluator 或你的服务 |
| JSON 结构校验 | Langfuse Code Evaluator |
| 测试用例必填字段校验 | Langfuse Code Evaluator |
| 测试 ID 唯一性校验 | Langfuse Code Evaluator |
| 数值型/分类 Score 写回 | Langfuse SDK/API |
| 人工审核 | Annotation Queue / UI Score |

---

# 二、本地开发：配置 `insecure-local`

如果当前只是内网开发、PoC，并且只有你们可信团队成员可以创建/修改 Evaluator，建议先使用：

```text
insecure-local
```

## Docker Compose 示例

假设你的 Docker Compose 服务名为：

```text
langfuse-web
langfuse-worker
```

则配置如下。

```yaml
services:
  langfuse-web:
    environment:
      LANGFUSE_CODE_EVAL_DISPATCHER: insecure-local

  langfuse-worker:
    environment:
      LANGFUSE_CODE_EVAL_DISPATCHER: insecure-local
      QUEUE_CONSUMER_CODE_EVAL_EXECUTION_QUEUE_IS_ENABLED: "true"
```

如果你的服务实际名称是：

```text
web
worker
```

则应按实际服务名调整：

```yaml
services:
  web:
    environment:
      LANGFUSE_CODE_EVAL_DISPATCHER: insecure-local

  worker:
    environment:
      LANGFUSE_CODE_EVAL_DISPATCHER: insecure-local
      QUEUE_CONSUMER_CODE_EVAL_EXECUTION_QUEUE_IS_ENABLED: "true"
```

重启服务：

```bash
docker compose up -d --force-recreate langfuse-web langfuse-worker
```

或者：

```bash
docker compose restart langfuse-web langfuse-worker
```

检查 Worker 日志：

```bash
docker compose logs -f langfuse-worker
```

你需要确认：

- Worker 正常启动；
- 没有环境变量解析错误；
- Code Evaluator Queue 已启用；
- UI 中能够看到 Code Evaluator 模板或创建入口。

---

## `insecure-local` 的重要限制

根据你贴出的文档：

```text
insecure-local
    = 仅支持 TypeScript / JavaScript
    = 在 Langfuse Worker 进程内直接执行
    = 不提供沙箱隔离
```

因此：

- 不能在此模式直接执行 Python Evaluator；
- 不能让不可信用户提交代码；
- 不要给 Langfuse UI 的普通用户随意创建可执行代码的权限；
- 不要把数据库密码、Azure Key、Langfuse Secret、业务 API Key 暴露给 Evaluator 运行环境；
- 不建议在公网、多租户或权限边界复杂的生产环境使用。

---

# 三、生产环境：配置 `aws-lambda`

如果你需要在 Langfuse 中运行 Python Code Evaluator，例如：

```text
- 用 Python 校验 Agent 输出 JSON；
- 校验 Excel 文件结构；
- 调用内部规则库；
- 使用 Python 自定义业务逻辑；
```

建议使用 AWS Lambda Dispatcher。

## Langfuse Web 服务配置

```yaml
services:
  langfuse-web:
    environment:
      LANGFUSE_CODE_EVAL_DISPATCHER: aws-lambda
```

## Langfuse Worker 服务配置

```yaml
services:
  langfuse-worker:
    environment:
      LANGFUSE_CODE_EVAL_DISPATCHER: aws-lambda
      QUEUE_CONSUMER_CODE_EVAL_EXECUTION_QUEUE_IS_ENABLED: "true"

      LANGFUSE_CODE_EVAL_AWS_LAMBDA_NODE_FUNCTION_NAME: code-based-eval-executor-node
      LANGFUSE_CODE_EVAL_AWS_LAMBDA_PYTHON_FUNCTION_NAME: code-based-eval-executor-python
```

如果你创建的 Lambda 名称不同，例如：

```text
langfuse-evaluator-python-prod
langfuse-evaluator-node-prod
```

则改为：

```yaml
services:
  langfuse-worker:
    environment:
      LANGFUSE_CODE_EVAL_AWS_LAMBDA_NODE_FUNCTION_NAME: langfuse-evaluator-node-prod
      LANGFUSE_CODE_EVAL_AWS_LAMBDA_PYTHON_FUNCTION_NAME: langfuse-evaluator-python-prod
```

---

## Worker 的 IAM 权限

Langfuse Worker 所使用的 AWS IAM Role，最低只需要允许调用指定 Lambda。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "lambda:InvokeFunction",
      "Resource": [
        "arn:aws:lambda:<region>:<account-id>:function:code-based-eval-executor-node",
        "arn:aws:lambda:<region>:<account-id>:function:code-based-eval-executor-python"
      ]
    }
  ]
}
```

不要给 Langfuse Worker：

```text
- AdministratorAccess
- AWSLambda_FullAccess
- 数据库访问权限
- S3 全权限
- Secrets Manager 读取权限
- 业务系统访问权限
```

---

## Lambda Runner 权限原则

因为用户编写的 Evaluator 代码会在 Lambda Runner 中运行，所以 Lambda 执行角色应尽量只拥有：

```text
- CloudWatch 日志写入权限
- Lambda 运行基础权限
```

不应直接拥有：

```text
- 生产数据库访问权限
- Azure OpenAI API Key
- 内部业务 API Token
- 对象存储读写权限
- Langfuse 数据库权限
- 企业文档访问权限
```

对于你的场景，尤其不能让代码评测器直接访问：

```text
内部设计书
单体测试式样书
员工信息
LDAP
业务数据库
```

除非你已经有严格的权限、网络和审计控制。

---

# 四、Code Evaluator 在 UI 中如何创建？

完成 Dispatcher 配置、重启 Langfuse 后，按以下方式操作。

```text
1. 进入 Langfuse 的 Evaluators / Evaluation 页面
2. 新建 Code Evaluator
3. 选择运行时
   - 本地 insecure-local：JavaScript / TypeScript
   - AWS Lambda：JavaScript / TypeScript 或 Python
4. 选择评分类型
   - Numeric
   - Categorical
5. 选择评测对象
   - Trace
   - Observation
   - Experiment Result
6. 保留 Langfuse UI 自动生成的函数入口和参数签名
7. 只替换内部评测逻辑
8. 保存并测试
9. 将 Evaluator 应用于 Dataset Experiment 或生产 Trace
```

> 不要自行猜测 UI 模板函数的入参、返回值和入口函数名称。  
> 应保留当前 Langfuse UI 自动生成的 Evaluator 框架，只填写核心逻辑。

---

# 五、适合你的第一个 Code Evaluator

建议先做一个最简单、最稳定的：

```text
case_completeness_rate
```

它检查 Agent 生成的每条测试用例是否包含：

```text
- test_id
- design_ids
- target
- category
- viewpoint
- preconditions
- test_data
- steps
- expected_results
- verification_method
```

逻辑示例如下。请将其核心判断逻辑填入 Langfuse UI 自动生成的 JavaScript/TypeScript 模板中。

```javascript
function calculateCaseCompleteness(agentOutput) {
  const testCases = agentOutput?.test_cases ?? [];

  if (testCases.length === 0) {
    return {
      value: 0,
      comment: "未生成任何测试用例。"
    };
  }

  const requiredFields = [
    "test_id",
    "design_ids",
    "target",
    "category",
    "viewpoint",
    "preconditions",
    "test_data",
    "steps",
    "expected_results",
    "verification_method"
  ];

  const incompleteCases = [];

  for (const testCase of testCases) {
    const missingFields = requiredFields.filter((field) => {
      const value = testCase[field];

      return (
        value === undefined ||
        value === null ||
        value === "" ||
        (Array.isArray(value) && value.length === 0)
      );
    });

    if (missingFields.length > 0) {
      incompleteCases.push({
        testId: testCase.test_id ?? "unknown",
        missingFields
      });
    }
  }

  const completeCount = testCases.length - incompleteCases.length;
  const value = completeCount / testCases.length;

  return {
    value,
    comment: JSON.stringify({
      totalCases: testCases.length,
      completeCases: completeCount,
      incompleteCases
    })
  };
}
```

预期评分：

```text
1.0
= 所有测试用例字段完整

0.8
= 80% 的测试用例字段完整

0.0
= 没有测试用例，或全部缺少关键字段
```

对应的 Score Config：

```text
名称：
case_completeness_rate

类型：
Numeric

范围：
0 ～ 1

含义：
越高越好
```

---

# 六、建议建立的首批 Code Evaluator

| Evaluator 名称 | 类型 | 用途 |
|---|---|---|
| `case_completeness_rate` | Numeric | 用例字段是否完整 |
| `test_id_uniqueness` | Numeric | 测试 ID 是否重复 |
| `design_id_presence_rate` | Numeric | 测试用例是否关联设计书 ID |
| `required_sheet_validation` | Categorical / Numeric | Excel 必需 Sheet 是否存在 |
| `output_json_validation` | Categorical | Agent 输出是否符合 JSON Schema |
| `empty_test_case_check` | Categorical | 是否生成空测试用例 |
| `duplicate_case_rate` | Numeric | 重复测试用例比例 |
| `excel_validation_pass` | Numeric | Excel 是否生成且符合模板基础要求 |
| `forbidden_term_check` | Numeric | 是否包含明确禁止的虚构功能 |
| `execution_success` | Numeric | Agent / Evaluator 是否执行成功 |

例如：

```text
excel_validation_pass:
1 = 通过
0 = 失败
```

这样比使用字符串：

```text
pass / fail
```

更方便做 Experiment 聚合和趋势图。

---

# 七、哪些评测不建议放到 Code Evaluator？

以下任务建议保留在你的 FastAPI / Deep Agent 服务中，再通过 SDK 写回 Score。

| 任务 | 不建议放入 Code Evaluator 的原因 |
|---|---|
| 调用 Deep Agent | 耗时长、流程复杂、需要模型和工具调用 |
| 读取大量 Excel | 文件访问、内存、依赖、权限复杂 |
| 用 `openpyxl` 深度校验 Excel | 本地模式不支持 Python；Lambda 时间和包依赖受限 |
| RAG 检索 | 需要向量库、Embedding、网络访问 |
| 调用企业内部 API | 权限和网络隔离风险高 |
| 调用 Azure OpenAI | 不应将 Azure 密钥暴露给用户可编辑代码 |
| LLM-as-a-Judge | 应使用独立模型连接、独立权限与成本控制 |
| 批量 Dataset 执行 | 应由 Remote Experiment Webhook / Worker 执行 |
| 复杂设计书语义解析 | 更适合 Deep Agent 或独立评测服务 |

---

# 八、你当前最实用的配置方案

## 开发环境

```text
Langfuse Worker
    └── insecure-local
         └── JavaScript Code Evaluator
              ├── JSON 完整性
              ├── 测试 ID 检查
              ├── 必填字段检查
              └── 简单重复检查

FastAPI / Deep Agent
    ├── 设计书 Excel 解析
    ├── RAG
    ├── 测试式样书生成
    ├── Excel 输出
    ├── 复杂规则评测
    └── Langfuse SDK 回传结果和 Score
```

## 生产环境

```text
Langfuse Worker
    └── AWS Lambda Dispatcher
         ├── Python / JavaScript Code Evaluator
         └── 最小 IAM 权限、限时、限并发

FastAPI / Worker Queue
    ├── Deep Agent
    ├── Excel 生成
    ├── LLM Judge
    ├── 复杂业务规则评测
    └── Langfuse SDK / API 写回
```

---

# 九、最小配置检查清单

```text
□ 已确认 Langfuse 是自托管版本
□ Web 服务设置 LANGFUSE_CODE_EVAL_DISPATCHER
□ Worker 服务设置 LANGFUSE_CODE_EVAL_DISPATCHER
□ Worker 设置 QUEUE_CONSUMER_CODE_EVAL_EXECUTION_QUEUE_IS_ENABLED=true
□ 已重启 Web 和 Worker
□ Worker 日志无错误
□ UI 中能看到 Code Evaluator 创建入口
□ 已创建 Numeric Score Config
□ 已新建并测试 case_completeness_rate
□ 只允许可信管理员编辑 Evaluator 代码
□ 未将 API Key、数据库密码、业务 Token 写入 Evaluator
□ 复杂 Excel / LLM / Agent 评测仍在独立服务执行
```

最建议你先从：

```text
insecure-local
+
JavaScript 的 case_completeness_rate
```

开始跑通。确认 Langfuse Worker 能执行并写入 Score 后，再决定是否需要 AWS Lambda Python Runner。



---

**User** _(2026-08-24 17:51:28)_

 with item.run(
                    run_name=run_name,
                    run_description=run_description,
                    metadata={
                        **run_metadata,
                        **config,
                        "dataset_item_id": item.id,
                    },  ： 'DatasetItem 里没有run

---

**Bot** _(2026-08-24 17:56:00)_

你说得对。**`DatasetItem` 没有 `.run()`**，我前面给出的：

```python
with item.run(...):
```

对于你当前使用的 Langfuse Python SDK 是错误的，抱歉。

当前正确模式是：

```text
langfuse.api.dataset_items.get_many(...)
        ↓
获得 DatasetItem 列表
        ↓
langfuse.run_experiment(...)
        ↓
task 返回 Agent Output
evaluator 返回 Evaluation Score
        ↓
Langfuse SDK 自动写入 Dataset Run、Item Output、Trace 和 Score
```

也就是说，**不需要自行实现 `post_result_to_langfuse()`**，也不需要对每个 Item 手工调用 `create_score()` 来关联 Experiment。

---

# 正确的核心代码

```python
from langfuse import get_client
from langfuse.experiment import Evaluation

langfuse = get_client()


def task(*, item, **kwargs):
    """
    item 是 Langfuse DatasetItem。

    当前实验运行器中，使用 item["input"] 读取输入，
    不要假设 item 有 item.run() 或 item.input 属性。
    """
    design_input = item["input"]

    # 替换为你的 Deep Agent 调用。
    return {
        "test_cases": [
            {
                "test_id": "UT-LOGIN-001",
                "category": "異常系",
                "viewpoint": "USERID/PASSWORD 必填检查",
                "expected_results": [
                    "显示必填错误消息",
                    "不执行 LDAP 认证",
                    "不写入登录履历"
                ]
            }
        ],
        "excel_url": "https://storage.example.com/UT_BTG00000.xlsx"
    }


def case_completeness_evaluator(*, output, **kwargs):
    """
    Code Evaluator 示例：
    检查生成的测试用例是否具备必要字段。
    """

    test_cases = output.get("test_cases", [])

    required_fields = [
        "test_id",
        "category",
        "viewpoint",
        "expected_results",
    ]

    if not test_cases:
        return Evaluation(
            name="case_completeness_rate",
            value=0.0,
            comment="未生成测试用例。",
        )

    complete_count = 0

    for test_case in test_cases:
        is_complete = all(
            test_case.get(field) not in (None, "", [])
            for field in required_fields
        )

        if is_complete:
            complete_count += 1

    score = complete_count / len(test_cases)

    return Evaluation(
        name="case_completeness_rate",
        value=score,
        comment=(
            f"total_cases={len(test_cases)}, "
            f"complete_cases={complete_count}"
        ),
    )


def design_coverage_evaluator(*, output, expected_output, **kwargs):
    """
    示例：
    基于 Dataset Item 的 expected_output，
    评估 Agent 是否覆盖 required_test_intents。

    实际项目中可改为：
    - 规则匹配；
    - Embedding Top-K 候选匹配；
    - LLM Judge 语义判断。
    """

    if expected_output is None:
        return Evaluation(
            name="design_coverage",
            value=0.0,
            comment="Dataset Item 未配置 expected_output。",
        )

    required_intents = expected_output.get(
        "required_test_intents",
        [],
    )

    generated_cases = output.get(
        "test_cases",
        [],
    )

    if not required_intents:
        return Evaluation(
            name="design_coverage",
            value=1.0,
            comment="该 Dataset Item 未定义 required_test_intents。",
        )

    generated_text = str(generated_cases).lower()

    covered_count = 0

    for intent in required_intents:
        viewpoint = intent.get("viewpoint", "").lower()

        if viewpoint and viewpoint in generated_text:
            covered_count += 1

    score = covered_count / len(required_intents)

    return Evaluation(
        name="design_coverage",
        value=score,
        comment=(
            f"required_intents={len(required_intents)}, "
            f"covered_intents={covered_count}"
        ),
    )


# 1. 从 Langfuse 获取 Dataset Items。
page = langfuse.api.dataset_items.get_many(
    dataset_name="test",
    limit=100,
)

dataset_items = page.data

# 2. 运行 Experiment。
# Langfuse 会自动：
# - 创建 Dataset Run；
# - 为每条 Item 创建关联的执行记录；
# - 记录 task 的 output；
# - 将 Evaluation 转换为对应 Score。
result = langfuse.run_experiment(
    name="unit-test-spec-deepagent-v1",
    data=dataset_items,
    task=task,
    evaluators=[
        case_completeness_evaluator,
        design_coverage_evaluator,
    ],
)

# 3. 确保事件已发送到 Langfuse。
langfuse.flush()

print("dataset_run_id:", result.dataset_run_id)
print("dataset_run_url:", result.dataset_run_url)
```

---

# 对应关系

| 你的需求 | 正确实现 |
|---|---|
| 获取 Dataset Item | `langfuse.api.dataset_items.get_many(...)` |
| 调用 Deep Agent | `task(*, item, **kwargs)` 内执行 |
| 返回 Agent 输出 | `return {...}` |
| 写入 Experiment Item Output | `run_experiment()` 自动完成 |
| 写入 Score | Evaluator 返回 `Evaluation(...)` |
| 建立 Dataset Item 与 Trace 的关联 | `run_experiment()` 自动完成 |
| 查看 Experiment URL | `result.dataset_run_url` |
| 强制发送埋点数据 | `langfuse.flush()` |

---

# Remote Experiment Webhook 的正确写法

你的 FastAPI Webhook 接收到 Langfuse 的 Dataset 信息和 Default Config 后，内部调用 `run_experiment()`。

```python
import json
from typing import Any

from fastapi import FastAPI, HTTPException, Request
from langfuse import get_client
from langfuse.experiment import Evaluation

app = FastAPI()
langfuse = get_client()


def get_dataset_name(payload: dict[str, Any]) -> str | None:
    """
    不同版本或配置下，Webhook Payload 的字段层级可能不同。
    首次联调时，应打印实际 payload 后再收紧 Schema。
    """

    return (
        payload.get("datasetName")
        or payload.get("dataset_name")
        or payload.get("dataset", {}).get("name")
    )


def get_run_name(payload: dict[str, Any]) -> str:
    return (
        payload.get("runName")
        or payload.get("run_name")
        or payload.get("config", {}).get("run_name")
        or "unit-test-spec-deepagent-v1"
    )


def create_deep_agent_task(config: dict[str, Any]):
    """
    用闭包把 Langfuse Default config 传给 Agent。
    """

    def task(*, item, **kwargs):
        design_input = item["input"]

        # 这里替换为你的真实 Deep Agent 调用。
        #
        # result = deep_agent.invoke({
        #     "design_input": design_input,
        #     "agent_version": config.get("agent_version"),
        #     "prompt_version": config.get("prompt_version"),
        #     "model_deployment": config.get("model_deployment"),
        # })
        #
        # return result

        return {
            "test_cases": [
                {
                    "test_id": "UT-LOGIN-001",
                    "design_ids": ["GF-LOGIN-001"],
                    "target": "ログイン画面",
                    "category": "異常系",
                    "viewpoint": "USERID/PASSWORD 必填检查",
                    "preconditions": [
                        "登录画面已显示"
                    ],
                    "test_data": [
                        "USERID：空",
                        "PASSWORD：有效值"
                    ],
                    "steps": [
                        "输入测试数据",
                        "点击 LOGIN"
                    ],
                    "expected_results": [
                        "显示必填错误消息",
                        "不执行 LDAP 认证",
                        "不新增登录履历"
                    ],
                    "verification_method": [
                        "画面确认",
                        "日志确认",
                        "DB 确认"
                    ]
                }
            ],
            "excel_url": None,
            "open_issues": [],
            "agent_metadata": {
                "agent_version": config.get("agent_version"),
                "prompt_version": config.get("prompt_version"),
                "model_deployment": config.get("model_deployment"),
            }
        }

    return task


def execution_success_evaluator(*, output, **kwargs):
    success = bool(output.get("test_cases"))

    return Evaluation(
        name="execution_success",
        value=1.0 if success else 0.0,
        comment=(
            "测试用例生成成功。"
            if success
            else "未生成测试用例。"
        ),
    )


@app.post("/eval/unit_test_eval")
async def run_remote_experiment(request: Request) -> dict[str, Any]:
    payload = await request.json()

    # 首次联调时务必保留日志。
    print(
        json.dumps(
            payload,
            ensure_ascii=False,
            indent=2,
        )
    )

    dataset_name = get_dataset_name(payload)
    run_name = get_run_name(payload)
    config = payload.get("config", {})

    if not dataset_name:
        raise HTTPException(
            status_code=400,
            detail=(
                "无法从 Webhook Payload 获取 dataset_name。"
                "请查看 FastAPI 日志中的实际请求内容。"
            ),
        )

    # 1. 获取当前 Dataset 的 Item。
    page = langfuse.api.dataset_items.get_many(
        dataset_name=dataset_name,
        limit=100,
    )

    if not page.data:
        raise HTTPException(
            status_code=404,
            detail=f"Dataset '{dataset_name}' 没有可用 Item。",
        )

    # 2. 通过 config 创建本次实验的 Agent Task。
    task = create_deep_agent_task(config)

    # 3. 执行实验。
    experiment_result = langfuse.run_experiment(
        name=run_name,
        data=page.data,
        task=task,
        evaluators=[
            execution_success_evaluator,
            case_completeness_evaluator,
            design_coverage_evaluator,
        ],
    )

    # 4. 将 SDK 缓冲的 Trace、Output、Score 发送到 Langfuse。
    langfuse.flush()

    return {
        "accepted": True,
        "dataset_name": dataset_name,
        "dataset_run_id": experiment_result.dataset_run_id,
        "dataset_run_url": experiment_result.dataset_run_url,
    }
```

---

# `Default config` 如何在代码中使用？

例如在 Langfuse UI 的 Default config 中填：

```json
{
  "agent_version": "deepagent-v1",
  "prompt_version": "unit-test-spec-v3",
  "model_deployment": "azure-gpt-4o",
  "generate_excel": true,
  "enable_llm_judge": true
}
```

你的 Webhook 中读取：

```python
config = payload.get("config", {})
```

然后传给 Agent：

```python
task = create_deep_agent_task(config)
```

最终 Agent 输出中建议保留版本信息：

```json
{
  "agent_metadata": {
    "agent_version": "deepagent-v1",
    "prompt_version": "unit-test-spec-v3",
    "model_deployment": "azure-gpt-4o"
  }
}
```

这样你可以在 Langfuse 中比较：

```text
deepagent-v1 + prompt-v3
        vs
deepagent-v2 + prompt-v4
```

的：

```text
design_coverage
semantic_coverage
hallucination_rate
case_completeness_rate
excel_validation_pass
```

---

# 不要再使用的模式

以下写法不适用于你当前的 DatasetItem：

```python
with item.run(...):
    ...
```

也不建议为了 Experiment 结果手工写：

```python
langfuse.create_score(...)
```

原因是：

```text
create_score()
    → 适合为已有 Trace / Observation 单独追加分数

run_experiment()
    → 自动维护 Dataset Item、Experiment Run、
      Trace、Output、Evaluator Score 的关联
```

对于你的 Remote Experiment，优先采用：

```python
langfuse.run_experiment(...)
```

即可。 [1][2]

---

**Sources**:

1. [Experiments via SDK](https://langfuse.com/docs/evaluation/experiments/experiments-via-sdk)  
2. [Langfuse Python Experiment API Reference](https://python.reference.langfuse.com/langfuse/experiment)  
3. [Experiments Data Model](https://langfuse.com/docs/evaluation/experiments/data-model)  
4. [Query Data via SDKs](https://langfuse.com/docs/api-and-data-platform/features/query-via-sdk)  
5. [Scores via API/SDK](https://langfuse.com/docs/evaluation/evaluation-methods/scores-via-sdk)



---

**User** _(2026-08-24 18:12:16)_

应为类型 'TaskFunction'，但实际为 '(design_input: dict[str, Any], config: dict[str, Any]) -> dict[str, Any]' 

---

**Bot** _(2026-08-24 18:12:54)_

这个报错说明你把“实际执行 Agent 的函数”直接传给了 `run_experiment()`。

`run_experiment()` 的 `task` 参数要求的是 `TaskFunction`，它接收的不是：

```python
(design_input, config)
```

而是 Langfuse 在遍历 Dataset 时传入的：

```python
item
```

也就是：

```text
Langfuse Dataset Item
    ↓
TaskFunction(item)
    ↓
从 item 中取 input
    ↓
调用你的 Deep Agent 函数
```

---

## 错误写法

你当前可能类似这样写了：

```python
def generate_unit_test_spec(
    design_input: dict[str, Any],
    config: dict[str, Any],
) -> dict[str, Any]:
    ...

langfuse.run_experiment(
    name=run_name,
    data=dataset_items,
    task=generate_unit_test_spec,  # 错误：签名不符合 TaskFunction
)
```

这里 `generate_unit_test_spec()` 是你的业务函数，它需要：

```text
design_input + config
```

但 Langfuse 调用 Task 时只会传入 Dataset Item。

---

# 正确做法：增加一层 Adapter / Wrapper

保留你的 Agent 函数不变：

```python
from typing import Any


def generate_unit_test_spec(
    design_input: dict[str, Any],
    config: dict[str, Any],
) -> dict[str, Any]:
    """
    这里调用你的 LangChain Deep Agent，
    返回结构化测试式样书结果。
    """

    return {
        "test_cases": [],
        "excel_url": None,
        "open_issues": [],
    }
```

然后创建一个符合 `TaskFunction` 签名的包装函数。

```python
from typing import Any

from langfuse.experiment import TaskFunction


def create_experiment_task(
    config: dict[str, Any],
) -> TaskFunction:
    """
    将 Langfuse Dataset Item 转换为你的 Agent 输入。
    """

    def task(
        *,
        item: Any,
        **kwargs: Any,
    ) -> dict[str, Any]:
        # Langfuse Dataset Item 的 input，
        # 即你在 Dataset 中保存的设计书内容。
        design_input = item["input"]

        # 调用你真正的 Agent 业务函数。
        return generate_unit_test_spec(
            design_input=design_input,
            config=config,
        )

    return task
```

运行 Experiment 时，注意要传入：

```python
create_experiment_task(config)
```

而不是传入：

```python
generate_unit_test_spec
```

完整调用：

```python
task_function = create_experiment_task(config)

result = langfuse.run_experiment(
    name=run_name,
    data=dataset_items,
    task=task_function,
    evaluators=[
        case_completeness_evaluator,
        design_coverage_evaluator,
    ],
)

langfuse.flush()
```

---

# 最终完整结构

```python
from typing import Any

from langfuse import get_client
from langfuse.experiment import Evaluation, TaskFunction

langfuse = get_client()


def generate_unit_test_spec(
    design_input: dict[str, Any],
    config: dict[str, Any],
) -> dict[str, Any]:
    """
    你的真实 Deep Agent 调用逻辑。
    """

    function_id = design_input.get("function_id")
    agent_version = config.get("agent_version")

    print(
        f"Running Agent: function_id={function_id}, "
        f"agent_version={agent_version}"
    )

    # 此处替换为：
    # result = deep_agent.invoke(...)
    # return normalized_result

    return {
        "test_cases": [
            {
                "test_id": "UT-LOGIN-001",
                "target": "ログイン画面",
                "category": "異常系",
                "viewpoint": "USERID/PASSWORD 必填检查",
                "preconditions": [
                    "ログイン画面が表示されていること"
                ],
                "test_data": [
                    "USERID：空白",
                    "PASSWORD：ValidPass123"
                ],
                "steps": [
                    "USERID を空白のままにする",
                    "PASSWORD を入力する",
                    "LOGIN ボタンを押下する"
                ],
                "expected_results": [
                    "必須エラーメッセージが表示される",
                    "LDAP 認証を実行しない",
                    "ログイン履歴を登録しない"
                ],
                "verification_method": [
                    "画面確認",
                    "ログ確認",
                    "DB確認"
                ]
            }
        ],
        "excel_url": None,
        "open_issues": [],
    }


def create_experiment_task(
    config: dict[str, Any],
) -> TaskFunction:
    """
    返回符合 Langfuse TaskFunction 要求的函数。
    """

    def task(
        *,
        item: Any,
        **kwargs: Any,
    ) -> dict[str, Any]:
        # Dataset Item 的输入内容。
        design_input = item["input"]

        # 可选：查看当前 Dataset Item 的完整结构。
        print("Dataset item:", item)

        return generate_unit_test_spec(
            design_input=design_input,
            config=config,
        )

    return task


def case_completeness_evaluator(
    *,
    output: dict[str, Any],
    **kwargs: Any,
) -> Evaluation:
    test_cases = output.get("test_cases", [])

    if not test_cases:
        return Evaluation(
            name="case_completeness_rate",
            value=0.0,
            comment="未生成任何测试用例。",
        )

    required_fields = [
        "test_id",
        "target",
        "category",
        "viewpoint",
        "preconditions",
        "test_data",
        "steps",
        "expected_results",
        "verification_method",
    ]

    complete_count = 0

    for test_case in test_cases:
        is_complete = all(
            test_case.get(field) not in (None, "", [])
            for field in required_fields
        )

        if is_complete:
            complete_count += 1

    return Evaluation(
        name="case_completeness_rate",
        value=complete_count / len(test_cases),
        comment=(
            f"total_cases={len(test_cases)}, "
            f"complete_cases={complete_count}"
        ),
    )


# 从 Langfuse 读取 Dataset Item。
page = langfuse.api.dataset_items.get_many(
    dataset_name="test",
    limit=100,
)

# Default config 或 Webhook config。
config = {
    "agent_version": "deepagent-v1",
    "prompt_version": "unit-test-spec-v1",
    "model_deployment": "azure-gpt-4o",
    "generate_excel": True,
}

# 注意：这里调用 create_experiment_task(config)，
# 得到的 task 才符合 TaskFunction。
experiment_task = create_experiment_task(config)

result = langfuse.run_experiment(
    name="unit-test-spec-deepagent-v1",
    data=page.data,
    task=experiment_task,
    evaluators=[
        case_completeness_evaluator,
    ],
)

langfuse.flush()

print(result.dataset_run_url)
```

---

# 最关键的区别

```text
generate_unit_test_spec()
    = 你的业务 Agent 函数
    = 接收 design_input 和 config
```

```text
task()
    = Langfuse Experiment TaskFunction
    = 接收 Dataset Item
    = 从 Item 中取得 input
    = 再调用 generate_unit_test_spec()
```

调用关系：

```text
Langfuse.run_experiment()
    ↓
task(item)
    ↓
design_input = item["input"]
    ↓
generate_unit_test_spec(design_input, config)
    ↓
return Agent Output
    ↓
Langfuse 自动记录 Experiment Item Output
```

---

# 一个常见错误

下面这种写法仍然是错误的：

```python
langfuse.run_experiment(
    name=run_name,
    data=page.data,
    task=create_experiment_task,  # 错误：没有传 config，也没有得到真正 TaskFunction
)
```

正确写法是：

```python
langfuse.run_experiment(
    name=run_name,
    data=page.data,
    task=create_experiment_task(config),  # 正确
)
```

也就是必须有括号：

```text
create_experiment_task(config)
```

因为它返回的才是：

```text
task(*, item, **kwargs)
```

这个符合 Langfuse `TaskFunction` 要求的函数。 [1][2]

**Sources**:

1. [Experiments via SDK](https://langfuse.com/docs/evaluation/experiments/experiments-via-sdk)  
2. [Langfuse Python Experiment API Reference](https://python.reference.langfuse.com/langfuse/experiment)

