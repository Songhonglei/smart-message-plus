---
name: smart-message-plus
description: >
  Cross-platform IM messaging CLI for DingTalk and Feishu (Lark) enterprise
  apps. Send P2P / group messages with text, Markdown, images, files,
  @mentions, recall, message templates, contact aliases, multi-account
  management, department broadcast, and a configurable safety gate with
  admin approval codes. Use when the user asks to send messages via
  DingTalk / Feishu ("用钉钉/飞书发消息", "smart-message 发给 XX",
  "跨平台发消息"). Message content must be provided by the user - AI must
  never draft it autonomously.
---

# smart-message-plus

- **Version**: 1.0.0
- **License**: MIT
- **Author**: Evan Song · [github.com/Songhonglei](https://github.com/Songhonglei)
- **Repository**: https://github.com/Songhonglei/smart-message-plus

> Unified CLI to send messages via **DingTalk (钉钉)** and **Feishu (飞书)** enterprise apps — with contact aliases, templates, media, recall, department broadcast, and a configurable safety gate with admin approval codes.

## ⚠️ 消息内容红线（必读）

**`--msg` 的内容必须由用户明确提供。AI/Agent 禁止代拟、补全、润色后直接发送。**
唯一例外：用户明确授权的自测场景，且收件人仅为用户本人。

## 快速开始

```bash
SCRIPT="python3 <SKILL_DIR>/scripts/send.py"

# 1. 保存账号（钉钉企业内部应用）
$SCRIPT --save-account my-dingtalk dingtalk "钉钉Bot" \
  --app-key <AppKey/ClientID> --app-secret <AppSecret> [--robot-code <robotCode>]

# 飞书自建应用
$SCRIPT --save-account my-feishu feishu "飞书Bot" \
  --app-id <App ID> --app-secret <App Secret>

# 2. 验证凭证
$SCRIPT --test-account my-dingtalk

# 3. 发送
$SCRIPT --provider dingtalk --to 13800000000 --msg "内容"
$SCRIPT --provider feishu --to user@example.com --msg "内容"
```

## 收件人写法（三级解析）

1. **联系人别名**（推荐）：`--to 张三` — 一个别名可存双平台 ID
2. **平台原生标识**：钉钉=手机号/userId；飞书=邮箱/手机号/open_id
3. 解析失败时会给出维护别名的具体命令

```bash
# 维护别名（single: 一人多平台；members: 多人组）
$SCRIPT --save-contact 张三 single "dingtalk:0144xxx,feishu:ou_xxx"
$SCRIPT --save-contact 核心组 members "张三,李四"
$SCRIPT --save-group 项目群 dingtalk cidXXXX==
$SCRIPT --list-contacts
```

> 钉钉**不支持邮箱直发**（无邮箱查询接口）；飞书账号需绑定邮箱/手机号才能反查，
> 查不到时用别名兜底。

## 常用命令

```bash
# 单聊 / 批量单聊
$SCRIPT --to 张三 --msg "内容"
$SCRIPT --to 张三 李四 13800000000 --msg "内容"

# 群聊（群别名或原生群ID）+ @
$SCRIPT --chat-id 项目群 --msg "通知" --mention 张三
$SCRIPT --chat-id 项目群 --msg "通知" --mention-all

# Markdown（飞书自动走卡片 lark_md 渲染）
$SCRIPT --to 张三 --markdown --msg "**加粗** [链接](https://example.com)"

# 图片 / 文件
$SCRIPT --to 张三 --image /path/a.png /path/b.png
$SCRIPT --chat-id 项目群 --file /path/report.xlsx

# 模板（templates/<name>.txt，{{var}} 占位）
$SCRIPT --to 张三 --template notice --vars "title=标题" "date=今天"

# 卡片消息（v1.1：标题 + 正文 + 跳转按钮）
$SCRIPT --to 张三 --card --card-title "发布通知" \
  --button "查看详情|https://example.com" --button "审批|https://example.com/ok" \
  --msg "**v2.0 已发布**\n\n请及时验收"
# 钉钉=ActionCard（最多5按钮，无按钮时降级带标题MD）；飞书=interactive卡片（header+lark_md+按钮）

# 部门广播（builtin=平台通讯录，custom=自定义接口）
$SCRIPT --broadcast --department "技术部" --msg "通知" --dry-run   # 先预览
$SCRIPT --broadcast --department "技术部" --msg "通知"

# 建群（v2.0）
$SCRIPT --provider feishu --create-group "项目群" --members 张三 李四     # 飞书（机器人自动入群，全能力）
$SCRIPT --provider dingtalk --create-group "项目群" --owner 张三 --members 李四  # 钉钉必须给群主
$SCRIPT --provider feishu --list-groups     # 机器人所在群列表（仅飞书）
# 建群成功自动保存群别名，之后直接 --chat-id 项目群 发消息
#
# 钉钉建群后会输出「添加机器人」提醒：说明补加机器人可解锁按钮卡片/撤回/@机器人，
# 并给出手动添加步骤（群设置→机器人→添加）和切换 cid 通道的命令。
# 高级：已有场景群模板ID时可 --with-bot --scene-template <ID>（机器人建群即在群内）

# 撤回 / 日志
$SCRIPT --list-log
$SCRIPT --recall <MSG_ID>          # 钉钉不限时；飞书 24h 内

# 账号管理
$SCRIPT --list-accounts / --set-default-account <slug> / --remove-account <slug>

# 通用
--dry-run       # 只预览不发送
--account <slug> # 指定账号
--provider dingtalk|feishu
```

## 安全门控（可配置）

| 人数 | 行为 |
|------|------|
| ≤50 | 直接发送 |
| 51-100 | ⚠️ 需 `--force-send` |
| >100 | 🔐 通知管理员，需 `--approve-code <码>` |

- 审核码：一次性、30 分钟有效、绑定收件人+内容（改内容作废）
- 管理员通知通过钉钉/飞书 DM 送达（配置见下）
- 数据目录 `config.json`：

```json
{
  "safety_gate": {
    "enabled": true,
    "warn_threshold": 50,
    "review_threshold": 100,
    "admins": [{"provider": "dingtalk", "user_id": "<直接userId，不允许别名>"}],
    "notify_provider": "dingtalk"
  }
}
```

- 关闭门控：`safety_gate.enabled = false`
- 群发审计日志：数据目录 `broadcast_log.jsonl`
- 退出码：`0` 成功 / `1` 一般错误 / `3` 触发警告线（需 `--force-send`）/ `4` 触发审核线（需 `--approve-code`）

## 数据与安全

- 数据目录：`$SMP_DATA_DIR` > `<workspace>/.smart-message/` > `~/.smart-message/`
- 凭证文件 600 权限、原子写入、永不进 git；日志和回显自动脱敏
- 所有 HTTP 显式 timeout + 429/5xx 退避重试；token 缓存提前 5 分钟刷新

## 平台差异速查

| | 钉钉 | 飞书 |
|--|------|------|
| 单聊收件人 | 手机号/userId | 邮箱/手机号/open_id |
| Markdown | 原生 | 卡片 lark_md 渲染 |
| 图片上限 | 20MB | 10MB |
| 文件上限 | 20MB | 30MB |
| 撤回时限 | 不限 | 24 小时 |
| 群聊前提 | 机器人先进群 | 机器人先进群 |

详细：`references/provider-matrix.md`（能力矩阵）、`references/console-setup.md`（后台配置+权限清单）、`references/org-lookup-contract.md`（自定义组织架构接口契约）。

## 部门广播的组织架构来源

- **builtin**（默认）：钉钉/飞书通讯录 API（需通讯录读权限）
- **custom**：`config.json` 配 `org_lookup.mode=custom` + `custom_url`，按契约实现自己的查询服务（鉴权头走环境变量，见契约文档）

## 依赖

- **零第三方依赖**：仅 Python ≥ 3.9 标准库（urllib/json/argparse/hashlib/secrets），无需 pip install
- `scripts/core/`、`scripts/providers/` 为本 skill 内部本地包（经 sys.path 导入），**不是** PyPI 包
- `config.json` / `contacts.json` / `accounts.json` 等均为运行时在数据目录自动生成的文件，不随 skill 分发
- 可选环境变量：`SMP_DATA_DIR`（覆盖数据目录）、`SMP_ORG_AUTH`（自定义组织架构接口的鉴权头）

### 内部模块清单

| 文件 | 职责 |
|------|------|
| `scripts/send.py` | CLI 主入口（参数解析/流程编排） |
| `scripts/core/config.py` | 数据目录/配置读写（原子写+600） |
| `scripts/core/accounts.py` | 多账号管理 + token 缓存 |
| `scripts/core/contacts.py` | 联系人/群别名 |
| `scripts/core/http.py` | stdlib HTTP（超时+退避重试+multipart） |
| `scripts/core/safety.py` | 安全门控 + 审核码 |
| `scripts/core/sendlog.py` | 发送日志（撤回路由） |
| `scripts/core/templates.py` | 消息模板渲染 |
| `scripts/providers/base.py` | Provider 抽象基类 + 能力矩阵 |
| `scripts/providers/dingtalk.py` | 钉钉适配器 |
| `scripts/providers/feishu.py` | 飞书适配器 |
| `scripts/providers/org_lookup.py` | 部门查询分发（builtin/custom） |

## 版本

- v1.0.0（开源首版）：钉钉+飞书双平台；文本/MD/图片/文件/@/撤回/单聊/群聊/部门广播；卡片消息（ActionCard/interactive）；建群+群列表；多账号；联系人别名；模板；dry-run；安全门控+审核码
- 规划：企业微信（WeCom）/Slack provider

### 钉钉建群的通道差异（重要）

钉钉普通群**无法通过 API 把机器人加进群**（平台限制），且场景群模板的控制台配置页在钉钉客户端外无法正常使用（实测 JSAPI 4040 报错）。因此钉钉 `--create-group` 策略为：

1. **直接建普通群**（chatId）——建群即可发消息，走「应用消息通道」（chat/send）：文本/MD/图片/文件，**不支持按钮卡片和撤回**
2. **建群后自动输出提醒**：列出补加机器人可解锁的能力（按钮卡片/撤回/@提醒/@机器人触发），附手动添加步骤（群设置→机器人→添加，约30秒）和切换命令 `--save-group <别名> dingtalk <cid>`
3. 高级路径：已有可用场景群模板ID时 `--with-bot --scene-template <ID>` 可让机器人建群即在群内（全能力）

飞书无此差异：建群时机器人自动入群，全能力可用。
