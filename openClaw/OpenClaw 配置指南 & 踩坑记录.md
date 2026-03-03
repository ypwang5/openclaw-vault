

> 环境：Mac mini + Clash Verge + OpenClaw 2026.2.9

---

## 一、代理配置

### 问题

OpenClaw 网关进程连不上 Telegram/Feishu，报 `fetch failed`。  
OpenAI OAuth 报 `403 unsupported_country_region_territory`。

### 原因

OpenClaw 的后台进程不走系统代理，需要显式配置。

### 解决方案：修改 LaunchAgent plist

```bash
# 查看代理端口配置
grep proxy ~/Library/LaunchAgents/ai.openclaw.gateway.plist

# 替换端口（比如从 7890 换成 7897）
sed -i '' 's|127.0.0.1:7890|127.0.0.1:7897|g' ~/Library/LaunchAgents/ai.openclaw.gateway.plist

# 重新加载
launchctl unload ~/Library/LaunchAgents/ai.openclaw.gateway.plist
launchctl load ~/Library/LaunchAgents/ai.openclaw.gateway.plist
```

> ⚠️ 以后换代理软件或端口变了，记得来这里改。

### 临时方案（当前终端生效）

```bash
export https_proxy=http://127.0.0.1:7897
export http_proxy=http://127.0.0.1:7897
```

---

## 二、OpenAI OAuth 接入

### 问题

`code->token failed: 403` — OAuth 回调时暴露了真实 IP。

### 解决

在终端设置代理后再运行 OpenClaw 的 OpenAI 接入流程：

```bash
export https_proxy=http://127.0.0.1:7897
export http_proxy=http://127.0.0.1:7897
# 然后执行 OpenAI 绑定流程
```

---

## 三、模型配置

### 关键文件路径

|文件|作用|
|---|---|
|`~/.openclaw/openclaw.json`|全局配置（渠道、网关、插件等）|
|`~/.openclaw/agents/main/agent/auth-profiles.json`|OAuth/Token 授权信息|
|`~/.openclaw/agents/main/agent/models.json`|Agent 层模型配置|

### 设置默认模型

```bash
openclaw models set anthropic/claude-sonnet-4-5
```

### 配置 Fallback 自动切换

```bash
# 添加 fallback（按优先级依次切换）
openclaw models fallbacks add openai-codex/gpt-5.3-codex
openclaw models fallbacks add anthropic/claude-haiku-4-5-20251001
openclaw models fallbacks add custom/glm-4.7

# 查看 fallback 列表
openclaw models fallbacks list

# 删除某个 fallback
openclaw models fallbacks remove openai-codex/gpt-5.3-codex
```

### 当前模型配置（正常状态）

```
anthropic/claude-sonnet-4-5      → 主力默认 ✅
openai-codex/gpt-5.3-codex       → fallback #1 ✅
anthropic/claude-haiku-4-5-20251001 → fallback #2 ✅
custom/glm-4.7                   → fallback #3（备用）
```

> 某个模型到达用量上限后，自动切换到下一个，无需手动操作。

### 查看模型列表

```bash
openclaw models list
```

### 手动切换默认模型

```bash
# 切换到 Claude
openclaw models set anthropic/claude-sonnet-4-5

# 切换到 OpenAI
openclaw models set openai-codex/gpt-5.3-codex
```

---

## 四、网关管理

```bash
# 启动网关
openclaw gateway start

# 停止网关
openclaw gateway stop

# 重新加载（首次或 plist 修改后）
launchctl bootstrap gui/$UID ~/Library/LaunchAgents/ai.openclaw.gateway.plist

# 检查健康状态
openclaw doctor
```

---

## 五、渠道状态检查

```bash
openclaw doctor
```

正常状态应显示：

```
Telegram: ok
Feishu: ok (或 configured)
```

---

## 六、常用命令速查

```bash
openclaw onboard                         # 初始化配置
openclaw doctor                          # 健康检查
openclaw dashboard                       # 打开控制面板
openclaw models list                     # 查看所有模型
openclaw models set <model>              # 切换默认模型
openclaw models fallbacks list           # 查看 fallback 列表
openclaw models fallbacks add <model>    # 添加 fallback
openclaw models auth --help              # 模型授权管理
openclaw gateway start/stop              # 网关启停
openclaw logs                            # 查看网关日志
openclaw status                          # 渠道状态
openclaw update                          # 更新版本
```

---

## 七、注意事项

1. **换代理软件/端口后**：必须更新 `ai.openclaw.gateway.plist` 里的端口
2. **`openclaw.json` 不支持 `proxy` 字段**：代理只能通过 LaunchAgent 环境变量配置
3. **模型 ID 格式**：`提供商/模型ID`，如 `anthropic/claude-sonnet-4-5`
4. **Go 套餐限制**：`gpt-5.3-codex` 官方说明仅限 Pro，实测 Go 套餐可用，但不保证长期
5. **飞书默认锁定**：陌生人消息会被拦截，需要配对：
    
    ```bash
    openclaw pairing list feishuopenclaw pairing approve feishu <code>
    ```