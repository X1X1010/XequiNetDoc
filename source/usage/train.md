# 训练
## 准备数据
按照[制作数据集](../dataset/create_data.md)部分准备数据集，按照如下方式放置数据。

```shell
<path_to_data>
  ├─ data.lmdb
  ├─ info.json
  └─ <split>.json
```

条件允许的话尽量放在节点的固态硬盘上，然后避免多个任务读同一个 LMDB 文件。

## 准备配置文件
在工作目录设置配置文件 `config.yaml`，该 YAML 文件分成三部分，[模型部分](./model_cfg.md)、[数据部分](./data_cfg.md)和[训练部分](./trainer_cfg.md)，参数设置详见相应链接。

配置完的 CONFIG 文件如下：

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
trainer:
  run_name: xxx
  lossfn: smoothl1
  ...
```

## 开始训练

目前训练只支持英伟达 GPU，执行命令如下：

```bash
torchrun --nproc-per-node=<n_gpu> --master-port=<port> --no-python \
xeq train --config config.yaml
```

首先是 PyTorch 自己的命令行的参数
- `torchrun`：用于启动 PyTorch 数据分布式并行的 Python 脚本，见 [TorchRun](https://pytorch.org/docs/stable/elastic/run.html)
- `--nproc-per-node`：GPU 个数
- `--master-port`：主节点的端口号，用于和其他节点通信，注意端口号不要冲突
- `--no-python`：因为后面执行的是 Entry Point，而不是 Python 文件，所以要加这个命令

接下来是 XequiNet 的训练命令
- `xeq`：XequiNet 的 Entry Point
- `train`：表示执行训练模块
- `config` / `-C`：CONFIG 文件的名字，默认为 `config.yaml`

## 示例
我们使用 Water 1593 这个数据集作为例子（https://doi.org/10.1073/pnas.1815117116）

