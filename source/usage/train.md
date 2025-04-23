# 训练
## 准备数据
按照[制作数据集](../dataset/create_data.md)部分准备数据集，按照如下方式放置数据。

```shell
<path_to_data>
  ├─ data.lmdb
  ├─ info.json
  └─ <split>.json
```

训练的时候，只需要读取 `<split>,json` 中的 `"train"` 和 `"valid"` 部分的 Index 就行了。条件允许的话尽量将上述文件放在节点的固态硬盘上，然后避免多个任务读同一个 LMDB 文件。

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
我们使用 Water 1593 这个数据集作为例子（<https://doi.org/10.1073/pnas.1815117116>）

我们首先准备数据集([下载链接](https://github.com/X1X1010/XequiNetDoc/raw/refs/heads/main/source/example/water-1593.zip))，解压放在某处，比如`/scratch/datasets/water-1593/`

目录如下

```shell
/scratch/datasets/water-1593
  ├─ data.lmdb
  ├─ info.json
  └─ random42.json
```

然后准备 `config.yaml` 文件，如下：

```yaml
model:
  model_name: xpainn
  model_kwargs:
    node_dim: 128
    node_irreps: 128x0e+64x1o+32x2e
    embed_basis: gfn2-xtb
    aux_basis: aux56
    num_basis: 20
    rbf_kernel: bessel
    cutoff: 5.0
    cutoff_fn: cosine
    action_blocks: 3
    activation: silu
    norm_type: layer
    hidden_dim: 64
    output_modes: [energy]
  default_units:
    energy: eV
    pos: Angstrom

data:
  db_path: /scratch/datasets/water-1593
  cutoff: 5.0
  split: random42
  targets: [energy, forces]
  default_dtype: float32
  batch_size: 16
  valid_batch_size: 16

trainer:
  run_name: water
  warmup_scheduler: linear
  warmup_epochs: 10
  max_epochs: 100
  max_lr: 5e-4
  min_lr: 0.0
  lossfn: smoothl1
  losses_weight:
    energy: 1.0
    forces: 100.0
  optimizer: AdamW
  lr_scheduler: cosine_annealing
  lr_scheduler_kwargs:
    T_max: 100
  ema_decay: 0.995
  num_workers: 2
  
  save_dir: ./
  best_k: 1
  log_steps: 100
  log_file: loss.log
```

之后运行命令

```bash
torchrun --nproc_per_node=1 --master_port=30000 --no-python xeq train --config config.yaml > net_config.log 2>&1
```

接着会出现若干文件，其中一个是 `net_config.log`，这个是将标准输出流的内容重定向到的文件，主要信息包括模型的结构，

```
INFO - XPaiNN(
  (mods): ModuleDict(
    (embedding): XEmbedding(
      (embedding): Sequential(
        (0): Int2c1eEmbedding()
        (1): Linear(in_features=56, out_features=128, bias=True)
      )
      (sph_harm): SphericalHarmonics()
      (rbf): SphericalBesselj0()
      (cutoff_fn): CosineCutoff()
    )
    (message_0): XPainnMessage(
...
```

以及参数名和参数量（当然还有警告和报错信息🤡）。

```
INFO - module.mods.embedding.embedding.1.weight: 7168
INFO - module.mods.embedding.embedding.1.bias: 128
INFO - module.mods.embedding.rbf.freq: 20
...
INFO - module.mods.output_energy.out_mlp.2.weight: 64
INFO - module.mods.output_energy.out_mlp.2.bias: 1
INFO - Total number of parameters to be optimized: 865141
```

另一个文件是 `loss.log`，事实记录误差情况。

```
2025-04-15 18:50:45 - INFO - Start training
2025-04-15 18:50:45 - INFO - Task Name: water
2025-04-15 18:50:45 - INFO - Property: pos Unit: Angstrom
2025-04-15 18:50:45 - INFO - Property: energy Unit: eV
2025-04-15 18:50:45 - INFO - Property: forces Unit: eV/Angstrom
2025-04-15 18:50:45 - INFO - Property: virial Unit: eV/Angstrom^3
2025-04-15 18:52:01 - INFO -               Epoch     Step         LR    Energy    Energy/atom    Forces
2025-04-15 18:52:01 - INFO - Train  MAE        1  [79/79]  5.062e-05     57.82         0.3012    0.7386
2025-04-15 18:52:01 - INFO - Train RMSE                                  62.82         0.3272    1.012
2025-04-15 18:52:06 - INFO -                   Energy    Energy/atom    Forces
2025-04-15 18:52:06 - INFO - EMA Valid  MAE       556          2.896     1.177
2025-04-15 18:52:06 - INFO - EMA Valid RMSE     556.2          2.897     1.768
...
```

另外就是检查点文件了，因为 `best_k` 设的 1，所以会报错 MAE 最低的 1 个模型，即 `water_0.pt`；另外会保存上一个 Epoch 结束时的模型 `water_last.pt`。

训练完成之后 `loss.log` 会出现以下信息：
```
...
2025-04-15 20:36:18 - INFO - Training Completed
2025-04-15 20:36:18 - INFO - Best Valid MAE: 7.19884
2025-04-15 20:36:18 - INFO - Best Checkpoint: ./water-bulk_0.pt at Epoch 100
```

这里的最好的 MAE 是乘以 Loss 权重之后的结果，最好的结果来自第 100 个 Epoch。当然这个模型只是随便训训的，效果可能不怎么样😄。