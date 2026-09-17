# 关键架构决策(ADR 摘要)

## ADR-001 → ADR-004 编排框架

**决策**:从「Matrix 分布式」收敛到「LangGraph 直调」为主,对话模式保留 Manager。

**理由**:代码现状已用 LangGraph 工作;Matrix 分布式需额外实现,当前单机场景足够。

## 四段管道而非端到端单 Agent

**决策**:Parse → Write → Check → Guard 拆成独立阶段。

**理由**:招投标场景需可审计、可拦截。单 Agent 黑盒无法保证合规,拆段后每段可验证、Guard 可拦废标。

## DeepSeek 直连

**决策**:5 Agent 直连 DeepSeek,不绕中间网关。

**理由**:降延迟、少一层故障点;推理模型需更大 max_tokens(实测 4096 被 reasoning 占满导致截断,提到 16384)。

## 知识库 RAG 而非全量入 prompt

**决策**:10 类 166 份领域文档走 RAG(ChromaDB),不塞 prompt。

**理由**:投标领域知识量大,全量入 prompt 超上下文且贵;RAG 按需检索。

## 五门安全网关(MCP Guard)

**决策**:输出前过 5 gate。

**理由**:废标是招投标最大风险,五门(合规/格式/价格一致性/披露完整性等)拦在交付前。
