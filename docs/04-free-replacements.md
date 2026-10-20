# 04 · 中国免费替身方案（0 元跑通全链）

## 替身对照表

| 模板依赖 | 付费？ | 免费替身 | 语义保留度 |
|---|---|---|---|
| KlickTipp Trigger（社区节点 + 付费 SaaS） | 付费 | **标准 webhook 节点**：任何表单系统（金数据/问卷星/自建 HTML 表单/飞书问卷）POST 到 `/webhook/zoom_webinar_registration` | 100%（都是 HTTP 进站） |
| KlickTipp subscribe 写回（社区节点） | 付费 | **Code 节点写本地 JSON**（custom 版默认）/ **飞书多维表**（免费，tenant token 直连 OpenAPI）/ 任意 HTTP CRM | 100%（字段+tag+list 一体写入语义保留：record 一条文档同写三样） |
| Zoom OAuth2 webinar 报名 | 付费（Pro+webinar 许可） | ①**本地 mock HTTP 服务**（本课 C 关做法，httpRequest 真实 POST）②腾讯会议/钉钉直播 API（企业认证免费）③Zoom 免费账号的普通 meeting + registration（功能裁剪但可跑） | mock=100% 教学语义；国产会议 API 需自适配字段 |

## 替身版数据流（workflow-custom.json）

```text
webhook (POST /webhook/zoom_webinar_registration)
  → Switch（含 fallbackOutput 兜底分支）
  → httpRequest（Zoom 或你的国产会议 API 或 mock）
  → Code（字段+tag+list 一体 → 本地 JSON / 飞书多维表）
  → 未匹配-兜底 → 日志/告警
```

## 接飞书多维表的最小改动

把 custom 版写回节点里的「fs 写本地 JSON」换成 tenant token 直连：

```js
const appToken = '你的多维表 app_token';
const tableId = '你的 table_id';
const { token } = (await this.helpers.httpRequest({
  method: 'POST', url: 'https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal',
  body: { app_id: '你的 app_id', app_secret: '你的 app_secret' }, json: true
})).tenant_access_token ? await (async () => { const r = await this.helpers.httpRequest({ method: 'POST', url: 'https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal', body: { app_id: '你的 app_id', app_secret: '你的 app_secret' }, json: true }); return { token: r.tenant_access_token }; })() : {};
await this.helpers.httpRequest({
  method: 'POST',
  url: `https://open.feishu.cn/open-apis/bitable/v1/apps/${appToken}/tables/${tableId}/records`,
  headers: { Authorization: 'Bearer ' + token },
  body: { fields: { '邮箱': record.email, '参会链接': { link: record.field219991 }, '开课Unix秒': record.field219990, '报名tag': record.tagId } },
  json: true
});
```

> ⚠️ bitable URL 型字段必须传 `{"link": url}` 对象，纯字符串会报 URLFieldConvFail(1254068)；数字字段传 int。

## 本地 mock Zoom 服务（教学/测试用）

本课 C 关同款：30 行 python http.server，`POST /v2/webinars/{id}/registrants` 返回 201 + join_url + occurrences（A 场带 deleted+多场、B 场单场），源码见课程 GitHub 讨论区或自行按 docs/01 的响应结构仿写。