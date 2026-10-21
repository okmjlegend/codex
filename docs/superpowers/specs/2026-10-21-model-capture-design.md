# 模型请求与可见思考流保存设计

基线：2026-10-21，源码提交 `ab6095dde0`。状态：待评审方案，尚未实现。

## 目标与验收边界

保存实际提交给模型接口的请求 payload、接口返回的流式事件，以及后端准备交付给 App 的可见推理内容；正常完成和主动取消后均可离线查询，并关联到会话、轮次及具体发送尝试。

“已记录”“已落盘”“已发往 App”和“App 已显示”是不同事实。客户端渲染结果通过手工验收确认；后端记录不得声称证明 App 已显示。

范围包含 Responses HTTP/SSE、Responses WebSocket、预热、HTTP 内部重试、鉴权恢复、WebSocket 到 HTTP 回退。范围不包含其他模型协议、实时语音、图片上传接口、TLS 报文或服务端未返回的内部推理。历史缺失数据不补造。

## 方案选择

1. 推荐：扩展现有 `rollout-trace`，增加请求发送与流式采集观察器。复用关联 ID、bundle 和 reducer，补齐现有完整响应项记录的缺口。
2. 只扩展会话历史：可以保留思考增量，但难以准确关联底层重试与请求字节，不满足本次完整目标。
3. 外置代理：需要处理 WebSocket、压缩、路由和证书，不作为本项目首选。

## 现有源码依据

- `core/src/client.rs`：HTTP/WS 调用 `InferenceTraceAttempt::record_started`；WS 的预热复用场景可能记录逻辑完整请求。
- `rollout-trace/src/inference.rs`：记录完整输出项，并恢复常规序列化可能省略的 reasoning content；不保存所有 delta。
- `codex-api/src/sse/responses.rs`：解析 reasoning summary/text 增量。
- `rollout/src/policy.rs`：普通持久化过滤 reasoning delta。
- `http-client/src/request.rs`：支持最终压缩 body 的准备；最终字节与可读 JSON 必须分别标记。

## 组件与依赖

新增名称均为拟议接口，不代表已有功能。

- `http-client/src/capture.rs`：定义无文件系统依赖的 `HttpCaptureObserver` 及请求尝试快照，提供默认关闭实现。观察最终 body、结果和关联信息。
- `codex-api/src/capture.rs`：定义 `ResponsesCaptureObserver`，观察 SSE data、WS 文本消息及消息发送结果。
- `core/src/model_capture.rs`：把观察器连接到 trace sink，维护 inference/transport/response/item 的关联。
- `rollout-trace/src/capture.rs`：采集生命周期、队列、分段日志、错误和完整性元数据。
- `rollout-trace/src/reducer/capture.rs`：离线生成查询索引和按分段聚合的可读思考。

底层 crate 只依赖观察器契约；不依赖 `core` 或具体 trace writer，避免循环依赖。普通 OpenTelemetry 与此本地采集分离，不通过现有遥测接口上传正文。

## 采集点

### 请求

- 保留现有 `record_started` 作为逻辑请求证据，标记 `logical_request`。
- HTTP：在 Responses 请求路径的每次重试实际调用 transport 前建立 attempt，在最终 body 已准备的位置记录发送字节；可读 JSON 在压缩前记录，两个文件显式关联。记录 URL、方法、允许保留的头、压缩类型、字节数、摘要和发送结果。
- WS：在 `send_websocket_request` 消费 `request_text` 前保存该文本，每次发送单独编号；保留 `previous_response_id`，预热 `generate=false` 也记录。
- 失败发送只证明尝试发送，不能标记服务端已接收；HTTP 路由重定向若未逐跳观察，标记覆盖范围，不宣称包含每跳。

### 返回流

- SSE：在反序列化前记录 event 类型、ID 和 data 字符串。这是应用层 SSE 事件，不能标记为包含空行、注释、TCP 分块的原始网络字节。
- WS：在反序列化前保存接收到的应用层文本消息；不宣称保留 WebSocket 帧压缩或帧边界。
- 保留未知事件和无法解析的文本，并记录解析失败。连接关闭、流错误和取消独立记录。
- 所有响应事件纳入采集，因此正常回复和工具调用也可与推理关联。

### App 可见内容

- 在 core 对事件进行交付前记录 reasoning summary/content 增量、段落分隔和完整推理项，并在 app-server 映射后的通知边界记录实际准备发往 App 的推理通知。
- 原始流与 App 通知用不同来源标记，避免摘要延迟或过滤造成混淆。
- 如当前通知缺少 inference ID，通过内部运行时映射关联，不改变对外 App 协议；无法唯一关联时保留未知状态。
- App 通知发送成功表示交付层接受，不代表渲染确认。

## 存储与字段

沿用 `CODEX_ROLLOUT_TRACE_ROOT` 的显式启用方式，不在分析阶段改动用户环境。增强采集只作用于启用后初始化的运行时会话。

```text
trace-<trace_id>-<thread_id>/
  manifest.json
  trace.jsonl
  payloads/*.json
  capture/
    requests/<transport_attempt_id>.body
    requests/<transport_attempt_id>.json
    streams/<stream_id>-<segment>.jsonl
    app-events/<thread_id>-<segment>.jsonl
    status.json
```

事件公共字段：`schema_version`、`trace_id`、`thread_id`、`turn_id`、`inference_call_id`、`transport_attempt_id`、`connection_id`、`response_id`、`item_id`、`source`、`event_type`、`seq`、`observed_at`、`payload_ref`。未观察到的关联字段为空，不生成猜测值。

`seq` 是本地观察顺序，`observed_at` 是本机时间；不把跨连接并发事件的本地顺序解释为服务端因果顺序。每个分段保留字节数、内容摘要和引用范围。

分别记录：执行状态 `completed/cancelled/failed/unknown`，采集状态 `complete/partial/not_captured`，持久化状态 `queued/flushed/synced/failed`。`complete` 指该采集范围内、观察到的全部事件均保存；取消后仍可完成保存，但模型输出本身是未完成状态。

## 聚合规则

- 依据 `(transport_attempt_id, response_id, item_id, reasoning_kind, part_index)` 组织文本，按 source 内序号拼接。
- 摘要与 reasoning text 分开；App 通知视图单独聚合，不把原始流和 App 流拼接到一起。
- `.done` 是该段快照，不能再次追加到已有 delta。流式文本与完整项存在差异时保留两份证据并标记。
- 不按文本值去重；相同文本可能是合法连续输出。不同 attempt 不能合并。
- 取消保留收到的全部片段，不等待完整项；崩溃后恢复可解析的日志前缀，未见终态则标记 `unknown/partial`。
- 加密 reasoning 原样作为 opaque 字段保存，不尝试解密或将其伪装为可读思考。

## 写入、故障与资源约束

使用独立写线程和有界队列，不在流式消费线程同步逐 token 写文件。队列按占用字节预算，不只按事件数量；请求大 body 使用共享字节引用，禁用采集时不复制正文。

建议初始默认：队列待写预算 16 MiB，流分段目标 16 MiB，每个 bundle 采集额度 1 GiB，批量刷新间隔 100 ms。这些是拟议配置值，不是已测性能指标。单个 payload 无法纳入预算时明确停止相关采集并标记 partial，不静默截断。

在完成、取消和正常退出时执行有超时的 drain 与 flush；正常关闭执行同步落盘。异常断电可能损失尚未同步的数据，不承诺零丢失。

队列满、磁盘满、写入错误或额度耗尽时不终止模型请求：记录缺失起点/原因、将采集状态标记 partial、输出明确诊断，并通过独立的小型状态通道尽可能保存故障状态；磁盘完全不可写时至少保留内存状态和 stderr 错误。此时不能声称所有 App 内容已持久化。严格的“落盘后才交付”模式不纳入首版。

凭证头不落盘；请求 body 默认按用户授权保留，可能包含代码、提示词和个人数据。目录权限 0700、文件权限 0600；复用目录时验证权限。首版不自动删除已有 bundle，额度耗尽后停止新增采集并告警。

## 兼容与回滚

现有 App 协议保持不变。新 bundle 使用版本化扩展，旧 bundle 仍可读取；旧 reader 遇到新版本明确报告不支持。新增采集文件通过独立引用挂接，避免高频事件大量污染主 trace。

关闭环境开关并重启后端可停止未来采集；必要时切回原二进制。已有文件保留供复查。部署前独立确认 App 实际加载的可执行文件，避免只验证 shell 中的 codex。

## 验收

1. HTTP 普通请求、zstd 请求：可读 JSON 与实际发送体一致；压缩体解压后匹配，摘要与长度可核验。
2. WS 首次、增量、预热、取消消息：记录文本与 mock 服务实际收到的文本逐字一致；逻辑请求单独存在。
3. SSE/WS 推理流：包含中文、跨网络分块字符、相同连续片段、多段、summary/text 并存、未知事件和非法 JSON。
4. delta/done/完整项：聚合不重复；主动取消保留此前全部收到的片段。
5. HTTP 重试、鉴权恢复、WS 回退、并发子线程：各 attempt 独立、关联不串线。
6. 队列满、磁盘写失败、额度耗尽和进程强杀：主链路行为符合约定，采集完整性不能误标 complete。
7. 禁用采集：不创建 capture 文件，无正文复制与额外文件写入；新旧 bundle 回放兼容。
8. 构建完整本地包并确认 App 实际加载，再手工对照思考显示、主动取消及保存结果；API mock 通过不能代替此验收。
