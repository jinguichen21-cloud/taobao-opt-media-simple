---
name: taobao-opt-media-simple
description: 淘宝 OPT 极简媒体生成器。触发词："生成商品图"、"制作商品视频"。输出极简：只交付产物，无过程描述。
version: v1.9
last_updated: 2026-03-17
---

# 淘宝 OPT 极简媒体生成器 v1.9

## 前置检查
**MCP安装状态**: 执行前先检查淘宝 OPT 服务 MCP 是否已经安装，如果没有安装就按照附录里的说明文档引导用户安装。链接地址为：https://mcp.dingtalk.com/#/detail?detailType=marketMcpDetail&mcpId=9645

## ⛔ NEVER DO

- 禁止输出过程描述和技术细节（状态码、taskId、curl 命令等）
- 禁止视频提交后不执行后台轮询（每 60 秒自动查询直到完成）
- 禁止图片内容文字描述和图片查看（直接使用上传图片）
- 禁止串行提交图片任务（必须同时并发提交）
- 禁止 prompt 模糊时直接生成（必须先推荐风格选项）
- **禁止使用非淘宝 OPT 服务的替代方案**（唯一依赖：server_id=19cea41ab9a）
- 禁止请求用户下载权限（必须直接用 deliver_artifacts 交付）
- 禁止 MCP 调用失败后重复相同错误超过 2 次（立即切换 real_cli 降级）

## 📦 输出规范

| 模式 | 交付时机 | 话术 |
|------|---------|------|
| 图片 | 每完成一张立即交付 | ✅ 图{N}已生成 |
| 视频 | 完成后单独交付 | ⏳ 生成中 → ✅ 已生成 |
| 混合 | 图片依次交付，视频后台生成 | 分别使用上述话术 |

## 🔧 核心参数

| 参数 | 类型 | 必填 | 说明 | 默认值 |
|------|------|------|------|--------|
| media_type | text | ✅ | image/video/mixed | mixed |
| image_url | text | ✅ | 商品图片 URL（HTTP 地址） | - |
| prompt | text | ⚠️ | 风格描述（具体非空） | - |
| batch_size | number | ❌ | 生成数量 | 3 |

## 🔄 核心工作流

```yaml
Step 1: 静默上传图片 → 获取 URL
Step 2: 【全并行】同时提交 3 图任务（居中/侧面全景/局部特写）
Step 3: 【竞速监控】每 10 秒批量查询 → 每完成一张立即交付
Step 4: 【首图触发】交付第一张图时 → 立即提交视频任务
Step 5: 继续交付剩余图片 → 后台轮询视频（60 秒/次）→ 交付视频
```

## ⚙️ MCP 服务配置

```json
{
  "server_id": "19cea41ab9a",
  "name": "淘宝 OPT 服务",
  "tools": ["create_picture_from_tb", "query_picture_from_tb", "create_vedio_from_pic_tb", "query_vedio_from_query_taskid"]
}
```

## 📋 错误处理

- **MCP 未安装**: 引导用户安装，**禁止使用替代方案**
- **工具调用失败**: 静默重试≤2 次 → 切换 real_cli 降级策略
- **HTTP 504/Transport closed**: 立即重试，仍失败则等待 10 秒后再次尝试
- **状态码映射**: 0=处理中，1=已完成，2=处理中，-1=失败（**禁止向用户展示**）

## 💡 多视角 Prompt

- **居中视角**: 正面展示，突出主体，柔和光线
- **侧面全景**: 展示整体轮廓和版型
- **局部特写**: 聚焦面料质感和细节工艺

## ⚠️ 参数格式要求

- `create_picture_from_tb`: params 必须为 JSON 字符串，包含 image/batchSize/width/height/prompt
- `query_picture_from_tb`: taskId 参数类型为字符串
- `query_vedio_from_query_taskid`: taskId 参数类型为数字
- `create_vedio_from_pic_tb`: clips 为 JSON 字符串数组格式

## 🚪 HARD-GATE

**调用前验证**:
- [ ] 淘宝 OPT MCP 服务已安装（server_id=19cea41ab9a）
- [ ] image_url 有效（HTTP 地址）
- [ ] prompt 具体非空
- [ ] 用户已确认风格

---

**更新日志**:
- v1.7: 增强错误处理（HTTP 504/Transport closed 智能重试），强化参数格式验证提示
- v1.6: 强化唯一依赖（仅淘宝 OPT 服务），删除冗余说明，过程描述极简
- v1.5: 依次交付策略（每完成一张立即交付 + 首图交付时触发视频）
- v1.4: 全并行竞速策略，跳过图片查看，轮询频率 10 秒/次


---------------------------------------------------------
附录：淘宝 MCP 技能安装指南.md

# DingTalk MCP 市场服务安装指南

本指南介绍如何从钉钉 MCP 市场安装 MCP 服务到本地使用，需要你通过浏览器自动化来替用户执行大部分操作

## 安装流程概览

1. 🤖 **打开 MCP 详情页面** → Agent 用浏览器访问目标 MCP 的服务页面
2. 🤖 **点击登录按钮** → Agent 在页面上找到并点击"登录"按钮，弹出扫码登录框
3. 👤 **扫码登录** → **用户**使用钉钉 APP 扫描二维码完成认证（唯一需要用户操作的步骤）
4. 🤖 **获取服务配置** → Agent 轮询检测登录完成后，点击"获取 MCP Server 配置"按钮
5. 🤖 **提取连接信息** → Agent 从页面提取 StreamableHttp URL
6. 🤖 **本地安装配置** → Agent 使用 `real_cli mcp add` 命令添加服务
7. 🤖 **验证安装成功** → Agent 检查工具列表确认服务可用

---

## 详细步骤说明

### 步骤 1：打开 MCP 详情页面

访问钉钉 MCP 市场，找到需要安装的服务：

```
https://mcp.dingtalk.com/#/detail?mcpId=<MCP_ID>&detailType=marketMcpDetail
```

常见 MCP 服务 ID：
- **淘宝OPT服务**: `mcpId=9645`

### 步骤 2：登录钉钉账号（必需）

**重要**：必须登录钉钉账号后才能获取 MCP 服务的访问凭证。

#### 🤖 Agent 操作：点击登录按钮
1. 在 MCP 详情页面找到"登录"按钮并点击
2. 提醒用户登录

#### 👤 用户操作：扫码登录（唯一需要用户介入的步骤）
- 在 MCP 登录页面上进行登录

#### 🤖 Agent 操作：轮询检查登录状态
- 每隔 60 秒检查一次页面是否已登录，持续三轮
- 每轮检测到用户还未登录就主动提醒用户登录
- 如果检测到组织列表页面提醒用户选择组织，这是登录环节的一部分

#### 登录成功标志：
- ✅ 页面右上角显示用户头像
- ✅ 出现"获取 MCP Server 配置"按钮
- ✅ 不再显示"登录"或"登陆"按钮

### 步骤 3：获取服务配置 🤖

登录成功后，Agent 自动操作：

1. 确认页面显示"获取 MCP Server 配置"按钮
2. 点击该按钮，系统自动激活 MCP 服务
3. 页面跳转到实例详情页面：
```
https://mcp.dingtalk.com/#/detail?detailType=instanceMcpDetail&instanceId=<实例ID>
```

### 步骤 4：提取连接信息 🤖

Agent 从实例详情页面提取连接配置：

#### 方式一：StreamableHttp URL（推荐）
```
https://mcp-gw.dingtalk.com/server/<service_hash>?key=<access_key>
```

#### 方式二：JSON Config
```json
{
  "mcpServers": {
    "服务名称": {
      "type": "streamable-http",
      "url": "https://mcp-gw.dingtalk.com/server/<hash>?key=<key>"
    }
  }
}
```

**安全提示**：
- ⚠️ URL 中包含个人访问密钥（API Key）
- ⚠️ 请勿泄露给他人
- ⚠️ 仅限个人使用

### 步骤 5：本地安装配置 🤖

Agent 使用 `real_cli` 命令添加 MCP 服务器：

```bash
real_cli mcp add --json '{
  "name": "服务名称",
  "type": "streamable-http",
  "baseUrl": "https://mcp-gw.dingtalk.com/server/<hash>?key=<key>"
}'
```

**参数说明**：
- `name`: 服务名称（自定义，建议使用中文）
- `type`: 固定为 `streamable-http`
- `baseUrl`: 从页面复制的完整 URL（包含 key 参数）

### 步骤 6：验证安装成功 🤖

Agent 检查安装结果：

#### 查看已安装的 MCP 列表
```bash
real_cli mcp list --json '{}'
```

#### 检查服务工具列表
```bash
real_cli mcp tools --json '{"id":"<server_id>"}'
```

如果返回工具列表，说明安装成功并可正常使用。

---

## 常见问题

### Q1: 为什么需要先登录？
DingTalk MCP 市场的服务需要通过钉钉身份认证，系统会为每个用户生成独立的访问密钥（access_key），确保服务调用的安全性和权限控制。

### Q2: 安装后无法连接怎么办？
1. 检查网络连接是否正常
2. 确认 URL 格式正确（包含 `?key=` 参数）
3. 检查 MCP 服务状态：`real_cli mcp show --json '{"id":"<server_id>"}'`
4. 如果状态为 `disconnected`，尝试移除后重新添加

## 示例：安装淘宝OPT服务 MCP

以下是完整的安装示例（仅步骤 2 需要用户登录，其余均由 Agent 自动完成）：

### 1. 🤖 打开页面
Agent 用浏览器访问：`https://mcp.dingtalk.com/#/detail?detailType=marketMcpDetail&mcpId=9645`

### 2. 🤖 点击登录 + 👤 登录
Agent 点击页面上的"登录"按钮，然后提醒用户登录

### 3. 🤖 获取配置
Agent 检测到登录成功后，点击"获取 MCP Server 配置"按钮

### 4. 🤖 提取 URL
Agent 从页面提取类似以下的 URL：
```
https://mcp-gw.dingtalk.com/server/13f784a120d9324790732641a9495e2ae931bf7f7421b0f5b030661e49a9196d?key=a6ffa618243f5ca23a5c93e6f3a99981
```

### 5. 🤖 安装服务
```bash
real_cli mcp add --json '{
  "name": "淘宝OPT服务",
  "type": "streamable-http",
  "baseUrl": "https://mcp-gw.dingtalk.com/server/13f784a120d9324790732641a9495e2ae931bf7f7421b0f5b030661e49a9196d?key=a6ffa618243f5ca23a5c93e6f3a99981"
}'
```

### 6. 🤖 验证安装
```bash
real_cli mcp tools --json '{"id":"<返回的 server_id>"}'
```
