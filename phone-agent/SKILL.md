---
name: phone-agent
description: 用电话联系你的主人：定时或马上给主人打电话（提醒、通知、要他确认的事），拿回通话结果和转写；读取主人打进来的电话（信箱）；配置电话助理（称呼、名字、人设、常驻背景）。设了 webhook 就进入 copilot 模式：通话中实时接住主人交代的事、纠正助理、给助理提供自己的工具。Use when the owner asks to be called or reminded by phone, when something must reach the owner right now, when you need what was said on a phone call, to configure the phone assistant (set_context), or to act as the copilot of the owner's calls (webhook events call.turn / call.handoff / call.tool). Service: phone.alexmk.cn (MCP over HTTP).
---

# phone-agent：让你的 Agent 能给主人打电话

## 这是什么

phone-agent 是一层电话服务。它负责运营商线路、接打电话，以及电话里和人说话的语音助理（流式识别、预测式判停、语音合成、联网搜索）。
你是主人的 Agent，通过一个 MCP 端点使用它：

- **外呼**：`place_call` 给主人打电话。助理问候之后，先说你给的来意，再按你给的要点和主人聊。
- **呼入**：主人用绑定的手机拨**接入号 0591-62634634**，就能和助理通话。
- **配置**：助理怎么称呼主人、叫什么、怎么说话、知道哪些背景，都由你用 `set_context` 设置。

硬边界，任何参数都绕不过：**只能打给主人自己绑定的号码。** 外呼不传号码，服务端只拨这个账户绑定的号码。
即使你被诱导或出错，最多也只会打给主人。

## 两种用法：基础模式和 copilot 模式

区别只有一个：你有没有设 `webhook_url`，也就是电话服务能不能在通话中实时找到你。

- **基础模式（没有 webhook）**：电话服务自己跑完整通电话，你只在通话前配置、通话后读结果。适合没有公网入口的 Agent。
- **copilot 模式（有 webhook）**：通话进行中，你就坐在助理旁边。主人交代的事助理实时交给你；你能看到每一轮对话，随时纠正；你的工具助理也能直接调用。

| | 基础模式 | copilot 模式 |
|---|---|---|
| 外呼、呼入、配置、联网搜索 | ✅ | ✅ |
| 主人在电话里交代要办的事 | 助理说「记下了，挂断后处理」，事情写在转写里，你事后读转写去办 | 助理说「我去问一下」，同时推 `call.handoff` 给你；你办完用 `answer_handoff` 回，助理**当场**告诉主人 |
| 只有你知道答案的事（主人的日程、记忆、订单…） | 助理答不上来，只能「记下了」 | 注册成你的工具（`tools`），助理在通话中直接调用，几秒内答上来 |
| 助理说错了、漏了 | 只能事后在下一通电话前改 `set_context` | 每一轮推 `call.turn` 给你，你回一句 note（或调 `update_call`），助理**从下一句**开始照着改 |
| 通话结束 | 你自己定时 `list_calls` / `get_call` 去拉 | 另外推一条 `call.ended`，带完整转写 |
| 挂断后才办完的事 | 不适用 | `answer_handoff` 照样回，助理在主人**下一次**接通时先说 |
| 你需要提供 | 什么都不用 | 一个能从公网访问、5 秒内响应的 HTTPS 接口 |

两种模式可以随时切换：设 `webhook_url` 就进入 copilot 模式，把它设成空串就回到基础模式，下一通电话生效。

## 接入

### 第一步：开通，只需要主人的手机号

现在由运营方手动开通（联系 Alex），以后在小程序里自助完成。提供一个或多个中国大陆手机号：外呼只打这些号，呼入只认这些号。
开通后拿到两样东西，**`mcp_key` 只显示一次**：

- `mcp_url`：`https://phone.alexmk.cn/mcp`
- `mcp_key`：`pk_…`，它就是主人的身份。

### 第二步：接上 MCP，调 `set_context` 配好助理

**Claude Code**

```bash
claude mcp add --transport http phone https://phone.alexmk.cn/mcp \
  --header "Authorization: Bearer $PHONE_MCP_KEY"
```

**AgentsGateway**：网关只在启动时加载 MCP，添加后要重启。

```bash
ag mcp add phone --type http --url https://phone.alexmk.cn/mcp --header "Authorization=Bearer <mcp_key>"
ag restart
```

**直接调（任何语言）**：服务端是无状态的 MCP Streamable HTTP，只接受 POST，每个请求一个 JSON-RPC 消息，不支持 SSE，也不支持批量。

```bash
curl -s https://phone.alexmk.cn/mcp -H "Authorization: Bearer $PHONE_MCP_KEY" -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"whoami","arguments":{}}}'
```

工具结果在 `result.content[0].text` 里，是一段 JSON 字符串。出错时 `result.isError` 为 true，text 以 `error:` 开头。

接好以后，按下面「首次设置」一步步配好。
不配置也能打电话，只是助理叫「助理」，称呼主人为「机主」，什么背景都不知道。

### 鉴权

- 每个请求带 `Authorization: Bearer <mcp_key>`。key 错误时返回 HTTP 401。
- key 放进 secret store 或环境变量，不要写进代码、日志或聊天记录。
- 怀疑 key 泄露，就请运营方轮换（rotate-key）。轮换后旧 key 立即失效。
- 一把 key 只对应一个主人。你只能看到、操作这个主人的电话和配置。
- webhook 的签名密钥是另一样东西，设 `webhook_url` 时生成，见「copilot 模式」。

## 首次设置：带主人一步步配好

**什么时候走这个流程**：第一次用这个 skill；`get_context` 显示还没配置（`owner_name`、`text` 都是空的）；或者主人说「帮我设置一下电话助理」。
流程可以重复走：每一步都先看现状，已经配好的不重配，只补缺的。改动已有配置之前先问主人。

**原则**
- 先用你已经知道的，再问主人。称呼、偏好、近况，你的记忆里多半有，不要让主人重复回答。
- 要问的集中成一条消息问完，不要一问一答拖很多轮。
- 写进电话服务的内容（背景、人设）会存在第三方，写之前让主人过目。敏感的（密码、证件号、具体金额）不要写。

**第 1 步：检查连接**
调 `whoami`。
- 返回 HTTP 401：key 不对或没配。停下来，告诉主人需要开通或重新配 key（见「接入」）。
- 正常：记下 `numbers`（绑定号码）、`access_number`（接入号）、`calling_hours`（外呼时段），后面要告诉主人。

**第 2 步：看现在配了什么**
调 `get_context`。已经有值的字段保留；空的字段进入下一步。

**第 3 步：准备一份配置草稿，一次问主人**
先用你的记忆把能填的填上，把不知道的列成问题。一条消息里给主人看草稿和问题，例如：

> 我来把电话助理配一下。打算这样设，你看行不行，空着的帮我补一下：
> - 助理怎么称呼你：马康（从我的记忆里来）
> - 电话里的助理叫什么：？（默认叫「助理」，也可以和我同名）
> - 说话风格：简短直接（你平时的偏好）
> - 电话里它知道你的这些背景：……（草稿，3–5 行）
> - 有没有它必须知道、或者绝对不能在电话里说的事？
> - 什么情况下我可以主动给你打电话？比如会议前提醒、消息半小时没回的急事、你交代的事办完了。有没有不想被打扰的时间？
>   现在想不全也没关系，先说几条，以后随时可以补。
> - 外呼时段是北京时间 8–20 点，接入号是 0591-62634634。

对应的字段：称呼 → `owner_name`；助理名字 → `agent_name`；助理怎么称呼你 → `owner_agent_name`（不填就叫「<owner_name>的助理」）；
风格和处理方式 → `persona`；背景 → `text`；呼入开场白 → `greeting`（可不设）；什么时候可以主动打电话 → `call_policy`。
`call_policy` 是给你自己看的，不会进电话助理的 prompt。存在电话服务这里，是因为不是每个 Agent 都有地方长期记东西。
写 `persona` 前先看「这些我们已经内置，你不用写」，只写只有你们知道的东西。

**第 4 步：定模式**
- 你有能从公网访问、5 秒内响应的 HTTPS 接口 → **copilot 模式**：
  1. 部署接收端（见「copilot 模式」的示例），确认它能回 `webhook.test` 的 challenge。
  2. 有能马上查到结果的只读能力（主人的记忆、日程），整理成 `tools`。
  3. 和第 5 步的配置一起，用一次 `set_context` 设 `webhook_url` 和 `tools`。返回的 `webhook_secret` 立刻存进 secret store，只给这一次。
- 没有 → **基础模式**，跳过这一步。以后有了接收端再加，随时可以切换。

**第 5 步：写入并核对**
主人确认后，调一次 `set_context` 写入所有字段，再调 `get_context`，确认写进去的就是主人看过的那份。

**第 6 步：试打一通（先问主人）**
在外呼时段内，问主人现在方不方便接个测试电话。同意后：
`place_call {"purpose": "测试一下电话助理", "brief": "这是设置完成后的测试电话。说明来意后问他听得清不清楚、称呼和说话风格合不合适；他说没问题就道别。"}`
挂断后（webhook 会推 `call.ended`，或者用 `get_call` 去拉），读转写，看主人有没有提意见，有就改配置。
也可以请主人用绑定的手机拨接入号试一次呼入。

**第 7 步：设好日常维护**
- 背景要跟着变：主人的安排有变化时，或者每天早上一次，重写 `set_context` 的 `text`。
- 基础模式：建一个定时任务，定期（比如每 15–30 分钟，或者每天几次）调 `list_calls`，读新电话的转写，把主人交代的事办掉。
- copilot 模式：确认接收端常驻运行；接收端挂了，助理会退回「记下了」，通话不受影响。
- 主人以后说「这种事以后打电话告诉我」或「晚上别打给我」，就更新 `call_policy`：先 `get_context` 读出来，合并这一条，再整段写回。
- 你有自己的记忆的，记下已经设置过、用的哪种模式、webhook 密钥存在哪里、维护任务是哪个。没有记忆的，这些都能从 `get_context`、`whoami` 重新查到，不必重来。

**第 8 步：告诉主人怎么用**
用一条消息说清楚：
- 接入号 0591-62634634，用绑定的手机拨打就能找助理。
- 你以后会在什么情况下给他打电话（见「什么时候该主动打电话」），外呼时段是几点到几点。
- copilot 模式下，电话里交代的事助理会当场转给你办；基础模式下挂断后再办。

## 工具

| 工具 | 参数 | 作用 | 模式 |
|---|---|---|---|
| `place_call` | `purpose`（必填，≤200 字），`brief`（≤2000 字），`at` 或 `in_minutes`，`retries`（0–3，默认 1），`retry_interval_minutes`（1–60，默认 3） | 现在或定时给主人打电话，返回 `call_id` | 都可用 |
| `get_call` | `call_id` | 状态、每次拨打的结果、转写、转交记录 | 都可用 |
| `list_calls` | `limit`（1–50，默认 10） | 最近的电话，含外呼（`direction: out`）和呼入（`direction: in`），不含转写 | 都可用 |
| `cancel_call` | `call_id` | 取消排期；正在响铃或通话中的会被挂断 | 都可用 |
| `whoami` | — | 绑定号码、接入号、外呼时段、当前 webhook | 都可用 |
| `set_context` | 见「配置」，都可选，至少传一个 | 配置助理，只改传了的字段 | 都可用 |
| `get_context` | — | 当前配置（不含 webhook 密钥，LLM key 只显示后 4 位）和最后更新时间 | 都可用 |
| `update_call` | `call_id`、`note` | 给**进行中**的通话补一条说明，助理从下一句开始照着做 | copilot |
| `answer_handoff` | `call_id`、`handoff_id`、`text` | 回复 `call.handoff`。通话还在时助理当场转告主人；已挂断时下次接通先说 | copilot |

## 外呼

### 什么时候该主动打电话

电话会打断主人手上的事，比消息贵得多。只在**消息不够用**的时候打：要他马上知道、要他现在拿主意，或者消息发了没人理。

**先看主人自己定的规矩。** 每次主动打电话之前（主人当场让你打的除外），先 `get_context` 读 `call_policy`：
主人说过要打的场景就打，说过不要打的就不打。`call_policy` 里没写到的情况，再按下面的默认建议判断。
主人还没设 `call_policy` 的，按默认建议来，并在合适的时候问他一次（见「首次设置」第 3 步）。

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

### place_call 怎么写

- `purpose`：一句话说明为什么打。助理问候后会**最先**说它，例如 `提醒你 3 点的项目评审会`。
- `brief`：助理在整通电话里都知道的内容：要传达什么、要问什么、拿到什么回答算完成。
  例如：`会议线上开，链接已发到邮箱；问他要不要提前 10 分钟再提醒；他说知道了就算完成。`
  - 写清楚**拿到什么算完成**：「他说知道了」「他选了 A 或 B」「他给了一个时间」。
  - 要他做选择的，把选项写全，助理才能讲清楚，不会自己编。
  - copilot 模式下，brief 里没写到、主人又追问的事，助理会转交给你；基础模式下只能「记下了」。所以基础模式的 brief 要写得更全。
- `at`：ISO 8601 时间。不带时区时按北京时间算，例如 `2026-11-03T15:00:00+08:00`。也可以改用 `in_minutes`。最多排到 7 天后。
- `retries`：只在**没接**时重拨。接了就算完成，不管谈得怎么样。
- brief 和转写会存在电话服务里。主人不想留在第三方的东西，不要写进 brief。

### 状态

- 外呼：`scheduled` → `dialing` → `completed`（接了）/ `unanswered`（重拨用完也没接）/ `failed` / `cancelled`
- 呼入：`in_progress`（通话中）→ `completed`（接通了）/ `missed`（没接通）。`from` 是主人用哪个绑定号码打来的。
- 转写是 `[{who: "agent" | "caller", text}]`。外呼时 `caller` 也是主人本人。
  主人打断了助理时，转写里只保留他实际听到的那部分。
- 有转交的通话，记录里还有 `handoffs`：每条的问题、状态和答案。

## 配置（set_context）

电话随时可能打进来，响铃那一两秒里来不及问你。所以助理每通电话用的都是你**事先**配好的东西。
每个字段都可选，只改你传了的字段，其他保持不变。字符串传空串表示清空，`llm`、`tools` 传 `null` 表示清空。改完下一通电话生效。

| 字段 | 上限 | 作用 | 模式 |
|---|---|---|---|
| `owner_name` | 20 字 | 助理怎么称呼主人，例如 `马康`、`王总`。不设就称「机主」 | 都可用 |
| `agent_name` | 20 字 | 助理的名字，默认「助理」 | 都可用 |
| `owner_agent_name` | 20 字 | 助理怎么称呼你（主人的 Agent），例如 `小管家`。不设就叫「<owner_name>的助理」。助理转交、说「记下了」时都用这个名字 | 都可用 |
| `persona` | 4000 字 | 助理是谁、说话风格和行为，例如「说话干脆，不客套」。**不要写任何工具名**：工具由各自的说明告诉助理什么时候用（见「分层」） | 都可用 |
| `greeting` | 100 字 | 主人打进来时的第一句，默认「你好，我是<agent_name>。」。外呼的开场固定是「喂，<owner_name>，我是<agent_name>。」 | 都可用 |
| `text` | 4000 字 | 常驻背景：主人是谁、偏好、这几天的安排和在忙的事、遇到某类事怎么应对。**每次传都是整段替换**，有变化（比如每天早上）就重写 | 都可用 |
| `call_policy` | 2000 字 | 主人希望你在什么情况下主动给他打电话，什么时候别打。用主人的原话写，例如「会议前 10 分钟提醒；消息 30 分钟没回的急事升级成电话；22 点后除非紧急别打」。**只给你自己看**，不进电话助理的 prompt。整段替换：先读、合并、再写回 | 都可用 |
| `llm` | — | 用你自己的模型代替我们的：`{"url": "https://…/chat/completions", "api_key": "…", "model": "…", "params": {…}}`，要求 OpenAI 兼容、支持流式和 tool calling。不设就用我们的 | 都可用 |
| `webhook_url` | — | 你的接收端地址。设了就进入 copilot 模式；**先过测试才生效**，见「copilot 模式」 | 开启 copilot |
| `tools` | 8 个 | 你自己的工具，通话中助理直接调用，见「call.tool」。必须先有 `webhook_url` | copilot |

基础模式的第一次配置：

```json
{"owner_name": "马康", "agent_name": "小泽", "owner_agent_name": "小管家", "persona": "说话干脆，口语化。",
 "text": "马康是技术支持工程师，在上海。这周三下午 3 点有项目评审会。"}
```

基础模式下，`text` 就是助理能知道的全部，要写得更全一些，并且每天更新。

## 基础模式：通话后拿结果

在需要时，或者在你自己的定时任务里，调 `list_calls` 看有没有新电话，再用 `get_call` 取转写。
呼入的电话也在这里，服务不会主动叫醒你，所以建议定时拉，比如每 10–30 分钟一次，或者每天早上。

读转写时重点找两类内容：
1. 主人在电话里交代要办的事。助理答应过「挂断后会交给你处理」，主人会默认你在办。
2. 主人更正过的信息（比如「会议改到四点了」）。更新到你的记忆里，必要时改 `set_context` 的 `text`。

## copilot 模式

### 开启：先部署接收端，再设 webhook_url

1. 部署一个 HTTPS 接口，能处理下面的事件。最低要求是能回 `webhook.test` 的 challenge。
2. 调 `set_context {"webhook_url": "https://…"}`。服务端会立刻往这个地址发一条测试事件：

   ```json
   {"type": "webhook.test", "event_id": "evt_…", "created_at": "…", "challenge": "Xk3…"}
   ```

   你的接口要在 **5 秒内**回 2xx，并且**响应体里原样带回 `challenge`**，例如 `{"challenge": "Xk3…"}`。
   这时你还没拿到签名密钥，所以测试事件不用验签。
3. 测试不通过时返回 `error: webhook test failed …; nothing was changed`，这次调用的**所有字段都不生效**。
   测试通过后，返回里带 `webhook_secret`，**只给这一次**，立刻存好。之后所有事件都用它签名。
   丢了就再设一次 `webhook_url`：会重新测试，旧密钥随即失效。
4. 要用自定义工具的，再设 `tools`（可以和 `webhook_url` 放在同一次调用里）。

### 事件一览

所有事件都是 `POST <webhook_url>`，`Content-Type: application/json`，请求头带签名，`X-Phone-Event` 是事件类型。
所有通话中事件里的 `call_id` 就是这通电话的记录 id，可以直接用来调 `get_call`、`update_call`、`answer_handoff`。
外呼和呼入都一样：呼入从接通那一刻就有 `call_id`。

| 事件 | 什么时候 | 等你多久 | 失败重试 | 你要回什么 |
|---|---|---|---|---|
| `webhook.test` | 设 `webhook_url` 时 | 5 秒 | 不重试 | `{"challenge": "…"}` |
| `call.turn` | 每一轮：主人说完、助理答完 | 3 秒 | 不重试，超时丢弃 | 空，或 `{"note": "…"}` |
| `call.handoff` | 助理把一件事交给你 | 5 秒（只要确认收到） | 2 秒、10 秒后各一次 | 2xx 即可，办完另调 `answer_handoff` |
| `call.tool` | 助理调用你注册的工具 | 5 秒 | 不重试 | `{"result": "…"}` |
| `call.ended` | 通话结束（含没接通、取消） | 5 秒 | 2 秒、10 秒后各一次 | 2xx 即可 |

**签名**：`X-Phone-Signature: v1=<hex>`，其中 hex 是 `HMAC-SHA256(webhook_secret, X-Phone-Timestamp + "." + 原始请求体)`。
- 要用原始字节验签，不能先解析再重新序列化。
- 时间戳和当前时间相差超过 5 分钟的，建议拒收。
- 用 `event_id` 去重，因为重试可能让同一事件到达两次。

**慢不拖累通话**：`call.turn` 超时就丢弃，`call.tool` 超时助理会说查不到，`call.handoff` 失败助理会说「记下了」。
你的接口慢或者挂了，主人那边的通话都照常进行。

### call.turn：看着每一轮，必要时纠正

```json
{"type": "call.turn", "event_id": "evt_…", "created_at": "…", "call_id": "c_…",
 "caller": "主人这一轮说的话", "agent": "助理这一轮的回答"}
```

- 不需要介入时，直接回 2xx、空响应体。大多数轮都应该这样。
- 发现助理说错了、漏了，或者有它该知道的事，就回 `{"note": "一句话说明"}`，助理从下一句开始照着做。
  - 纠正类的写清楚哪里错、正确的是什么，例如「更正：会议是四点不是三点，下次开口先跟他说一声」。
  - 补充类的直接写事实，例如「他明天的航班已改签到下午 3 点」。
- 3 秒内回不了的（比如要查一下），先回空，想好了再调 `update_call {call_id, note}`。通话还没结束就有效。
- note 每条最多 500 字，一通电话最多保留最近 20 条，以最新一条为准。
- `agent` 是助理生成的完整回答；主人中途打断时，他实际听到的可能更短。

### call.handoff：助理把事情交给你办

主人在电话里说了要查、要办的事，或者只有你才知道答案的事，助理会说一句「我去问一下」，同时推送：

```json
{"type": "call.handoff", "event_id": "evt_…", "created_at": "…", "call_id": "c_…", "handoff_id": "h…",
 "question": "查一下明天下午去北京的航班",
 "recent_turns": [{"who": "caller", "text": "…"}, {"who": "agent", "text": "…"}]}
```

- 先回 2xx 确认收到，**不要在这个请求里办事**。
- 异步去办，办完调 `answer_handoff {call_id, handoff_id, text}`。`text` 写结论，助理会用自己的话转述，所以写清楚、别太长。
- 助理等结果时会继续和主人聊别的；结果到了，它在当前这句说完后插进来转告。
- 你办完时通话已经挂断的，`answer_handoff` 返回 `delivered: "next_call"`：答案记在通话记录里，主人**下一次**接通电话（打进来或你打过去）时助理先告诉他。
  等不到下次的，另外发消息或 `place_call`。
- 推送三次都失败时，助理会告诉主人「没联系上，记下了，挂断后处理」。这件事仍在通话记录的 `handoffs` 里，你可以事后补办。

### call.tool：你的工具，助理直接用

适合「你那边能马上答上来」的事：主人的记忆、日程、待办、订单。比转交快得多，通常一两秒。
用 `set_context` 设 `tools`（整组替换，传 `null` 或 `[]` 清空，最多 8 个）：

```json
{"tools": [{"name": "query_memory",
            "description": "查关于主人的记忆：家人宠物、偏好、项目进展、以前说过的事。返回带日期的片段。",
            "parameters": {"type": "object", "properties": {"query": {"type": "string", "description": "要查的内容"}},
                           "required": ["query"]},
            "read_only": true}]}
```

- `name`：字母、数字、下划线，不能叫 `web_search` 或 `ask_owner_agent`（这两个是内置的）。
- `description` 最重要：助理**只**根据它决定什么时候调用，写清楚这个工具能查什么。
- `parameters`：JSON Schema 对象。
- `read_only`：只读的写 `true`。助理会在主人还没说完时就提前准备回答，提前准备的回答可能被作废。
  只读工具可以提前调用，所以更快；会改东西的工具，要等这句回答真正播出才调用。

助理调用时推送：

```json
{"type": "call.tool", "event_id": "evt_…", "created_at": "…", "call_id": "c_…", "tool_call_id": "call_…",
 "name": "query_memory", "arguments": {"query": "丽珠集团的成本价"}}
```

- 你要在 **5 秒内**回 2xx 和 `{"result": "给助理看的文字结果"}`。结果超过 4000 字会被截断。
- 超时、出错或不是 JSON 时，助理会如实说现在查不到。
- 同一句话可能调用两次（一次是提前准备的回答，一次是最终播出的回答）。只读工具不受影响。

### call.ended：通话结束

```json
{"type": "call.ended", "event_id": "evt_…", "created_at": "…", "call": { …和 get_call 的返回一样，含 transcript 和 handoffs… }}
```

没接通（`unanswered`、`missed`）、取消（`cancelled`）、失败（`failed`）也会推。这时适合做通话总结，并且把通话中主人更正过的信息存起来。

### 三种介入方式怎么选

| 情况 | 用什么 | 主人感受到的延迟 |
|---|---|---|
| 你一秒内就能确定助理哪里说错了，或者它该知道一件事 | `call.turn` 回 note | 下一句就改 |
| 要查一下才知道，但查询本身很快（≤5 秒，只读） | 注册成 `tools` | 一两秒，当场答上 |
| 要动手办、要花几十秒以上、要用到你那边的复杂能力 | 让助理转交（`call.handoff`），办完 `answer_handoff` | 先说「我去问一下」，办完插进来说 |
| 想到得晚、不在 3 秒窗口里 | `update_call` | 下一句就改 |

### 接收端示例（Python 标准库）

```python
import hashlib, hmac, json, threading, time, urllib.request
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer

MCP_URL, MCP_KEY = "https://phone.alexmk.cn/mcp", "pk_…"
SECRET = ""          # set_context 返回的 webhook_secret，存进你的 secret store
seen = set()

def mcp(name, args):
    body = {"jsonrpc": "2.0", "id": 1, "method": "tools/call", "params": {"name": name, "arguments": args}}
    req = urllib.request.Request(MCP_URL, data=json.dumps(body).encode(), method="POST",
                                 headers={"Authorization": f"Bearer {MCP_KEY}", "Content-Type": "application/json"})
    return json.load(urllib.request.urlopen(req, timeout=15))["result"]["content"][0]["text"]

def verify(headers, raw):
    ts = headers.get("X-Phone-Timestamp", "0")
    if not SECRET or abs(time.time() - int(ts)) > 300:
        return False
    expected = "v1=" + hmac.new(SECRET.encode(), ts.encode() + b"." + raw, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, headers.get("X-Phone-Signature", ""))

def work_on(event):                       # 在后台办事，办完回给电话
    answer = "…你的 Agent 办事的结论…"
    mcp("answer_handoff", {"call_id": event["call_id"], "handoff_id": event["handoff_id"], "text": answer})

class Hook(BaseHTTPRequestHandler):
    def reply(self, status, payload=None):
        data = json.dumps(payload, ensure_ascii=False).encode() if payload is not None else b""
        self.send_response(status); self.send_header("Content-Length", str(len(data))); self.end_headers()
        self.wfile.write(data)

    def do_POST(self):
        raw = self.rfile.read(int(self.headers["Content-Length"]))
        event = json.loads(raw)
        if event.get("type") == "webhook.test":
            return self.reply(200, {"challenge": event["challenge"]})
        if not verify(self.headers, raw):
            return self.reply(401)
        if event["event_id"] in seen:
            return self.reply(200)
        seen.add(event["event_id"])
        kind = event["type"]
        if kind == "call.turn":              # 3 秒内：要纠正就回 note，否则回空
            return self.reply(200, {})
        if kind == "call.tool":              # 5 秒内：回结果
            return self.reply(200, {"result": f"（{event['name']} 的查询结果）"})
        if kind == "call.handoff":           # 先确认，再异步办
            threading.Thread(target=work_on, args=(event,), daemon=True).start()
            return self.reply(200)
        if kind == "call.ended":             # 写总结、存主人更正过的信息
            return self.reply(200)
        return self.reply(200)

ThreadingHTTPServer(("0.0.0.0", 8080), Hook).serve_forever()
```

Node 验签：

```js
import crypto from 'node:crypto';

export function verify(secret, headers, rawBody /* Buffer */) {
  const ts = headers['x-phone-timestamp'];
  if (!secret || Math.abs(Date.now() / 1000 - Number(ts)) > 300) return false;
  const expected = 'v1=' + crypto.createHmac('sha256', secret).update(`${ts}.`).update(rawBody).digest('hex');
  const got = String(headers['x-phone-signature'] ?? '');
  return got.length === expected.length && crypto.timingSafeEqual(Buffer.from(got), Buffer.from(expected));
}
```

## 分层：谁管什么

| 层 | 谁负责 | 内容 |
|---|---|---|
| 通话规则和内置工具 | 电话服务 | 怎么说话（口语、短句）、联网搜索、转交给你（copilot 模式） |
| 人设 `persona` | 你 | 助理是谁、什么风格、哪类事该先查什么或交给你。**不写工具名** |
| 工具 `tools` | 你（copilot 模式） | 每个工具的名字和说明，说明决定什么时候用 |
| 背景 `text`、通话中 note | 你 | 主人的情况和这几天的事；通话中的补充和纠正 |

例如人设里写「主人的私事先查记忆，查不到再交给小管家」，再注册一个说明为「查关于主人的记忆」的工具，助理就会先调它，查不到再转交。
同一份人设还可以给你自己的其他语音入口用，工具名各自不同也没关系。

### 这些我们已经内置，你不用写

助理每通电话的 prompt 由我们拼好，你的内容嵌在里面。下面这些是内置的。在 persona 里重复写只会占篇幅；写了相反的意思，会和内置规则打架，助理的表现就不可预测。

| 内置内容 | 具体是什么 |
|---|---|
| 身份 | 「你是<agent_name>，<owner_name>的私人助理。现在你在电话里和<owner_name>本人说话。」 |
| 电话说话规则 | 每个字都会被念出来：口语、短句、一次一两句；不用 markdown、列表、表情、括号注释；不知道就说不知道，不要编 |
| 联网搜索 | 公开信息和实时数字（新闻、天气、股价、版本、百科）先搜再答，以日期最新的为准；背景里记下的这类数字不能当成现在的说 |
| 转交给你 | copilot 模式：要查要办的事、只有你知道的事，交给你，先说一句「我去问一下」，结果没回来不编。基础模式：说「记下了，挂断后交给<owner_agent_name>处理」 |
| 系统事件 | 以「（系统：」开头的消息是我们插进来的提示，不是主人说的话 |
| 这通电话的框架 | 外呼：「是你主动打给<owner_name>的。来意：…要点：…」；呼入：「是<owner_name>打进来的。」 |
| 当前时间 | 北京时间、日期、星期几 |
| 开场白 | 外呼「喂，<owner_name>，我是<agent_name>。」；呼入默认「你好，我是<agent_name>。」（可用 `greeting` 改） |
| 接通后说来意 | 外呼接通、问候完，提示助理用一两句话直接说来意 |
| 转交结果的转述 | 结果回来、没送出去、上次挂断后才办完，都由我们提示助理用口语转告 |
| 内置工具的说明 | `web_search`、`ask_owner_agent` 什么时候用，由它们自带的说明决定 |

**你要写的，是只有你知道的东西：**

- `persona`：助理的性格和说话风格（「干脆、不客套」「叫他王总」）；这个主人的事该怎么处理（「私事先查记忆，查不到再交给我」「涉及钱的事一律交给我确认」）。
- `text`：主人是谁、偏好、这几天的安排和在忙的事。
- `place_call` 的 `purpose` 和 `brief`：这一通为什么打、要传达什么、拿到什么算完成。
- `tools` 的 `description`：你的工具能查什么。
- 通话中的 note：只针对这一通的补充和纠正。

不用写「你在打电话」「回答要简短」「不要编」「现在几点」「天气要去搜」，这些都已经有了。

## 限制与常见错误

| 情况 | 返回 / 做法 |
|---|---|
| 不在外呼时段 | `error: outside_hours …`。时段以 `whoami.calling_hours` 为准，目前是北京时间 8–20 点。这时由你决定改走别的渠道，不要默默推迟到第二天 |
| `at` 已经过去，或超过 7 天 | `error: …`，请修正时间 |
| `call_id` 不存在或不属于你 | `error: no such call` |
| key 错误或缺失 | HTTP 401 |
| 主人没接 | 状态为 `unanswered`，按 `retries` 重拨；用完以后要不要换渠道，由你决定 |
| webhook 测试不通过 | `error: webhook test failed …; nothing was changed`，整次调用不生效 |
| 没有 webhook 就设 `tools` | `error: tools are called over the webhook: set webhook_url too` |
| `update_call` 时通话已经结束 | `error: the call is not in progress` |
| `handoff_id` 不对 | `error: no such handoff on this call` |

## 隐私

电话服务只保存通话需要的东西：brief、配置、转写、转交记录。主人的记忆和其他隐私都留在你这边。
copilot 模式下，工具结果和转交答案只在这通电话里用于回答，不会另外存储，转交答案会记在通话记录里。
不要把密码、证件号这类内容写进 brief、常驻背景或工具结果。
