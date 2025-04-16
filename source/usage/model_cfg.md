# 模型超参数设置
## 模型选择

| 关键词 | 类型 | 默认值 | 描述 |
| - | - | - | - |
| `model_name` | `str` | `xpainn` | 使用的模型名字 |

现在也没得选，也就 `xpainn` 这一种模型。虽然还有一种叫 `xpainn-ewald`，这个是 XPaiNN 的基础上加了 Ewald-MP （<https://doi.org/10.48550/arXiv.2303.04791>）但是效果没有得到验证，不知道好不好用。

## 模型超参数

| 关键词 | 类型 | 默认值 | 描述 |
| - | - | - | - |
| `model_kwargs` | `dict[str, Any]` | - | 模型具体的超参数 |

### XPaiNN 超参数

原子嵌入

| 关键词 | 类型 | 默认值 | 描述 | 可选参数 |
| - | - | - | - | - |
| `embed_basis` | `str` | `gfn2-xtb` | 用于原子嵌入的原子轨道基 | `gfn2-xtb`, `gfn1-xtb`, `one-hot` |
| `aux_basis` | `str` | `aux56` | 原子轨道基投影到的辅助基组 | `aux28`, `aux56` |

节点特征

| 关键词 | 类型 | 默认值 | 描述 |
| - | - | - | - |
| `node_dim` | `int` | `128` | 标量特征的长度 |
| `node_irreps` | `str` | `128x0e + 64x1o + 32x2e` | 等变特征的不可约表示 |

径向基函数

| 关键词 | 类型 | 默认值 | 描述 | 可选参数 |
| - | - | - | - | - |
| `rbf_kernel` | `str` | `bessel` | 径向基函数类型 | `bessel`, `gaussian` |
| `num_basis` | `int` | `20` | 径向基函数数量 | |
| `cutoff_fn` | `str` | `cosine` | 平滑截断函数的类型 | `cosine`, `polynomial` |
| `cutoff` | `float` | `5.0` | 截断半径长度 | |

其他参数

| 关键词 | 类型 | 默认值 | 描述 |
| - | - | - | - |
| `action_blocks` | `int` | `3` | 消息传递和节点更新的层数 |
| `activation` | `str` | `silu` | 激活函数类型 | `silu`, `relu`, `leakyrelu`, `softplus` |
| `layer_norm` | `bool` | `true` | 是否使用层归一化 |

电子嵌入

| 关键词 | 类型 | 默认值 | 描述 |
| - | - | - | - |
| `charge_embed` | `bool` | `false` | 是否使用电荷嵌入层 |
| `spin_embed` | `bool` | `false` | 是否使用自旋嵌入层 |

输出相关
| 关键词 | 类型 | 默认值 | 描述 | 可选参数 |
| - | - | - | - | - |
| `output_mode` | `str`/`list[str]` | `energy` | 输出层的类型，可以有多重输出 | `energy`, `charges`, `polar`, `dipole`, `polar` |
| `hidden_dim` | `int` | `64` | 输出层 MLP 的中间层维度 | |
| `hidden_irreps` | `str` | `64x0e + 32x1o + 16x2e` | 张量输出的 等变 MLP 中间层不可约表示 | |
| `conservation` | `bool` | `true` | 电荷布居输出专用，是否遵循电荷守恒 | |
| `magnitude` | `bool` | `false` | 偶极矩输出专用，是否转化为偶极矩大小 | |
| `isotropic` | `bool` | `false` | 极化率输出专用，是否转化为各项同性极化率 | |


## 单位设置

| 关键词 | 类型 | 默认值 | 描述 |
| - | - | - | - |
| `default_units` | `dict[str, str]` | - | 模型使用的默认单位 |

设置模型所使用的单位，具体可用单位见[单位](./units.md)。主要就是输入结构的坐标的单位，和输出物理量的单位，如：

```yaml
model:
  ...
  default_units:
    pos: Angstrom
    energy: eV
```

表述输入坐标单位是 Å，输出能量单位是 eV。要注意的是对于梯度性质，比如力，会自动根据能量和坐标的单位组合出 eV/Å，不要单独指定以免引起歧义。
