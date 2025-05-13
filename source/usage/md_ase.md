# 分子动力学 ASE

一些简单的 MD 任务可以使用 ASE 进行。

## 配置文件
首先需要准备 MD 任务的配置文件，具体选项如下：

|  关键词 | 类型 | 默认值 | 描述 |
| - | - | - | - |
| `emsenbles` | `list[dict]` | 系综设置，具体可以参见 [ASE MD](https://wiki.fysik.dtu.dk/ase/ase/md.html) 模块. (`logfile` 和 `trajectory` 需要在之后独立设置) | - |
| `input_file` | `str` | 输入坐标文件 | `input.xyz` |
| `model_file` | `str` | 模型检查点文件 | `model.pt`|
| `delta_method` | `str` | Δ-ML 的基底方法，包括 `GFN2-xTB` 或 `GFN1-xTB`  | `null` |
| init_temperature | `float` | 初始温度，用于生成 Maxwell-Boltzmann 分布 | `300.0` |
| `logfile` | `str` | 用于记录的文件名 | `md.log` |
| `append_logfile` | `bool` | 是否追加写入记录文件 | `False` |
| `trajectory` | `str` | 用于保存 ASE 轨迹文件的文件名 | `null` |
| `append_trajectory` | `bool` | 是否追加写入轨迹文件 | `False` |
| `xyz_traj` | `str` | 将 ASE 轨迹文件转化为 XYZ 文件的文件名 | `null` |
| `columns` | `list[str]` | XYZ 文件的内容，可选项包括 [`positions`, `numbers`, `charges`, `symbols`] | `[symbol, position]` |

## 运行模拟

运行的命令如下：

```shell
xeq md --config <md_config>.yaml
```

`--config` / `-C`：MD 配置文件

## 示例

我们用一个简单的乙醇分子作为例子，`ethanol.xyz` 文件如下：
```
9

C         -1.09349        0.55265       -0.00631
C         -0.05821       -0.56395        0.03427
O          1.23615       -0.02850        0.07966
H         -2.11427        0.11600       -0.02264
H         -0.95185        1.17082       -0.91799
H         -0.99150        1.19819        0.89170
H         -0.17341       -1.23408       -0.84802
H         -0.21882       -1.18197        0.94271
H          1.47083        0.22681       -0.85074
```

我们先跑一个能量最小化，使用 FIRE 算法。然后跑 Berendsen 系综的 NVT，配置文件 `md_config.json` 如下：

```yaml
input_file: ethanol.xyz
model_file: xpainn-spice-v1.pt

init_temperature: 300.  # Kelvin

ensembles:
- name: FIRE
  fmax: 0.05  # eV/Å
- name: NVTBerendsen
  timestep: 1.  # fs
  taut: 100.  # fs
  temperature: 300.  Kelvin
  loginterval: 1
  steps: 100

logfile: ethanol_md.log
trajectory: ethanol.traj

xyz_traj: ethanol_traj.xyz

columns: ["symbols", "positions"]
```

运行时，会出现 `ethanol_md.log` 文件记录信息，第一部分大致如下：

```
      Step     Time          Energy         fmax
FIRE:    0 19:33:25      -38.074837        1.3917
FIRE:    1 19:33:25      -38.103050        0.3436
FIRE:    2 19:33:27      -38.098492        1.2037
FIRE:    3 19:33:27      -38.104485        0.9112
...
```

记录结构优化的信息，直到最大的力小于 0.05 eV/Å 停止优化，进入 NVT 模拟部分，以 1 fs 为步长，跑 100 步，也就是 0.1 ps，每 1 步记录一次，输出如下：

```
Time[ps]      Etot[eV]     Epot[eV]     Ekin[eV]    T[K]
0.0000         -37.7606     -38.1290       0.3684   316.7
0.0010         -37.7556     -38.0653       0.3097   266.2
0.0020         -37.7475     -37.9465       0.1990   171.1
0.0030         -37.7449     -37.8849       0.1400   120.3
0.0040         -37.7481     -37.8991       0.1509   129.8
0.0050         -37.7500     -37.9124       0.1624   139.6
0.0060         -37.7448     -37.8690       0.1242   106.8
...
```

同时生成的还有 `ethanol.traj` 文件，是一个二进制文件，用于储存 MD 轨迹，再运行结束后，会将 `ethanol.traj` 文件，转化为连续 XYZ 文件 `ethanol_traj.xyz`，如下：

```
9
Properties=species:S:1:pos:R:3 pbc="F F F"
C       -1.10218225       0.54875848      -0.01193573
C       -0.07140309      -0.57074911       0.02332978
O        1.27128108      -0.07655755       0.06601411
H       -2.12822571       0.14284681      -0.02565174
H       -0.98439850       1.17690996      -0.90962738
H       -1.00436368       1.19629290       0.87186750
H       -0.19869335      -1.25090208      -0.83515112
H       -0.17554791      -1.17049901       0.93904376
H        1.49896341       0.25986959      -0.81524918
9
Properties=species:S:1:pos:R:3 pbc="F F F"
C       -1.09846592       0.54709868      -0.00898858
C       -0.07575809      -0.56946777       0.01911753
O        1.26732655      -0.07780340       0.06985297
H       -2.13117389       0.13437607      -0.02308674
H       -0.97311841       1.16350768      -0.93634557
H       -1.00280830       1.21402345       0.89267969
H       -0.17896847      -1.24766090      -0.83289757
H       -0.13472330      -1.14195926       0.91959215
H        1.49890315       0.25651489      -0.84056620
...
```

保存了 `["symbols", "positions"]` 这两个信息。以上是简单的 MD 模拟，如果需要更多功能，可以使用 LAMMPS 接口。