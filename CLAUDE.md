# MolSim — 分子模拟工作流平台

## 项目概述

MolSim 是一个跨平台桌面端科学计算辅助工具，面向量子化学/第一性原理/分子动力学计算研究人员，覆盖从建模、输入文件生成、作业提交、运行监控到结果处理的完整工作流。

**目标平台**：Windows 10/11、macOS 12+  
**目标计算软件（当前）**：CP2K（优先）、CPMD  
**扩展目标（规划）**：LAMMPS、Quantum ESPRESSO、NAMD 等  
**目标用户**：从事周期体系、分子簇、界面体系模拟的研究人员

---

## 核心功能模块

| 模块 | 说明 |
|------|------|
| **结构建模** | 构建/导入周期体系、分子簇、界面超胞模型，3D GPU 加速实时预览 |
| **输入生成** | 向导式生成各插件（CP2K/CPMD/…）的输入文件，基于 Jinja2 模板 |
| **作业提交** | 本机直接运行或通过 SSH 提交到远程 HPC 集群（PBS/SLURM/SGE） |
| **作业监控** | 实时轮询作业状态，错误关键词检测，桌面通知 |
| **结果下载** | SFTP 拉取远程结果到本地项目目录，支持断点续传 |
| **结果处理** | 解析能量、结构、轨迹、电荷、光谱；3D 等值面/轨迹动画可视化 |

---

## 技术选型（已确认）

### 编程语言
- **Python 3.11+**（主语言）

### GUI 框架
- **PySide6（Qt6）**：LGPL 协议，跨平台（Windows Direct3D / macOS Metal / Linux OpenGL），控件体系成熟，信号槽机制天然支持模块解耦

### 3D 可视化
- **PyVista（VTK 封装）**：GPU 加速渲染（OpenGL/Metal/Direct3D），嵌入 `QVTKRenderWindowInteractor` Qt 控件
  - 原子结构：球棍 / 多面体 / 超胞渲染，键长键角测量
  - 等值面渲染：读取 cube 文件，渲染电荷密度/ELF/LDOS 等值面
  - 轨迹动画：帧播放器，支持导出 MP4/GIF

### 能带 / DOS 图表
- **pyqtgraph**：OpenGL 加速 2D 图表，嵌入 Qt 窗口，适合实时数据刷新

### 原子结构处理
| 库 | 职责 |
|----|------|
| **ASE** | 结构读写（XYZ/CIF/POSCAR/PDB/mol2）、超胞构建、表面切割、界面生成 |
| **pymatgen** | 周期体系对称性分析、空间群、Materials Project 接口 |

### 插件体系
- 每个仿真软件封装为 `SimulationPlugin` 子类，由 **插件注册表（PluginRegistry）** 统一管理
- 新软件支持 = 新增一个 `plugins/<name>/` 目录并注册，无需改动核心代码
- **当前插件**：`cp2k`（Phase 2 优先开发）、`cpmd`（紧随其后）
- **规划插件**：`lammps`、`qe`（Quantum ESPRESSO）、`namd`

### 输入文件生成
- **Jinja2**：每个插件维护自己的模板集 `templates/<plugin>/`
- 参数验证：Pydantic v2 数据模型（必填字段、取值范围、互斥选项校验）

### 远程连接 / 作业管理
- **Paramiko**：SSH/SFTP 底层连接
- 调度器适配器：PBS、SLURM、SGE、Local（本机直接运行）

### 数据持久化
- **SQLite**（via SQLAlchemy 2.0）：项目元数据、作业记录、服务器配置
- **TOML**：用户配置文件（`config.toml`）
- **HDF5**（h5py）：大型轨迹/结果数据集

### 打包发布
- **PyInstaller**：Windows `.exe` / macOS `.app` 单文件包

---

## 项目目录结构

```
molsim/
├── molsim/                        # 主包
│   ├── core/                      # 核心数据模型（平台无关）
│   │   ├── structure.py           # AtomicStructure：原子列表、晶格、PBC
│   │   ├── system.py              # SystemType 枚举：PERIODIC/CLUSTER/INTERFACE
│   │   └── project.py             # Project / CalculationJob 数据模型
│   │
│   ├── builders/                  # 结构建模
│   │   ├── periodic.py            # 超胞构建、K 点建议
│   │   ├── cluster.py             # 分子簇截取
│   │   └── interface.py           # 界面超胞、晶格匹配
│   │
│   ├── plugins/                   # 仿真软件插件体系
│   │   ├── base.py                # AbstractSimulationPlugin ABC
│   │   ├── registry.py            # PluginRegistry：注册、发现、查询
│   │   ├── cp2k/                  # CP2K 插件（Phase 2 优先）
│   │   │   ├── __init__.py        # 注册入口
│   │   │   ├── generator.py       # 输入文件生成（Jinja2）
│   │   │   ├── scheduler.py       # 作业脚本模板
│   │   │   └── parser.py          # 输出解析（能量/轨迹/电荷/DOS）
│   │   └── cpmd/                  # CPMD 插件（Phase 2 次优先）
│   │       ├── __init__.py
│   │       ├── generator.py
│   │       ├── scheduler.py
│   │       └── parser.py
│   │
│   ├── remote/                    # 远程连接与作业管理
│   │   ├── ssh.py                 # SSH/SFTP（Paramiko 封装）
│   │   ├── transfer.py            # 文件传输、断点续传
│   │   └── schedulers/            # 调度器适配器
│   │       ├── base.py            # AbstractScheduler
│   │       ├── pbs.py
│   │       ├── slurm.py
│   │       ├── sge.py
│   │       └── local.py
│   │
│   ├── ui/                        # GUI 层（PySide6）
│   │   ├── main_window.py         # 主窗口：左侧项目树 + 中央工作区 + 右侧属性面板
│   │   ├── structure_view/        # 3D 结构视图（PyVista + QVTKWidget）
│   │   │   ├── viewer.py          # 球棍/多面体/等值面渲染
│   │   │   └── trajectory.py     # 轨迹播放器
│   │   ├── input_wizard/          # 输入文件向导（插件感知）
│   │   ├── job_panel/             # 作业提交 + 监控面板
│   │   └── results_view/          # 结果查看：能量曲线/DOS/轨迹
│   │
│   └── db/                        # 数据库层
│       ├── models.py              # SQLAlchemy 模型
│       └── migrations/            # Alembic 迁移脚本
│
├── templates/                     # Jinja2 模板（随插件组织）
│   ├── cp2k/
│   └── cpmd/
│
├── tests/
│   ├── test_systems/              # 测试结构文件
│   ├── reference_inputs/          # 参考输入文件
│   └── reference_outputs/         # 参考输出文件
│
├── docs/
│   ├── requirements.md
│   ├── tech-selection.md
│   ├── data-model.md
│   └── ui-wireframes/
│
├── pyproject.toml
├── environment.yml
├── CLAUDE.md
└── README.md
```

---

## 插件接口规范（AbstractSimulationPlugin）

每个插件必须实现以下接口，新软件支持只需继承并实现此 ABC：

```python
class AbstractSimulationPlugin(ABC):
    name: str                        # 插件唯一标识，如 "cp2k"
    display_name: str                # UI 显示名称
    version: str
    supported_calc_types: list[str]  # 如 ["energy", "opt", "md", "aimd"]

    @abstractmethod
    def generate_input(self, structure, params) -> dict[str, str]: ...
    # 返回 {filename: content} 字典

    @abstractmethod
    def generate_job_script(self, job_config, scheduler_type) -> str: ...

    @abstractmethod
    def parse_output(self, output_dir) -> ParsedResult: ...

    @abstractmethod
    def validate_params(self, params) -> list[ValidationError]: ...
```

---

## 开发约定

- Python 代码遵循 PEP 8，使用 `ruff` 做 lint
- 类型注解覆盖公共 API（`mypy --strict`）
- 数据模型验证使用 Pydantic v2
- 单元测试用 `pytest`，测试覆盖率目标 ≥ 80%
- 提交信息格式：`<type>(<scope>): <summary>`（feat/fix/docs/refactor/test）
- 分支策略：`main`（稳定）、`develop`（集成）、`feature/xxx`（功能）

---

## 阶段规划

| 阶段 | 内容 | 状态 |
|------|------|------|
| Phase 1 | 需求设计、技术选型确认、数据结构设计、界面原型、测试体系准备 | **进行中** |
| Phase 2 | 编程调试：核心模块 → CP2K 插件 → CPMD 插件 → UI 集成 | 待启动 |
| Phase 3 | 打包发布（Win/macOS）、文档完善、持续迭代 | 待启动 |

---

## 测试体系（规划）

| 体系 | 类型 | 测试重点 |
|------|------|---------|
| H₂O 单分子 | 分子簇 | 最小功能验证，CP2K/CPMD 单点能 |
| 液态水超胞（64 H₂O） | 周期体系 | AIMD 完整流程，轨迹解析 |
| MgO(001) 表面 | 界面/表面 | 表面建模、结构弛豫 |
| Pt(111)/水界面 | 界面体系 | 界面超胞建模、晶格匹配 |
| 乙醇分子 | 分子簇 | 多元素 CPMD 测试 |
