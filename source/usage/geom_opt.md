# 几何结构优化和振动分析
## 输入文件
和推理任务的[输入文件](./inference.md#输入文件)要求相同，也支持连续文件，虽然处理的时候是一个一个处理的。

## 结构优化
几何结构优化的功能是使用 [PySCF](https://pyscf.org/) 提供的 [geomeTRIC](https://geometric.readthedocs.io/en/latest/index.html) 库的接口实现的。命令如下：

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

## 示例
首先准备模型文件，我们用在 SPICE 上训练的模型 `xpainn-spice-v1.pt`，模型可以在 [Zenodo 链接](https://zenodo.org/records/14676636)下载。简单起见，我们拿水分子作为例子，输入文件 `water.xyz` 如下：

```
3

O          0.00000        0.01840       -0.00000
H         -0.00000       -0.53835        0.78304
H          0.00000       -0.53835       -0.78304
```

执行命令，给能开的选项都打开：

```shell
xeq opt -c xpainn-spice-v1.pt -in water.xyz --freq --shermo --save-hessian --verbose
```

过程中 geomeTRIC 程序会在标准输出流打印优化的详细过程。结束后会出现如下文件。首先是 `water_opt.xyz`，为优化完成后的结构。

```
3
Energy: -0.4127226288 Hartree
O    -0.000000    0.051452   -0.000000
H     0.000000   -0.554876    0.750175
H    -0.000000   -0.554876   -0.750175
```

然后是 `water_freq.log`，记录了振动分析的信息。


```
XequiNet Frequency Calculation

Mode                                 0                   1                   2
Irrep
Freq [cm^-1]                     1570.3419           4068.7545           4175.7368
Reduced mass [au]                   1.0719              1.0557              1.0796
Force const [Dyne/A]                1.5574             10.2973             11.0917
Char temp [K]                    2259.3723           5854.0318           6007.9556
Normal mode                   x     y     z       x     y     z       x     y     z
       0   O                -0.00  0.06 -0.00   -0.00 -0.05  0.00    0.00  0.00  0.07
       1   H                -0.00 -0.50 -0.46    0.00  0.44 -0.53   -0.00  0.43 -0.53
       2   H                 0.00 -0.50  0.46    0.00  0.44  0.53    0.00 -0.43 -0.53
temperature 298.1500 [K]
pressure 101325.00 [Pa]
Rotational constants [GHz] 910.64198 408.84509 282.16381
Symmetry number 2
Zero-point energy (ZPE) 0.02236 [Eh]   58705.739 [J/mol]
                           tot       elec      trans        rot        vib
Entropy [J/mol/K]       188.456      0.000    144.804     43.615      0.037
Cv [J/mol/K]             25.188      0.000     12.472     12.472      0.245
Cp [J/mol/K]             33.502      0.000     20.786     12.472      0.245
Internal energy [J/mol]                      3718.434   3718.434  58715.355
Internal energy [Eh]   -0.38753  -0.41272
Enthalpy [J/mol]                             6197.391   3718.434  58715.355
Gibbs free energy [J/mol]                  -36975.931  -9285.410  58704.471
Gibbs free energy [Eh] -0.40798
```

然后还有 Shermo 输入文件 `water_freq.shm`，和 Hessian 矩阵 `water_h.txt`，可以直接用 NumPy 读。
