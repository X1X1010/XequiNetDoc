# 测试

## 准备数据
同样按照[制作数据集](../dataset/create_data.md)部分准备和如下放置数据集。

```shell
<path_to_data>
  ├─ data.lmdb
  ├─ info.json
  └─ <split>.json
```

训练的时候，只需要读取 `<split>,json` 中的 `"test"` 部分的 Index。

## 准备配置文件
在工作目录设置配置文件 `config.yaml`，[模型部分](./model_cfg.md)和训练保持一致即可；[数据部分](./data_cfg.md)指定测试用的数据集，如果和训练集相同则不用修改；[训练部分](./trainer_cfg.md)可以不要。

CONFIG 文件如下：

```yaml
model:
  model_name: xpainn
  model_kwargs:
    node_dim: 128
    node_irreps: 128x0e+64x1o+32x2e
    ...
  default_units:
    energy: eV
    pos: Ang
data:
  db_path: /scratch/.../dataset/
  cutoff: 5.0
  split: random42
  targets: [energy, forces]
  ...
```

上述 CONFIG 读取 `db_path` 下的 `data.lmdb`，然后根据 `random42.json` 中的 `"test"` 字段的索引作为测试集进行测试，其中测试批次的大小根据 CONFIG 文件中的 `batch_size` 字段设置。

## 准备模型文件
准备好训练完成的模型检查点文件，`xxx.pt`。

## 开始测试

```bash
xeq test --config config.yaml --ckpt xxx.pt
```

其他命令行参数如下：
- `--config` / `-C`：测试用的 CONFIG 文件的名字，默认为 `config.yaml`
- `--ckpt` / `-c`：检查点模型文件名
- `--device`：`cuda` 或 `cpu`，默认会检查是否有可用的 GPU，有的话使用 `cuda`，没有的话使用 `cpu`
- `--output` / `-o`：输出文件的名字，默认为 `<run_name>.log`，`<run_name>` 为 CONFIG 文件中指定的任务名
- `--verbose` / `-v`：是否打印更详细的输出，并将测试结果保存到 `.pt` 文件中。

## 示例
我们接着使用 Water 1593 这个数据集作为例子（<https://doi.org/10.1073/pnas.1815117116>）。经过[训练](./train.md#示例)之后会获得模型文件——`water_0.pt`，然后 CONFIG 文件沿用训练的 CONFIG 即可。

执行命令

```bash
xeq test --config config.yaml --ckpt water_0.pt --verbose
```

这里我们输出的详细一些，之后会获得两个文件，一个是 `water_test.log`，记录每个结构的预测和真实值以及总体误差，大致如下：

```
...
Cell            x          y          z
a       12.537996   0.000000   0.000000
b       -0.000001  12.537996   0.000000
c       -0.000001  -0.000001  12.537996
Atom            x          y          z     Pred Fx     Pred Fy     Pred Fz     Real Fx     Real Fy     Real Fz
O        3.448780   2.593947  10.376476    0.486231    1.216614   -6.801627    0.507661    1.197034   -6.773417
H        3.213423   2.359072  11.194902   -1.625152   -1.201921    4.734303   -1.537880   -1.257017    4.766877
H        2.729660   3.125744   9.859947    2.210131   -1.178337    1.859693    2.089911   -1.119479    1.749672
...
O        3.527506  11.306347   7.186543    3.705233    3.209195   -3.313435    3.769155    3.049925   -3.100159
H        3.648962  10.402830   7.362230   -0.291566   -1.858727    0.181758   -0.256045   -1.944309    0.041742
H        4.433796  11.549081   6.384364   -4.000643   -0.291637    3.824838   -3.955241   -0.085130    3.710076
Pred Energy: -594.282593  Real Energy: -594.163086
...

            Forces    Energy    Energy/atom
Test MAE   0.072082  0.237395    0.001236
Test RMSE  0.138642  0.351181    0.001829
```

另一个文件是 `water_test_results.pt`，需要用 PyTroch 载入，如下：

```python
import torch
results = torch.load("water-bulk_test_results.pt")
```

这个 `results` 是一个字典，一个键为 `"data"`，值是一个大 DataBatch，（有关 Batch 操作可详见[批操作](../dataset/batch.md)），另一个键为，`"results"`，值是一个字典，包含预测的结果。根据这些结果可以做进一步的统计，这里不再赘述了。