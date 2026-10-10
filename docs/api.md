# API 调用与数据契约

依据：[AGENTS.md](../AGENTS.md)、[产品需求与验收标准](requirements.md)。核验日期：2026-10-10。官方 CLI 核验版本：`command-code@1.79.2`。

## 1. 接入结论

公开资料能够确定本地会话、用量事件和 API Key 的使用方式；官方 npm 发布包能够确定账户、余额、订阅和账期汇总的读取接口；Token Monitor 源码补充了使用网页 Cookie 的 `/internal/billing/*` 接入方式。**现有证据不足以通过公开、稳定的 HTTP API 完整实现全部需求。**

| 数据能力 | 已确认的入口 | 可实现范围 | 缺口 |
| --- | --- | --- | --- |
| 登录与身份 | Studio API Key；`GET /alpha/whoami` | 验证密钥，识别用户与组织 | 第三方浏览器登录协议未公开 |
| 套餐 | `/alpha/billing/subscriptions`；网页 `/internal/billing/subscriptions` | 套餐 ID、订阅状态、当前账期 | 显示名称须维护版本映射；下次付款日期未确认 |
| 5h、周额度 | `/alpha/billing/credits`；网页 `/internal/billing/credits` | 已用、上限、重置时间；第三方仓库有标注为 Go 实际抓取的样本 | 真实账户响应与不同套餐仍需联调 |
| 月额度 | 余额与订阅接口 | 月度剩余额度、账期起止；Token Monitor 用静态套餐表补分母 | 缺少经确认的服务端月额度上限或已用比例，静态表不能替代账户事实 |
| 本地日、周、月统计及排行 | 会话 JSONL；Mods 用量事件 | 本地已记录请求的时间、模型、输入、缓存读取、输出 | 本地记录不等于账户全量记录；账户归属、复制去重需验证 |
| 账户全量历史明细 | Studio Usage 页面具备逐请求展示 | 确认产品存在此能力 | 明细路径、鉴权、分页、时间过滤、软件来源与缓存字段未确认 |

### 是否需要逆向 Command Code

1. 接入官方会话文件和 Mods 事件，不需要反编译 CLI；字段细节可用固定版本的发布包辅助核验。
2. 账户额度有两条未文档化路径：CLI 的 API Key＋`/alpha/*`，Token Monitor 的网页 Cookie＋`/internal/billing/*`。前者已通过官方发布包静态分析确认，后者可直接参考开源实现，不需要再为定位这两个网页接口逆向 CLI；两者均需真实账户联调。
3. Token Monitor 没有提供账户全量明细、服务端月额度上限或独立付款日期接口。完整覆盖这些字段仍需核验本人已登录的 Studio 请求或取得官方说明，不能由其月额度进度条反推这些 API 已存在。
4. 不把本地统计自动替换成账户全量统计，不把账期汇总改名为本日、本周、本月。需求及验收标准保持不变；缺少数据时按 D04、E02、E03 显示对应状态。

## 2. 证据等级与来源

| 等级 | 含义 | 开发使用规则 |
| --- | --- | --- |
| P | 官方公开文档明确说明 | 可据此设计适配器；响应仍需校验 |
| S | 固定版本官方发布包静态确认 | 只在版本化适配器中使用，联调通过前不标记已验证 |
| R | 固定提交的第三方源码、测试或作者标注的抓取样本 | 可作为接入依据；区分源码事实、模拟样本与作者报告，不提升为本项目 V 级证据 |
| V | 使用真实账户或会话样本验证 | 必须保存脱敏样本、版本、验证日期和结果 |
| U | 未确认 | 不猜测路径、字段、单位或返回结构 |

目前没有 V 级证据：环境未配置 Command Code 凭据，也没有真实会话样本。下列结构是客户端消费契约，不是完整的服务端 OpenAPI Schema。

| 来源 | 用途 |
| --- | --- |
| [官方 Provider API](https://commandcode.ai/docs/provider) | 模型调用入口与鉴权；未提供账户历史查询契约 |
| [官方 Studio 文档](https://commandcode.ai/docs/studio) | API Key、组织、逐请求用量页面 |
| [官方会话文档](https://commandcode.ai/docs/sessions) | JSONL 位置、分支、导出与会话持久化 |
| [官方 Mods 文档](https://commandcode.ai/docs/mods) | `cmd.on` 与 `model_request_end` 等事件 |
| [官方 CLI Reference](https://commandcode.ai/docs/reference/cli) | CLI 子命令与参数 |
| [官方 Usage Limits](https://commandcode.ai/docs/resources/usage-limits) | 额度窗口与自然统计周期的区别 |
| [官方 npm 固定版本元数据](https://registry.npmjs.org/command-code/1.79.2) | 发布版本、包完整性 |
| [官方 npm 发布包](https://registry.npmjs.org/command-code/-/command-code-1.79.2.tgz) | 本文 S 级调用点与字段 |
| [Token Monitor 额度实现](https://github.com/Javis603/token-monitor/blob/3e98bff8c0cbc1d81e9eaa86bbe9e64b53226f52/src/shared/providers/commandcode/limits.js) | 网页 Cookie、两个 internal 接口、窗口字段和月额度推算 |
| [Token Monitor 账户入口](https://github.com/Javis603/token-monitor/blob/3e98bff8c0cbc1d81e9eaa86bbe9e64b53226f52/src/shared/providers/commandcode/account.js) | 用户打开 Usage 页面并提供 Cookie 的流程 |
| [Token Monitor 测试与样本](https://github.com/Javis603/token-monitor/blob/3e98bff8c0cbc1d81e9eaa86bbe9e64b53226f52/tests/shared/commandcodeLimits.test.js) | 第 491 行起为作者标注的 Go 抓取样本；其余包含合成和修改后的 fixture |

Token Monitor 核验版本为 `0.68.0`，提交 `3e98bff8c0cbc1d81e9eaa86bbe9e64b53226f52`。执行 `node --test tests/shared/commandcodeLimits.test.js`，22 项通过；这些测试注入模拟 fetch，没有请求真实账户。

发布包的 SHA-512 与 npm `dist.integrity` 一致。`dist/cli.mjs` 的 SHA-256：

```text
d976c2ad89f93a2b81af4d150dca99aeedfd679f15e8b8c032314e61e484da7b
```

静态证据定位使用符号名，避免压缩文件行号失效：

| 符号／文件 | 确认内容 |
| --- | --- |
| `fetchUsageData`、`fetchUsageWhoami`、`fetchUsageCredits`、`fetchUsageSubscription`、`fetchUsageSummary` | 四个 GET 接口、`limits`、`orgId`、`since` 与调用顺序 |
| `validateCommandApiKey` | Bearer 鉴权和身份校验 |
| `projectUsageView`、`WindowLimitMeter` | 余额、订阅、窗口字段；窗口重置值按毫秒参与运算 |
| `isSessionHeaderV3`、`isSessionEntryV3`、`appendMessage`、`toSessionUsage` | v3 会话头、消息条目、用量所在层级 |
| `estimateSessionCostUsd`、`consumeFullStream` | `inputTokens` 包含缓存读取／写入，二者另有细分 |
| `serializeAgentEventLine`、`createPrintJsonEventStream` | headless 事件封装与运行累计结果 |
| `dist/bundled/mod-builder/reference/hooks-and-events.md` | 公开事件与 hook 顺序 |
| `CHANGELOG.md` 的 `0.38.7` | 已移除 `/usage` 和用量 API 的按模型细分；旧示例不能当作当前契约 |

## 3. 调用边界

- UI 只调用本项目应用服务；上游 HTTP、凭据、本地文件解析均由原生后端适配器处理。
- 上游集中在 `CommandCodeAccountAdapter`；会话读取集中在 `CommandCodeSessionAdapter`。名称是项目规划，不表示已有代码或指定框架。
- 上游基础地址固定为 `https://api.commandcode.ai`。生产请求只允许 HTTPS，不把凭据发送给重定向目标。
- `/alpha/*`、网页 `/internal/billing/*` 与 `/provider/v1/*` 是不同 API。前两者分别使用 API Key、网页会话 Cookie，不混用凭据；用量客户端不通过生成请求验证登录。
- 每个请求记录接口标识、耗时、HTTP 状态和适配器版本；不记录密钥、认证头、会话正文或完整响应。
- 未知字段允许存在；已使用字段逐项校验。必要字段缺失、类型变化、负 Token、非有限数值返回 `SCHEMA_MISMATCH`，不得默认补零。
- HTTP 接口只读取账户数据；套餐购买、续费、充值、取消订阅不属于本产品 API。

## 4. 账户与额度 HTTP 契约

### 4.1 API Key 路径的请求与顺序

```http
GET /alpha/whoami?limits=1 HTTP/1.1
Host: api.commandcode.ai
Authorization: Bearer <COMMAND_CODE_API_KEY>
Accept: application/json
```

GET 不发送请求体。查询参数用 URL 编码；空值参数省略，不发送字符串 `null` 或 `undefined`。

1. 先读取身份，取 `org.id`，固定这次刷新使用的用户／组织上下文。
2. 身份成功后，并行读取 credits、subscriptions；个人上下文省略 `orgId`。
3. 若需诊断账期汇总，再取订阅的 `data.currentPeriodStart` 作为 `since` 调用 summary。
4. 身份读取失败时不按个人账户继续请求。不得把未知组织上下文解释为“没有组织”。
5. 账户切换、退出登录使所有旧请求失效。旧账户的迟到响应不得覆盖新账户数据。

### 4.2 API Key 路径的接口清单

| 接口标识 | 方法与路径 | 查询参数 | 已观察到的 JSON 路径 | 等级 |
| --- | --- | --- | --- | --- |
| CC01 | `GET /alpha/whoami` | `limits=1`，CLI 用量页使用 | `user.id`、`user.userName`、`user.name`、`org.id`、`org.login`、`orgLimits` | S |
| CC02 | `GET /alpha/billing/credits` | `orgId`：CC01 的 `org.id`，无组织时省略 | `credits.planId`、`credits.monthlyCredits`、`credits.purchasedCredits`、`credits.freeCredits`、`windowLimits` | S |
| CC03 | `GET /alpha/billing/subscriptions` | 同上 | `data.planId`、`data.status`、`data.currentPeriodStart`、`data.currentPeriodEnd` | S |
| CC04 | `GET /alpha/usage/summary` | `orgId`；`since`：CC03 的账期开始值 | `totalCost`；其他汇总字段未建立契约 | S |

CC01、CC02、CC04 使用根对象；CC03 的订阅在 `data` 内。不要统一添加或删除一层 `data`。

#### CC01：身份

- 身份和组织 ID 按不透明字符串处理；组织 ID 必须来自当前密钥可访问的身份响应。
- `user.userName`／`user.name` 用于显示，不作为持久化主键。组织名称与个人名称不能混为同一账户标识。
- `orgLimits` 是额外的组织限额信息，不能替代套餐的 5h、周、月限制。
- 登录成功需要 HTTP 成功及可识别的有效身份；仅看到 200 或本地存在密钥不算成功。

#### CC02：余额与窗口

| JSON 路径 | 含义与映射 | 校验 |
| --- | --- | --- |
| `credits.monthlyCredits` | 当前月度剩余额度，**不是月度总上限** | 有限非负数；单位为上游 credit |
| `credits.purchasedCredits` | 购买的剩余额度 | 同上；不加入月额度分母 |
| `credits.freeCredits` | 免费剩余额度 | 同上；不加入月额度分母 |
| `windowLimits.limited` | 上游窗口适用标志 | boolean；缺失不解释成 false |
| `windowLimits.fiveHour.used`、`.cap` | 5h 已用与上限 | 有限非负数，单位一致 |
| `windowLimits.weekly.used`、`.cap` | 周已用与上限 | 同上 |
| 两个窗口的 `.resetAt` | 重置时间 | CLI 按 Unix 毫秒与 `Date.now()` 比较；联调核实后转 UTC ISO 8601 |

字段缺失与明确不适用分别保留状态。`cap <= 0` 不做除法；仅上游明确表达无此限制时标记不适用。重置时间已过不在本地加 5 小时或 7 天，触发重新读取。

上游提供有效 `used` 和正数 `cap` 时：`usedPercent = used / cap × 100`，文本保留一位小数，条形比例限制在 0–100%。文本不截断超过 100% 的真实数值。

#### CC03：订阅

- 保留 `planId` 和原始 `status`；未知状态不能当作 active。
- 显示名称优先使用经核实的服务端名称字段。当前仅确认 `planId`，可用固定版本精确映射；未知 ID 显示原 ID，不推断套餐。
- `currentPeriodStart`、`currentPeriodEnd` 是账期起止。CLI 传给日期解析器，但生产适配器须经样本固定格式；不按数值大小猜秒／毫秒。
- **`currentPeriodEnd` 不等于已确认的下次付款日期。** 试用、取消、延期、欠费和变更套餐均不能靠账期结束日推算付款。
- `nextPaymentAt` 没有确认的上游字段，显示“数据暂不可用”；只有来源明确表示没有下一次付款安排时才显示“无下次付款”。
- `data: null` 只有经真实样本确认其含义后才映射为无订阅；空对象／缺失 `data` 不能按无订阅处理。

#### CC04：账期汇总

- 当前 CLI 使用订阅开始值发送 `since`；这只证明客户端发送该参数，未证明服务端按任意时间精确过滤。
- 没有确认 `until`、分页、时区和按软件／模型分组参数。不得自行添加参数并假定生效。
- `totalCost` 只作为独立账期诊断数据；不能由费用反算 Token 或缓存命中率。
- 不将 summary 与逐请求记录相加，不作为顶部自然周期汇总和排行的数据源。

### 4.3 月限制与日期处理

月限制必须来自经确认的账户 `used + cap`、`remaining + cap` 或直接比例，以及对应月限制周期。字段名在验证前保持 U 级，不预设 `/alpha/billing/limits` 等路径。

CLI 的展示逻辑结合本地套餐表、余额和费用构造进度；本项目不复用此推算：余额可能含充值，费用也不一定等于额度消耗。不得从官网价格表硬编码账户月限额。

Token Monitor 同样没有从接口获取月度上限，而是用 `planId` 查询静态表，再用 5h／周 cap 和剩余额度做一致性检查。该方法可显示估算进度，不能满足本项目 E03 的真实账户限制口径，具体计算见 4.5。

同理，月重置时间只有确认与月额度周期绑定后才使用账期结束日。缺少限额或重置数据按 E03 显示“未提供”，读取失败则显示“读取失败”。

### 4.4 登录约定

- E01 的应用登录采用 API Key 校验：用户在官方 Studio 获取密钥，应用后端调用 CC01，成功后保存到系统凭据存储。
- API Key 获取页由官方 Studio 文档指向的页面进入；前端不接收持久化密钥，不把密钥写入普通配置或日志。
- 如接入已安装 CLI 的凭据，凭据来源必须显式选择。官方文档说明 `COMMAND_CODE_API_KEY` 优先于 `~/.commandcode/auth.json`；适配器应按所选来源校验身份。
- CLI 的 `cmd login`／`cmd status` 不是独立第三方 OAuth 协议。不得编造 OAuth client ID、回调、scope 或刷新接口。
- 取消登录返回 `cancelled`，失败返回可重试错误；退出只移除本应用凭据和对应缓存，不调用全局 CLI logout。

### 4.5 Token Monitor 网页额度路径（R 级）

#### 请求与鉴权

| 接口标识 | 请求 | 输入 | 响应消费位置 |
| --- | --- | --- | --- |
| CC05 | `GET https://api.commandcode.ai/internal/billing/credits` | 网页 Cookie；源码不发送查询参数或请求体 | `credits`；根部或 `credits` 内的 `windowLimits` |
| CC06 | `GET https://api.commandcode.ai/internal/billing/subscriptions` | 同上 | `success`、`data` |

对应函数为 `fetchCommandcodeLimits`。两个请求并行，credits 默认超时 12 秒，subscriptions 默认 6 秒且不超过 credits 预算。订阅读取失败仍返回有效的余额及 5h／周窗口；credits 快速失败时取消订阅读取。这是参考仓库的策略，项目默认预算仍按第 7 节执行。

源码发送 `Cookie`、`Accept: application/json, text/plain, */*`、`Accept-Language: en-US,en;q=0.9`、浏览器 User-Agent、`Origin: https://commandcode.ai`、`Referer: https://commandcode.ai/`。这些是已观察请求头，不表示每个头都已证实为服务端必需。

- Cookie 从用户在已登录 Usage 页的 Network 请求中复制。配置优先于 `COMMANDCODE_COOKIE`，再取 `TOKEN_MONITOR_COMMANDCODE_COOKIE`；这些变量是参考仓库约定，不是官方 API Key 环境变量。
- 必须包含 `commandcode_prod_.session_token`，或其 `__Secure-`／`__Host-` 前缀形式；原样保留实际名称。可以附带同系列 `session_data`，只转发这六种 Cookie 名称，其余过滤。
- 支持解析用户提供的“Copy as cURL”文本，但不执行该命令。参考代码验证来源主机为 `commandcode.ai`、`www.commandcode.ai` 或 `api.commandcode.ai`；本项目还要求来源 HTTPS。
- 此模块没有自动网页登录或会话续期协议，不能据此承诺“一键浏览器登录”。Cookie 到期需重新取得；不尝试改用 API Key 调用 internal 路径。
- 源码不传 `orgId`，公开抓取样本为个人 Go 账户。组织切换、作用域及组织请求参数保持 U 级；不拼接猜测的组织参数。

#### 字段与时间

| 字段 | Token Monitor 的行为 | 本项目采用规则 |
| --- | --- | --- |
| `credits.monthlyCredits` | 作为月度剩余额度 | 同一含义；不是月度 cap |
| `credits.purchasedCredits` | 单独显示 Top-up，无进度条 | 保留原始额度，不加入月限额分母；不新增需求外 UI |
| `premiumMonthlyCredits`、`opensourceMonthlyCredits` | 样本中是剩余额度的拆分 | 不作为 cap，不与 monthlyCredits 再相加 |
| `credits.windowLimits`／根部 `windowLimits` | 优先嵌套值，回退根部值 | 为 internal 适配器显式支持两个形状；两处冲突时报契约异常，不静默覆盖 |
| `fiveHour`／`weekly` 下的 `used`、`cap`、`resetAt` | 直接读取，计算 used/cap | 校验后保留真实百分比；仅条形封顶 |
| `resetAt: 0` | 转为 null | 作者 Go 样本中未启动窗口如此表示；显示“未提供”，不显示 1970 年，也不触发无休止刷新 |
| `success: true, data: null` | 解释为无订阅 | 已有 R 级解析行为；本项目真实无订阅样本验证后启用该映射 |
| `data.planId`、`status` | 套餐表匹配与订阅状态 | 保留原 ID 和状态；源码取得 status，但月分母算法未据此筛选 active |
| `data.userId`、`metadata.commandCodeUserId` | 稳定账户标识的优先来源 | 与独立身份核验一致后才能关联本项目账户 |
| `data.currentPeriodStart`、`currentPeriodEnd` | 样本为 ISO 字符串，结束值用于月重置 | 保留账期语义；不能改名为付款日期 |
| `cancelAtPeriodEnd`、`cancelAt`、`canceledAt`、`endedAt`、`pendingPhase` | 作者样本有字段，额度模块未用于付款推导 | 记录为验证候选，不推导下次付款 |

参考解析器还容忍 snake_case、数值字符串，并按数值大小猜秒／毫秒。这些容错分支不等于存在对应真实响应；本项目按验证后的字段版本确定类型和单位，不照搬猜测。作者标注的 Go 样本使用根部 windowLimits、ISO 账期日期及 `resetAt: 0`；非零毫秒和嵌套窗口等情况另有模拟测试。

#### 月额度算法与边界

`COMMANDCODE_PLANS` 的固定值如下，仅用于说明参考实现，不能作为本项目账户限额：

| planId | 显示名称 | 月度分母 | 5h 校验 cap | 周校验 cap |
| --- | --- | --- | --- | --- |
| `individual-go` | Go | 10 | 3 | 6 |
| `individual-goat` | GOAT | 70 | 14 | 35 |
| `individual-pro` | Pro | 80 | 16 | 40 |
| `individual-max` | Max 10x | 150 | 45 | 90 |
| `individual-ultra` | Max 20x | 300 | 90 | 180 |

`trustedMonthlyAllowance` 要求：套餐已知、剩余额度不超过静态月分母、接口同时提供两个窗口 cap，且 cap 与表中值完全相等。通过后 `billingWindows` 计算 `used = clamp(月分母 - 月剩余, 0, 月分母)`，通用层再算百分比；未通过则仅显示剩余额度并隐藏月进度条。重置时间取 `currentPeriodEnd`，充值余额单独显示。

例如参考测试中的 Go 剩余为 8.7784，静态月分母为 10，得到已用 1.2216，即 12.216%。只有 8.7784 和窗口 cap 来自响应，10 来自代码表。源码把额度标为 USD，但注释同时说明部分套餐使用 usage-value credit；本项目不默认把每个 credit 当作 1 美元。

**采用结论：** 可借鉴网页读取路径、Cookie 最小转发、两个窗口映射及订阅失败不拖垮余额的处理；月度分母仍需账户事实。参考模块把缺失 `used` 设为 0、把百分比钳制到 100%，不符合本项目 D04、E03，不能原样采用。该模块也没有独立付款日期或逐请求历史接口。

这两个入口纳入 `CommandCodeWebAccountAdapter` 的候选契约，保留与 API Key 适配器的凭据／缓存隔离。订阅 ID 或 Cookie 哈希只能作为临时来源身份，不能当作稳定用户 ID；无法验证身份时不得将 Cookie 数据与 API Key 账户合并。完成 API10 联调后，才把网页凭据接入第 6 节应用服务；现有 API Key 登录契约不变。

## 5. 用量明细来源

### 5.1 本地会话 JSONL

官方路径：`~/.commandcode/projects/<project-slug>/<session-id>.jsonl`。`~` 使用当前用户主目录；Windows 使用对应用户目录，不拼接固定用户名。`/session-file` 可确认当前文件，`/export` 可导出 JSONL。[官方会话文档](https://commandcode.ai/docs/sessions)

1.79.2 的 v3 最小消费结构（合成示例，不是真实账户响应）：

```json
{"type":"session","version":3,"id":"example-session","timestamp":"2026-10-10T00:00:00Z","cwd":"/example"}
{"type":"message","id":"example-entry","parentId":null,"timestamp":"2026-10-10T01:00:00Z","message":{"role":"assistant","content":[]},"model":"example-model","usage":{"inputTokens":400,"cacheReadTokens":300,"cacheWriteTokens":20,"outputTokens":50}}
```

| 源字段 | 规范化字段／规则 |
| --- | --- |
| 头部 `version` | 仅 v3 启用本适配器；旧版本单独迁移适配，不猜字段 |
| 条目 `type`、`message.role` | 只消费 `message` 且角色为 `assistant`、有合法 usage 的条目 |
| 条目 `timestamp` | `occurredAt` 候选；是落盘时间，非已验证的服务端计费时间，边界准确性需验证 |
| 条目 `model` | `modelId`；缺失时按同分支已确认的模型变更记录恢复，不能套用会话最终模型 |
| 条目 `usage.inputTokens` | 包含缓存读取和缓存写入的输入总量 |
| 条目 `usage.cacheReadTokens` | 命中缓存的输入量 |
| 条目 `usage.cacheWriteTokens` | 缓存写入细分；属于未命中输入，不能再次加到总量 |
| 条目 `usage.outputTokens` | 输出量；不重复追加推理细分 Token |
| `id`、`message.meta.messageId`、会话 lineage | 建立去重与复制来源关系；不能只用文件名＋行号 |

归一化公式：

```text
uncachedInput = inputTokens - cacheReadTokens
cachedInput   = cacheReadTokens
output        = outputTokens
total         = uncachedInput + cachedInput + output
hitRate       = cachedInput / (uncachedInput + cachedInput) × 100
```

示例结果：未命中输入 100，缓存输入 300，输出 50，总量 450，命中率 75.0%。缓存写入 20 已经包含在未命中输入 100 内。`cacheReadTokens > inputTokens` 视为字段异常，不能钳制后继续统计。

文件读取规则：

- 流式按行读取，只提取统计元数据，不缓存会话正文。忽略 `.meta.json`、`.checkpoints.jsonl`、`.prompts.jsonl`、备份及导出副本作为独立请求来源。
- 未完成的末行留待下次读取；中间损坏行记录诊断并将覆盖状态设为 partial，不能静默跳过后宣称数据完整。
- 按文件标识和字节偏移增量读取；文件替换或缩短后重建该文件索引，并由稳定记录身份去重。
- fork／clone 可能复制已有用量。优先使用已验证的请求／消息 ID，加来源会话 lineage 识别继承条目。普通 entry ID 只在文件内可靠；不得跨不相关会话仅凭相同 ID 或内容哈希去重。
- 已发生的请求即使退出活动分支仍是用量；不能只统计当前叶分支，也不能把恢复／复制历史当新调用。
- 不同真实调用即使模型、时间、Token 相同也必须保留。缺少可验证去重身份时将 coverage 设为 partial，不输出“完整统计”。
- 本地文件不保证有可靠账户标识；不能把当前登录用户追溯绑定到所有旧文件。归属不明的记录只标记为本地来源，不混入账户统计。
- 被删除、`--no-session`、其他设备、Cloud 和外部 Provider 调用可能没有本地文件。本地数据源固定标记 `scope=local`，不能标成 `account`。

### 5.2 Mods 与 headless 事件

官方扩展点为 `cmd.on('model_request_end', handler)`，事件含 `model`、`usage`、`stopReason`。headless JSON 流在当前版本将事件包装为 `{"type":"event","event":{...}}`，最终 `type=result` 中的 usage 为运行累计值。

- 收集器只取逐模型请求完成事件；不再累计 `turn_end`、`run_end`、最终 result 或 subagent 汇总，否则会重复。
- 事件与会话文件不直接相加；关联记录稳定 ID 后择一入库。没有关联依据时整段使用一个来源。
- 插件只能捕获加载后的活动，不能补齐历史。事件未确认有服务端时间／账户 ID 时，采集时间与来源范围必须显式记录。
- 子代理、自动压缩、失败但已计费请求是否完整上报列入契约验证。未验证前保持 partial。
- Mods 是候选补充来源，不作为用户默认必须安装的依赖；是否采用由对应模块规划确定。

### 5.3 账户历史明细

Studio 文档确认逐请求用量页面，但尚未确认第三方读取契约。该适配器必须先取得以下事实：

| 必需信息 | 用途 |
| --- | --- |
| 实际 URL、method、请求体／查询参数、响应封装 | 固定调用契约 |
| API Key 或网页登录鉴权及账户／组织作用域 | 正确账户隔离 |
| 请求唯一 ID、发生时间及其单位 | 去重与自然周期归属 |
| 模型、真实软件／客户端来源 | F01 的软件＋模型分组 |
| 输入、缓存读取、缓存写入、输出及包含关系 | D02 的三类 Token |
| 分页方式、排序、终止条件、最大范围、迟到数据规则 | 确认完整性与增量同步 |
| 时间过滤和时区语义 | 本日、本周、本月与跨时区边界 |

验证方式为在本人已登录 Studio 中查看正常 Usage、Billing 读取请求并保存脱敏契约；不枚举猜测路径。若需浏览器会话而不支持 API Key，单独设计会话生命周期，不能把网页 Cookie 当永久 API Key。

## 6. 项目内部应用服务契约

以下是本项目规划的 IPC／函数接口，**不是 Command Code 上游路径**。原生后端按此契约提供数据；不额外启动公网 HTTP 服务。

| 方法 | 输入 | 输出及规则 |
| --- | --- | --- |
| `connectAccount` | 用户提供的 `apiKey`，仅传给后端 | 校验后返回 `accountContext`；密钥不回传 |
| `getAccountSnapshot` | `accountContextId`、`forceRefresh=false` | 套餐、付款、5h／周／月额度，字段状态独立 |
| `getUsageSnapshot` | `period: day/week/month`、`timeZone`、`scope`、可选 `accountContextId` | 同一快照的汇总与完整分组排行 |
| `refresh` | `target: account/usage/all`、相应上下文 | 刷新操作 ID；可取消，同目标合并请求 |
| `disconnectAccount` | `accountContextId` | 清除应用凭据、账户缓存并使在途请求失效 |

### 6.1 通用状态与类型

```ts
type Field<T> =
  | { state: 'available'; value: T }
  | { state: 'not_provided' | 'not_applicable' | 'unavailable'; reason: string };

type Result<T> =
  | { state: 'ready'; data: T; fetchedAt: string; revision: string }
  | { state: 'loading' }
  | { state: 'error'; code: string; retryable: boolean }
  | { state: 'stale'; data: T; fetchedAt: string; revision: string; code: string };

type TokenCount = string; // 非负整数字符串，后端精确求和，避免 JS 安全整数溢出
type Decimal = string;    // credit 与比例的十进制值，禁止用浮点格式化值反算

type UsageRecord = {
  recordId: string;       // 适配器确认的稳定唯一身份
  accountContextId: string | null;
  source: 'local_session' | 'mod_event' | 'account_api';
  occurredAt: string;    // UTC ISO 8601，必须记录源时间语义
  softwareId: string;
  softwareName: string;
  modelId: string;
  modelName: string;
  uncachedInput: TokenCount;
  cachedInput: TokenCount;
  output: TokenCount;
};
```

无法完成必要字段映射的记录保留诊断，不能填零制造合法 `UsageRecord`。软件由来源适配器识别；只有已确认的 Command Code CLI 会话才能固定为 `CommandCode`，未知账户客户端来源不能一律改成 CommandCode。

### 6.2 返回对象

| 对象 | 必备内容 |
| --- | --- |
| `accountContext` | `id`、`userId`、`orgId`、`displayName`；由已验证身份建立，凭据另存 |
| `accountSnapshot` | `accountContextId`；`planId`、`planName`、`subscriptionStatus`、`nextPaymentAt` 均用 Field；`limits.fiveHour/weekly/monthly` |
| 每个 limit | 独立的 `used`、`cap`、`usedPercent`、`resetAt` Field，以及 `unit` 和 `applicability`；不因缺少日期丢掉有效百分比 |
| `usageSnapshot` | `queryKey`、`revision`、`scope`、`accountContextId`、`period`、`timeZone`、`startInclusive`、`endExclusive`、`asOf`、`coverage`、`totals`、`ranking` |
| `coverage` | `status: complete/partial/unknown`、`reason`；complete 必须限定在声明的 scope 和时间范围内 |
| `totals`／排行项 | 三类 Token、总量和 `hitRate`；排行另有软件／模型标识及显示名 |

`nextPaymentAt` 的 `not_applicable` 仅表示来源确认无付款安排；缺字段使用 unavailable。输入总量为零时 `hitRate` 使用 not_applicable，显示“—”。

### 6.3 统计与展示

- 以运行设备当前 IANA 时区计算自然周期，周一开始。分别构造本地周期起点和下个起点再转换 UTC，不用 24 小时、7×24 小时或 30 天代替日历运算。
- 记录过滤：`startInclusive <= occurredAt < endExclusive` 且 `occurredAt <= asOf`。缺时间的记录不能用导入时间补齐。
- 汇总与排行使用同一规范化、去重、过滤后的记录集和 revision。切换周期的迟到响应不能覆盖当前 queryKey。
- 先按 `softwareId + modelId` 分组，再按原始总 Token 降序；相同总量按软件名、模型名升序，再按稳定 ID 破同名平局。固定比较规则，不受本机默认 locale 漂移影响。
- 命中率用汇总后的缓存输入除以汇总输入，不平均各条百分比。排行三类 Token 分别求和必须等于顶部对应数量。
- K/M/B 采用 1,000 进位，十进制四舍五入保留两位；进位后再选单位。百分比保留一位。使用精确整数／十进制计算。
- 分段条使用原始三类 Token 的占比；缓存写入归未命中输入，零总量显示空轨道。
- 仅 complete 且成功读取无记录才显示真实零和空列表；partial 明示“数据不完整”，unknown 明示“数据不可用”。
- 周期边界和时区变化在 60 秒内触发本地重算；已载入 10,000 条数据时切换目标 ≤ 200 毫秒。统计切换不重取或改变额度周期。

## 7. 刷新、缓存与失败处理

| 项目 | 项目默认值／规则 |
| --- | --- |
| HTTP 超时 | 单次 10 秒，整次账户刷新 30 秒，重试计入总预算 |
| 自动重试 | 仅幂等 GET 的网络错误、408、429、5xx；至多 2 次，1 秒／2 秒退避并加抖动 |
| 429 | 优先遵守合法 `Retry-After`；超过预算则返回等待时间，不立即重试 |
| 并发 | 每账户最多 2 个上游读取；相同键在途请求合并 |
| 缓存键 | 适配器版本＋账户上下文＋接口＋规范化参数；用量另含 scope、周期边界和时区 |
| 账户刷新 | 登录后立即读取；可见时 60 秒，隐藏时 5 分钟；手动刷新可跳过 TTL，但不跳过退避 |
| 本地用量 | 文件变化后 500 毫秒防抖增量读取，30 秒低频核对；可用文件通知时不反复全量扫描 |
| 过期缓存 | 同账户旧值可保留为 stale 并标注更新时间，不冒充实时；退出／换账户立即隔离 |
| 数据提交 | 每次快照原子更新；部分接口失败保留字段级状态，不拼接未标识的不同刷新结果 |

这些值是本项目客户端策略，不代表官方公布的限流额度。

| 条件 | 错误码 | 处理 |
| --- | --- | --- |
| 无凭据 | `AUTH_REQUIRED` | 显示登录入口 |
| 401 | `AUTH_INVALID` | 不重试；提示重新登录 |
| 403 | `FORBIDDEN` | 保留身份，提示无访问权限；不能一律当作密钥失效 |
| 404／410 | `ENDPOINT_UNAVAILABLE` | 停用对应能力，检查协议版本 |
| 429 | `RATE_LIMITED` | 按退避重试并显示短状态 |
| 超时／离线 | `TIMEOUT`／`NETWORK_ERROR` | 保留带时间戳的旧值 |
| 5xx | `UPSTREAM_ERROR` | 有限重试 |
| 200 但非 JSON／已消费字段不匹配 | `SCHEMA_MISMATCH` | 不重试掩盖错误，不补默认值 |
| 来源不能提供功能 | `CAPABILITY_UNAVAILABLE` | 对应区域显示未提供或不可用 |
| 文件损坏／明细分页不完整 | `INCOMPLETE_DATA` | coverage=partial，不能显示完整总量 |

## 8. 实施与契约验收任务

| 编号 | 任务名 | 具体实现方式 | 完成状态 |
| --- | --- | --- | --- |
| API01 | 公开接口核验 | 核对官方文档与 1.79.2 发布包；记录四个 HTTP 入口、会话和事件字段及证据等级 | ✔ 完成 |
| API02 | 账户接口联调 | 用真实个人／组织账户验证 CC01–CC04 的请求、类型、时间单位、空订阅和失败响应，形成脱敏 fixture | 未开始 |
| API03 | 补齐付款与月额度契约 | 核验 Studio 正常 Billing 请求中的真实字段；验证取消、试用、充值、套餐变更，禁止推算 | 未开始 |
| API04 | 账户历史明细核验 | 确认正常 Usage 请求的鉴权、分页、模型／软件、缓存和时间字段；验证跨页去重和迟到记录 | 未开始 |
| API05 | 会话适配器 | 按 v3 解析、增量索引与 lineage 去重；验证缓存输入、fork／clone、损坏末行及账户归属 | 未开始 |
| API06 | 账户适配器与凭据 | 实现 CC01–CC03、字段校验、系统凭据存储、刷新与账户隔离；CC04 保持独立诊断用途 | 未开始 |
| API07 | 统一统计应用服务 | 按内部契约输出同 revision 汇总和排行；验证时区、精确计算与查询竞态 | 未开始 |
| API08 | 数据契约验收 | 执行下表并记录预期／实际；接口样本验证后才能把对应能力提升至 V | 未开始 |
| API09 | 网页额度参考核验 | 核对 Token Monitor 固定提交的 Cookie 流程、CC05／CC06、月额度静态表及作者样本；22 项模拟响应测试通过 | ✔ 完成 |
| API10 | 网页额度契约联调 | 验证 CC05／CC06 的真实个人／组织上下文、Cookie 到期、字段层级、resetAt=0、无订阅和额度单位；通过后确定网页凭据的应用服务输入 | 未开始 |

| 验收目标 | 必测输入与预期 |
| --- | --- |
| D02 | 输入总数 400、缓存读取 300、缓存写入 20、输出 50 → 未命中 100、总量 450、命中率 75.0% |
| D03 | 1,234 → 1.23K；13,215,000 → 13.22M；999,995 → 1.00M；94.25% → 94.3% |
| D01 | 午夜、周一、月初、跨年、闰日、夏令时和系统时区变化；边界不重不漏 |
| D04 | 空记录、零输入、缺缓存字段、坏 JSON、离线、旧缓存分别显示正确状态 |
| E02 | 账期结束但无付款字段 → 数据暂不可用；来源明确无付款 → 无下次付款 |
| E03 | used=110、cap=100 → 110.0%，条形 100%；缺 cap 或 cap=0 不除零、不冒充无限 |
| 网页额度 | 根部／嵌套窗口各自解析，冲突不覆盖；resetAt=0 显示未提供；缺 used 不补零；订阅失败不清除有效窗口；静态套餐表不冒充真实 cap |
| E04 | 切换统计范围不改变额度周期；到达重置时间重新读取而非本地推算 |
| F01 | 分组后三类数量与汇总分别相等；同量排序稳定；未知软件来源不改写 |
| 完整性 | 重复导入、fork／clone、多分支、分页失败、迟到数据、子代理事件不重复也不隐瞒缺口 |
| A03、C03 | 10,000 条记录下增量导入、内存及切换延迟满足需求；文件正文不常驻 |
| 身份隔离 | 登录取消可重试；401 停止重试；403 不抹除身份；账户切换后旧响应无效 |

## 9. 契约维护规则

- 新增接口功能先查本文的接口标识与映射，不在 UI 或业务模块临时查找、拼接上游接口。
- 已列明的接口直接按契约实现；只有 U 级缺口、协议不匹配或版本升级需要重新核验。
- 协议变更同时更新适配器、本文字段表及脱敏 fixture；固定 npm 版本与哈希，不以 latest 作为可复现基线。
- 真实样本只保存统计字段，删除密钥、用户信息、会话正文和可识别项目路径；合成 fixture 必须标明合成。
- 本文不替代产品验收。账户全量能力、付款日期和月限额未核实前，对应任务保持未开始或进行中。
