# 在 Claude Code 里接入飞书 / Lark（MCP）

> 目标：让 Claude Code 能直接读写飞书文档、表格、消息、日历、多维表格等。
> 原理：飞书官方提供了开源的 MCP server `@larksuiteoapi/lark-mcp`，把它注册给 Claude Code 即可。
>
> 直接把这个文件丢给 Claude Code，让它带你一步步做。

---

## 0. 前置条件

- 已安装 Node.js 18+（`node -v` 检查），`npx` 可用
- 已安装并能运行 Claude Code（`claude --version`）
- 一个飞书账号，且有权限在「飞书开放平台」创建企业自建应用

---

## 1. 创建飞书自建应用，拿到 App ID / App Secret

1. 打开开放平台：
   - 中国版飞书：<https://open.feishu.cn/>
   - 国际版 Lark：<https://open.larksuite.com/>
2. 「开发者后台」→「创建企业自建应用」，填名称和图标。
3. 进入应用 →「凭证与基础信息」，记下：
   - **App ID**（形如 `cli_xxxxxxxxxxxxxxxx`）
   - **App Secret**
4. 「权限管理」里按需开通 API 权限范围（scope），常用的：
   - 云文档：`docx:document`、`docs:doc`、`drive:drive`
   - 多维表格：`bitable:app`
   - 电子表格：`sheets:spreadsheet`
   - 即时消息：`im:message`、`im:chat`
   - 通讯录（读用户）：`contact:user.base:readonly`
   - 日历：`calendar:calendar`
   > 不确定先开通文档相关的几个，后面缺什么再补。改完权限要点「发布版本」或在测试企业里启用。
5. 如果要让它**以「你本人」的身份**操作（而不是机器人身份），还需要在「安全设置」里配置 OAuth 重定向 URL，并准备走 user access token（见第 4 节进阶）。先跑通 app 身份即可。

---

## 2. 把 MCP server 注册给 Claude Code

### 方式 A：命令行一行注册（推荐）

```bash
claude mcp add lark -- npx -y @larksuiteoapi/lark-mcp mcp \
  -a cli_xxxxxxxxxxxxxxxx \
  -s your_app_secret
```

国际版 Lark 加上 `--domain https://open.larksuite.com`：

```bash
claude mcp add lark -- npx -y @larksuiteoapi/lark-mcp mcp \
  -a cli_xxxxxxxxxxxxxxxx -s your_app_secret \
  --domain https://open.larksuite.com
```

### 方式 B：写进项目里的 `.mcp.json`（团队共享）

在项目根目录建 `.mcp.json`：

```json
{
  "mcpServers": {
    "lark": {
      "command": "npx",
      "args": [
        "-y", "@larksuiteoapi/lark-mcp", "mcp",
        "-a", "cli_xxxxxxxxxxxxxxxx",
        "-s", "your_app_secret"
      ]
    }
  }
}
```

> ⚠️ 别把带 secret 的 `.mcp.json` 提交到公开仓库。要么用环境变量，要么把它加进 `.gitignore`。
> 用环境变量的写法：把 `args` 里的 secret 换成不传，改用 `"env": { "APP_ID": "...", "APP_SECRET": "..." }`，并在 args 用 `-a $APP_ID -s $APP_SECRET`（或直接用 `--config` 指向一个本地配置文件）。

### 验证

```bash
claude mcp list
```

应该能看到 `lark` 处于 connected。进入 Claude Code 后输入 `/mcp` 也能看到它和它暴露的工具。

---

## 3. 试跑

在 Claude Code 里直接说人话，比如：

- “用 lark 列一下我最近的云文档”
- “把这个文档的内容读出来：<飞书文档链接>”
- “在多维表格 <link> 里新增一行：……”
- “给群 <chat 名/ID> 发一条消息：……”

Claude 会调用 `lark` 这个 MCP 下对应的工具。第一次调用某类接口若报权限错误，回第 1 步去开通对应 scope 再发布。

---

## 4. 进阶选项

- **精简工具集**：默认会注册很多工具，可能超出模型工具上限。用 `-t` 指定预设：
  ```bash
  claude mcp add lark -- npx -y @larksuiteoapi/lark-mcp mcp \
    -a cli_xxx -s yyy -t preset.doc.default
  ```
  常见预设：`preset.default`、`preset.doc.default`（云文档）、`preset.im.default`（消息）、`preset.bitable.default`（多维表格）、`preset.calendar.default`。也可以传具体接口名，逗号分隔。
- **以本人身份操作（user access token）**：加 `-u <user_access_token>`，或配置 OAuth 让它自助换取。适合需要访问「只有你能看到的」文档的场景。
- **rememberLogin / 本地配置文件**：可以把 app 信息写到一个本地 json，用 `--config ./lark.config.json` 引用，避免在命令行里出现 secret。
- **更新/删除**：
  ```bash
  claude mcp remove lark
  claude mcp get lark
  ```

---

## 5. 常见问题

| 现象 | 原因 / 解决 |
| --- | --- |
| `claude mcp list` 里 lark 显示 failed | Node 版本太低；或 `npx` 拉包失败（先手动 `npx -y @larksuiteoapi/lark-mcp --help` 看报错） |
| 调用接口报 `permission denied` / scope 不足 | 开放平台「权限管理」加对应 scope，然后**发布新版本** |
| 报 app 不可用 / 未启用 | 自建应用要在你所在企业「启用」或「发布」后才能用 |
| 工具太多导致报错 | 用 `-t preset.xxx` 收窄工具集 |
| 国际版账号连不上 | 命令里加 `--domain https://open.larksuite.com` |

---

## 6. 一句话给 Claude Code 的提示

> “我要在 Claude Code 里接飞书 MCP。我的 App ID 是 `cli_...`，App Secret 是 `...`（国际版/中国版）。请帮我用 `claude mcp add` 注册名为 `lark` 的 server，然后 `claude mcp list` 验证，并演示读一个飞书文档。”

参考：飞书官方 MCP 仓库 <https://github.com/larksuite/lark-openapi-mcp>
