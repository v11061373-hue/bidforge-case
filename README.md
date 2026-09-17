# BidForge — 招投标智能体落地案例

> **前向部署工程师（FDE）客户落地案例。** 客户为一家传统招投标代理公司，客户信息与业务数据已脱敏。本仓库仅公开技术架构、决策记录与效果数据，客户定制代码、知识库、规则与真实标书数据保留在私有仓库。

## 一句话

把招投标代理公司的标书生产，从「人工 3–5 天/份」变成「多 Agent 管道 1.5 小时/份」。

## 背景与问题

传统招投标代理公司的三个真实痛点：

1. **标书周期长** — 每份标书需人工逐条响应招标文件，平均 3–5 天。
2. **文件不可复用** — 历史投标文件散落在各项目文件夹，无法检索、无法沉淀。
3. **废标靠人肉** — 评分标准、废标条款靠人工核对，漏一条即废标。

## 方案

确定性管道 + 双模式混合架构 + RAG 知识库 + 五门安全网关：

| 阶段 | 职责 | 关键产出 |
|------|------|----------|
| **Parse** | 解析招标文件 PDF，提取结构化需求 | 需求清单 + 废标模式识别 |
| **Write** | 按需求生成商务标 + 技术标 | 投标书正文 |
| **Check** | 逐条核对评分标准与废标条款（L1/L2/L3） | 合规报告 + 拦截清单 |
| **Guard** | 五门安全网关，管住每一次 Agent 工具调用 | 拦截 / 放行 + 审计日志 |

详见 [docs/architecture.md](docs/architecture.md)。

## 架构概览

双模式混合架构，共享同一套 Agent 函数与存储：

- **对话模式** — Manager（CoPaw LLM）理解意图，逐阶段调度 Worker，人类逐阶段确认。
- **管道模式** — LangGraph 确定性执行 6 节点流水线（parse → match → write → check → export → 人工审批），批量、可恢复、强类型状态。

```
招标文件 PDF → Parse → Write → Check → Guard → 放行/拦截
                    ↑            ↑
              RAG 知识库（10 类 166 份，ChromaDB）
```

## 技术栈

| 层 | 选型 |
|----|------|
| **语言 / 构建** | Python 3.11/3.12 · uv |
| **多 Agent 底座** | AgentTeams（AGENTS.md + SOUL.md + identities）· CoPaw Worker |
| **确定性编排** | LangGraph（StateGraph，6 节点，checkpoint 持久化，HITL `interrupt()`） |
| **对话式编排** | Manager（CoPaw LLM，Matrix @mention 调度） |
| **MCP 协议** | FastMCP · mcporter（SSE transport） |
| **安全网关** | MCP Guard 五门（独立容器 :8104 / 管道内联，HMAC 审计） |
| **模型** | DeepSeek V4 Pro（LiteLLM 接入，5 Agent 直连） |
| **知识库** | RAG · ChromaDB（10 类 166 份领域文档） |
| **存储** | MinIO（S3 对象存储）· SQLite |
| **协同通信** | Matrix（Tuwunel homeserver）· Element Web 客户端 |
| **文档管理** | docstore（MinIO + SQLite + Gradio） |
| **网关** | Higress AI Gateway（Phase 4，计划） |
| **容器化** | Docker Compose（4 Worker + Manager） |
| **质量门禁** | pytest · ruff · mypy |

Worker 端口拓扑：Parse `:8101` · Write `:8102` · Check `:8103` · Guard `:8104`；基础设施 Matrix `:18080` · Element Web `:18088` · MinIO `:9000`。

## 五门安全网关（MCP Guard）

每一次 Agent 工具调用都先过五门，全部通过才放行：

| 门 | 检查内容 | 失败行为 |
|----|----------|----------|
| **G1 白名单** | agent_id + tool_name 必须在注册白名单内 | 硬阻断 |
| **G2 动作分类** | read / draft / external_write 分类，不同风险档 | 按档处理 |
| **G3 风险评估** | low / medium / high（基于身份 + 工具敏感度 + 动作） | 高风险拦截 |
| **G4 速率限制** | 60s 滑动窗口，每 Agent 100 请求 | 超限阻断 |
| **G5 HMAC 审计** | HMAC-SHA256 签名写 audit.jsonl，10MB 轮转 | 防篡改可追责 |

五门是独立审计 sidecar，与 Parse→Write→Check 主流程解耦，可独立部署到任何 MCP 项目。

## 效果

### 业务指标（客户现场）

| 指标 | 改造前 | 改造后 |
|------|--------|--------|
| 标书周期 | 3–5 天/份 | 1.5 小时/份 |
| 文件管理 | 散落各项目文件夹 | 数据库检索 |
| 废标防控 | 人工核对 | 专门 Agent + 五门网关 |

### 工程指标（实测）

- 386 tests 全绿，ruff = 0，mypy = 0
- E2E 实测：Parse 产出 result.md（9KB，15 条需求）→ Write 产出 bid_result.md（33KB / 677 行）→ Check 产出 compliance_check.md（189 行，识别报价内部不一致 100 万差额 + 投标人信息披露缺失）→ Guard 五门 9/9 通过
- DeepSeek reasoning 模型 max_tokens 4096 → 16384（修复推理链占满导致内容截断）

## 我的角色（FDE）

需求调研 → 架构设计 → 编码落地 → 部署 → 验收，现场交付，并负责客户定制知识库与规则的沉淀。

## 关键决策

见 [docs/decisions.md](docs/decisions.md)。

## 目录

```
docs/
  architecture.md  架构与数据流
  decisions.md     关键架构决策（ADR）
```

## 说明

本仓库是部署案例的**公开档案**，不是开源项目。客户定制部分（知识库、规则、prompt、真实标书数据）因保密不公开。
