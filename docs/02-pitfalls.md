# 02 · 坑集（全部经断言/真跑实锤）

## 坑#1：Switch 值不匹配 → 整条链静默丢弃

**现象**：表单下拉框的值与分支名不完全一致（大小写、空格、德语变体、新增场次忘了加分支）时，该提交**不进任何分支、无输出、无报错**，这条报名无声消失。

**实锤**（A 关+B 关断言）：4 条投喂（含 1 条小写变体 `e-mail zustellung…`、1 条完全不在列表），Switch 严格匹配下只有 2 条出队，2 条无声消失，执行状态照样 success。

**修复**（custom 版已做）：`options.fallbackOutput = "extra"` 开出第三个「未匹配-兜底」输出，接日志/告警节点。

## 坑#2：occurrence 表达式无 NaN 守卫

**现象**：`start_time` 非法（空串、格式错、时区怪）时 `new Date(...)` = Invalid Date，`Math.floor(NaN)` = **NaN 直接写进联系人字段**，KlickTipp 侧无任何报错。

**实锤**（A 关断言）：输入 `start_time: "not-a-date"` → 表达式返回 NaN，无异常抛出。

**修复**（custom 版已做）：`Number.isNaN(unix) ? null : unix`，null 比 NaN 更容易在 CRM 侧被校验拦下。

## 坑#3：报名 jsonBody 缺字段无守卫

**现象**：表单没填 first_name 时，`"first_name": "undefined"`——模板表达式求值结果是 JS `undefined`，发给 Zoom 的请求体里这个键就是空的，Zoom 对 first_name 是必填校验，直接 400。

**实锤**（A 关断言）：缺 CustomFieldFirstName 的投喂 → 构建出的 body.first_name === undefined，模板无任何兜底。

**修复**（custom 版已做）：`... || 'Guest'`、`... || ''` 兜底。

## 坑#4：Zoom 手动审批模式不返回 join_url（模板注释明示）

模板便签原文警告：**webinar 设为 manual registration approval 时，报名响应里没有 join_url**（要审批通过后才生成），此时 field219991 写入 undefined。生产上要么用 automatic approval，要么轮询 registrant 状态。

## 坑#5：一堆硬编码 ID 全是模板作者账号的

直接导入必踩的「替换清单」：

| 硬编码 | 在哪 | 换成 |
|---|---|---|
| webinar ID `89062978982` / `84861299706` | 两个 httpRequest URL | 你的 webinar ID |
| listId `358895` | 两个 KlickTipp 节点 | 你的订阅列表 |
| tagId `12687810` / `12932286` | 两个 KlickTipp 节点 | 你的报名 tag |
| field219990 / field219991 | 两个 KlickTipp 节点 fields | 你建的自定义字段 ID（Zoom_webinar_Start / Zoom_Join_URL） |
| CustomField219989 | Switch 左值 | 你落地页下拉框绑定的字段 ID |
| 分支名（德语） | Switch rightValue | 必须与下拉框选项值**逐字符一致**（含 ä/ü 变音） |

## 坑#6：Trigger 和写回都是社区节点

`n8n-nodes-klicktipp.klicktippTrigger` / `.klicktipp` 是社区节点，自托管 n8n 没装就导入不了（云端 n8n Cloud 更是默认禁社区节点）。**这不是可选项，是必须替身化**——custom 版用标准 webhook + Code 节点完整保留语义。

## 坑#7：跨节点引用写法对 pairedItem 的依赖

`$('KlickTipp Trigger').item.json` 在分支里能取到触发器数据，靠的是 item 链路（pairedItem）完整。如果中间插入「重组/合并/去重」类节点把链路打断，这种写法会取到错误的 item 或报 `ERROR: Couldn't find the parent item`。A/B 两关均验证：模板现有结构下取数正确。