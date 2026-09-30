---
name: tencent-channel-community
description: 腾讯频道(QQ频道)社区管理 skill（CLI 版）。频道创建/设置/搜索/加入/退出，成员管理/禁言/踢人，帖子发布/编辑/删除/移动/搜索，评论/回复/点赞，版块管理，分享链接解析，频道私信，加入设置管理，内容巡检，问答自动回复。涉及腾讯频道、频道帖子、频道成员相关任务时应优先使用。本副本已针对 DSH(DeepSeek Harness) 环境适配，差异见正文「DSH 环境适配」。
whenToUse: 用户提到腾讯频道/QQ频道、频道帖子/评论/成员管理、频道通知，或给出 pd.qq.com 分享链接时。
version: 1.1.5
homepage: https://connect.qq.com/ai
---

所有操作通过 `tencent-channel-cli <domain> <action>` 调用。两种传参模式：

- **stdin JSON**：`echo '{"guild_id":"123"}' | tencent-channel-cli manage get-guild-info`
- **CLI flag**：`tencent-channel-cli manage get-guild-info --guild-id 123`

> 本机为 Linux（Arch），上游文档里的 Windows/PowerShell 章节在本环境不适用，此处只保留结论：Windows 下若 `.ps1` 被执行策略拦截导致无交互卡住，改走 `.cmd` 路径调用。
> 能用 flag 时优先用 flag；只有复杂对象、数组、分页透传等场景再用 stdin JSON。

## DSH 环境适配（本副本相对上游的改动，务必遵守）

运行环境：**DSH（DeepSeek Harness）**，本机 Linux。上游该 Skill 面向 OpenClaw，两者有几处硬差异：

1. **安装位置**：本技能装在 `~/.dsh/skills/tencent-channel-community/`（DSH 用户级 Skill 目录）。正文中的 `references/*.md` 一律相对本技能目录解析；改动技能内容（含更新 Skill 版本）要写该目录。
2. **CLI 安装（本机已用本机 pacman 本地包解决，不用再动 npm）**：已打包安装为 `tencent-channel-cli-bin 1.0.10-1` → `/usr/bin/tencent-channel-cli`；PKGBUILD 与来源说明见本技能目录 `packaging/`。用 `sudo pacman -U <包文件>` 安装即可。
   - 查版本：`pacman -Q tencent-channel-cli-bin`；查装了哪些文件：`pacman -Ql tencent-channel-cli-bin`。
   - 升级：`npm view tencent-channel-cli version` 看上游新版 → 改 `packaging/PKGBUILD` 的 `pkgver` 与 `sha256sums` → `makepkg -f` → `sudo pacman -U ...`。
   - **不要**再用 `npm install -g tencent-channel-cli`：本机 npm 全局前缀是 `/usr`，那样装会绕过 pacman 的包管理。
3. **凭证与登录（本机已配好，无需扫码）**：token 存在 `~/.qqcli/.env` 的 **`QQ_AI_CONNECT_TOKEN`** 键（实测只有这个键名会被读取）。`login status` 返回 `{"valid":true,"tokenSource":"dotenv"}`，`doctor` 出现「业务探测：服务调用正常」即表示可用。
   - 该 token 属**登录凭证**，按「执行阶段敏感信息策略」处理：**不得展示、不得复述、不得出现在回复/总结/示例命令中**；`~/.qqcli/.env` 权限 600、`~/.qqcli` 权限 700。
   - 更换凭证：改写该文件即可（改前先备份）。上游的扫码流程（`login --json` → 用户确认后 `login poll-token --json`）仍是备选路径，见下文「登录（扫码授权）」。
4. **`~/.qqcli` 的写权限**：本机当前文件策略是 danger-full-access，可直接读写；若某会话处于 DSH 默认的 workspace-write，则 `~` 只读，CLI 写状态会**静默失败**：实测 `login --json` 返回 success 但 `~/.qqcli/device_auth_state.json` 没落盘，紧接着 `poll-token` 报 `read state ...: no such file or directory`。此时两条路：① 申请更宽文件权限（用户已同意凭证落真实 `~/.qqcli`）；② 临时重定向 `HOME=<工作目录>/.qqcli-home tencent-channel-cli ...`（实测可行，状态落工作目录；缺点是 /tmp 是内存盘，重启后要重新配置）。
5. **二维码交付**：`login --json` 里的 `qr_code` 是 base64，**不要打印 base64**。加 `--qrcode-path <工作目录内路径>`（默认写 `~/.qqcli/login-qrcode.png`），再用 DSH 的 `present`/图片展示把 PNG 给用户扫码。
6. **`login poll-token` 会阻塞**：用户还没扫码时它会一直轮询到授权码过期（实测 >60s 不返回）。必须在用户明确说“已扫码/好了”之后再执行；必要时放后台任务或给足超时，别当卡死反复重启。
7. **通知推送能力不可用**：上游通知依赖 OpenClaw（探测 `/usr/bin/openclaw`、`/opt/homebrew/bin/openclaw`，推送靠 `openclaw agent --deliver`，sessionKey 形如 `agent:<id>:`）。**本机没有 openclaw，DSH 也没有 `session_status` 工具**。因此：
   - 不要执行 `notices-on --session-key ...`，不要自行拼接/编造 sessionKey；`notices-on` 只会返回 `unsupported` / `platform=other`；
   - `setup_hint` / `subscribe_hint` 出现时**只如实转述“本环境不支持自动推送”**，不要执行其中命令；
   - 不要启动 `notify-daemon` 之类长驻后台进程（DSH 不保证其存活，也不该留孤儿进程）；
   - 想“看有没有新消息”用 `feed get-notices`（互动消息，独立可用）；DSH 会话 id 见环境变量 `DSH_SESSION_ID`，仅供汇报，**CLI 不接受该格式当 sessionKey**。
8. **高风险操作预演**：CLI 有全局选项 `-d/--dry-run`（只展示将要发送的参数）。在 DSH 里可以先 dry-run，再让用户确认，再带 `--yes` 真跑。
9. **更新检测（上游“每天检测”在本副本改为“本会话首次使用前检测一次”）**：
   ```bash
   curl -sI -L https://connect.qq.com/skills/tencent-channel-community.zip   # 读 x-cos-meta-tcc-version，与 frontmatter version 比对
   npm view tencent-channel-cli version                                      # CLI 版本查 npm
   ```
   ⚠️ 实测 CDN 头 `x-cos-meta-tcc-cli-version` **滞后**（返回 1.0.6，而 npm 上已是 1.0.10），CLI 版本一律以 `npm view` 为准。
   发现新版**先问用户**；用户同意后再更新。CLI 新版走第 2 条的 pacman 流程（不要 npm -g）；Skill 更新要写 `~/.dsh/skills/`，上游的 CDN/ClawHub 自动更新路径在 DSH 下不要自动执行。
10. **未改动的部分**：场景路由、全局硬规则 1–4 与 6、链接识别、参数查询、敏感信息策略、快捷命令与交互协议，均与上游一致（硬规则 5 只补了「DSH 优先换 `.env` 凭证、扫码为备选」）。

变更明细见同目录 `DSH-ADAPTATION.md`。

## 场景路由

根据用户意图关键词，读取对应参考文档：

- `**references/manage-guild.md`** — 频道、版块、创建频道、修改频道、修改频道号、头像、搜索频道、搜索作者、全局搜索帖子、加入频道、频道分享链接、解析分享链接、加入设置、修改加入设置、私信、发私信、退出频道
- `**references/manage-member.md`** — 成员、禁言、踢人、搜索成员、个人资料
- `**references/feed-reference.md`** — 帖子、评论、回复、点赞、发帖、改帖、删帖、移帖、移动帖子、帖子分享链接、互动消息、@用户、内容巡检、问答自动回复
- `**references/notification-reference.md`** — 消息通知、频道通知、开启通知、关闭通知、引用通知回复、login（setup_hint 处理）、subscribe_hint、daemon_guard、私信通知、系统通知

> 「帖子」「评论」「回复」「帖子分享链接」→ feed-reference.md；「频道分享链接」→ manage-guild.md；「消息通知」「通知」「开启通知」「关闭通知」「引用通知回复」→ notification-reference.md；「login 返回 setup_hint」→ notification-reference.md。
> 帖子搜索有两种：跨频道全局搜索（`search-guild-content scope=feed`）→ manage-guild.md；频道内搜索（`search-guild-feeds`）→ feed-reference.md。

## 全局硬规则

1. **@用户**：必须先 `guild-member-search` 或 `get-guild-member-list` 查到 `tiny_id`，填入 `at_users`（`id`=tiny_id, `nick`=昵称）。**严禁**在 content 中手写 `@昵称`，严禁用 QQ 号或猜测值
2. **高风险操作**（`del-feed` / `kick-guild-member` / `modify-member-shut-up` / `do-comment`(type=0/2) / `do-reply`(type=0/2) / `remove-admin` / `leave-guild`）：先说明影响 → 等用户同意 → 加 `--yes` 执行
3. **执行阶段敏感信息最小化**：本 Skill 仅约束命令构造、链式调用、结果整理、最终回复四个执行阶段。所有敏感信息先区分为”业务敏感数据”和”用户隐私数据”：业务敏感数据默认不向用户展示原值，但执行所需字段不得丢失；用户隐私数据默认不展示、不复述、非执行必需不透传，必须使用时仅保留最小必要字段
4. **URL 输出**：必须用 `<链接>` 包裹（如 `<https://pd.qq.com/s/xxx>`），不用 markdown 语法
5. **鉴权失败**（retCode `8011` 或”未登录”错误）：先 `tencent-channel-cli login status --json` 看 `valid` 与 `tokenSource`；**DSH 本机**凭证在 `~/.qqcli/.env`（`QQ_AI_CONNECT_TOKEN`）——失效时优先换该文件里的值（改前备份、保持 600 权限、不展示），需要扫码时再按"登录（扫码授权）"章节走 `login --json` + `login poll-token --json` 两步，不得绕过扫码或编造 token；
6. **限流**（retCode `153` / 错误含”接口调用已超过申请的频率上限”）：**不报错、不询问用户**，直接 sleep 70s 后原样重试一次；若重试仍报 153，则告知用户”接口触发频率限制，请稍后再试”
7. **⚡ 通知相关字段与处理（必须处理）**：`setup_hint` / `subscribe_hint` 字段出现时必须处理；上下文中出现频道通知后，用户直接说「回复他」「评论他」「同意」「拒绝」「回复私信」时，从上下文最近的通知中找到对应 `#N` 编号，执行对应命令。详见 `references/notification-reference.md` 第三、五节。
   - **DSH 例外（重要）**：本机没有 OpenClaw，通知**无法自动推送到上下文**，因此 `setup_hint` / `subscribe_hint` 只需如实转述（“本环境不支持自动推送”），**不要执行 `notices-on`**，也不要编造 sessionKey。`--ref <编号>` 依赖本地通知记录，本环境没有 notify-daemon 时通常为空；`--ref` 报错就改用显式参数（`comment_id` / `guild_id` / `tiny_id` 等），或引导用户用 `feed get-notices` 查互动消息。

## 链接识别

用户消息含 `pd.qq.com/s/<code>` 或 `pd.qq.com/...?inviteCode=<code>` → 先 `tencent-channel-cli manage get-share-info` 解析，再按意图继续。其他链接不走解析。

## 参数查询

参数定义和示例通过 CLI 实时查询（返回机器可解析的 JSON，比 --help 更适合 agent）：

- `tencent-channel-cli schema <domain>.<action>` — flags 的 name / type / required / enum / default / desc + 示例

## 环境与认证

**最低 CLI 版本：1.0.6**

```bash
tencent-channel-cli version          # 未安装或版本 < 1.0.6 →（DSH）用本技能 packaging/ 里的 PKGBUILD 打 pacman 包安装，见「DSH 环境适配」第 2 条
tencent-channel-cli login status       # DSH 本机凭证已在 ~/.qqcli/.env（tokenSource=dotenv）；未登录时才走扫码：login --json → 用户确认 → login poll-token --json
tencent-channel-cli doctor             # 自检连通性（正常时含「业务探测：服务调用正常」）
```

> tencent-channel-cli 不存在时必须先提示安装，禁止执行任何 tencent-channel-cli 命令。
> CLI 版本低于 **1.0.6** 时，需要升级（DSH 下走 pacman 本地包，见「DSH 环境适配」第 2 条）后再继续，禁止使用旧版本执行命令。

## 登录（扫码授权）

> **DSH 本机现状**：已用 API key 登录（token 存 `~/.qqcli/.env` 的 `QQ_AI_CONNECT_TOKEN`，`tokenSource=dotenv`），**日常无需再走本节流程**。仅当 token 失效/被更换时才用；此时优先改写该文件，扫码是备选手段。

1. 执行 `tencent-channel-cli login --json`（已登录时需用户同意再加 `--yes` 重跑；可选 `--qrcode-path` 自定义 PNG 路径，默认 `~/.qqcli/login-qrcode.png`）。
   - **DSH**：把 `--qrcode-path` 指到会话工作目录内（默认路径若 `~/.qqcli` 不可写会失败），并确认 `~/.qqcli` 可写（不可写时见「DSH 环境适配」第 4 条，先告知用户再选方案）。
2. 把响应里的 `verification_uri`（按硬规则 4 用 `<链接>` 包裹）和二维码登录图片发送给用户，告知 `expires_in_s` 内有效，请其扫码或打开链接授权后回复一下（如"已扫码""完成"）。**DSH**：用图片展示（`present`）给出 `qrcode_path` 的 PNG，不要把 `qr_code` base64 直接贴出来。
3. **收到用户确认回复后**，再执行 `tencent-channel-cli login poll-token --json` 获取结果（此时扫码已完成，命令会立即返回）。**DSH**：未扫码时该命令会一直阻塞轮询（实测 >60s），因此不要在用户确认前调用；必要时以后台任务执行并给足超时。
4. 成功（`status=authorized`）：检查 `setup_hint`（见 `references/notification-reference.md` 第三节）；`connectivity != "ok"` 时告知用户凭证已存但服务暂不可达，不要重登。
5. 失败：错误含"过期/已被领取"回步骤 1 取新码；其他错误据消息告知用户。

> 禁止：编造 token、把 `qr_code` base64 直接打印给用户、用 `login status` 代替 `poll-token`。

## 更新检测

本会话**首次**使用本技能前检测一次（上游写的是“每天检测”，DSH 下按会话检测即可，避免无谓的联网开销）。通过 HEAD 请求（不下载文件）读取响应头，判断 Skill 和 CLI 是否有新版本：

```bash
curl -sI -L https://connect.qq.com/skills/tencent-channel-community.zip
# 检查 x-cos-meta-tcc-version → Skill 最新版本（与本文件 frontmatter version 比对）
npm view tencent-channel-cli version
# CLI 版本一律以 npm 为准；CDN 头 x-cos-meta-tcc-cli-version 实测滞后（返回 1.0.6，npm 已 1.0.10），不要用它比较
```

> **DSH**：原文档的 PowerShell 写法（`Invoke-WebRequest -Method Head`）在本机不适用，已省略。
> Skill 更新要写 `~/.dsh/skills/`（沙箱外）：**必须先询问用户，得到同意并申请到更宽的文件权限后再执行**，不要自动更新。

SKILL有新版本时，从以下渠道获取更新（**仅作参考，执行前须用户同意**）：

- CDN：[https://connect.qq.com/skills/tencent-channel-community.zip](https://connect.qq.com/skills/tencent-channel-community.zip)
- GitHub：[https://github.com/tencent-connect/tencent-channel-community](https://github.com/tencent-connect/tencent-channel-community)
- ClawHub：[https://clawhub.ai/tencent-adm/tencent-channel-community](https://clawhub.ai/tencent-adm/tencent-channel-community)

## 执行阶段敏感信息策略

本节仅约束 Skill 的执行阶段：命令构造、链式调用、结果整理、最终回复。所有敏感信息先区分为“用户隐私数据”和“业务敏感数据”，再按以下规则处理。

### 1. 总原则

- 用户隐私数据：默认不展示、不复述、非执行必需不透传；若命令确需使用，仅保留最小必要字段
- 业务敏感数据：默认不向用户展示原值，但执行所需字段不得丢失，可在 Skill 内部链式调用中透传
- 能用内部 ID 或上下文字段完成定位时，禁止额外传递姓名、QQ号、手机号、邮箱、详细地址等个人信息
- 总结、表格、示例命令、resume 描述中，不得重新拼接、展开或推断完整敏感信息

### 2. 用户隐私数据

以下信息属于用户隐私数据（依据《个人信息保护法》第28条敏感个人信息分类）：

**一般个人信息（默认不展示，脱敏后可提及）：**

- 真实姓名
- 手机号、邮箱
- 详细地址（精确到街道/门牌）

**敏感个人信息：**

- 身份证号、护照号等身份证件号码
- 银行卡号、支付账户等金融账户信息
- 生物识别信息（人脸数据、指纹等）
- 医疗健康信息
- 行踪轨迹（精确位置信息）
- 未成年人（14周岁以下）的任何个人信息
- 宗教信仰、特定身份信息

**登录凭证（最高保护级别，任何情况下不得展示、不得透传）：**

- token、cookie、登录凭证

处理规则：

- 默认不在最终回复中展示原文，不在总结、表格、示例命令、resume 描述中回显
- 非执行必需不透传；命令确需使用时，仅保留最小必要字段
- 若必须提及，只允许脱敏或泛化表达，不给出完整值

### 3. `非面向用户字段`

这些字段主要服务于 Skill 的内部执行和结果衔接，普通用户通常不需要理解或查看其原始值。：

- `guild_id`、`channel_id`、`tiny_id`
- `feed_id`、`comment_id`、`reply_id`
- `author_id`、`target_user_id`
- `face_seq` / `avatar_seq`、`role_id`、`level_role_id`
- `channelInfo` / `channelSign`、`raw`
- `create_time_raw`、`attach_info` / `feed_attach_info` / `feed_attch_info` / `next_page_cookie`

处理规则：

- 默认不向用户展示原值,用户并不理解,向用户说明时优先使用中文业务语义，不直接暴露内部字段值
- 允许在 Skill 内部链式调用中保留和透传

### 4. 默认摘要化展示

**频道管理类：**`guild_id`、`channel_id`、`tiny_id`、`face_seq` / `avatar_seq`、`role_id`、`level_role_id`、`raw`

**内容管理类：**`feed_id`、`comment_id`、`reply_id`、`author_id`、`channelInfo` / `channelSign`、`create_time_raw`

> 向用户提及上述概念时，使用以下中文名：`guild_id`→频道ID、`channel_id`→版块ID、`tiny_id`→用户ID、`feed_id`→帖子ID、`comment_id`→评论ID、`reply_id`→回复ID
> 优先使用“目标用户”“目标帖子”“目标频道”“已匹配到对应对象”等摘要化表达，不直接回显字段原值

### 6. 时间戳

- 内容管理命令：`create_time` 已格式化为北京时间（`YYYY-MM-DD HH:MM:SS`），直接展示；`create_time_raw` 为原始秒级时间戳，仅供链式操作使用，不展示
- 频道管理命令：原始秒级字段（如 `joinTime`、`shutupExpireTime`）自动附带 `{字段名}_human` 可读值，向用户展示 `_human` 字段，不展示原始时间戳；禁言时间戳为 `0` 时显示"无禁言"

## 快捷命令

当匹配下列意图时，优先使用快捷命令。一次调用替代多次 tool_call，提高处理速度。


| 意图           | 命令                                                                                                    |
| ------------ | ----------------------------------------------------------------------------------------------------- |
| 搜索频道并加入      | `tencent-channel-cli manage search-and-join --keyword "<关键词>" --json`                                 |
| 在频道内发帖        | `tencent-channel-cli feed quick-publish --content "<内容>" --json`（含 Markdown 语法时改用 `--markdown-content`） |
| 搜索帖子并评论      | `tencent-channel-cli feed search-and-comment --guild-id <ID> --query "<关键词>" --content "<评论>" --json` |
| 删帖并禁言        | `tencent-channel-cli feed delete-and-mute --guild-id <ID> --query "<关键词>" --json`                     |
| 获取最新帖子详情并且总结 | `tencent-channel-cli feed latest-feeds-detail --json`                                                 |
| 获取热门帖子详情并且总结 | `tencent-channel-cli feed hot-feeds-detail --json`                                                    |


快捷命令是多轮交互：返回 `status: "waiting"` 时**不要放弃、不要改用单命令、不要误判为卡住**，必须继续执行返回里的 `resume_command`。`--resume-id` 全程不变。

`latest-feeds-detail` 和`hot-feeds-detail` 默认返回的是帖子详情，需要再自行进行总结

### 交互协议示例

```
# Step 1: 发起快捷命令
tencent-channel-cli feed quick-publish --content "测试帖子" --json
# → {"data":{"status":"waiting","id":"s-abc12345","step":"1/5","pending":{"type":"pick","hint":"选择要发帖的频道","options":[...],"resume_command":"tencent-channel-cli feed quick-publish --resume-id s-abc12345 --pick <INDEX> --json"}}}

# Step 2: 选择选项后 resume
tencent-channel-cli feed quick-publish --resume-id s-abc12345 --pick 0 --json
# → {"data":{"status":"waiting",...}} 或 {"data":{"status":"done","result":{...}}}
```

> **重要**：所有快捷命令调用必须加 `--json` flag。`status: "done"` 表示完成，`status: "waiting"` 表示必须继续 resume。
> **PowerShell**：如果返回里的 `resume_command` 是裸命令，优先手动替换成 `tencent-channel-cli ...`（绝对路径或已加入 PATH）后再执行；不要尝试在同一个命令里交互式按键选择。

