状态目录是 OpenClaw 微信插件的持久化存储中心，负责统一管理账号凭证、会话状态、同步游标等关键数据。通过集中化的目录结构，插件能够在网关重启后恢复完整的运行上下文，确保消息处理的连续性和会话状态的持久性。

## 核心解析机制

状态目录的路径解析由 `resolveStateDir()` 函数实现，采用三级优先级查找策略：优先读取 `OPENCLAW_STATE_DIR` 环境变量，其次尝试 `CLAWDBOT_STATE_DIR`，最终回退到用户主目录下的 `.openclaw` 目录。这种设计既支持自定义存储位置，又保证了默认环境下的可用性。

Sources: [src/storage/state-dir.ts](src/storage/state-dir.ts#L3-L11)

## 目录结构详解

状态目录采用分层设计，将不同类型的数据隔离在独立的子目录中，既便于管理又避免了文件冲突。完整的目录结构如下：

```mermaid
graph TD
    A[~/.openclaw<br/>状态根目录] --> B[credentials/<br/>框架凭证目录]
    A --> C[openclaw-weixin/<br/>插件主目录]
    A --> D[openclaw.json<br/>全局配置文件]
    A --> E[agents/<br/>遗留单账号路径]
    
    B --> B1[openclaw-weixin-{accountId}<br/>-allowFrom.json]
    
    C --> C1[accounts.json<br/>账号索引文件]
    C --> C2[debug-mode.json<br/>调试模式状态]
    C --> C3[accounts/<br/>账号数据目录]
    
    C3 --> C3A[{accountId}.json<br/>账号凭证]
    C3 --> C3B[{accountId}.sync.json<br/>同步游标]
    C3 --> C3C[{accountId}.context-tokens.json<br/>上下文令牌]
    
    E --> E1[default/sessions/<br/>.openclaw-weixin-sync/]
    E1 --> E1A[default.json<br/>旧版同步游标]
```

### 目录职责划分

| 目录/文件 | 路径 | 数据类型 | 生命周期 | 主要用途 |
|----------|------|----------|----------|----------|
| **credentials/** | `credentials/` | 框架级 | 跨插件共享 | 存储框架授权白名单 |
| **openclaw-weixin/** | `openclaw-weixin/` | 插件级 | 插件专属 | 微信插件所有持久化数据 |
| **accounts/** | `openclaw-weixin/accounts/` | 账号级 | 多账号隔离 | 每个账号的独立数据 |
| **agents/** | `agents/` | 遗留级 | 向后兼容 | 旧版单账号安装的迁移路径 |

Sources: [src/auth/accounts.ts](src/auth/accounts.ts#L35-L38), [src/auth/pairing.ts](src/auth/pairing.ts#L12-L16), [src/storage/sync-buf.ts](src/storage/sync-buf.ts#L7-L11)

## 文件用途说明

### 账号索引文件

`accounts.json` 文件维护已注册账号 ID 的列表，用于插件启动时加载所有可用账号。每当新账号通过扫码登录成功后，其标准化后的 `accountId` 会被追加到该文件中；账号注销时则从列表中移除。该文件采用 JSON 数组格式，每个元素为字符串类型的账号 ID。

Sources: [src/auth/accounts.ts](src/auth/accounts.ts#L40-L64)

### 账号凭证文件

每个账号的凭证存储在 `{accountId}.json` 文件中，包含 `token`（访问令牌）、`baseUrl`（网关地址）、`userId`（绑定的微信用户 ID）以及 `savedAt`（保存时间戳）。文件采用原子写入模式，并设置了严格的权限控制（0o600），确保敏感信息的安全。当同一用户重复扫码登录时，旧账号会被清理以避免上下文令牌匹配的歧义。

Sources: [src/auth/accounts.ts](src/auth/accounts.ts#L66-L214)

### 同步游标文件

`{accountId}.sync.json` 文件持久化 `get_updates_buf` 参数，这是长轮询 API 的关键游标，记录了最后一次拉取的消息位置。通过保存该游标，网关重启后能够从断点继续拉取新消息，避免重复处理或消息丢失。该文件的数据结构为 `{ "get_updates_buf": "string" }`。

Sources: [src/storage/sync-buf.ts](src/storage/sync-buf.ts#L13-L58)

### 上下文令牌文件

`{accountId}.context-tokens.json` 文件存储每个用户与 bot 交互的 `contextToken`，这是微信 API 要求每次发送消息时必须回传的上下文标识。令牌采用内存+磁盘双缓存机制：内存 Map 提供快速查询，磁盘文件确保进程重启后的恢复能力。文件格式为键值对，键为用户 ID，值为对应的 contextToken。

Sources: [src/messaging/inbound.ts](src/messaging/inbound.ts#L28-L76)

### 调试模式文件

`debug-mode.json` 文件控制每个账号的调试开关状态，格式为 `{ "accounts": { "{accountId}": true/false } }`。当调试模式启用时，插件会在每条 AI 回复后附加详细的处理时序信息，用于性能分析和链路追踪。该状态的持久化使得调试设置在网关重启后仍然生效。

Sources: [src/messaging/debug-mode.ts](src/messaging/debug-mode.ts#L1-L70)

### 授权白名单文件

`openclaw-weixin-{accountId}-allowFrom.json` 文件存储通过配对授权的用户 ID 白名单，由框架授权管道直接读取。文件结构包含版本号和授权用户列表：`{ "version": 1, "allowFrom": ["userId1", "userId2"] }`。写入操作采用文件锁机制，避免并发访问导致的数据竞争。

Sources: [src/auth/pairing.ts](src/auth/pairing.ts#L32-L121)

### 全局配置文件

`openclaw.json` 文件位于状态根目录，是 OpenClaw 框架的配置中心。微信插件在该文件的 `channels.openclaw-weixin` 段落下维护 `routeTag`（路由标签）和 `channelConfigUpdatedAt`（配置更新时间戳）等配置项。每次扫码登录成功后，插件会更新 `channelConfigUpdatedAt` 字段，触发网关重新加载配置。

Sources: [src/auth/accounts.ts](src/auth/accounts.ts#L243-L317)

## 向后兼容性处理

插件在多个层面实现了向后兼容机制，确保旧版本安装能够平滑升级到新的多账号架构。

### 账号 ID 兼容

早期版本使用原始 ID 格式（如 `b0f5860fdecb@im.bot`），新版本采用标准化 ID（如 `b0f5860fdecb-im-bot`）。加载账号数据时，插件会先尝试新格式文件名，如果不存在则根据后缀模式反推旧文件名进行回退读取。这种双路径策略避免了升级过程中的数据丢失。

Sources: [src/auth/accounts.ts](src/auth/accounts.ts#L10-L28), [src/auth/accounts.ts](src/auth/accounts.ts#L148-L169)

### 遗留单账号路径

在引入多账号支持之前，同步游标存储在 `agents/default/sessions/.openclaw-weixin-sync/default.json`。插件在加载游标时会依次尝试新路径、兼容路径和遗留路径，确保老安装的配置能够被正确迁移。凭证文件也有类似的遗留路径处理机制。

Sources: [src/storage/sync-buf.ts](src/storage/sync-buf.ts#L24-L37)

### 旧版凭证迁移

早期的单账号凭证存储在 `credentials/openclaw-weixin/credentials.json`。插件在读取新格式凭证失败时，会尝试从这个遗留位置加载 token，并在首次登录时自动迁移到新的多账号结构中。

Sources: [src/auth/accounts.ts](src/auth/accounts.ts#L128-L142)

## 环境变量配置

状态目录的位置可以通过环境变量进行自定义，提供了部署灵活性：

- **OPENCLAW_STATE_DIR**：框架标准的环境变量，优先级最高
- **CLAWDBOT_STATE_DIR**：历史兼容的环境变量，当标准变量未设置时生效

当这两个变量都未设置时，系统会使用用户主目录下的 `.openclaw` 作为默认路径。这种设计既支持开发测试环境中的自定义目录，又保证了生产环境的开箱即用体验。

Sources: [src/storage/state-dir.ts](src/storage/state-dir.ts#L3-L11)

## 账号清理机制

当账号被注销时，插件会执行完整的清理操作，删除该账号关联的所有文件：

- 凭证文件 `{accountId}.json`
- 同步游标文件 `{accountId}.sync.json`
- 上下文令牌文件 `{accountId}.context-tokens.json`
- 授权白名单文件 `openclaw-weixin-{accountId}-allowFrom.json`

同时，账号 ID 也会从 `accounts.json` 索引文件中移除，确保插件不再尝试加载该账号。

Sources: [src/auth/accounts.ts](src/auth/accounts.ts#L216-L235)

## 相关文档

状态目录是插件持久化层的基础设施，其数据结构直接影响其他功能模块的运行。建议继续阅读以下文档以深入理解各持久化文件的具体用途：

- [同步游标持久化](24-tong-bu-you-biao-chi-jiu-hua) - 了解 `get_updates_buf` 的生命周期和恢复机制
- [上下文令牌缓存与恢复](25-shang-xia-wen-ling-pai-huan-cun-yu-hui-fu) - 掌握 contextToken 的内存+磁盘双缓存策略
- [账号存储与管理](8-zhang-hao-cun-chu-yu-guan-li) - 学习多账号隔离的实现细节