# cc-connect Windows 部署手册

> 本手册让 Windows 上的 AI agent（Claude Code / Codex）直接执行每一步。
> 修改版源码：https://github.com/DengHaijin/cc-connect

## 前置检查

执行以下命令确认环境，缺失的组件按需安装：

```powershell
node --version        # >= v22
npm --version
go version            # >= 1.22
codex --version       # Codex CLI
```

安装缺失的组件：

```powershell
winget install OpenJS.NodeJS          # Node.js
winget install GoLang.Go              # Go
npm install -g @openai/codex          # Codex CLI
codex login --with-api-key            # 或 codex login（OAuth）
```

---

## Step 1：从源码构建

```powershell
git clone https://github.com/DengHaijin/cc-connect.git
cd cc-connect
go build -tags no_web -o cc-connect.exe .\cmd\cc-connect\
```

## Step 2：安装 npm 包并替换二进制

```powershell
npm install -g cc-connect
Copy-Item -Force cc-connect.exe "$env:APPDATA\npm\node_modules\cc-connect\bin\cc-connect.exe"
```

## Step 3：禁用自动更新

```powershell
notepad "$env:APPDATA\npm\node_modules\cc-connect\run.js"
```

找到 `if (needsReinstall())` 那段（约第 60-71 行），整段替换为：

```js
// skip auto-update (using custom build)
```

---

## Step 4：配置文件

```powershell
mkdir "$env:USERPROFILE\.cc-connect"
```

创建 `$env:USERPROFILE\.cc-connect\config.toml`：

```toml
language = "zh"

[log]
level = "info"

[stream_preview]
enabled = true
interval_ms = 1000
max_chars = 3000

[display]
mode = "full"
thinking_messages = true
tool_messages = true

[[projects]]
name = "default"
admin_from = "ou_你的open_id"

[projects.agent]
type = "codex"
mode = "multi-workspace"

[projects.agent.options]
base_dir = "C:\\Users\\你的用户名\\workspaces"

[projects.agent.options.env]
CODEX_FEATURES = "hooks"

[[projects.platforms]]
type = "feishu"

[projects.platforms.options]
app_id = "cli_你的app_id"
app_secret = "你的app_secret"
allow_from = "ou_你的open_id"
thread_isolation = true
progress_style = "compact"
```

## Step 5：飞书 Bot 配置

在 Windows 终端执行：

```powershell
cc-connect feishu setup --project default
```

用手机飞书扫描终端显示的二维码，完成 bot 创建与授权。完成后 `config.toml` 中的 `app_id` 和 `app_secret` 会自动填入。

如果已有 bot（复用之前创建的），直接填入 `app_id` 和 `app_secret` 到 config.toml，跳过此步。

## Step 6：开机自启

### 方案 1：启动文件夹（最简单）

```powershell
$startup = [Environment]::GetFolderPath("Startup")
$batPath = Join-Path $startup "cc-connect.bat"
@"
@echo off
start /B "" "%APPDATA%\npm\node_modules\cc-connect\bin\cc-connect.exe" --config "%USERPROFILE%\.cc-connect\config.toml"
"@ | Out-File -FilePath $batPath -Encoding ASCII
```

### 方案 2：Task Scheduler（更可靠）

```powershell
$action = New-ScheduledTaskAction -Execute "$env:APPDATA\npm\node_modules\cc-connect\bin\cc-connect.exe" -Argument "--config $env:USERPROFILE\.cc-connect\config.toml"
$trigger = New-ScheduledTaskTrigger -AtLogon
$principal = New-ScheduledTaskPrincipal -UserId $env:USERNAME -RunLevel Limited
Register-ScheduledTask -TaskName "cc-connect" -Action $action -Trigger $trigger -Principal $principal
```

## Step 7：启动并验证

```powershell
# 启动（如果还没启动）
start /B "" "%APPDATA%\npm\node_modules\cc-connect\bin\cc-connect.exe" --config "%USERPROFILE%\.cc-connect\config.toml"

# 在飞书里给 bot 发消息测试
# 查看进程
tasklist | findstr cc-connect
```

---

## 日常使用

| 飞书命令 | 作用 |
|---------|------|
| `/help` | 查看所有命令 |
| `/dir C:\path\to\project` | 切换工作目录 |
| `/list` | 查看会话列表 |
| `/new 名称` | 创建新会话 |
| `/switch <id>` | 切换会话 |
| `/current` | 查看当前会话 |
| `/mode` | 切换权限模式 |

## 终端联动

从飞书的 `/list` 或 `/current` 获取 `agent_session_id`（UUID 格式），然后在终端：

```powershell
codex exec resume <agent_session_id> "你的指令"
```

同一 session 在飞书和终端轮换使用。

## 故障排查

| 问题 | 解决 |
|------|------|
| 飞书收不到消息 | 确认飞书后台已启用 `im.message.receive_v1` 事件（长连接模式） |
| 卡片按钮无反应 | 确认 `card.action.trigger` 事件已订阅 |
| cc-connect 启动失败 | 检查 `%USERPROFILE%\.cc-connect\logs\` 日志 |
| Codex session 报错 | 确认 `codex login` 已完成 |
| 二进制自动更新 | 确认 `run.js` 的 `needsReinstall()` 已注释 |
