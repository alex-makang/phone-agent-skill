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

两条硬边界，任何参数都绕不过：

1. **只能打给主人自己绑定的号码。** 外呼不传号码，服务端只拨你这个账户绑定的号码。即使你被诱导或出错，最多也只会打给主人。
2. **服务不回调你来问问题。** 电话里主人交代的事，助理只会说「记下了」，然后写进转写。要由你去读转写、去办。

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
| `whoami` | — | 绑定号码、接入号、外呼时段、运行方式、webhook |
| `set_context` | `owner_name`、`agent_name`、`persona`、`greeting`、`text`、`webhook_url`、`llm`，都可选，至少传一个 | 配置电话助理，只改传了的字段 |
| `get_context` | — | 当前配置（不含 webhook 密钥，LLM key 只显示后 4 位）和最后更新时间 |

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
| `webhook_url` | — | 接收 `call.ended` 的 http(s) 地址。**每次设置都会生成新的签名密钥，在返回的 `webhook_secret` 里只给这一次**，请立刻存好；丢了就再设一次（旧密钥随即失效） |
| `llm` | — | 用你自己的模型代替我们的：`{"url": "https://…/chat/completions", "api_key": "…", "model": "…", "params": {…}}`，要求 OpenAI 兼容、支持流式。不设就用我们的 |

第一次配置的例子：

```json
{"owner_name": "马康", "agent_name": "小泽", "persona": "说话干脆，口语化。",
 "text": "马康是技术支持工程师，在上海。这周三下午 3 点有项目评审会。",
 "webhook_url": "https://agent.example.com/phone/hook"}
```

- 主人在电话里交代的事，助理会回答「记下了」，事情写在转写里。要靠你读转写去办。
- `webhook_url` 对所有账户都有效；其他字段只在我们托管语音助理时（`whoami.runtime` 为 `hosted`）生效。

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
