# LAMMPS 接口
## 环境安装
和安装 Conda 环境一样，拉取 Github 仓库并安装环境。

```shell
git clone https://github.com/X1X1010/xequinet-lammps.git
cd xequinet-lammps
conda env create -f scripts/environment.yaml -n <env_name>
```

因为在手动编译时会需要用到 NVCC 等工具，所以手动安装的时候注意需要额外安装 [cuda-nvcc](https://anaconda.org/nvidia/cuda-nvcc)。

## 编译 LAMMPS
执行自动编译脚本

```shell
bash scripts/build.sh <env_name> <nproc>
```

上述命令中，`<env_name>` 是 Conda 环境的名字，`<nproc>` 是编译所使用的线程数。编译完成之后会生成可执行文件 `bin/lmp`，可以将其添加到系统的路径当中。

⚠ Cmake 的版本一定要在 3.16 和 **3.24** 之间，版本太高会有问题。

如果需要修改编译命令，比如打开某些 LAMMPS package 的开关，可以在 `scripts/build.sh` 文件中以下部分增删选项。

```
...
cmake -D CMAKE_INSTALL_PREFIX=$root_dir \
      -D CUDA_TOOLKIT_ROOT_DIR=$conda_dir \
      -D CMAKE_CUDA_ARCHITECTURES=native \
      -D PKG_MOLECULE=on \
      -D PKG_CORESHELL=on \
      -D PKG_DIPOLE=on \
      -D PKG_DRUDE=on \
      -D PKG_KSPACE=on \
      -D PKG_REAXFF=on \
      -D PKG_QEQ=on \
...
```

## 使用

在使用前，需要在你的终端中或者任务提交脚本中设置以下环境变量：

```bash
source activate archetype

libtorch_path=$(realpath $(dirname $(python -c 'import torch; print(torch.__file__)')))/lib
conda_lib_path=$(realpath $(dirname $(which python))/..)/lib

export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/.../path_to_lammps/lib64
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$libtorch_path:$conda_lib_path
```

LAMMPS 的输入卡的设置方法如下：

```
pair_style   xequinet/xequinet32/xequinet64
pair_coeff   * * <model>.jit <element1> <element2> ... <elementn>
```

其中 `xequinet` 或 `xequinet32` 指使用 FP32 精度，`xequinet64` 指使用 FP64 精度。`<model>.jit` 是 JIT 编译的模型的路径。`<elementi>` 是 LAMMPS 的结构文件中所包含的元素的种类，而且顺序必须和元素种类编号的顺序一致。
