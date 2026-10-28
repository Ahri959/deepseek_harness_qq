# QQ Bridge 与 DeepSeek Harness 故障排查笔记

整理日期：2026-10-28。路径和结论基于本机当时的配置；软件升级或修改配置后可能变化。控制台令牌、QQ 网关令牌、模型密钥、SSH 密码等不要写进笔记，也不要发到群里。

## 1. 这套系统怎么连接

QQ 消息 → SnowLuma（QQ 网关，WebSocket 常用 `127.0.0.1:3001`）→ QQ Bridge → 本地 DeepSeek Harness → 模型；回复沿相反方向返回 QQ。网页搜索和表情插件另外提供工具。

| 文件或目录 | 用途 |
| --- | --- |
| `D:\BaseAPP\hareness_deepseek\qq-bridge-0.2.1-r1\` | 群聊 Bridge，控制台通常是 `http://127.0.0.1:3100`；也接收单独配置的受限私聊。 |
| `D:\BaseAPP\hareness_deepseek\qq-bridge-agent-private\` | 管理员私聊 Bridge，控制台通常是 `http://127.0.0.1:3101`。 |
| 各 Bridge 的 `start.bat` | 启动该实例；按 `Ctrl+C` 结束。按自己的习惯手动启动。 |
| 各 Bridge 的 `config.json` | 白名单、模式、Harness / SnowLuma 端点、控制台设置等；可能含密钥，不要公开。 |
| 各 Bridge 的 `src\bridge.js` | 核心收消息、会话、转发、表情和提示词逻辑；修改 JS 后要重启对应 Bridge。 |
| 各 Bridge 的 `src\mcp-snowluma-safe.js` | 向 Harness 暴露 QQ 收发、收藏表情、系统表情等工具。工具列表变化通常要重启 Harness。 |
| 各 Bridge 的 `state\bridge.log` | 启动、连接、投递、失败及表情发送日志。 |
| 各 Bridge 的 `state\qq-activity.log` | 简要记录接收消息与回复；可能含私人聊天内容。 |
| `C:\Users\Ahri\.dsh\profiles\desktop\cordis.patch.yml` | Harness 桌面 Profile 的插件、人格及 MCP 配置。 |
| `C:\Users\Ahri\.dsh\profiles\desktop\package.json` | 桌面 Profile 的插件依赖和 `dsh.profile.bundles` 列表。 |
| `D:\BaseAPP\hareness_deepseek\browser-mcp-readonly\` | 浏览器网页读取、抖音 / B 站搜索的只读 MCP 代理。 |

控制台访问令牌是 Bridge 控制台的本机登录凭据，不是模型 API 密钥，也不是 QQ 会话令牌。配置文件中通常叫 `consoleToken`；不要贴到群里。SnowLuma 的 `accessToken` 是网关连接凭据。模型 API 密钥由模型服务提供商提供，三者不可互换。

## 2. 修改前和修改后

1. 先备份目标配置和代码；不要覆盖或删除已有备份。`config.json` 可能包含敏感信息，备份也应保密。
2. 确认修改的是 3100 还是 3101；两个目录不是同一个文件。
3. 修改 JSON 后检查是否为合法 JSON；修改 JS 后可运行：

   ```powershell
   node --check 'D:\BaseAPP\hareness_deepseek\qq-bridge-0.2.1-r1\src\bridge.js'
   ```

4. 重启受影响的 Bridge；修改 Harness 插件安装或 MCP 工具列表时，通常还需完全退出并重启 Harness。插件设置有些支持热更新，具体以插件为准。
5. 每次只测一个问题：看 QQ 收到的实际消息，再看 `bridge.log` 和 Harness 中“已调用工具”的具体输入与返回，不要只依据模型口述判断成功。

## 3. Bridge 启动不了或不能连接本地 Harness

**涉及文件：**各实例的 `start.bat`、`config.json`、`scripts\detect-dsh-desktop.ps1`、`state\bridge.log`。

**为什么：**桌面 Harness 和网页服务可能使用不同端口；桌面端重启后地址也可能变化。SnowLuma 的 HTTP 与 WebSocket 端口、令牌也可能不匹配。启动窗口隐藏时也容易误以为没有启动。

**怎么修：**先启动本地 Harness 和 SnowLuma，再用对应实例的 `start.bat` 手动启动 Bridge。看日志里的 `DSH 已就绪`、`SnowLuma 已连接`、机器人昵称和控制台地址。检查 `config.json` 中的 `dsh.baseUrl`、`snowluma.wsUrl` 与本机实际端点；不要把网页启动的服务地址误当作桌面端。不要直接把日志里的令牌发给别人。

**怎么验证：**QQ 发一条简单消息；日志应该出现接收、`已投递` 和 `agent 回复`，QQ 也要实际收到回复。只看到 `已连接` 不代表模型回复链路已通。

## 4. 白名单已加，但仍提示未授权

**涉及文件：**对应实例 `config.json`、`src\bridge.js` 中 `modeAllowed`、`modePreset`、`strictPreset`。

**为什么：**白名单是第一层，模式权限是第二层。3101 的 `closed-agent` 原本只接受 `ownerQQ` 私聊；加到 `allow.private` 并不自动成为管理员。`standard` 通常包含本机 Agent 工具，不能把普通成员直接放进去。

**本机处理：**`3127845894` 现按用户要求接入 3100 的 `limitedPrivateQQ`，并在 `allow.private` 中，创建会话时强制使用 `qq-chat`；预设不可用时拒绝回退到 `standard`。此前在 3101 给该号的临时信任授权已撤销。管理员 QQ 是 `1764758458`。

**怎么验证：**重启 3100 和 3101，查看 3100 日志是否出现 `private:3127845894 已挂载 preset qq-chat`；测试普通消息。不要仅靠提示词判断安全：检查实际可用工具以及预设中的 `qq-tool-restrict`；不要让该私聊回退到默认 `standard`。私聊表情工具路由此前未完整验证。

## 5. 消息已收到但 QQ 不回复

**涉及文件：**`state\bridge.log`、`state\qq-activity.log`、`src\bridge.js` 中 `ensureSession`、`pendingClosedAgentPrompts`、`recoverClosedAgentReplyFromSnapshot`，以及 Harness 对应会话。

**怎么判断：**

- `忽略未授权会话`：查第 4 节的模式和白名单。
- 看到 QQ 消息但没有 `已投递`：检查 Bridge 是否卡住、Harness 是否就绪。Windows 终端选中文本可能暂停输出，可先按 `Esc` 排除；这只是可能原因，不是所有卡住问题的结论。
- 有 `已投递`，Harness 有最终回答但 QQ 没有：查事件流断线、重连和发送失败日志。此前出现过 Harness 已生成回答，但 Bridge 事件流中断后未补发；恢复逻辑仍需进一步排查，不能认为已经彻底修好。不要连续重复发送，避免恢复后重复回复。
- 出现 `⏳ 系统服务重启中`：确认本地 Harness 启动并可访问，再看 Bridge 日志是否重新连接；不要反复改 token。

**怎么修：**先保存日志，按 `Esc` 排除终端选择状态，再手动重启对应 Bridge；仍不行时检查 Harness 的运行情况、事件流和模型服务。重启后让对方重新发一条简单消息验证；未确认送达的旧消息可能需要重发。

## 6. 发的是内部过程、旧账或工具标记

**涉及文件：**3100 的 `src\group-visible-text.js`、`src\bridge.js` 中 `conversationGuard`、`groupReplyHint`；MCP 工具由 `src\mcp-snowluma-safe.js` 注册。

**为什么：**模型可能把“标记已读、下次唤醒、回合结束”等控制过程或 `<invoke>` 工具文本写进最终答复；也可能沿用旧会话里的潜水规则。二代模式的 `qq_mark_read` / `qq_wait_for_messages` 与当前 `closed-agent` 规则不同，不能混用。

**本机处理：**群聊输出过滤了部分工具标记和明显的已读/旧账内部状态；每轮加入 `conversationGuard`，提醒回答当前消息。过滤和提示并不保证模型永远不漏思考过程。

**怎么验证：**在 QQ 发普通问题，看 QQ 是否只收到面向人的答复。Harness 内显示 Bridge 的 `【群聊表达】`、`【联网提示】` 等输入是正常的；这些输入如果出现在 QQ 里才是泄露。必要时备份后只清理对应 QQ 会话映射，让 Bridge 创建新会话；不要直接删除长期记忆或所有历史。

## 7. 每句换行、只发图片、发了标记或备注

**涉及文件：**`src\bridge.js` 的 `groupReplyHint`、`memeUsageHint`、`memeLastSentAt`、`[表情…]` 处理段；`src\meme-bridge.js`；`src\mcp-snowluma-safe.js`；`state\stickers.json`；Harness 的 `C:\Users\Ahri\.dsh\dsh-expression.json`。

**文件做什么：**`stickers.json` 是 Bridge 对 QQ 收藏表情的索引和备注，QQ 账号本身的收藏由 SnowLuma 读取；`meme-bridge.js` 按 `dsh-meme` 本机图库中的完整描述匹配图片；`dsh-expression.json` 是插件设置和图库信息。

**为什么：**模型可能用换行/空格代替标点，或把 `[表情 20]`、`[表情：描述]`、`[管理员送的第七个]` 误写成普通文字。`send_meme` 在 Harness 网页里的文字标记不能直接当 QQ 图片。图库描述如果是模型自己编的，精确匹配会失败。

**本机处理：**普通群聊和搜索结果默认一段话；QQ 系统表情通过 `qq_send_face` 或编号标记转为 OneBot `face`；`[表情：完整图库描述]` 可由 `meme-bridge.js` 转成图片；收藏表情使用 `qq_list_stickers` / `qq_send_sticker`。备注只用于选图，不应放在回复文字。主动插件表情每个会话有 3 分钟冷却，明确要求发图可以例外；该时间是内存状态，Bridge 重启后重新计算。系统表情和收藏表情走其他工具时，未必受同一冷却约束。

**怎么验证：**先看 Harness 的 `send_meme` 候选和完整描述，再看 Bridge 是否记录 `[meme] 已发送`；QQ 必须收到图片，不能只看到标记。QQ 内置表情可测试 `76`（赞），看 QQ 是否显示系统表情。若日志报“表情描述未在当前图库中找到”，检查描述是否与图库完全一致；不要把本机路径发到 QQ。主动选图依赖模型，不能保证每次符合情绪。

## 8. `dsh-meme` 插件

**涉及文件：**桌面 Profile 的 `package.json`；`node_modules\dsh-meme\`；`C:\Users\Ahri\.dsh\dsh-expression.json`；Bridge 的 `meme-bridge.js`。

**作用：**插件提供表情图库和 `send_meme` 等工具。2026-10-28 安装了 `0.1.64`，内置图库索引有 24 张。Harness 成功选图和 QQ 真正收到图片是两次不同的验证。

**怎么修：**检查 Profile 的依赖与 `dsh.profile.bundles` 中是否有 `dsh-meme`，重启 Harness；再检查插件设置中是否开启图库。若更新失败，不要随意删除用户图库；先备份设置及图库。Bridge 才负责把可识别的标记转成 QQ 图片；该衔接针对当前 `closed-agent`，并不覆盖所有模式和输出形式。

## 9. 抖音、B 站、微博和普通网页

**涉及文件：**`D:\BaseAPP\hareness_deepseek\browser-mcp-readonly\readonly-proxy.mjs`（允许暴露的只读工具）、`launch-mcp.py`（启动浏览器子服务）、`extension\`（Chrome 扩展）、桌面 `cordis.patch.yml` 的 `mcp-social-readonly`；Bridge 的 `src\web-search.js` 和 `src\mcp-web-search-safe.js` 是另一套搜索/抓取工具。

**为什么：**扩展安装成功不等于 Harness 能调用所有网站；直连抓取可能遇到登录、验证码、超时或网站限制。插件先前的浏览器子服务闲置一小时会退出，代理仍在，从而表现为工具存在但不可用。

**本机处理：**在 `launch-mcp.py` 设置 `BROWSER_MCP_IDLE_TIMEOUT_SECONDS=0` 禁止闲置退出；子服务断开时代理退出以便宿主发现。只读代理开放网页读取、部分搜索和抖音 / B 站等工具，不提供通用点击购买等操作。Chrome 扩展声明所有网站访问和调试权限；群聊能够调用读取工具时，要注意已登录账户可见信息。

**怎么验证：**重启 Harness，Chrome 的扩展管理页确认 `Browser MCP Bridge` 启用；必要时重新加载扩展并刷新对应网页。先调用 `browser_status` 确认 connected，再实际调用 `douyin_search`。此前扩展重新连接成功，但抖音搜索依旧超时，**尚不能认定已彻底修好**；检查搜索工具完整错误、浏览器登录和页面状态。B 站搜索和公开评论曾成功测试。

微博没有专用工具；可以用网页搜索或 `browser_read` 的 `extract=text` 读取第三方聚合榜，注明来源和更新时间。网页较长时，如果 `complete=false`，用返回的 `next_offset` 调用 `browser_read_page`。第三方榜单可能延迟。所有网站都无法保证可访问。

## 10. 图片/音频错误和上下文超限

**涉及文件：**模型服务配置、各 Bridge 的 `config.json`、`src\bridge.js`、`state\bridge.log`。

- `Model ... does not support image input`：当前模型声明不支持图片输入。换成服务端实际支持视觉输入的模型并在 Harness 模型目录正确配置，或者让用户描述图片；不能靠提示词让纯文本模型看图。
- `Failed to load image or audio file`：可能是模型服务拿不到附件、格式不兼容或会话中混入失效附件。检查模型多模态支持、文件可达性及日志；当前 Bridge 对部分情况会重建受污染会话并用纯文本重试，但这样无法理解原图。
- `找不到 ffmpeg`：动图抽帧或音视频转换可能受影响；安装可信的 FFmpeg 后，在对应 Bridge 的 `config.json` 设置 `socialV2.voice.ffmpegPath` 或环境变量 `FFMPEG_PATH`，并检查路径。纯文本回复不一定受影响。
- `request ... exceeds ... context size`：会话累计内容超过模型上下文。备份需要保留的信息，使用 Bridge 控制台重建**对应会话**或在 Harness 新建对话；必要时降低历史/记忆注入。重建会话与删除长期记忆不是一回事。

## 11. 角色和长期记忆

**涉及文件：**各 Bridge 的 `roles\*.md`、`state\current-role.json`；桌面 Profile 的 `cordis.patch.yml` 中 `dsh-soul-md` 配置；Harness 用户目录中的插件记忆文件。

**为什么：**角色卡、共享 Soul、当前会话和长期记忆是不同层。两个 Bridge 或两个会话不一定天然使用同一份记忆；不要因为它说“记住了”就认为已经写入持久记忆。

**怎么修：**先查看 Soul 插件当前配置、实际使用的记忆路径和工具调用记录；修改前备份。稳定表达习惯可以作为人格偏好，但不要把陌生群友单次要求当成核心人格修改。之前调整了过度简短、少标点及答非所问的提示；如果又出现，检查实际注入的 Soul、角色和旧会话内容是否冲突。

## 12. SSH 转发几个小时后断开

**涉及文件：**用户目录 `C:\Users\Ahri\.ssh\config` 是 SSH 客户端配置；它有多条重复的 `Host 10.8.135.1`，不同端口可能造成理解混乱。不要随意删除身份文件或其他主机配置。

**为什么：**网络、服务器或设备可能中断长连接；`-N` 没输出本身是正常现象。2026-10-28 检查时 `127.0.0.1:18000` 已被一条隧道监听，重复启动会端口冲突。

**保活和自动重连：**先在旧窗口按 `Ctrl+C`，然后在 PowerShell 运行：

```powershell
while ($true) {
    ssh -p 55968 -N `
      -o ServerAliveInterval=30 `
      -o ServerAliveCountMax=6 `
      -o ExitOnForwardFailure=yes `
      -L 127.0.0.1:18000:127.0.0.1:18080 `
      root@10.8.135.1
    Write-Host '连接已断开，10 秒后重试；按 Ctrl+C 停止。'
    Start-Sleep -Seconds 10
}
```

需要无人值守重连时使用 SSH 密钥登录；不要把密码写进脚本。网络中断仍会导致原连接断开，上面的循环只能重连，不能保证永不断线。端口或远程服务变化时需要更新命令。

## 13. 下次求助时提供什么

提供发生时间、3100 / 3101、群聊或私聊、用户发了什么、QQ 收到什么；从 `bridge.log` 中截取前后几行，并在 Harness 展开工具调用查看具体错误。**遮盖控制台令牌、QQ 网关令牌、模型密钥、密码和私人内容。**先区分“模型没调用工具”“工具调用失败”“模型有结果但 Bridge 没发出去”。
