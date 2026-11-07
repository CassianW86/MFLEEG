# MFLEEG

基于 [FLEEG](https://github.com/RuiLiu-cc/FLEEG) 改进的运动想象脑电（MI-EEG）分类研究代码，用于跨数据集、跨受试者的个性化联邦学习实验。本项目对应王奕淞的本科毕业论文《联邦学习场景下跨受试者脑电图分类方法》，论文中将基于特征相似度聚合的改进方法命名为 **FFLEEG**。

## 原版项目

本仓库基于 Rui Liu 等人公开的 FLEEG 实现进行修改，沿用其本地模块与全局模块组成的分层个性化结构：本地模块适配不同数据集的通道数、采样长度等差异，全局模块通过参数聚合共享知识。

- 原版仓库：[RuiLiu-cc/FLEEG](https://github.com/RuiLiu-cc/FLEEG)
- 原版论文：[Aggregating intrinsic information to enhance BCI performance through federated learning](https://www.sciencedirect.com/science/article/pii/S0893608024000145)，Neural Networks，2024。

感谢原作者提供基础框架与公开实现。

## 主要改进

核心改进是**基于特征相似度的动态加权聚合（FS）**。原有 FedAvg 按客户端样本量分配权重；本方法进一步引入本地特征与全局中心特征的相似度，调整每轮聚合中各客户端的贡献，以缓解异构脑电数据带来的聚合偏差。

1. 客户端完成本地训练后，提取分类层之前的对齐特征，并对本地训练样本求平均。
2. 服务端对各客户端的平均特征取算术平均，得到本轮全局中心特征。
3. 计算客户端特征与全局中心的相似度，支持余弦相似度（`Cos`）、皮尔逊相关系数（`Pearson`）和欧氏距离转换的相似度（`Euclidean`，即 `1 / (1 + distance)`）。
4. 将相似度与样本量相乘、归一化后，用于聚合共享的全局层参数，本地个性化层继续保留在客户端。

聚合权重为：

$$
\alpha_k = \frac{n_k\max(s_k,\varepsilon)}{\sum_{j=1}^{K}n_j\max(s_j,\varepsilon)}
$$

其中，`n_k` 为客户端配置中的样本量，`s_k` 为特征相似度，当前实现的下限阈值为 `ε = 0.05`。

代码还提供 **FedProx 本地近端正则项**与 **SENet 脑电通道注意力模块**，可作为扩展实验选项，默认均关闭。服务端保留 `Fedavg` 和 `EqualWeights` 聚合方式，便于开展对比实验。

## 仓库结构

主要代码位于 `MFLEEG-main/`：

| 文件 | 说明 |
| --- | --- |
| `FFLEEG-MP-main.py` | 多进程联邦训练入口 |
| `ArgumentsSetup.py` | 数据路径、客户端配置与命令行参数 |
| `Clients.py` | 本地训练、特征提取、验证与测试 |
| `Server.py` | FedAvg、等权重与特征相似度聚合 |
| `model.py`、`SENet.py` | 分类网络与可选通道注意力模块 |
| `SaveSharedArray.py`、`DataLoaderBase.py` | 数据加载、共享内存与训练集划分 |

## 快速开始

建议使用 Python 3.10，依赖版本见 `MFLEEG-main/requirements.txt`。在仓库根目录执行：

```bash
cd MFLEEG-main
pip install -r requirements.txt
```

运行前，请先准备预处理后的数据，并修改 `ArgumentsSetup.py` 中的 `Common_config["data_path"]`。当前配置包含 `KoreaU_MI`、`Shin_MI`、`Weibo2014`、`Cho2017`、`MunichMI` 和 `Murat2018` 六个数据集。预处理可参考[原版脚本](https://github.com/RuiLiu-cc/FLEEG/tree/main/Data_preprocessing)，并核对输出格式与本仓库配置。

数据按 `<data_path>/<数据集名称>/<filtering_setting>/` 存放，每个被试一个 `.npz` 文件，包含 `x_data`（形状为 `试次 × 通道 × 时间点`）和 `y_data`（二分类标签 `0/1`）。通道数、时间长度和被试数应与客户端配置一致。

使用余弦相似度进行聚合：

```bash
python FFLEEG-MP-main.py --Server_aggregation FS --Similarity_strategy Cos
```

运行按样本量加权的 FedAvg 对比实验：

```bash
python FFLEEG-MP-main.py --Server_aggregation Fedavg
```

直接运行 `python FFLEEG-MP-main.py` 时，默认使用 `FS` 聚合与 `Euclidean` 相似度。默认进行 5 次训练，依次选取每个数据集文件排序后的倒数第 1 至第 5 个被试作为测试集；结果、日志和聚合权重保存在运行目录下的 `Results/` 中。

## 引用

使用本仓库开展研究时，请注明本仓库及其改进方法，并引用原始 FLEEG 论文：

```bibtex
@article{liu2024aggregating,
  title={Aggregating intrinsic information to enhance BCI performance through federated learning},
  author={Liu, Rui and Chen, Yuanyuan and Li, Anran and Ding, Yi and Yu, Han and Guan, Cuntai},
  journal={Neural Networks},
  volume={172},
  pages={106100},
  year={2024},
  publisher={Elsevier}
}
```

## 版权与许可

- 原始 FLEEG 代码：Copyright 2024 Centre for Brain Computing Research (CBCR)。
- 本仓库新增与修改部分：Copyright © 2026 王奕淞（[wys1234567890](https://github.com/wys1234567890)）。

本项目基于采用 **CBCR License 1.0** 的 FLEEG 代码，使用与再分发须遵守该许可，包括非商业用途限制、保留原始版权声明、许可条件和免责声明等要求。完整原版许可保留于 [LICENSE](LICENSE)，亦可查看[上游许可](https://github.com/RuiLiu-cc/FLEEG/blob/main/LICENSE)。本仓库新增与修改部分同样按 CBCR License 1.0 提供。

## 实验截图

<img width="692" height="126" alt="image" src="https://github.com/user-attachments/assets/87383986-8d17-4e01-ad36-2038b00f85c7" />
<img width="644" height="353" alt="image" src="https://github.com/user-attachments/assets/9a7a7fdc-f957-41ba-8094-23802ef9002e" />
<img width="693" height="606" alt="image" src="https://github.com/user-attachments/assets/1ed90e52-3cc2-41aa-811a-21c98a811b48" />
