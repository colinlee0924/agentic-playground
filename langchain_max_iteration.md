# mask-kernel AI Agentic Platform — 開發需求規格與設計

> **版本**: v0.1.0-draft  
> **日期**: 2026-03-31  
> **作者**: JIRA x AI Task Force  
> **狀態**: Draft

-----

## 1. 背景與目標

### 1.1 專案背景

M公司內部約 15 個事業單位（BU）、9,000+ 使用者，目前由 JIRA x AI Task Force 推動企業級 multi-agent 平台（代號 mask-kernel SDK）。平台採用 LangChain v1.x 為核心框架，搭配 A2A protocol 做 agent 間協調，Open WebUI 為統一前端入口，Langfuse（self-hosted）為 LLM observability 平台。

### 1.2 核心目標

- 以 LangChain v1.x `create_agent` + middleware 架構，建構 production-grade multi-tenant agent 服務
- 部署至 Kubernetes，agent 為 stateless Pod，所有 persistent state 由外部 PostgreSQL 承載
- 支援 per-BU 動態配置（model、tools、skills），透過 IT Gateway JWT 注入 tenant context
- 提供 graceful degradation 機制處理 recursion limit，確保企業級服務品質
- 保留日後遷移至 Deep Agents SDK（`create_deep_agent`）的彈性

-----

## 2. 技術選型決策

### 2.1 create_agent vs create_deep_agent

經評估後，Phase 1 採用 `create_agent`（LangChain v1.x）而非 `create_deep_agent`（Deep Agents SDK），理由如下：

|評估項目                   |create_agent                   |create_deep_agent                                                |
|-----------------------|-------------------------------|-----------------------------------------------------------------|
|Middleware 控制粒度        |完全自主組裝                         |預裝 4 個 middleware，客製需 override                                   |
|A2A 整合                 |不衝突，agent 間協調走 A2A executor    |SubAgent 機制與 A2A executor 功能重疊                                   |
|Langfuse tracing       |callback-based tracing 正常運作    |已知 `create_deep_agent` 內部 `trace=False` 阻擋 callback-based tracing|
|Subagent bug           |不使用 SubAgentMiddleware，不受影響    |v0.4.4 已知 bug：subagent 不繼承 `recursion_limit`（issue #1698）        |
|Deep Agents VFS backend|可透過 `FilesystemMiddleware` 直接引用|內建                                                               |
|學習曲線                   |較低，middleware 行為透明             |較高，需理解預裝 middleware 交互影響                                         |

### 2.2 關鍵依賴版本

```
langchain >= 1.2.0
langgraph >= 1.0.6        # RemainingSteps + 預設 recursion_limit=1000
deepagents >= 0.4.4       # 僅引用 FilesystemMiddleware 與 Backend 模組
langgraph-checkpoint-postgres
langfuse (self-hosted)
```

-----

## 3. 系統架構

### 3.1 整體架構概觀

```
┌──────────────────────────────────────────────────────────┐
│  Open WebUI (前端)                                        │
│  └─ SSE streaming + <details> 視覺化                      │
└───────────────────────┬──────────────────────────────────┘
                        │ HTTP / WebSocket
                        ▼
┌──────────────────────────────────────────────────────────┐
│  IT Gateway                                               │
│  ├─ JWT 驗證 / 解析 bu_id, user_id                        │
│  └─ Per-BU 動態 config 注入                               │
└───────────────────────┬──────────────────────────────────┘
                        │
                        ▼
┌──────────────────────────────────────────────────────────┐
│  K8s: mask-kernel Agent Service (Stateless Pod)           │
│  ├─ create_agent + Middleware Stack                       │
│  │   ├─ RecursionGuardMiddleware    (recursion 防護)      │
│  │   ├─ StepTrackerMiddleware       (Langfuse 監控)       │
│  │   ├─ TenantContextMiddleware     (多租戶 context 注入) │
│  │   └─ FilesystemMiddleware        (VFS backend)         │
│  ├─ JIRA Agent Skills (per-request dynamic loading)      │
│  └─ A2A Executor (agent 間協調)                           │
└──────────┬──────────────┬────────────────────────────────┘
           │              │
           ▼              ▼
┌─────────────────┐  ┌─────────────────────────────────────┐
│  PostgreSQL      │  │  Langfuse (self-hosted)             │
│  ├─ PostgresSaver│  │  ├─ Trace / Span                    │
│  │  (checkpoint) │  │  ├─ Step count metrics               │
│  └─ PostgresStore│  │  └─ Per-BU dashboard                │
│    (durable VFS) │  └─────────────────────────────────────┘
└─────────────────┘
```

### 3.2 Middleware Stack 設計

Middleware 執行順序（由外到內）：

```
Request
  → RecursionGuardMiddleware     # 1. 最先執行，檢查剩餘步數
    → StepTrackerMiddleware      # 2. 記錄 step 到 Langfuse
      → TenantContextMiddleware  # 3. 注入 BU/User context
        → FilesystemMiddleware   # 4. VFS 檔案工具
          → Agent (model + tools)
```

-----

## 4. VFS Backend 與多租戶設計

### 4.1 Backend 架構

採用 CompositeBackend，將不同路徑導向不同 backend：

```
CompositeBackend
├─ /* (default)        → StateBackend    (ephemeral scratch pad)
├─ /memories/*         → StoreBackend    (user-scoped persistent)
└─ /skills/*           → StoreBackend    (BU-scoped shared)
```

StateBackend 的資料透過 `PostgresSaver`（checkpointer）持久化，確保 K8s Pod 重啟後同一 thread 的暫存資料可恢復。StoreBackend 的資料透過 `PostgresStore` 天生持久化，支援跨 thread、跨 session 存取。

### 4.2 多租戶 Namespace 設計

利用 LangGraph Store 的 hierarchical namespace tuple 實現租戶隔離：

```python
# 個人記憶 — 僅該使用者可存取
namespace = (bu_id, user_id, "memories")

# BU 共用 Skills — 該 BU 所有使用者可存取
namespace = (bu_id, "shared", "skills")

# 全公司共用 — 所有 BU 可存取
namespace = ("global", "skills")

# BU 共用知識庫
namespace = (bu_id, "shared", "knowledge")
```

### 4.3 Tenant Context 注入流程

```
IT Gateway JWT
  → 解析出 bu_id, user_id, roles
    → 注入 LangGraph configurable:

config = {
    "configurable": {
        "thread_id": f"{bu_id}-{user_id}-{session_id}",
        "user_id": user_id,
        "bu_id": bu_id,
    },
    "recursion_limit": 150,  # 可根據 BU 動態調整
}
```

### 4.4 實作範例

```python
from langchain.agents import create_agent
from deepagents.middleware.filesystem import FilesystemMiddleware
from deepagents.backends import CompositeBackend, StateBackend, StoreBackend
from langgraph.checkpoint.postgres import PostgresSaver
from langgraph.store.postgres import PostgresStore

DB_URI = "postgresql://user:pass@pg-host:5432/agents?sslmode=require"

def make_backend(rt):
    return CompositeBackend(
        default=StateBackend(rt),
        routes={
            "/memories/": StoreBackend(rt),
            "/skills/": StoreBackend(rt),
        },
    )

def create_mask_kernel_agent(tools, system_prompt, store, checkpointer):
    return create_agent(
        model="anthropic:claude-sonnet-4-5-20250929",
        tools=tools,
        system_prompt=system_prompt,
        store=store,
        checkpointer=checkpointer,
        middleware=[
            RecursionGuardMiddleware(),
            StepTrackerMiddleware(),
            TenantContextMiddleware(),
            FilesystemMiddleware(
                backend=make_backend,
            ),
        ],
    )
```

-----

## 5. Recursion Limit 防護機制

### 5.1 問題分析

LangGraph 的 agent 本質為 tool-calling loop，存在以下風險：

- LLM 重複呼叫同一 tool 不收斂（infinite loop）
- 複雜任務自然需要大量步驟，撞到 limit
- `GraphRecursionError` 為硬中斷，直接 crash，無法回傳部分結果
- 社群已提出 `RecursionLimitFallbackMiddleware` 的 feature request（langchain#33653），但官方尚未實作

### 5.2 三層防護架構

```
┌─────────────────────────────────────────────────┐
│  Layer 1: Proactive — RecursionGuardMiddleware   │
│  在 graph 內部用 RemainingSteps 監控              │
│  快撞牆時注入 system hint → LLM 主動收斂          │
│  剩餘 ≤ 3 步時強制 jump_to __end__               │
├─────────────────────────────────────────────────┤
│  Layer 2: Reactive — try/except 兜底             │
│  在 request handler 外層 catch                   │
│  GraphRecursionError                             │
│  回傳已完成的部分結果 + truncated 標記             │
├─────────────────────────────────────────────────┤
│  Layer 3: Observability — StepTrackerMiddleware   │
│  每一步推送 step count 到 Langfuse               │
│  追蹤 per-BU / per-skill 的步數分佈              │
│  設定 p95 alert                                  │
└─────────────────────────────────────────────────┘
```

### 5.3 RecursionGuardMiddleware 設計

```python
from langchain.agents.middleware import AgentMiddleware, AgentState
from langgraph.managed import RemainingSteps
from typing import Any, TypedDict

class RecursionGuardState(TypedDict):
    remaining_steps: RemainingSteps

class RecursionGuardMiddleware(AgentMiddleware):
    """
    三階段漸進式 recursion 防護：
    - remaining > 10: 正常執行
    - remaining ≤ 10: 注入 system hint 引導收斂
    - remaining ≤ 3:  強制結束，回傳部分結果
    """

    state_schema = RecursionGuardState

    def __init__(self, soft_threshold: int = 10, hard_threshold: int = 3):
        self.soft_threshold = soft_threshold
        self.hard_threshold = hard_threshold

    def before_model(self, state: AgentState) -> dict[str, Any] | None:
        remaining = state.get("remaining_steps", 999)

        if remaining <= self.hard_threshold:
            return {
                "messages": [{
                    "role": "assistant",
                    "content": self._build_partial_summary(state),
                }],
                "jump_to": "__end__",
            }

        if remaining <= self.soft_threshold:
            return {
                "messages": [{
                    "role": "system",
                    "content": (
                        f"[系統提示] 僅剩 {remaining} 步可用。"
                        "請立即總結目前已取得的資訊並回覆最終結果，"
                        "不要再呼叫額外的工具。"
                    ),
                }],
            }

        return None

    def _build_partial_summary(self, state: AgentState) -> str:
        """從當前 messages 中擷取已完成的部分結果"""
        messages = state.get("messages", [])
        tool_results = [
            m for m in messages
            if hasattr(m, "type") and m.type == "tool"
        ]
        if tool_results:
            return (
                "已達到處理步數上限。以下是目前已取得的部分結果：\n\n"
                + "\n".join(str(r.content)[:200] for r in tool_results[-3:])
                + "\n\n建議將任務拆分為更小的範圍後重試。"
            )
        return "已達到處理步數上限，請縮小任務範圍後重試。"
```

### 5.4 StepTrackerMiddleware 設計

```python
class StepTrackerMiddleware(AgentMiddleware):
    """推送 step metrics 到 Langfuse"""

    def after_model(self, state: AgentState, config=None) -> dict | None:
        if config is None:
            return None

        step = config.get("metadata", {}).get("langgraph_step", 0)
        limit = config.get("recursion_limit", 1000)
        bu_id = config.get("configurable", {}).get("bu_id", "unknown")

        # 推送到 Langfuse
        langfuse.score(
            trace_id=config.get("metadata", {}).get("run_id"),
            name="agent_step_count",
            value=step,
            comment=f"BU: {bu_id}, limit: {limit}, utilization: {step/limit:.2%}",
        )

        return None
```

### 5.5 Request Handler 兜底

```python
from langgraph.errors import GraphRecursionError

async def handle_agent_request(messages: list, config: dict) -> dict:
    try:
        result = await agent.ainvoke({"messages": messages}, config=config)
        return {
            "success": True,
            "result": result,
        }

    except GraphRecursionError:
        # 從 checkpoint 取最後成功的 state（如有需要）
        return {
            "success": False,
            "result": {
                "messages": [{
                    "role": "assistant",
                    "content": "此任務較為複雜，已超出單次處理上限。建議拆分為更小的子任務重試。",
                }],
            },
            "metadata": {
                "truncated": True,
                "reason": "recursion_limit_exceeded",
            },
        }
```

-----

## 6. Recursion Limit 參數調校策略

### 6.1 初始設定

不要一開始就猜 `recursion_limit` 的最佳值。採用 data-driven 方式：

1. Phase 1 初始設定 `recursion_limit=150`（寬鬆值）
1. 透過 StepTrackerMiddleware 收集 2 週以上的 step 分佈數據
1. 依據 p95 數據分 skill type 設定差異化 limit

### 6.2 Per-BU / Per-Skill 差異化設定

不同 skill type 消耗步數差異極大：

|Skill 類型              |預估步數範圍|建議 limit|
|----------------------|------|--------|
|JIRA 查詢（單一 issue）     |3–8   |30      |
|JIRA 批次修改             |10–30 |80      |
|報表產生（跨專案彙整）           |20–60 |150     |
|研究型任務（web search + 分析）|30–100|200     |

透過 IT Gateway 注入的 per-request config 可動態調整：

```python
# IT Gateway 根據 skill type 決定 recursion_limit
config = {
    "recursion_limit": skill_config.get("recursion_limit", 150),
    "configurable": {
        "thread_id": thread_id,
        "user_id": user_id,
        "bu_id": bu_id,
    },
}
```

-----

## 7. K8s 部署設計

### 7.1 Pod 設計原則

- Agent Pod 為 **stateless**，不依賴 local disk
- 所有 persistent state 由 PostgreSQL 承載（PostgresSaver + PostgresStore）
- Pod 可水平擴展，request 可被任意 Pod 處理
- 不使用 `FilesystemBackend`（官方明確警告不適合 production multi-tenant 環境）

### 7.2 PostgreSQL 連線管理

```python
# 建議使用 connection pool
DB_URI = "postgresql://user:pass@pg-host:5432/agents?sslmode=require"

# Application startup
store = PostgresStore.from_conn_string(DB_URI)
checkpointer = PostgresSaver.from_conn_string(DB_URI)

# Application shutdown
store.close()
checkpointer.close()
```

### 7.3 健康檢查

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 10
  periodSeconds: 30

readinessProbe:
  httpGet:
    path: /ready    # 檢查 PostgreSQL 連線
    port: 8000
  initialDelaySeconds: 5
  periodSeconds: 10
```

-----

## 8. 遷移路徑

### Phase 1（現階段）— create_agent + 自選 Middleware

- `create_agent` + `FilesystemMiddleware` + `CompositeBackend`
- 多租戶隔離：Store namespace `(bu_id, user_id, category)`
- Agent 間協調：A2A executor（mask-kernel orchestrator）
- Observability：Langfuse callback
- Recursion 防護：三層防護架構

### Phase 2 — 加入 SummarizationMiddleware

- 引入 Deep Agents 的 `SummarizationMiddleware` 處理長對話
- 評估 `ClearToolUsesEdit` middleware 的 context 壓縮效果
- 調校 fraction-based vs absolute token triggers（需配合 per-BU model config）

### Phase 3 — 評估 create_deep_agent

待以下條件滿足後評估是否遷移：

- Deep Agents `SubAgentMiddleware` config propagation bug 修復
- `create_deep_agent` 內部 `trace=False` 問題解決（影響 Langfuse tracing）
- 或確認 Deep Agents 提供官方的 `RecursionLimitFallbackMiddleware`

遷移時只需將 `create_agent` 替換為 `create_deep_agent`，middleware stack 可複用。

-----

## 9. 已知風險與緩解措施

|風險                                                |影響                                    |緩解措施                                                 |
|--------------------------------------------------|--------------------------------------|-----------------------------------------------------|
|LangGraph 1.0.6 infinite loop bug（issue #6731）    |Agent 無視 prompt stop condition 持續 loop|RecursionGuardMiddleware 強制介入                        |
|`.with_config()` recursion_limit 不生效              |設定被忽略，撞到預設 limit                      |統一在 `invoke()` 時傳 config，不用 `.with_config()`         |
|Middleware 過多導致 recursion_limit 消耗加速（issue #33740）|每個 middleware hook 也算 step            |監控 step utilization，必要時調高 limit                      |
|PostgreSQL 單點故障                                   |所有 state 丟失                           |HA PostgreSQL + 定期備份                                 |
|LLM 重複呼叫同 tool（迴圈）                                |浪費 token + 撞 limit                    |Tool output 加入 dedup 提示 + RecursionGuardMiddleware 收斂|

-----

## 10. 驗收標準

### 10.1 功能驗收

- [ ] Agent 可在 K8s Pod 中正常啟動並處理請求
- [ ] Pod 重啟後，同一 thread 的對話可從 checkpoint 恢復
- [ ] 不同 BU 的使用者資料完全隔離（namespace 驗證）
- [ ] 跨 session 的 /memories/ 路徑資料可持久存取
- [ ] RecursionGuardMiddleware 在 soft threshold 時注入 hint
- [ ] RecursionGuardMiddleware 在 hard threshold 時強制結束並回傳部分結果
- [ ] GraphRecursionError 被正確 catch，不回傳 500 Error

### 10.2 效能驗收

- [ ] 單一 agent request 平均回應時間 < 30s（簡單查詢 < 5s）
- [ ] PostgreSQL 連線池穩定，無連線洩漏
- [ ] Langfuse 可正確顯示每個 request 的 step count

### 10.3 監控驗收

- [ ] Langfuse dashboard 顯示 per-BU step 分佈
- [ ] Step utilization > 80% 時觸發 alert
- [ ] Recursion limit exceeded 事件有獨立 alert channel

-----

## 附錄 A：StateBackend vs StoreBackend 比較

|           |StateBackend                    |StoreBackend              |
|-----------|--------------------------------|--------------------------|
|資料範圍       |單一 thread 內                     |跨 thread、跨 session        |
|持久化方式      |靠 checkpointer（PostgresSaver）   |靠 BaseStore（PostgresStore）|
|Pod 重啟後    |有 PostgresSaver 可恢復同 thread     |天生持久                      |
|適用場景       |暫存、中間結果、大型 tool output eviction |長期記憶、Skills、使用者偏好         |
|Subagent 共享|Supervisor 與 subagent 共享同一 state|依 namespace 控制共享範圍        |

## 附錄 B：社群現況與官方態度

截至 2026-03 的生態系現況：

|議題                                   |狀態                 |參考                    |
|-------------------------------------|-------------------|----------------------|
|RecursionLimitFallbackMiddleware     |Feature request，未實作|langchain#33653       |
|Subgraph GraphRecursionError catch   |Forum 討論，無官方解答     |LangChain Forum       |
|SubAgentMiddleware config propagation|已知 bug，未修復         |deepagents#1698       |
|LangGraph 1.0.6 infinite loop        |Bug report，pending |langgraph#6731        |
|RemainingSteps API                   |官方推薦方案             |LangGraph Graph API 文件|
|預設 recursion_limit 改為 1000           |LangGraph ≥ 1.0.6  |LangGraph Graph API 文件|
