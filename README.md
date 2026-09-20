
# 食物图像卡路里估计：数据集与基线整理


## 1. 任务说明

现有研究主要采用三种思路：

1. **直接回归**：输入食物图像，直接预测卡路里；
2. **分量估计后计算**：先估计食物体积或重量，再结合单位重量热量计算总卡路里；
3. **多模态估计**：结合图像、食材、菜名或烹饪方式，直接预测卡路里。

不同数据集的输入信息和标签来源不同，因此结果不能简单横向比较。使用 RGB、RGB-D、3D点云或食材文本的模型，应分别进行比较。


## 2. MM-Food-100K

**论文**：MM-Food-100K: A 100,000-Sample Multimodal Food Intelligence Dataset with Verifiable Provenance
**链接**：https://arxiv.org/abs/2508.10429

MM-Food-100K 包含约10万组食物图像和文本标注，提供菜名、食材、份量、营养信息和烹饪方式等字段。数据集的营养和份量标签部分由视觉语言模型生成，更适合作为大规模多模态训练数据或弱监督数据。

### 卡路里回归结果

| 方法 | 思路 | 训练数据量 | MAE/kcal↓ | RMSE/kcal↓ | R²↑ |
|---|---|---:|---:|---:|---:|
| Qwen-Max | 视觉语言模型直接回归 | 0 | 126.5 | 185.3 | 0.521 |
| Qwen-Max | 视觉语言模型直接回归 | 1,000 | 123.8 | 181.5 | 0.539 |
| Qwen-Max | 视觉语言模型直接回归 | 10,000 | 107.5 | 159.1 | 0.612 |
| Qwen-Max | 视觉语言模型直接回归 | 50,000 | 104.2 | 154.5 | 0.638 |
| GPT-4o | 视觉语言模型直接回归 | 0 | 98.7 | 148.1 | 0.685 |
| GPT-4o | 视觉语言模型直接回归 | 1,000 | 97.9 | 147.1 | 0.690 |
| GPT-4o | 视觉语言模型直接回归 | 10,000 | 96.2 | 144.9 | 0.702 |
| GPT-4o | 视觉语言模型直接回归 | 50,000 | 95.8 | 144.3 | 0.706 |

主要结论：GPT-4o 整体优于 Qwen-Max；增加训练数据后 MAE 和 RMSE 均下降；由于标签主要是AI估计值，结果不能直接和真实称重数据集进行公平比较。
ViT-FoodNA: An End-to-End Transformer-Based Multimodal Framework for Food Recognition and Nutrition Analysis（端到端transformer），ViT-FoodNA 的 MAE 11.3 因评估协议不明需要单独标注。

## 3. Nutrition5k

**论文**：Thames et al., *Nutrition5k: Towards Automatic Nutritional Understanding of Generic Food*, CVPR 2021
**链接**：https://arxiv.org/abs/2103.03375
**数据集**：https://github.com/google-research-datasets/Nutrition5k

Nutrition5k 是目前最权威的卡路里估计基准数据集之一。它包含约5000道真实菜品，提供俯视RGB图像、深度图、侧视角视频、配料重量以及基于USDA数据库计算的高精度营养标注。数据划分为90%训练和10%测试。原论文的核心贡献在于：首次证明了计算机视觉模型在复杂真实菜品上的卡路里预测精度可以超越专业营养师。

### 3.1 代表性结果

| 方法 | 思路 | 输入信息 | Calorie MAE/kcal↓ | PMAE↓ | 来源 |
|---|---|---|---:|---:|---|
| 均值预测 | 始终预测训练集均值 | 无图像特征 | 150.8 | 60.2% | 原论文 |
| 2D Direct Prediction | 直接回归整盘卡路里 | RGB图像 | 70.6 | 26.1% | 原论文 |
| Depth as 4th Channel | 深度作为第4通道输入，融合空间结构 | RGB-D | 47.6 | 18.8% | 原论文 |
| Volume Scalar | 先估计体积，再结合营养密度回归 | RGB+体积信息 | 41.3 | 16.5% | 原论文 |
| RGB-D分割与预测 | 先分割再预测营养，ResNet101主干 | RGB-D | — | 15.68% | Frontiers 2024 |
| FLAVA对比学习 | 视觉语言对比对齐，Swin V2处理RGB-D | RGB-D/视觉语言预训练 | 33.55 | 13.80% | 2025 |

### 3.2 VLM零样本基准

| 方法 | 思路 | 提示策略 | Calorie MAE/kcal↓ | MAPE↓ |
|---|---|---|---:|---:|
| Gemini 3.0 Flash | 视觉语言模型零样本推理 | P3 | 80.7 | 44.4% |
| GPT-4o | 视觉语言模型零样本推理 | Standard | 88.86 | — |
| GPT-4o | 两步推理（先识别再估算） | Two-Step | 80.32 | — |
| GPT-4o + 体积注入 | 加入体积辅助信息 | 体积辅助 | 78.8 | 43.4% |

> 注：VLM零样本结果不与训练后的模型公平比较。原论文已证明，训练后的模型在卡路里预测上超越专业营养师。

### 3.3 综述论文中的方法分类

2026年发表在 *Information Fusion* 的综述 *Multimodal Information Fusion for Food Nutrition Estimation: A Survey* 是本领域目前最系统的文献梳理。该综述采用**基于模态感知和问题导向的分类方法**，将现有方法归纳为四类：

1. **视觉-几何多模态学习**：融合RGB图像与深度/3D信息，解决份量和物理尺度模糊问题。代表性结果：Depth as 4th Channel（47.6 kcal）、Volume Scalar（41.3 kcal）。
2. **语义与知识增强估计**：引入食材文本、烹饪方式、营养知识，解决隐藏成分推断问题。代表性结果：FLAVA对比学习（13.80% PMAE）。
3. **组合多模态推理**：对食物各组分进行分解和营养聚合，解决混合菜品推理问题。
4. **其他多模态感知**：利用光谱、近红外等传感器获取内部物质信息。

该综述在 Nutrition5k 上进行了定量对比，发现几何感知建模和食材引导推理对提升估计精度影响最为显著。综述指出当前最大挑战包括：隐藏成分推断、份量尺度校准、数据集偏差、缺失模态下的鲁棒融合、以及临床级可靠性部署。

### 3.4 关键发现

- **模型架构主导性能**：一项系统评估发现，在VLM系统中，模型架构解释了99.6%的性能方差，多角度拍摄和提示工程的影响可忽略不计。
- **专业营养师仍是标杆**：VLM最佳模型的RMSLE为0.443，而专业营养师为0.176，差距达152%。
- **深度信息价值明确**：原论文证明，加入深度信息后卡路里MAE从70.6降至47.6 kcal，体积辅助进一步降至41.3 kcal。


## 4. SimpleFood45

**论文**：Vinod et al., *Food Portion Estimation via 3D Object Scaling*, CVPR 2024 Workshop
**链接**：https://arxiv.org/abs/2404.12257
**数据集**：https://lorenz.ecn.purdue.edu/~gvinod/simplefood45/
**代码**：https://gitlab.com/viper-purdue/monocular-food-volume-3d

SimpleFood45 包含约513张真实食物图像，提供食物类别、体积、重量和卡路里标注。

### 代表性结果

| 方法 | 思路 | Energy MAE/kcal↓ | Energy MAPE↓ | 来源 |
|---|---|---:|---:|---|
| 均值预测 | 始终预测均值 | 120.09 | 547.34% | MFP3D论文 |
| RGB Only | 仅用RGB图像直接回归 | 273.56 | 222.72% | MFP3D论文 |
| Density Map Only | 仅用密度图估计 | 216.73 | 159.48% | MFP3D论文 |
| Density Map Summing | 密度图求和后换算 | 192.76 | 93.16% | MFP3D论文 |
| 3D Assisted | 引入3D信息辅助体积估计 | 32.01 | 25.13% | MFP3D论文 |
| MFP3D | 单目图像+3D点云学习几何特征 | 29.38 | 24.03% | MFP3D论文 |
| PortionNet | 训练时蒸馏点云知识，推理时仅用RGB | — | 12.17% | PortionNet跨数据集评估 |

主要结论：单纯使用RGB图像效果较差；3D几何信息对食物分量估计非常重要；MFP3D和PortionNet通过引入或蒸馏3D信息明显降低了误差；但不同方法的输入信息不同，不能视为同一条件下的公平比较。


## 5. ECUSTFD

**原始论文**：Liang & Li, *Computer vision-based food calorie estimation: dataset, method, and experiment*, arXiv 2017
**链接**：https://arxiv.org/pdf/1705.07632

**后续论文**：Integrative AI driven microbiome analysis for optimizing sports nutrition and enhancing athletic performance through personalized dietary interventions
**链接**：https://www.frontiersin.org/journals/nutrition/articles/10.3389/fnut.2026.1754203/full

ECUSTFD 包含约2978张图像、19类食物，并提供食物体积和重量信息。原始论文使用俯视图和侧视图进行检测、体积估计和卡路里计算。

### 统一实验结果

| 方法 | 思路 | MAE/kcal↓ | RMSE↓ | MAPE↓ | R²↑ |
|---|---|---:|---:|---:|---:|
| Random Forest | 传统机器学习回归 | 61.4 | 84.7 | 16.1% | 0.804 |
| XGBoost | 梯度提升树回归 | 58.2 | 80.9 | 15.3% | 0.821 |
| MLP | 多层感知机回归 | 54.6 | 76.5 | 14.2% | 0.842 |
| 1D CNN | 一维卷积回归 | 53.1 | 74.8 | 13.8% | 0.850 |
| ResNet-50 | 深度残差网络回归 | 49.5 | 70.1 | 12.9% | 0.869 |
| MobileNetV3-Large | 轻量级网络回归 | 50.9 | 72.0 | 13.3% | 0.861 |

> 注：以上结果来自 Frontiers in Nutrition 2026 论文的对比表，实验设定统一。ECUSTFD 原始论文本身没有报告端到端的卡路里 MAE，它评估的是体积估计的准确性。


## 6. CC-Food-100

**论文**：Gao et al., *A vision-based dietary survey and assessment system for college students in China*, Food Chemistry, 2025
**DOI**：10.1016/j.foodchem.2024.141739
**GitHub**：https://github.com/zichengzichengzi/CC-FOOD-100

CC-Food-100 主要用于中国大学食堂场景下的食物检测和识别，包含RGB图像和深度图像。核心标注是食物类别、深度信息和边界框。

### 方法思路与结果

| 方法 | 思路 | 评估指标 | 结果 |
|---|---|---|---|
| YOLOv8 | 检测食物类别和位置 | AP/mAP | 优于其他深度学习模型（具体数值未报告） |
| SAM | 辅助分割，改进体积估计 | 体积估计精度 | 提升标注和体积估计精度 |
| 营养数据库查询 | 基于检测+体积结果查表计算卡路里 | 卡路里 MAE | 未报告 |

论文主要报告食物检测性能，没有报告统一的卡路里 MAE、RMSE 或 R²。因此，CC-Food-100 更适合作为食物检测数据集或RGB-D食物识别数据集，不应与 Nutrition5k 或 SimpleFood45 的卡路里回归结果直接比较。

##补充：pic2kcal（插入到 ECUSTFD 之后、CC-Food-100 之前，作为新的一节）

论文：Ruede et al., Multi-Task Learning for Calorie Prediction on a Novel Large-Scale Recipe Dataset Enriched with Nutritional Information, arXiv 2011.01082（ICPR 2020 workshop）
链接：https://arxiv.org/abs/2011.01082
代码：https://github.com/phiresky/pic2kcal/

pic2kcal 是一个从德国菜谱网站爬取的大规模真实场景数据集：70,000 份菜谱、308,000 张图片，卡路里真值不靠用户自报，而是通过食材文本匹配营养数据库反推（食材量+每份克数→加总卡路里）。数据集按 70%/15%/15% 划分训练/验证/测试，并保证同一菜谱的图片不跨集分布——这个切分方式的细致程度，可以直接作为你 ECUSTFD 划分方案的参考模板。

代表性结果（按每100g食物预测，DenseNet121 主干）
方法	思路	相对误差↓	kcal MAE↓
随机基线	随机取一份菜谱的值作为预测	0.595	83.3
均值基线	始终预测训练集均值	0.464	60.5
仅卡路里回归	单任务，只预测卡路里	0.362	50.3
+宏量营养素	同时预测蛋白质/脂肪/碳水	0.345	49.0
+食材多标签（完整多任务）	同时预测卡路里+宏量营养素+前100食材	0.326	46.9

##补充：Nutrition5k 当前权威榜单（插入到 3.4 "关键发现"之后，作为新的 3.5 节）

Nutrition5k 上有一个持续更新的公开排行榜（SOTA2 Research，聚合各论文自报的 Calories PMAE 等指标）。以下给出网页链接：

https://www.sota2.com/research/sota/nutrition-estimation-on-nutrition5k

## 7. Food-101

**原始论文**：Bossard et al., *Food-101 – Mining Discriminative Components with Random Forests*, ECCV 2014
**链接**：https://data.vision.ee.ethz.ch/cvl/datasets_extra/food-101/

Food-101 是食物分类数据集，本身不包含卡路里、重量或体积标签。其常见使用方式是：图像 → 食物类别识别 → 营养数据库查询 → 结合估计份量计算卡路里。

### 方法思路与结果

| 方法 | 思路 | 结果 | 来源 |
|---|---|---|---|
| Toward Personalized Nutrition (2026) | YOLOv8检测 → EfficientNet-B3分类 → USDA数据库回归卡路里 | 卡路里 MAE 48.2 kcal；YOLOv8 mAP@0.5 76.7% | 2026 |
| CNN-Based Estimation (2025) | ResNet50识别 + Gemini Pro Vision估算卡路里 | 系统准确率 85%，无卡路里 MAE | 2025 |
| MobileNetV3 Mobile (2026) | MobileNetV3检测 + 外部Calories Labels数据集查表 | 测试准确率 61.79%，无卡路里 MAE | 2026 |

Food-101 更适合作为食物识别预训练数据，而不是卡路里估计的主数据集。不同论文的卡路里 ground truth 来源不同（USDA映射、VLM估算、外部数据集），填表时必须标注来源。


## 8. UECFood-100

**论文**：Kawano & Yanai, *FoodCam: Food Recognition and Calorie Estimation*, 2014
**链接**：http://www.ii.t.u-tokyo.ac.jp/~kawano/FoodRecognition/

UECFood-100 主要提供食物图像、类别和检测框，本身没有完整的卡路里标签。相关研究通常使用其食物检测能力，卡路里估计则依赖其他数据集或外部营养标签。

### 代表性多任务方法

| 方法 | 思路 | 绝对误差/kcal↓ | 相对误差↓ | 来源 |
|---|---|---:|---:|---|
| Baseline（固定卡路里） | 分类后赋固定卡路里值 | 93.6 | 32.4% | Ege & Yanai 2017 |
| VGG16单任务 | 仅回归卡路里，不预测类别 | 100.4 | 29.2% | Ege & Yanai 2017 |
| VGG16多任务 | 同时预测类别和卡路里，利用任务关联 | 96.5 | 28.0% | Ege & Yanai 2017 |
| 检测→卡路里顺序模型 | 先检测再估计卡路里，两阶段独立 | 92.5 | 27.3% | Ege & Yanai 2018 |
| 检测+卡路里多任务模型 | 检测和卡路里共享特征，联合学习 | 89.4 | 26.6% | Ege & Yanai 2018 |

> 注：这些卡路里数值来自 UECFood-100 的 15 类子集，卡路里标签来自外部数据集，不是 UECFood-100 自带的 100 类标注。


## 9. 综合比较

| 数据集 | 主要输入 | 主要任务 | 代表性最佳结果 | 适合作用 |
|---|---|---|---|---|
| MM-Food-100K | 图像+文本 | 多模态营养预测 | GPT-4o：MAE 95.8 kcal | 大规模预训练、弱监督 |
| Nutrition5k | RGB、RGB-D、体积 | 卡路里/营养回归 | MAE约33.55 kcal | 主要卡路里 benchmark |
| SimpleFood45 | RGB、3D信息 | 分量和能量估计 | MFP3D：MAE 29.38 kcal | 评估分量估计和3D信息 |
| ECUSTFD | 双视角图像 | 体积和卡路里估计 | ResNet-50：MAE 49.5 kcal | 多视角卡路里估计 |
| CC-Food-100 | RGB-D | 检测和体积估计 | 主要报告AP/mAP | 检测、分割、体积估计 |
| Food-101 | RGB | 食物分类 | 分类指标为主；卡路里 MAE 48.2 kcal | 食物识别预训练 |
| UECFood-100 | RGB、检测框 | 检测和多任务学习 | 绝对误差约89.4 kcal | 食物检测和辅助识别 |


## 10. 主要结论

1. **直接使用RGB图像预测卡路里，误差通常较大。** 单张图像难以准确反映食物的重量和体积。
2. **加入深度、体积、重量或3D信息后，性能明显提升。** 食物分量估计是卡路里估计的关键。
3. **多任务学习通常优于单任务学习。** 同时预测食物类别、食材、分量和卡路里，可以利用任务之间的关联信息。
4. **多模态模型具有较大潜力。** 图像与菜名、食材和烹饪方式结合后，可以增强模型对食物成分和营养密度的理解。
5. **不同数据集的结果不能直接混合比较。** 需要同时考虑输入模态、标签来源、评价指标和任务范式的差异。
6. **建议优先选择 Nutrition5k 或带有真实重量/体积标注的数据集。** MM-Food-100K 更适合作为多模态辅助数据集，不宜直接作为唯一的真实卡路里测试集。


## 参考文献

[^1]: [MM-Food-100K: A 100,000-Sample Multimodal Food Intelligence Dataset with Verifiable Provenance](https://arxiv.org/abs/2508.10429)

[^2]: [MFP3D: Monocular Food Portion Estimation Leveraging 3D Point Clouds](https://arxiv.org/abs/2411.10492)

[^3]: [PortionNet: Distilling 3D Geometric Knowledge for Food Nutrition Estimation](https://arxiv.org/abs/2512.22304)

[^4]: [Liang & Li, Computer vision-based food calorie estimation: dataset, method, and experiment](https://arxiv.org/pdf/1705.07632)

[^5]: [Teng et al., Multimodal Information Fusion for Food Nutrition Estimation: A Survey, Information Fusion, 2026](https://www.sciencedirect.com/science/article/pii/S1566253526004987)

[^6]: [Nutrition5k: Towards Automatic Nutritional Understanding of Generic Food, CVPR 2021](https://openaccess.thecvf.com/content/CVPR2021/html/Thames_Nutrition5k_Towards_Automatic_Nutritional_Understanding_of_Generic_Food_CVPR_2021_paper.html)

[^7]: [Frontiers in Nutrition 2026: Comparative study of food calorie estimation methods](https://www.frontiersin.org/journals/nutrition/articles/10.3389/fnut.2026.1754203/full)

[^8]: [Gao et al., A vision-based dietary survey and assessment system for college students in China, Food Chemistry, 2025](https://www.sciencedirect.com/science/article/abs/pii/S0308814624033892)

[^9]: [Bossard et al., Food-101 – Mining Discriminative Components with Random Forests, ECCV 2014](https://data.vision.ee.ethz.ch/cvl/datasets_extra/food-101/)

[^10]: [Kawano & Yanai, FoodCam: Food Recognition and Calorie Estimation, 2014](http://www.ii.t.u-tokyo.ac.jp/~kawano/FoodRecognition/)

[^11]: [Ege & Yanai, Comparison of Two Approaches for Direct Food Calorie Estimation, MVA 2017](http://madima.org/wp-content/uploads/2017/11/15-170913ege_1_ppt.pdf)

[^12]: [Ege & Yanai, Multi-task Learning of Dish Detection and Calorie Estimation, MADiMa 2018](http://madima.org/wp-content/uploads/2018/09/16_ege_poster.pdf)

[^13]: [Vinod et al., Food Portion Estimation via 3D Object Scaling, CVPR 2024 Workshop](https://arxiv.org/abs/2404.12257)
