### 无人机浮游植物识别项目 - AI视觉（Tianyu）专属 README
#### 一、核心定位
作为项目AI视觉算法负责人，聚焦浮游植物视觉识别全流程。核心职责：基于贺一冉交付的标准化预处理图像及天气标签，**首先训练天气分类模型（晴天/阴天/雾天）以路由不同环境，进而训练三个专用的 DeepLabV3+ 语义分割模型**，实现高鲁棒性的浮游植物像素级定位、分类、量化。不参与数据采集、预处理等非视觉算法环节。

#### 二、绝对清晰的工作边界
✅ 核心负责（天气自适应AI视觉全流程）
1. 基于贺一冉输出的预处理干净图像与天气标签，开展数据集标注复核与天气标签核验。
2. **天气分类模型搭建与训练**：选择轻量级CNN（如ResNet-18或MobileNet）作为天气分类器，实现晴天/阴天/雾天三分类，作为分割模型的路由开关。
3. **多条件分割模型训练**：由于需要区分微囊藻、水华鱼腥藻、水绵等多种浮游植物，它们在水体中的色彩表现差异细微，而晴天高光、阴天柔光、雾天低散射会显著改变这些色差特征。因此分别针对晴天、阴天、雾天三个子数据集，搭建、训练、调优三个独立的 DeepLabV3+ 语义分割模型。
   - **选用 DeepLabV3+ 的理由**：
     1. **多尺度特征提取能力强**：ASPP模块能同时捕捉大面积水华斑块和细碎浮游植物聚集区。
     2. **边界分割精细**：编码器-解码器结构恢复空间细节，生成边缘清晰掩码。
     3. **鲁棒性高**：在遥感、航拍图像分割领域应用成熟，社区支持丰富。
4. 执行推理链路：输入图像 → 天气分类 → 调用对应 DeepLabV3+ 模型 → 输出浮游植物分割掩码图，识别有无、位置、具体类别。
5. 量化计算：浮游植物覆盖面积、占比、浓度等级。
6. 输出标准化识别结果，对接数模组。

❌ 绝不涉及
- 不参与任何原始图像采集、预处理。
- 不使用LabelMe开展初始像素级标注（仅复核）。
- 不进行时序分析、预测建模。

#### 三、分步执行任务（含路径约定）

**1. 数据接收与核验**
- 读取路径：`../data/02_preprocessed/images/`（预处理图像）、`../data/03_labeled/`（标注JSON与掩码）。
- 文件名格式：`YYYYMMDD_天气_序号.png`，从中解析天气标签。
- 核验数据完整性，反馈异常至贺一冉修正。

**2. 天气分类模型训练**
- 脚本位置：`models/weather_classifier/train.py`
- 数据集构建：遍历 `../data/02_preprocessed/images/`，根据文件名解析天气标签（sunny/cloudy/foggy），生成训练集与验证集。
- 模型保存路径：`models/weather_classifier/model.pth`

**3. 多条件分割模型训练**
- 脚本位置：
  - `models/deeplabv3plus/sunny/train.py`
  - `models/deeplabv3plus/cloudy/train.py`
  - `models/deeplabv3plus/foggy/train.py`
- 数据集划分：每个训练脚本读取 `../../../data/03_labeled/` 下对应天气的图像与掩码（通过文件名过滤）。
- 模型保存路径：各天气目录下的 `best_model.pth`

**4. 模型推理与量化计算**
- 脚本位置：`models/predict_pipeline.py`
- 输入：单张预处理图像路径（如 `test_images/20240501_sunny_test.png`）
- 执行逻辑：
  1. 加载天气分类模型 `models/weather_classifier/model.pth`，预测天气。
  2. 根据预测结果动态加载对应 DeepLabV3+ 模型（`models/deeplabv3plus/{weather}/best_model.pth`）。
  3. 执行分割推理，生成掩码图。
  4. 计算面积、占比、浓度等级。
- 输出路径：
  - 掩码可视化图 → `outputs/masks/输入文件名_mask.png`
  - 量化数据 → `outputs/quantifications/输入文件名.json`

**5. 结果交付与对接**
- 确保 `outputs/` 目录下数据格式与数模组、展示端约定一致。
- 交付物清单见第五节。

#### 四、模块分工（一句话界定）
- 航拍组：采集多天气原始图像 → `data/01_raw/`
- 贺一冉：预处理 → `data/02_preprocessed/images/`；标注 → `data/03_labeled/`
- **本人**：读取 `data/03_labeled/` 训练模型 → 保存至 `models/`；运行推理 → 输出至 `outputs/`
- 数模组：读取 `outputs/quantifications/` 进行预测建模

#### 五、最终交付物
1. 天气分类模型权重（`models/weather_classifier/model.pth`）及训练推理代码。
2. 三个 DeepLabV3+ 分割模型权重（`models/deeplabv3plus/sunny/best_model.pth` 等）及训练脚本。
3. 总推理脚本 `models/predict_pipeline.py`。
4. 实测分割掩码图（`outputs/masks/`）与量化数据（`outputs/quantifications/`）。
5. 数模组对接接口文档。
6. 模型评估报告。