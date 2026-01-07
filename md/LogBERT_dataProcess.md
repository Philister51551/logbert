这是一个非常棒的复盘时刻！看着文件列表，我们就能把之前学到的所有理论知识和代码逻辑完全对应上了。

这份目录结构完整地展示了从**原始数据清洗**到**模型输入准备**的全过程。

下面我为你生成完整的 Markdown 教学文档。

---

# HDFS 日志异常检测：数据流转全景图

## 1. 文件清单与阶段对应表

这张表是你目录下所有文件的身份说明书。我们将整个流程划分为 **解析 (Parsing)** -> **映射 (Mapping)** -> **序列化 (Grouping)** -> **切分 (Splitting)** -> **词表构建 (Vocab)** 五个阶段。

| 文件名                      | 文件大小 | 所属阶段  | 核心作用                                                     | 内容形式                                     |
| :-------------------------- | :------- | :-------- | :----------------------------------------------------------- | :------------------------------------------- |
| **HDFS.log_structured.csv** | 2.3 GB   | 1. 解析   | **结构化日志表**。原始日志经过 Drain 解析后，提取出时间、内容、模板ID等字段的大表。 | CSV 表格 (含 `Content`, `EventId` 等列)      |
| **HDFS.log_templates.csv**  | 4.7 KB   | 1. 解析   | **模板总表**。列出了整个数据集中出现过的所有唯一日志模板。   | CSV 表格 (含 `EventTemplate`, `Occurrences`) |
| **hdfs_log_templates.json** | 743 B    | 2. 映射   | **ID 映射字典**。将复杂的 `EventId` (如 E5) 映射为简单的整数索引 (如 5)。 | JSON (`{"E5": 5, "E22": 22...}`)             |
| **hdfs_sequence.csv**       | 49 MB    | 3. 序列化 | **序列数据表**。按 `BlockId` 聚合后的结果，每一行代表一个 Block 的完整生命周期。 | CSV 表格 (`BlockId`, `EventSequence`)        |
| **train**                   | 190 KB   | 4. 切分   | **训练集**。纯正常数据的数字序列，供模型学习“正常语法”。(你设置了 n=4855，所以比较小) | 纯文本 (数字序列，空格分隔)                  |
| **test_normal**             | 21 MB    | 4. 切分   | **测试集(正常)**。用来测试模型会不会误报 (False Positive)。  | 纯文本 (数字序列，空格分隔)                  |
| **test_abnormal**           | 610 KB   | 4. 切分   | **测试集(异常)**。用来测试模型能不能抓到异常 (False Negative)。 | 纯文本 (数字序列，空格分隔)                  |
| **vocab.pkl**               | 466 B    | 5. 词表   | **LogBERT 专用词表**。Python 对象，包含特殊 Token (`[DIST]`, `[MASK]`) 和上述数字 ID 的最终映射。 | 二进制文件 (Pickle)                          |

---

## 2. 实例演示：一条日志的“前世今生”

为了让你彻底理解，我们追踪 **HDFS 中最经典的一条日志流程**。

**假设原始日志 (Raw Log) 是：**
> `081109 203615 148 INFO dfs.DataNode$PacketResponder: PacketResponder 1 for block blk_38865049064139660 terminating`

---

### 第一阶段：解析 (Parsing)
**目标：** 把人类读得懂的文字，变成计算机好处理的结构。
**输入：** 原始日志文件 `HDFS.log`
**处理逻辑：** 使用 **Drain** 算法，利用正则去除变量（如 `blk_388...` 变成 `*`）。

**产出文件：**
1.  **`HDFS.log_templates.csv`** 里会新增一行模板：
    *   *EventTemplate:* `PacketResponder * for block * terminating`
    *   *EventId:* `E5` (假设 Drain 给它分配的哈希 ID 是 E5)
2.  **`HDFS.log_structured.csv`** 里会存储这一行具体数据：
    *   `LineId`: 1001
    *   `Time`: 203615
    *   `Content`: ... terminating
    *   `EventId`: `E5`
    *   `ParameterList`: `['1', 'blk_38865049064139660']`

---

### 第二阶段：映射 (Mapping)
**目标：** 把字符串 `E5` 变成神经网络喜欢的数字 `5`。
**输入：** `HDFS.log_templates.csv`
**处理逻辑：** 根据模板出现频率排序，频率越高 ID 越小。

**产出文件：**
**`hdfs_log_templates.json`**：
```json
{
  "E5": 5,
  "E22": 22,
  "E11": 9
  ...
}
```
*(注：这里假设 E5 对应的数字是 5)*

---

### 第三阶段：序列化/分组 (Session Grouping)
**目标：** 把散落在不同时间的日志，按“业务逻辑”串起来。
**输入：** `HDFS.log_structured.csv` + `hdfs_log_templates.json`
**处理逻辑 (关键)：**
1.  代码提取每行日志的 `BlockId` (例如 `blk_388...`)。
2.  找到所有 `BlockId` 相同的日志。
3.  按时间排序，把它们的 EventId (数字版) 拼在一起。

**产出文件：**
**`hdfs_sequence.csv`**：
```csv
BlockId,                EventSequence
blk_38865049064139660,  [5, 22, 5, 5, 11]
```
*(这意味着这个 Block 经历了：E5 -> E22 -> E5 -> E5 -> E11 的过程)*

---

### 第四阶段：数据集切分 (Train/Test Splitting)
**目标：** 准备好喂给模型的最终粮草。
**输入：** `hdfs_sequence.csv` + `anomaly_label.csv` (标签表)
**处理逻辑：**
1.  查表得知 `blk_388...` 是 **Normal (正常)** 的。
2.  把它分到正常组。
3.  根据你的代码 `n=4855`，它可能被随机分到了训练集，也可能分到了测试集。

**产出文件 (假设被分到了 `train`)：**
**`train`** (纯文本文件，内容如下)：
```text
5 22 5 5 11
... (其他序列)
```
*(注意：这里已经没有 BlockId 了，只剩下纯粹的数字序列)*

---

### 第五阶段：词表构建 (Vocab Building)
**目标：** 为 LogBERT 增加“特殊能力”标记。
**输入：** `train` 文件
**处理逻辑：** `WordVocab` 读取 `train` 文件，统计数字出现的频率，并添加 BERT 专用的 Special Tokens。

**产出文件：**
**`vocab.pkl`** (加载到内存后是这样的字典)：
```python
{
  "[PAD]": 0,   # 补齐用
  "[UNK]": 1,   # 未知词
  "[DIST]": 2,  # 句首标记 (LogBERT核心)
  "[MASK]": 3,  # 填空题掩码
  "5": 4,       # 原始的 ID 5 可能会被重新映射，或者保留
  "22": 5,
  "11": 6
}
```

---

## 3. 总结与微服务迁移指南

你看，整个过程其实就是：**文本 -> 结构化表格 -> ID 字典 -> ID 序列 -> 纯数字文本**。

### 如果你要做 TrainTicket (微服务)：

你不需要改动 Step 1, 2, 4, 5 的核心逻辑。**你唯一要伤筋动骨修改的，只有 Step 3 (序列化)。**

*   **HDFS:** `Group by BlockId` (因为 HDFS 的上下文是 Block)
*   **TrainTicket:** `Group by TraceId` (因为微服务的上下文是 Trace)

**你的修改清单：**
1.  在生成 `hdfs_sequence.csv` (或者叫 `trainticket_sequence.csv`) 时，把正则提取 `blk_` 的代码，改成读取 CSV 列中的 `trace_id`。
2.  生成的 `train` 文件内容依然是一串串数字，LogBERT 根本不知道这些数字背后是 Block 还是 Trace，它只管训练。这就是为什么说 LogBERT 是通用的。