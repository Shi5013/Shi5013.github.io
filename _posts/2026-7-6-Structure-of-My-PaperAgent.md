---
layout: post
title: Structure-of-My-PaperAgent
tags: record
math: false
date: 2026-7-6 15:30 +0800
---

设计、实现一个复杂科研问题检索袭系统。

## asta-paper-finder

asta-paper-finder核心流程：查询理解 → 执行规划 → 多工作流检索 → LLM 相关性判断 → 排序

asta-paper-finder不是一个端到端“让 LLM 自由操作”的 agent，而是强约束的 orchestrator：先把自然语言转成结构化规格，再按 query type 路由到专门检索器，并在最后补字段、重排和解释结果。实现还覆盖 citation snowballing：先通过 dense/Semantic Scholar 检索拿到候选，再从候选论文片段里的引用反向扩展候选集合，类似 SPAR/PaSa 论文里说的 RefChain，只是 PaperFinder 采用手写流程和LLM 判断混合，而不是 RL 训练。

主入口在 asta-paper-finder/agents/mabool/api/mabool/agents/paper_finder/paper_finder_agent.py。流程大致是：

1. Query Analyzer 把自然语言 query 转成结构化对象：content、authors、venues、time_range、recency、centrality、query_type、relevance criteria。
2. Planner 根据 query_type 路由到不同 workflow：specific by title、specific by name、by author、broad search、metadata-only。
3. Broad search 使用 dense retrieval、Semantic Scholar search、keyword search、LLM suggestion、citation/snowball expansion。
4. 对候选论文运行 LLM relevance judgement，判断是否满足细粒度 criteria，并抽取 relevant snippet。
5. 最后按内容相关性、rerank score、年份、引用量、snippet 数、原始排名等做加权排序。

它的价值在于工程稳定性：query 结构化、按 query type 路由、多源召回、citation snowballing、LLM relevance judgement、加权 rerank。缺点也明确：query evolution 和 retrieval control 还偏静态，RefChain 深度有限，缺少像SPARBench/LitQA2 这样的系统评测闭环。

## 参考架构：
架构建议：

1. Query Parser
    抽取：主题、方法、任务、数据集、领域、时间、venue、作者、结果偏好。
    输出 JSON，不直接让 LLM 自由发挥。

2. Stage-aware Query Generator
    生成 3 类 query：
    - broad_query：保证召回，尽量不加过细约束。
    - constraint_query：方法/数据集/venue/时间等精确约束。
    - citation_seed_query：用于找种子论文，后续走引用链。

3. Multi-source Retrieval
    至少接 OpenAlex 或 Semantic Scholar。建议优先：
    - Semantic Scholar：相关性、citation、references 信息更方便。
    - OpenAlex：兜底、免费、覆盖广。

4. One-hop RefChain
    只对 top seed papers 做一跳 references/citations 扩展。
    不做多跳，容易噪声爆炸、延迟高。SPAR 也倾向单层 RefChain 控制噪声。

5. Two-stage Rerank
    第一阶段便宜打分：
    - BM25/embedding 相似度
    - title/abstract constraint match
    - citation count
    - year recency
    - venue/field match
    - 是否来自 RefChain

    第二阶段只对 top 30 或 top 50 用 LLM judge：
    - Highly relevant
    - Partially relevant
    - Not relevant
    - 给出匹配到的 query 条件和一句 evidence

6. Final Rank
    建议公式：
    score = 0.45 * relevance + 0.20 * constraint_match + 0.15 * citation_or_authority + 0.10 * recency + 0.10 * diversity
    final_score = 0.35 * LLM_relevance + 0.25 * must_have_coverage + 0.15 * topic_match + 0.10 * graph_score + 0.10 * citation/recency+ 0.05 * diversity

    如果 query 明确说“latest”，提高 recency；如果说“seminal/classic/influential”，提高 citation；如果说“survey”，提高 publication type / title abstract 中 survey signals。
    总之不由单一的条件决定得分，由LLM和硬约束共同决定。

7. Structured Output
    这部分占 10%，但容易拿分。输出：
    - Top papers table
    - Highly relevant / partially relevant 分区
    - 每篇论文：title、year、authors、venue、citation、reason、matched constraints
    - 一个“关系图”文本结构：主题 -> 方法 -> 代表论文
    - 一个 brief synthesis：当前方向有哪些方法簇、缺口是什么

创新点说法：

- 预算感知：不是所有候选都调用 LLM，先 cheap rerank，再 selective LLM judge。
- 阶段感知查询分解：初检索少拆分保召回，重排阶段再使用细粒度约束。
- 单跳 RefChain + 噪声门控：只扩展高置信 seed 的引用网络，并用 constraint gate 过滤。
- 多目标排序：相关性、权威性、时效性、多样性统一建模。
- 结构化科研综述输出：不只是 list papers，而是按研究意图组织结果。

### 三个创新点
1. Query Constraint Graph
不要只把 query 解析成关键词列表，而是解析成一个“约束图”：

核心主题: non-stationary reinforcement learning
硬约束: UCB, value-based
软约束: non-stationary MDPs, dynamic regret
排序偏好: relevance > recency > citation
排除倾向: generic RL applications

然后检索和排序都围绕这个约束图做。
这比普通 query rewriting 更强，因为它区分了：

- topic match
- method match
- dataset match
- venue/year match
- must-have vs nice-to-have

专家评分时可以说：系统不是简单改写 query，而是把复杂学术查询转成可执行的多维约束结构。
2. Budget-Aware Adaptive Search
不是固定搜 6 个 query、固定调用 LLM，而是根据预算动态决策：

- 第一轮：OpenAlex + Semantic Scholar 检索
- 如果候选数少：扩展 query
- 如果候选数多但约束命中低：生成更精确 query
- 如果 top 结果置信度低：才调用 LLM judge
- 如果已经有足够高置信 top20：停止

这可以写成创新点：基于候选池置信度的早停机制。
这和 ADORE / SciRAG 的 adaptive retrieval 思路一致：ADORE 用 retrieval-grounded feedback 做迭代扩展，报告在 TREC DL 上 nDCG@10 相比 BM25 提升 24.5%；SciRAG 的 adaptive controller 在
SciFact/PubMedQA/ScholarQA-CS 上分别提升 +2.8 / +9.3 / +11.3（Bigdeli et al., 2026; Ding et al., 2025 - Paper Lantern, 116 papers explored）。

3. Selective LLM Reranking
不要逐篇生成长解释。先规则分筛 top40，再只对 top20 或 top30 做 LLM 三分类：

{
"label": "highly_relevant | partially_relevant | irrelevant",
"matched_constraints": ["UCB", "non-stationary RL"],
"missing_constraints": ["value-based"],
"reason": "..."
}

这就是你的核心提分点：LLM 只处理最难的候选判断。
这个方向也有研究支撑：Rerank-Before-You-Reason 说明 moderate reranking 可以用更低 token 成本达到接近高 reasoning agent 的效果；CoRanking 报告降低约 70% ranking latency，同时比 pointwise
baseline 有 NDCG@10 增益（Sharifymoghaddam and Lin, 2026; Liu et al., 2025 - Paper Lantern）。

### 未来方向
深入研究的方向：Graph-aware retrieval

1. Query Constraint Graph
    把用户 query 解析成多维约束图：
    - 主题
    - 方法
    - 任务
    - 数据集
    - 时间
    - venue
    - must-have / nice-to-have / negative constraints

2. Paper Citation Graph
    用 OpenAlex + Semantic Scholar 建一个局部或领域级 citation graph：
    - paper node
    - author/venue/year/citation_count metadata
    - reference/citation edges
    - title/abstract embeddings
    - optional: citation context / field-of-study

3. Graph-aware Retrieval + Learned Reranking
    不是只看 title/abstract match，而是同时看：
    - 论文本身是否匹配 query
    - 它引用了哪些相关论文
    - 哪些相关论文引用它
    - 它是否处在某个方法簇/领域簇中心
    - 它是否连接 query 中多个子主题

## Step1.论文获取
第一步获取论文是后续的基础，必须做到宁杀错勿放过，把相关的论文都收集过来。这样后续才能在此基础上进行筛选排序，否则巧妇难为无米之炊，后续筛选排序模型再好也不行。

获取论文的手段就是使用各个平台的API,目前知道的有Semantic Scholar、OpenAlex、arxiv以及谷歌搜索。如果不只是计算机领域的论文，还应该加上Pubmed这样的网站。

具体应该怎么使用这些api，除了这些网站的高级搜索，第一步是要将用户的query转化成搜索的关键词，也就是搜索引擎看到的内容是什么。这一步需要通过推理能力更强的**闭源商用大模型**理解用户query，并且合理地扩展。(question：为什么不能用微调的开源小模型，个人思考：小模型本身并不包含相关的知识，这也不是一种有范式地处理流程，需要使用到大模型更多的知识与推理能力)

```text
user query --> key words
```
**这一步具体怎么做呢？**

给定一个prompt,使用大模型对query进行扩充
### 检索方式
asta-paper-finder的方式宽而浅，pasa和SPAR的递归搜索树的方式深而窄，两种方式可以结合。如果只靠 dense retrieval，召回不足；如果沿引用链扩展太深，噪声爆炸
### Prompt

### paper metadata
这一步中，获取的论文都应该包含哪些源数据

## Step2.筛选排序
### 粗筛
考虑到LLM成本，对于检索得到的大量的论文，先进行一步出筛，使用一个rerank模型：[cohere](https://docs.cohere.com/docs/rerank)。Cohere是一家专注于企业级AI的公司，而Rerank模型是它提供的“搜索和检索”解决方案中的关键一环，这个reranker模型目前是收费的，虽然并不贵，但是可以寻求体该，根据一个[榜单](https://github.com/agentset-ai/reranker-eval),Qwen3-reranker-8B可以做替代。

### 精确筛选与排序

[pasa](https://arxiv.org/abs/2501.10120)论文4.3节中有一句话：“Additionally, the token probability of the decision token can be used to rank search results.”也就是说,这个二分类任务的最后输出的token对应的概率，可以作为相关性来排序。

SPAR论文训练模型输出 0-1 relevance score。但是其不是像 PaSa 那样用 AutoScholarQuery 做 SFT/RL 训练，而是用现成 LLM + prompt + 多 agent 流程来做检索、判断和重排

## Step3.结构化输出

## Step4.Token感知与检索效率

### AstaBench
AstaBench 成本计算机制：
Step 1 — Inspect 记录原始用量：Inspect 评测框架（UK AI Safety Institute开发）会在 Agent 执行过程中记录所有 LLM 调用的 token 使用量（input/output tokens），但不记录价格。HAL benchmark 也用了类似方式。

Step 2 — agent-eval 转换为美元价格：这是 AstaBench 的核心创新。agent-eval 工具包将 Inspect 记录的 token 数量映射为 标准化的美元成本，使用的是一个 冻结的 litellm 价格快照（frozen snapshot）。
Inspect框架 记录每一次LLM调用的 model usages

传统 benchmark 只看准确率（accuracy），忽略了成本。AstaBench 认为这是严重缺陷：

- 任何 Agent 可以把一个问题重复跑 100 次然后投票，准确率会上去
- 但成本会是 100 倍
- 如果不报告成本，这种策略看起来像"技术进步"，实际上是"烧钱"

所以 AstaBench Leaderboard 显式标注每个 Agent 的成本，并用 Pareto 前沿（Pareto Frontier）展示"在给定成本下能达到的最佳性能"，而不是只看谁分数最高。