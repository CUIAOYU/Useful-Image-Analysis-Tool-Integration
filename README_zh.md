# Useful Image Analysis Tool Integration (基于 SAM 和 PyQt)

这是一个使用 PyQt6 构建的图形用户界面 (GUI) 工具，旨在简化和加速科研或实验中常见的图像分析与处理任务，特别是针对需要批量处理大量图像（如显微照片、组织切片扫描图等）的场景。它可以帮助您自动提取定量信息、比较图像特征或改善图像视觉效果。

## 主要功能

* **对比度增强 (Contrast Enhancement)**:
    * **功能说明**: 调整图像中最亮和最暗区域的差异，提升图像清晰度和细节可见度。
    * **批量处理**: 对输入文件夹中的所有图像应用对比度增强。
    * **可调因子**: 用户可以指定对比度增强的因子 (通常大于1表示增强，小于1表示减弱)。
    * **应用场景示例**:
        * 增强医学影像（如X光片、MRI）中病灶区域的轮廓。
        * 使显微图像中细胞或组织的边缘、内部结构更清晰可见。
        * 改善扫描的电泳凝胶图谱、印迹膜或旧实验记录的清晰度。

* **聚合分析 (Aggregate Analysis)**:
    * **亮度到聚合度的转换**:
        * 本工具定义了一个“聚合度”(Aggregate)指标，范围是 0-100，用于量化像素的“暗度”或“非亮度”程度。
        * 它是根据像素的**灰度亮度** (0-255范围，0代表全黑，255代表全白) 计算得出的。
        * **计算步骤**:
            1.  将像素的灰度亮度值除以 255，将其标准化到 0-1 范围 (`灰度亮度 / 255.0`)。
            2.  将标准化后的值乘以 100。
            3.  用 100 减去上一步得到的结果。
        * **计算公式**:
            ```
            聚合度 = 100 - (灰度亮度 / 255.0) * 100
            ```
        * **核心关系**: 这个公式意味着 **像素亮度越低（颜色越深），其聚合度数值越高**。（例如：纯黑色像素亮度为0，聚合度为100；纯白色像素亮度为255，聚合度为0）。
        * **潜在关联**: 在某些应用场景（例如生物染色图像）中，较高的聚合度可能与较高的“物质密度”或“信号强度”（特指暗信号标记）存在一定的关联性。
    * **快速评估**: 计算单个图像或文件夹中所有图像的**平均聚合度**，提供图像整体“暗度”或“聚合水平”的快速评估指标。
    * **应用场景示例**:
        * 快速比较不同实验处理组样本图像的整体染色深度或信号水平。
        * 对大量图像进行初步筛选，识别整体偏暗或偏亮的图像。

* **热力图生成 (Heatmap Generation)**:
    * **功能说明**: 根据每个像素点的“聚合度”分数赋予其不同颜色，生成一张直观显示聚合度空间分布的彩色图。
    * **聚合值计算**: 对整个灰度图像计算每个像素的“聚合度” (0-100范围，如上所述)。
    * **基于阈值的热力图可视化**:
        * 用户设定**最小聚合度阈值 (`Min Aggregate Threshold`)** 和**最大聚合度阈值 (`Max Aggregate Threshold`)**，定义感兴趣的聚合度分数区间 [最低值, 最高值]。
        * **阈值与颜色映射逻辑**:
            * 聚合度**低于**最低值的像素：显示为**黑色** (代表信号过低或背景)。
            * 聚合度**高于**最高值的像素：显示为**亮红色** (代表信号饱和或聚集)。
            * 聚合度**介于** [最低值, 最高值] 区间内的像素：根据其相对位置，颜色从**紫色** (接近最低值) -> **蓝色** -> **绿色** -> **黄色** -> **橙色** -> **红色** (接近最高值) 平滑过渡。
        * **目的**: 通过调整阈值聚焦可视化范围，有效过滤背景或饱和区域，突出目标区域的分布和强度变化。
    * **应用场景示例**:
        * 在组织切片图像上，用颜色直观显示不同区域的染色强度或特定分子表达水平（若与聚合度相关）。
        * 在材料扫描图像中，可视化密度或成分分布在特定范围内的区域。

* **SAM 处理 (SAM Processing)**:
    * **功能说明**: 结合 Segment Anything Model (AI自动分割) 与聚合度分析。它能自动识别图像中的独立区域，并筛选出那些内部像素满足特定聚合度范围的区域，然后进行定量分析和可视化标记。
    * **模型加载**: 支持加载用户选择的预训练 Segment Anything Model (`.pth`) 文件。
    * **自动掩码生成**: 利用 SAM 对输入图像进行全自动实例分割，生成多个独立的区域掩码 (masks)。
    * **两阶段阈值过滤与区域属性计算**:
        * **阶段一：像素级聚合度过滤 (基于 Min/Max Aggregate Threshold)**:
            * 程序检查 SAM 生成的**每个**区域内部的所有像素点。
            * 只有当一个区域内部**含有聚合度分数落在**用户设定的 [最低值, 最高值] 区间内的像素时，该区域才被初步认定为“包含有效信号”。聚合度在此区间外的像素不参与后续计算。
            * 如果一个区域内**没有任何**像素的聚合度在此区间内，则该区域被完全忽略。
        * **阶段二：区域级属性阈值过滤 (基于 A/I/R Threshold)**:
            * 对于通过阶段一筛选的区域，程序**仅使用其内部聚合度在[最低值, 最高值]范围内的像素**来计算以下属性：
                * **面积 (Area)**: 这些有效像素的总数量。
                * **强度 (Intensity)**: 这些有效像素的聚合度之和。
                * **比率 (Ratio)**: Intensity / Area。
            * **最终筛选**: 只有当一个区域计算出的 A, I, R 值**同时满足**用户设定的相应最小阈值（`A Threshold`, `I Threshold`, `R Threshold` - 取决于用户是否勾选了使用这些指标进行过滤）时，该区域才会被最终确定为需要关注的目标区域。
    * **可视化**: 在输出图像上绘制通过**两阶段阈值过滤**后筛选出的目标区域的轮廓（不同区域用不同颜色区分），并可以选择性地标注计算出的 A/I/R 数值。
    * **详细参数调整 (SAM Auto Mask Generator Parameters)**: SAM 提供一些高级选项，让您可以微调它的工作方式。一般用默认设置就好，但了解它们有助于解决特定问题：
        * **`points_per_side`** (每边点数, 默认 32): 想象在图上撒网格找物体，这个值是网格每边的点数。**值越高，网格越密**，更容易找到小东西，但也**更慢、更吃内存/显存**。反之则更快，但可能漏掉小的。
        * **`points_per_batch`** (每批点数, 默认 64): 决定了显卡 (GPU) 一次处理多少个网格点。**主要影响显存**。如果遇到 **"Out of Memory" 错误，就降低这个值** (比如 32 或 16)。如果显存很大，可以稍微提高试试能否加速。
        * **`pred_iou_thresh`** (预测 IoU 阈值, 默认 0.88): SAM 对自己找到的区域打的分数 (0-1)，表示它有多自信这个区域找对了。**值越高 (接近1)，要求越严**，结果噪点少，但可能把一些模糊的或者不太确定的目标漏掉。**值越低，要求越松**，可能找回一些目标，但也可能引入更多错误分割。
        * **`stability_score_thresh`** (稳定性阈值, 默认 0.95): 衡量找到的区域边界稳不稳固 (0-1)。稍微变动一下判断标准，形状会不会大变？**值越高，要求边界越稳定清晰**，结果更可靠，但可能丢掉边界模糊的目标。**值越低，能容忍更模糊的边界**，可能找回这类目标，但也可能得到形状奇怪的结果。
        * **`stability_score_offset`** (稳定性偏移, 默认 1.0): 计算稳定性分数时用的一个内部参数，**一般不用改**。
        * **`box_nms_thresh`** (包围盒 NMS 阈值, 默认 0.7): 防止同一个物体被画上好几个框。如果两个框重叠度高于这个值，就可能只保留一个。**值越低，去重越狠**，可能把靠太近 不同物体当成一个；**值越高，越容忍重叠**，可能同一个物体出好几个框。**默认值通常挺好，一般不用改**。
        * **`crop_n_layers`** (裁剪层数, 默认 0): 处理超大图片的“切块”开关。**0 表示不切**，对整图处理。**设为 1 或更大**，会把大图切成很多重叠的小块分别处理再拼起来。优点是**能处理内存/显存装不下的大图**，缺点是**速度会慢很多很多**，而且拼接处可能不太自然。**只在处理超大图片遇到内存问题时，才考虑设为 1**。
        * **`crop_nms_thresh`** (裁剪 NMS 阈值, 默认 0.7): 切块模式下，合并小块结果时用的去重叠阈值。**一般不用改**。
        * **`crop_overlap_ratio`** (裁剪重叠率, 默认 ~0.341): 切块模式下，小块之间重叠部分的比例。**一般不用改**。
        * **`crop_n_points_downscale_factor`** (裁剪点数缩减因子, 默认 1): 切块模式下，每个小块里撒的点是否要稀疏一些。1 表示不稀疏。**值越大，小块处理越快，但精度可能下降。一般保持为 1**。
        * **`min_mask_region_area`** (最小掩码区域面积, 默认 0): 这是个**后期处理**步骤。设一个**大于 0** 的值（比如 50），程序会自动删掉所有面积小于这个像素数的、独立的、细小的分割结果。**非常适合用来自动清理背景上的小噪点**。设为 0 则保留所有结果。
        * **`output_mode`** (输出模式, 默认 `binary_mask`): 控制 SAM 输出掩码的内部数据格式。**请务必保持默认值 `binary_mask`**，这是最标准、最通用的格式，其他格式可能导致后续处理出错。
    * **应用场景示例**:
        * 自动识别并定量分析显微图像中所有细胞的大小(Area)和特定蛋白表达量(Intensity)，且只统计并标记那些表达量（对应特定聚合度范围）达标的细胞。
        * 在免疫组化(IHC)或免疫荧光(IF)图像中，自动分割阳性信号区域（通过设置合适的聚合度阈值定义阳性信号），并测量其面积和总信号强度。
        * 自动识别并测量活/死细胞染色图像中，特定颜色（对应特定聚合度范围）细胞的数量和面积，忽略其他颜色或背景。

* **批量处理与易用性**:
    * **文件夹处理**: 所有主要功能都支持对整个文件夹的图像进行批量处理。
    * **用户界面**: 提供直观的图形界面 (PyQt6)，方便用户选择文件、设置参数、启动处理、查看日志和打开输出文件夹。
    * **日志记录**: 在界面右侧提供详细的操作和处理日志。
    * **输出管理**: 自动在输入文件夹旁边创建带有参数信息的输出文件夹，方便结果管理。

## 系统要求 (Requirements)

* **Python**: 本项目在 **Python 3.11.9** 版本下开发和测试。建议使用 Python 3.9+。
* **操作系统**: 建议使用 Windows 10 或 Windows 11。
* **硬件**: 为了获得较好的 SAM 处理性能（如果使用 GPU 加速），建议使用 NVIDIA 显卡，性能至少达到 GeForce RTX 3060 或同等水平。CPU 也可以运行，但速度会慢很多。
* **主要依赖库**: 您需要安装以下 Python 包：
    * `PyQt6`
    * `opencv-python`
    * `numpy`
    * `Pillow`
    * `torch`
    * `torchvision`
    * `segment-anything`
    * `tifffile`
    * `pycocotools` (可选, 但建议安装以获得完整的 SAM 功能)

## 安装 (Installation)

**前提条件:**

* **安装 Git:** 你需要先在你的系统上安装 [Git](https://git-scm.com/downloads)，才能使用 `git clone` 命令下载代码。
* **安装 Python:** 确保你安装了符合版本要求的 Python (本项目基于 Python 3.11.9 开发，建议 3.9+)。你可以在终端或命令行运行 `python --version` 或 `python -V` 来检查你的版本。

**安装步骤:**

1.  **克隆仓库**: 打开终端或命令行，执行以下命令将代码下载到本地：
    ```bash
    git clone [https://github.com/CUIAOYU/Useful-Image-Analysis-Tool-Integration.git](https://github.com/CUIAOYU/Useful-Image-Analysis-Tool-Integration.git)
    cd Useful-Image-Analysis-Tool-Integration
    ```
2.  **(推荐) 创建并激活 Python 虚拟环境**: 为了避免不同项目间的库冲突，强烈建议创建一个虚拟环境：
    ```bash
    # 创建虚拟环境 (例如命名为 venv)
    python -m venv venv

    # 激活虚拟环境
    # Windows (cmd/powershell):
    .\venv\Scripts\activate
    # macOS/Linux (bash/zsh):
    source venv/bin/activate
    ```
    *后续的 `pip install` 命令都应在**激活虚拟环境后**执行。*
3.  **安装依赖库**: 您需要手动使用 pip 安装所需的包。运行：
    ```bash
    pip install PyQt6 opencv-python numpy Pillow torch torchvision segment-anything tifffile pycocotools
    ```
    * **依赖安装说明与常见问题**:
        * **PyTorch/Torchvision**: 这两个库的安装与你的操作系统、是否使用 NVIDIA GPU 及 CUDA 版本密切相关。**强烈建议**先访问 [PyTorch 官方网站](https://pytorch.org/)，根据你的具体环境（OS, Package, Compute Platform）获取**官方推荐的安装命令**，并**首先单独执行**这个命令来安装 PyTorch 和 Torchvision。之后再运行上面的 `pip install ...` 命令安装其余的包（可以省略掉 torch 和 torchvision）。
        * **Pycocotools**: 在 Windows 上安装 `pycocotools` 可能需要预先安装 Microsoft C++ Build Tools。如果遇到编译错误，请搜索相关错误信息查找解决方案，或者如果你的应用场景不涉及 RLE 格式的掩码，可以暂时不安装它（但 SAM 的某些输出模式将不可用）。
        * **其他错误**: 如果安装其他库时遇到问题，请仔细阅读 pip 输出的错误信息，通常会包含解决问题的线索。

4.  **(可选) 验证安装**: 安装完所有依赖后，可以尝试运行主程序（参考“如何使用”部分的第3步）看是否能成功启动 GUI 界面，以此作为安装成功的基本验证。

## SAM 模型设置 (SAM Model Setup) - 重要!

本工具的 SAM 处理功能需要预训练的 Segment Anything Model 文件 (`.pth`)。**这些模型文件非常大，不会包含在本仓库中，你需要自行下载。**

1.  **下载模型**:
    * 请从 Meta AI Research 的官方 SAM 仓库或其他可信来源下载所需的 SAM 模型文件。
    * 常见的模型检查点 (checkpoints) 包括 `vit_h` (最大), `vit_l`, `vit_b` (最小)。选择哪个取决于你对精度和速度/资源消耗的权衡。
    * **官方下载链接**: [SAM Model Checkpoints](https://github.com/facebookresearch/segment-anything#model-checkpoints)
2.  **放置模型**:
    * 下载后，将 `.pth` 文件保存在你的电脑上一个方便访问的位置。
    * 运行本工具时，需要在 "SAM Processing" 标签页通过 "Select SAM Model File (.pth)" 按钮来指定你下载的模型文件的具体路径。程序需要知道模型文件在哪里才能加载它。

## 如何使用 (Usage)

1.  **准备工作**: 确保所有依赖库已正确安装，并且你已经下载了所需的 SAM 模型文件。
2.  **激活环境**: 如果你使用了 Python 虚拟环境，请先激活它。
3.  **运行程序**: 在项目根目录下，打开终端或命令行，运行主 Python 脚本：
    ```bash
    python [你的主脚本文件名].py
    # 例如: python main_gui.py 或你实际使用的脚本名
    ```
4.  **熟悉界面**: 程序启动后，会显示图形用户界面。
    * **左侧**: 控制区，包含功能标签页、文件/文件夹选择按钮、参数设置滑块/选框等。
    * **右侧**: 日志区，显示程序运行信息、进度和错误提示。
5.  **选择功能**: 点击顶部的不同功能标签页 (例如 "Contrast Enhancement", "Heatmap Generation", "SAM Processing") 来切换功能。
6.  **设置与执行**: 在选定的功能标签页中：
    * 使用 "Select Input Folder" 选择包含待处理图像的文件夹。
    * (若使用 SAM Processing) 使用 "Select SAM Model File (.pth)" 选择你下载的模型。
    * 仔细调整界面上提供的各项参数和阈值。
    * 点击蓝色的处理按钮 (例如 "Process All Images", "Generate Heatmaps") 开始执行任务。按钮通常在所有必要输入（如文件夹、模型）被指定后才可点击。
7.  **监控与查看结果**:
    * 处理过程中，请关注右侧日志区的输出信息，了解进度或可能的错误。
    * 处理完成后，日志区会提示完成信息。
    * 点击 "Open Output Folder" 按钮可以方便地打开存放结果文件的文件夹。输出文件夹通常会在输入文件夹旁边自动创建，并带有描述性的名称（包含所用参数）。

## 如何贡献 (Contributing)

欢迎各种形式的贡献！

* **报告 Bug 或提出建议**: 如果您在使用中发现任何问题，或者有改进功能的想法，请通过 GitHub 的 [Issues](https://github.com/CUIAOYU/Useful-Image-Analysis-Tool-Integration/issues) 页面提交。请尽可能详细地描述问题或建议，包括复现步骤、截图（如果适用）、您的操作系统和软件版本等信息。
* **提交代码**: 如果您希望直接改进代码，请遵循标准的 GitHub Fork & Pull Request 流程。建议先创建一个 Issue 讨论您想要进行的更改。

## 报告问题 (Reporting Issues)

如果您遇到任何 Bug 或有功能请求，请在本仓库的 [Issues](https://github.com/CUIAOYU/Useful-Image-Analysis-Tool-Integration/issues) 页面创建新的 Issue。这是获取支持和反馈的主要途径。

## 许可证 (License)

本项目采用 **MIT 许可证**。详情请参阅 `LICENSE` 文件。

