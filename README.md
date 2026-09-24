# opencode-night-shift

> 给 opencode 的**班次制值守插件**：让多个会话一直干活，到点自动收工。
> Shift-based session supervisor for opencode — keep multiple sessions working until an end time, then stop.

[![CI](https://github.com/kujiangmudao/opencode-night-shift/actions/workflows/ci.yml/badge.svg)](https://github.com/kujiangmudao/opencode-night-shift/actions/workflows/ci.yml)
[![npm](https://img.shields.io/npm/v/opencode-night-shift)](https://www.npmjs.com/package/opencode-night-shift)
[![license](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

---

## 为什么不是「又一个自动续跑」

市面上已有不少 auto-continue / loop 插件（`opencode-auto-resume`、`opencode-loop`、goal 模式等），它们解决的是「别停下来」。本项目解决的是另一个问题：**「替我值一个班」**。

| 能力 | 常见 auto-continue / loop | 本项目（班次制） |
| --- | --- | --- |
| 停止条件 | 无限循环 / 目标完成 | **到点收工**（`endTime`，如 08:30） |
| 会话数量 | 通常单会话 | **一个班次看多个会话**，各自计数 |
| 消息内容 | 固定「继续」 | **模板 + 真实时间锚点**（`{time}` / `{remaining}`） |
| 忙碌感知 | 少数支持 | **发送前检查会话状态**，绝不插队 |
| 完成语义 | 全局停止 | **按会话移除**，其余继续 |
| 依赖 | 不一 | **零依赖** |

核心使用场景：**晚上把任务交给它，睡觉；早上到点它自动收工**，日志里能看到夜里被拉起几次、每个会话干到几点。

## 功能

- ⏰ **班次窗口**：`enabled` + `endTime`，到点自动全局停止
- 🗂 **多会话**：一次值守多个会话；某个会话输出完成标记后只移除它
- 🕐 **真实时间锚点**：续跑消息里的 `{time}` / `{remaining}` 由插件替换成本地真实时间，专治“天亮了我不干了”的幻觉
- 🛡 **误判防护**：自己发的消息不会触发停止；`别收工` 不会触发停止；完成标记只认最后一行（兼容 markdown 加粗与全角括号）
- 😴 **忙碌感知**：会话正忙（busy/retry）时跳过，等下次空闲
- 🔁 **卡死自愈（可选搭配）**：配合网络错误自动续跑类插件使用，本项目只负责“该不该继续”，不抢网络错误的活
- ⚡ **性能**：通过 `message.updated` 跟踪最后一条助手消息，检查时只取那一条（大会话不再全量拉取）
- 🧪 **可测试**：22 个单元测试，CI 覆盖 Linux + Windows

## 安装

要求 opencode（插件 API `@opencode-ai/plugin` 1.18.x），Node ≥ 20，无需任何运行依赖。

**方式一：npm 插件（推荐）**

```jsonc
// opencode.json
{
  "$schema": "https://opencode.ai/config.json",
  "plugin": ["opencode-night-shift"]
}
```

**方式二：手动安装**

把 `src/night-shift.js` 复制到 `~/.config/opencode/plugins/` 目录，重启 opencode。

> 从源码 checkout 运行时会自动把数据存到 `<repo>/data/`；npm 安装时存到 `~/.local/share/opencode-night-shift/`。都可用环境变量覆盖。

## 使用

插件没有图形界面，两种方式开始一个班次：

**A. 编辑配置文件（无需重启，插件按修改时间热加载）**

数据目录下的 `night-shift.json`：

```json
{
  "enabled": true,
  "endTime": "08:30",
  "message": "现在是 {time}（距离结束还有 {remaining}）。\n\n请继续未完成的任务，不要停止、不要等待用户输入。\n如果全部完成并通过验证，最后单独输出一行：[夜间任务完成]",
  "sessions": [
    { "id": "ses_你的会话ID", "sent": 0, "lastSentAt": 0 }
  ]
}
```

- `{time}` / `{remaining}` 会被替换为真实时间与剩余时长
- `endAt` 可省略，插件会根据 `endTime` 自动算出下一次该时间点
- 从旧版单会话配置（`sessionId`）会自动迁移

**B. 用可选的网页面板**（另一个工具：`opencode-dashboard`，本仓库不包含）勾选会话、设置结束时间与消息模板。

**停止方式（任一）**

- 到 `endTime` 自动收工
- 在任一被值守的会话里发一条**以停止词开头**的短消息：`收工` / `先收工吧` / `停止夜班，睡觉了`（`别收工` 不会误触发）
- 模型最后一行输出 `[夜间任务完成]` / `[task done]`（只停那一个会话）

**日志**：数据目录下的 `night-shift.log`，记录每次续跑、跳过原因、自动停止原因。

## 配置参考

| 字段 | 说明 |
| --- | --- |
| `enabled` | 班次开关 |
| `endTime` | 本地时间 `HH:MM`，到点全局停止 |
| `endAt` | 绝对时间戳（毫秒），可省略 |
| `message` | 续跑消息模板（支持 `{time}` `{remaining}`） |
| `sessions[]` | 值守的会话列表：`{ id, sent, lastSentAt }` |

环境变量：

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `OC_NIGHT_DIR` | 见上 | 数据目录 |
| `OC_NIGHT_CONFIG` / `OC_NIGHT_LOG` | 同上 | 单独指定配置文件 / 日志路径 |
| `OC_NIGHT_DELAY_MS` | `15000` | 空闲后延迟多久发续跑（给手动干预留时间） |
| `OC_NIGHT_MIN_INTERVAL_MS` | `30000` | 同一会话两次续跑的最小间隔（被挡住的空闲会补发，不丢事件） |
| `OC_NIGHT_SHIFT` | — | 设为 `0` 完全禁用插件 |

## 工作原理（30 秒版）

插件订阅 opencode 的三类事件：

- `session.idle` → 该会话是否在被值守列表里？上一轮是错误吗（交给网络恢复类插件）？末行有完成标记吗？都不是 → 延迟后发送模板消息
- `session.status` → 记录 busy/retry，发送前复查，避免插队
- `message.updated` → 记录最后一条助手消息 ID，检查时只取这一条

配置读写对多进程（插件 + 外部面板）做了兼容：BOM 容错、未知字段保留、旧格式迁移、mtime 热加载。

## 开发

```bash
npm test        # node:test，22 个用例
```

仓库结构：

```
src/night-shift.js        插件本体（零依赖）
test/night-shift.test.mjs 单元测试（含 mock opencode client）
.github/workflows/ci.yml  CI：Linux + Windows × Node 20/22
```

## FAQ

- **它会强制中止卡住的会话吗？** 不会。设计上从不 abort 会话，避免误杀你正在跑的长任务；卡死场景交给专门的卡死恢复类插件。
- **会上传数据吗？** 不会。只读写本地文件，不发起任何网络请求。
- **和网络错误自动续跑冲突吗？** 不冲突：本插件遇到错误消息会跳过，让专门的错误恢复插件处理。
- **日志/提示是中文？** 默认中文，`message` 模板可自定义为任意语言；完成标记同时识别 `[task done]`。

## English (condensed)

**opencode-night-shift** is a shift-based session supervisor plugin for [opencode](https://opencode.ai): arm one or more sessions with an end time, and the plugin sends your continuation template whenever they go idle — with real-clock anchors, busy-awareness, per-session completion markers and zero dependencies.

```jsonc
// opencode.json
{ "plugin": ["opencode-night-shift"] }
```

Config lives in `night-shift.json` (see the schema above); `{time}` and `{remaining}` placeholders are rendered with the real local clock. Stop with the end time, a short stop phrase (`收工`), or a `[task done]` marker on the last line. No network calls, no session aborts. MIT licensed.

## License

[MIT](LICENSE)
