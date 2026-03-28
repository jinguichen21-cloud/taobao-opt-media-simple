# 错误处理详细策略

## MCP 调用失败处理

### 场景 1: HTTP 504 Gateway Timeout
```bash
# 立即重试（最多 2 次）
mcp call --json '{...}'

# 仍失败则等待 10 秒后再次尝试
sleep 10
mcp call --json '{...}'
```

### 场景 2: Transport closed
```bash
# 静默重试 1 次
mcp call --json '{...}'

# 仍失败则切换 real_cli 降级策略
real_cli mcp call --json '{...}'
```

### 场景 3: params 不能为空
```json
// 错误示例：直接传对象
{"image": "...", "batchSize": 1}

// 正确示例：JSON 字符串
"{\"image\":\"...\",\"batchSize\":1,\"width\":800,\"height\":800,\"prompt\":\"...\"}"
```

## 状态码映射表

| 状态码 | 含义 | 用户可见行为 |
|--------|------|-------------|
| 0 | 处理中 | 继续轮询 |
| 1 | 已完成 | 交付成品 |
| 2 | 处理中 | 继续轮询 |
| -1 | 失败 | 重试或报错 |

**重要**: 禁止向用户展示原始状态码，仅使用"处理中"/"已完成"等自然语言描述。
