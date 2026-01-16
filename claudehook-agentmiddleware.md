# Claude Code Hooks vs LangChain v1.0 AgentMiddleware 深度比較

> 兩套 Agent 行為攔截機制的異曲同工之妙

-----

## 1. 核心概念：什麼是「Hook」與「Middleware」？

### 1.1 共同的設計哲學

兩者都是實現 **「橫切關注點」(Cross-cutting Concerns)** 的機制，讓開發者能在 Agent 執行流程的關鍵節點插入自定義邏輯，而不需要修改核心程式碼。

```
┌─────────────────────────────────────────────────────────────┐
│                    Agent 執行流程                            │
│                                                             │
│  [輸入] ──► [前處理 Hook] ──► [核心邏輯] ──► [後處理 Hook] ──► [輸出] │
│              ▲                              ▲               │
│              │                              │               │
│         可攔截/修改/阻斷                 可攔截/修改/記錄         │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 類比理解

|概念                      |Web 開發類比                 |Agent 世界            |
|------------------------|-------------------------|--------------------|
|**Claude Code Hooks**   |Git Hooks / Shell Scripts|事件驅動的 Bash/Python 腳本|
|**LangChain Middleware**|Express.js Middleware    |物件導向的 Python 類別     |

-----

## 2. Claude Code Hooks 深入解析

### 2.1 Hooks 生命週期事件

Claude Code 提供 **8 種 Hook 事件**，覆蓋完整的 Agent 執行週期：

```
┌──────────────────────────────────────────────────────────────────┐
│                    Claude Code Hook 生命週期                      │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  SessionStart ─────► UserPromptSubmit ─────► PreToolUse          │
│       │                    │                     │               │
│       ▼                    ▼                     ▼               │
│  「會話開始」         「Prompt 提交前」        「工具執行前」        │
│                                                                  │
│                     PostToolUse ─────► Stop/SubagentStop         │
│                          │                    │                  │
│                          ▼                    ▼                  │
│                    「工具執行後」          「Claude 要結束時」      │
│                                                                  │
│  其他：Notification（通知）、PreCompact（壓縮前）、               │
│        PermissionRequest（權限請求）                              │
└──────────────────────────────────────────────────────────────────┘
```

### 2.2 Hook 事件詳解

|Hook 事件             |觸發時機                    |常見用途                           |
|--------------------|------------------------|-------------------------------|
|**SessionStart**    |新會話開始或恢復                |載入開發環境上下文（git status、issues）   |
|**UserPromptSubmit**|用戶提交 Prompt 後，Claude 處理前|Prompt 驗證、注入上下文、安全過濾           |
|**PreToolUse**      |工具參數建立後，執行前             |阻擋危險命令（rm -rf）、權限控制            |
|**PostToolUse**     |工具成功執行後                 |自動格式化、Lint 檢查、記錄日誌             |
|**Stop**            |Claude 嘗試結束回應時          |**這是 Ralph Wiggum 的核心！** 品質檢查門檻|
|**SubagentStop**    |子代理完成時                  |確保子任務完整性                       |
|**PreCompact**      |上下文壓縮前                  |備份對話記錄                         |
|**Notification**    |發送通知時                   |自定義通知方式（TTS、Slack）             |

### 2.3 Exit Code 控制機制

```python
# Hook 通過 Exit Code 控制流程
# ┌─────────────────────────────────────────────┐
# │  Exit Code 0  │  成功，繼續執行              │
# │  Exit Code 2  │  阻斷！回饋 stderr 給 Claude │
# │  其他 Exit    │  非阻斷錯誤，顯示給用戶       │
# └─────────────────────────────────────────────┘
```

### 2.4 實際範例：PreToolUse Hook（阻擋危險命令）

**設定檔 `.claude/settings.json`：**

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash|Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "python .claude/hooks/safety_check.py"
          }
        ]
      }
    ]
  }
}
```

**Hook 腳本 `.claude/hooks/safety_check.py`：**

```python
#!/usr/bin/env python3
import json
import sys

# 從 stdin 讀取 Hook 輸入
input_data = json.load(sys.stdin)
tool_name = input_data.get("tool_name", "")
tool_input = input_data.get("tool_input", {})

# 定義危險模式
DANGEROUS_PATTERNS = ["rm -rf", ".env", "DROP TABLE", "sudo"]

# 檢查 Bash 命令
if tool_name == "Bash":
    command = tool_input.get("command", "")
    for pattern in DANGEROUS_PATTERNS:
        if pattern in command:
            # Exit 2 = 阻斷執行，stderr 回饋給 Claude
            print(f"⚠️ 危險命令被阻擋: {pattern}", file=sys.stderr)
            sys.exit(2)

# 自動批准文檔檔案讀取
if tool_name == "Read":
    file_path = tool_input.get("file_path", "")
    if file_path.endswith((".md", ".txt", ".json")):
        output = {
            "decision": "approve",
            "reason": "文檔檔案自動批准",
            "suppressOutput": True
        }
        print(json.dumps(output))
        sys.exit(0)

# 其他情況：正常流程
sys.exit(0)
```

-----

## 3. LangChain v1.0 AgentMiddleware 深入解析

### 3.1 Middleware 架構

LangChain 1.0 引入了全新的 Middleware 系統，採用 **物件導向** 設計：

```python
from langchain.agents.middleware import AgentMiddleware, AgentState, ModelRequest
from langchain.tools.tool_node import ToolCallRequest
from langchain_core.messages import ToolMessage
from typing import Callable, Any

class MyMiddleware(AgentMiddleware):
    """
    AgentMiddleware 基類提供 5 個可覆寫的 Hook 方法
    """
    
    def before_model(self, state: AgentState, runtime: Any) -> dict | None:
        """LLM 呼叫前"""
        pass
    
    def after_model(self, state: AgentState, runtime: Any) -> dict | None:
        """LLM 呼叫後"""
        pass
    
    def before_agent(self, state: AgentState, runtime: Any) -> dict | None:
        """Agent 整體執行前"""
        pass
    
    def after_agent(self, state: AgentState, runtime: Any) -> dict | None:
        """Agent 整體執行後"""
        pass
    
    def wrap_tool_call(
        self, 
        request: ToolCallRequest, 
        handler: Callable[[ToolCallRequest], ToolMessage]
    ) -> ToolMessage:
        """工具呼叫包裝器（類似裝飾器模式）"""
        pass
    
    def wrap_model_call(
        self,
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse]
    ) -> ModelResponse:
        """模型呼叫包裝器"""
        pass
```

### 3.2 Middleware 執行流程

```
┌────────────────────────────────────────────────────────────────┐
│              LangChain Middleware 執行順序                      │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  before_agent() ──► before_model() ──► [LLM 呼叫]              │
│                                           │                    │
│                                           ▼                    │
│                     after_model() ◄── [LLM 回應]               │
│                          │                                     │
│                          ▼                                     │
│  wrap_tool_call() ──► [工具執行] ──► wrap_tool_call() 返回      │
│                                           │                    │
│                                           ▼                    │
│                     (重複直到完成)                              │
│                          │                                     │
│                          ▼                                     │
│                     after_agent()                              │
└────────────────────────────────────────────────────────────────┘
```

### 3.3 實際範例：完整的 Middleware 類別

```python
from langchain.agents import create_agent
from langchain.agents.middleware import AgentMiddleware, AgentState, ModelRequest
from langchain.tools.tool_node import ToolCallRequest
from langchain_core.messages import ToolMessage
from typing import Callable, Any
import logging

logger = logging.getLogger(__name__)

class ComprehensiveMiddleware(AgentMiddleware):
    """
    展示所有 Hook 點的完整 Middleware
    """
    
    def __init__(self, max_retries: int = 3):
        self.max_retries = max_retries
        self.tool_call_count = 0
    
    def before_model(self, state: AgentState, runtime: Any) -> dict | None:
        """LLM 呼叫前：記錄訊息數量"""
        msg_count = len(state.get("messages", []))
        logger.info(f"📝 Before model: {msg_count} messages in context")
        
        # 返回 None 表示正常繼續
        # 返回 dict 可修改 state
        return None
    
    def after_model(self, state: AgentState, runtime: Any) -> dict | None:
        """LLM 呼叫後：檢查回應品質"""
        last_msg = state["messages"][-1]
        logger.info(f"🤖 Model replied: {last_msg.content[:100]}...")
        
        # 可以在這裡實現重試邏輯
        return None
    
    def wrap_tool_call(
        self, 
        request: ToolCallRequest, 
        handler: Callable[[ToolCallRequest], ToolMessage]
    ) -> ToolMessage:
        """工具呼叫包裝：加入重試邏輯與監控"""
        tool_name = request.tool_call["name"]
        tool_args = request.tool_call["args"]
        
        logger.info(f"🔧 Calling tool: {tool_name}")
        logger.info(f"   Arguments: {tool_args}")
        
        self.tool_call_count += 1
        
        # 實現重試邏輯
        for attempt in range(self.max_retries):
            try:
                result = handler(request)
                logger.info(f"✅ Tool {tool_name} succeeded on attempt {attempt + 1}")
                return result
            except Exception as e:
                logger.warning(f"⚠️ Tool {tool_name} failed (attempt {attempt + 1}): {e}")
                if attempt == self.max_retries - 1:
                    raise
        
        return handler(request)
    
    def wrap_model_call(
        self,
        request: ModelRequest,
        handler: Callable
    ):
        """模型呼叫包裝：可實現 Fallback 邏輯"""
        try:
            return handler(request)
        except Exception as e:
            # 可以在這裡切換到備用模型
            logger.error(f"Primary model failed: {e}")
            raise


# 使用 Middleware
agent = create_agent(
    model="claude-sonnet-4-5-20250929",
    tools=[my_search_tool, my_calculator_tool],
    middleware=[ComprehensiveMiddleware(max_retries=3)]
)
```

### 3.4 函數式 Middleware（裝飾器風格）

```python
from langchain.agents.middleware import wrap_tool_call, wrap_model_call, dynamic_prompt

# 使用裝飾器快速創建 Middleware
@wrap_tool_call
def caching_middleware(request, handler):
    """為工具呼叫加入快取"""
    cache_key = f"{request.tool_call['name']}:{request.tool_call['args']}"
    
    if cached := get_cache(cache_key):
        return ToolMessage(content=cached, tool_call_id=request.tool_call["id"])
    
    result = handler(request)
    save_cache(cache_key, result.content)
    return result


@wrap_model_call
def retry_middleware(request, handler):
    """模型呼叫重試"""
    for attempt in range(3):
        try:
            return handler(request)
        except Exception:
            if attempt == 2:
                raise


@dynamic_prompt
def user_aware_prompt(request: ModelRequest) -> str:
    """動態 System Prompt"""
    user_name = request.runtime.context.get("user_name", "User")
    return f"你是一個專門協助 {user_name} 的 AI 助手。"


# 組合使用
agent = create_agent(
    model="gpt-4o",
    tools=[...],
    middleware=[
        caching_middleware,
        retry_middleware,
        user_aware_prompt
    ]
)
```

-----

## 4. 對比分析：異曲同工之妙

### 4.1 功能對照表

|功能           |Claude Code Hooks               |LangChain Middleware                |
|-------------|--------------------------------|------------------------------------|
|**執行環境**     |Shell/Python 腳本                 |Python 類別/函數                        |
|**前置攔截**     |`PreToolUse`, `UserPromptSubmit`|`before_model()`, `wrap_tool_call()`|
|**後置處理**     |`PostToolUse`, `Stop`           |`after_model()`, `wrap_tool_call()` |
|**阻斷機制**     |Exit Code 2                     |拋出異常 / 返回 `jump_to`                 |
|**狀態管理**     |透過 JSON stdin/stdout            |透過 `AgentState` 物件                  |
|**重試邏輯**     |需自行在腳本中實現                       |內建於 `wrap_*` handler pattern        |
|**動態配置**     |settings.json 靜態配置              |Runtime 動態組合                        |
|**Plugin 系統**|支援（如 Ralph Wiggum）              |透過 Middleware chain                 |

### 4.2 設計模式比較

```
┌─────────────────────────────────────────────────────────────────────┐
│                        設計模式對比                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Claude Code Hooks                   LangChain Middleware           │
│  ════════════════                    ═══════════════════            │
│                                                                     │
│  ┌──────────────┐                    ┌──────────────────────┐       │
│  │ Observer     │                    │ Chain of             │       │
│  │ Pattern      │                    │ Responsibility       │       │
│  │              │                    │                      │       │
│  │ 事件驅動     │                    │ 責任鏈模式            │       │
│  │ 訂閱通知     │                    │ 依序處理              │       │
│  └──────────────┘                    └──────────────────────┘       │
│                                                                     │
│  ┌──────────────┐                    ┌──────────────────────┐       │
│  │ 鬆耦合       │                    │ Decorator Pattern    │       │
│  │              │                    │                      │       │
│  │ 腳本獨立於   │                    │ wrap_* handlers      │       │
│  │ Claude Code  │                    │ 包裝原始行為          │       │
│  └──────────────┘                    └──────────────────────┘       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.3 使用場景建議

|場景             |推薦方案                |原因                |
|---------------|--------------------|------------------|
|**本地開發自動化**    |Claude Code Hooks   |直接整合 Git、Shell 工具鏈|
|**企業 Agent 系統**|LangChain Middleware|更好的可測試性、型別安全      |
|**簡單品質檢查**     |Claude Code Hooks   |低門檻、快速部署          |
|**複雜重試邏輯**     |LangChain Middleware|內建 handler pattern|
|**安全審計**       |兩者皆可                |都支援前置攔截           |
|**自治迴圈**       |Claude Code Hooks   |Ralph Wiggum 模式   |

-----

## 5. Ralph Wiggum Plugin 深度剖析

### 5.1 什麼是 Ralph Wiggum？

Ralph Wiggum 是一個實現 **「持續自治迴圈」** 的 Claude Code Plugin，核心概念來自 Geoffrey Huntley：

> “Ralph is a Bash loop” — 一個簡單的 `while true` 迴圈，反覆餵給 AI Agent 同一個 Prompt，直到任務完成。

名字來自《辛普森家庭》中的 Ralph Wiggum：**不斷犯錯、但永不放棄**。

### 5.2 技術實現：Stop Hook 的巧妙運用

```
┌────────────────────────────────────────────────────────────────────┐
│                    Ralph Wiggum 運作原理                            │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│   1. 用戶執行：                                                     │
│      /ralph-loop "實現功能 X" --max-iterations 20                   │
│                    --completion-promise "DONE"                     │
│                                                                    │
│   2. Claude 開始工作...                                             │
│                                                                    │
│   3. Claude 嘗試結束 ──► Stop Hook 攔截！                           │
│                              │                                     │
│                              ▼                                     │
│   4. 檢查輸出是否包含 "<promise>DONE</promise>"                     │
│              │                                                     │
│      ┌──────┴──────┐                                               │
│      │             │                                               │
│      ▼             ▼                                               │
│   包含 DONE     不包含 DONE                                         │
│      │             │                                               │
│      ▼             ▼                                               │
│   Exit 0        Exit 2                                             │
│   任務完成      阻斷退出！                                           │
│                將原始 Prompt                                        │
│                重新餵給 Claude                                      │
│                    │                                               │
│                    ▼                                               │
│              回到步驟 2（迭代 +1）                                   │
│                                                                    │
│   5. 直到達成 DONE 或 max-iterations 上限                           │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 5.3 Ralph Wiggum 核心程式碼邏輯（概念實現）

```python
#!/usr/bin/env python3
"""
Ralph Wiggum Stop Hook - 概念實現
"""
import json
import sys
import os

# 從環境變數或狀態檔讀取配置
COMPLETION_PROMISE = os.environ.get("RALPH_COMPLETION_PROMISE", "DONE")
MAX_ITERATIONS = int(os.environ.get("RALPH_MAX_ITERATIONS", "20"))
ORIGINAL_PROMPT = os.environ.get("RALPH_ORIGINAL_PROMPT", "")

def load_state():
    """讀取迭代狀態"""
    try:
        with open(".ralph_state.json") as f:
            return json.load(f)
    except FileNotFoundError:
        return {"iteration": 0, "active": False}

def save_state(state):
    """保存迭代狀態"""
    with open(".ralph_state.json", "w") as f:
        json.dump(state, f)

def main():
    # 讀取 Hook 輸入
    input_data = json.load(sys.stdin)
    
    state = load_state()
    
    # 如果 Ralph 迴圈未啟動，正常退出
    if not state.get("active"):
        sys.exit(0)
    
    # 檢查是否達到 completion promise
    # （實際實現會檢查 Claude 的完整輸出）
    claude_output = input_data.get("transcript", "")
    
    if f"<promise>{COMPLETION_PROMISE}</promise>" in claude_output:
        # 任務完成！
        print(f"✅ Ralph Loop 完成！共迭代 {state['iteration']} 次")
        state["active"] = False
        save_state(state)
        sys.exit(0)
    
    # 檢查是否達到迭代上限
    if state["iteration"] >= MAX_ITERATIONS:
        print(f"⚠️ 達到最大迭代次數 {MAX_ITERATIONS}", file=sys.stderr)
        state["active"] = False
        save_state(state)
        sys.exit(0)
    
    # 阻斷退出，重新注入 Prompt
    state["iteration"] += 1
    save_state(state)
    
    # Exit 2 + stderr 內容會被餵回給 Claude
    print(f"""
🔄 Ralph Wiggum 迭代 #{state['iteration']}/{MAX_ITERATIONS}

繼續執行任務：
{ORIGINAL_PROMPT}

上次嘗試未達成目標。請基於已修改的檔案和 Git 歷史繼續工作。
當任務完成時，輸出 <promise>{COMPLETION_PROMISE}</promise>
""", file=sys.stderr)
    
    sys.exit(2)  # 關鍵：Exit 2 阻斷退出

if __name__ == "__main__":
    main()
```

### 5.4 使用 Ralph Wiggum

```bash
# 安裝 Plugin
/plugin ralph-wiggum

# 基本使用
/ralph-loop "將所有測試從 Jest 遷移到 Vitest" \
  --max-iterations 25 \
  --completion-promise "DONE"

# 複雜任務範例
/ralph-loop "實現用戶認證模組。
要求：
- JWT token 驗證
- Password hashing with bcrypt
- Rate limiting
- 測試覆蓋率 > 80%

成功標準：
- 所有測試通過
- 無 Lint 錯誤
- 文檔已更新

完成後輸出 <promise>COMPLETE</promise>" \
  --max-iterations 50 \
  --completion-promise "COMPLETE"

# 取消迴圈
/cancel-ralph
```

### 5.5 Ralph Wiggum 最佳實踐

```
┌─────────────────────────────────────────────────────────────────┐
│                  Ralph Wiggum 使用指南                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ 適合的任務                    ❌ 不適合的任務                 │
│  ─────────────────                ───────────────────           │
│  • 大規模重構                     • 需要人類判斷的決策            │
│  • 測試覆蓋率提升                 • 創意寫作                     │
│  • 框架/依賴遷移                  • 模糊定義的需求               │
│  • 批量文檔更新                   • 涉及敏感操作的任務            │
│  • Bug 修復（有測試驗證）         • 小型簡單任務                  │
│                                                                 │
│  📋 Prompt 撰寫要點                                              │
│  ─────────────────                                              │
│  1. 明確定義「完成」的標準                                       │
│  2. 提供可驗證的成功條件（測試通過、Build 成功）                  │
│  3. 包含失敗時的處理指引                                         │
│  4. 設定合理的 max-iterations                                    │
│                                                                 │
│  💰 成本考量                                                     │
│  ─────────────                                                  │
│  • 50 次迭代 on 大型 codebase = $50-100+ API credits             │
│  • 建議從 max-iterations 10-20 開始                              │
│  • 監控用量，適時調整                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

-----

## 6. 實戰範例對比

### 6.1 場景：工具呼叫監控與日誌

**Claude Code Hooks 實現：**

```python
# .claude/hooks/tool_monitor.py
#!/usr/bin/env python3
import json
import sys
import logging
from datetime import datetime

logging.basicConfig(
    filename=".claude/logs/tool_calls.log",
    level=logging.INFO
)

input_data = json.load(sys.stdin)
tool_name = input_data.get("tool_name")
tool_input = input_data.get("tool_input")

logging.info(f"""
[{datetime.now().isoformat()}]
Tool: {tool_name}
Input: {json.dumps(tool_input, indent=2)}
""")

sys.exit(0)
```

**LangChain Middleware 實現：**

```python
from langchain.agents.middleware import AgentMiddleware
import logging

class ToolMonitorMiddleware(AgentMiddleware):
    def __init__(self):
        self.logger = logging.getLogger("tool_monitor")
    
    def wrap_tool_call(self, request, handler):
        self.logger.info(f"Tool: {request.tool_call['name']}")
        self.logger.info(f"Args: {request.tool_call['args']}")
        
        result = handler(request)
        
        self.logger.info(f"Result: {result.content[:100]}")
        return result
```

### 6.2 場景：安全命令攔截

**Claude Code Hooks 實現：**

```json
// .claude/settings.json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{
          "type": "command",
          "command": "python .claude/hooks/security.py"
        }]
      }
    ]
  }
}
```

```python
# .claude/hooks/security.py
import json, sys

BLOCKED = ["rm -rf /", "sudo rm", "DROP DATABASE"]
data = json.load(sys.stdin)
cmd = data.get("tool_input", {}).get("command", "")

if any(b in cmd for b in BLOCKED):
    print(f"🚫 Blocked: {cmd}", file=sys.stderr)
    sys.exit(2)

sys.exit(0)
```

**LangChain Middleware 實現：**

```python
from langchain.agents.middleware import wrap_tool_call

BLOCKED_COMMANDS = ["rm -rf /", "sudo rm", "DROP DATABASE"]

@wrap_tool_call(tools=["bash_tool"])
def security_middleware(request, handler):
    command = request.tool_call["args"].get("command", "")
    
    for blocked in BLOCKED_COMMANDS:
        if blocked in command:
            raise ValueError(f"Blocked dangerous command: {command}")
    
    return handler(request)
```

-----

## 7. 總結

### 7.1 核心異同

|面向      |Claude Code Hooks  |LangChain Middleware|
|--------|-------------------|--------------------|
|**定位**  |Terminal 開發工具擴展    |企業級 Agent 框架        |
|**執行方式**|外部進程（Shell/Python） |內嵌 Python 程式碼       |
|**配置方式**|JSON 設定檔           |Python 程式碼組合        |
|**學習曲線**|較低（熟悉 Shell 即可）    |中等（需理解 LangChain）   |
|**可測試性**|較低（需模擬 stdin）      |較高（標準 Python 測試）    |
|**生態系統**|Claude Code Plugins|LangChain 生態        |

### 7.2 選擇建議

```
┌────────────────────────────────────────────────────────────────┐
│                     選擇決策樹                                  │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  你在建構什麼？                                                 │
│       │                                                        │
│  ┌────┴────┐                                                   │
│  │         │                                                   │
│  ▼         ▼                                                   │
│ 本地開發   企業應用                                              │
│ 自動化     AI Agent                                             │
│  │         │                                                   │
│  ▼         ▼                                                   │
│ Claude     LangChain                                           │
│ Code       Middleware                                          │
│ Hooks         │                                                │
│  │            │                                                │
│  │         需要自治迴圈？                                        │
│  │            │                                                │
│  │       ┌────┴────┐                                           │
│  │       │         │                                           │
│  │       ▼         ▼                                           │
│  │      是        否                                            │
│  │       │         │                                           │
│  │       ▼         ▼                                           │
│  │   參考 Ralph   標準                                          │
│  │   Wiggum       Middleware                                   │
│  │   模式         即可                                          │
│  │                                                             │
│  ▼                                                             │
│ 需要自治迴圈？ ──► Ralph Wiggum Plugin                          │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 7.3 延伸學習資源

**Claude Code Hooks：**

- 官方文檔：https://docs.claude.com/en/docs/claude-code/hooks
- Hooks 參考：https://code.claude.com/docs/en/hooks
- Ralph Wiggum Plugin：https://github.com/anthropics/claude-code/tree/main/plugins/ralph-wiggum

**LangChain v1.0 Middleware：**

- 官方文檔：https://docs.langchain.com/oss/python/releases/langchain-v1
- Middleware API Reference：https://reference.langchain.com/python/langchain/middleware/
- create_agent 指南：https://docs.langchain.com/

-----

*文檔版本：2026-01-09*
*適用於：Claude Code、LangChain v1.0+*