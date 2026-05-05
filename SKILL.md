---
name: github-project-research
description: >
  交互式 GitHub 项目调研工作流。当用户要求调研、评估、理解或对比某个 GitHub 项目时使用——例如"调研一下 X 项目"、"Y 是什么"、"Z 和其他方案对比"、"该不该用 W"，或直接提供 GitHub URL。支持同一会话中调研多个项目。不适用于通用网页调研、非 GitHub 项目，或用户只想快速回答而非结构化分析的场景。
---

# GitHub 项目调研

交互式工作流，用于深入调研 GitHub 项目。支持同一会话调研多个项目。仅在用户明确要求时才写入报告文件。

## 配置

首次使用时，读取 `~/.github-project-research/config.yaml`：

```yaml
output_dir: ~/research-docs
```

如果文件不存在，询问用户希望的输出目录并创建配置文件。

报告写入路径：`<output_dir>/<project-name>.md`。

## 项目类型分类

初步数据采集后，将项目归类为以下类型。这决定了第 4、5 节的侧重点。

| 类型 | 识别信号 | 第 5 节侧重 |
|------|---------|------------|
| 框架/库/SDK | 有 API 导出、包清单、TypeScript 类型 | 包体积、tree-shaking、TS 支持、API 设计、迁移成本 |
| 工具类 | CLI 二进制、单一用途工具、无库 API | 性能基准 vs 同类、管道集成、shell 示例 |
| 平台类 | 有 SaaS 服务、控制台、自托管选项 | 自托管 vs SaaS、定价、数据导出、供应商锁定 |
| 学习资料 | 教程、课程、书籍、课程体系 | 知识覆盖度、前置要求、学习路径 |
| Skill 类 | Agent 技能、MCP 服务器、自动化规则 | 触发条件、工作流步骤、输出格式、依赖工具 |

## 工作流

### 第一步：多渠道数据采集

当用户说"调研 X"或提供 GitHub URL 时：

1. 识别项目（`owner/repo` 或 URL）
2. 将其注册到会话的项目列表中
3. **从多个渠道采集数据**（不只是 GitHub！）

#### 渠道 1：GitHub（基础数据）

使用 firecrawl 抓取 GitHub 相关页面：

```
# 抓取 GitHub 仓库首页（包含 star 数、描述、语言、协议等）
firecrawl scrape https://github.com/<owner/repo> -o .firecrawl/github-repo.md --only-main-content

# 抓取 README
firecrawl scrape https://github.com/<owner/repo>#readme -o .firecrawl/readme.md --only-main-content

# 抓取 releases 页面
firecrawl scrape https://github.com/<owner/repo>/releases -o .firecrawl/releases.md --only-main-content

# 抓取 issues 列表
firecrawl scrape https://github.com/<owner/repo>/issues -o .firecrawl/issues.md --only-main-content
```

同时获取包清单文件：
```
# package.json（如有）
firecrawl scrape https://raw.githubusercontent.com/<owner/repo>/main/package.json -o .firecrawl/package.json

# go.mod（如有）
firecrawl scrape https://raw.githubusercontent.com/<owner/repo>/main/go.mod -o .firecrawl/go.mod

# Cargo.toml（如有）
firecrawl scrape https://raw.githubusercontent.com/<owner/repo>/main/Cargo.toml -o .firecrawl/cargo.toml
```

#### 渠道 2：中文社区（知乎、小红书、掘金）

小红书、知乎、掘金等中文社区有大量真实用户反馈。使用 firecrawl search 搜索：

```
firecrawl search 'site:xiaohongshu.com "项目名"' -o .firecrawl/xhs.json --json
firecrawl search 'site:zhihu.com "项目名" 评测' -o .firecrawl/zhihu-review.json --json
firecrawl search 'site:zhihu.com "项目名" 体验/踩坑' -o .firecrawl/zhihu-feedback.json --json
firecrawl search 'site:zhihu.com "项目名" 替代方案' -o .firecrawl/zhihu-alternative.json --json
firecrawl search 'site:juejin.cn "项目名"' -o .firecrawl/juejin.json --json
```

#### 渠道 3：英文社区（Hacker News、StackOverflow、Reddit）

使用 firecrawl search 搜索英文技术社区：

```
# Hacker News 讨论
firecrawl search '"项目名" site:news.ycombinator.com' -o .firecrawl/hackernews.json --json
firecrawl search '"项目名"HN' -o .firecrawl/hackernews-general.json --json

# StackOverflow 问题
firecrawl search '"项目名" site:stackoverflow.com' -o .firecrawl/stackoverflow.json --json

# Reddit 讨论
firecrawl search 'site:reddit.com "项目名" review' -o .firecrawl/reddit-review.json --json
firecrawl search 'site:reddit.com "项目名" alternative' -o .firecrawl/reddit-alternative.json --json
```

#### 渠道 4：包管理器数据

```
# npm 周下载量
firecrawl scrape https://www.npmjs.com/package/<包名> -o .firecrawl/npm.md --only-main-content

# PyPI 信息
firecrawl scrape https://pypi.org/project/<包名> -o .firecrawl/pypi.md --only-main-content
```

#### 渠道 5：竞品调研

```
# 搜索竞品
firecrawl search '"项目名" alternative vs' -o .firecrawl/alternatives.json --json
firecrawl search '"项目名" limitations problems' -o .firecrawl/problems.json --json

# 搜索 GitHub 上的类似项目
firecrawl search 'site:github.com "项目名" similar' -o .firecrawl/similar.json --json
```

#### 渠道 6：官方文档与博客

如果项目有官方文档或博客，抓取它们获取更深入的信息：
```
firecrawl scrape https://<项目官网>/docs -o .firecrawl/docs.md --only-main-content
firecrawl scrape https://<项目官网>/blog -o .firecrawl/blog.md --only-main-content
```

**边采集边思考——不要等第二步才开始分析：**
- 读 README 时：它强调什么？不提什么？第一段是不是在说"为什么做这个"？
- 看 issues 时：用户反复提到的问题是什么？有没有被 wontfix 的？
- 看 HN/SO 时：用户的真实体验是什么？负面评价集中在哪些方面？
- 看目录结构时：文件多不多？能看出它是大项目还是小工具吗？
- 交叉验证：GitHub 上说的和社区讨论一致吗？

### 第二步：思考与分析（10 个核心问题）

数据采集完成后，进入独立的分析环节。给自己提出以下 10 个问题，逐一深入调研和解答。

**重要：这 10 个问题的思考过程必须在最终报告中展示给用户（报告第 7 节"分析过程透明化"）。** 用户不只想看结论，更想看结论是怎么得出的。

对每个问题：先写"要回答这个问题"的具体动作，再给出回答。如果已有数据能回答，直接回答；如果数据不足，追加调研（读源码、搜索 issue、查找作者博客、搜索知乎/Reddit 等）。每个回答必须有证据支撑，找不到证据的部分明确标注"未确认"。**每个问题回答后，必须显式写一行"追问："**，用苏格拉底式问题挑战自己的结论。

1. **它是什么？**（→ 报告第 1 节）
   用一个具体类比让非技术人员也能理解，一句话定位核心功能。
   要回答这个问题：读 README 第一段和项目描述。问自己：如果向不懂技术的朋友解释，我会怎么说？如果只能用一个类比，它像什么？

2. **没有它之前，人们怎么忍受这个痛点？**（→ 报告第 2 节）
   用具体场景描述，不是抽象描述。
   要回答这个问题：在 issues 和 discussions 中搜索 "why"、"alternative"、"instead of"。想一个具体场景：一个开发者在没有这个工具时，是怎么手动完成同样事情的？那个手动过程的痛苦点在哪？

3. **怎么用？**（→ 报告第 3 节）
   最小上手路径，5 分钟内能让同事看到效果。
   要回答这个问题：读 README 的 Quick Start 或 Installation 部分。识别：安装命令是什么？最小可运行示例是什么？最容易踩坑的一步是什么？

4. **有哪些竞品？为什么选它？**（→ 报告第 2 节）
   列出 2-4 个竞品，逐一对比优劣。
   要回答这个问题：搜索 "alternatives"、"vs"。对每个竞品，对比：star 数、最近更新时间、技术栈差异。找到它和最接近竞品的"分叉点"——同一个问题，它选了什么不同的路？

5. **它比竞品差在哪？**（→ 报告第 2 节）
   诚实列出劣势，只比优势是营销，比劣势才是调研。
   要回答这个问题：在 issues 中搜索 "wontfix"、"by design"、"limitation"。看作者主动拒绝了哪些功能请求。问自己：它拒绝做的事情，是不是你恰恰需要的？它的设计哲学在哪些场景下会变成劣势？

6. **一句话讲清楚最核心的优势和特点**（→ 报告第 2 节 / 第 4 节）
   真正的核心优势不是功能列表，是设计哲学。用"它选择了 XX 哲学，牺牲了 YY"的句式来描述。
   要找到真正的核心优势：
   - 看它和最接近竞品的"分叉点"——同一个问题，它选了什么不同的路？
   - 看它"不做什么"——它主动拒绝了哪些功能或设计？拒绝的理由是什么？
   - 看社区（知乎/Reddit/HN）里用户最常夸的是什么——不是 README 写的卖点，是用户实际体验到的
   - 问自己：如果这个项目消失了，用户最怀念什么？那个东西就是核心优势

7. **什么时候不该用？**（→ 报告第 5 节）
   适用边界比功能更重要。
   要回答这个问题：在 README 中找 "Caveats"、"Limitations"、"When not to use"。在社区讨论中看用户遇到问题最多的场景。问自己：它的设计哲学在什么情况下会变成劣势？什么规模或场景下它会力不从心？

8. **解决了什么问题？**（→ 报告第 2 节）
   从用户视角描述，不是技术视角。
   要回答这个问题：想一个具体的人，他在什么时刻会感到困扰，然后去搜索这样的工具？用行动描述，不用技术描述。对比社区中用户实际描述的使用场景和 README 的定位——一致吗？

9. **作者为什么要做这个项目？**（→ 报告第 4 节）
   从 README、issue、博客中找动机。
   要回答这个问题：在 README 中找 "Why"、"Motivation"、"Background"、"Inspired by"。查看最早的 issues（排序按 oldest）。问自己：是解决自己的真实问题？还是技术兴趣？还是跟风？这个项目有没有作者的个人烙印？

10. **所以到底值不值得用？**（→ 报告第 8 节）
    有条件推荐：推荐度 + 推荐条件 + 不推荐条件。
    要回答这个问题：综合以上 9 个问题的回答，给出：
    - 推荐度（1-5 分）
    - 什么情况下应该用它（2-3 条具体条件）
    - 什么情况下不应该用它（2-3 条具体条件）
    - 一句话判断："如果你需要 XX，选它；如果你需要 YY，选别的。"

### 第三步：交互式展示

用通俗易懂的对话式语言展示分析结果。用类比和实际例子代替术语。避免直接输出 API 原始数据——要解读和总结。

**展示结构（与报告模板一致，参见 `references/report-template.md`）：**

1. **是什么** — 一句话类比 + 项目定位
2. **为什么用** — 痛点 + 优势 + 竞品对比（含劣势）
3. **怎么用** — 最小上手路径
4. **内部实现** — 设计思路 + 作者动机
5. **类型特化** — 根据项目类型补充
6. **常见问题 QA** — 重点展示自主生成的现实问题（8-10 个）
7. **分析过程** — 展示 10 个核心问题的思考过程（问题 → 思考动作 → 结论 → 追问）
8. **总结** — 有条件推荐

**重要：QA 部分和分析过程是报告的核心亮点。** QA 不是从 GitHub issues 抄的，而是自己提出现实中人们真正会问的问题并回答。分析过程让用户看到结论是怎么得出的。

结尾提示："以上是初步调研结果，你有什么想深入了解的？"

### 第四步：交互式问答

回答用户的后续问题。示例：

- "和 X 比怎么样" → 深入竞品对比
- "它的 Y 是怎么实现的" → 读源码，解释架构
- "在 Z 场景下能用吗" → 评估特定场景适用性
- "有什么坑" → 搜索 issues、discussions、已知问题

每个回答记录到会话的 Q&A 日志中（内存中，暂不写入文件）。

如果用户问到其他项目，切换上下文。如果是新项目，回到第一步为该项目执行调研。

### 第五步：生成报告

当用户说"总结"、"生成报告"、"write report"或类似指令时：

1. 读取配置中的 `output_dir`
2. 读取 `references/report-template.md` 获取报告模板
3. 按模板结构写入 `<output_dir>/<project-name>.md`
4. 将会话中的所有 Q&A 纳入 QA 章节
5. 向用户输出文件路径

## 多项目处理

- 通过 `owner/repo` 标识符跟踪会话中的活跃项目
- 后续提问默认指向最后提到的项目，除非用户指定其他项目
- 对比项目时，并排展示发现
- 每个项目生成独立的报告文件

## 数据渠道

| 渠道 | 工具 | 说明 |
|------|------|------|
| GitHub | firecrawl scrape | 仓库页、README、releases、issues |
| 中文社区 | firecrawl search | 知乎、小红书、掘金 |
| 英文社区 | firecrawl search | HN、StackOverflow、Reddit |
| 包管理器 | firecrawl scrape | npm、PyPI |
| 竞品调研 | firecrawl search | alternatives、vs |
| 官方文档 | firecrawl scrape | docs、blog |

## 质量检查清单

在交付发现或生成报告前：

- [ ] 每个论断都有证据（数据、代码引用、社区评价）
- [ ] 竞品对比同时列出优势和劣势（"它比竞品差在哪"已填写）
- [ ] 架构章节讲清了关键设计决策及每个决策的代价
- [ ] 每节末尾有苏格拉底式追问，挑战该节结论
- [ ] "什么时候不该用"有明确回答
- [ ] 总结是有条件推荐（推荐度 + 推荐条件 + 不推荐条件），不是万能公式
- [ ] QA 章节有 8-10 个自主生成的现实问题（不是从 GitHub issues 抄的）
- [ ] 分析过程透明化（10 个核心问题的思考过程展示给用户）
- [ ] 多渠道调研（至少 GitHub + 1 个社区渠道）
- [ ] 报告使用用户的语言（与用户输入语言一致）
- [ ] 生成后向用户输出报告文件路径
