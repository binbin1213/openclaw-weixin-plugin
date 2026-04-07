错误通知机制是 OpenClaw 微信插件中的关键容错组件，负责在消息处理失败时向用户提供友好的错误提示，同时确保主流程不受影响。该机制采用 fire-and-forget 设计模式，将错误通知与主业务流程解耦，确保即使在通知发送失败的情况下，也不会影响正常的消息处理流程。

## 核心设计原则

错误通知机制的核心设计遵循三个关键原则：**非阻断性**、**上下文感知**和**错误分类**。非阻断性意味着通知失败永远不会抛出异常或影响调用方逻辑，所有错误仅被记录到日志中。上下文感知则通过 `contextToken` 确保错误消息能够回复到正确的对话上下文。错误分类则根据不同的错误类型提供差异化的用户提示，例如媒体下载失败、CDN 上传失败或通用错误等场景。

Sources: [src/messaging/error-notice.ts](src/messaging/error-notice.ts#L1-L31)

## 架构概览

错误通知机制的架构由三个层次组成：通知触发层、通知执行层和基础服务层。触发层位于消息处理流程中的错误回调中，当 AI 回复发送失败时被调用。执行层包含核心的 `sendWeixinErrorNotice` 函数，负责构建并发送错误通知消息。基础服务层提供日志记录和消息发送能力。

```mermaid
flowchart TB
    subgraph "消息处理流程"
        A[入站消息] --> B[消息处理]
        B --> C[AI 回复分发器]
        C --> D{发送成功?}
        D -->|是| E[完成]
        D -->|否| F[onError 回调]
    end
    
    subgraph "错误通知层"
        F --> G[错误分类判断]
        G -->|媒体下载失败| H[⚠️ 媒体文件下载失败<br/>请检查链接是否可访问]
        G -->|CDN 上传失败| I[⚠️ 媒体文件上传失败<br/>请稍后重试]
        G -->|其他错误| J[⚠️ 消息发送失败: {errMsg}]
    end
    
    subgraph "通知执行层"
        H --> K[sendWeixinErrorNotice]
        I --> K
        J --> K
        K --> L{contextToken 存在?}
        L -->|是| M[携带上下文发送]
        L -->|否| N[记录警告日志]
    end
    
    subgraph "基础服务层"
        M --> O[sendMessageWeixin API]
        O --> P{发送成功?}
        P -->|是| Q[记录调试日志]
        P -->|否| R[记录错误日志<br/>errLog 回调]
        N --> R
    end
    
    R --> S[主流程继续执行]
```

Sources: [src/messaging/process-message.ts](src/messaging/process-message.ts#L358-L408)

## 错误类型与通知映射

系统根据错误消息的特征自动分类，并映射到相应的用户友好提示。这种设计避免了向用户暴露技术细节，同时提供了可操作的错误信息。

| 错误特征 | 触发条件 | 用户提示 | 典型场景 |
|---------|---------|---------|---------|
| 媒体下载失败 | `errMsg` 包含 "remote media download failed" 或 "fetch" | ⚠️ 媒体文件下载失败，请检查链接是否可访问。 | AI 生成的图片 URL 无法访问或超时 |
| CDN 上传失败 | `errMsg` 包含 "getUploadUrl"、"CDN upload" 或 "upload_param" | ⚠️ 媒体文件上传失败，请稍后重试。 | CDN 服务暂时不可用或文件过大 |
| 通用错误 | 其他所有错误类型 | ⚠️ 消息发送失败：{具体错误信息} | 网络异常、API 调用失败等未知错误 |

Sources: [src/messaging/process-message.ts](src/messaging/process-message.ts#L360-L376)

## 通知函数实现

`sendWeixinErrorNotice` 是错误通知的核心函数，接收包含目标用户、上下文令牌、错误消息等参数。该函数内部采用 try-catch 保护，确保即使在发送失败时也不会影响调用方。当 `contextToken` 缺失时，函数仅记录警告日志而不发送通知，因为没有对话上下文可以回复。

函数通过调用底层的 `sendMessageWeixin` API 发送纯文本错误消息，发送成功或失败都会记录相应的日志。这种双重日志策略（debug 级别记录成功，error 回调记录失败）为问题排查提供了完整的审计轨迹。

Sources: [src/messaging/error-notice.ts](src/messaging/error-notice.ts#L1-L31)

## 集成到消息处理流程

错误通知机制深度集成到消息处理流程中，通过回复分发器的 `onError` 回调触发。当 `createReplyDispatcherWithTyping` 创建回复分发器时，会注册 `onError` 处理函数，该函数在 `deliver` 回调抛出异常时被调用。

在错误处理回调中，系统首先通过错误消息特征判断错误类型，然后构建相应的用户提示文本，最后调用 `sendWeixinErrorNotice` 发送通知。整个过程是异步的（使用 `void` 忽略 Promise），确保不会阻塞主流程的执行。

Sources: [src/messaging/process-message.ts](src/messaging/process-message.ts#L316-L408)

## 上下文令牌处理

上下文令牌（`contextToken`）是错误通知机制中的关键参数，它决定了错误消息能否正确地回复到用户所在的对话上下文。在消息入站阶段，系统会从原始消息中提取 `contextToken` 并存储到内存缓存中。在发送错误通知时，会从消息上下文中获取该令牌并传递给消息发送 API。

如果 `contextToken` 缺失（例如在某些异常场景下），系统不会中断流程，而是记录警告日志并继续执行。这种优雅降级策略确保了即使在上下文信息不完整的情况下，系统也能保持稳定运行。

Sources: [src/messaging/error-notice.ts](src/messaging/error-notice.ts#L10-L14)

## 日志记录策略

错误通知机制与结构化日志系统紧密集成，遵循分层日志记录原则。调试级别日志记录通知发送成功的事件，用于问题排查时的状态跟踪。错误级别日志通过 `errLog` 回调记录通知发送失败的情况，确保运维人员能及时发现通知系统本身的问题。

日志记录器采用 JSON 格式输出，包含时间戳、日志级别、子系统标识（`gateway/channels/openclaw-weixin`）和账号信息等元数据，便于日志聚合和分析工具进行统一处理。

Sources: [src/util/logger.ts](src/util/logger.ts#L1-L146)

## 与其他模块的交互

错误通知机制作为消息处理流程的容错组件，与多个核心模块协作。与消息发送模块（`sendMessageWeixin`）的交互是最直接的，错误通知复用了常规消息发送的 API，确保通知消息的格式和编码与正常消息保持一致。

与监控循环模块（`monitor.ts`）的交互体现在错误回调的传递上，监控循环将 `errLog` 回调注入到消息处理依赖中，最终传递到错误通知函数。这种依赖注入模式使得错误日志能够统一记录到监控层面。

与调试模式模块的交互体现在错误通知不干扰调试追踪信息的发送。当调试模式启用时，系统会在消息处理完成后发送详细的时序追踪信息，而错误通知则作为独立的流程在错误发生时立即发送。

Sources: [src/monitor/monitor.ts](src/monitor/monitor.ts#L147-L192)

## 错误边界处理

错误通知机制本身也包含完善的错误边界处理。在 `sendWeixinErrorNotice` 函数内部，所有发送操作都被 try-catch 包裹，捕获任何异常并通过 `errLog` 回调记录，然后静默返回。这种自我保护机制确保了即使消息发送 API 出现严重故障，也不会导致整个插件崩溃。

在更上层的消息处理流程中，`deliver` 回调抛出的异常会触发 `onError` 处理，但异常不会向上传播，而是被分发器内部消化。这种设计遵循了"错误局部化"原则，将错误的影响范围限制在单个消息处理周期内。

Sources: [src/messaging/process-message.ts](src/messaging/process-message.ts#L358-L408)

## 下一步阅读

要深入理解错误通知机制的上下文，建议按照以下顺序阅读相关文档：

- [长轮询监控循环实现](26-chang-lun-xun-jian-kong-xun-huan-shi-xian) - 了解错误回调的来源和监控架构
- [消息发送 sendMessage API](11-xiao-xi-fa-song-sendmessage-api) - 深入了解底层消息发送机制
- [结构化日志系统](27-jie-gou-hua-ri-zhi-xi-tong) - 掌握日志记录和问题排查方法
- [调试模式与链路追踪](21-diao-shi-mo-shi-yu-lian-lu-zhui-zong) - 了解如何使用调试模式诊断问题