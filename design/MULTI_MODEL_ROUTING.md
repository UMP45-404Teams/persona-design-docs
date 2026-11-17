# 多模型路由设计（MULTI_MODEL_ROUTING）

> **状态：草案，已敲定框架 + 输出格式，若干细节待拍板（见 §7）。未批准不落代码。**
>
> **来源**：我 2026-11-01 提出——「思考和回复用不同模型」「视觉模型单独做」，
> 参考 maibot 的模型类别面板（replier/planner/memory/utils/vlm/embedding/learner 等）。
>
> **一句话方向**：多模型的价值不在「配很多」，而在「该省的地方省、该强的地方强」。
> 回复用好的、打分用同一个、摘要用便宜的、向量单独接。

---

## 0. 决策速查（已敲定）

| # | 决策 | 依据 |
|---|---|---|
| 1 | 档位 7 个：`reply` / `think` / `summary` / `learner` / `vision` / `web_search` / `embedding` | 我认可 |
| 2 | 缺省一律回落到全局 `model_name`（= maibot「留空时用 utils」） | 现有 `stage_model` 已是这个语义 |
| 3 | **打分相关不设独立档**（importance / impulse / mood） | 我担心「不同模型打分有偏移」 |
| 4 | **think/reply 不拆成两次调用** | 输入占 96.8%，拆开输入翻倍，得不偿失（§3） |
| 5 | **复杂场景用 PRO = 同一次调用换模型**，不是两次 | 我的原话认可 |
| 6 | embedding 用硅基流动 **BGE-M3**（月免费 1000 万 token） | 我找到 maibot 现成配置 |
| 7 | 重要性分布目标：**贴近正态分布**（阈值后续按分布调） | 我直觉，非理论 |
| 8 | 百分位方案**暂缓**，作为未来需求进文档 | 我：「先不做，直接作为未来需求」 |
| 9 | gossip（碎片/冲动）**暂不开**，已登记台账 | 我：「先不打开，但记住没开」 |
| 10 | **普通回复 = flash 4.1；视觉 = flash 4.1；联网搜索 = flash 4.1** | 我 2026-11-01 拍板 |
| 11 | **复杂场景 = V4 PRO** | 我拍板 |
| 12 | **黑话/表达/行为学习 = V4 PRO**（确保学习质量） | 我拍板 |
| 12b | **summary 单独给设置项**，不填回落主回复模型（flash） | 我：「单独给个设置项，可设置，不设置就回落主回复模型」 |
| 13 | **联想（embedding）本期做**（模型已定 BGE-M3） | 我：「确定了模型的话可以很快做掉」 |
| 14 | **模型能力做成可扩展的类插件注册表**（接生图/语音等） | 我：「得做成开放式的类插件的可拓展的」 |
| 15 | **COMPLEXITY 判断一直开**（所有触发后都判，不只在热聊） | 我：「前置判断已经确定了值得聊，这是质量问题」 |
| 16 | **学习异步跑**（回复 flash 4.1 / 学习 V4 PRO，模型不同=必须异步） | 我确认 + 模型不同硬约束（§2.2） |
| 17 | **PRO 重写不丢旁路数据**（NEXT_INTERVAL/MOOD/PROFILE 等重新输出） | 我：「不能忘记其他需要 LLM 在回复时就生成的东西」 |
| 18 | 黑话学习**数据库化**，按卡分开，可导入他人库，导入用 agent 清洗 | 我拍板 |
| 20 | 复杂度判断由 **flash 自己判**（不用客观信号，prompt 锚定「多人多话题」） | 我：「只要提一嘴……让它不要默认是简单场景」 |
| 21 | **多模型提前到阶段 0**（相关功能直接依赖多模型） | 我：「多模型应该提前到现在的吧」 |
| 22 | embedding API key **已提供**（硅基流动，2026-11-01） | 我贴出 key |
| 23 | `aliases=[]` 私聊称呼 **明确要修** | 我：「这么在意那就明确要修」 |
| 24 | **GitHub 发布**：脱敏 + 即插即用模板 | 我：「上传 GitHub 的那份肯定是脱敏的同时方便部署」 |
| 25 | **插件生态愿景**：预留拓展性，不承诺维护 | 我：「没有创意工坊，没有 mod 就很没意思」 |
| 26 | **酒馆角色卡导入 = 正式功能 + 对外亮点**（同构转译、几乎无损；世界书复用黑话库知识条目管道；跑团仅设想） | 我 2026-11-02，见 `TAVERN_IMPORT_DESIGN.md` |

---

## 1. 现状盘点（2026-11-01 实测）

### 1.1 已有基础（半个系统）

- `dsh/llm/routes.py` 已有 `stage_model(core_cfg, stage)`，但**只认 `vision` / `web_search`** 两档
- `api_profiles`（secrets.json）已支持 `vision_model_name` / `web_search_model_name` 字段
- `vision.py:107` 和 `think.py:433` 已在用这两个档

### 1.2 缺口

- `auxiliary.py` 有 **12 处硬编码** `core_cfg.get("model_name")`
- 无 `reply` / `think` / `summary` / `learner` / `embedding` 档
- `api_profiles` 是平铺结构，无法表达「某档位用不同 endpoint + key」（embedding 需要）

### 1.3 真实 token 构成（为什么 think/reply 不能拆两次）

| 用途 | 次数 | 平均输入 | 平均输出 | 出/入比 |
|---|---|---|---|---|
| group | 2577 | 6,353 tok | 178 tok | 0.028 |
| private | 1303 | 11,298 tok | 162 tok | 0.014 |
| summary | 258 | — | — | 0.109 |

**总量：输入占 96.8%，输出仅 3.2%。**

→ 若拆「think 一次 + reply 一次」，上下文要送两遍，**输入翻倍、总成本接近翻倍**，
而想省的输出只占 3.2%。**这就是决策 #4 的数据依据。**

---

## 2. 档位定义

| 档位 | 用途 | 模型 | 说明 |
|---|---|---|---|
| `reply` | 回复（群聊/私聊/主动开口） | **flash 4.1** | 普通回复 |
| `reply_complex` | 复杂场景回复 | **V4 PRO** | 同一次调用换模型（§3） |
| `think` | 思考/工具决策 | = reply | 不单独配 |
| `summary` | 长期记忆摘要 | flash（**单独给设置项，可配可不配**） | 不填回落 `reply` |
| `learner` | 学习（黑话/行为/表达） | **V4 PRO** | 学习质量优先（决策 #12） |
| `vision` | 视觉 | **flash 4.1** | |
| `web_search` | 联网搜索 | **flash 4.1** | |
| `embedding` | 向量 | **BGE-M3（硅基流动）** | 必须单独接 |

**打分（importance/impulse/mood）故意不进表** —— 决策 #3：换模型必偏移，
最安全就是打分永远用同一个模型（当前主模型）。

## 2.1 模型能力注册表（类插件，决策 #14）

> **我**：「得做成开放式的类插件的可拓展的；以后我们可能需要接入生图甚至语音模型。」

**设计原则**：档位**不是写死的 7 个**，而是一张「能力注册表」：

| 能力类型 | 当前模型 | 未来可加 |
|---|---|---|
| `text` 文本生成 | flash 4.1 / V4 PRO | 任何对话模型 |
| `vision` 视觉 | flash 4.1 | 任何多模态模型 |
| `embedding` 嵌入 | BGE-M3 | 任何向量模型 |
| `image_gen` 生图 | （空） | **待接入** |
| `voice` 语音 | （空） | **待接入** |
| `...` | | 任意新能力 |

**规则**：
1. 每个能力类型**可挂多个模型**（主/备/按档）
2. 留空 → 回落到 `text` 的默认模型
3. 新增能力类型**不改核心代码**，只加配置 + 一个调用函数

> **实现（2026-11-01 已敲定落地方式）**：
> - **第一步（本次做）**：平铺字段 `{stage}_model_name` + `embedding` 独立对象
>   （因为 embedding 是不同厂商 endpoint+key，不能混进对话 core_cfg）。兼容旧配置。
> - **第二步（将来插件化时再做）**：升级为真正的「按能力类型分组」注册表 JSON。
> - 新旧之间用 `stage_model()` 的映射表做胶水：加档位只改映射表，不改调用方。

## 2.2 学习为什么必须异步（决策 #16）

> **我**：「理论上黑话学习、行为学习、表达风格学习，其实都应该在回复的时候直接做了；
> 要不然，回复的时候只判断是否值得学，后续再跑？这三个东西我没有头绪，还是得看看别人怎么做的。」

**结论：异步，而且回复时连「是否值得学」都别判断。** 两条理由：

1. **模型不同的硬约束**：普通回复 = flash 4.1，学习 = V4 PRO。一次 LLM 调用只能用一个模型，
   学习不可能在回复那一次调用里完成——否则学习只能跟着 flash 4.1，违背「学习用 PRO 保质量」。
2. **社区同款**：maibot 截图里 `learner` 是独立档位，就是异步单独跑的。

**链路**：

```
回复（flash 4.1）→ 正常回复 + 旁路数据（MOOD/PROFILE/NEXT_INTERVAL）
        ↓ 与回复解耦
learner（V4 PRO 异步）→ 自行拉取近期上下文 → 判断并沉淀黑话/行为/表达
```

**学习判断由 PRO 自己看上下文做**，不依赖 flash 在回复时打标记——隔离、更准、不拖慢回复。

### 2.2.0 学习必须带人设（2026-11-01 我补充）

> **我**：「不带人设文件的话，那么学习会变味。效果可能会不太贴合角色卡。」

**learner 的输入必须包含对应 persona 的完整文本**，否则 PRO 会用「通用 AI 口味」学习，
学出来的东西和角色卡对不上（同一个黑话，角色B和角色A的用法不同）。

| 学习类型 | 用哪张卡的人设 |
|---|---|
| 黑话学习 | 该群绑定的群聊卡 |
| 行为学习（auto_reply） | 该流绑定的卡 |
| 表达学习（profile_facts tag） | 说话的那张卡 |

> 连带代价：persona 文本较长（角色B override 7804 字符），学习输入会变大。
> **先保证「不变味」，成本以后再优化。**

### 2.2.1 黑话学习数据库化（决策 #18）

> **我**：「黑话学习可以做成一个数据库的形式；理论上学习结果按照角色卡分开，
> 但我们可以导入其他人分享的数据库；只不过导入操作可能得用 agent 清洗一下。」

- 黑话库**按角色卡隔离**（`persona_id` 维度）
- 支持**导入他人分享的库**（黑话 → 释义 → 使用场景）
- 导入**先经 agent 清洗**（去重、去冲突、过滤不适用条目）再落库
- 落地文件见 `AIreadme/UNVERIFIED_FEATURES_DIAGNOSIS.md`（群黑话诊断条目）

## 2.3 API 优先级（决策 #19）

> **我**：「中转站虽然不稳，但是问题不大。而且现在应该是优先调用的官方 API。」

- **官方 API 优先**，中转仅为备用
- 备用中转的密钥失效（401）——**不修**，因为它只是备用，且不稳
- 备用通道保留「可用则用、不可用静默回落官方」的语义即可

---

## 3. 复杂场景用 PRO（核心机制）

### 3.1 为什么不是「think 拆出来」

见 §1.3。拆两次输入翻倍。

### 3.2 真正的做法：同一次调用换模型

```
防抖批次 → 触发（随机 0.1 / @ / 热聊）→ 调 LLM
   ↓
LLM 一次输出（三选一，互斥）：
   [SKIP] [NEXT_INTERVAL:秒]        ← 先判断：不参与就到此为止
   [COMPLEXITY:1] 回复内容           ← 参与且简单 → flash 结果直接用
   [COMPLEXITY:3] 回复内容           ← 参与且复杂 → 拿去给 PRO 重写
```

**关键约束（我点明）**：

1. **SKIP 靠前生效**——先 SKIP 掉，复杂度判断就不用做了
2. **复杂度判断和回复在同一次调用**——不能先判断再生成（会拆成两次）

### 3.3 简单场景零额外成本，复杂场景才两次调用

- `COMPLEXITY:1` → flash 这次输出直接发，**零额外成本**
- `COMPLEXITY:3` → 再用 PRO 生成一次（第二次调用，输入不含 flash 输出则成本可控）

### 3.4 COMPLEXITY 判断一直开（决策 #15，替代「只在热聊」）

> **我**：「COMPLEXITY 这个我建议一直都可以，因为前置判断已经确定了值得聊，这是质量问题。」

**修订**：不只在热聊（participation）模式，而是**所有触发后的回复都输出 `[COMPLEXITY:n]`**。
前置触发（随机 0.1 / @ / 热聊）已筛掉「不值得聊」，到 LLM 这一步就是质量问题，
复杂度判断的额外输出只多几个 token，**值得一直开**。

### 3.5 PRO 重写不丢旁路数据（决策 #17）

> **我**：「PRO 重写如果不带 flash 的回复是对的，但是不能忘记其他需要 LLM 在回复时就生成的东西，
> 比如后续什么时候再看一眼；应该还有一些其他需要判断的东西。」

**PRO 重写（COMPLEXITY:3）时，必须重新输出完整的旁路数据**，不能只回一个正文：

| 旁路数据 | 标记 | 说明 |
|---|---|---|
| 下次评估间隔 | `[NEXT_INTERVAL:秒]` | 后续什么时候再看一眼 |
| 好感变化 | `[MOOD:+N]` | |
| 用户画像 | `[PROFILE]{...}[/PROFILE]` | |
| 引用/表情意图 | `[REPLY]` / `request_sticker` 等 | |

**即**：`COMPLEXITY:3` 时 PRO 的第二次调用，等价于**一次完整的回复**，
只是不参考 flash 的正文；所有旁路数据一律由 PRO 重新生成，避免丢失。

### 3.5b 完整输出清单（2026-11-01 实测核对，PRO 重跑时一个都不能漏）

> **我**：「实际上我们大量的功能都依赖 LLM 在回复时给出的输出，里头有很多虽然没发出来，
> 但很重要的东西……转用 PRO 来输入输出的，后面这些其他，一个都不能漏。」

**一次正常回复里，LLM 除正文外的全部产出**（逐项核对自 `prompt.py` / `purify.py` / `think.py`）：

**A. 正文内的方括号标记**（`purify_reply` 剥离成动作）：

| 标记 | 作用 |
|---|---|
| `[REPLY]` | 引用回复意图 |
| `[SKIP]` | 本轮不参与 |
| `[NEXT_INTERVAL:秒]` | 下次评估间隔 |
| `[MOOD:+N/-N]` | 好感变化 |
| `[POKE]` | 戳一戳 |
| `[SEND:file_key]` | 发文件（仅我） |
| `[PROFILE]{...}[/PROFILE]` | 画像增量（facts/tags/mood_reason） |

**B. 工具调用**（`tool_calls` → 后续动作）：

| 工具 | 后续 |
|---|---|
| `request_mention` / `request_sticker` | 第二段 LLM 选人/选图 |
| `set_mood` | 落好感 |
| `query_cognitive` | 转 planner 查库 |
| `web_search` | 第三段改写 |
| `query_config` / `modify_config` / `update_cognitive` / `request_file` | 仅我 |

**C. 内部字段**：`reasoning_content`（推理内容，泄漏会拦截）。

**处理铁则**：

| 场景 | 处理 |
|---|---|
| `COMPLEXITY:1` | flash 输出**整体采用**（正文 + A + B + C 全部） |
| `COMPLEXITY:3` | flash 输出**整体作废**，PRO 完整重跑 `_route_with_messages`，PRO 的输出（正文 + A + B + C + 第二段选择 + 第三段改写）全部采用 |

> **不存在「只重写正文」的中间态。** 这就是我给的框架：
> `LLM调用 → 判断skip → 判断复杂度 → 其他` —— 复杂度触发通过后转 PRO，
> 「其他」一个都不能漏。

### 3.6 复杂度判断：由 flash 自己判，不用客观信号（2026-11-01 定）

> **我**：「flash 能力不是那么强，但判断场景应该能有效做到吧，
> 只要提一嘴可能是多人多话题，让它不要默认是简单场景，应该就没问题的吧。」

**定案**：不再用「人数/@数/话题聚类」等客观信号，直接让 flash 在回复时判断复杂度。
客观信号是代理指标，可能误判（3 人聊 4 个话题，人数信号就漏了）；
而「多人多话题」就写在上下文里，flash 看一眼就知道。

prompt 锚定语（拟）：

> 「群里可能是多人多话题交叉。判断本次场景复杂度时，**不要默认是简单场景**；
> 确实复杂就标 [COMPLEXITY:3]，不要为了省事全标 1。拿不准时宁可按复杂处理。」

**代价**：flash 误标 3 会多花一次 PRO 调用。这是「买质量」的合理成本，
且只在标 3 时才发生；标 1 零额外成本。

**注**：SKIP 机制已存在（`prompt.py:497-519` 第 7/8/9 条），本次是**在既有输出上多一个字段**。

---

## 4. embedding（BGE-M3）

| 项 | 值 |
|---|---|
| 厂商 | 硅基流动 SiliconFlow |
| 模型 | BGE-M3（1024 维） |
| 接口 | `https://api.siliconflow.cn/v1/embeddings`（OpenAI 兼容） |
| 免费额度 | **1000 万 token/月** |
| 需求估算 | 320 条记忆 × 4KB ≈ 1.3MB；额度够用极久 |

### 4.1 落地方式

- **向量存表**（`memory_summaries` 加 `embedding BLOB` 列，float32），写入时算一次
- 联想触发：**写记忆时离线批量**（不做实时）
- **本期做**（决策 #13 / #21）：embedding 数据层 + 联想 + dormant 关联唤醒

### 4.2 未敲定

- API key 谁提供、放哪个 secrets 字段（**待我提供**）

---

## 5. 打分分布（正态分布目标）

> **我**：「我们的设计，应该要让最后的结果贴近正态分布；
> 这不是什么理论，只是我觉得应该这样；后续这块还得具体调整分数阈值。」

### 5.1 现状（2026-11-01 实测）

`memory_summaries` 的 importance 分布：

| 值 | 2 | 4 | 5 | 6 | 7 | 8 | 9 |
|---|---|---|---|---|---|---|---|
| 条数 | 4 | 7 | **46** | 9 | 11 | 9 | 11 |

**峰值压在 5 分**——典型「保守打分」。目标是从这里调成更接近正态。

### 5.2 陷阱（必须写死）

**不能为了正态而正态。** 若一周都很平淡，「这周没有重要的事」是真实结果，
不能强行凑出 9/10 分。判断标准：**分数高的那几条，翻回去看是否真的重要。**

### 5.3 百分位方案（暂缓，未来需求）

> **我**：「现在还不够，先记录数据；等积累够了，再自动切换。这样不用你操心。……那也干脆先不做，直接作为未来需求进入文档。」

未来需求（暂不实现）：

```
样本 < 200 条 → 用固定阈值（现有 6/7/8）
样本 ≥ 200 条 → 滚动窗口（最近 500 条）算 P60/P85/P95，阈值改百分位
```

**抗偏移原理**：换模型时分布自动重算，阈值语义不变（「中等重要/很重要/极重要」）。

---

## 6. 实施顺序（2026-11-01 我拍板：多模型提前）
> **我**：「多模型应该提前到现在的吧，相关功能直接依赖多模型。」

**为什么多模型是地基**：记忆衰减的「关联唤醒」要 embedding、复杂场景要换 PRO、
learner 要独立模型——**三者全挂在多模型上**。先做多模型，后面才不用返工。

| 阶段 | 内容 | 依赖 |
|---|---|---|
| 0 | **多模型地基**：`stage_model()` 通用化 + 能力注册表 + 配置结构升级 | 无 |
| 1 | 复杂场景换 PRO（`COMPLEXITY` 字段） | 0 |
| 2 | learner 独立档 + 学习功能切 PRO | 0 |
| 3 | embedding 数据层 + 联想（本期做） | 0（等 key） |
| 4 | 记忆衰减（稳定度/间隔效应/dormant，关联唤醒依赖 3） | 3 |
| 5 | 好感度收尾（配置接入 → 调用方适配 → 重置） | 无（可插队） |
| 6 | 心情系统 | 无（可插队） |
| 7 | **酒馆角色卡导入**（正式功能 + 对外亮点，设计定稿，见 `TAVERN_IMPORT_DESIGN.md`） | 5（黑话库知识条目管道） |

> **说明**：好感度（5）和心情（6）不依赖多模型，随时可插队；
> 但主线上多模型排第一，因为它被 1/2/3/4 共同依赖。

## 6.5 阶段 0 实施记录（2026-11-01/02）

**已改代码（本地，未上服务器）**：

| 文件 | 改动 |
|---|---|
| `dsh/llm/routes.py` | 白名单三表加 `reply/think/summary/learner` 四档字段；新增 `STAGE_MODEL_KEYS` 映射表（档位 → 字段 + 回落链）；`stage_model()` 泛化为按映射表回落；新增 `embedding_cfg()`（独立 endpoint+key）；`for_account()` 透传 `embedding` 子对象 |
| `dsh/services/core/auxiliary.py` | 10 处硬编码 `model_name` → `stage_model(core_cfg, 档位)`：summary 用 `"summary"`，其余（rewrite/peek/bili×3/qzone/llm_chat）用 `"reply"` |
| `dsh/services/core/think.py` | `_prepare` 的 model 取 `stage_model(core_cfg, "reply")` |

**验证**（独立数据目录，未碰生产）：
- ast.parse 三轮全过（语法 / 引用完整性 / 行为不变量）
- 本地行为验证 24 项全过（回落链、空串视未配、embedding 透传、for_account 合并）

**未做**：
- 配置补档（`secrets.json` 加新字段 + embedding）——待上服务器前一起做
- 服务器部署 —— 需我点头

**待办提醒**：`COMPLEXITY` 字段、learner 档接学习、embedding 数据层尚未动（属阶段 1/2/3）。

## 6.6 阶段 1 实施记录（2026-11-02）

**已改代码（本地，未上服务器）**：

| 文件 | 改动 |
|---|---|
| `dsh/llm/routes.py` | 加 `reply_complex` 档（`reply_complex_model_name`，回落 `reply_model_name` → `model_name`） |
| `dsh/domain/prompt.py` | 输出格式加第 10 条：回复开头输出 `[COMPLEXITY:1/3]`，锚定「多人多话题别默认简单、拿不准按复杂」 |
| `dsh/services/core/purify.py` | actions 加 `complexity` 字段，解析 `[COMPLEXITY:1/3]`（非法值忽略，正文由现有 `strip_internal_meta` 兜底剥除） |
| `dsh/services/core/think.py` | 拆出 `_run_once()`（单次完整调用+提纯）；`_route_with_messages` 变复杂度路由：SKIP 或复杂度≠3 → 用 flash；=3 且配了独立 PRO 档 → 整体作废 flash、PRO 完整重跑；PRO 失败回落 flash |

**核心铁则落地（决策 #17）**：
- COMPLEXITY:3 → flash 输出**整体作废**，PRO 完整重跑 `_run_once`（正文 + 全部旁路标记 + 工具 + 第二段选择），一个不漏
- PRO 未配置（reply_complex 回落 reply）→ **不重跑**，避免同模型白花一次钱
- PRO 重跑失败 → 回落 flash 首次输出（不丢回复）

**验证**（独立数据目录，未碰生产）：
- ast.parse 三轮全过
- 本地行为验证 16 项全过（purify 解析 8 项 + 换 PRO 决策 5 项 + 边界 3 项）

**未做**：配置补档（`secrets.json` 加 `reply_complex_model_name=deepseek-v4-pro`）——上服务器前一起做。

## 6.7 阶段 3 实施记录（2026-11-02）

**已改代码（本地，未上服务器）**：

| 文件 | 改动 |
|---|---|
| `schema/migrations/004_embedding.sql` | `memory_summaries` 加 `embedding BLOB` 列 + 建 `embedding_cursors` 表（4 游标：mem_incr/mem_backfill/diary_incr/diary_backfill） |
| `schema/migrations/004_embedding.down.sql` | 删游标表 + 版本号回退；embedding 列保留（空 BLOB 零副作用） |
| `dsh/llm/embedding.py` | embedding 客户端（已前置写好）：`embed_texts`/`vec_to_blob`/`blob_to_vec`，stdlib urllib，未配置静默 None |
| `dsh/services/core/app.py` | 新增 `/v1/embed` 端点（读 embedding_cfg → embed_texts → 返回 vectors） |
| `dsh/services/planner/core_client.py` | 新增 `embed(texts)`，条数校验 |
| `dsh/services/planner/embedding_worker.py` | **新**：巡查员，双表（记忆+日记）扫描 + 历史回填，游标持久化，stdlib struct 转 BLOB |
| `dsh/services/planner/repositories.py` | 新增 embedding 仓储组（游标读写、增量/回填查询、回填写列、`_table_exists` 保护） |
| `dsh/services/planner/supervisor.py` | 全局单例 embedding_worker 生命周期（start/stop） |
| `dsh/services/planner/main.py` | 组装 EmbeddingWorker 注入 supervisor |
| `dsh/storage/roles.py` | 矩阵加 `embedding_cursors` / `persona_diary`（planner 写方） |
| `dsh/services/planner/__init__.py` | 导出 EmbeddingWorker |

**架构铁则落地**：
- planner **不 import dsh.llm**（边界检查 0 违规）——向量经 core HTTP `/v1/embed` 拿
- 召回零变化：`fetch_memories` SQL 与排序**逐字节未动**（`_MEM_COLS` 不含 embedding 列）
- 日记库 `persona_diary` 由 SPACE_DIARY_DESIGN 定义，巡查员用 `_table_exists` 保护（表未建前安全跳过）

**验证**（独立数据目录，未碰生产）：
- ast.parse 三轮全过（9 文件）
- `tools/check_boundaries.py dsh` → **0 违规**（78 文件）
- 本地验证脚本 `scripts/_verify_embedding.py` 全过：写记忆 → 巡查员扫到 → mock 向量回填 → 游标前进 → 向量还原 → core 未配置安全 → fetch_memories 不变
- 回归：`test_vision_worker` / `test_planner_package` / `test_boundaries` 14 项全过

**未做**：
- 配置补档（`secrets.json` 加 embedding 子对象：endpoint+key+model）——上服务器前一起做
- 日记库 `persona_diary` 建表（属 SPACE_DIARY_DESIGN，与空间功能一起迁移）
- 语义召回（阶段 4 记忆衰减一起上）

## 6.8 golden 债清理记录（2026-11-02）

**背景**：阶段 3 收尾时全量测试 13 个 golden 测试失败。审计确认**全部是阶段 0/1 有意变更、测试预期滞后，无一真 bug**。

| 失败测试 | 根因 | 性质 |
|---|---|---|
| `test_core_prompt_golden`（9个） | system prompt 加第10条 COMPLEXITY + 私聊「关于内部机制」分支 | ✅ 阶段1预期 |
| `test_core_think_golden`（1个） | 同上 + actions 多 `complexity` 字段 | ✅ 阶段1预期 |
| `test_core_build_golden`（1个） | system prompt 加 COMPLEXITY 段 | ✅ 阶段1预期 |
| `test_core_purify_golden`（1个） | actions 多 `complexity: None` | ✅ 阶段1预期 |
| `test_core_rewrite`（1个） | `_render_config` 脱敏（2026-11-01 拍板「角色不掌握配置真实数值」） | ✅ 已拍板 |

**处理方式**：更新测试预期，登记「允许差异」，保留其余逐字比对（非粗暴改绿）。

- 测试文件：`test_core_prompt_golden.py` / `test_core_think_golden.py` / `test_core_build_golden.py`（`_strip_complexity` 剥离 COMPLEXITY 段 + 补齐换行）/ `test_core_purify_golden.py`（actions 补 complexity）/ `test_core_rewrite.py`（改验证脱敏行为）
- **结果：全量 249 测试全绿（OK）**

> ⚠️ 注意：`tools/make_golden.py` 是另一套外置 golden（生产库→jsonl），与这 13 个内嵌逐字比对测试无关，此处未用。

---

## 7. 待拍板

| # | 问题 | 我的建议 | 状态 |
|---|---|---|---|
| 1 | ~~COMPLEXITY 只在热聊还是所有~~ | ~~只在热聊~~ → **一直开**（决策 #15） | ✅ 已定 |
| 2 | ~~复杂场景客观信号~~ | ~~四个信号~~ → **flash 自己判**（§3.6，prompt 锚定） | ✅ 已定 |
| 3 | ~~PRO 重写带不带 flash 回复~~ | ~~不带~~ → **不带正文，但重出全部旁路数据**（决策 #17） | ✅ 已定 |
| 4 | ~~embedding 的 API key 来源~~ | ~~我提供~~ → **已提供**（2026-11-01，硅基流动） | ✅ 已提供 |
| 5 | ~~联想本期/下期~~ | ~~下期~~ → **本期做**（决策 #13） | ✅ 已定 |
| 6 | ~~`aliases=[]` 导致私聊称呼识别失效~~ | ~~待定~~ → **明确要修** | ✅ 已定 |
| 7 | ~~备用中转密钥 401~~ | ~~修~~ → **不修**（官方优先，中转不稳问题不大） | ✅ 已定 |

---

## 8. 顺带发现的真问题

| 问题 | 证据 | 影响 | 状态 |
|---|---|---|---|
| 私聊称呼识别失效 | `pipeline.py:226` `aliases = []`，`mention_probability` 拿到空列表永远返回 -1 | 她认不出我叫她的名字，私聊直接走随机概率 | ✅ **已修（2026-11-02）** |
| 备用中转密钥 401 | 中转服务返回 `INVALID_API_KEY` | fallback 通道失效 | 不修（决策 #7） |
| summary 高频重跑 | 配置 `schedule_hours=[13,24]` 但今日已跑 77 次 | 烧 token，待查 | 未修 |
| `/v1/summary` 偶发 500 | JSON decode 错误 | 不阻塞聊天，待修 | 未修 |

**aliases 修复（2026-11-02）**：
- `repositories.py` 加 `fetch_bot_aliases(store, bot_uin)`（读 bot_aliases 表，bot_uin 隔离，use_count DESC，对齐 v2 `_sync_load_bot_aliases`）
- `pipeline.py` `_process_locked` 私聊前读「官方名（account.display_name）+ DB 别名」传入 `_evaluate_trigger(batch, aliases)`
- 验证：`scripts/_verify_aliases_fix.py`（bot_uin 隔离 / 命中官方名+别名 / 空 aliases 安全兜底）全过；全量 249 测试仍绿

---

## 9. 空间+日记（第③步）进度记录

**2026-11-02 设计定稿 v5（SPACE_DIARY_DESIGN.md v5）**，关键演进：

1. **空间 = 账号级展示平台**（树洞，发了就过，无连贯性），**不是卡的能力开关**。
2. **空间 = 流**：复用 `chat_stream_settings`，`(bot_uin, 'qzone', 'qzone') → persona_id` 归某张卡用；**没绑卡 = 空间关闭**。
3. **不同账号空间分开**：角色A号一个、角色C号一个，各归不同卡（角色C号→虎鲸；角色A号→角色B或角色A未定）。
4. **订阅模型**：碎片打分属于订阅方侧，不是群聊侧。群聊侧只产出碎片；「谁要看」由 `stream_subscription`（订阅方 × 源）决定；订阅方含私聊会话（target_id）和空间流（'qzone'）。
5. **术语**：订阅（subscription）≠ 窥屏（peek）。
6. **多订阅方并发**：同一份碎片并发多个订阅方判断，各用各的卡人设 + 对各自我的了解，结果自然分叉。
7. **我不默认订阅所有源**；配置入口保底前端，chatops 放权租户留作后续。
8. **群聊线完全不动**：碎片 importance 打分、≥7 升级 summary 照旧在群聊侧完成。
9. **公开审查任何源都过**（群+私聊），本地日记不受影响。

**已完成（数据层 + 订阅投递业务层 + qzone 驱动 + 定时日记）**：
- `schema/migrations/005_qzone_diary.sql` + `.down.sql`：建 `persona_diary` + 建 `stream_subscription`（空间不新增列/表，复用 `chat_stream_settings`），up/down 成对，验证脚本 `scripts/_verify_migration_005.py` 全过
- `dsh/storage/roles.py` 注册 `stream_subscription`（webui 写，chatops 放权后续再加 planner）
- **回滚了 v4 的 `personas.qzone_enabled` 错设计**（空间开关不在卡上）
- **订阅投递（§7 步骤 2+3+4）**：`_gossip_impulse` 重构为 `_gossip_dispatch`（查订阅表→并发投递）+ `_subscriber_impulse`（单订阅方判断：私聊→push+记忆库 / 空间→写日记）；`repositories.py` 加 `fetch_subscribers` + `write_diary`；`_impulse_prompt` 支持「私聊/空间」双出口
- **qzone 驱动（步骤 5，B 路线）**：`_once_qzone` 扫空间绑卡→未推日记→公开审查（客观视角判公序良俗/隐私）→和谐→落 `public_text`+`synced_at`+`harmonized`；审查「有问题且和谐后空」→不发，本地日记保留
- **定时日记（步骤 6）**：`_once_diary` 每天 23:00（可配）每卡写一篇 `scheduled` 日记，数据源=当天碎片骨架（控成本），UNIQUE(persona_id,date,source_kind) 兜底一天一篇
- 验证：`scripts/_verify_subscription_dispatch.py` + `scripts/_verify_qzone_diary.py` 全过；全量 249 测试仍绿；ast 3 轮 + 边界 0 违规

**遗留 TODO（发说说出站，非设计问题）**：
- gateway `outbox.py` 尚无 `qzone` 分支（只有 group/private），消息协议端 `send_qzone_msg` 端点未迁 v3；`_review_and_sync_diary` 里发说说处留 TODO。需去服务器/v2 确认端点后补。

---

*本文档由 2026-11-01 对话产生。框架已敲定，细节待 §7 拍板后实施。*

---

> **关于平台**：本文脱胎于一个 QQ 机器人项目。文中出现的「消息网关 / 消息协议端」是泛化写法，
> 指代 IM 平台的接入层。类似的设计在 Telegram、Discord 等平台上同样适用 ——
> 使用场景是一致的，不同的只是接入层的具体实现。
