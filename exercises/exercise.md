# 动手练习（含答案）

## 练习 1：新增第三场「进阶场」

模板只有两个分支。请新增第三场研讨会「E-Mail Zustellung für Profis」（webinar ID 假定为 999000111222，tagId 假定为 13000001），写出需要改动的所有位置。

<details><summary>答案</summary>

1. Switch `rules.values` 加第三条：outputKey/rightValue = `E-Mail Zustellung für Profis`（与下拉框选项逐字符一致）
2. 复制一个 Register httpRequest 节点，URL webinar ID 换成 999000111222，接 Switch 输出 2
3. 复制一个写回节点，tagId 换 13000001，接新 Register 节点
4. （生产加固做法）更优：按 docs/05 第 5 点把分支改成字典驱动，加一行 `{ "E-Mail Zustellung für Profis": { webinarId: 999000111222, tagId: 13000001 } }` 即可，Switch 只需一个动态查表分支

</details>

## 练习 2：occurrence 表达式心算

Zoom 响应如下，写回 field219990 的 Unix 秒是多少（假设「现在」= 2026-10-20T12:00:00Z）？

```json
{
  "start_time": "2026-10-18T18:00:00Z",
  "occurrences": [
    { "start_time": "2026-10-19T18:00:00Z", "status": "available" },
    { "start_time": "2026-10-22T18:00:00Z", "status": "deleted" },
    { "start_time": "2026-10-25T18:00:00Z", "status": "available" }
  ]
}
```

<details><summary>答案</summary>

过滤：10-19 场是过去（早于现在）滤掉；10-22 场 deleted 滤掉；剩 10-25 场 → 取它。
`2026-10-25T18:00:00Z` 的 Unix 秒 = **1792951200**。
（本课 C 关真跑落盘的正是这个值。）

</details>

## 练习 3：修复「报名成功但用户没收到链接」

学员反馈：表单提交成功、Zoom 后台能看到报名记录，但 CRM 里 join_url 字段是空的。列出至少两个可能原因和对应排查动作。

<details><summary>答案</summary>

1. **webinar 是手动审批模式**（坑#4）：报名响应无 join_url → 查 Zoom 后台该 webinar 的 approval 设置；改 automatic 或加轮询
2. **值不匹配被 Switch 静默丢弃**（坑#1）：其实根本没进报名分支，「Zoom 有记录」是别人手动加的 → 查 n8n 执行日志里 Switch 输出是否为空；开 fallbackOutput 兜底验证
3. **跨节点引用断链**（坑#7）：中间插了重组节点导致 `$('KlickTipp Trigger').item` 取错 → 查执行详情里写回节点的输入 email 是否等于报名者
4. **回写节点凭证/字段 ID 错**：写入报错误吞（若有 onError=continue）→ 查执行状态与 KlickTipp API 响应

</details>