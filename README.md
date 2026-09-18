# dsh-router-flash

> 让 **opencode-go 的 DeepSeek V4 Flash** 从「鬼模式」切换到「神模式」的 dsh agent preset 标准分发包。

标准 DSH bundle 插件包：安装后自动把 `router-flash` preset 同步到 `~/.dsh/.agent-presets/router-flash`，重启 DSH 即生效。原 `install.sh` 手动复制方式已由本 bundle 替代。

## 这是什么

一个 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)（dsh）的 agent preset 分发包。它把 [dsh-router-standard](https://github.com/yjh051108/dsh-router-standard) 研究里为 **Flash** 标定的最优引导（w7 persona + 深度思考锚）适配到 **opencode-go provider 的 `deepseek-v4-flash`** 上。

**核心问题**：Flash 在默认（无引导）条件下会进入「鬼模式」——思考浅、草草动手、质量差。这不是 Flash 能力不行，而是**引导条件不对**。补上正确的引导（分类 + 回顾 + 反跑题 + 深度思考 + 决策闭环），Flash 就能进入「神模式」——深度规划、高质量交付、自测零错误。

## 安装

```bash
dsh plugin --profile web add -w github:xiaoxianyu-office/dsh-router-flash#v0.1.0
```

安装后**重启 DSH**：`router-flash` preset 会被自动安装到 `~/.dsh/.agent-presets/router-flash`。

然后编辑 `~/.dsh/settings.yaml`，把默认 agent preset 指向它（模型也需指向 Flash）：

```yaml
agent-default-model:
  provider: opencode-go
  model: deepseek-v4-flash
  reasoningEffort: max
agent-presets:
  default: router-flash
```

再次重启 dsh，新会话即使用 `router-flash` preset + `deepseek-v4-flash` 模型。

## 升级

重复 `add` 并指定最新 tag，**不要使用 update 选择 Git 引用**：

```bash
dsh plugin --profile web add -w github:xiaoxianyu-office/dsh-router-flash#v0.1.0
```

重启 DSH 后，包内变更的 preset 文件会自动覆盖到 `~/.dsh/.agent-presets/router-flash`。

## 卸载

```bash
dsh plugin --profile web remove dsh-router-flash
```

卸载后重启 DSH。注意：bundle 安装的 preset 文件（`~/.dsh/.agent-presets/router-flash`）会保留，如需彻底移除可手动删除该目录，并在 `~/.dsh/settings.yaml` 中移除 `agent-presets.default` 指向。

## 使用

1. 安装并配置后重启 dsh，新会话会自动使用 `router-flash` preset + `deepseek-v4-flash` 模型。
2. 直接提交任务即可。persona 会自动注入「分类 + 回顾 + 反跑题 + 深度思考 + 决策闭环」五个锚。
3. 会话启动后，模型的 system prompt 应包含：
   ```
   You are a helpful assistant.
   Before acting, decide the task type (build or fix)...
   Before acting, briefly review what you have already done...
   Do not run environment checks (echo, whoami, uname...)...
   Think deeply about the architecture, edge cases...
   Produce when your information is complete...
   ```

## 跨平台支持

本 preset **三平台通用**，dsh 会自动选择 shell 工具：

| 系统 | shell | 状态 |
|---|---|---|
| Linux | bash | ✅ 实测 |
| macOS | bash | ✅ 自动适配 |
| Windows | pwsh（PowerShell） | ✅ 自动适配 |

> 说明：`agent.cordis.yml` 内已有 `process.platform === 'win32'` 判断，Windows 自动禁用 bash、启用 pwsh。核心路由逻辑（persona / 引导 / 模型识别）与操作系统无关。

## 依赖

- [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)（dsh）rc.6+
- opencode-go provider 的 `deepseek-v4-flash` 模型（`https://opencode.ai/zen/go/v1`）

## 适配了什么（与原版 dsh-router-standard 的区别）

原版依赖三个在你的 dsh 版本上会失效的机制，本 preset 已修复：

1. `ctx.on('session/event')` 注入 —— dsh rc.6 中 session/event 是 session-scoped，agent-plane preset 收不到。
2. `target.inbox.append` —— agent 对象没有 `.inbox` 属性。
3. assemble 时 `session.events` 里还没有 user/message —— 时序问题。

**修复方式**：把引导静态合并进 `WEAK_FLASH` persona，避免依赖任何动态注入机制。对固定任务同样有效，且更简单可靠。

### dsh 0.1.5 适配

`0.1.5-rc.1` 起有两处破坏性变更，本 preset 已适配：

1. `@deepseek-ai/dsh-persona` 的配置 schema 由 `text` 拆分为 `prefix` / `suffix` —— 旧写法在挂载时报 `invalid config: $.prefix missing required value`，现已改为 `prefix`。
2. `session.events` 已移除 —— 读会话事件改用 `session.snapshotEvents()`（否则首轮运行报 `Cannot read properties of undefined (reading 'some')`）。

验证环境：dsh `0.1.5-rc.2` + `opencode-go/deepseek-v4-flash`。切换预设后可正常对话，并实测工具调用（`echo hi-from-router` 由 PowerShell 执行、退出码 0）。

兼容性：persona 双写 `prefix` + `text`（schemastery 忽略未声明键），事件读取用特性探测（`snapshotEvents()` 优先、回退 `session.events`），因此同一份 preset 在 `0.1.2-rc.1` ~ `0.1.5-rc.2` 两代宿主上均可挂载运行。

## 适用范围

- ✅ 本 preset 专为 **Flash** 设计：命中 `isFlashModel` 后一律走 weak 模式（作者实测 w7 最优解）。
- ✅ 非 Flash 模型（如 pro）不受影响，走原版关键词分类逻辑。
- ✅ 复杂构建任务（如大型工程、从零开发）尤其受益于深度思考锚。

## 包结构

```
dsh-router-flash/
├── package.json      # dsh.bundle（组合补丁）+ dsh.client（标准双面）声明
├── cordis.patch.yml  # 组合补丁：insert dsh-router-flash 插件行
├── lib/
│   ├── index.js      # host 端：启动时同步安装 preset 到 ~/.dsh/.agent-presets
│   └── client.js     # client 端：空实现占位（无浏览器 UI）
└── preset/           # router-flash agent preset 本体
    ├── preset.yml
    ├── agent.cordis.yml
    ├── router-bootstrap.mjs
    └── router-core.mjs
```

## 说明

- host 插件依赖宿主服务：`fs` / `sandboxPolicy`；写 `~/.dsh/.agent-presets` 使用 `danger-full-access` 策略
- preset 同步为幂等内容比对：无变化不写入；升级时包内变更文件覆盖目标；不删除目标目录中包内没有的文件
- 卸载 bundle 不会自动删除已安装的 preset 文件，避免误删用户修改

## 致谢与来源

本仓库是衍生作品，内容来源如下：

| 项目 | 作者 | 许可 | 用途 |
|---|---|---|---|
| [SheberDavid/v4-flash-godmode-opencode-go](https://github.com/SheberDavid/v4-flash-godmode-opencode-go) | SheberDavid | 无独立 LICENSE（其 README 声明基于 MIT 项目） | `preset/` 本体与 dsh rc.6 适配逻辑的直接来源 |
| [dsh-router-standard](https://github.com/yjh051108/dsh-router-standard) | yjh051108 | MIT（© 2026 yjh051108） | Flash w7 persona、深度思考锚与路由研究的源头 |
| [dsh-routing-suite](https://github.com/yjh051108/dsh-routing-suite) | yjh051108 | MIT | 路由研究 |
| [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) | DeepSeek | MIT | 宿主框架 |

本仓库新增的 bundle 包装层（`package.json` / `cordis.patch.yml` / `lib/`）为本仓库独立新增代码。

完整许可与衍生链说明见 [LICENSE](./LICENSE) 与 [NOTICE](./NOTICE)。DeepSeek 是 DeepSeek 公司商标；本项目为社区产物，与 DeepSeek 无关联。
