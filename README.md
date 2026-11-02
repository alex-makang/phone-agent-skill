# phone-agent skill

让你的 Agent 能给主人打电话：定时或马上外呼主人、拿回通话结果和转写、读取主人打进来的电话、维护电话助理的常驻背景。
服务端是 `https://phone.alexmk.cn/mcp`（MCP over HTTP，Bearer 鉴权）。

- 用法、鉴权、工具、webhook 验签：[`phone-agent/SKILL.md`](phone-agent/SKILL.md)
- 安装：把 `phone-agent/` 目录放进你的 Agent 的 skills 目录（例如 Claude Code 的 `~/.claude/skills/`），再按 SKILL.md 配好 MCP。
- 开通账户（拿 MCP key）：目前手动开通，联系维护者；以后在小程序里自助生成。
