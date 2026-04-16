### 无人机浮游植物识别项目 - AI视觉（Tianyu）专属 README

#### 一、核心定位
作为项目AI视觉算法负责人，聚焦浮游植物视觉识别全流程。核心职责：**提供正射校正脚本**；**使用公开数据集训练天气分类模型（晴天/阴天/雾天）**；**基于贺一冉交付的标注数据集训练三个专用的 DeepLabV3+ 语义分割模型**，实现高鲁棒性的浮游植物像素级定位、分类与面积量化。不参与数据采集、标注等非视觉算法环节。

#### 二、绝对清晰的工作边界
✅ 核心负责（天气自适应AI视觉全流程）
1. 正射校正脚本开发与交付。
2. 天气分类模型：使用公开数据集（存放于 `data/weather_public/`）训练，无需贺一冉提供标注。
3. 分割模型训练：从 `data/03_labeled/` 读取贺一冉交付的标注数据，按文件名中的天气标签划分三个子数据集，分别训练 DeepLabV3+ 模型。
4. 推理链路：输入图像 + 无人机参数文件 → 天气分类 → 调用对应分割模型 → 输出掩码图与量化结果（含实际面积）。
5. 量化计算：浮游植物覆盖面积（m²/亩）、占比、浓度等级。

❌ 绝不涉及
- 不参与任何原始图像采集、标注。
- 不进行时序分析、预测建模。

#### 三、分步执行任务（含路径约定）

**0. 正射校正脚本开发与交付**
- 脚本位置：`utils/preprocess/orthorectify.py`
- 功能要求：
  - 读取 `data/01_raw/images/` 中的 RAW 图像及同名参数文件（位于 `data/01_raw/drone_json/`）；
  - 利用无人机高度、姿态角、相机内参，将图像纠正为垂直正射影像（消除倾斜畸变）；
  - 将校正后图像**等比缩放**至长边 1024 像素，保持原始宽高比，然后通过**黑色像素填充**补全为 1024×1024 正方形 PNG 图像；
  - 输出最终图像至 `data/02_preprocessed/images/`，保持原始文件名（仅扩展名改为 .png）；
  - **同步计算缩放后的地面采样距离（GSD，单位：米/像素），并生成同名元数据 JSON 文件（如 `20240315_sunny_001_meta.json`）存放至 `data/02_preprocessed/meta/`**。
- 交付对象：贺一冉
- 交付物：Python 脚本 + 参数文件格式说明 + 依赖库列表

**1. 数据接收与核验**
- 读取路径：
  - 校正图像：`../data/02_preprocessed/images/`
  - 标注数据：`../data/03_labeled/`（JSON标注与掩码）
- 文件名格式：`YYYYMMDD_天气_序号.png`，从中解析天气标签。
- 核验数据完整性，反馈异常至贺一冉修正。

**2. 天气分类模型训练**
- 数据来源：`../../data/weather_public/`，使用 **多类天气图片数据集**（深圳大学VCC，和鲸社区下载：https://www.heywhale.com/mw/dataset/5e732227c59d610036227d89）。
- 数据结构：下载后解压，仅保留 `sunny/`、`cloudy/`、`haze/` 三个文件夹，放入 `data/weather_public/` 下。
- 脚本位置：`models/weather_classifier/train.py`
- 模型保存路径：`models/weather_classifier/best_weather_model.pth`
- 训练参数：批次大小 32，20 轮，Adam 优化器，初始学习率 1e-4。

**3. 多条件分割模型训练**
- 脚本位置：
  - `models/deeplabv3plus/sunny/train.py`
  - `models/deeplabv3plus/cloudy/train.py`
  - `models/deeplabv3plus/foggy/train.py`
- 数据集划分：每个训练脚本读取 `../../../data/03_labeled/` 下对应天气的图像与掩码（通过文件名过滤）。
- 模型保存路径：各天气目录下的 `best_model.pth`

**4. 模型推理与量化计算**
- 脚本位置：`models/predict_pipeline.py`
- 输入：单张校正图像路径（如 `test_images/20240501_sunny_test.png`）及**对应的无人机参数文件路径**（如 `test_images/20240501_sunny_test.json`）
- 执行逻辑：
  1. 加载天气分类模型，预测天气；
  2. 加载对应 DeepLabV3+ 模型，执行分割推理；
  3. 生成分割掩码图并保存；
  4. **读取校正图像对应的元数据文件（`data/02_preprocessed/meta/` 中的同名 `_meta.json`），提取缩放后的 GSD（米/像素）。对分割掩码中非填充区域的浮游植物像素进行计数，计算真实面积：面积(m²) = 像素数 × (GSD)²**；
  5. 输出面积、占比、浓度等级等量化数据。
- 输出路径：
  - 掩码可视化图 → `outputs/masks/输入文件名_mask.png`
  - 量化数据 → `outputs/quantifications/输入文件名.json`（含实际面积字段）

**5. 结果交付与对接**
- 确保 `outputs/` 目录下数据格式与数模组、展示端约定一致。

#### 四、模块分工（一句话界定）
- 航拍组：采集多天气原始图像及配套参数 → `data/01_raw/`
- 贺一冉：执行正射校正 → `data/02_preprocessed/images/`；标注 → `data/03_labeled/`
- **本人**：提供校正脚本；公开数据集 → 训练天气分类模型；标注数据集 → 训练分割模型；推理 → 输出至 `outputs/`（含实际面积）
- 数模组：读取 `outputs/quantifications/` 进行预测建模

#### 五、最终交付物
1. 正射校正脚本（`utils/preprocess/`）及使用文档。
2. 天气分类模型权重（`models/weather_classifier/best_weather_model.pth`）及训练推理代码。
3. 三个 DeepLabV3+ 分割模型权重（`models/deeplabv3plus/{weather}/best_model.pth`）及训练脚本。
4. 总推理脚本 `models/predict_pipeline.py`。
5. 实测分割掩码图（`outputs/masks/`）与量化数据（`outputs/quantifications/`，含实际面积）。
6. 数模组对接接口文档。
7. 模型评估报告。