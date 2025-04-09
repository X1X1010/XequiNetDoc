# 制作数据集
一个完整的 XequiNet 数据集包含三个文件，数据集本体`data.lmdb`，记录数据集单位信息的文件`info.json`，以及记录数据集划分的文件`<split>.json`。前两个文件的文件名固定为`data.lmdb`和`info.json`，而`<split>.json`的名字可以自定义，因此可以同时存在复数个不同的划分。

## LMDB 数据集文件
LMDB（Lightning Memory-Mapped Database）是一种高性能的嵌入式键值存储数据库，非常适合作为机器学习数据集的格式。主要优点包括，LMDB 是使用基于内存映射的读写方式，加载数据时的效率非常高，而且支持多进程的并发读写，这一点非常适应分布式训练的需求。

但是 LMDB 好像是不太支持并发写入的（这一点不太确定），而且写入的效率比较一般，因此制作数据集的脚本通常需要运行很长时间。API 又比较抽象，不是很容易掌握（我也只是依葫芦画瓢地在用）。

XequiNet 使用的 LMDB 文件是由一个个键值对组成的，键是**从 0 开始的序号所对应的 8 位二进制字节序列**，值是**一个`XequiData`类数据的实例**。

```
index  ->                key                  ->    value
  0    -> b'\x00\x00\x00\x00\x00\x00\x00\x00' -> xequidata_0
  1    -> b'\x01\x00\x00\x00\x00\x00\x00\x00' -> xequidata_1
...
133884 ->  b'\xfc\n\x02\x00\x00\x00\x00\x00'  -> xequidata_133884
```

构建数据集的脚本可以参考以下代码：
```python
import lmdb
import pickle
from xequinet.data import XequiData

lmdb_data = lmdb.open(
    path="data.lmdb",
    map_size=2**40,  # Here is 1 TB. You can set other size suitable for your dataset
    subdir=False,
    sync=False,
    writemap=False,
    meminit=False,
    map_async=True,
    create=True,
    readonly=False,
    lock=True,
)

for i in range(<dataset_len>):
    # all the components are torch.Tensor
    # components of float points must be consistent (dtype should be the same)
    datapoint = XequiData(
        atomic_numbers=...  # [n_atoms,] int
        pos=...  # [n_atoms, 3] float/double
        pbc=...  # [1, 3]  bool (optional)
        cell=...  # [1, 3, 3]  float/double (optional)
        energy=...  # [1,]  float/double
        forces=...  # [n_atoms, 3]  float/double
        virial=...  # [1, 3, 3]  float/double
        xxx=...  # any other property is OK
    )

    with lmdb_data.begin(write=True) as txn:
        key = i.to_bytes(8, byteorder="little")
        txn.put(key, pickle.dumps(datapoint))
    ...
```

小贴士💡，在训练分子体系的能量性质时，这里推荐扣除分子中每个原子单独的能量，这样可以有效降低能量性质的量级，使训练变得简单。而且原子能量相当于一个常数，因此不会影响梯度性质的计算。

举一个简单的例子，对于水分子 H2O，我们在 B3LYP/def2-SVP 等级下计算单点能，结果为,
$$
    E(\text{H}_2\text{O}) =  -76.3574 \text{ Eh}
$$
而单独计算 H 和 O 原子的单点能结果分别为 $E(\text{H}) = -0.5012 \text{ Eh}$ 和 $E(\text{O}) = -74.9987 \text{ Eh}$，所以水分子扣除原子能量之后的能量数值为，
$$
    E^\circ (\text{H}_2\text{O}) = E(\text{H}_2\text{O}) - 2E(\text{H}) - E(\text{O}) = -0.3563 \text{ Eh}
$$

可见量级一下子就减小了下来。此处的原子能量可以使用与标签相同等级的方法计算获得，也可以对整个数据集的能量进行线性拟合获得。

对于周期性的材料体系（也就是 VASP 计算得到的结果），通常不扣除原子能量也可以很好地拟合，因为通常第一性原理软件输出的能量为相对赝势的能量，本身量级不大。如果需要扣除能量的话建议扣除相应元素最稳定单质的能量，也就是拟合生成能（具体单质结构可以在 [Material Project](https://legacy.materialsproject.org/) 上查询）。

小贴士2🙂，强烈建议使用 Pandas 通过`csv`或`parquet`等格式的文件记录数据集的信息，包括序号、化学组分、所属子集、能量值等简单且重要的信息，比如像这样，

| Index | Formula | Subset | Energy | Charge | ... |
| - | - | - | - | - | - |
| 0 | CH4 | monomer | -3.054 | 0 | |
| 1 | H2O | monomer | -3.032 | 0 | |
| 2 | H3O+ | ion | -3.033 | 1 | |
| ... | | | | |

这样的好处是，因为 LMDB 读起来还是很费劲的，所以要对数据集进行统计或筛选时，不需要每次都打开 LMDB 进行信息获取，只需要读轻量级的 Pandas Frame 就可以了。

## INFO 文件
`info.json`文件主要是用来储存数据集的相关信息的，其中大部分信息其实是写给自己和别人看的，比如计算等级、数据集来源等等。**只有单位部分**是程序真正会去读的。比如下面这个示例：

```json
{
  "units": {
    "energy": "eV",
    "pos": "Ang",
    "forces": "eV/Ang"
  },
  "method": "XYGJ-OS/cc-pVQZ",
  "atomic_energies": {
    "H": -0.5,
    "C": -37.8,
    "O": -75.1
  }
}
```

在这个 JSON 文件中，程序在加载数据时**只会读取这一部分**，

```json
{
  "units": {
    "energy": "eV",
    "pos": "Ang",
    "forces": "eV/Ang"
  }
}
```
这里的单位就是 LMDB 文件中，数据所使用的单位。由于不同的数据集使用的单位都不一样，一般量化软件喜欢用原子单位，第一性原理软件用电子伏特会多一点。程序会根据声明的单位，将数据的单位换算成模型使用的单位。具体支持的单位见[单位](../usage/unit.md)。

小贴士👀，请注意 JSON 文件的格式，推荐使用 vscode 之类的编辑器来判断格式是否正确。

## 训练、测试和验证集划分文件
一般来说数据集需要划分成训练集、验证集和测试集。使用 JSON 文件来划分数据集，如：

```json
{
  "trian": [0, 1, 2, 3, 4, 5],
  "valid": [6, 7, 8],
  "test": [9, 10, 11]
}
```

该 JSON 文件表示第 0, 1, 2, 3, 4, 5 个数据点为训练集，第 6, 7, 8 个数据点为验证集，第 9, 10, 11 个数据点为测试集。其中训练过程中仅仅会用到训练集和验证集，而测试过程中仅会读取测试集。由于同一数据集可能会用到不同的划分，因此可以同时存在复数个`<split>.json`文件，只需要在 CONFIG 文件中声明所使用的 JSON 文件的名字（不包含`.json`后缀）即可。

## 数据集存放方式
将上述三个文件存放到同一个目录中，这里展示一个例子：

```shell
spice-v2/
├── data.lmdb
├── info.json
└── random42.json
```

数据集存放的路径为`dir_to_data_path/spice-v2/`（这个路径也就是之后写在 CONFIG 文件中的数据集路径），其中`data.lmdb`是数据集文件，`info.json`是记录单位信息的文件，这两个文件名字是固定的；`random42.json`是数据集划分文件，名字是 random42（这个名字是写在 CONFIG 文件中的 split 文件名）。

这里要注意的是，虽然 LMDB 支持并发读，但是并发的太多也是不行的，会很卡，所以如果有多个训练任务需要读取同一个数据集的话，可以复制几份到临时目录上进行读取（比如节点专属的固态硬盘上），避免进程之间互相打架。
