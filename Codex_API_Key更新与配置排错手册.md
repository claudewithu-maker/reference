# VS Code Codex 插件 API Key 更新与维护操作手册

本手册记录在 macOS 环境下，当 IBM / 自定义网关的 API Key 过期或更换时，从**更新配置**、**刷新进程**到**模型权限排查**的标准操作流程。

> ⚠️ **安全提示**：请勿将真实的 API Key 提交到公开或团队 Git 仓库中。下文示例命令中的 `<NEW_API_KEY>` 均代表您的实际 Key。

---

## 一、核心配置位置概览

| 目标位置 | 文件路径 | 作用说明 |
| :--- | :--- | :--- |
| **系统环境变量** | `~/.zshrc` | 终端及系统级环境变量 `IBM_API_KEY` |
| **Codex 认证文件** | `~/.codex/auth.json` | Codex 本地认证凭证 |
| **Codex 运行时配置** | `~/.codex/config.toml` | 供应商网关、默认模型、Token 覆盖项 |

---

## 二、标准更新步骤（Step-by-Step）

假设您的新 Key 为：`<NEW_API_KEY>`

### 步骤 1：更新环境变量（`~/.zshrc`）
在终端中执行以下命令，将旧 Key 替换为新 Key：
```bash
# 1. 替换 ~/.zshrc 中的 IBM_API_KEY
sed -i '' 's/export IBM_API_KEY=.*/export IBM_API_KEY="<NEW_API_KEY>"/' ~/.zshrc

# 2. 使配置在当前终端生效
source ~/.zshrc
```

### 步骤 2：更新 Codex 插件配置文件
修改 `~/.codex/auth.json` 与 `~/.codex/config.toml`：

1. **更新 `~/.codex/auth.json`**：
   确保里面对应的值为新 Key：
   ```json
   {
     "OPENAI_API_KEY": "<NEW_API_KEY>"
   }
   ```
2. **更新 `~/.codex/config.toml`**：
   找到 `[model_providers.ibm_custom]` 块，更新 `experimental_bearer_token`：
   ```toml
   [model_providers.ibm_custom]
   name = "IBM Custom"
   base_url = "https://api.servicesessentials.ibm.com/v1"
   wire_api = "responses"
   env_key = "IBM_API_KEY"
   experimental_bearer_token = "<NEW_API_KEY>"
   ```

### 步骤 3：终止旧后台进程（至关重要 ⚠️）
VS Code 的 Codex 插件会在后台常驻一个 `app-server` 守护进程。**如果只改文件不杀进程，插件会一直使用内存中缓存的旧 Key 导致 401 报错**。

在终端执行：
```bash
pkill -f "codex.*app-server"
```
*(杀掉后无需手动启动，在 VS Code 中重新发问时会自动按新配置拉起。)*

### 步骤 4：验证连通性
运行以下自检命令：
```bash
# 验证 API Key 在网关处是否正常认证（返回 200 即正常）
curl -s -o /dev/null -w "%{http_code}\n" -H "Authorization: Bearer <NEW_API_KEY>" https://api.servicesessentials.ibm.com/v1/models
```
- 返回 **`200`**：API Key 认证成功且生效。
- 返回 **`401`**：Key 输入有误或已失效。

---

## 三、常见排错手册（Troubleshooting）

### 场景 1：报错 `403 Forbidden: The requested model 'xxx' is not available`
- **原因**：API Key 已经认证成功，但当前 Key 所在账号**没有配置该模型的权限**（例如 `gpt-5.5-gus` 在某些 Key 中未分配）。
- **解决方法**：
  1. **查询当前 Key 拥有权限的模型列表**：
     ```bash
     curl -s -H "Authorization: Bearer $IBM_API_KEY" https://api.servicesessentials.ibm.com/v1/models | jq -r '.data[].id'
     ```
  2. **测试具体某个模型是否能在 Coding Agent 中调用**：
     ```bash
     # 例如测试 gemini-3.7-flash
     curl -s -X POST https://api.servicesessentials.ibm.com/v1/responses \
       -H "Authorization: Bearer $IBM_API_KEY" \
       -H "Content-Type: application/json" \
       -d '{"model": "gemini-3.7-flash", "input": "hi"}' | jq .
     ```
  3. **修改 `~/.codex/config.toml` 中的模型名称**：
     打开 `~/.codex/config.toml`，修改首行：
     ```toml
     model = "gemini-3.7-flash"
     ```
  4. 修改后再次杀死后台进程生效：
     ```bash
     pkill -f "codex.*app-server"
     ```

---

### 场景 2：终端执行 `echo $env:IBM_API_KEY` 找不到值
- **原因**：`$env:VAR` 是 **Windows PowerShell** 语法。Mac 终端默认使用的是 **zsh**，会将 `$env` 视为空变量，只打印 `:IBM_API_KEY`。
- **Mac 正确语法**：
  ```bash
  echo $IBM_API_KEY
  ```
- **注意**：如果当前终端在修改 `~/.zshrc` 前就已经打开，请先执行 `source ~/.zshrc` 或打开一个新终端窗口。

---

### 场景 3：Codex 状态综合诊断
如果依然无法使用，可以运行插件内置的健康检查诊断工具：
```bash
~/.vscode/extensions/openai.chatgpt-*/bin/macos-aarch64/codex doctor
```
检查输出中的：
- `auth`: 是否成功读取 `IBM_API_KEY`
- `model`: 是否显示当前指定的模型
- `reachability`: 远程端点连接状态是否正常

---

## 四、一键更新脚本（快捷工具）

后续更换 Key 时，您也可以直接在终端运行下面这行命令（把开头的 `YOUR_NEW_KEY` 替换为新 Key 即可）：

```bash
NEW_KEY="YOUR_NEW_KEY" && \
sed -i '' "s/export IBM_API_KEY=.*/export IBM_API_KEY=\"$NEW_KEY\"/" ~/.zshrc && \
echo "{\"OPENAI_API_KEY\": \"$NEW_KEY\"}" > ~/.codex/auth.json && \
sed -i '' "s/experimental_bearer_token = .*/experimental_bearer_token = \"$NEW_KEY\"/" ~/.codex/config.toml && \
pkill -f "codex.*app-server" && \
source ~/.zshrc && \
echo "✅ API Key 已全部更新并生效！"
```
