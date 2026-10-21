---
title: "状态不再藏在协议里：MCP 2026-07-28 之后，AI 应用进生产要补的四件事"
date: 2026-10-21T09:20:29+08:00
draft: false
tags: ["MCP", "AI 应用", "协议", "工程化", "Python"]
categories: ["技术"]
description: "MCP 2026-07-28 把状态从协议里拿掉，靠 session 偷懒的集成会成批坏掉；用四个可运行的例子，讲清状态归属、错误分类、缓存新鲜度和权限边界这四笔账怎么补。"
---

## 一、一个内部助手是怎么坏掉的

去年底你给运营团队写了个内部助手：八个工具，查订单、改地址、发通知、退款。demo 阶段一切正常，因为 demo 只有你一个人在用。推给二十人的内部群之后，四件事同时冒出来。

- 网关做了轮询负载均衡，第二个请求落到另一台实例上。`Mcp-Session-Id` 对不上，工具列表变了，客户端还在用旧的 schema。
- 改地址那次 HTTP 超时，调用层重试了一次，用户的收货地址被改了两次。第二次覆盖第一次，没人发现。
- 排障时日志里只有一个 `500`。分不清是模型给错了参数、凭据过期，还是上游订单系统在超时。
- 给只读账号签发的凭据，拿到的是全量工具 schema，包括退款。

这四件事分别对应状态归属、失败分类、可观测性、权限边界。MIT NANDA《The GenAI Divide: State of AI in Business 2025》给的分层是：企业级 GenAI 系统里 60% 评估过、20% 走到试点、只有 5% 进了生产，方法为 150 场企业负责人访谈加 350 份员工问卷加 300 个公开部署案例分析。注意口径，它说的是企业级定制或采购型 GenAI，不是「所有 AI 项目的失败率」。

我的判断是：这 5% 与剩下 95% 的分界线基本不在模型能力上，而在工具层这四笔账有没有结。

## 二、协议把状态拿掉了，责任没拿走

MCP 规范 `2026-07-28` 已从 release candidate 转为正式版，`specification/latest` 当前指向它，上一版是 `2025-11-25`，RC 锁定于 2026-05-21。官方 changelog 里 major changes 是 **9 条**（流传较广的「8 条」版本漏了移除 SSE 可恢复性那一条）。与落地直接相关的是四条。

**删掉握手。** `initialize` 与 `notifications/initialized` 不再存在，协议版本和客户端能力改为每个请求的 `_meta` 携带，键名固定为 `io.modelcontextprotocol/protocolVersion` 与 `io.modelcontextprotocol/clientCapabilities`；客户端 SHOULD 每请求带 `clientInfo`，服务端 SHOULD 在结果的 `_meta` 里回 `serverInfo`。版本不匹配返回 `UnsupportedProtocolVersionError`，错误码 `-32022`，HTTP `400`。

**删掉协议级会话。** `Mcp-Session-Id` 移除，`tools/list`、`resources/list`、`prompts/list` 明确不再 per-connection。

**新增 `server/discover`。** 它是 MUST 实现：请求除标准 `_meta` 外没有参数，响应 `DiscoverResult` 带 `supportedVersions`、`capabilities`、`_meta['io.modelcontextprotocol/serverInfo']` 和 `instructions`，并且支持缓存。在 STDIO 上它同时充当向后兼容探针，现代客户端先发它，收到「非现代错误」才回落到 `initialize`。

**让网关能在不看 body 的情况下工作。** `Mcp-Method`（所有请求）与 `Mcp-Name`（`tools/call`、`resources/read`、`prompts/get`）是 REQUIRED，`MCP-Protocol-Version` 头 MUST 与 body 里的 `_meta` 一致；头与 body 不一致或缺头，返回 HTTP `400` 加 JSON-RPC `-32020` `HeaderMismatch`。这里有个容易写错的细节：这条属于 minor 变更（SEP-2243），不要算进那 9 条 major 里。

官方发布博客有一句可以直接当判据的话：「A remote MCP server that previously needed sticky sessions, a shared session store, and deep packet inspection at the gateway can now run behind a plain round-robin load balancer, route traffic on an `Mcp-Method` header, and let clients cache `tools/list` responses for as long as the server's `ttlMs` permits.」

省掉了什么运维件，这句话说得很清楚：粘性会话、共享 session store、网关上的深度包解析。代价是状态得由你自己建模。

还有两条契约变更值得单独记住。其一是 `subscriptions/listen`：它取代了 HTTP GET 端点和 `resources/subscribe` / `unsubscribe`，客户端用 `toolsListChanged`、`promptsListChanged`、`resourcesListChanged`、`resourceSubscriptions` 这些字段逐项 opt-in，服务端 MUST 先发一条 `notifications/subscriptions/acknowledged`，这条流上所有消息都带 `_meta['io.modelcontextprotocol/subscriptionId']`，其值就是 `subscriptions/listen` 请求的 JSON-RPC id；`notifications/progress` 与 `notifications/message` 不走这条流，仍旧挂在各自请求的响应流上。其二是 `resultType` 成为所有结果的必填字段（`"complete"` 或 `"input_required"`），而旧服务端缺这个字段时客户端 MUST 按 `complete` 处理。这两条合起来意味着：你的客户端要把「不知道协议版本的对方」当成一等公民，否则升级当天会同时坏在新老两端。

Streamable HTTP 还关掉了 SSE 的可恢复性：`Last-Event-ID` 和事件 id 移除，流断了请求就丢了，客户端 MUST 用新的 id 重发；旧客户端打来的 GET 和 DELETE 返回 `405`，带 `Mcp-Session-Id` 或 `Last-Event-ID` 的服务端忽略不报错。这条对排障心态有直接影响，断线重连从「续传」变成「重来」，重来就意味着幂等性必须做对。

## 三、状态搬到应用层：显式句柄加 MRTR

最小可跑的形态长这样，本机实测环境是 Python 3.14.7 加 `mcp==2.0.0`：

```python
# A_stateless_roundtrip.py
import asyncio
from typing import TypedDict

from mcp.server import MCPServer, CacheHint
from mcp import Client


class Basket(TypedDict):
    basket_id: str


srv = MCPServer(
    "order-bot",
    instructions="订单助手的工具层，无状态",
    cache_hints={"tools/list": CacheHint(ttl_ms=60_000, scope="public")},
)


@srv.tool()
def create_basket(customer_id: str) -> Basket:
    """新建购物车，返回显式句柄 basket_id。"""
    return {"basket_id": f"basket-{customer_id}-001"}


@srv.tool()
def add_item(basket_id: str, sku: str, qty: int) -> Basket:
    """往指定购物车里加货。"""
    return {"basket_id": f"{basket_id}+{sku}x{qty}"}


async def main() -> None:
    async with Client(srv) as c:                      # 没有握手，首个请求就带 _meta
        print("protocol_version:", c.protocol_version)
        tools = await c.list_tools()
        print("tools/list ttl_ms/scope:", tools.ttl_ms, tools.cache_scope)
        bid = (await c.call_tool("create_basket", {"customer_id": "c-42"})).structured_content
        print("create_basket ->", bid)
        r = await c.call_tool("add_item", {"basket_id": bid["basket_id"], "sku": "SKU-1", "qty": 2})
        print("add_item ->", r.structured_content, "| result_type:", r.result_type)


asyncio.run(main())
```

真实输出：

```text
protocol_version: 2026-07-28
tools/list ttl_ms/scope: 60000 public
create_basket -> {'basket_id': 'basket-c-42-001'}
add_item -> {'basket_id': 'basket-c-42-001+SKU-1x2'} | result_type: complete
```

第一行就是「无状态」的证据：没有握手请求，客户端第一个请求直接带着 `_meta` 进来，协商结果就是 `2026-07-28`。`create_basket` 返回的 `basket_id` 是显式句柄，它出现在工具结果里，模型看得见、能传下去、能写进日志。以前这一层状态往往藏在 session 里，模型既推理不了也审计不了。

一个实测到的坑：返回类型标注决定 `structured_content`。标 `-> dict` 时客户端拿到的 `structured_content` 是 `None`，只有 `content` 里那段文本 JSON；标 `TypedDict` 或 pydantic 模型才会有结构化输出。想让模型拿到可校验的结果，别偷懒写 `dict`。

真正需要问用户一句的时候，`2026-07-28` 的做法是 MRTR，多轮往返请求：

```python
# B_mrtr.py
import asyncio
from typing import Annotated, TypedDict

from pydantic import BaseModel
from mcp.server import MCPServer, CacheHint
from mcp.server.mcpserver import Context, Elicit, RequestStateSecurity, Resolve
from mcp import Client
import mcp.types as types

STATE_KEY = "5f0c9a1d3b7e4a2c8d6f0b1e4a7c9d2f"  # >= 32 字节，生产从 KMS 或环境变量读


class Basket(TypedDict):
    basket_id: str


class Refund(TypedDict):
    basket_id: str
    refunded_cny: int
    reason: str


class ConfirmRefund(BaseModel):
    confirm: bool


srv = MCPServer(
    "order-bot",
    cache_hints={"tools/list": CacheHint(ttl_ms=60_000, scope="public")},
    request_state_security=RequestStateSecurity(keys=[STATE_KEY], ttl=300.0),
)


@srv.tool()
def create_basket(customer_id: str) -> Basket:
    """新建购物车，返回显式句柄 basket_id。"""
    return {"basket_id": f"basket-{customer_id}-001"}


async def ask_refund_confirmation(
    ctx: Context, basket_id: str, amount_cny: int
) -> Elicit[ConfirmRefund]:
    """Resolver：声明这次调用需要一次人工确认，怎么问由框架按协议版本决定。"""
    print("   [server] 第 N 轮，声明需要确认，request_id =", ctx.request_id)
    return Elicit(f"确认对 {basket_id} 退款 {amount_cny} 元？", ConfirmRefund)


@srv.tool()
async def refund(
    basket_id: str,
    amount_cny: int,
    answer: Annotated[ConfirmRefund, Resolve(ask_refund_confirmation)],
) -> Refund:
    """退款；需要人工确认后才执行。"""
    if not answer.confirm:
        return {"basket_id": basket_id, "refunded_cny": 0, "reason": "declined"}
    return {"basket_id": basket_id, "refunded_cny": amount_cny, "reason": "accepted"}


async def on_elicit(context, params):
    """客户端补信息：生产里是弹确认框，agent 场景也可以按策略自动答。"""
    print("   [client] 收到 inputRequests 里的 elicit:", params.message)
    return types.ElicitResult(action="accept", content={"confirm": True})


async def main() -> None:
    async with Client(srv, elicitation_callback=on_elicit) as c:
        print("protocol_version:", c.protocol_version)
        tl = await c.list_tools()
        print("tools/list ttl_ms/scope:", tl.ttl_ms, tl.cache_scope)
        bid = (await c.call_tool("create_basket", {"customer_id": "c-42"})).structured_content["basket_id"]
        print("basket_id:", bid)
        r = await c.call_tool("refund", {"basket_id": bid, "amount_cny": 300})
        print("refund ->", r.structured_content, "| result_type:", r.result_type, "| is_error:", r.is_error)


asyncio.run(main())
```

真实输出：

```text
protocol_version: 2026-07-28
tools/list ttl_ms/scope: 60000 public
basket_id: basket-c-42-001
   [server] 第 N 轮，声明需要确认，request_id = 4
   [client] 收到 inputRequests 里的 elicit: 确认对 basket-c-42-001 退款 300 元？
   [server] 第 N 轮，声明需要确认，request_id = 5
refund -> {'basket_id': 'basket-c-42-001', 'refunded_cny': 300, 'reason': 'accepted'} | result_type: complete | is_error: False
```

两个 `request_id`（4 到 5）是这一节的重点：第一次调用返回 `resultType: "input_required"` 的 `InputRequiredResult`，带上 `inputRequests` 和一个不透明的 `requestState`；客户端补完信息后用新的 id 重发原请求，携带 `inputResponses` 并把 `requestState` 原样回传。

这里有个很容易踩的坑：在 `2026-07-28` 连接上直接 `await ctx.elicit(...)` 会失败，报 `NoBackChannelError: this transport context has no back-channel for server-initiated requests`。原因写在 SDK 源码注释里，现代连接上装的是 `NotifyOnlyOutbound`，「the modern protocol forbids server-initiated JSON-RPC requests, so `send_raw_request` refuses by construction」。服务端不能再主动发请求，只能由客户端带着 `inputResponses` 回来。SDK 的 resolver 文档把两种传输说得很直白：`>= 2026-07-28` 把请求批量塞进 `InputRequiredResult`，`<= 2025-11-25` 仍在调用中途逐个发服务端到客户端的请求。写业务代码时用 `Annotated[T, Resolve(fn)]` 声明依赖，让框架去挑传输方式，比手写 `ctx.elicit` 稳。

报文层面再看一眼，顺便做一次篡改测试：

```python
# C_wire.py
import asyncio
from typing import Annotated, TypedDict

from pydantic import BaseModel
from mcp.server import MCPServer
from mcp.server.mcpserver import Context, Elicit, RequestStateSecurity, Resolve
from mcp import Client
import mcp.types as types

KEY = "5f0c9a1d3b7e4a2c8d6f0b1e4a7c9d2f"


class Refund(TypedDict):
    basket_id: str
    refunded_cny: int


class Confirm(BaseModel):
    confirm: bool


srv = MCPServer("wire-demo", request_state_security=RequestStateSecurity(keys=[KEY], ttl=300.0))


async def ask(ctx: Context, basket_id: str, amount_cny: int) -> Elicit[Confirm]:
    return Elicit(f"确认对 {basket_id} 退款 {amount_cny} 元？", Confirm)


@srv.tool()
async def refund(basket_id: str, amount_cny: int,
                 answer: Annotated[Confirm, Resolve(ask)]) -> Refund:
    return {"basket_id": basket_id, "refunded_cny": amount_cny if answer.confirm else 0}


async def decline(context, params):
    return types.ElicitResult(action="decline")


async def main() -> None:
    # elicitation_callback 的存在本身就是「声明具备 form elicitation 能力」
    async with Client(srv, elicitation_callback=decline) as c:
        args = {"basket_id": "basket-c-42-001", "amount_cny": 300}
        first = await c.session.call_tool("refund", args, allow_input_required=True)
        print("round-1 class:", type(first).__name__)
        print("result_type:", first.result_type)
        print("input_requests keys:", list(first.input_requests.keys()))
        print("request_state:", str(first.request_state)[:24], "...", f"(len={len(str(first.request_state))})")

        key = next(iter(first.input_requests))
        ok = types.ElicitResult(action="accept", content={"confirm": True})
        second = await c.session.call_tool(
            "refund", args,
            input_responses={key: ok},
            request_state=first.request_state,
            allow_input_required=True,
        )
        print("round-2 class:", type(second).__name__, "| structured:", second.structured_content)

        try:
            await c.session.call_tool(
                "refund", args,
                input_responses={key: ok},
                request_state=str(first.request_state)[:-4] + "AAAA",
                allow_input_required=True,
            )
            print("tampered: 被接受了（不该发生）")
        except Exception as e:
            print("tampered ->", type(e).__name__, "|", e)


asyncio.run(main())
```

真实输出：

```text
round-1 class: InputRequiredResult
result_type: input_required
input_requests keys: ['__main__:ask']
request_state: v1.owEyPuQYBxsUIW2HTUPue ... (len=342)   # 前缀随 nonce 变化，这是某次真实输出
round-2 class: CallToolResult | structured: {'basket_id': 'basket-c-42-001', 'refunded_cny': 300}
tampered -> MCPError | Invalid or expired requestState
```

`request_state` 是 342 字符、`v1.` 开头的不透明串，改掉尾部四个字符就被拒，服务端 stderr 会多出一行 `requestState rejected on tools/call: seal`。规范侧的硬要求在这里：`requestState` 客户端 MUST NOT 解析；服务端 MUST 把它当成攻击者可控输入，凡是影响授权、资源访问或业务逻辑的用法 MUST 做完整性保护（HMAC 或 AEAD），并 SHOULD 在受保护载荷里放 principal、短 TTL 和原请求标识来防重放。只有 `prompts/get`、`resources/read`、`tools/call` 能返回这个结果。SDK 给的是 AES-256-GCM 编解码器，文档原话是「Tokens are encrypted, not merely signed, so clients cannot read the state」，配置入口为 `RequestStateSecurity(keys=[...], ttl=300.0)`，实测密钥少于 32 字节会直接抛 `ValueError`。

规范对这套模式的运维评价是「without requiring a shared storage layer across server instances or requiring stateful load balancing」。我的判断是：你省掉的是粘性会话和共享 session store，换来的是一个必须管好的密钥环，轮换、TTL、principal 绑定一个都不能少。密钥没管好，`requestState` 就成了伪造上下文的入口。

## 四、四条生产护栏

**错误分类与重试。** 把工具失败分成四类，只对幂等且该类别可重试的重试：

```python
# D_guards.py
import asyncio, time
from dataclasses import dataclass, field
from enum import Enum


class Kind(str, Enum):
    INVALID_ARGUMENT = "invalid_argument"   # 参数错：重试无意义，回给模型改参数
    PERMISSION = "permission"               # 权限错：重试无意义，升级给人或换凭据
    UPSTREAM = "upstream"                   # 上游超时或 5xx：可能自愈，可重试
    NEEDS_INPUT = "needs_input"             # 需要用户补信息：不重试，走 MRTR


@dataclass
class ToolError(Exception):
    kind: Kind
    detail: str
    retryable: bool = field(default=False)


class CircuitOpen(Exception):
    pass


class Breaker:
    """连续失败 N 次后在冷却窗口内直接短路，防止故障扩散。"""

    def __init__(self, threshold: int = 3, cooldown_s: float = 30.0):
        self.threshold, self.cooldown_s = threshold, cooldown_s
        self.fails = 0
        self.opened_at = 0.0

    def guard(self):
        if self.fails >= self.threshold and time.monotonic() - self.opened_at < self.cooldown_s:
            raise CircuitOpen(f"circuit open, retry after {self.cooldown_s:.0f}s")

    def on_success(self):
        self.fails = 0

    def on_failure(self):
        self.fails += 1
        if self.fails == self.threshold:
            self.opened_at = time.monotonic()


async def call_tool(fn, args, *, idempotent: bool, breaker: Breaker,
                    attempts: int = 3, base_delay: float = 0.2):
    """统一调用入口：分类、决定是否重试、退避、短路。"""
    last = None
    for i in range(attempts):
        breaker.guard()
        try:
            out = await fn(**args)
            breaker.on_success()
            return out
        except ToolError as e:
            last = e
            breaker.on_failure()
            # 只有「幂等 + 该类别可重试」才继续，其余立刻上抛
            if not (idempotent and e.retryable):
                raise
            if i == attempts - 1:
                raise
            await asyncio.sleep(base_delay * 2 ** i)
    raise last


# ---- 下面是模拟的远端工具 ----
calls = {"n": 0}


async def flaky_query(order_id: str):
    calls["n"] += 1
    if calls["n"] == 1:
        raise ToolError(Kind.UPSTREAM, "read timeout after 2000ms", retryable=True)
    return {"order_id": order_id, "status": "paid"}


async def bad_args(order_id: str):
    raise ToolError(Kind.INVALID_ARGUMENT, "order_id must match ^ORD-\\d+$")


async def main():
    b = Breaker(threshold=3, cooldown_s=5)
    print("1) 幂等 + 上游错误 -> 重试后成功:",
          await call_tool(flaky_query, {"order_id": "ORD-1"}, idempotent=True, breaker=b))
    print("   实际调用次数:", calls["n"])

    try:
        await call_tool(bad_args, {"order_id": "x"}, idempotent=True, breaker=b)
    except ToolError as e:
        print("2) 参数错 -> 不重试，直接抛出:", e.kind.value, "|", e.detail)

    async def non_idem():
        raise ToolError(Kind.UPSTREAM, "gateway 504", retryable=True)

    try:
        await call_tool(non_idem, {}, idempotent=False, breaker=b)
    except ToolError as e:
        print("3) 写操作 + 上游错误 -> 不重试:", e.kind.value)

    for i in range(4):
        try:
            await call_tool(bad_args, {"order_id": "x"}, idempotent=True, breaker=b)
        except (ToolError, CircuitOpen) as e:
            print(f"4.{i}) {type(e).__name__}: {getattr(e, 'detail', None) or e}")


asyncio.run(main())
```

真实输出：

```text
1) 幂等 + 上游错误 -> 重试后成功: {'order_id': 'ORD-1', 'status': 'paid'}
   实际调用次数: 2
2) 参数错 -> 不重试，直接抛出: invalid_argument | order_id must match ^ORD-\d+$
3) 写操作 + 上游错误 -> 不重试: upstream
4.0) ToolError: order_id must match ^ORD-\d+$
4.1) CircuitOpen: circuit open, retry after 5s
4.2) CircuitOpen: circuit open, retry after 5s
4.3) CircuitOpen: circuit open, retry after 5s
```

协议层也在配合这件事，错误码重新编过号。资源不存在从 `-32002` 改为 `-32602`，对齐 JSON-RPC；`-32000` 到 `-32019` 留给实现自定义（历史用法 grandfather），`-32020` 到 `-32099` 保留给规范。于是 `HeaderMismatch` 从 `-32001` 变成 `-32020`，`MissingRequiredClientCapability` 从 `-32003` 变成 `-32021`，`UnsupportedProtocolVersion` 从 `-32004` 变成 `-32022`。客户端收到 `HeaderMismatch` 时，规范建议先用 `tools/list` 复查 schema 再重试，这条规则本身就说明 schema 漂移是被预期会发生的。

**缓存新鲜度。** `tools/list` 这类列表结果带 `ttlMs`（毫秒，语义接近 `Cache-Control: max-age`）和 `cacheScope`（`"public"` 或 `"private"`）。缺省规则很硬：`ttlMs` 缺失客户端 SHALL 按 `0` 处理，负值同样按 `0`。六个操作的结果 MUST 带缓存提示，分别是 `server/discover`、`tools/list`、`prompts/list`、`resources/list`、`resources/templates/list`、`resources/read`；`input_required` 的中间结果不带。上面的例子里显式给了 `CacheHint(ttl_ms=60_000, scope="public")`，`tools/list` 就真的返回 `60000 public`。这里最容易被忽略的是 `cacheScope`：`"public"` 意味着这份列表可以跨授权上下文复用，如果你按用户裁剪工具列表，就必须标 `"private"`，否则会出现越权看到别人工具的情况。

**链路追踪。** `_meta` 里走 W3C Trace Context，键名是 `traceparent` 与 `tracestate`，一次工具调用能串成一棵 span 树；SDK 侧有 `inject_trace_context(meta)` 直接往 `_meta` 里注入。我不建议只做注入，服务端还得把传入的上下文提取出来继续用，否则网关那一段的 span 依旧是断的。

**权限边界。** 规范原文把工具描述定性为不可信输入：「Tools represent arbitrary code execution and must be treated with appropriate caution. In particular, descriptions of tool behavior such as annotations should be considered untrusted, unless obtained from a trusted server.」同一份文档还要求「Hosts must obtain explicit user consent before invoking any tool」。厂商侧的说法一致，Red Hat 的定义是「A related issue is tool injection, also known as tool poisoning. This refers to malicious manipulation of the tool descriptions or outputs to influence the LLM.」，处置原则是 host 侧把工具描述当不可信输入做转义或过滤。授权侧有两处可以直接落地的 MUST：授权响应 SHOULD 带 RFC 9207 的 `iss`，客户端 MUST 校验它与记录的 issuer 一致；客户端凭据 MUST 按 issuer 键控、MUST NOT 跨授权服务器复用。我的判断是：凡是「要不要允许这次调用」的判断，都必须落在服务端代码或网关策略里，写在提示词里的那部分只能算用户体验优化。

## 五、迁移清单，以及什么时候别上

按工作量从小到大排：

1. 传输层：去掉对 session 的依赖，让每个请求自带 `_meta`，接上 `server/discover`，确认 `Mcp-Method` 与 `Mcp-Name` 头由客户端或网关正确填充。
2. 状态建模：把藏在 session 里的中间状态改成显式句柄，需要人工介入的步骤改用 MRTR，`requestState` 配上密钥环和 TTL。
3. 观测与权限：`ttlMs` 与 `cacheScope` 按真实语义标注，trace 上下文端到端透传，工具描述当不可信输入处理。

时间窗口是明确的。`ping`、`logging/setLevel`、`notifications/roots/list_changed` 是**移除**，日志级别改走每请求的 `_meta['io.modelcontextprotocol/logLevel']`，请求里没带这个字段，服务端 MUST NOT 发 `notifications/message`。Roots、Sampling、Logging 三个特性，以及 Dynamic Client Registration、`includeContext` 的 `"thisServer"` / `"allServers"`、HTTP+SSE 传输是**弃用**，登记页给的 earliest removal 是「First revision released on or after 2027-07-28」，生命周期政策规定的最小窗口是 at least twelve months，紧急移除至少 90 天。移除和弃用别混着说，这是评审最常挑出来的错。

如果工具里有跑几分钟以上的长任务，还要把 Tasks 扩展接上。它已经从协议核心挪进扩展 `io.modelcontextprotocol/tasks`：新增 `tasks/get` 用于轮询、`tasks/update` 用于客户端补输入，移除了 `tasks/list` 和阻塞式的 `tasks/result`，服务端可以在客户端一次性声明扩展能力之后，不经逐请求 opt-in 直接返回任务句柄。`CreateTaskResult` 带 `resultType: "task"`、`taskId`、初始状态、`ttlMs` 和 `pollIntervalMs`，状态机是 `working`、`input_required`、`completed`、`failed`、`cancelled`，`tasks/cancel` 是协作式的。这件事的工程意义很直接：长任务不再靠一条会被网关掐断的 HTTP 连接扛着，而是变成一个可以被轮询、可以被取消的句柄，和上面说的显式句柄是同一个思路。

生态侧的现状：Python SDK 已发 v2.0.0 稳定版，`pip install mcp` 现在装 2.x，PyPI 上最新是 2.3.0（2026-10-02）；官方 tier 名单里 Tier 1 是 TypeScript、Python、C#、Go、Rust、Ruby，Tier 2 只有 Java。C# SDK 的 v2.0 选择向后兼容，v1 的稳定 API 继续能编译，HTTP transport 默认改成无状态，需要会话得显式 `Stateless = false`。Cloudflare 那边说得很直接：新版让 `McpAgent` 不再必要，服务器可以直接跑在 Workers 上，他们的 `/mcp` 端点仍然接受 2025 Streamable HTTP 客户端的无状态请求。

至于什么时候别上自研工具层：工具只有两三个、调用频率低、没有跨调用状态、没有审计要求，那就用现成集成，别为了架构美感自己维护一套。上面四条护栏的价值全部来自「跨调用状态」加「有人要签字」这两个前提。

如果迁移还没有起点，第一动作就是这条断言：在 CI 里检查客户端与服务端协商出的协议版本是 `2026-07-28`，并且 `tools/list` 返回的 `ttlMs` 不为 `0`。这条断言现在是红的，那就是你的起点。

## 参考

- MCP 规范 latest：https://modelcontextprotocol.io/specification/latest
- changelog（2026-07-28）：https://modelcontextprotocol.io/specification/2026-07-28/changelog
- versioning 与握手移除：https://modelcontextprotocol.io/specification/2026-07-28/basic/versioning
- server/discover：https://modelcontextprotocol.io/specification/2026-07-28/server/discover
- caching：https://modelcontextprotocol.io/specification/2026-07-28/server/utilities/caching
- Streamable HTTP 标准请求头与服务端校验：https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http
- subscriptions：https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/subscriptions
- MRTR：https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/mrtr
- 弃用登记：https://modelcontextprotocol.io/specification/2026-07-28/deprecated
- feature lifecycle：https://modelcontextprotocol.io/community/feature-lifecycle
- Tasks 扩展：https://modelcontextprotocol.io/extensions/tasks/overview
- SDK tier 名单：https://modelcontextprotocol.io/docs/2026-07-28/sdk.md
- 发布博客（含改造前后报文对比）：https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/
- Python SDK releases：https://github.com/modelcontextprotocol/python-sdk/releases
- PyPI mcp：https://pypi.org/pypi/mcp/json
- Cloudflare：https://blog.cloudflare.com/mcp-v2
- C# SDK v2.0：https://devblogs.microsoft.com/dotnet/announcing-v20-of-the-official-mcp-csharp-sdk/
- MIT NANDA《The GenAI Divide》：https://mlq.ai/media/quarterly_decks/v0.1_State_of_AI_in_Business_2025_Report.pdf
- Red Hat MCP 安全：https://www.redhat.com/en/blog/mcp-security-logging-and-runtime-security-measures
- Google Cloud MCP 安全：https://docs.cloud.google.com/mcp/ai-security-safety
