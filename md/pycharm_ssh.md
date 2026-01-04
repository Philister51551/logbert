### 1. 如何在 PyCharm 2024 中配置“经典模式”？

不要点击欢迎界面的 "Remote Development"。请按以下步骤操作（这是最稳的方案）：

**第一步：创建一个本地项目**
在你的电脑上新建一个文件夹（比如 `LogBERT_Project`），用 PyCharm 打开它。

**第二步：配置代码同步 (SFTP)**
1.  顶部菜单栏：`Tools` -> `Deployment` -> `Configuration...`
2.  点击左上角 `+` 号，选择 `SFTP`。
3.  **SSH configuration**: 点击 `...` 新建。
    *   填入 AutoDL 提供的 Host (IP), Port, User (root), Password。
    *   点击 `Test Connection` 确保通了。
4.  **Root path**: 点击 Autodetect（通常是 `/root`）。
5.  **Mappings** (最关键的一步)：
    *   `Local path`: 你的本地项目路径（自动填好的）。
    *   `Deployment path`: 服务器上的路径，建议填 `/root/LogBERT_Reproduction` (自己起个名)。
6.  点击 OK。
7.  **开启自动上传**：`Tools` -> `Deployment` -> 勾选 `Automatic Upload (Always)`。这样你改完代码不需要手动传。

**第三步：配置远程解释器 (让代码用显卡跑)**
1.  打开设置：Windows 是 `File` -> `Settings` (Mac 是 `PyCharm` -> `Settings`)。
2.  找到 `Project: ...` -> `Python Interpreter`。
3.  点击右上角的 `Add Interpreter` -> `On SSH...`。
4.  选择 `Existing` (现有的)，选择刚才配好的 SSH 连接。
5.  **Introscope / System Interpreter**：
    *   不要选 System Interpreter。
    *   **要选 Conda Environment**（如果你之前按我建议装了 conda）。
    *   或者直接指定路径：点击 `...`，找到 `/root/miniconda3/envs/你的环境名/bin/python`。
6.  一路 Next 点击 Finish。

### 总结

*   **Remote Development (Beta)** = 在服务器上跑 IDE，本地看画面。**（AutoDL 别用，卡顿且坑多）**
*   **Deployment + Remote Interpreter** = 在本地跑 IDE，自动同步文件，远程跑代码。**（强力推荐，稳定，响应快）**

用经典模式，你可以享受 PyCharm 的所有高级功能（重构、跳转），同时完全利用 3090/4090 的算力，而且即使断网了，你也能看代码、写代码，网好了自动同步上去。

---

这是一个非常经典的新手误区，别担心，配置其实已经生效了，只是**PyCharm 的逻辑是“增量更新”**。

“自动上传”的意思是：**当你修改并保存一个文件时，它会自动上传。** 它**不会**在你刚配好连接时，就自动把你本地现有的所有代码一股脑传上去。

请按以下步骤解决，只需操作一次：

### 解决方法：手动执行第一次全量上传

由于这是你第一次连接，服务器上是空的，你需要手动把本地现有代码全部推上去。

1.  **右键点击** PyCharm 左侧项目栏（Project 面板）最顶部的文件夹（也就是你的项目根目录）。
2.  在右键菜单中找到 **`Deployment` (部署)**。
3.  选择 **`Upload to ...` (上传到...)** -> 选择你刚才配置的那个 AutoDL 服务器名字。
4.  你会看到底部进度条开始走，等它传完。

---

### 如果后续修改代码还不自动上传

如果你已经手动上传过一次了，但之后**修改代码按 Ctrl+S** 还是没有反应，请检查以下两个最容易忽略的开关：

#### 1. 检查是否设置了“默认服务器” (最常见原因)
如果你的 Deployment 列表里以后配了多个服务器，PyCharm 不知道该往哪个自动传。
*   打开 `Tools` -> `Deployment` -> `Configuration`。
*   在左侧列表选中你的 AutoDL 服务器。
*   **关键点**：点击列表上方的 **“打钩”图标** (Tooltip 显示为 `Use as Default`)。
*   确保服务器名字变成了 **粗体**，这就代表它是默认目标了。

#### 2. 检查自动保存触发机制
*   打开 `Tools` -> `Deployment` -> `Options`。
*   找到 `Upload changed files automatically to the default server`。
*   建议选择 **`Always`** (或者 `On explicit save action`，这样你按 Ctrl+S 时才会传，防止乱改的代码误传)。

### 验证是否成功
1.  随便打开一个代码文件，加个空行。
2.  按 `Ctrl + S` 保存。
3.  看 PyCharm 底部状态栏（最下面一行），应该会瞬间闪过一行小字：`Uploading 'xxx.py' to 'root@...'`。
4.  如果看到了这行字，恭喜你，环境配置完美结束！

现在你可以愉快地在本地写代码，然后直接在 PyCharm 下方的 Terminal 里运行 `python train.py` (确保 Terminal 也是 SSH 远程的) 跑实验了。