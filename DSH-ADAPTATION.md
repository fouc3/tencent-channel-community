# DSH 适配说明（本副本相对上游的改动）

- **上游来源**：<https://github.com/tencent-connect/tencent-channel-community>（clone 到 commit `06a726b`，2026-04-29，Skill 版本 1.1.5，`_meta.json` slug `tencent-channel-community`）
- **目标环境**：DSH（DeepSeek Harness），Arch Linux，node v26.10.0 / npm 12.1.0，npm 全局前缀 `/usr`、需 sudo
- **Skill 安装位置**：`~/.dsh/skills/tencent-channel-community/`（DSH 用户级 Skill 根目录；DSH 按目录包读取 `SKILL.md` + `references/`）
- **CLI**：`tencent-channel-cli` 1.0.10，已由本机 pacman 本地包 **`tencent-channel-cli-bin-1.0.10-1`** 安装到 `/usr/bin/tencent-channel-cli`（来源与 PKGBUILD 见 `packaging/`）

## 一、审查发现的不适配项与处理

| # | 上游写法/假设 | 本环境实测 | 处理 |
|---|---|---|---|
| 1 | 装到 Agent「全局 Skill 目录」(OpenClaw/ClawHub 体系) | DSH 用户级技能目录是 `~/.dsh/skills/` | 按 DSH 目录安装；DSH 热加载，装完即被识别 |
| 2 | frontmatter 里 `metadata: {"openclaw":{"emoji":"📢"}}` | DSH 无 OpenClaw/ClawHub 概念 | 删除该字段；补 DSH 原生字段 `whenToUse` |
| 3 | `npm install -g tencent-channel-cli` | 前缀 `/usr`，需 root，且会绕过 pacman 包管理 | 改为打成 pacman 本地包安装（见下一行） |
| 4 | —（新增能力） | 上游只有 npm 包，无源码仓库 | 写了 PKGBUILD（只取 `tencent-channel-cli-linux-x64` 内的静态二进制，不含 node wrapper）→ `makepkg -f` → `sudo pacman -U`，`pacman -Q tencent-channel-cli-bin` = 1.0.10-1；`namcap` 除上游二进制固有的 PIE/RELRO 警告外无错（license 已按 `LicenseRef-custom` + 许可文件修正） |
| 5 | 登录只有「扫码授权」一条路 | CLI 另有 `.env` 凭证通道：`~/.qqcli/.env` 的 **`QQ_AI_CONNECT_TOKEN`**（用假值逐个键名探测得到，其它常见键名如 `TENCENT_CHANNEL_TOKEN`/`ACCESS_TOKEN` 均不被读取） | 用 API key 登录，`login status` → `{"valid":true,"tokenSource":"dotenv"}`，`doctor` 含「业务探测：服务调用正常」；扫码流程保留为备选 |
| 6 | CLI 状态目录 `~/.qqcli` 可直接读写 | 本会话策略为 danger-full-access 可写；但 DSH 默认 workspace-write 下 `~` 只读：`login --json` 返回 success 而 `device_auth_state.json` 未落盘，`poll-token` 报 `no such file or directory` | 写进 SKILL.md 第 4 条：①申请更宽权限（已选，凭证落真实 `~/.qqcli`）②临时 `HOME=<工作目录>/.qqcli-home` |
| 7 | 通知推送（`notices-on --session-key agent:<id>:`、`notify-daemon`、`openclaw agent --deliver`） | 本机无 `openclaw`（探测 `/usr/bin/openclaw`、`/opt/homebrew/bin/openclaw`），DSH 无 `session_status` 工具 | 标记不可用：不执行 `notices-on`、不编造 sessionKey、hint 只转述、禁起 daemon；改用 `feed get-notices` |
| 8 | CLI 版本对比 CDN 头 `x-cos-meta-tcc-cli-version` | 该头滞后：返回 `1.0.6`，npm 上已是 `1.0.10` | CLI 版本改查 `npm view tencent-channel-cli version` |
| 9 | 大段 Windows / PowerShell 专项说明 | 本机 Linux | 压缩为一行备注（.ps1 执行策略坑的结论保留） |
| 10 | 「每天使用前检测更新」+ 从 CDN/ClawHub 拿新版 | 每次联网开销；且自动更新等于改写技能本体 | 改为「本会话首次使用前检测一次」，更新**必须先问用户** |
| 11 | `login poll-token` 描述为「扫码已完成会立即返回」 | 实测未扫码时阻塞 >60s 直到授权码过期 | 注明必须等用户确认后再调用 |
| 12 | 二维码以 `qr_code`(base64) 回显 | — | 注明 `--qrcode-path` 指向工作目录 + 用 DSH 图片展示，禁止打印 base64 |
| 13 | —（新增提示） | CLI 有全局 `-d/--dry-run` | 建议先 dry-run，再让用户确认，再带 `--yes` 真跑 |

## 二、未改动的部分（与上游一致）

场景路由、全局硬规则 1–4、6（@用户用 tiny_id、高风险操作先确认、敏感信息最小化、URL 用 `<>` 包裹、153 限流 sleep 70s 重试一次）、链接识别、参数查询（`schema`）、执行阶段敏感信息策略、快捷命令与 `status: waiting` / `resume_command` 交互协议，全部保持原样。硬规则 5（鉴权失败）只补了「DSH 先换 `.env` 凭证、扫码为备选」。

## 三、安装与验证（本机已完成）

```bash
pacman -Q tencent-channel-cli-bin          # tencent-channel-cli-bin 1.0.10-1
pacman -Ql tencent-channel-cli-bin         # /usr/bin/tencent-channel-cli 等
tencent-channel-cli version                # {"version":"1.0.10"}
tencent-channel-cli login status --json    # {"valid":true,"tokenSource":"dotenv"}
tencent-channel-cli doctor --json          # 四项全 pass，含「业务探测：服务调用正常」
tencent-channel-cli manage get-my-join-guild-info --json   # 实测返回真实数据（9 个已加入频道）
```

## 四、凭证与安全

- token 存放：`~/.qqcli/.env`（`QQ_AI_CONNECT_TOKEN`），目录 700、文件 600。**不得展示、复述或写入任何输出/总结/示例命令。**
- 首次写入前若存在旧 `.env`，已备份为 `~/.qqcli/.env.bak-<时间戳>`。
- 上游 npm 包未声明 license 字段，本包按 `LicenseRef-custom` 处理并在 `/usr/share/licenses/tencent-channel-cli-bin/LICENSE` 说明权属。

## 五、已知限制

- 频道通知**无法自动推送**（原因见上表 #7），需要新消息时手动 `feed get-notices`。
- 打包产物（PKGBUILD 与 `.pkg.tar.zst`）位于本机 `/tmp`（内存盘），重启即失；PKGBUILD 已复制进本技能目录 `packaging/`，需要时重新 `makepkg -f` 即可。
- 上游 README 中的 ClawHub / OpenClaw 集成说明保留未改（仅供人阅读）。
