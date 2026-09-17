# 关键架构决策（ADR 摘要）

## ADR-001 → ADR-004 编排框架：Matrix 分布式 → LangGraph 直调

**决策**：从「Matrix 分布式调度」收敛为「LangGraph 直调为主」，对话模式保留 Manager，形成双模式混合架构。

**理由**：实测 Matrix 协议在 Agent 编排场景有系统性可靠性问题（DM 静默丢弃、消息分片截断、单队列串行化），而代码现状已用 LangGraph 工作。管道模式确定性执行，对话模式保留灵活性，两模式共享同一套 Agent 函数。

## 四段管道而非端到端单 Agent

**决策**：Parse → Write → Check → Guard 拆成独立阶段，而非一个黑盒 Agent 端到端生成。

**理由**：招投标场景必须可审计、可拦截。单 Agent 黑盒无法保证合规；拆段后每段可独立验证，Check 可拦废标条款，Guard 可拦越权调用。

## DeepSeek 直连

**决策**：5 Agent 经 LiteLLM 直连 DeepSeek V4 Pro，不绕中间网关。

**理由**：降延迟、少一层故障点。推理模型需更大 max_tokens——实测 4096 被 reasoning 推理链占满导致内容截断（finish_reason=length），提到 16384。

## 知识库 RAG 而非全量入 prompt

**决策**：10 类 166 份领域文档走 RAG（ChromaDB），不塞进 prompt。

**理由**：投标领域知识量大，全量入 prompt 超上下文且贵；RAG 按需检索，Parse/Write/Check 各取所需。

## 五门安全网关（MCP Guard）

**决策**：每一次 Agent 工具调用先过五门，全部通过才放行。

**理由**：多 Agent 系统的安全边界必须在工具调用层收紧。五门各司其职——G1 白名单硬阻断未注册调用，G2 动作分类区分读/写风险，G3 风险评估量化敏感度，G4 速率限制防滥用，G5 HMAC 审计留不可篡改证据。五门与 BidForge 业务解耦，可独立部署到任意 MCP 项目。

| 门 | 职责 |
|----|------|
| G1 白名单 | agent_id + tool_name 匹配注册白名单 |
| G2 动作分类 | read / draft / external_write |
| G3 风险评估 | low / medium / high |
| G4 速率限制 | 60s 滑动窗口，每 Agent 100 请求 |
| G5 HMAC 审计 | HMAC-SHA256 签名 audit.jsonl，10MB 轮转 |
