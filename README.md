# n8n 实战课 16：落地页表单 → Zoom 网络研讨会自动报名漏斗（KlickTipp 模板中国免费替身版）

> 官方模板：[Register Zoom webinar attendees from KlickTipp landing page forms](https://n8n.io/workflows/8416/)（模板 ID 8416，3 功能节点 + 5 张教学便签）

访客在落地页填表（含「选哪个研讨会」下拉框）→ 出站 webhook 触发 n8n → Switch 按选择分流 → 调 Zoom API 完成报名拿回 `join_url` 和开始时间 → 把参会链接、未来最近一场的开课时间（Unix 秒）、报名 tag **一次调用**写回联系人档案，营销自动化活动在数据就绪后才启动。零手工搬运、按选题动态分流、可扩展到任意多场次。

## 你会学到

1. **Webhook 触发的营销漏斗骨架**：表单提交 → 分流 → 外部 API 报名 → CRM 回写，4 步闭环
2. **Switch v3.2 严格匹配**：`typeValidation: strict` + `caseSensitive` 的真实行为（小写变体直接被拒）
3. **「取未来最近一场」表达式逐行拆解**：`(occurrences||[]).filter(非 deleted 且未来).sort(升序)[0] || 回退 $json.start_time` 再转 Unix 秒——5 种分支全部单测覆盖
4. **Zoom Webinar Registrants API**：`POST /v2/webinars/{id}/registrants`（OAuth2，scope `webinar:write:registrant`）
5. **字段+tag+list 一体写入**的精妙设计：规避「活动先发、数据后到」的竞态
6. **三大实锤坑**（全部经断言复现）：Switch 静默丢弃 / occurrence 表达式无 NaN 守卫 / jsonBody 缺字段无守卫（见 docs/02-pitfalls.md）

## 课程文件

| 文件 | 内容 |
|---|---|
| `workflow.json` | 官方模板原样（3 功能节点，n8n 可导入形态；注意含社区节点与原作者凭证占位） |
| `workflow-custom.json` | **中国免费替身修复版**：webhook 替代社区 Trigger、Code 替代 KlickTipp 写回（落本地 JSON）、三大坑全部修复 |
| `docs/00~05` | 总览 / 架构 / 坑集 / 三层验证报告 / 免费替身 / 生产加固 |
| `exercises/exercise.md` | 3 道动手练习（含答案） |
| `script/short-video.md` | 获客短视频口播脚本 |

## 三层验证结论（本课全部实测通过）

- **A 单元级**：occurrences 表达式 5 分支 + 真 Switch v3.2 严格匹配 + 报名体构建，**12/12 断言全过**（含静默丢弃、NaN 静默两处坑复现）
- **B mock 全链**：保留模板全节点名全连线，外部依赖全替身，三场景（新手场/专家场/未匹配）**10/10 断言全过**
- **C 免费真跑**：httpRequest 节点**真实 POST** 本地 mock Zoom 服务 + 落盘联系人档案回读验证，端到端 0 元 **7/7 全过**（A 场正确取到未来最近场 2026-10-25T18:00Z、B 场正确回退单场 start_time）

## 一分钟看懂它干嘛

```text
KlickTipp 落地页表单提交（含研讨会选择）
  → 出站 webhook 触发 n8n
  → Switch 按 CustomField219989 分流（新手场 / 专家场）
  → Zoom API 报名，拿回 join_url + occurrences
  → KlickTipp 联系人写回：join_url + 未来最近场 Unix 秒 + 报名 tag（一次调用）
  → 营销活动在数据就绪后才启动
```