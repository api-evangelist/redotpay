---
name: redotpay-payment
description: >-
  已安装的 redotpay-payment MCP：通用 MPP 402 代付。默认先 pay；仅当需重新授权时再
  start_authorization 与扫码。适用于 Cursor、Claude Code、Claude Desktop、Windsurf、
  Codex、Cline 等支持 MCP 的 Agent。勿要求用户配置环境变量或从源码构建。
disable-model-invocation: true
---

# redotpay-payment（已安装 · 通用代付）

本 skill 供 **带 MCP 能力的 Agent** 使用（Cursor、Claude Code、Claude Desktop、Windsurf、Codex CLI、Cline 等）。由 **Agent 代为调用** MCP 工具或 CLI；**不要**把安装、配置、支付步骤推给用户手敲，除非用户明确拒绝 Agent 代跑。

## 前提

- 用户应已通过官方 **`install.sh`** 完成安装：PATH 上有 **`redotpay-payment`**，且当前宿主已配置 MCP server **`redotpay-payment`**。
- 二进制 **stdout** 仅用于 MCP JSON-RPC；日志在 stderr。
- 网络与 OAuth 身份已在**安装时**固化；Agent **不得**要求用户设置或导出安装器写入的任何环境变量，**不得**在对话中向用户写出这些变量名。
- MCP 未就绪时：引导用户重新运行官方安装命令，或在**当前 Agent 宿主**的设置里确认 MCP 已启用；**禁止** Agent 手改 MCP 配置或向用户解释底层环境项。

## 主流程（MCP，默认）

1. **先 `pay`**  
   同一设备上可能仍有有效 `mpp_charge` 凭据（钥匙串/密钥链）。**默认直接调 `pay`**，不要每笔都先扫码。

2. **仅当需要重新设备授权时**（例如 `pay` 返回/提示 `authorization_required`，或确定无凭据）：
   - **`start_authorization`** → 必须把返回的 MCP **`type: "image"`**（PNG）展示给用户可扫  
   - **紧接着** **`wait_for_authorization`**（使用返回的 `polling_token`）  
   - **再 `pay`**  
   `start` 与 `wait` 须在**同一轮 Agent 工具链**内连续调用；**不要**等用户在聊天里发「好了」再 `wait`。

3. **可选**：`list_authorizations` 向用户说明当前仍有哪些代付授权（非每单必调）。

4. 若对话里只有文字、没有二维码图：检查 MCP 是否已连接、宿主是否支持 MCP 图片块；**不要**用 Markdown `data:` 链接冒充 MCP 图块。

## 工具一览

| 工具 | 用途 |
|------|------|
| `pay` | 访问 MPP 资源，处理 402 与代付 |
| `start_authorization` | 设备授权；返回 QR + `polling_token` |
| `wait_for_authorization` | 轮询至用户在 RedotPay App 确认或超时 |
| `list_authorizations` / `revoke` | 查看、撤销授权 |
| `recent_payments` / `device_logout` | 本地支付记录、本机登出 |
| `show_image` / `inline_png` | 仅展示图片（非支付逻辑） |
| `check_for_updates` / `self_update` | 版本检查与升级 |

`pay` 走 POST 时须传 `method` 与 `body`（与工具 schema 一致）。

## CLI 附录（仅当 MCP 不可用时）

由 **Agent** 在终端代为执行（仍勿让用户手敲）；**不要**加任何环境变量前缀：

```bash
redotpay-payment auth start --scope mpp_charge
redotpay-payment auth wait --device-code '<device_code>'
redotpay-payment pay --resource '<资源 URL>' --method GET
```

扫码展示**优先** MCP `start_authorization`。CLI `auth start` 无法在对话中自动出图，除非再配合 MCP `inline_png` / `show_image` 展示已保存的 PNG。

## 硬规则

- **默认先 `pay`**；补授权时 **`start` → 展示图 → `wait` 同轮连续**，`wait` 成功后再 `pay`。
- 不向用户粘贴 access token、refresh token、完整 `Authorization: Payment` 或支付凭据正文。
- **对用户**：不输出环境变量名、完整上游 HTTP JSON、可复盘的工具返回全文；只说明**当前状态**与**下一步**（例如：请扫上方二维码完成授权；支付未完成请稍后重试）。
- 安装或升级后，提示用户在**当前 Agent 宿主**中**重新加载 MCP**（重启 IDE 或刷新 MCP 连接）。
