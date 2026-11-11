# Gemini 4 Carbon 中转 API：国内直连谷歌最强推理模型，`gemini-4-carbon` 怎么调

Gemini 4 Carbon 是谷歌目前最强的模型。10 月 28 日发布时先只给了一批网络安全防御者，走的是 Fairwind 计划；11 月 10 日才正式对所有付费开发者和企业开放。它的定位和 Argon 不一样：Argon 覆盖大多数软件工程和知识工作，Carbon 专门啃那些要榨干模型智力的活——多步编码调试、跨子系统根因定位、开放式科研。

`generativelanguage.googleapis.com` 在国内直连不通，这个仓库说的是怎么通过 Token173（https://token173.net）的国内入口调 `gemini-4-carbon`，Gemini 原生协议和 OpenAI 兼容协议都能走。

## Carbon 和 Argon 怎么选

| | Gemini 4 Argon | Gemini 4 Carbon |
|---|---|---|
| 定位 | 日常主力，覆盖面广 | 极限任务，深度优先 |
| token 消耗 | 省 | 多，会自己反复验证 |
| 官方单价 | $2 输入 / $10 输出 | $2 输入 / $10 输出（同价） |
| 适合 | 一般开发、文档、日常问答 | 长任务、漏洞挖掘、复杂重构 |

两个现在同价，这是首发优惠期（缓存输入另享 95% 折扣）。优惠结束后 Carbon 会回到常规价 $4 输入 / $20 输出，那时候按任务挑就更重要了。

结论很简单：**日常别用 Carbon，贵在 token 消耗**；真要啃硬骨头，它的单任务总成本反而更低——因为它一次能跑对，而小模型要试很多次。

## 强在哪，具体数字

| 评测 | Carbon | 说明 |
|---|---|---|
| Artificial Analysis 智能指数 v5.0 | 59 分，第 1 | 综合榜第一 |
| SEC-bench Pro（找漏洞） | 88.2% | 同类最高 |
| ExploitGym（验证可利用性） | 54.8% | 判断漏洞真能不能打 |
| CWE-bench（根因修补） | 71.0% | 直接产出可用补丁 |
| CursorBench 4.0 | 第 1 | 单任务成本比第二名低一半以上 |
| Perplexity WANDR | 第 1 | 大规模检索类任务 |
| Zapier AutomationBench | 第 1 | 自动化流程 |
| WebDev Arena | 1768 分，第 4（+89） | 全栈网页生成 |
| MKA 多模态知识智能体 | 第 2 | 单任务成本是最强的六分之一 |

诚实度这一块也有数据：375 项智能体诚实度测试通过 92%（Argon 70%），缺数据时 98% 的情况会拒绝编造，68% 的情况会主动指出方法学缺陷。Gray Swan 的间接提示注入测试攻击成功率 0.0%（Argon 是 0.7%），Terminal-Bench 4.0 上的作弊成功率被压到 0.0%。

对做安全的人来说，这些数字的实际含义是：它不会给你一堆看起来像漏洞的噪音，会把站不住的结论自己筛掉。Google 内部 Project Fortify 用它的结果，扫描召回 100%、分诊精度 95%；Wiz 扫描两家大型 SaaS，两小时内找到 0-day，原本要几周。

## Gemini 原生协议调用

```bash
curl "https://token173.net/v1beta/models/gemini-4-carbon:generateContent" \
  -H "x-goog-api-key: sk-你的令牌" \
  -H "Content-Type: application/json" \
  -d '{
    "contents": [{"parts": [{"text": "这个 Go 服务在并发 2000 时偶发 goroutine 泄漏，帮我列出最可能的三个根因和验证方法"}]}],
    "generationConfig": {
      "maxOutputTokens": 4096,
      "thinkingConfig": {"thinkingLevel": "high"}
    }
  }'
```

注意几个点：
- 模型名在 URL 里，冒号后面是动作，`generateContent` 不能省
- 鉴权用 `x-goog-api-key` 头，不是 `Authorization: Bearer`
- `thinkingLevel` 只有 `low` / `medium` / `high` 三档，Carbon 没有 `minimal`

## OpenAI 兼容协议调用

SDK 不换就能用：

```python
from openai import OpenAI

client = OpenAI(api_key="sk-你的令牌", base_url="https://token173.net/v1")

r = client.chat.completions.create(
    model="gemini-4-carbon",
    messages=[{"role": "user", "content": "读这段 Rust 代码，找出可能导致 UAF 的地方，给出最小修复"}],
    max_tokens=8192,
)
print(r.choices[0].message.content)
```

跑批量或者接现有应用，走这条最省事。原生协议里的 `thinkingLevel`，兼容协议下对应请求体的 `thinking` 字段，具体看文档站的参数表。

## 流式输出

```bash
curl "https://token173.net/v1beta/models/gemini-4-carbon:streamGenerateContent?alt=sse" \
  -H "x-goog-api-key: sk-你的令牌" \
  -H "Content-Type: application/json" \
  -d '{"contents": [{"parts": [{"text": "写一份跨包迁移方案，分阶段列出来"}]}]}'
```

`alt=sse` 不能漏，漏了返回的是整包 JSON 数组，不是流。

## 长上下文和长输出

Carbon 的统一上下文窗口是 1,048,576 tokens，约等于一本 700 页的技术手册全塞进去。同步请求最大输出 65,536 tokens，超出这个量就得走 Long Decode——它的长解码机制能把输出拉到 1,048,576 tokens。

实际用的时候注意两点：

- 输入塞满 1M 会让首 token 延迟明显变长，能裁的文档先裁
- 长输出一定要配 `maxOutputTokens`，默认值远小于你想象的上限，容易半截断掉

## 推理档位怎么省钱

| 档位 | 什么时候用 |
|---|---|
| `low` | 格式转换、抽取、简单问答 |
| `medium` | 大多数编码任务，性价比最高 |
| `high` | 根因定位、架构设计、漏洞链推理 |

同一个 prompt，从 low 拉到 high，思考 token 能翻几倍。做批量任务时先用 low 跑通流程，确认输出结构对了再升档。

## 计费

Carbon 在站点上的倍率是输入 1、补全 5，对应官方 $2 / $10 每百万 tokens。账户 2 元 = 1 美元，折算下来是输入 4 元、输出 20 元每百万 tokens。缓存输入打 95% 折扣，重复喂同样的长文档时差别很大。

和 Claude Fable 5.2 对比一下：Fable 是 $8 / $40，Carbon 是 $2 / $10，正好是四分之一。同样的长任务，两边都可以试，成本差四倍。

## 什么活值得用它

| 任务 | 建议 |
|---|---|
| 遗留系统迁移、跨多个包的依赖重构 | 用它，能连续跑几天不放弃 |
| 线上偶发故障的根因定位 | 用它，会自己构造验证步骤 |
| 代码库安全审计、漏洞可利用性验证 | 用它，SEC-bench 那套能力就是干这个的 |
| 全栈小应用从零做出来 | 用它，跑完自己会验证渲染结果 |
| 蛋白质、材料、物理类科研推理 | 用它，异构序列检索这类活实测有效 |
| 日常问答、写文案、改文档 | 别用，`gemini-3.8-flash` 或 `claude-sonnet-5-5` 够了 |

## 常见问题

**Q：返回 404 是什么原因？**
多半是模型名写错或者 URL 少了动作段。原生协议必须是 `/v1beta/models/gemini-4-carbon:generateContent`，`-` 不能写成 `.`。

**Q：403 提示无权限？**
Carbon 只对部分分组开放，看令牌绑定的分组。站点上它在 `Aistudio-Gemini-2` 到 `4`、`Vertex-Gemini-2` 到 `3` 这几组，`gemini-4-argon` 的组更多。换到有权限的分组再试。

**Q：能当日常主力模型吗？**
不建议。同样一个任务，它为了验证会多花几倍 token，快问快答的场景明显不划算。

**Q：输出总是被截断？**
两个原因：没设 `maxOutputTokens`，或者设了但超过同步上限。同步最多 65,536，要更长就走 Long Decode。

**Q：支持图片、视频输入吗？**
支持，`parts` 里塞 `inlineData`（base64）或 `fileData`（文件 URI）都行。读图表、看架构图这类任务是它的强项，MKA 榜第二。

**Q：和 Cursor、Cherry Studio 这类客户端怎么配？**
供应商选 Gemini，API 地址填 `https://token173.net`，模型填 `gemini-4-carbon`。走 OpenAI 兼容的话地址填 `https://token173.net/v1`。

## 相关

- 官方发布说明：https://blog.google/innovation-and-ai/models-and-research/gemini-models/gemini-4-carbon-now-available/
- 文档站：https://docs.token173.net
- Gemini 全系接入：[gemini-api](https://github.com/Token173/gemini-api)
- Gemini 3 Pro 单独一篇：[gemini-3-pro-proxy](https://github.com/Token173/gemini-3-pro-proxy)
- Claude 前沿模型对照：[claude-fable-5.2](https://github.com/Token173/claude-fable-5.2)
