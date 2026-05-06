# OpenViking OpenCode 插件

一个统一的 OpenCode 插件，用于 OpenViking 仓库检索与长期记忆管理。

该 PR 在旧版拆分示例旁新增了一个统一插件包。旧示例目前仍然保留，后续会下线：

- `examples/opencode`：已索引仓库的提示词注入，以及面向 CLI 的使用引导
- `examples/opencode-memory-plugin`：长期记忆、session 同步、commit 与 recall

新插件通过 OpenCode 的 tool hooks 暴露所有能力，并通过 HTTP API 与 OpenViking 通信。它不会安装或要求使用 OpenCode skill，agent 也不需要运行 `ov` shell 命令。

## 功能概览

- 将已索引的 `viking://resources/` 仓库注入 system prompt。
- 以工具形式暴露仓库 search、grep、glob、read、browse、add、remove 与 queue status 能力。
- 将每个 OpenCode session 映射到一个 OpenViking session。
- 将用户与 assistant 的文本消息捕获到 OpenViking。
- 在生命周期边界提交 session，用于记忆抽取。
- 自动召回相关记忆，并将其追加到最新用户消息中。

## 文件结构

```text
examples/opencode-plugin/
├── index.mjs
├── package.json
├── README.md
├── INSTALL-ZH.md
├── lib/
│   ├── runtime.mjs
│   ├── repo-context.mjs
│   ├── memory-session.mjs
│   ├── memadd-local.mjs
│   ├── memory-tools.mjs
│   ├── memory-recall.mjs
│   └── utils.mjs
└── wrappers/
    └── openviking.mjs
```

该结构中有意不再包含 `skills/openviking/SKILL.md`。原 skill 的行为已经通过工具实现。

## 前置要求

- OpenCode
- OpenViking HTTP Server
- Node.js / npm，用于安装插件依赖
- 如果 OpenViking 服务端启用了认证，需要一个 OpenViking API key

请先启动 OpenViking：

```bash
openviking-server --config ~/.openviking/ov.conf
```

## 安装

### 发布包安装

当该插件发布为 npm 包后，普通用户应通过 OpenCode 的 package plugin 机制启用：

```json
{
  "plugin": ["openviking-opencode-plugin"]
}
```

如果正式发布前包名发生变化，请使用最终发布包名。

### 源码安装

用于开发或 PR 测试时，可将插件包复制到 OpenCode 的插件目录，并使用顶层 wrapper：

```bash
mkdir -p ~/.config/opencode/plugins/openviking
cp examples/opencode-plugin/wrappers/openviking.mjs ~/.config/opencode/plugins/openviking.mjs
cp examples/opencode-plugin/index.mjs examples/opencode-plugin/package.json ~/.config/opencode/plugins/openviking/
cp -r examples/opencode-plugin/lib ~/.config/opencode/plugins/openviking/
cd ~/.config/opencode/plugins/openviking
npm install
```

这样会生成一个稳定的 OpenCode 插件布局：

```text
~/.config/opencode/plugins/
├── openviking.mjs
└── openviking/
    ├── index.mjs
    ├── package.json
    ├── lib/
    └── node_modules/
```

顶层 `openviking.mjs` 只是一个 wrapper：

```js
export { OpenVikingPlugin, default } from "./openviking/index.mjs"
```

这个 wrapper 只用于上面这种源码安装目录结构。npm 包安装会通过 `package.json` 直接加载 `index.mjs`。

## 配置

创建 `~/.config/opencode/openviking-config.json`：

```json
{
  "endpoint": "http://localhost:1933",
  "apiKey": "",
  "account": "",
  "user": "",
  "agentId": "",
  "enabled": true,
  "timeoutMs": 30000,
  "repoContext": { "enabled": true, "cacheTtlMs": 60000 },
  "autoRecall": {
    "enabled": true,
    "limit": 6,
    "scoreThreshold": 0.15,
    "maxContentChars": 500,
    "preferAbstract": true,
    "tokenBudget": 2000
  }
}
```

`apiKey` 会作为 `X-API-Key` 发送。`account`、`user` 和 `agentId` 分别会作为 `X-OpenViking-Account`、`X-OpenViking-User` 和 `X-OpenViking-Agent` 发送。

对于多租户 OpenViking 服务端，租户级 API 通常需要这些字段。

`OPENVIKING_API_KEY`、`OPENVIKING_ACCOUNT`、`OPENVIKING_USER` 和 `OPENVIKING_AGENT_ID` 的优先级高于该配置文件中的值。

高级场景下，可以使用 `OPENVIKING_PLUGIN_CONFIG` 指向其他配置文件路径。

## 工具

### `memsearch`

在 memories、resources 和 skills 中进行语义搜索。

适用于概念性问题、仓库内部实现、用户偏好以及上下文感知检索。可以使用 `target_uri` 缩小范围，例如 `viking://resources/fastapi/`。

### `memread`

读取指定的 `viking://` URI，支持 `abstract`、`overview`、`read` 或 `auto`。

通常在 `memsearch`、`memgrep`、`memglob` 或 `membrowse` 返回 URI 后使用。

### `membrowse`

使用 `list`、`tree` 或 `stat` 浏览 OpenViking 文件系统结构。

适用于在读取内容前发现精确 URI。

### `memcommit`

将当前 OpenCode session 提交到 OpenViking，并触发记忆抽取。

插件也会在 session 删除、session 错误、上下文压缩和插件关闭等生命周期边界提交。

### `memgrep`

在 OpenViking 内容中进行模式搜索。

适用于精确符号、类名、函数名、错误字符串或已知关键词。

### `memglob`

在 OpenViking 内容中进行 glob 文件匹配。

适用于枚举文件，例如 `**/*.py`、`**/test_*.ts` 或 `**/*.md`。

### `memadd`

向 OpenViking 添加远程 URL 或本地文件资源。

远程 `http(s)` URL 会直接通过 `POST /api/v1/resources` 添加。

本地文件使用更安全的两步服务端流程：先将文件上传到 `POST /api/v1/resources/temp_upload`，再使用返回的 `temp_file_id` 通过 `POST /api/v1/resources` 添加。

本地路径可以是绝对路径、相对于 OpenCode 项目目录的路径，或 `file://` URL。当前暂不支持本地目录上传。

示例：

```text
memadd path="https://example.com/spec.md" to="viking://resources/spec"
memadd path="./docs/notes.md" parent="viking://resources/"
memadd path="file:///home/alice/project/notes.md" reason="project notes"
```

添加资源后，该工具也会返回 `GET /api/v1/observer/queue` 的状态。

### `memremove`

通过 `DELETE /api/v1/fs` 删除一个 `viking://` URI。

该工具要求 `confirm: true`。agent 调用前必须先获得用户的明确删除确认。

### `memqueue`

返回 OpenViking observer queue 状态，用于查看 embedding 与语义处理队列。

## 运行时文件

插件默认会将运行时文件写入 `~/.config/opencode/openviking/`：

- `openviking-memory.log`
- `openviking-session-map.json`

可以在配置中设置 `runtime.dataDir` 覆盖该目录。
