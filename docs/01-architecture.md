# 01 · 架构

## 节点与连线（模板原样）

```text
KlickTipp Trigger (社区节点, webhook path=zoom_webinar_registration)
  └→ Switch (v3.2, 按 CustomField219989 严格匹配)
       ├─输出0 "E-Mail Zustellung für Anfänger"→ Register for webinar A (httpRequest→Zoom)
       │    └→ Add "Anfänger" webinar data to contact (KlickTipp subscribe)
       └─输出1 "E-Mail Zustellung für Experten"→ Register for webinar B (httpRequest→Zoom)
            └→ Add "Experten" webinar data to contact (KlickTipp subscribe)
```

两条分支结构完全对称，只是 webinar ID / tagId 不同。5 张 StickyNote 是教学说明（含一张超长社区节点免责+配置指引）。

## 数据流

**入口载荷**（KlickTipp 出站钩子 POST 体，模板 pinData 实例）：

```json
{
  "email": "ricardo.kluenter+...@klicktipptest.com",
  "CustomFieldFirstName": "Ricardo",
  "CustomFieldLastName": "Klünter",
  "CustomField219989": "E-Mail Zustellung für Anfänger"
}
```

- `CustomField219989` = 落地页下拉框「选哪场」写入的自定义字段（ID 是模板作者账号的，见坑集）
- Switch 左值 `{{ $json.CustomField219989 }}`，与两个德语分支名严格相等比较

**Zoom 报名请求体**（两个 httpRequest 节点，jsonBody）：

```json
{
  "first_name": "{{ $('KlickTipp Trigger').item.json.CustomFieldFirstName }}",
  "last_name": "{{ $('KlickTipp Trigger').item.json.CustomFieldLastName }}",
  "email": "{{ $('KlickTipp Trigger').item.json.email }}"
}
```

注意引用方式是 `$('节点名').item.json` 跨节点回捞触发器数据（因为 httpRequest 的输入是 Switch 路由后的同一条 item，这样写更稳）。

**Zoom 响应**（关键字段）：

```json
{
  "id": 89062978982,
  "registrant_id": "...",
  "join_url": "https://zoom.us/w/89062978982?tk=...",
  "start_time": "2026-10-18T18:00:00Z",
  "occurrences": [
    { "start_time": "2026-10-25T18:00:00Z", "status": "available" },
    { "start_time": "2026-11-01T18:00:00Z", "status": "available" }
  ]
}
```

单场 webinar 没有 `occurrences` 键；循环 webinar 才有，且可能含 `status: "deleted"` 和已过去的场次。

**KlickTipp 写回**（subscribe 操作一次调用写入）：

| 写入 | 值 |
|---|---|
| listId | 358895（订阅列表） |
| tagId | 12687810 / 12932286（报名 tag，触发营销活动） |
| field219991 | `{{ $json.join_url }}` |
| field219990 | 未来最近场 Unix 秒（核心表达式，见下） |

## 核心表达式逐行拆解（field219990）

```js
Math.floor(new Date(
  (
    ($json.occurrences || [])
      .filter(o => o.status !== 'deleted' && new Date(o.start_time) > new Date())
      .sort((a, b) => new Date(a.start_time) - new Date(b.start_time))[0]
    || $json
  ).start_time
).getTime() / 1000)
```

1. `($json.occurrences || [])`——单场 webinar 无此键，兜底空数组
2. `.filter(...)`——只要「非 deleted 且开始时间在未来」的场次
3. `.sort(升序)[0]`——取**未来最近**一场
4. `|| $json`——没有合格场次（单场/全过期/全 deleted）时回退用响应顶层 `start_time`
5. `Math.floor(new Date(...).getTime()/1000)`——ISO → Unix 秒

本课 A 关用 5 个分支用例把它全部实测覆盖（含垃圾日期 → NaN 的坑，见 02-pitfalls.md）。

## 凭证

| 凭证 | 类型 | 用途 |
|---|---|---|
| KlickTipp API | 社区节点凭证 | Trigger + 两个写回节点 |
| Zoom OAuth2 | zoomOAuth2Api（scope `webinar:write:registrant`） | 两个报名 httpRequest |