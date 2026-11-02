---
name: phone-agent
description: 用电话联系你的主人：定时或马上给主人打电话（提醒、通知、要他确认的事），拿回通话结果和转写；读取主人打进来的电话（信箱）；配置电话助理（称呼、名字、人设、常驻背景、webhook、自带 LLM）。Use when the owner asks to be called or reminded by phone, when something must reach the owner right now, when you need what was said on a phone call, or to configure the phone assistant (set_context). Service: phone.alexmk.cn (MCP over HTTP).
---

# phone-agent：让你的 Agent 能给主人打电话

## 这是什么

phone-agent 是一层电话服务。它负责运营商线路、接打电话，以及电话里和人对话的语音助理（流式 ASR、预测式判停、TTS）。
你（主人的 Agent）通过一个 MCP 端点使用它：

- **外呼**：`place_call` 给主人打电话。语音助理问候之后，会先说你给的来意，再按你给的要点和主人对话。
  挂断后，结果和转写回到你这里。
- **呼入**：主人用绑定的手机拨**接入号 0591-62634634**，就能和电话助理通话。这通电话的转写会进你的信箱。
- **配置**：电话助理怎么称呼主人、叫什么、怎么说话、知道哪些背景、结果推到哪里，全部由你用 `set_context` 设置。
- **通话中参与（要有 webhook）**：主人在电话里交代要办的事，助理会实时转给你（`call.handoff`），你办完用 `answer_handoff` 回，
  助理当场告诉主人；每说完一轮也会推给你（`call.turn`），你可以随时补充或纠正助理（`update_call`）。
  没设 webhook 时，助理只会说「记下了」，事情在转写里，由你事后去办。

硬边界，任何参数都绕不过：**只能打给主人自己绑定的号码。** 外呼不传号码，服务端只拨你这个账户绑定的号码。
即使你被诱导或出错，最多也只会打给主人。

## 接入只有两步

**第一步：开通，只需要主人的手机号。**
现在由运营方手动开通（联系 Alex），以后在小程序里自助完成。提供一个或多个中国大陆手机号：外呼只打这些号，呼入只认这些号。
开通后拿到下面两样，**`mcp_key` 只显示一次**：

- `mcp_url`：`https://phone.alexmk.cn/mcp`
- `mcp_key`：`pk_…`，它就是主人的身份。

**第二步：接上 MCP，调 `set_context` 把电话助理配好。** 其他所有设置都在这一步完成，见下文「配置（set_context）」。
不配置也能打电话，只是助理叫「助理」，称呼主人为「机主」，什么背景都不知道。

### 鉴权

- 每个请求带 `Authorization: Bearer <mcp_key>`。key 错误时返回 HTTP 401。
- key 放进 secret store 或环境变量，不要写进代码、日志或聊天记录。
- 怀疑 key 泄露，就请运营方轮换（rotate-key）。轮换后旧 key 立即失效。
- 一把 key 只对应一个主人。你只能看到、操作这个主人的电话和配置。
- webhook 的签名密钥是另一样东西，由 `set_context` 设置 `webhook_url` 时生成，见下文。

## 接入

**Claude Code**

```bash
claude mcp add --transport http phone https://phone.alexmk.cn/mcp \
  --header "Authorization: Bearer $PHONE_MCP_KEY"
```

**AgentsGateway**：网关只在启动时加载 MCP，所以添加后要重启。

```bash
ag mcp add phone --type http --url https://phone.alexmk.cn/mcp --header "Authorization=Bearer <mcp_key>"
ag restart
```

**直接调（任何语言）**

服务端是无状态的 MCP Streamable HTTP：只接受 POST，每个请求一个 JSON-RPC 消息，不支持 SSE，也不支持批量。

```bash
curl -s https://phone.alexmk.cn/mcp -H "Authorization: Bearer $PHONE_MCP_KEY" -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"whoami","arguments":{}}}'
```

工具的结果在 `result.content[0].text` 里，是一段 JSON 字符串。出错时 `result.isError` 为 true，text 以 `error:` 开头。

接好后先调一次 `whoami`，确认绑定号码、接入号和外呼时段，然后调 `set_context` 完成配置。

## 工具

| 工具 | 参数 | 作用 |
|---|---|---|
| `place_call` | `purpose`（必填，≤200 字），`brief`（≤2000 字），`at` 或 `in_minutes`，`retries`（0–3，默认 1），`retry_interval_minutes`（1–60，默认 3） | 现在或定时给主人打电话，返回 `call_id` |
| `get_call` | `call_id` | 状态、每次拨打的结果、转写 |
| `list_calls` | `limit`（1–50，默认 10） | 最近的电话，含外呼（`direction: out`）和呼入（`direction: in`），不含转写 |
| `cancel_call` | `call_id` | 取消排期；正在响铃或通话中的会被挂断 |
| `whoami` | — | 绑定号码、接入号、外呼时段、webhook |
| `set_context` | `owner_name`、`agent_name`、`persona`、`greeting`、`text`、`webhook_url`、`llm`，都可选，至少传一个 | 配置电话助理，只改传了的字段 |
| `get_context` | — | 当前配置（不含 webhook 密钥，LLM key 只显示后 4 位）和最后更新时间 |
| `update_call` | `call_id`、`note` | 给**进行中**的通话补一条说明，助理从下一句开始照着做 |
| `answer_handoff` | `call_id`、`handoff_id`、`text` | 回复 `call.handoff`。通话还在时助理当场转告主人；已挂断时只记在通话记录里 |

### 什么时候该主动打电话

电话会打断主人手上的事，比消息贵得多。只在**消息不够用**的时候打：要他马上知道、要他现在拿主意，或者消息发了没人理。

**适合打的场景**

| 场景 | 例子（purpose） | 建议 |
|---|---|---|
| 准点提醒 | `提醒你 3 点的项目评审会`、`该吃药了`、`该出门去机场了` | 用 `at` 排在事前几分钟；要出门的，按路程多留时间 |
| 消息没人回，升级成电话 | `刚才发你的合同还没确认，今天下午 5 点截止` | 先发消息，过一阵没回再打；brief 里写清楚消息说的是什么 |
| 要他现在拿主意，事情才能往下走 | `订机票时价格涨了 300，问你还订不订` | brief 写清楚选项和各自的后果，「他选了其中一个」算完成；挂断后按转写去办 |
| 有时效的突发情况 | `你关注的航班延误了 2 小时`、`服务器告警，网站打不开了` | 只打你确认过、真要他处理的事；同一件事别反复打 |
| 交代的长任务办完了 | `你上午让我查的资料整理好了` | 电话里只说结论和要他做的事，细节发消息或文档 |
| 他主动要求的固定电话 | `早上的日程播报` | 只在主人明确要求时才排；内容放 brief，控制在一两分钟能说完 |

**不适合打的场景**

- 发条消息就能解决、晚点看也不耽误的事。
- 内容很长，或者需要他看着才能懂：表格、链接、代码。电话里说个要点就行，正文发消息。
- 敏感内容：电话可能外放，或者身边有人。brief 里只写「有件私事要你看消息」，正文走别的渠道。
- 外呼时段以外（见 `whoami.calling_hours`）。除非主人明确要求，不要排到深夜或清晨。
- 刚打过、他已经说「知道了」的同一件事。

**写 brief 时的几个要点**

- 写清楚**拿到什么算完成**：「他说知道了」「他选了 A 或 B」「他给了一个时间」。
- 要他做选择的，把选项写全，助理才能在电话里讲清楚，不会自己编。
- 挂断后用 `get_call`（或 webhook）读转写，把他在电话里说的事办掉，或者记下来。

### place_call 怎么写

- `purpose`：一句话说明为什么打。助理问候后会**最先**说它，例如 `提醒你 3 点的项目评审会`。
- `brief`：助理在整通电话里都知道的内容：要传达什么、要问什么、拿到什么回答算完成。
  例如：`会议线上开，链接已发到邮箱；问他要不要提前 10 分钟再提醒；他说知道了就算完成。`
- `at`：ISO 8601 时间。不带时区时按北京时间算，例如 `2026-11-03T15:00:00+08:00`。也可以改用 `in_minutes`。
  最多排到 7 天后。
- `retries`：只在**没接**时重拨。接了就算完成，不管谈得怎么样。
- brief 和转写会存在电话服务里。主人不想留在第三方的东西，不要写进 brief。

### 状态

- 外呼：`scheduled` → `dialing` → `completed`（接了）/ `unanswered`（重拨用完也没接）/ `failed` / `cancelled`
- 呼入：`completed`（接通了）/ `missed`（没接通）。`from` 是主人用哪个绑定号码打来的。
- 转写是 `[{who: "agent" | "caller", text}]`。外呼时 `caller` 也是主人本人。
  如果主人打断了助理，转写里只保留他实际听到的那部分。

## 拿结果：信箱或 webhook

**信箱（总能用）**：在需要时，或者在你自己的定时任务里，调 `list_calls` 看有没有新电话，再用 `get_call` 取转写。
呼入的电话只在信箱里，服务不会主动叫醒你。

**webhook（可选，用 `set_context` 设 `webhook_url`）**：外呼或呼入一结束就推送一次。

**先过测试才算设置成功。** 调 `set_context` 设 `webhook_url` 时，服务端会先往这个地址发一条测试事件：

```
POST <你的 URL>
X-Phone-Event: webhook.test
X-Phone-Timestamp: …
X-Phone-Signature: v1=…

{"type": "webhook.test", "event_id": "evt_…", "created_at": "…", "challenge": "Xk3…"}
```

你的接口要在 **5 秒内回 2xx**，并且**响应体里原样带回 `challenge`**，例如 `{"challenge": "Xk3…"}`。
这时你还没拿到签名密钥，所以测试事件不用验签。
测试不通过时，`set_context` 返回 `error: webhook test failed …; nothing was changed`，这次调用的**所有字段都不生效**。
测试通过后，返回里会带 `webhook_secret`，只给这一次。之后的 `call.ended` 都用它签名。

所以接入顺序是：先把接口部署好（能回 challenge），再调 `set_context`。拿到密钥后存好，之后的事件都要验签。

事件格式：

```
POST <你的 URL>
Content-Type: application/json
X-Phone-Event: call.ended
X-Phone-Timestamp: 1793606400
X-Phone-Signature: v1=<hex>

{"type": "call.ended", "event_id": "evt_…", "created_at": "2026-11-03T15:01:12+08:00",
 "call": { …和 get_call 的返回一样，含 transcript… }}
```

- 下文「通话中参与」的 `call.turn`、`call.handoff` 用同一套请求头和签名。
- 签名：`hex(HMAC-SHA256(webhook_secret, timestamp + "." + 原始请求体))`。要用原始字节验，不能用重新序列化的 JSON。
- 5 秒内回 2xx 算送达。失败会在 2 秒、10 秒后各重试一次，之后就只留在信箱。
- 用 `event_id` 去重，因为重试可能让同一事件到达两次。时间戳和当前时间差超过 5 分钟的，建议拒收。

Python（含测试事件的处理）：

```python
import hashlib, hmac, json, time

def handle(secret: str | None, headers: dict, raw_body: bytes) -> tuple[int, bytes]:
    event = json.loads(raw_body)
    if event.get("type") == "webhook.test":
        return 200, json.dumps({"challenge": event["challenge"]}).encode()
    if not secret or not verify(secret, headers, raw_body):
        return 401, b""
    # 用 event_id 去重。按 event["type"] 分：
    #   call.ended   -> 读 event["call"]["transcript"] 去办事
    #   call.handoff -> 记下 call_id / handoff_id / question，异步去办，办完调 MCP answer_handoff
    #   call.turn    -> 要纠正助理就回 {"note": "…"}，否则回空
    return 200, b""

def verify(secret: str, headers: dict, raw_body: bytes) -> bool:
    ts = headers["X-Phone-Timestamp"]
    if abs(time.time() - int(ts)) > 300:
        return False
    expected = "v1=" + hmac.new(secret.encode(), ts.encode() + b"." + raw_body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, headers["X-Phone-Signature"])
```

Node（测试事件同样处理：`type === 'webhook.test'` 时回 `{"challenge": event.challenge}`）：

```js
import crypto from 'node:crypto';

export function verify(secret, headers, rawBody /* Buffer */) {
  const ts = headers['x-phone-timestamp'];
  if (Math.abs(Date.now() / 1000 - Number(ts)) > 300) return false;
  const expected = 'v1=' + crypto.createHmac('sha256', secret).update(`${ts}.`).update(rawBody).digest('hex');
  const got = String(headers['x-phone-signature'] ?? '');
  return got.length === expected.length && crypto.timingSafeEqual(Buffer.from(got), Buffer.from(expected));
}
```

## 配置（set_context）

电话随时可能打进来，响铃那一两秒里来不及问你。所以助理每通电话只能用你**事先**配好的东西。
每个字段都可选，只改你传了的字段，其他保持不变。字符串传空串表示清空，`llm` 传 `null` 表示清空。改完下一通电话生效。

| 字段 | 上限 | 作用 |
|---|---|---|
| `owner_name` | 20 字 | 助理怎么称呼主人，例如 `马康`、`王总`。不设就称「机主」 |
| `agent_name` | 20 字 | 助理的名字，默认「助理」 |
| `persona` | 2000 字 | 说话风格和行为，例如「说话干脆，不客套」 |
| `greeting` | 100 字 | 主人打进来时的第一句，默认「你好，我是<agent_name>。」。外呼的开场固定是「喂，<owner_name>，我是<agent_name>。」 |
| `text` | 4000 字 | 常驻背景：主人是谁、偏好、这几天的安排和在忙的事、电话里遇到某类事怎么应对。**每次传都是整段替换**，有变化（比如每天早上）就重写 |
| `webhook_url` | — | 接收 `call.ended` 的 http(s) 地址。**先过测试（回带 challenge）才生效**，见上文。通过后生成新的签名密钥，在返回的 `webhook_secret` 里只给这一次，请立刻存好；丢了就再设一次（会重新测试，旧密钥随即失效） |
| `llm` | — | 用你自己的模型代替我们的：`{"url": "https://…/chat/completions", "api_key": "…", "model": "…", "params": {…}}`，要求 OpenAI 兼容、支持流式。不设就用我们的 |

第一次配置的例子：

```json
{"owner_name": "马康", "agent_name": "小泽", "persona": "说话干脆，口语化。",
 "text": "马康是技术支持工程师，在上海。这周三下午 3 点有项目评审会。",
 "webhook_url": "https://agent.example.com/phone/hook"}
```

- 主人在电话里交代的事：有 webhook 时实时转给你（见下一节）；没有时助理回答「记下了」，事情写在转写里，靠你事后去办。

## 通话中参与（webhook）

设了 `webhook_url` 以后，除了挂断时的 `call.ended`，通话进行中还会收到两种事件。签名、请求头和 `call.ended` 一样，`X-Phone-Event` 写的是事件类型。
事件里的 `call_id` 就是这通电话的记录 id，可以直接用来调 `get_call`、`update_call`、`answer_handoff`。
外呼和呼入都一样：呼入从接通那一刻就有 `call_id`。

### call.handoff：助理把事情交给你办

主人在电话里说了要查、要办的事，或者只有你才知道答案的事，助理会说一句「我去问一下」，同时推送：

```json
{"type": "call.handoff", "event_id": "evt_…", "created_at": "…", "call_id": "c_…", "handoff_id": "h…",
 "question": "查一下明天下午去北京的航班",
 "recent_turns": [{"who": "caller", "text": "…"}, {"who": "agent", "text": "…"}]}
```

- 尽快回 2xx 表示收到，**不要在这个请求里办事**（5 秒超时）。失败会在 2 秒、10 秒后各重试一次；
  都失败时，助理会告诉主人「没联系上助理，记下了，挂断后处理」。
- 办完调 `answer_handoff {call_id, handoff_id, text}`。`text` 写结论，助理会用自己的话说给主人听，所以写清楚、别太长。
- 助理在等结果时可以继续聊别的；结果到了，它会在当前这句说完后插进来转告。
- 如果你办完时通话已经挂断，`answer_handoff` 返回 `delivered: false`，答案只记在通话记录里（`get_call` 的 `handoffs`）。
  这时主人还没听到，你要换个方式告诉他：发消息，或者 `place_call`。

### call.turn：每一轮的对话（copilot）

主人说完、助理回答完，推一条：

```json
{"type": "call.turn", "event_id": "evt_…", "created_at": "…", "call_id": "c_…",
 "caller": "主人这一轮说的话", "agent": "助理这一轮的回答"}
```

- 不需要介入时，直接回 2xx、空响应体。
- 发现助理说错了、漏了，或者有它该知道的事，就在响应体里回 `{"note": "一句话说明"}`，助理从下一句开始照着做。
  也可以稍后随时调 `update_call {call_id, note}`。note 每条最多 500 字，以最新一条为准。
- 只等 3 秒、不重试，**超时就丢弃**，不会拖慢通话。所以要在 3 秒内回；来不及判断的，先回空，想好了再用 `update_call`。
- `agent` 是助理生成的完整回答；主人如果中途打断，他实际听到的可能更短。
- 每一轮都会推，量比较大。不想当 copilot 的，收到后直接回 2xx 就行。

## 限制与常见错误

| 情况 | 返回 / 做法 |
|---|---|
| 不在外呼时段 | `error: outside_hours …`。时段以 `whoami.calling_hours` 为准，目前是北京时间 8–20 点。这时由你决定改走别的渠道，不要默默推迟到第二天 |
| `at` 已经过去，或超过 7 天 | `error: …`，请修正时间 |
| `call_id` 不存在或不属于你 | `error: no such call` |
| key 错误或缺失 | HTTP 401 |
| 主人没接 | 状态为 `unanswered`，按 `retries` 重拨；用完以后要不要换渠道，由你决定 |
| `update_call` 时通话已经结束 | `error: the call is not in progress` |
| `handoff_id` 不对 | `error: no such handoff on this call` |

## 隐私

电话服务只保存通话需要的东西：brief、常驻背景、转写。主人的记忆和其他隐私都留在你这边。
不要把密码、证件号这类内容写进 brief 或常驻背景。
