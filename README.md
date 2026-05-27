# 多智能体旅行助手

一个基于 LangGraph、`qwen3-max` 和高德 MCP 的多agent旅行助手项目。

- 使用 `MessagesState` 维护多轮对话
- 使用 `qwen3-max` 作为多个专职 agent 的底座模型
- 使用高德 MCP 工具提供天气、地理编码、逆地理编码和地点提示能力
- 在进入模型前增加参数澄清与地点消歧层
- 将单 assistant 拆分为 `weather / geo / travel / general` 四个专职 agent
- 使用 `checkpointer + thread_id` 恢复多轮上下文
- 使用 LangGraph 的 `specialist agent -> tools -> specialist agent` 循环完成工具调用

## 工作流程

一个真正的多智能体 Agent loop：

1. 用户输入旅行问题
2. `analyze_query` 节点先抽取意图、地点和时间线索
3. 如果地点不清晰，`clarify` 节点先向用户追问
4. 如果信息足够，`select_agent` 节点选择最合适的专职 agent
5. 专职 agent 调用 `qwen3-max` 进行决策
6. 模型判断需要工具时，调用该 agent 允许使用的高德 MCP 工具
7. 工具结果回到当前专职 agent
8. `checkpointer` 按 `thread_id` 持久化状态
9. 专职 agent 整理结果并输出最终回答

## 环境准备

### 1. 安装 uv

```
curl -LsSf https://astral.sh/uv/install.sh | sh
```

如果已经安装过 `uv`，可以跳过这一步。

### 2. 创建虚拟环境

```
uv venv .venv --python 3.11
source .venv/bin/activate
```

Windows PowerShell：

```
.venv\Scripts\Activate.ps1
```

### 3. 安装依赖

```
uv pip install -e ".[dev]"
```

### 4. 配置模型与高德 Key

```
export DASHSCOPE_API_KEY="你的 DashScope Key"
export QWEN_MODEL="qwen3-max"
export AMAP_API_KEY="你的高德 Key"
```

Windows PowerShell：

```
$env:DASHSCOPE_API_KEY="你的 DashScope Key"
$env:QWEN_MODEL="qwen3-max"
$env:AMAP_API_KEY="你的高德 Key"
```

说明：

- 默认模型名已经是 `qwen3-max`
- 如果不设置 `QWEN_MODEL`，代码也会默认使用 `qwen3-max`
- 高德 MCP 默认读取 `AMAP_API_KEY`

## 运行方式

### 1. 启动旅行助手

```
python -m langgraph_study.app.cli
```

启动后可以多轮提问，例如：

- `北京今天天气怎么样？`
- `上海和杭州这周末哪个更适合出行？`
- `帮我把天安门广场转成坐标`

如果只想单次执行：

```
python -m langgraph_study.app.cli --input "北京今天天气怎么样？"
```

如果你希望在不同命令之间继续同一段对话，显式传同一个 `thread_id`：

```
python -m langgraph_study.app.cli --thread-id trip-demo --input "北京今天天气怎么样？"
python -m langgraph_study.app.cli --thread-id trip-demo --input "那上海呢？"
```

说明：

- 当前 CLI 会打印正在使用的 `thread_id`
- `/reset` 会创建新的 `thread_id`
- checkpoint 默认写入 `.langgraph_data/checkpoints.sqlite`

### 2. 查看图结构

```
python -m langgraph_study.app.cli --show-mermaid --no-prompt
```

### 3. 使用 LangGraph Studio

```
langgraph dev
```

说明：
- Studio 使用 `langgraph_study.assistant.graph:build_studio_graph`
- `build_studio_graph()` 使用占位工具来展示图结构和工具调用路径，不会在导入阶段真实启动高德 MCP `stdio` 子进程
- CLI 继续使用 `build_persistent_graph()`，会真实加载高德 MCP 工具并接入 SQLite checkpoint

Studio 中可以直接提交：

```json
{
  "messages": [
    {
      "role": "user",
      "content": "北京今天天气怎么样？"
    }
  ]
}
```

### 4. 单独运行高德 MCP 服务

如果你想把高德工具单独暴露给别的客户端：

```
python -m langgraph_study.mcp.amap_server --transport streamable-http
```

当前提供的 MCP 工具：

- `geocode`
- `reverse_geocode`
- `weather`
- `input_tips`
