# agentrouter-cc-fix

本地 HTTP 转发代理，专为修复 [agentrouter]国产模型在 Claude Code 中无法正常使用的问题。

## 背景

agentrouter 国模提供了 Anthropic API 兼容接口，但实际转发时会产生多种协议破损，导致 Claude Code 无法正常工作：

- 把 OpenAI 风格的响应原样塞到 Anthropic SSE 框架里，形成"半成品"流
- `message_start` 里带着 `chatcmpl-` 开头的 id
- `content_block_start` 的 thinking 块缺少 `thinking` 字段、tool_use 块缺少 `id/name/input`
- 流末尾缺失 `content_block_stop` / `message_delta` / `message_stop`
- `content_block_delta` 出现残缺的半成品事件（缺 `text` / `partial_json`）

这些都会导致 Claude Code 解析断言失败、连接中断或输出截断。

本脚本在本地拦截这些破损响应，自动修复后喂给 Claude Code，让国模在 Claude Code 中恢复可用。

## 功能

| 能力 | 说明 |
|------|------|
| **OpenAI SSE → Anthropic SSE** | 把 OpenAI `chat.completion.chunk` 流实时转成 Anthropic Messages SSE |
| **OpenAI JSON → Anthropic JSON** | 把非流式 OpenAI `chat.completion` 响应转成 Anthropic `message` JSON |
| **破损 Anthropic SSE 修复** | 逐事件修补半成品 SSE：id 替换、usage 补齐、thinking 处理、index 重编号、收尾补全 |
| **thinking 块处理** | `drop_thinking=True` 时丢弃 thinking/redacted_thinking 块；`False` 时改写成 text 块 |
| **流式状态机** | 支持 chunk 不在事件边界喂入的跨 push 缓冲，适配真流式场景 |
| **可配置代理** | 固定 upstream 或 absolute-form URL 模式；超时、日志、转换上限等独立配置项 |

## 快速开始

**⚠️ 运行前必须先安装依赖：**

```bash
pip install -r requirements.txt
```

然后启动代理：

```bash
# 直接运行（所有参数在 CONFIG 中维护）
python http_forward.py
```

代理默认监听 `127.0.0.1:8080`，upstream 为 `https://agentrouter.org/`。

在 Claude Code 中将 API base URL 设为 `http://127.0.0.1:8080` 即可：

```bash
export ANTHROPIC_BASE_URL=http://127.0.0.1:8080
claude
```

## 配置

所有运行参数集中在 `ProxyConfig` dataclass 中（`http_forward.py` 顶部），不从命令行或环境变量读取，避免生产环境启动参数漂移。

```python
@dataclass(frozen=True)
class ProxyConfig:
    host: str = "127.0.0.1"
    port: int = 8080
    upstream: Optional[str] = "https://agentrouter.org/"

    # 日志
    enable_logging: bool = True
    log_request_headers: bool = True
    log_response_headers: bool = True
    log_request_body: bool = True
    log_response_body: bool = True
    request_body_log_limit_bytes: Optional[int] = 20   # None = 不限
    response_body_log_limit_bytes: Optional[int] = 80   # None = 不限

    # 超时
    non_stream_timeout_seconds: float = 30.0
    stream_chunk_timeout_seconds: float = 10.0

    # 转换
    enable_response_transform: bool = True
    drop_thinking: bool = True
    transform_response_limit_bytes: Optional[int] = None  # None = 不限
```

修改后直接重启生效，无需额外操作。

## 自测

脚本内置 4 组自测用例，覆盖所有修复能力：

```bash
# 在 ProxyConfig 中临时设置：
run_selftest_on_start: bool = True
```

然后运行 `python http_forward.py`，脚本会执行自测后退出。自测覆盖：

1. 破损 Anthropic SSE 修复（含收尾事件）
2. 缺收尾事件的流自动补全
3. redacted_thinking / tool_use 兜底 / 未知 block 丢弃 / 残缺 delta 丢弃
4. chunk 边界切在事件中间的跨 push 缓冲一致性

## 不支持

- HTTPS CONNECT 隧道（浏览器用代理访问 HTTPS 网站不适用）
- chunked 请求体（标准库服务端直接透传容易产生边界歧义）

## 许可

MIT
