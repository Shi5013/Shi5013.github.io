---
layout: post
title: paper-search-agent prompt
tags: record
math: false
date: 2026-7-5 12:30 +0800
---

paper search agent所用到的智能体解析。

## 1.[PaSa](https://github.com/bytedance/pasa)
这个的prompt是最简单的，一共是4个。

### 1.1 generate_query
第一个是生成query,目的是将用户模糊的查询转化为精准的搜索关键词，其中关键设计是"互斥性"和偏好综述。
```json
"Please generate some mutually exclusive queries in a list to search the relevant papers according to the User Query. Searching for survey papers would be better.
User Query: {user_query}"
```
实际例子：
```text
用户输入："transformer models"
生成结果：["transformer architecture survey", "attention mechanism review", "transformer applications NLP"]
```

### 1.2 select_section
分析已经找到的论文的章节结构，预测哪些章节包含最相关的引用。
```json
"You are conducting research on `{user_query}`. You need to predict which sections to look at for getting more relevant papers. 
Title: {title}
Abstract: {abstract}
Sections: {sections}"
```
输入当前论文的标题、摘要和章节列表，LLM需要判断：这篇论文的哪些章节最可能引用到与用户查询高度相关的其他论文，这是一个上下文感知的决策过程。


### 1.3 get_selected
严格判断检索到的论文是否真正符合用户需求
```json
"You are an elite researcher in the field of AI, conducting research on {user_query}. 
Evaluate whether the following paper fully satisfies the detailed requirements of the user query and provide your reasoning.
Ensure that your decision and reasoning are consistent.

Searched Paper:
Title: {title}
Abstract: {abstract}

User Query: {user_query}

Output format: Decision: True/False
Reason:... 
Decision:"
```
- 角色扮演："elite researcher in the field of AI"，设定专家身份提升判断质量
- 严格标准："fully satisfies" - 要求完全满足而非部分相关
- 一致性要求：决策和推理必须逻辑一致
- 结构化输出：强制要求 True/False 决策 + 推理过程

### 1.4 get_value

与`get_selected`几乎相同，但用于不同的阶段。
```json
{
    "get_value": "You are conducting research on {user_query}. Evaluate whether the following paper fully satisfies the detailed requirements of the user query and provide your reasoning. Ensure that your decision and reasoning are consistent.\n\nSearched Paper:\nTitle: {title}\nAbstract: {abstract}\n\nUser Query: {user_query}\n\nOutput format: Decision: True/False\nReason:... \nDecision:"
}
```
### 1.5 总结
设计模式分析：首先是应用了思维链，每个提示词都要求LLM展示推理过程，提高判断准确性，其次是使用渐进式过滤，从宽泛检索-章节选择-初步筛选-严格确认层层递进，然后好似上下文注入，使用`{user_query},{title},{abstract}`等占位符，实现动态内容填充。最后是用“Output format”强制结构化输出，便于程序解析。

### 1.6 使用
```python
def search(self):
    # 1. 构造提示词
    prompt = self.prompts["generate_query"].format(user_query=self.user_query).strip()
    
    # 2. 调用LLM生成查询
    queries = self.crawler.infer(prompt)
    
    # 3. 解析和提取查询
    queries = [q.strip() for q in re.findall(
        self.templates["search_template"],  # r"Search\](.*?)\["
        queries, flags=re.DOTALL
    )][:self.search_queries]  # 默认取前5个
```
## 2.[SPAR](https://github.com/xiaofengShi/SPAR)
### 2.1概览
本项目的提示词放在instruction.py中，文件1254行，包含28个prompt模板，按照功能可以分为6大类，所有模板使用python的`str.format(**kwargs)`方式进行变量插值。

总体架构
```tetx
instruction.py
├── 1. 查询理解与分析 (4 个)
├── 2. 查询扩展与融合 (12 个) ← 最大类别
├── 3. 关键词提取 (4 个)
├── 4. 相关性评分 (5 个)
├── 5. 重排序 (1 个)
└── 6. 辅助/通用 (2 个)
```
### 2.2 查询理解与分析
### 使用举例
