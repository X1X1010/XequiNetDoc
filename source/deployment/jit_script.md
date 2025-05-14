# 编译模型

## 即时编译（JIT）
PyTorch 的即时编译（Just-in-Time, JIT）功能是为了提升 PyTorch 模型在部署时的运行速度和可移植性而设计的。它的核心思想是将 Python 中的动态图模型转换成静态图，从而实现更高效的执行和跨平台部署。

## 运行命令
JIT 编译的命令如下：

```shell
xeq jit -c <ckpt>.pt
```

- `--ckpt` / `-c`：模型检查点文件名
- `--device`：`cuda` 或 `cpu`，默认会检查是否有可用的 GPU，有的话使用 `cuda`，没有的话使用 `cpu`
- `--fusion-strategy`: 算子融合策略，格式为 `TYPE1,DEPTH1;TYPE2,DEPTH2;...`，详见 [torch.jit.set_fusion_strategy](https://docs.pytorch.org/docs/stable/generated/torch.jit.set_fusion_strategy.html)，默认为 `DYNAMIC,3`
- `--mode`：`lmp` 表示用于 LAMMPS 的 `pair xequinet`；`dipole` 表示用于 LAMMPS 的 `compute dipole/xequinet`, 见 [xequinet-lammps](https://github.com/X1X1010/xequinet-lammps)；`gmx` 代表 GROMACS 的 NNP，见 [NNP/MM](https://manual.gromacs.org/2025.0-beta/reference-manual/special/nnpot.html)
- `--unit-style`: LAMMPS 的单位风格，见 [lammps/unit](https://docs.lammps.org/units.html)，默认为 `metal`
- `--net-charge` 在 NNP/MM 模拟时，NNP 部分所带的净电荷数量

运行之后就可以获得相应的文件，如 `<ckpt>-lmp.jit` 或 `<ckpt>-gmx.pt`（GROMACS 只接受 `.pt` 后缀的模型文件，注意和检查点文件做区分）。

⚠ 在部署时，CUDA 12 目前会有严重的 BUG，在 C++ 程序中会无法执行，因此 PyTorh 也尽量使用 CUDA 11 的版本。

由于在跑动力学的时候一般不会带有净电荷，动力学程序的输入也没有留有相应的接口，因此如果有需要跑带电体系的时候，需要使用一些 Tricks，也就是将所带电荷数提前放进模型编译，但是这样的话该模型就变成了指定电荷数目下的特定模型，注意不要混用。