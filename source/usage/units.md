# 单位
## 单位换算

XequiNet 中的单位换算功能是仿照 [ASE 单位模组](https://wiki.fysik.dtu.dk/ase/ase/units.html)实现的，不同的是，XequiNet 将原子单位设成 1，其他单位根据 CODATA 定义的标准常数换算获得。手动调用的方式为：

```python
from xequinet.utils import unit_conversion

energy_in_hartree = -0.5
energy_in_ev = energy_in_hartree * unit_conversion("Ha", "eV")
print("Energy in eV is", energy_in_ev)

force_in_au = 1.0
force_in_ev_per_ang = force_in_au * unit_conversion("AU", "eV/Ang")
print("Force in eV/Ang is", force_in_ev_per_ang)
```

运行结果为：

```plain_text
Energy in eV is -13.605693108071435
Force in eV/Ang is 51.422067391631764
```

## 内置单位
💡注意区分大小写

| 物理量 | 支持的单位（括号内为等价表示）|
| - | - |
| 原子单位 | `AU`(`au`) |
| 物质的量 | `mol` |
| 电荷量 | `e`, `Coulomb`(`C`) |
| 长度 | `Bohr`(`a0`), `meter`(`m`), `Angstrom`(`Ang`), `cm`, `nm` |
| 质量 | `kg`, `g` |
| 能量 | `Hartree`(`Ha`,`Eh`), `Joule`(`J`), `kJoule`(`kJ`), `eV`, `meV`, `cal`, `kcal` |
| 偶极矩 | `Debye`(`D`) |
| 时间 | `second`(`s`), `fs`, `ps` |
| 压强 | `Pascal`(`Pa`), `GPa`, `bar`, `kbar` |
| 磁矩 | `Bohr_magneton`(`muB`) |

除此以外，可以用以上单位组合成符合单位，如力的单位可以表示为 `eV/Ang`，能量单位可以表示为 `kcal/mol` ，偶极矩单位可以表示为 `e*Ang`，压强单位可以表示为 `eV/Ang^3` 等。
