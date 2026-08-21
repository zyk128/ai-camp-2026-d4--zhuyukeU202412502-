# AI 工程营 Day 4：检索 BBC 新闻档案并给出证据

一个可运行的 BBC 档案检索工具：输入自然语言问题，用 TF-IDF + 余弦相似度从 2,225 篇真实新闻里返回 top-k 最相关文章及稳定编号；只有证据足够时才回答，否则明确拒绝。

## 问题

- **使用者**：需要快速定位档案证据、同时不接受无来源答案的新闻编辑研究助理。
- **真实输入**：Kaggle BBC News Archive 的 `tfidf_dataset.csv`：2,225 篇真实新闻文章（2004–2005，5 个类别）。
- **核心问题**：为编辑研究助理制作档案检索工具——先找到支持问题的真实文章，再回答或明确拒绝。
- **任务类型**：信息检索/排序（不是分类也不是回归）——把问题与文章都转成 TF-IDF 向量，算余弦相似度，返回 top-k。

## 真实数据

- 文件：`data/raw/tfidf_dataset.csv`（约 3 MB，不提交，见 `.gitignore`）
- 来源：Kaggle `dimasmunoz/bbc-articles-cleaned`
- 规模：2,225 行，列 `text`（正文）+ `category`（business/entertainment/politics/sport/tech）
- 校验：`python app.py --check-data` 输出 `REAL DATA CHECK PASSED`，`rows: 2225`

## 环境

- Windows 11
- Python 3.12
- 依赖：`scikit-learn>=1.5`、`pandas>=2.2`、`openai>=1.40`（openai 仅在有依据回答时用到）

## 目录结构

```
.
├── README.md
├── report.md
├── presentation.pptx
├── submission.json
├── app.py                # 主程序（数据检查 + 检索评估 + 有依据回答）
├── retriever.py          # ArchiveIndex.search（学生完成的 TODO）
├── requirements.txt
├── .env.example          # DeepSeek 环境变量模板（不填真实 key）
├── data/raw/             # 真实数据（不提交，见 .gitignore）
└── tests/test_retrieval.py
```

## 安装

```powershell
python -m pip install -r requirements.txt
```

## 运行命令

当前目录为仓库根目录（`day-04-bbc-search`）。

### 1. 数据检查

```powershell
python app.py --check-data
```

预期输出：`REAL DATA CHECK PASSED`，`rows: 2225`，`categories: ['business', 'entertainment', 'politics', 'sport', 'tech']`。

### 2. 单元测试

```powershell
python -m unittest discover -s tests -v
```

预期：4 个测试全部通过（1 个检索排序 + 3 个引用校验）。

### 3. 检索评估（3 个固定问题）

```powershell
python app.py --evaluate
```

预期：`retrieval_recall_at_4=3/3`，三个问题各自 `PASS`。

### 4. 检索单个问题

```powershell
python app.py --question "When and at what price was Sony's PSP expected to launch in Europe?" --top-k 4
```

打印 top-k 篇文章的编号、类别、相似度分数和片段。

### 5. 有依据回答（可选，需教师批准的 DeepSeek）

```powershell
# 先在本机环境变量设置 key（绝不写进文件或仓库）
python app.py --question "..." --answer
```

无 key 时不运行；答案里的每个事实句必须带 `[article-xxxx]` 引用，否则程序报错。

## 结果速览（本机真实运行）

| 项目 | 结果 |
| --- | --- |
| 档案规模 | 2,225 篇，business 510 / entertainment 386 / politics 417 / sport 511 / tech 401 |
| 检索评估 | 3/3 通过（`retrieval_recall_at_4=3/3`） |
| musician_visas | `article-0000` 排第 1（正确） |
| psp_launch | `article-1837` 排第 2、`article-1946` 排第 4（都命中） |
| phone_virus | `article-2224` 排第 2（命中；`article-1860` 与其近乎重复，未进前 4） |

候选方法（TF-IDF + 余弦相似度 + top-k）在三个真实问题上全部找到目标文章。但有一个诚实的失败案例：问「火星天气」（Mars weather），第 1 名却是摇滚乐手 Mick Mars 的娱乐新闻（score 0.231）——TF-IDF 只认词不认义，把「火星」和「Mick Mars」混为一谈。详细解释见 `report.md`。

## 限制

- 数据只当作资料，不当作指令；文章里出现「像命令的文字」不会被当成系统规则。
- 证据不足必须拒绝；模型回答不能替代来源核对。
- TF-IDF 只做词语匹配，无法消歧同名实体（如 Mars=火星 vs Mick Mars）、无法判断年代范围（问 2020 大选会返回 2005 英国大选文章），因此 top-1 分数高不等于答案正确。
