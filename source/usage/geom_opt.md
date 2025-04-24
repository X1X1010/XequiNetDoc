# 几何结构优化和振动分析
## 输入文件
和推理任务的[输入文件](./inference.md#输入文件)要求相同，也支持连续文件，虽然处理的时候是一个一个处理的。

## 结构优化
几何结构优化的功能是使用 [PySCF](https://pyscf.org/) 提供的 [geomTRIC](https://geometric.readthedocs.io/en/latest/index.html) 库的接口实现的。命令如下：

```shell
xeq opt -c <ckpt>.pt -in <mol>.xyz
```

其他相关的命令行参数为

- `--ckpt` / `-c`：模型检查点文件名
- `--device`：`cuda` 或 `cpu`，默认会检查是否有可用的 GPU，有的话使用 `cuda`，没有的话使用 `cpu`
- `--input` / `-in`：输入文件名
- `opt-params`：这个是用一个 `.json` 文件来设定优化的选项，具体的参数可以看[优化选项](https://geometric.readthedocs.io/en/latest/options.html#optimization-parameters)
- `constraints`: 设定限制性优化的文件，具体可以看[限制性优化](https://geometric.readthedocs.io/en/latest/constraints.html)

（上述两个选项设置的不是很好，要设定这些设定还是比较麻烦的，希望能提供一些建议，怎么传入这些参数比较好）

- `--max-steps`：最大优化迭代次数
- `--delta` / `-d`：Δ-ML 模型用的底层的半经验方法，可选项：`gfn2-xtb`, `gfn1-xtb`

## 振动分析

一般来说振动分析是跟在结构优化后面的，所以完整的命令如下：

```shell
xeq opt -c <ckpt>.pt -in <mol>.xyz --freq
```

具体的相关命令行参数如下：

- `--freq`：是否进行振动分析
- `--temp` / `-T`：振动分析使用的温度，默认为 298.15 K
- `--no-opt`：不进行结构优化，直接振动分析，但是要小心虚频
- `--shermo`：是否保存成 [Shermo](http://sobereva.com/soft/shermo/) 的输入文件用于 Quasi-harmonic 分析
- `--save-hessian`：是否保存 Hessian 矩阵
- `--verbose` / `-v`：是否打印详细的振动分析的结果
