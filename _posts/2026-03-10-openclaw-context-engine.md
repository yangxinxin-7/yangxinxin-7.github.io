---
layout: post
title: "OpenClaw 最新 ContextEngine 详解"
date: 2026-03-10
---

## Hook 详解

### 背景：为什么需要 ContextEngine

在 v2026.3.7 之前，OpenClaw 的 context 管理（上下文组装、历史压缩）全部硬编码在核心代码里，插件无法干预。PR 作者在提交说明里明确写道：

> "OpenClaw's context management is hardcoded in core, making it impossible for plugins to provide alternative context strategies."

ContextEngine 就是为了解决这个问题而引入的插件接口，允许插件完全接管 context 组装、`/compact` 压缩、以及子 agent 上下文生命周期。**不配置任何插件时，行为与之前完全一致，零变化。**

配置方式：

```json
{
  "plugins": {
    "slots": {
      "contextEngine": "your-engine-id"
    }
  }
}
```

注册方式：

```typescript
export default function (api) {
  api.registerContextEngine("your-engine-id", () => ({
    info: { id: "your-engine-id", name: "My Engine", ownsCompaction: true },
    async bootstrap(ctx) { ... },
    async ingest(ctx) { ... },
    async assemble(ctx) { ... },
    async compact(ctx) { ... },
    async afterTurn(ctx) { ... },
    async prepareSubagentSpawn(ctx) { ... },
    async onSubagentEnded(ctx) { ... },
  }));
}
```

---

### 前置概念：Session、Run、Turn 的区别

理解这七个 hook 的触发时机，需要先搞清楚 OpenClaw 里三个容易混淆的概念：

- **Session**：整个对话的容器，从用户开始聊到结束，贯穿始终
- **Run**：每次用户发一条消息，触发一次 run。一个 session 包含很多次 run
- **Turn**：和 run 基本等价，指一轮完整的"用户消息 → 模型回复"交互，包括中间所有 tool call 执行完毕、最终输出结束

此外，OpenClaw 的 agent 有两种运行模式：

- **embedded runner**：OpenClaw 自己内置的引擎，跑在 Gateway 进程里，支持工具调用、session 管理、compaction 等完整功能。平时通过 WhatsApp/Telegram/Discord 发消息走的就是这条路
- **CLI runner**：专门给 Claude Code、Codex CLI、Gemini CLI 等自带 agentic CLI 的 provider 用的。这种模式下 OpenClaw 只是个外壳，ContextEngine 的 hook **不会被调用**

ContextEngine 只在 embedded runner 模式下生效。

---

### 七个 Hook 详解

#### 1. `bootstrap`

**触发时机：**
- 每次 embedded run 开始前
- `/compact` 触发前
- 子 agent spawn 前

注意：**不是整个 session 只跑一次**，而是每次引擎被唤醒时都会调用。一个 session 里用户发了多少条消息，bootstrap 就会被调用多少次（每次 run 各一次），加上 compact 和 subagent 的次数。

**做什么：**
初始化引擎所需的资源，比如建立数据库连接、执行 schema 迁移、初始化检索索引等。

由于会被反复调用，实现时必须做幂等处理：

```typescript
async bootstrap({ sessionKey }) {
  if (this.db) return; // 已初始化则跳过
  this.db = await openDatabase(config.databasePath);
  await runMigrations(this.db);
}
```

**注意：** 如果你想在 session **第一次**开始时做某件事（真正的一次性初始化），正确的时机应该放在 `ingest` 里——判断这是不是这个 session 的第一条消息再做处理，而不是放在 bootstrap 里。

---

#### 2. `ingest`

**触发时机：** 每条新消息进入 session 时，包括用户消息、助手回复、tool call、tool result 等所有消息类型。

**做什么：**
这是消息"入库"的地方。引擎在这里把每条消息持久化到自己的存储系统，并做预处理：

- 解析 tool call / tool result 的配对关系，修复可能的孤立消息
- 把结构化内容块（文本、tool call、图片等）拆分存储
- 计算 token 数
- 对超大文件内容做截获处理，避免单条消息就撑爆 context

处理完后，把消息加入当前 conversation 的有序 context 列表，供后续 `assemble` 使用。

```typescript
async ingest({ message, sessionKey }) {
  const processed = await interceptLargeFiles(message);
  await this.db.saveMessage(processed);
  await this.db.appendContextItem({ messageId, conversationId });
  return { ingested: true };
}
```

---

#### 3. `assemble`

**触发时机：** 每次 run 开始前，OpenClaw 调用这个 hook 来获取最终送给模型的消息列表。

**做什么：**
这是整个 ContextEngine 里**最核心的 hook**，它的返回值直接决定模型能看到什么。

传统实现是简单地截取 session history 最后 N 条消息。自定义引擎可以在这里做任意复杂的事情：把旧消息替换成摘要节点、注入 RAG 检索结果、动态调整 token 预算等。

`assemble` 接收当前会话的原始消息列表作为输入，但完全不必照单全收，可以返回一个经过重新组织的列表。

```typescript
async assemble({ messages, tokenBudget }) {
  const contextItems = await this.db.getContextItems(conversationId);
  const resolved = await resolveItems(contextItems); // 摘要节点 + 原始消息混合
  const trimmed = trimToTokenBudget(resolved, tokenBudget);
  return {
    messages: trimmed,
    estimatedTokens: countTokens(trimmed),
  };
}
```

`assemble` 同时也是**被动式压缩的触发点**：如果发现组装出来的 context 已经超过 `contextThreshold × tokenBudget`，会在返回给模型之前先触发一次 compact。

---

#### 4. `compact`

**触发时机有三条路径：**

1. **afterTurn 触发（主动式）**：每轮结束后检测到 token 积累超阈值，后台异步触发，提前预防，用户无感知
2. **assemble 触发（被动式）**：下一轮 run 开始时，assemble 发现 context 已经超限，在组装前强制触发，是最后一道保险
3. **`/compact` 命令**：用户手动触发全量压缩

三条路径最终都走进同一个 `compact` hook，只是触发时机不同。

**做什么：**
执行"压缩"操作，把旧的对话历史变成更紧凑的形式，释放 context window 空间。如果插件声明了 `ownsCompaction: true`，OpenClaw 的原生 compact 逻辑会被完全绕过，由插件自己处理。

```typescript
async compact({ sessionKey, force }) {
  const chunks = await this.findEligibleChunks();
  for (const chunk of chunks) {
    const summary = await this.summarize(chunk.messages);
    await this.db.saveSummary(summary);
    await this.db.replaceContextItems(chunk.messageIds, summary.id);
  }
  return { ok: true, compacted: chunks.length > 0 };
}
```

---

#### 5. `afterTurn`

**触发时机：** 每轮对话完成后——用户消息 → 模型回复（含所有 tool call 执行完毕）→ 最终输出结束，此时触发。

注意：**一个 session 里每次 turn 结束都会触发**，不是 session 结束才触发。

**做什么：**
收尾工作，最典型的用途是作为**主动式压缩的入口**。

此时这一轮所有的消息和 tool result 都已经被 `ingest` 处理完毕入库，材料是完整的，适合做异步压缩。实现时通常不 `await` 压缩操作，避免阻塞用户看到下一条回复：

```typescript
async afterTurn({ sessionKey }) {
  const shouldCompact = await this.shouldRunIncrementalPass();
  if (shouldCompact) {
    // 异步，不 await，不阻塞响应
    this.runIncrementalCompaction().catch(logger.error);
  }
}
```

---

#### 6. `prepareSubagentSpawn`

**触发时机：** 主 agent 通过 `sessions_spawn` 工具启动子 agent 之前。

**做什么：**
为子 agent 准备专属的 context 视图。默认行为可能是把父 session 的整个历史复制给子 agent，但这往往是浪费的——子 agent 通常只需要完成某个子任务，给它太多无关历史反而有害。

在这里可以为子 agent 创建独立的 conversation 记录，并精确控制子 agent 能访问哪些父 session 的历史内容：

```typescript
async prepareSubagentSpawn({ parentSessionKey, subagentSessionKey, task }) {
  await this.db.ensureConversation(subagentSessionKey);
  // 只授予子 agent 访问与任务相关的摘要节点的权限
  const relevant = await this.findRelevantSummaries(task);
  await this.expansionAuth.grantAccess(subagentSessionKey, relevant);
}
```

子 agent 启动后同样会触发自己的 `bootstrap` → `ingest` → `assemble` → `afterTurn` → `compact` 生命周期。

---

#### 7. `onSubagentEnded`

**触发时机：** 子 agent 的 session 结束（正常完成或被终止）后。

**做什么：**
处理子 agent 结果的"回收"工作：清理之前授予的访问权限，以及可选地把子 agent 的关键输出摘要化后注入到父 session 的 context 里：

```typescript
async onSubagentEnded({ parentSessionKey, subagentSessionKey, result }) {
  await this.expansionAuth.revokeAccess(subagentSessionKey);
  if (result.summary) {
    await this.db.injectSubagentResult(parentSessionKey, result.summary);
  }
}
```

---

### 整体生命周期流程

```
Gateway 启动，用户发第一条消息
  └─ bootstrap()              ← 引擎初始化（DB 连接、索引等）
  └─ ingest()                 ← 用户消息入库
  └─ assemble()               ← 组装送给模型的 context
     └─ (若 context 已超限)  → compact() 先压缩再组装
  └─ 模型回复，tool call 执行...
  └─ ingest()                 ← 模型回复、tool result 逐条入库
  └─ afterTurn()              ← 检查是否需要增量压缩，异步触发

用户发第二条消息
  └─ bootstrap()              ← 再次调用（幂等，已初始化则跳过）
  └─ ingest() → assemble() → afterTurn() ...（同上）

用户手动 /compact
  └─ bootstrap()              ← 再次调用
  └─ compact()                ← 全量压缩

主 agent 派生子 agent
  └─ prepareSubagentSpawn()   ← 为子 agent 准备独立 context，授权
     ├─ 子 agent 自己的完整生命周期（bootstrap/ingest/assemble/...）
     └─ onSubagentEnded()     ← 子 agent 结束，清理授权，回收结果
```

---

## lossless-claw 说明

lossless-claw 是随 ContextEngine 接口同期发布的参考实现插件，基于 [LCM（Lossless Context Management）论文](https://papers.voltropy.com/LCM)，完整实现了上述所有七个 hook。

### 解决什么问题

OpenClaw 原生的 compact 是一次性的"一锅端"式压缩：context 快满了，就把所有旧消息喂给 LLM 总结成一段文字，原始消息丢弃。这是有损的——压缩后细节永久丢失，而且压缩时机是被动触发的，用户会感知到明显的卡顿。

lossless-claw 的核心思路是：**消息永远不丢，只是被分层摘要替代，随时可以向下展开取回原始内容。**

### 核心架构：DAG 摘要树

```
原始消息 → Leaf 摘要 (d0) → Condensed 摘要 (d1) → Condensed 摘要 (d2) → ...
```

所有原始消息持久化到 SQLite 数据库。压缩时，把一批旧消息用 LLM 摘要成一个 Leaf 摘要（d0）。多个 d0 摘要再凝练成 d1，多个 d1 再凝练成 d2，形成一棵 DAG（有向无环图）。

每层摘要用不同的 prompt 策略：

| 深度 | 类型 | 摘要策略 |
|------|------|---------|
| d0 | Leaf | 叙事式，带时间戳，保留文件操作、决策细节 |
| d1 | Condensed | 按时间顺序的 session 摘要，去重 |
| d2 | Condensed | 以目标/成果/后续为导向，自包含 |
| d3+ | Condensed | 只保留关键决策、关系、经验教训 |

### 每轮 context 组装方式（assemble）

每次给模型组装 context 时：

1. 从 `context_items` 表拿到有序的 context 列表（摘要引用 + 原始消息引用混合）
2. 保护最近 `freshTailCount`（默认 32）条原始消息，不参与裁剪
3. 从最旧的 item 开始，按 token 预算从前往后填充，超出预算的部分不放入 context（但仍在 DB 里）
4. 摘要节点以 XML 格式注入，让模型知道这是压缩过的内容：

```xml
<summary id="sum_abc123" kind="condensed" depth="1"
         descendant_count="8"
         earliest_at="2026-02-17T07:37:00"
         latest_at="2026-02-17T15:43:00">
  <content>...摘要文本...</content>
</summary>
```

### 压缩触发机制（compact + afterTurn）

**主动式（afterTurn 触发）：** 每轮结束后，检查 freshTailCount 保护区之外的原始消息是否超过 `leafChunkTokens`（默认 20000 token）。超了就在后台异步跑 leaf pass，生成 d0 摘要。如果配置了 `incrementalMaxDepth > 0`，还会接着把够数量的 d0 凝练成 d1。

**被动式（assemble 触发）：** 如果主动压缩没跟上，下一轮 assemble 发现超过 `contextThreshold × tokenBudget`（默认 75%），会触发全量 sweep：先对所有符合条件的 leaf chunk 生成 d0，再逐层凝练直到稳定。

**手动式：** 用户执行 `/compact`，直接触发全量 sweep。

正常使用下基本不需要手动 `/compact`，afterTurn 的主动压缩会持续在后台维护 DAG。

### 大文件处理

消息里如果包含超过 `largeFileTokenThreshold`（默认 25000 token）的文件内容，lossless-claw 会在 `ingest` 时拦截：把文件内容存到磁盘（`~/.openclaw/lcm-files/`），在消息里用约 200 token 的探索摘要替换原始内容。需要时可用工具取回。

### Agent 工具

lossless-claw 注册了四个 agent 可以主动调用的工具，用于在压缩历史里搜索和回溯细节：

- **`lcm_grep`**：全文或正则搜索所有消息和摘要
- **`lcm_describe`**：查看某个摘要节点或文件的详细内容、元信息和父子关系
- **`lcm_expand_query`**：向下展开 DAG，委托子 agent 从摘要回溯到原始消息，回答一个具体问题
- **`lcm_expand`**：底层 DAG 展开工具，仅供 `lcm_expand_query` 委托的子 agent 使用

### 子 agent 支持（prepareSubagentSpawn / onSubagentEnded）

主 agent spawn 子 agent 时，lossless-claw 为子 agent 建立独立的 conversation 记录，并通过 `expansion_grants` 表授权子 agent 可以展开哪些父 session 的摘要节点。子 agent 通过 `lcm_expand_query` 按需查询父 session 历史，而不是一次性把所有历史塞进子 agent 的 context。子 agent 结束后，授权被清理。

### 安装和配置

```bash
openclaw plugins install @martian-engineering/lossless-claw
```

```json
{
  "plugins": {
    "slots": {
      "contextEngine": "lossless-claw"
    },
    "entries": {
      "lossless-claw": {
        "enabled": true,
        "config": {
          "freshTailCount": 32,
          "contextThreshold": 0.75,
          "incrementalMaxDepth": 1
        }
      }
    }
  }
}
```

重启 Gateway 后生效。

### 性能表现

在 OOLONG benchmark 上，lossless-claw 得分 74.8，Claude Code 是 70.3（同等模型），且上下文越长优势越明显。正常运行时 context 窗口始终保持在 30k–100k token 范围内，基本不需要人工干预。
