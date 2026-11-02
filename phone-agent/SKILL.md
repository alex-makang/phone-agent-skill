---
name: phone-agent
description: 用电话联系你的主人：定时或马上给主人打电话（提醒、通知、要他确认的事），拿回通话结果和转写；读取主人打进来的电话（信箱）；维护电话助理每通电话都会用的常驻背景。Use when the owner asks to be called or reminded by phone, when something must reach the owner right now, when you need what was said on a phone call, or to keep the phone assistant's background current. Service: phone.alexmk.cn (MCP over HTTP).
---

# phone-agent：让你的 Agent 能给主人打电话

## 这是什么

phone-agent 是一层电话服务。它负责运营商线路、接打电话，以及电话里和人对话的语音助理（流式 ASR、预测式判停、TTS）。
你（主人的 Agent）通过一个 MCP 端点使用它：

- **外呼**：`place_call` 给主人打电话。语音助理问候之后，会先说你给的来意，再按你给的要点和主人对话。
  挂断后，结果和转写回到你这里。
- **呼入**：主人用绑定的手机拨**接入号 0591-62634634**，就能和电话助理通话。这通电话的转写会进你的信箱。
- **常驻背景**：你用 `set_context` 写一段背景，比如主人是谁、这几天在忙什么。电话助理每通电话都会带着它。

两条硬边界，任何参数都绕不过：

1. **只能打给主人自己绑定的号码。** 外呼不传号码，服务端只拨你这个账户绑定的号码。即使你被诱导或出错，最多也只会打给主人。
2. **服务不回调你来问问题。** 电话里主人交代的事，助理只会说「记下了」，然后写进转写。要由你去读转写、去办。

## 开通与鉴权

现在由运营方手动开通（联系 Alex）。以后会在小程序里自助生成。开通时需要提供：

| 项 | 说明 |
|---|---|
| 主人称呼 | 电话助理怎么称呼他 |
| 绑定手机号 | 一个或多个，中国大陆号码；外呼只打这些号，呼入只认这些号 |
| 电话助理的名字、人设 | 可选。名字默认「助理」，人设写说话风格等，最多 2000 字 |
| webhook URL | 可选。挂断后会推送结果，见下文 |
| 自己的 LLM 端点 | 可选。OpenAI 兼容的 `/chat/completions`（流式）及 key。不填就用我们的模型 |

开通后会拿到下面这些，**只显示一次**：

- `mcp_url`：`https://phone.alexmk.cn/mcp`
- `mcp_key`：`pk_…`。它就是主人的身份。
- `webhook_secret`：填了 webhook 才有，用来验签。

鉴权规则：

- 每个请求带 `Authorization: Bearer <mcp_key>`。key 错误时返回 HTTP 401。
- key 放进 secret store 或环境变量，不要写进代码、日志或聊天记录。
- 怀疑 key 泄露，就请运营方轮换（rotate-key）。轮换后旧 key 立即失效。
- 一把 key 只对应一个主人。你只能看到、操作这个主人的电话。

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

接好后先调一次 `whoami`，确认绑定号码、接入号和外呼时段。

## 工具

| 工具 | 参数 | 作用 |
|---|---|---|
| `place_call` | `purpose`（必填，≤200 字），`brief`（≤2000 字），`at` 或 `in_minutes`，`retries`（0–3，默认 1），`retry_interval_minutes`（1–60，默认 3） | 现在或定时给主人打电话，返回 `call_id` |
| `get_call` | `call_id` | 状态、每次拨打的结果、转写 |
| `list_calls` | `limit`（1–50，默认 10） | 最近的电话，含外呼（`direction: out`）和呼入（`direction: in`），不含转写 |
| `cancel_call` | `call_id` | 取消排期；正在响铃或通话中的会被挂断 |
| `whoami` | — | 绑定号码、接入号、外呼时段、运行方式、webhook |
| `set_context` | `text`（≤4000 字，传空串表示清空） | 整段替换常驻背景 |
| `get_context` | — | 当前常驻背景和最后更新时间 |

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

**webhook（可选，开通时填 URL）**：外呼或呼入一结束就推送一次。

```
POST <你的 URL>
Content-Type: application/json
X-Phone-Event: call.ended
X-Phone-Timestamp: 1793606400
X-Phone-Signature: v1=<hex>

{"type": "call.ended", "event_id": "evt_…", "created_at": "2026-11-03T15:01:12+08:00",
 "call": { …和 get_call 的返回一样，含 transcript… }}
```

- 签名：`hex(HMAC-SHA256(webhook_secret, timestamp + "." + 原始请求体))`。要用原始字节验，不能用重新序列化的 JSON。
- 5 秒内回 2xx 算送达。失败会在 2 秒、10 秒后各重试一次，之后就只留在信箱。
- 用 `event_id` 去重，因为重试可能让同一事件到达两次。时间戳和当前时间差超过 5 分钟的，建议拒收。

Python：

```python
import hashlib, hmac, time

def verify(secret: str, headers: dict, raw_body: bytes) -> bool:
    ts = headers["X-Phone-Timestamp"]
    if abs(time.time() - int(ts)) > 300:
        return False
    expected = "v1=" + hmac.new(secret.encode(), ts.encode() + b"." + raw_body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, headers["X-Phone-Signature"])
```

Node：

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

## 常驻背景（set_context）

电话随时可能打进来，响铃那一两秒里来不及问你。所以助理每通电话只能用你**事先**写好的背景。

- 写什么：主人的称呼和偏好；这几天的安排和在忙的事；电话里该怎么应对某些事。
- 每次调用都是**整段替换**。内容有变化（比如每天早上）就重写一次。
- 主人在电话里交代的事，助理会回答「记下了」，事情写在转写里。要靠你读转写去办。
- 只有我们托管语音助理时（`whoami.runtime` 为 `hosted`）才会用到常驻背景。

## 限制与常见错误

| 情况 | 返回 / 做法 |
|---|---|
| 不在外呼时段 | `error: outside_hours …`。时段以 `whoami.calling_hours` 为准，目前是北京时间 8–20 点。这时由你决定改走别的渠道，不要默默推迟到第二天 |
| `at` 已经过去，或超过 7 天 | `error: …`，请修正时间 |
| `call_id` 不存在或不属于你 | `error: no such call` |
| key 错误或缺失 | HTTP 401 |
| 主人没接 | 状态为 `unanswered`，按 `retries` 重拨；用完以后要不要换渠道，由你决定 |

## 隐私

电话服务只保存通话需要的东西：brief、常驻背景、转写。主人的记忆和其他隐私都留在你这边。
不要把密码、证件号这类内容写进 brief 或常驻背景。
