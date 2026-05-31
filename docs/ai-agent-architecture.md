# MolSim AI Agent 架构评估报告

> 版本：v1.1 | 日期：2026-05-31

---

## 一、需求理解

在原有"GUI 工具软件"基础上，新增一个 **自然语言命令层**：用户用中/英文描述意图，平台内嵌 AI Agent 将其拆解为具体操作指令并执行，必要时向用户追问缺失参数，最终完成结构建模、输入生成、作业提交、结果分析等动作。

**典型交互示例（追问触发结构化弹窗）：**

```
用户：创建一个液态水 PBC 盒子
AI：检测到缺失参数，弹出参数输入面板 ↓

  ┌──────────────────────────────────────┐
  │  构建液态水超胞                        │
  │  水分子数目：[  64  ]                  │
  │  密度 (g/cm³)：[ 1.00 ]               │
  │  填充方法：[PACKMOL ▾]               │
  │  随机种子：[ 42  ]  □ 固定            │
  │              [取消]  [确认构建]         │
  └──────────────────────────────────────┘

用户：（填写后点击确认）
AI：✓ 正在构建 64-H₂O 超胞（约 12.4 × 12.4 × 12.4 Å）…
    结构已加载，3D 视图已更新。
```

**追问原则**：自然语言意图识别 → 检测缺失参数 → **弹出结构化表单**（而非继续文本追问）→ 用户填写提交 → 工具执行。科研工具的严谨性要求参数确认过程无歧义。

---

## 二、架构总览

引入 AI Agent 后，整体架构分为四层：

```
┌─────────────────────────────────────────────────────────────────┐
│  UI 层 (PySide6)                                                 │
│  ┌──────────────┐  ┌────────────────────────────────────────┐  │
│  │  Col 1       │  │  Col 2（主工作区）                       │  │
│  │  项目树 /     │  │  ┌──────────────────────────────────┐  │  │
│  │  文件导航 /   │  │  │ 上半：3D 结构视图 / 结果图表        │  │  │
│  │  作业列表     │  │  │ (PyVista / pyqtgraph)            │  │  │
│  │              │  │  ├──────────────────────────────────┤  │  │
│  │              │  │  │ 下半：AI 命令面板                  │  │  │
│  │              │  │  │ [自然语言输入框]  [发送]           │  │  │
│  │              │  │  │  AI 回复流式显示区                 │  │  │
│  │              │  │  └──────────────────────────────────┘  │  │
│  └──────────────┘  └────────────────────────────────────────┘  │
└──────────────────────────────┬──────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────┐
│  AI Agent 层                                                      │
│  ┌─────────────────┐  ┌──────────────┐  ┌────────────────────┐  │
│  │ ConversationMgr  │  │ AppStateCtx  │  │  LLMClient（可换）    │  │
│  │ （历史/会话管理） │  │ （当前项目、  │  │  Claude（默认）/      │  │
│  │                  │  │  结构、插件） │  │  DeepSeek V4        │  │
│  └─────────────────┘  └──────────────┘  └────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  AgentLoop：接收意图 → 规划工具调用 → 执行 → 追问 → 回复  │    │
│  └──────────────────────────────────────────────────────────┘    │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  ToolRegistry：动态注册所有可调用工具及其 JSON Schema       │    │
│  └──────────────────────────────────────────────────────────┘    │
└──────────────────────────────┬──────────────────────────────────┘
                               │ tool_call(name, args)
┌──────────────────────────────▼──────────────────────────────────┐
│  工具执行层（Tool Executor）                                       │
│  ┌────────────┐ ┌────────────┐ ┌──────────┐ ┌───────────────┐  │
│  │StructTools  │ │InputGenTool│ │ JobTools │ │ AnalysisTools │  │
│  │build_box    │ │gen_cp2k    │ │submit_job│ │parse_energy   │  │
│  │load_struct  │ │gen_cpmd    │ │job_status│ │plot_dos       │  │
│  │build_surface│ │preview_inp │ │cancel    │ │load_traj      │  │
│  └────────────┘ └────────────┘ └──────────┘ └───────────────┘  │
└──────────────────────────────┬──────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────┐
│  核心应用层（不变）                                                │
│  builders/  plugins/  remote/  parsers/  db/                     │
└─────────────────────────────────────────────────────────────────┘
```

**关键设计原则**：AI Agent 层只调用 Tool Executor 提供的工具，**不直接操作**核心层；工具执行结果（成功/失败/需确认）统一通过 ToolResult 对象返回给 Agent。核心应用层代码完全不感知 AI 的存在，可独立测试。

---

## 三、AI 后端选型

### 3.1 方案对比（已确认）

| 方案 | 角色 | 智能程度 | 工具调用 | 成本 | 备注 |
|------|------|---------|---------|------|------|
| **Claude API** | **默认** | ★★★★★ | 原生 tool use，质量最高 | API 费用 | Anthropic SDK，首选 |
| **DeepSeek V4** | **备选** | ★★★★☆ | OpenAI-compatible，稳定 | API 费用（较低） | 与 Claude 平替，适合成本敏感场景 |

### 3.2 后端抽象实现

DeepSeek V4 提供 OpenAI 兼容接口（`https://api.deepseek.com/v1`），因此 `DeepSeekBackend` 可复用 OpenAI SDK，仅替换 `base_url` 和模型名，实现成本极低：

```python
class LLMBackend(ABC):
    async def chat(self, messages: list, tools: list) -> AgentResponse: ...

class ClaudeBackend(LLMBackend):
    # Anthropic SDK，原生 tool use
    client = anthropic.AsyncAnthropic(api_key=...)

class DeepSeekBackend(LLMBackend):
    # OpenAI SDK + DeepSeek 兼容端点
    client = openai.AsyncOpenAI(
        api_key=...,
        base_url="https://api.deepseek.com/v1"
    )
    model = "deepseek-chat"   # DeepSeek V4 模型名
```

用户在设置中选择后端并填入对应 API Key。软件发布时默认为 Claude API。

---

## 四、工具系统设计

每个工具对应一个 Python 函数，配以 JSON Schema 描述，注册到 ToolRegistry。AI 调用时传入结构化参数，Tool Executor 执行并返回结果。

### 4.1 工具分类与示例

```python
# 示例：结构工具
@tool(name="build_water_box")
def build_water_box(
    n_molecules: int,           # 水分子数
    density: float = 1.0,       # 密度 g/cm³
    method: str = "packmol"     # 填充方法
) -> StructureResult: ...

# 示例：输入生成工具
@tool(name="generate_cp2k_input")
def generate_cp2k_input(
    calc_type: Literal["energy", "opt", "md", "aimd"],
    functional: str = "PBE",
    basis_set: str = "DZVP-MOLOPT-GTH",
    cutoff: float = 400.0,
    ...
) -> InputFileResult: ...

# 示例：作业工具
@tool(name="submit_job")
def submit_job(
    server_name: str,           # 或 "local"
    scheduler: str,
    n_cores: int,
    ...
) -> JobSubmitResult: ...
```

### 4.2 插件感知工具注册

每个 SimulationPlugin 在注册时同时向 ToolRegistry 注册自己的工具集，实现插件与 AI 的自动集成：

```python
class CP2KPlugin(AbstractSimulationPlugin):
    def register_tools(self, registry: ToolRegistry):
        registry.add(self.generate_input_tool)
        registry.add(self.parse_output_tool)
        ...
```

---

## 五、关键技术难点

### 难点 1：化学领域知识注入
**问题**：LLM 需要理解物理化学参数的合理范围（截断能 400 Ry 合理，4000 Ry 不合理），才能做有效的参数审核。  
**方案**：编写领域专属 System Prompt，包含 CP2K/CPMD 参数规范、常见陷阱、单位换算规则；结合 Pydantic 在工具层做参数硬校验作为兜底。

### 难点 2：结构数据的序列化传递
**问题**：`AtomicStructure` 对象（可能含数千原子）无法直接传给 LLM；LLM 上下文窗口有限。  
**方案**：工具不传递完整结构，而是传递**结构摘要**（原子数、分子式、晶格参数、系统类型）；需要结构时通过引用 ID（当前项目内的 structure_id）传递，Tool Executor 自行从内存/数据库取出。

### 难点 3：多步骤工作流编排
**问题**："跑几何优化再算 DOS" 需要 AI 等待第一个作业结束，再触发第二步；作业可能需要几小时。  
**方案**：
- 短流程（< 30s）：AI 在 AgentLoop 内同步等待
- 长流程（HPC 作业）：AI 注册回调，作业完成后由监控模块触发后续 Agent 动作，并在 NL 面板显示通知

### 难点 4：参数追问与结构化表单生成
**问题**：追问缺失参数不能依赖用户的自然语言回答再提取，文本解析有歧义风险；科研工具要求参数确认无歧义。  
**方案（已确认）**：AI 检测到缺失参数时，生成一份 **Qt 表单描述 JSON**（字段名、类型、默认值、单位、取值范围），UI 层将其渲染为模态对话框；用户填写并提交后，结构化数据直接进入工具调用，无需文本解析。

```python
# AI 返回 tool_call: "request_parameters"
{
  "form_title": "构建液态水超胞",
  "fields": [
    {"name": "n_molecules", "type": "int",   "default": 64,   "min": 1,  "label": "水分子数目"},
    {"name": "density",     "type": "float", "default": 1.00, "unit": "g/cm³", "label": "密度"},
    {"name": "method",      "type": "enum",  "options": ["packmol", "random"], "default": "packmol"}
  ]
}
# UI 渲染为 Qt 表单 → 用户填写 → 返回结构化 dict → 工具执行
```

### 难点 5：安全性与操作确认
**问题**：AI 可能误解用户意图，执行破坏性操作（覆盖结构、取消作业）。  
**方案**：定义"高风险工具"列表，执行前弹出 UI 确认框，用户点击确认后才真正执行；AI 无法绕过此机制。

### 难点 6：两套 tool use 协议的差异
**问题**：Claude 使用 Anthropic 原生 tool use 格式；DeepSeek V4 使用 OpenAI function calling 格式，二者 Schema 不完全相同。  
**方案**：在 `LLMBackend` 层做协议转换——`ToolRegistry` 维护统一的工具描述格式，`ClaudeBackend` 将其转换为 Anthropic 格式，`DeepSeekBackend` 将其转换为 OpenAI function calling 格式。切换后端时业务代码零改动。

```python
class ToolRegistry:
    def as_claude_tools(self) -> list[dict]: ...   # Anthropic 格式
    def as_openai_functions(self) -> list[dict]: ... # OpenAI/DeepSeek 格式
```

### 难点 7：上下文长度管理
**问题**：长对话 + 多次工具调用结果会逼近上下文窗口上限。  
**方案**：实现对话压缩策略：保留最近 N 轮对话，将历史工具调用结果摘要化；每次新会话时注入最新 AppState 而非全部历史。

---

## 六、对第一阶段任务的影响

引入 AI Agent 后，Phase 1 需新增以下设计任务：

| 新增任务 | 说明 |
|---------|------|
| T6.1 AI 工具清单设计 | 梳理所有可暴露给 AI 的工具，定义 Schema，标注风险等级 |
| T6.2 System Prompt 设计 | 化学领域知识、参数规范、追问策略的提示词工程 |
| T6.3 LLM 后端抽象接口设计 | `LLMBackend` ABC + Claude（Anthropic SDK）/ DeepSeek（OpenAI-compatible）两种实现骨架 |
| T6.4 AgentLoop 流程设计 | 状态机：idle → thinking → tool_call → waiting → responding |
| T6.5 NL 命令面板 UI 设计 | Col2 下半区：输入框 + 流式响应 + 结构化参数弹窗 + 历史回看；Col2 上半区：3D 视图/结果图表 |
| T6.6 对话测试用例设计 | 覆盖追问、多步、错误恢复、拒绝执行的典型对话场景 |

---

## 七、推荐方案总结

| 维度 | 决策 |
|------|------|
| AI 后端 | **Claude API 为默认**，**DeepSeek V4 为备选**；统一 LLMBackend 抽象 |
| 工具调用框架 | **Anthropic SDK 原生 tool use**（Claude）+ OpenAI SDK（DeepSeek），不引入 LangChain |
| 工具注册 | 插件注册时同步向 ToolRegistry 注册工具，AI 自动感知新插件 |
| 结构传递 | 工具间以 `structure_id` 传引用，AI 上下文中只传摘要 |
| 长流程处理 | 作业完成事件触发回调，Agent 异步续跑后续步骤 |
| 追问交互 | AI 生成参数 Schema → UI 渲染结构化表单 → 用户填写提交，全程无文本歧义 |
| 风险操作 | 高风险工具强制 UI 确认，不可由 AI 自动跳过 |
| 上下文管理 | 滚动压缩 + 每轮注入最新 AppState |

---

## 八、主要风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|---------|
| Claude / DeepSeek 工具调用格式差异导致 bug | 中 | 中 | ToolRegistry 统一格式转换层；两套后端各写集成测试 |
| 化学参数理解错误 | 中 | 高 | 领域 Prompt + Pydantic 硬校验；结构化弹窗强制用户确认 |
| 长流程状态同步复杂 | 中 | 高 | Phase 2 早期专项 spike 验证 |
| API 成本超预期 | 低 | 中 | 缓存工具调用结果；切换 DeepSeek（成本更低） |
| 用户隐私（结构数据上云） | 中 | 中 | 默认只传结构摘要 + structure_id，不传原始坐标 |
