# Day 4 报告：检索 BBC 新闻档案并给出证据

> 命令、路径、数字均为本机真实运行结果。

## 1. 本日问题

- 里程碑：day-04
- 学生或小组：刘语桐
- 使用者：需要快速定位档案证据、同时不接受无来源答案的新闻编辑研究助理。
- 真实输入：Kaggle BBC News Archive 的 `tfidf_dataset.csv`，2,225 篇真实新闻文章。
- 需要的输出：一个可运行的档案检索工具——对问题返回 top-k 真实文章及稳定编号；只有证据支持时才回答，否则明确拒绝。
- 与使用者最相关的错误：把不相关的文章排在前面、或对证据不足的问题给出看似合理但没有文章支持的答案。
- 本日产品边界：数据只当作资料，不当作指令；证据不足必须拒绝；模型回答不能替代来源核对。

## 2. 真实数据

- 所有者/发布者：Kaggle `dimasmunoz/bbc-articles-cleaned`（BBC News 档案的清洗版）。
- 标题：BBC Articles Cleaned（tfidf_dataset.csv）。
- 原始 URL：https://www.kaggle.com/datasets/dimasmunoz/bbc-articles-cleaned
- 许可标签或使用许可：Kaggle 公开数据集；本仓库仅用于本地教学，不重新分发原始 CSV（已写入 `.gitignore`）。
- 下载/取得日期：2026-08-21。
- 预期文件与结构：`data/raw/tfidf_dataset.csv`，列 `text`（正文）+ `category`，共 2,225 行。
- 检查命令：`python app.py --check-data`
- 实际检查结果：

  ```text
  REAL DATA CHECK PASSED
  rows: 2225
  categories: ['business', 'entertainment', 'politics', 'sport', 'tech']
  counts: {'business': 510, 'entertainment': 386, 'politics': 417, 'sport': 511, 'tech': 401}
  ```

- 已知缺失、偏差或限制：正文是停用词过滤后的文本（不是完整句子）；档案只覆盖 2004–2005 年的新闻，超出这个年代的问题没有依据；存在近乎重复的文章（如 `article-1860` 与 `article-2224` 只有一处措辞不同）。

## 3. 可复现运行

```powershell
# 当前目录：student-work/day-04-bbc-search（本仓库根目录）

# 安装
python -m pip install -r requirements.txt

# 数据检查
python app.py --check-data

# 单元测试
python -m unittest discover -s tests -v

# 检索评估（3 个固定问题）
python app.py --evaluate

# 检索单个问题
python app.py --question "When and at what price was Sony's PSP expected to launch in Europe?" --top-k 4
```

- 数据检查输出：见第 2 节，`REAL DATA CHECK PASSED`。
- 单元测试输出：4 个测试全部 OK（`test_specific_words_rank_matching_article`、`test_extracts_stable_citations`、`test_rejects_unretrieved_citation`、`test_requires_at_least_one_citation`）。
- 检索评估输出：`retrieval_recall_at_4=3/3`（见第 4 节）。

## 4. 基线与候选

### 简单基线

- 方法：返回空结果，或只做「直接关键词包含匹配」——问题里的每个词都必须原样出现在文章里才算命中，且无法排序（命中即平级）。
- 为什么足够简单：不学任何表示，只按字面判断；它是「最笨但最可解释」的对照，用来回答「复杂一点的检索到底有没有用」。
- 命令：无单独脚本，作为对照在本报告中描述（starter 未内置该基线）。
- 结果/失败点：对同义改写会失败。例如评估问题用的是 "visa rules"，但目标文章 `article-0000` 原文写的是 "visa regulations" 和 "red tape"，不含 "rules"——直接关键词匹配会漏掉它，而 TF-IDF 能通过 "visa/regulations/musicians" 的加权相似度把 `article-0000` 排到第 1。

### 候选方法

- 学生完成的核心改动：在 `retriever.py` 的 `ArchiveIndex.search` 中完成 TODO——① 用拟合好的 `self.vectorizer` 把问题转成向量；② `cosine_similarity` 计算该向量与 `self.matrix`（2,225 篇文章向量）的相似度；③ 按分数从大到小排序；④ 返回前 top_k 个 `(Article, float)`。
- 保持不变的数据、划分、指标或参数：同一份 2,225 篇真实文章、同一个 `TfidfVectorizer(ngram_range=(1,2), stop_words="english", min_df=1)`、同一套 3 个评估问题、同一 top_k=4——只有检索方法不同。
- 命令：`python app.py --evaluate`
- 结果：

  ```text
  PASS musician_visas: ['article-0000', 'article-2076', 'article-1483', 'article-0096']
  PASS psp_launch:     ['article-2176', 'article-1837', 'article-2080', 'article-1946']
  PASS phone_virus:    ['article-2121', 'article-2224', 'article-1917', 'article-2124']
  retrieval_recall_at_4=3/3
  ```

| 项目 | 基线（关键词包含） | 候选（TF-IDF + 余弦） | 含义 |
| --- | ---: | ---: | --- |
| 主指标（recall@4） | 未通过同义改写 | 3/3 | 候选能找到目标文章 |
| 与使用者最相关错误 | 漏掉同义表达的文章 | 同名实体混淆（见第 5 节） | 候选换了一种错误方式 |

公平性说明：两者使用同一份 2,225 篇真实文章、同一评估问题、同一 top_k=4。区别只在「表示问题与文章的方式」：基线只看字面是否包含，候选用 TF-IDF 加权 + 余弦相似度排序。候选的一个诚实瑕疵：`phone_virus` 只命中了 `article-2224`，没进前 4 的 `article-1860` 与其近乎逐字重复（一个写 "phone bugs appear"、一个写 "phone viruses appear"），因此命中其一已经足够——这不构成真正漏检。

## 5. 一个真实失败案例

- 样本位置/编号：检索问题 "What is the current weather forecast on Mars?"，返回第 1 名 `article-0372`（entertainment）。
- 真实结果：`article-0372` 讲的是 Mötley Crüe 吉他手 **Mick Mars** 被前女友起诉的娱乐新闻，与「火星天气」毫无关系。
- 系统输出：`[article-0372] entertainment score=0.231`，是 top-1；其余结果是油价/天气类文章（如 `article-0690` 讲油价受天气影响）。
- 可以观察到什么：TF-IDF 把查询里的 "Mars" 和 "weather" 分别字面匹配到 "Mick Mars" 和油价新闻里的 "weather"，于是给无关文章打了最高分（0.231，超过 0.08 阈值，不会被拒绝）。
- 说明的限制：TF-IDF 只做词频统计，**不理解词义**，无法区分「火星（Mars）」与「Mick Mars」这个同名实体，也无法判断文章是否真的回答了问题。
- 不能证明什么：一个坏例子不能说明 TF-IDF 整体无用——它在 3 个固定评估问题上都正确命中；它说明的是「分数高 ≠ 答案对」。
- 下一项最小检查：对 top 结果加一层「同实体消歧」或让有依据回答模型先判断「检索结果是否真的在回答这个问题」，证据不足就拒绝而不是照单全收。

## 6. 智能体与学生工作边界

- 智能体提出/生成/修改了什么：复制 starter 到 `day-04-bbc-search`，解压并验证 2,225 行真实数据，完成 `retriever.py` 的 `ArchiveIndex.search`（TODO），补齐 `app.py` 的 `--check-data` 命令，运行数据检查、测试、评估与多个问题检索，编写 `README.md`、`report.md`、`submission.json`。
- 学生怎样核对文件、来源、输出、测试和 diff：运行 `python app.py --check-data` 确认 2,225 行与 5 个类别；运行 `python -m unittest discover -s tests -v` 看 4 个测试是否 OK；运行 `python app.py --evaluate` 看 3/3 与各问题命中情况；`git diff` 确认只改了 `retriever.py` 的 `search` 与 `app.py` 的 `--check-data`，未动测试、数据或评估用例。
- 学生修改或拒绝了什么建议：接受「`phone_virus` 只命中一篇文章」与「火星天气返回 Mick Mars」这两个真实结果并如实记录，而不是改评估用例或调 `min-score` 硬凑全对。
- 每名成员能独立解释的代码或证据：`search` 里 `self.vectorizer.transform([question])` 与 `self.matrix` 为什么同源、`cosine_similarity(...)[0]` 返回什么、`argsort()[::-1]` 为什么是降序；`--check-data` 怎么用 `load_articles` 的校验把 2,225 行和 5 个类别钉死。

## 7. 结论与限制

在同一份 2,225 篇真实文章、同一评估问题、同一 top_k=4 下，候选（TF-IDF + 余弦相似度 + top-k）比直接关键词匹配更有用：三个固定问题全部命中目标文章（`retrieval_recall_at_4=3/3`），并能处理 "visa rules" 与 "visa regulations" 这类同义改写。但候选不是「理解语义」：它把「火星」错配成摇滚乐手 Mick Mars（top-1 score 0.231），也会把「2020 美国大选」匹配到 2005 英国大选文章且分数 0.119 仍高于拒绝阈值 0.08——说明「分数高」不等于「答案对」，单纯靠分数阈值不足以拒绝这类年代或实体的错配。数据上，档案只覆盖 2004–2005 年的 BBC 新闻，超出年代或主题的问题本就无据可依。用途上，该工具只做「候选资料召回」，最终答案必须人工核对原文，不能替代来源核实。下一项最小改进是为 top 结果加实体消歧或「是否真在回答问题」的二次判断。

## 8. 提交复核

- [ ] README 从新环境可以开始运行
- [ ] 数据检查、测试和主程序重新运行
- [ ] 报告数字与保存输出一致
- [ ] `presentation.pptx` 在 3 分钟内讲完（暂未生成）
- [ ] `submission.json` 路径正确
- [ ] 无密钥、大数据、私人信息、虚拟环境或缓存
- [ ] GitHub 网页复查并邮件发送 URL
