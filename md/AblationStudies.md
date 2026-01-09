要做消融实验（Ablation Study）来对比 **MLKP（纯掩码预测）**、**VHM（纯距离检测）** 和 **MLKP+VHM（两者结合）** 的效果，你需要通过修改 `logbert.py` 中的开关配置来控制**训练（Train）**和**预测（Predict）**的行为。

由于 BERT 需要通过 MLKP 任务来学习语义特征，单纯的 VHM（DeepSVDD）如果脱离了 MLKP 训练，往往无法提取出有效的特征（因为没有重建任务，Embedding 会塌缩）。

因此，最科学的消融实验方案是：**训练两个模型**（一个纯 MLKP，一个联合训练），然后在**预测阶段**通过开关不同的检测逻辑来得到三个结果。

以下是具体的操作步骤：

---

### 实验一：MLKP (Baseline) - 仅使用掩码预测
这是 LogBERT 的原生模式，不使用任何球体距离检测。

**修改 `logbert.py` 配置：**
```python
# 1. 关闭训练时的 VHM Loss
options["hypersphere_loss"] = False

# 2. 关闭测试时的 VHM 检测
options["hypersphere_loss_test"] = False

# 3. 开启 MLKP (掩码预测)
options["is_logkey"] = True
```

**执行步骤：**
1.  `python logbert.py train` (训练出的模型只懂语义补全)
2.  `python logbert.py predict`
3.  **结果**：记录输出的 F1，这就是 **MLKP Only** 的结果。

---

### 实验二：MLKP + VHM (Proposed) - 两者结合
这是你完整代码的模式，同时使用掩码预测错误和超球体距离超标来判定异常（通常逻辑是 OR，即任一触发即异常）。

**修改 `logbert.py` 配置：**
```python
# 1. 开启训练时的 VHM Loss (训练出具有紧凑球心的 Embedding)
options["hypersphere_loss"] = True

# 2. 开启测试时的 VHM 检测
options["hypersphere_loss_test"] = True

# 3. 开启 MLKP (掩码预测)
options["is_logkey"] = True
```

**执行步骤：**
1.  `python logbert.py train` (训练出的模型既懂语义，Embedding 又紧凑)
2.  `python logbert.py predict`
3.  **结果**：记录输出的 F1，这就是 **MLKP + VHM** 的结果。

---

### 实验三：VHM Only - 仅使用距离检测
这一步是为了证明引入 DeepSVDD 的距离检测是否独立有效。
**注意**：我们需要使用 **实验二** 训练好的模型（因为那个模型已经学好了球心和紧凑特征），但在**预测阶段**强制关闭 MLKP 检测，只看距离。

**修改 `logbert.py` 配置：**
*(保持实验二的训练结果不动，只修改配置并重新运行 predict)*

```python
# 1. 保持 VHM 开启 (为了加载球心参数)
options["hypersphere_loss"] = True
options["hypersphere_loss_test"] = True

# 2. 【关键】关闭 MLKP 检测
# 虽然模型会输出预测，但设为 False 后，predict_log.py 中的 compute_anomaly 会忽略 token 预测错误，只看 distance
options["is_logkey"] = False 
```

**执行步骤：**
1.  **不要重新训练** (直接使用实验二生成的 `best_bert.pth` 和 `best_center.pt`)。
2.  直接运行 `python logbert.py predict`。
3.  **结果**：此时输出的 F1 完全取决于日志向量距离球心的远近，这就是 **VHM Only** 的结果。

---

### 总结对照表

| 实验名称 | 训练需设置 (`logbert.py`) | 预测需设置 (`logbert.py`) | 操作说明 |
| :--- | :--- | :--- | :--- |
| **MLKP Only** | `hypersphere_loss = False`<br>`is_logkey = True` | `hypersphere_loss_test = False`<br>`is_logkey = True` | 需重新 Train，然后 Predict |
| **MLKP + VHM** | `hypersphere_loss = True`<br>`is_logkey = True` | `hypersphere_loss_test = True`<br>`is_logkey = True` | 需重新 Train，然后 Predict |
| **VHM Only** | (复用 MLKP+VHM 的模型) | `hypersphere_loss_test = True`<br>`is_logkey = False` | **不需要 Train**，改完配置直接 Predict |

通过这三组实验，你就可以在论文或报告中画出三个柱状图，分别展示：
1.  BERT 自身的预测能力 (MLKP)。
2.  DeepSVDD 距离聚类的能力 (VHM)。
3.  两者结合后的互补提升能力 (MLKP+VHM)。