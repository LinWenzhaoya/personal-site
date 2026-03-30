# Claude-Style Coding Agent: A/B 实验框架

一个最小化的 Claude Code 风格 Coding Agent 实现，用于验证两个关于 Agent 架构设计的核心假设：

1. **上下文治理是必要的** —— 压缩机制让 Agent 保持专注，避免在长任务中迷失
2. **工具是过滤器，不只是功能按钮** —— 搜索结果过滤控制信息流入量，减少模型幻觉风险

## 架构概览

```
┌─────────────────────────────────────────────────┐
│                   main.py                        │
│          (实验开关 & 组件装配)                      │
│  ENABLE_COMPRESSION = True/False                 │
│  SEARCH_FILTER_MODE = "filtered"/"unfiltered"    │
└────────────────────┬────────────────────────────┘
                     │
          ┌──────────▼──────────┐
          │  ClaudeStyleAgent   │
          │    (agent.py)       │
          │  Tool Loop + Trace  │
          └──┬────┬────┬────┬──┘
             │    │    │    │
    ┌────────▼┐ ┌─▼──┐ ▼   └──────────┐
    │ToolReg. │ │Ctx │Model   NoteManager
    │(tools.py│ │Mgr │Client  (notes.py)
    │ sandbox)│ │    │(Kimi)
    └─────────┘ └────┘
```

**核心 Tool Loop**：循环调用模型 → 解析决策（TOOL/FINAL）→ 执行工具 → 追加结果 → 判断是否压缩 → 下一轮

## 目录结构

```
.
├── main.py                 # 入口，实验开关（压缩/过滤模式），组件装配
├── agent.py                # 核心 Agent：Tool Loop、Trace 记录
├── agent_types.py          # 数据结构：Message, ToolCall, TraceEntry 等
├── models.py               # Kimi/Moonshot API 客户端（OpenAI 兼容）
├── tools.py                # 工具注册表：search_code, read_file, write_patch, run_command 等
├── context_manager.py      # 上下文治理：字符数估算、历史压缩、长输出外存
├── prompts.py              # System Prompt & Agent Prompt 模板
├── notes.py                # Agent 工作笔记管理
├── reset_workspace.sh      # 一键恢复 workspace 到有 bug 的初始状态
├── *_original.py           # workspace 各文件的基准备份（供 reset 使用）
├── workspace/              # Agent 操作的目标代码库（用户管理系统）
│   ├── auth.py             # 认证模块
│   ├── services.py         # 会话管理服务层（含植入 bug）
│   ├── utils.py            # 工具函数（generate_token 返回 dict）
│   ├── db.py               # 数据库模拟层
│   ├── routes.py           # 路由处理
│   ├── config.py           # 配置常量
│   └── test_all.py         # 16 个测试用例
└── traces/                 # 所有实验的 trace JSON 文件
```

## 快速开始

### 环境要求

- Python 3.10+
- [Moonshot API Key](https://platform.moonshot.cn/)（Kimi 系列模型）

### 安装 & 运行

```bash
# 克隆项目
git clone <repo-url>
cd claude-style-agent

# 设置 API Key
export MOONSHOT_API_KEY="your-api-key-here"

# 运行 Agent
python3 main.py
# 输入任务，例如：运行 python3 test_all.py，有两个测试失败了，请阅读相关代码找到失败原因并修复bug
```

### 切换实验参数

在 `main.py` 中修改实验开关：

```python
# ===== 实验开关 =====
ENABLE_COMPRESSION = True       # 实验一：改为 False 关闭上下文压缩
SEARCH_FILTER_MODE = "filtered" # 实验二：改为 "unfiltered" 关闭搜索过滤
MAX_STEPS = 20                  # Agent 最大步数
# ====================
```

切换模型：

```python
model_client = KimiModelClient(
    model_name="kimi-k2.5",      # 或 "moonshot-v1-8k"
    temperature=1,                # kimi-k2.5 仅支持 temperature=1
)
```

### 跑 A/B 实验

```bash
# 恢复 workspace 到有 bug 的初始状态
bash reset_workspace.sh

# 跑一次实验
echo '运行 python3 test_all.py，有两个测试失败了，请阅读相关代码找到失败原因并修复bug' | python3 main.py

# trace 自动保存到 trace_output.json，可手动归档
cp trace_output.json traces/experiment_name_runN.json
```

## 实验设计

### 实验一：上下文压缩

**假设**：上下文治理是必要的——压缩机制让 Agent 保持专注。

**方法**：相同任务（修复 workspace 中的 bug），分别在 `ENABLE_COMPRESSION=True` 和 `False` 下各跑 3 次，观察成功率和步数。

**Bug 设计**：`workspace/auth.py` 中存在两个 bug，Agent 需要定位并修复。

### 实验二：搜索结果过滤

**假设**：工具是过滤器——搜索结果过滤控制信息流入量，减少幻觉风险。

**方法**：`search_code` 工具在 `filtered` 模式下只返回 top 5 结果并附摘要，在 `unfiltered` 模式下返回全部匹配（最多 100 条）。

**Bug 设计（改进版 v2）**：三层调用链 `test → auth.py → services.py → utils.py`，搜索 `token` 关键词会命中大量噪音结果（db.py 28+ 处），考验 Agent 是否能在信息噪声中保持正确推理。

## 实验结果摘要

### 实验一结果

| 条件 | moonshot-v1-8k | kimi-k2.5 |
|------|---------------|-----------|
| 压缩 ON | **2/3 成功** (6-16步) | **3/3 成功** (6-8步) |
| 压缩 OFF | **0/3 成功** (全部 20 步超限) | **2/3 成功** (1次幻觉) |

**结论**：压缩对弱模型效果显著（2/3 vs 0/3），对强模型则降低幻觉风险。

### 实验二结果（改进版 v2）

| 条件 | moonshot-v1-8k | kimi-k2.5 |
|------|---------------|-----------|
| Filtered (top 5) | 1/3 成功 | **3/3 成功** |
| Unfiltered (全部) | 1/3 成功 | 2/3 成功 (1次幻觉) |

**结论**：
- 弱模型：两组都受限于 `write_patch` 工具使用能力，过滤差异被掩盖
- 强模型：Filtered 100% vs Unfiltered 67%，unfiltered 组因上下文膨胀（峰值 38K chars）引发幻觉

## 核心组件说明

### Agent (agent.py)

Tool Loop 执行引擎，每步记录 `TraceEntry`（步数、工具名、参数、结果摘要、消息数、总字符数、是否触发压缩），运行结束后输出 `trace_output.json`。

### ContextManager (context_manager.py)

- **压缩触发**：当消息总字符数超过 `max_chars`（默认 12000）时触发
- **压缩策略**：保留 system prompt + 最近 N 条消息，中间历史调用模型压缩为一条 summary
- **长输出外存**：工具输出超过 `max_inline_chars` 时写入 artifacts 目录，上下文中只保留首尾截断

### ToolRegistry (tools.py)

沙盒化工具集：
- `search_code` — 正则搜索代码，支持 filtered/unfiltered 模式
- `read_file` — 读取文件指定行范围
- `write_patch` — 基于 snippet 匹配的最小化代码修补
- `run_command` — 白名单命令执行（pytest, python3, ls, cat）
- `save_note` — Agent 工作笔记

### Trace 格式

每步输出一个 JSON 对象：

```json
{
  "step": 1,
  "action": "tool",
  "tool_name": "run_command",
  "tool_args": {"cmd": "python3 test_all.py"},
  "tool_summary": "Command failed with exit code 1.",
  "messages_count": 4,
  "messages_total_chars": 3758,
  "compressed": false
}
```

## 局限性

- **样本量小**：每组仅 3 次，结果有随机性
- **write_patch 瓶颈**：弱模型难以构造精确的 snippet 匹配文本，成为独立于实验变量的干扰因素
- **Bug 设计约束**：workspace 是一个简化的用户管理系统，真实项目的代码复杂度远高于此
- **单一任务类型**：仅测试了 bug 修复场景，未覆盖功能开发、重构等任务

## License

MIT
