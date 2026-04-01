# Claude Code 项目手册（恢复版）

> 版本：999.0.0-restored
> 路径：`e:\VSCode project\claude-code-rev`

## 1. 项目概貌

- 项目名：`@anthropic-ai/claude-code`（恢复版，`999.0.0-restored`）
- 类型：CLI + 智能助理（本地 / 远程 / agent / MCP）
- 平台：Bun（推荐）+ Node.js 24+
- 运行入口：`src/bootstrap-entry.ts` -> `src/entrypoints/cli.tsx` -> `src/main.tsx`

## 2. 快速运行（命令）

- `bun install`（依赖安装）
- `bun run version`（版本输出，复原版本）
- `bun run dev` / `bun run start`（本地 CLI 运行）
- `bun run dev --help`（命令帮助）
- `bun run dev:restore-check`（检查缺失相对依赖）

## 3. 关键启动流程

1. `package.json` 脚本指向
   - `src/bootstrap-entry.ts`
2. `bootstrap-entry.ts`
   - `ensureBootstrapMacro()`
   - `import('./entrypoints/cli.tsx')`
3. `entrypoints/cli.tsx`
   - 快速命令路由（仅导入最小模块）：
     - `--version/-v`
     - `--dump-system-prompt`
     - `--claude-in-chrome-mcp`、`--chrome-native-host`
     - `--computer-use-mcp`（`CHICAGO_MCP`）
     - `remote-control` / `daemon` / `ps/logs/attach/kill`
     - `new`/`list`/`reply`（模板）
     - `environment-runner` / `self-hosted-runner`
   - 其余转到 `main.tsx`
4. `main.tsx`
   - 入口逻辑最全：配置/插件/权限/会话/远程/API/交互/模型

## 4. 目录结构与核心模块

### 4.1 根目录

- `README.md`：项目说明、恢复说明、运行命令
- `shims/`：实用替代包（Chrome MCP、Computer Use、NAPI/本地依赖）
- `AGENTS.md`：贡献规范（非SDK源码）

### 4.2 核心子目录（按功能）

- `src/`
  - `bootstrap/`：启动状态、后续各模块共享状态
  - `commands/`：命令实现（subcommand）
  - `entrypoints/cli.tsx`：整体 CLI 快速路径
  - `main.tsx`：CLI 主循环、初始化、交互
  - `tools/`：Tool 组件集合（AgentTool、BashTool、WebSearch 等）
  - `skills/`：捆绑技能（bundled）
  - `services/`：API/mcp/策略/remote/plugin/lsp/limits
  - `bridge/`：桥接远程（remote-control）
  - `assistant/`：assistant 会话、人机对话策略
  - `utils/`：权限/配置/会话/模型/日志/远程/并发
  - `state/`：应用状态管理（AppStateStore、store 等）
  - `tasks/`：任务、异步运行、后台
  - `proactive/`：主动调度/监控相关

### 4.3 主要功能模块

- `tools.ts`：工具注册 `getAllBaseTools()` + preset
- `Tool.ts`：通用 Tool 类型、权限入口
- `commands.ts`：命令集（`getCommands`、远程过滤）
- `permissions/`：权限模式、自动模式、黑白名单（`utils/permissions`）
- `mcp/`：多服务器、项目内 MCP 配置、资源
- `plugins/`：插件发现/安装/缓存/启用
- `remote/`：远程会话（Teleport、DirectConnect）
- `bridge/`：本机桥接代理（`remote-control`）
- `daemon/`：后台 worker、守护进程

## 5. 重点功能说明

### 5.1 快速路径（`src/entrypoints/cli.tsx`）

- 版本路径、dump prompt、MCP、bridge、daemon、模板、环境运行。
- 具体命令首次命中即返回，减少模块加载。

### 5.2 工具框架（`src/tools.ts` + `src/tools/**`）

- `getAllBaseTools()`：核心工具列表；
- `getToolsForDefaultPreset()`：启用工具集合
- feature flag 控制：`PROACTIVE`, `KAIROS`, `AGENT_TRIGGERS`, `MCP`，及环境 `USER_TYPE`
- 特别说明：某些用户、内部渠道仅 ant 可用（`REPLTool`, `ConfigTool`）

### 5.3 MCP 和远程

- `src/services/mcp/**`：MCP 配置解析、策略 filter
- `src/tools/ListMcpResourcesTool` / `ReadMcpResourceTool`：MCP 数据查询
- `src/utils/claudeInChrome/`：Chrome MCP 集成提示、授权

### 5.4 状态与会话

- `bootstrap/state.ts`：全局 boot 状态（model/agent/teleport）
- `state/AppStateStore.tsx` + `state/store.ts`：UI 与命令状态
- `utils/sessionStorage.js`：resume/load() + 目录结构

### 5.5 权限/安全

- `utils/permissions/permissionSetup.js`：CLI 开始权限解析
- `utils/permissions/permissions.js`：具体工具允许/禁止
- `utils/permissions/autoModeState.js`：自动授权模式规则

## 6. 推荐开发流程

1. `bun install`
2. `bun run dev --help`（确认可运行）
3. 修改目标模块：
   - CLI：`src/entrypoints/cli.tsx` 或 `src/main.tsx`
   - 命令：`src/commands/**`
   - 工具/能力：`src/tools/**`
   - 插件：`src/plugins/**`
4. 手工验证命令：
   - `bun run dev <command>`
   - `bun run dev --version`
   - 需要时以 `--claude-in-chrome-mcp` 等分支路径测试
5. 运行 `bun run dev:restore-check` 查漏
6. 最后 `git diff` / `bun run version` 验收

## 7. 关键诊断（故障排查）

- `missing_relative_imports`（`src/dev-entry.ts`）：说明源码引用文件缺失；先补路径或还原文件
- feature flag：`BG_SESSIONS` / `DAEMON` / `BRIDGE_MODE` 影响行为
- 插件/配置目录：`~/.claude`、环境变量 `CLAUDE_CODE_REMOTE`
- 常见 `Bun/Node` debug：`--inspect` / `--debug`

## 8. 扩展与二次开发建议

### 8.1 新命令模板

- 放 `src/commands/**`
- 注册 `getCommands`（`commands.ts`）

### 8.2 新 Tool

- 新 `tools/<YourTool>` 构建 `Tool` 类型
- 在 `src/tools.ts` `getAllBaseTools()` 添加
- 权限规则：`utils/permissions/permissions.js`

### 8.3 新 MCP 资源

- `src/services/mcp/config.js` + `service/mcp/client.js`
- 相关 `tools/ListMcpResourcesTool` / `ReadMcpResourceTool`

### 8.4 远控与桥集成

- `src/bridge/**` 或 `src/remote/**`
- 关键：认证 (`utils/auth.js`) -> 策略 (`services/policyLimits`) -> 运行循环

## 9. 推荐学习路径

1. `src/entrypoints/cli.tsx`
2. `src/main.tsx`
3. `src/tools.ts`、`Tool.ts`
4. `src/commands.ts` + `src/commands/**`
5. `src/bootstrap/state.ts` + `src/state/**`
6. `src/services/**`
7. `src/utils/permissions/**` + `src/utils/config.js`
8. `shims/**`
9. `README.md`

## 10. 小贴士

- 这个仓库是“可运行恢复版”，部分私有/原生功能仍依赖 shim。
- Windows 路径混合：项目中使用 `path.resolve`、`join`，注意条件兼容。
- 新功能建议先从 fast-path (CLI short-circuit) 改造，然后慢慢进 `main.tsx`。
