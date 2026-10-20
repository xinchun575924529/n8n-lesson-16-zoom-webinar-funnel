# 05 · 生产加固清单

在 custom 修复版之上，真上生产还要做这些：

## 1. 兜底与告警（坑#1 配套）

- Switch `fallbackOutput=extra` 已开 → 兜底分支别只打日志，接**飞书群告警**（值不一致通常意味着落地页改了选项没同步分支，越早知道越好）
- 定期对账：兜底分支出现率 > 0 就要查下拉框值与分支名的逐字符一致性（尤其德语 ä/ü、尾部空格）

## 2. 数据质量守卫

- occurrence Unix 秒：`Number.isNaN(unix) ? null : unix`（custom 已做）；生产再加「null 时不启动营销活动」——field219990 为空就不打报名 tag，或拆成两次调用先字段后 tag
- 报名体：first_name/last_name 兜底（custom 已做）；email 加正则预校验，非法邮箱直接进兜底分支，不浪费 Zoom API 配额

## 3. Zoom 侧

- webinar 用 **automatic approval**（坑#4）；必须人工审批的场景：报名后轮询 `GET /webinars/{id}/registrants/{registrant_id}` 拿到 join_url 再回写
- OAuth2 token 有过期；连续测试会撞 rate limit——模板便签原文提醒 stagger test intervals
- Zoom 429/5xx：httpRequest 节点开 `retryOnFail`（maxTries 3 + waitBetweenTries 4000）

## 4. 幂等与防重

- 同一 email 重复提交表单：KlickTipp subscribe 天然幂等（同 email 覆盖字段）；替身到多维表/本地 JSON 时自己加「按 email 查重 → update 而非 insert」
- webhook 重放：出站钩子可能重发，按 `email + webinar 选择` 去重（本厂 L1B 的 dedupe 模式可复用）

## 5. 配置外置

硬编码 ID 清单（坑#5）全部外置到工作流开头一个 Set 节点（webinar IDs、listId、tagIds、fieldIds、分支名映射表），换环境只改一处。分支名与下拉框选项值建议做成 `{ "选项显示值": { "webinarId": ..., "tagId": ... } }` 字典驱动，新增场次 = 加一行字典，不用动 Switch。

## 6. 错误网

- Zoom 报名成功但回写失败 = 报了名没链接发给用户 → 回写节点挂 n8n 错误工作流（Error Trigger）+ 飞书告警，人工补写
- webhook 入口接 `Respond to Webhook` 立即 200，把慢链路（Zoom+回写）异步化，防止表单系统超时重发（与第 4 点去重配合）

## 7. 隐私合规

- 表单收集姓名+邮箱 → 中国 PIPL / 欧洲 GDPR（模板是德国 SaaS，原作者面向欧盟用户）：落地页要有勾选同意；写回 CRM 的字段最小化；兜底日志里别长期存明文邮箱