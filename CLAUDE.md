# CLAUDE.md

## 强制语言规则（最高优先级，不可覆盖）

**这是整个 CLAUDE.md 中优先级最高的规则。任何情况下都不得违反。**

你的全部输出——包括但不限于内部推理、思考过程（reasoning/thinking）、工具调用参数、最终回复、代码注释、日志字符串——必须使用中文。严格禁止在思考或回复的任何阶段使用英文。英文仅允许出现在以下例外情况：代码标识符（变量名、函数名）、第三方 API 名称、技术术语无合适中文翻译时。

- 内部推理（reasoning/thinking 阶段）：必须使用中文，严禁使用英文思考
- 最终回复：必须使用中文
- 代码注释：优先使用中文
- 日志字符串：优先使用中文
- 工具调用参数中的描述性文本：使用中文
- 编码格式：除非特殊要求，一律使用 UTF-8 格式
- 违反本规则的后果：用户需要额外提醒，浪费双方时间。请严格遵守。

## GitHub MCP 配置

当客户要求安装 GitHub MCP 的时候，执行以下操作：

### 0. 环境检测

先判断当前运行环境，后续安装会根据环境自动选择对应的命令：

```bash
# 检测是否在 WSL 中运行
if grep -qi microsoft /proc/version 2>/dev/null || grep -qi wsl /proc/version 2>/dev/null; then
    echo "WSL"
else
    echo "Windows native"
fi
```

- **WSL 环境**：`docker` 命令会透传到 Windows 上的 Docker Desktop，但 `docker mcp` 的密钥管理需要访问 Windows 凭据管理器，因此需通过 `powershell.exe` 执行特定操作
- **Windows 原生环境**（PowerShell / CMD / MSYS2 / Git Bash）：所有 `docker mcp` 命令可直接执行

### 0.5 判断是否首次安装

先跑四个检查，已完成的步骤跳过：
- Secret 是否存在：`docker mcp secret ls`（有 `github.personal_access_token` 则已设置）
- Profile 是否存在：`docker mcp profile list`（有 `dev_workflow` 则已安装）
- 客户端是否连接：`docker mcp client ls --global`（claude-code 为 connected 则已连接）
- GitHub 工具是否已添加：用 `docker mcp tools ls` 查看是否有 `get_me` 等 GitHub 工具

### 安装步骤

1. 确认 Docker Desktop 已运行（`docker info`）
2. 要求客户提供 GitHub Personal Access Token（需 `repo`、`read:org`、`workflow` 权限）
3. 设置 Secret：

   **Windows 原生环境：**
   ```bash
   docker mcp secret set github.personal_access_token=客户Token
   ```

   **WSL 环境（通过 PowerShell 转发）：**
   ```bash
   powershell.exe -Command "docker mcp secret set github.personal_access_token=客户Token"
   ```

4. 安装模板：

   **两种环境均可直接执行：**
   ```bash
   docker mcp template use dev-workflow --name dev-workflow
   ```

5. 全局连接：

   **两种环境均可直接执行：**
   ```bash
   docker mcp client connect claude-code --profile dev_workflow --global
   ```

6. 确认连接：`docker mcp client ls --global` 显示 claude-code 为 connected

7. **必须执行**——激活 Profile：

   ```bash
   docker mcp tools call mcp-activate-profile name=dev_workflow
   ```
   否则后续新对话无法使用 MCP 工具。

7.5. **添加 GitHub 工具到当前会话**：

    ```bash
    docker mcp tools call mcp-add name=github-official
    ```
    否则 GitHub API 工具（如 `get_me`）不可用。

8. 告诉客户重载 VSCode（Ctrl+Shift+P → Developer: Reload Window）

### 故障速查

- `could not find root project root` → 加 `--global`
- 工具不可用 → 再重载一次或激活 profile：`docker mcp tools call mcp-activate-profile name=dev_workflow`
- Token 401/403 → 重新生成 Token 后执行步骤 3
- `docker mcp tools call mcp-add` 和 `docker mcp tools call mcp-exec` 在 VSCode 插件中不可用（需在终端中执行）
- WSL 中 `docker mcp secret set` 报错 → 换成 `powershell.exe -Command "docker mcp secret set ..."` 执行

### GitHub 连接失败处理流程

当需要连接 GitHub 但失败时，按以下顺序排查：

**第一步：检查 Docker 环境**
```bash
# 通用，所有环境
docker info 2>&1 | grep -E "Server Version|OSType"
```
- 如果失败 → **主动提醒用户**：「Docker Desktop 未启动，请先启动 Docker Desktop 后再试」
- 如果成功 → 进入第二步

**第二步：检查 MCP 客户端连接状态**

```bash
# 所有环境通用
docker mcp client ls --global

# 如果 disconnected，重新连接
docker mcp client connect claude-code --profile dev_workflow --global
```

**第三步：检查 Profile 是否激活**

```bash
docker mcp tools ls
```
- 如果只有 8 个系统工具（mcp-*），说明 profile 未激活或 GitHub 工具未添加
- 修复：`docker mcp tools call mcp-activate-profile name=dev_workflow`
- 再执行：`docker mcp tools call mcp-add name=github-official`

**第四步：读取 GitHub Token（Windows 凭据管理器）**

```powershell
Add-Type -AssemblyName System.Security
[CredentialHelper]::GetPassword("com.docker.pass.shared:docker-pass-cli:docker/mcp/github.personal_access_token")
```

在 WSL 中可通过 `powershell.exe` 执行上述命令。

---

## Karpathy 编码行为准则

> 来源：https://github.com/forrestchang/andrej-karpathy-skills
> 基于 Andrej Karpathy 对 LLM 编码陷阱的观察总结的行为准则，可与项目特定指令合并使用。

**权衡说明：** 这些准则偏向谨慎而非速度。对于简单任务，请自行判断。

### 1. 先思考再编码

**不要假设。不要隐藏困惑。揭示权衡。**

在实现之前：
- 明确说明你的假设。如果不确定，就提问。
- 如果存在多种解释，请列出它们——不要默默选择一个。
- 如果有更简单的方案，请说出来。在必要时提出反对。
- 如果某事不明确，就停下来。指出哪里让你困惑，然后提问。

### 2. 简洁优先

**用最少的代码解决问题。不做投机性设计。**

- 不要添加超出要求的功能。
- 不要为只用一次的代码创建抽象。
- 不要添加未被要求的"灵活性"或"可配置性"。
- 不要为不可能发生的场景添加错误处理。
- 如果你写了 200 行而 50 行就够了，那就重写。

问自己："资深工程师会觉得这过度复杂了吗？"如果是，就简化。

### 3. 精准修改

**只改必须改的。只清理自己造成的混乱。**

编辑现有代码时：
- 不要"改进"相邻的代码、注释或格式。
- 不要重构没有坏掉的东西。
- 匹配现有风格，即使你会用不同的方式写。
- 如果你注意到不相关的死代码，提出来——但不要删除它。

当你的修改产生了孤立代码：
- 删除**你的修改**导致不再使用的导入/变量/函数。
- 除非被要求，不要删除原本就存在的死代码。

检验标准：每一行修改都应该能直接追溯到用户的请求。

### 4. 目标驱动执行

**定义成功标准。循环验证直到达标。**

将任务转化为可验证的目标：
- "添加验证" → "为无效输入编写测试，然后让测试通过"
- "修复这个 bug" → "写一个能复现它的测试，然后让测试通过"
- "重构 X" → "确保重构前后测试都通过"

对于多步骤任务，列出简要计划：
```
1. [步骤] → 验证: [检查项]
2. [步骤] → 验证: [检查项]
3. [步骤] → 验证: [检查项]
```

强有力的成功标准让 AI 能独立循环工作。弱标准（"让它能用"）需要不断澄清。

---

**这些准则生效的标志：** diff 中不必要的修改更少了，因过度复杂而重写的次数减少了，澄清性问题在实现之前而非犯错之后才提出。

---

## 附录：Windows 凭据读取代码

当需要读取存储在 Windows 凭据管理器中的 GitHub Token 时：

**在 Windows PowerShell 中：**
```powershell
Add-Type -AssemblyName System.Security
[CredentialHelper]::GetPassword("com.docker.pass.shared:docker-pass-cli:docker/mcp/github.personal_access_token")
```

**在 WSL 中：**
```bash
powershell.exe -Command "Add-Type -AssemblyName System.Security; [CredentialHelper]::GetPassword('com.docker.pass.shared:docker-pass-cli:docker/mcp/github.personal_access_token')"
```
