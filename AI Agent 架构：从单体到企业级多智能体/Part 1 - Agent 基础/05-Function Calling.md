## 核心概念

**三个关键术语：**

- **Tool（工具）**：你提供给模型的功能抽象（如 `get_weather`）
- **Tool Call（工具调用）**：模型决定调用某个工具的请求
- **Tool Call Output（工具输出）**：你执行工具后返回给模型的结果

## 技术实现要点

### 1. 函数定义结构（JSON Schema）

```json
{
  "type": "function",           // 固定值
  "name": "get_weather",        // 工具名
  "description": "获取天气",    // 描述用途
  "parameters": {
    "type": "object",
    "properties": {
      "location": {
        "type": "string",
        "description": "城市名称"
      }
    },
    "required": ["location"],
    "additionalProperties": false  // strict模式必须
  },
  "strict": true                // 推荐开启
}
```

### 2. 完整调用流程（5步）

```
1. 请求 → 模型（携带tools）
2. 模型 ← 工具调用（function_call）
3. 执行代码 → 获取结果
4. 请求 → 模型（携带tool result）
5. 模型 ← 最终响应
```

### 3. 关键配置选项

| 配置                         | 说明                   |
| ---------------------------- | ---------------------- |
| `tool_choice: "auto"`        | 模型决定调用几个       |
| `tool_choice: "required"`    | 必须调用至少一个       |
| `parallel_tool_calls: false` | 禁用并行，只调用一个   |
| `strict: true`               | 强制遵循schema（推荐） |

------

## 实战案例：构建天气查询助手

让我通过一个完整示例，教你实现从定义到执行的完整流程。



```python
import anthropic

client = anthropic.Anthropic()

# 步骤1：定义工具
tools = [
    {
        "type": "function",
        "name": "get_weather",
        "description": "获取指定城市的当前天气",
        "parameters": {
            "type": "object",
            "properties": {
                "location": {
                    "type": "string",
                    "description": "城市名，格式：城市,国家，如：北京，中国"
                },
                "units": {
                    "type": "string",
                    "enum": ["celsius", "fahrenheit"],
                    "description": "温度单位"
                }
            },
            "required": ["location"],
            "additionalProperties": false
        },
        "strict": True
    }
]

# 步骤2：首次请求（触发工具调用）
response = client.messages.create(
    model="claude-opus-4-7",
    max_tokens=1024,
    tools=tools,
    messages=[{
        "role": "user",
        "content": "北京现在天气怎么样？"
    }]
)

# 步骤3：检查是否有工具调用
tool_calls = response.content
for item in tool_calls:
    if item.type == "tool_use":
        # 模型要求调用 get_weather，参数在 item.name 和 item.input 中
        tool_name = item.name
        tool_args = item.input
        call_id = item.id
        print(f"模型要求调用: {tool_name}, 参数: {tool_args}")

# 步骤4：执行工具（模拟）
def execute_weather(location, units="celsius"):
    # 这里调用真实天气API
    return f"{location}当前气温22°C，晴朗"

# 步骤5：执行工具并构建结果消息
tool_results = []
for item in tool_calls:
    if item.type == "tool_use":
        result = execute_weather(**item.input)
        tool_results.append({
            "type": "tool_result",
            "tool_use_id": item.id,
            "content": result
        })

# 步骤6：第二次请求（携带工具结果）
response2 = client.messages.create(
    model="claude-opus-4-7",
    max_tokens=1024,
    tools=tools,
    messages=[
        {"role": "user", "content": "北京现在天气怎么样？"},
        *tool_calls,  # 模型的工具调用
        *tool_results  # 执行结果
    ]
)

print(response2.content[0].text)
# 输出：北京当前气温22°C，晴朗
```

------

## 关键实现细节

### 工具结果的格式



```python
{
    "type": "tool_result",
    "tool_use_id": "toolu_xxx",    # 必须与调用ID匹配
    "content": "结果内容"          # 字符串或图片/文件数组
}
```

### 处理多个并行调用



```python
# 模型可能一次返回多个工具调用
for item in response.content:
    if item.type == "tool_use":
        # 收集所有调用，批量执行，然后一起返回
        pass
```

### 强制指定调用某个工具



```python
tool_choice = {"type": "function", "name": "get_weather"}
# 模型会精确调用指定工具
```

### 流式输出处理



```python
with client.messages.stream(model="claude-opus-4-7", tools=tools, messages=[...]) as stream:
    for event in stream.events:
        if event.type == "response.output_item.added":
            # 新增了一个工具调用项
            pass
        elif event.type == "response.function_call_arguments.delta":
            # 实时收到参数片段
            delta = event.delta
        elif event.type == "response.function_call_arguments.done":
            # 工具调用完成，包含完整参数
            complete_args = event.item.arguments
```

------

## 最佳实践要点

1. **description 要详细** — 告诉模型何时使用、参数格式、输出含义
2. **strict 始终开启** — 保证参数可靠
3. **初始工具控制在 20 个以内** — 避免模型选择困难
4. **不要让模型填充你已知的参数** — 内部处理
5. **使用 namespace 分组** — 便于管理大型工具集

## 生产级的旅行助手案例，讲解完整的 Function Calling 架构与流程。

## 生产级案例：旅行助手 Agent

### 整体架构



```
┌─────────────────────────────────────────────────────────────────┐
│                        用户请求                                    │
│                   "帮我订明天北京到上海的机票"                       │
└─────────────────────────┬───────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                    API Gateway                                   │
│              (认证、限流、日志、路由)                              │
└─────────────────────────┬───────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Agent Orchestrator                              │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  状态机管理  │  对话上下文  │  工具选择策略  │  错误恢复     │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────┬───────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                   工具层 (Tools Layer)                          │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│   │ search   │  │ book     │  │ query    │  │ payment  │        │
│   │ _flights │  │ _flight  │  │ _hotels  │  │ _service │        │
│   └──────────┘  └──────────┘  └──────────┘  └──────────┘        │
└─────────────────────────┬───────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                  外部服务 (External Services)                    │
│       航空公司API   │   酒店API   │   支付网关   │   数据库      │
└─────────────────────────────────────────────────────────────────┘
```

------

## 第一步：生产级函数定义



```python
# tools/flight_tools.py

FLIGHT_TOOLS = [
    {
        "type": "function",
        "name": "search_flights",
        "description": "搜索符合条件的航班列表。此工具用于用户查询航班，不执行预订。建议先搜索再让用户选择，然后单独调用book_flight。",
        "parameters": {
            "type": "object",
            "properties": {
                "departure_city": {
                    "type": "string",
                    "description": "出发城市，格式：城市名或三字机场码，如：北京、PEK"
                },
                "arrival_city": {
                    "type": "string", 
                    "description": "到达城市，格式同出发城市"
                },
                "departure_date": {
                    "type": "string",
                    "description": "出发日期，格式：YYYY-MM-DD，如：2026-05-15"
                },
                "cabin_class": {
                    "type": "string",
                    "enum": ["economy", "business", "first"],
                    "description": "舱位等级，不填则默认经济舱"
                },
                "passengers": {
                    "type": "integer",
                    "description": "乘客人数，默认1人"
                }
            },
            "required": ["departure_city", "arrival_city", "departure_date"],
            "additionalProperties": False
        },
        "strict": True
    },
    {
        "type": "function",
        "name": "book_flight",
        "description": "预订航班座位。当用户确认航班后调用此工具完成预订。预订前需确认：1) 用户已选择航班 2) 用户已确认价格。预订成功返回订单号。",
        "parameters": {
            "type": "object",
            "properties": {
                "flight_id": {
                    "type": "string",
                    "description": "航班ID，来自search_flights的返回结果"
                },
                "passenger_info": {
                    "type": "object",
                    "description": "乘客信息",
                    "properties": {
                        "name": {"type": "string", "description": "乘客姓名"},
                        "id_type": {"type": "string", "enum": ["id_card", "passport"]},
                        "id_number": {"type": "string"},
                        "phone": {"type": "string"}
                    },
                    "required": ["name", "id_type", "id_number", "phone"]
                },
                "seat_preference": {
                    "type": "string",
                    "enum": ["window", "aisle", "none"],
                    "description": "座位偏好"
                }
            },
            "required": ["flight_id", "passenger_info"],
            "additionalProperties": False
        },
        "strict": True
    },
    {
        "type": "function",
        "name": "get_booking_detail",
        "description": "查询已有订单的详细信息。用户需要提供订单号。用于确认预订状态、查看行程细节。",
        "parameters": {
            "type": "object",
            "properties": {
                "booking_id": {
                    "type": "string",
                    "description": "订单号，格式：BK+时间戳+随机数"
                }
            },
            "required": ["booking_id"],
            "additionalProperties": False
        },
        "strict": True
    }
]
```

------

## 第二步：工具执行器（生产级）



```python
# core/tool_executor.py

import json
import logging
from typing import Any, Dict, List
from dataclasses import dataclass
from enum import Enum

logger = logging.getLogger(__name__)

class ToolStatus(Enum):
    SUCCESS = "success"
    FAILED = "failed"
    PARTIAL = "partial"

@dataclass
class ToolResult:
    tool_use_id: str
    tool_name: str
    status: ToolStatus
    content: str | Dict
    error: str | None = None
    execution_time_ms: int = 0

class ToolExecutor:
    """生产级工具执行器"""
    
    def __init__(self):
        self._handlers = {}
        self._timeout = 30  # 超时30秒
        self._max_retries = 2
    
    def register(self, tool_name: str, handler):
        """注册工具处理函数"""
        self._handlers[tool_name] = handler
    
    async def execute(
        self, 
        tool_calls: List[Dict]
    ) -> List[ToolResult]:
        """
        批量执行工具调用
        生产环境支持并行执行
        """
        results = []
        
        # 并行执行所有工具调用
        import asyncio
        tasks = [self._execute_single(call) for call in tool_calls]
        results = await asyncio.gather(*tasks, return_exceptions=True)
        
        # 处理结果
        processed = []
        for i, result in enumerate(results):
            if isinstance(result, Exception):
                processed.append(ToolResult(
                    tool_use_id=tool_calls[i].get("id", ""),
                    tool_name=tool_calls[i].get("name", "unknown"),
                    status=ToolStatus.FAILED,
                    content="",
                    error=str(result)
                ))
            else:
                processed.append(result)
        
        return processed
    
    async def _execute_single(self, tool_call: Dict) -> ToolResult:
        """执行单个工具调用"""
        import time
        start = time.time()
        
        tool_name = tool_call.get("name")
        arguments = tool_call.get("arguments", {})
        
        # 重试逻辑
        last_error = None
        for attempt in range(self._max_retries + 1):
            try:
                handler = self._handlers.get(tool_name)
                if not handler:
                    return ToolResult(
                        tool_use_id=tool_call.get("id", ""),
                        tool_name=tool_name,
                        status=ToolStatus.FAILED,
                        content="",
                        error=f"Tool '{tool_name}' not registered"
                    )
                
                # 调用处理器
                result = await asyncio.wait_for(
                    handler(**arguments),
                    timeout=self._timeout
                )
                
                return ToolResult(
                    tool_use_id=tool_call.get("id", ""),
                    tool_name=tool_name,
                    status=ToolStatus.SUCCESS,
                    content=result,
                    execution_time_ms=int((time.time() - start) * 1000)
                )
                
            except asyncio.TimeoutError:
                last_error = f"Timeout after {self._timeout}s"
            except Exception as e:
                last_error = str(e)
                logger.warning(f"Tool {tool_name} attempt {attempt+1} failed: {e}")
        
        return ToolResult(
            tool_use_id=tool_call.get("id", ""),
            tool_name=tool_name,
            status=ToolStatus.FAILED,
            content="",
            error=f"Failed after {self._max_retries + 1} attempts: {last_error}"
        )
```

------

## 第三步：实际工具实现



```python
# tools/flight_handler.py

import httpx
import logging

logger = logging.getLogger(__name__)

class FlightService:
    """航班服务封装"""
    
    def __init__(self, api_base: str, api_key: str):
        self.client = httpx.AsyncClient(
            base_url=api_base,
            headers={"Authorization": f"Bearer {api_key}"},
            timeout=httpx.Timeout(10.0)
        )
    
    async def search(self, departure_city: str, arrival_city: str, 
                     departure_date: str, cabin_class: str = "economy",
                     passengers: int = 1) -> Dict:
        """
        调用外部航班搜索API
        """
        try:
            # 实际调用外部API
            response = await self.client.post("/v1/flights/search", json={
                "from": departure_city,
                "to": arrival_city,
                "date": departure_date,
                "cabin": cabin_class,
                "count": passengers
            })
            response.raise_for_status()
            data = response.json()
            
            # 格式化返回结果
            return self._format_flights(data)
            
        except httpx.HTTPStatusError as e:
            logger.error(f"Flight API error: {e.response.status_code}")
            raise RuntimeError(f"航班查询失败，请稍后重试")
        except Exception as e:
            logger.exception("Flight search failed")
            raise RuntimeError("航班服务异常")
    
    def _format_flights(self, raw_data: Dict) -> Dict:
        """标准化返回格式"""
        flights = []
        for f in raw_data.get("flights", []):
            flights.append({
                "flight_id": f["id"],
                "airline": f["airline"],
                "flight_number": f["number"],
                "departure": {
                    "city": f["departure"]["city"],
                    "airport": f["departure"]["airport"],
                    "time": f["departure"]["time"]
                },
                "arrival": {
                    "city": f["arrival"]["city"],
                    "airport": f["arrival"]["airport"],
                    "time": f["arrival"]["time"]
                },
                "price": {
                    "amount": f["price"]["CNY"],
                    "currency": "CNY"
                },
                "seats_available": f.get("seats", 0)
            })
        
        return {
            "success": True,
            "count": len(flights),
            "flights": flights
        }

# 全局单例
flight_service = FlightService(
    api_base="https://api.flights.example.com",
    api_key="your-api-key"
)
```

------

## 第四步：Agent 核心逻辑



```python
# core/agent.py

import anthropic
from typing import List, Dict, Optional
from enum import Enum
from dataclasses import dataclass, field
import json

class AgentState(Enum):
    IDLE = "idle"
    WAITING_TOOL_RESULT = "waiting_tool_result"
    RESPONDING = "responding"
    ERROR = "error"

@dataclass
class ConversationTurn:
    role: str
    content: str
    tool_calls: List[Dict] = field(default_factory=list)
    tool_results: List[Dict] = field(default_factory=list)

@dataclass
class AgentContext:
    user_id: str
    session_id: str
    turns: List[ConversationTurn] = field(default_factory=list)
    state: AgentState = AgentState.IDLE
    variables: Dict = field(default_factory=dict)  # 存储中间状态

class TravelAgent:
    """
    旅行助手 Agent - 生产级实现
    """
    
    SYSTEM_PROMPT = """你是一个专业的旅行助手，帮助用户查询和预订航班。
    
    工作流程：
    1. 当用户需要查询航班时，先调用 search_flights
    2. 将搜索结果以清晰格式展示给用户
    3. 等待用户确认航班后，询问乘客信息
    4. 用户确认后调用 book_flight 完成预订
    5. 预订成功后，提供订单号和行程确认信息
    
    重要原则：
    - 每次只展示最相关的3-5个航班选择
    - 价格必须清晰标注币种
    - 预订前必须得到用户明确确认
    - 遇到错误要友好地告知用户并提供替代方案
    
    当前用户意图：{intent}
    已收集信息：{collected_info}
    """
    
    def __init__(self, client: anthropic.Anthropic, executor: ToolExecutor):
        self.client = client
        self.executor = executor
        self.model = "claude-opus-4-7"
    
    async def process(
        self, 
        user_message: str, 
        context: AgentContext
    ) -> tuple[str, AgentContext]:
        """
        处理用户消息，返回 (响应内容, 更新后的上下文)
        """
        
        # 1. 构建消息历史
        messages = self._build_messages(context, user_message)
        
        # 2. 构造系统提示（含上下文）
        system = self._build_system_prompt(context)
        
        # 3. 首次请求 LLM
        response = await self._call_llm(system, messages)
        
        # 4. 处理响应（可能是最终回复，也可能是工具调用）
        final_response, tool_calls = self._parse_response(response)
        
        # 5. 如果有工具调用，执行并递归
        if tool_calls:
            context.state = AgentState.WAITING_TOOL_RESULT
            
            # 执行工具
            tool_results = await self.executor.execute(tool_calls)
            
            # 记录本次对话
            context.turns.append(ConversationTurn(
                role="user",
                content=user_message,
                tool_calls=tool_calls
            ))
            
            # 追加工具结果到消息
            messages.extend(self._format_tool_results(tool_results))
            
            # 再次调用 LLM（携带工具结果）
            response2 = await self._call_llm(system, messages)
            final_response, _ = self._parse_response(response2)
            
            context.turns.append(ConversationTurn(
                role="assistant",
                content=final_response
            ))
        
        context.state = AgentState.IDLE
        return final_response, context
    
    def _build_system_prompt(self, context: AgentContext) -> str:
        """构建带有上下文的系统提示"""
        return self.SYSTEM_PROMPT.format(
            intent=context.variables.get("intent", "未知"),
            collected_info=json.dumps(context.variables.get("collected", {}), ensure_ascii=False)
        )
    
    def _build_messages(self, context: AgentContext, new_message: str) -> List[Dict]:
        """构建消息历史"""
        messages = []
        
        for turn in context.turns[-5:]:  # 只保留最近5轮，节省token
            if turn.role == "user":
                messages.append({"role": "user", "content": turn.content})
            else:
                messages.append({"role": "assistant", "content": turn.content})
        
        messages.append({"role": "user", "content": new_message})
        return messages
    
    async def _call_llm(self, system: str, messages: List[Dict]) -> anthropic.Message:
        """调用 LLM"""
        all_tools = FLIGHT_TOOLS  # 导入的工具定义
        
        return await self.client.messages.create(
            model=self.model,
            max_tokens=2048,
            system=system,
            tools=all_tools,
            messages=messages
        )
    
    def _parse_response(self, response: anthropic.Message) -> tuple[str, List[Dict]]:
        """解析 LLM 响应，区分最终回复和工具调用"""
        final_text_parts = []
        tool_calls = []
        
        for block in response.content:
            if block.type == "text":
                final_text_parts.append(block.text)
            elif block.type == "tool_use":
                tool_calls.append({
                    "id": block.id,
                    "name": block.name,
                    "arguments": block.input
                })
        
        final_text = "\n".join(final_text_parts)
        return final_text, tool_calls
    
    def _format_tool_results(self, results: List[ToolResult]) -> List[Dict]:
        """格式化工具结果用于下一轮对话"""
        formatted = []
        for r in results:
            if r.status == ToolStatus.SUCCESS:
                content = r.content if isinstance(r.content, str) else json.dumps(r.content, ensure_ascii=False)
            else:
                content = f"执行失败：{r.error}"
            
            formatted.append({
                "role": "user",
                "content": [
                    {
                        "type": "tool_result",
                        "tool_use_id": r.tool_use_id,
                        "content": content
                    }
                ]
            })
        return formatted
```

------

## 第五步：完整请求流程（按时间顺序）



```
┌─────────────────────────────────────────────────────────────────────────┐
│                           用户                                             │
│                    "帮我订明天北京到上海的机票"                               │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           Turn 1                                          │
│                                                                         │
│  请求:                                                                   │
│  {                                                                      │
│    "model": "claude-opus-4-7",                                          │
│    "tools": [search_flights, book_flight, get_booking_detail],          │
│    "messages": [{"role": "user", "content": "帮我订明天北京到上海的机票"}] │
│  }                                                                      │
│                                    │                                     │
│                                    ▼                                     │
│  响应:                                                                   │
│  {                                                                      │
│    "content": [                                                         │
│      {                                                                  │
│        "type": "tool_use",                                              │
│        "id": "toolu_001",                                               │
│        "name": "search_flights",                                        │
│        "input": {                                                       │
│          "departure_city": "北京",                                     │
│          "arrival_city": "上海",                                       │
│          "departure_date": "2026-05-03"                               │
│        }                                                               │
│      }                                                                  │
│    ]                                                                    │
│  }                                                                      │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                         检测到工具调用 → 执行工具
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           Turn 2                                          │
│                                                                         │
│  请求:                                                                   │
│  {                                                                      │
│    "model": "claude-opus-4-7",                                          │
│    "tools": [...],                                                      │
│    "messages": [                                                        │
│      {"role": "user", "content": "帮我订明天北京到上海的机票"},             │
│      {                                                                  │
│        "type": "tool_use",                                              │
│        "id": "toolu_001",                                               │
│        "name": "search_flights",                                        │
│        "input": {...}                                                  │
│      },                                                                 │
│      {                                                                  │
│        "role": "user",                                                  │
│        "content": [{                                                    │
│          "type": "tool_result",                                         │
│          "tool_use_id": "toolu_001",                                    │
│          "content": '{"success": true, "count": 5, "flights": [...]}'   │
│        }]                                                               │
│      }                                                                  │
│    ]                                                                    │
│  }                                                                      │
│                                    │                                     │
│                                    ▼                                     │
│  响应:                                                                   │
│  {                                                                      │
│    "content": [                                                         │
│      {                                                                  │
│        "type": "text",                                                  │
│        "text": "找到以下航班：\n\n1. 国航CA123 08:00-10:30 ¥680\n2. 东航MU..." │
│      }                                                                  │
│    ]                                                                    │
│  }                                                                      │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                              展示给用户
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           用户选择                                        │
│                     "我要第2个，东航MU5678"                               │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           Turn 3                                          │
│                                                                         │
│  请求:                                                                   │
│  {                                                                      │
│    "messages": [                                                        │
│      ...(历史消息),                                                       │
│      {"role": "user", "content": "我要第2个，东航MU5678"}                 │
│    ]                                                                    │
│  }                                                                      │
│                                    │                                     │
│                                    ▼                                     │
│  响应:                                                                   │
│  {                                                                      │
│    "content": [                                                         │
│      {                                                                  │
│        "type": "text",                                                  │
│        "text": "好的，您选择了东航MU5678，经济舱 ¥720。请提供乘客信息：姓名、证件类型、证件号码、手机号。" │
│      }                                                                  │
│    ]                                                                    │
│  }                                                                      │
│                                    │                                     │
│                         ⚠ 无工具调用 - 询问补充信息                        │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           用户                                            │
│               "张三，身份证，110101199001011234，13800138000"               │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           Turn 4                                          │
│                                                                         │
│  请求消息历史累积：                                                       │
│  - 用户：帮我订明天北京到上海的机票                                        │
│  - Assistant：找到以下航班...（展示5个选项）                               │
│  - 用户：我要第2个，东航MU5678                                            │
│  - Assistant：请提供乘客信息                                             │
│  - 用户：张三，身份证...                                                  │
│                                    │                                     │
│                                    ▼                                     │
│  响应:                                                                   │
│  {                                                                      │
│    "content": [                                                         │
│      {                                                                  │
│        "type": "tool_use",                                              │
│        "id": "toolu_002",                                               │
│        "name": "book_flight",                                           │
│        "input": {                                                       │
│          "flight_id": "MU5678_20260503",                                │
│          "passenger_info": {                                           │
│            "name": "张三",                                              │
│            "id_type": "id_card",                                        │
│            "id_number": "110101199001011234",                           │
│            "phone": "13800138000"                                       │
│          }                                                              │
│        }                                                               │
│      }                                                                  │
│    ]                                                                    │
│  }                                                                      │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                         执行 book_flight 工具
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           Turn 5                                          │
│                                                                         │
│  请求:                                                                   │
│  {                                                                      │
│    "messages": [                                                        │
│      ...(历史消息),                                                       │
│      {"role": "assistant", "content": [{"type": "tool_use", ...}]}      │
│      {"role": "user", "content": [{"type": "tool_result", "content":     │
│        "{"success": true, "booking_id": "BK20260503001", "confirmation":  │
│        "东航MU5678，经济舱，张三，2026-05-03 14:00"}"}]}                   │
│    ]                                                                    │
│  }                                                                      │
│                                    │                                     │
│                                    ▼                                     │
│  响应:                                                                   │
│  {                                                                      │
│    "content": [                                                         │
│      {                                                                  │
│        "type": "text",                                                  │
│        "text": "✅ 预订成功！\n\n订单号：BK20260503001\n航班：东航MU5678\n│        日期：2026-05-03 14:00\n乘客：张三\n舱位：经济舱\n\n请提前90分钟到机场值机。"│
      }                                                                  │
│    ]                                                                    │
│  }                                                                      │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                              最终响应 → 用户
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           流程结束                                        │
│                                                                         │
│  状态保存：                                                              │
│  context.variables = {                                                  │
│    "last_booking_id": "BK20260503001",                                  │
│    "last_flight": "MU5678",                                             │
│    "last_passenger": "张三"                                            │
│  }                                                                      │
└─────────────────────────────────────────────────────────────────────────┘
```

------

## 生产级关键设计说明

### 1. 状态管理为什么这样设计



```python
context.variables = {
    "intent": "booking_flight",      # 当前用户意图
    "selected_flight": {...},         # 用户选择的航班
    "passenger_info": {...},          # 已收集的乘客信息
    "pending_confirmation": True      # 等待用户确认
}
```

**为什么：** 当工具调用完成后，我们需要知道下一步该做什么。如果用户说"确认"，Agent 需要知道确认什么。

### 2. 消息历史为什么要裁剪



```python
for turn in context.turns[-5:]:  # 只保留最近5轮
```

**为什么：** Token 限制和成本控制。航班搜索结果可能很大，全部历史会很快超出上下文窗口。

### 3. 工具结果为什么要 JSON 字符串



```python
content = json.dumps(result, ensure_ascii=False)
```

**为什么：** 避免格式问题，保持传输一致性。模型能解析各种格式，但字符串最可靠。

### 4. 错误处理流程



```
工具执行失败
    │
    ├─→ 超时重试 (最多2次)
    │       │
    │       ├─→ 成功 → 继续流程
    │       └─→ 失败 → 返回友好错误消息
    │
    └─→ 立即返回用户友好消息
            "抱歉，航班查询暂时不可用，请稍后重试"
```

### 5. 关键的生产考量

| 设计点       | 说明                              |
| ------------ | --------------------------------- |
| **幂等性**   | 工具执行设计为幂等，失败可重试    |
| **超时控制** | 30秒超时，防止阻塞                |
| **并行执行** | 多个工具调用同时执行              |
| **状态恢复** | context 保存中间状态，断线可恢复  |
| **日志追踪** | 每个 tool_use_id 可追踪完整调用链 |