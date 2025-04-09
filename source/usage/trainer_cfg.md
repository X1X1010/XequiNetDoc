# 训练参数设置
## 检查点文件

| 关键词 | 类型 | 默认值 | 描述 |
| - | - | - | - |
| `run_name` | `str` | `xequinet` | 训练任务的名称，检查点文件将以此命名 |
| `resume` | `bool` | `false` | 是否断点续训 |
| `ckpt_file` | `str` | `null` | 用于载入参数的检查点文件名 |
| `finetune_modules` | `list[str]` | `null` | 含有列表中关键词的模型参数才会被优化 |

### 示例 1：断点续训
```yaml
trainer:
    run_name: spice-v1
    ckpt_file: spice-v1_last.pt
    resume: true
```
训练开始时会读取 `spice-v1_last.pt` 检查点文件，读取模型参数的同时，还会读取 Epoch 信息，优化器参数和学习率调整器的参数等。接着从检查点文件断开的 Epoch 开始继续训练。

### 示例 2：微调模型
```yaml
trainer:
    run_name: spice-v2
    ckpt_file: spice-v1.pt
    resume: false
    finetune_modules: [output, embed]
    ...
```
训练开始时将会读取 `spice-v1.pt` 检查点文件，只载入其中的模型参数。训练过程中仅优化名字中带有 `output` 和 `embed` 的参数（如 `module.mods.embedding.embedding.1.weight` 和 `module.mods.output_energy.out_mlp.0.weight` 等）。接着从头开始训练。训练过程中保存的参数将会命名成 `spice_v2_0.pt` 和 `spice_v2_last.pt` ，前者是验证误差最好的模型，后者是上一个 Epoch 结束时自动保存的检查点模型。

## 学习率

| 关键词 | 类型 | 默认值 | 描述 |
| - | - | - | - |
| `max_lr` | `float` | `5e-4` | 学习率最大值 |
| `min_lr` | `float` | `0.0` | 学习率最小值 |

学习率变化的上限和下限，在不同的学习率调整器下含义各不相同。

## 学习率预热

| 关键词 | 类型 | 默认值 | 描述 | 可选参数 |
| - | - | - | - | - |
| `warmup_scheduler` | `str` | `linear` | 学习率预热方式（不区分大小写） | `null`, `linear`, `exponential`, `untuned_linear`, `untuned_exponential`, `RAdam` |
| `warmup_epochs` | `int` | `10` | 学习率预热 Epoch 数 | - |

学习率预热是使用的 [pytorch-warmup](https://pypi.org/project/pytorch-warmup/) 库，简单来说就是将学习率缓缓从 0 开始上升，避免初始学习率过大导致训练不稳定的问题。`warmup_epochs` 仅针对 `linear` 和 `exponential` 生效，程序会将预热的 Epoch 数量换算成梯度下降的步数。以余弦学习率变化为例，下图为 2500 步（不是 Epoch）预热的学习率变化图：

![Warmup](../figures/Warmup.png)

相当于在学习率上 $\alpha_t$ 乘上一个预热系数 $\omega_t \in [0, 1]$，即

$$
    \hat{\alpha}_t = \alpha_t \cdot \omega_t
$$

### 线性预热
线性预热分两种，手动设置预热步数的 `linear` 和自动根据 Adam 类优化器的 $\beta_2$ 值设定预热步数的 `untuned_linear`。其预热系数为：

$$
    \omega_t = \min(1, \frac{t}{\tau})
$$

其中，$t$ 为当前步数，$\tau$ 为预热步数，对于 `untuned_linear`，$\tau = \lfloor \frac{2}{1-\beta_2} \rfloor$

### 指数预热
指数预热同样分两种，预热系数为（这了和官方不一样，官方这里没有 2，我这里是为了和 `untuned_exponential` 保持一致）：

$$
    \omega_t = 1 - \exp(-\frac{2t}{\tau})
$$

同样，$t$ 为当前步数，$\tau$ 为预热步数，对于 `untuned_exponential`，$\tau = \lfloor \frac{2}{1-\beta_2} \rfloor$

### RAdam 预热
这个有点复杂，懒得写了，看[官方解释](https://tony-y.github.io/pytorch_warmup/v0.2.0/radam_warmup.html)吧🤭

## 损失函数

| 关键词 | 类型 | 默认值 | 描述 | 可选参数 |
| - | - | - | - | - |
| `lossfn` | `str` | `smoothl1` | 损失函数类型（不区分大小写）|  `mae`(`l1`), `mse`(`l2`), `smoothl1` |
| `losses_weight` | `dict[str, float]` | - | 误差函数各项组分的权重 |

### 损失函数种类
这里可以用的损失函数有三种，L1 Loss、L2 Loss 和 两者兼具的 SmoothL1 Loss。三者的函数形式和图像如下（图中为了美观给 L2 Loss 成了一个 0.5 的系数）：

![Loss](../figures/Loss.png)

- MAE Loss，即 L1 Loss：

$$
    l(\hat{y}, y) = | \hat{y} - y |
$$

- MSE Loss，即 L2 Loss：

$$
    l(\hat{y}, y) = (\hat{y} - y)^2
$$

- SmoothL1 Loss：

$$
    l(\hat{y}, y) =  \begin{cases}
        0.5(\hat{y} - y)^2    & | \hat{y} - y | > 1 \\
        | \hat{y} - y | - 0.5 & | \hat{y} - y | \le 1 \\
    \end{cases}
$$

式中 $\hat{y}$ 表示真值，$y$ 表示预测值。

### 损失权重
`losses_weight` 接受一个字典，用于给损失函数的各组分分配一个权重。比较常见的情况时训练力场模型时同时监督能量和力，我们希望能量和力误差的权重比时 1:1000，就可以这样设置：
```yaml
trainer:
    ...
    lossfn: smoothl1
    losses_weight:
        energy: 1.0
        forces: 1000.0
    ...
```
这样就代表着总损失 $\mathcal{L} = 1 \times l(\hat{E}, E) + 1000 \times l(\hat{\mathbf{F}}, \mathbf{F})$。

当然 `energy` 和 `forces` 须在数据参数的 `target` 栏中声明，这样程序才会从数据集中读取相应的标签用于损失计算。