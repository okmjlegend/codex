# 模型请求与可见思考流保存实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 保存实际请求 payload 和返回的推理流，支持中断保留、按调用关联与离线查询。

**Architecture:** 扩展现有 rollout-trace；底层 HTTP/Responses 层通过可选观察器报告实际发送与接收，core 负责关联和适配，trace 独立写入与归约。App 交付事件单独记录，保持现有客户端协议。

**Tech Stack:** Rust，现有 HTTP/SSE/WebSocket、serde、JSONL 和本地文件系统。

**Spec:** [设计方案](../specs/2026-10-21-model-capture-design.md)

状态：待评审，未执行。基线提交 `ab6095dde0`；当前 Cargo.toml/Cargo.lock 已有用户改动，实施前必须保留并核对来源。

## Global Constraints

- 仅通过既有 `CODEX_ROLLOUT_TRACE_ROOT` 显式启用；关闭时不复制正文、不写采集文件。
- 不改变 App 对外协议，不把本地采集接入遥测上传。
- 执行终态、采集完整性和持久化状态分开记录；未观察的字段为空。
- 默认队列预算 16 MiB、流分段目标 16 MiB、bundle 额度 1 GiB、批量刷新间隔 100 ms；参数必须可测试注入。
- 目录 0700、文件 0600；凭证头不写入；错误不静默丢弃。
- 正常完成/取消/退出 drain；异常断电不承诺零丢失。

## Review Focus

- summary done 与 delta 重复：只替换同段快照，不重复追加。
- 预热与增量请求：实际文本和逻辑完整请求分开留存。
- 重试与并发连接：相同文本不去重，不同 attempt 不混合。
- 写入故障与强杀：保留可解析前缀，不能误标 complete。
- App 验收：必须确认加载的是新包，mock 不能证明实际显示和保存一致。

## Task 1: 定义采集契约并实现存储

**Files:** 新增 `codex-rs/http-client/src/capture.rs`、`codex-rs/codex-api/src/capture.rs`、`codex-rs/rollout-trace/src/capture.rs`；修改对应 `lib.rs`、`rollout-trace/src/writer.rs`、`bundle.rs`、`payload.rs`、`raw_event.rs`。

**Interfaces:** 提议类型 `HttpCaptureObserver`、`ResponsesCaptureObserver`、`CaptureEvent`、`CaptureContext`、`CaptureStatus`。具体 Rust 签名在实施时统一定义于契约文件，观察器只能提交不可变快照，不直接拥有文件。sink 提供 `try_record`、有超时的 `flush` 和只读 `status`。

- [ ] 先写失败测试：关闭不写文件；请求二进制原样保留；JSONL 序号有序；目录/文件权限正确；队列满和写失败标记 partial。
- [ ] 实现共享字节正文、独立写线程、有界字节预算、日志分段及状态文件；控制事件保留独立通道。
- [ ] 实现新 bundle 版本标识及旧 bundle 读取策略，添加兼容测试。
- [ ] 执行 `cargo test -p codex-rollout-trace`；确认新测试及原有测试通过。

## Task 2: HTTP 实际发送请求及重试采集

**Files:** 修改 `codex-rs/http-client/src/request.rs`、`transport.rs`、`codex-api/src/endpoint/session.rs`、`endpoint/responses.rs`、`core/src/client.rs`；新增 `core/src/model_capture.rs` 及对应模块注册和测试。

**Interfaces:** 由 core 创建 inference 关联上下文；每次实际 HTTP attempt 产生独立 ID，transport 报告最终 body 和发送结果，Responses 层保留压缩前 JSON。

- [ ] 先写 mock 服务对照测试：普通 JSON、zstd、连接失败、HTTP 重试、鉴权恢复；断言每次发送独立、字节可核对、凭证缺失。
- [ ] 接入可选观察器；采集点限定在模型 Responses 路径，不记录无关登录请求。
- [ ] 保留已有 logical_request，新增实际发送 body 引用及压缩信息；不做第二次可能产生不同结果的序列化替代发送字节。
- [ ] 执行 `cargo test -p codex-http-client`、`cargo test -p codex-api` 及相关 core 定向测试。

## Task 3: WebSocket 与原始返回事件

**Files:** 修改 `codex-rs/codex-api/src/endpoint/responses_websocket.rs`、`sse/responses.rs`、相关流构造入口和测试；修改 core 观察器适配。

**Interfaces:** SSE 报告应用层事件，WS 报告应用层文本消息；都有接收序号、发送尝试及连接关联。原始数据先观察再反序列化。

- [ ] 先写失败测试：WS 预热/增量/取消消息与服务端接收一致；SSE 网络分块、中文、多段推理和非法 JSON 被保留。
- [ ] 接入发送和接收观察器；捕获服务端关闭、解析失败、取消和回退终态。
- [ ] 写并发及回退测试，确保多个连接/attempt 不混合；相同片段合法重复保留。
- [ ] 执行 `cargo test -p codex-api` 与现有 core WebSocket/Responses 定向测试。

## Task 4: App 交付事件、聚合与中断保留

**Files:** 修改 `codex-rs/core/src/session/mod.rs`、`session/turn.rs`、`app-server/src/bespoke_event_handling.rs` 及实际推理通知映射路径；新增 `rollout-trace/src/reducer/capture.rs`，修改 reducer/model 导出及测试。

**Interfaces:** App 可见事件与原始流有不同 source；只使用实际可得的关联 ID。归约输出按 attempt/item/kind/part 分组，保存 delta 及最终快照引用。

- [ ] 先写失败测试：summary/text 分离、同段 delta/done 不重复、顺序调整、取消前片段完整保留、关联不明确时显式为空。
- [ ] 记录交付前事件和 app-server 推理通知；不改变发往 App 的协议或事件序列。
- [ ] 实现完成、取消和退出时 flush；进程中断后恢复有效日志前缀并标记 unknown/partial。
- [ ] 执行 `cargo test -p codex-rollout-trace`、core 定向测试和 app-server 相关测试。

## Task 5: 故障验收、文档与 App 实测

**Files:** 更新 `codex-rs/rollout-trace/README.md`、新增采集集成测试与实际验收记录。

**Interfaces:** 复用 `codex debug trace-reduce` 输出可读索引；记录真实二进制路径、构建来源及采集状态，不把 shell PATH 当作 App 使用证据。

- [ ] 注入慢磁盘、队列满、磁盘满、额度耗尽、强杀；核对 partial 状态和主链路表现。
- [ ] 完成 JSONL/payload 完整性与旧 bundle 回归测试，检查禁用路径及资源边界；不虚构性能承诺。
- [ ] 在 `codex-rs` 执行最终相关 crate 测试和项目规定的格式/静态检查；保存真实结果。
- [ ] 按项目现有打包方式生成完整本地包；确认 App 进程加载该包后测试普通思考、多段推理、主动取消及离线读取。
- [ ] 将 mock 验证、CLI 验证、真实 App 手工验收分别标注；只有实际完成的项目可标记通过。

## 回滚

关闭采集环境变量并重启实际 App 后端，停止未来记录；需要时切回之前完整本地包。保留已有采集文件和验收证据。
