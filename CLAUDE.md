# MolSim — 分子模拟工作流平台

## 项目概述

MolSim 是一个桌面端科学计算辅助工具，面向量子化学/第一性原理计算研究人员，覆盖从建模、输入文件生成、作业提交、运行监控到结果处理的完整工作流。

**目标计算代码**：CP2K、CPMD  
**目标用户**：从事周期体系、分子簇、界面体系模拟的研究人员

---

## 核心功能模块

| 模块 | 说明 |
|------|------|
| **结构建模** | 构建/导入周期体系、分子簇、界面超胞模型 |
| **输入生成** | 向导式生成 CP2K / CPMD 输入文件 |
| **作业提交** | 本机直接运行或通过 SSH 提交到远程 HPC 集群 |
| **作业监控** | 实时轮询作业状态（PBS/SLURM/SGE/本机） |
| **结果下载** | SFTP 拉取远程结果到本地项目目录 |
| **结果处理** | 解析能量、结构、轨迹、电荷、光谱等输出 |

---

## 技术选型（待第一阶段确认）

### 编程语言
- **Python 3.11+**（主语言）：NumPy/SciPy 生态、丰富的计算化学库、SSH 支持完善

### GUI 框架候选
| 方案 | 优势 | 劣势 |
|------|------|------|
| PyQt6 / PySide6 | 成熟、控件丰富、跨平台 | 体积较大、License 注意事项 |
| Dear PyGui | 现代 immediate-mode、高性能渲染 | 社区相对小 |
| Web 前端 (FastAPI + Vue3) | 跨平台、可远程访问 | 架构复杂、离线部署麻烦 |

### 原子结构处理
- **ASE** (Atomic Simulation Environment)：结构读写、超胞构建、界面生成
- **pymatgen**：周期体系、对称性分析、Materials Project 接口

### 输入文件生成
- **Jinja2** 模板引擎：CP2K/CPMD 输入模板渲染
- ASE 原生 CP2K 计算器（作为参考）

### 远程连接 / 作业管理
- **Paramiko**：SSH/SFTP 连接
- **Fabric**：高级 SSH 命令封装
- 调度器适配器：PBS (`qstat`/`qsub`)、SLURM (`squeue`/`sbatch`)、SGE (`qstat`/`qsub`)

### 数据持久化
- **SQLite** (via SQLAlchemy)：项目元数据、作业记录、服务器配置
- **JSON/YAML**：用户配置、模板参数
- **HDF5** (h5py)：大型轨迹/结果数据集

### 结果解析
- 自定义 CP2K/CPMD 输出解析器
- **cclib**：通用量化输出解析（补充）
- **MDAnalysis**：轨迹分析

---

## 项目目录结构（规划）

```
molsim/
├── molsim/                  # 主包
│   ├── core/                # 核心数据模型
│   │   ├── structure.py     # 原子结构对象
│   │   ├── system.py        # 体系类型（周期/簇/界面）
│   │   └── project.py       # 项目管理
│   ├── builders/            # 结构建模
│   │   ├── periodic.py
│   │   ├── cluster.py
│   │   └── interface.py
│   ├── generators/          # 输入文件生成
│   │   ├── cp2k/
│   │   └── cpmd/
│   ├── remote/              # 远程连接与作业管理
│   │   ├── ssh.py
│   │   ├── schedulers/      # PBS/SLURM/SGE/Local
│   │   └── transfer.py      # SFTP
│   ├── parsers/             # 结果解析
│   │   ├── cp2k/
│   │   └── cpmd/
│   ├── ui/                  # GUI 层
│   │   ├── main_window.py
│   │   ├── structure_view/
│   │   ├── input_wizard/
│   │   ├── job_panel/
│   │   └── results_view/
│   └── db/                  # 数据库层
│       ├── models.py
│       └── migrations/
├── templates/               # CP2K/CPMD Jinja2 模板
│   ├── cp2k/
│   └── cpmd/
├── tests/                   # 测试套件
│   ├── test_systems/        # 测试结构文件
│   └── reference_outputs/   # 参考输出文件
├── docs/                    # 文档
├── pyproject.toml
└── README.md
```

---

## 开发约定

- Python 代码遵循 PEP 8，使用 `ruff` 做 lint
- 类型注解覆盖公共 API（`mypy --strict`）
- 单元测试用 `pytest`，测试覆盖率目标 ≥ 80%
- 提交信息格式：`<type>(<scope>): <summary>`（feat/fix/docs/refactor/test）
- 分支策略：`main`（稳定）、`develop`（集成）、`feature/xxx`（功能）

---

## 阶段规划

| 阶段 | 内容 | 状态 |
|------|------|------|
| Phase 1 | 需求设计、技术选型、数据结构设计、界面设计、测试体系准备 | **进行中** |
| Phase 2 | 编程调试（模块实现、集成测试） | 待启动 |
| Phase 3 | 上线维护（打包发布、文档完善、持续迭代） | 待启动 |

---

## 测试体系（规划）

| 体系 | 类型 | 用途 |
|------|------|------|
| H₂O 单分子 | 分子簇 | 最小功能验证 |
| 液态水超胞 (64 H₂O) | 周期体系 | 周期计算完整流程 |
| MgO(001) 表面 | 界面/表面 | 表面建模与弛豫 |
| Pt(111)/水 界面 | 界面体系 | 界面超胞建模 |
| 有机小分子（乙醇） | 分子簇 | 多元素 CPMD 测试 |
