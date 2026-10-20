# 00 · 总览

## 业务场景

你在做「邮件送达率」系列网络研讨会（Webinar），分**新手场**和**专家场**。落地页用 KlickTipp 搭建，访客填表时下拉选择想参加的场次。模板把后续动作全部自动化：

1. 访客提交落地页表单 → KlickTipp 出站 webhook 把数据推给 n8n
2. n8n 按「选的哪场」分流
3. 调 Zoom API 把这个人注册进对应 webinar，拿回专属参会链接 `join_url`
4. 把 `join_url`、**未来最近一场**的开课时间（Unix 秒）、报名 tag 写回 KlickTipp 联系人档案
5. KlickTipp 的营销自动化（提醒邮件序列）按 tag 启动——因为数据是同一次调用写入的，**永远不会出现「提醒邮件发了但链接还没到」**

## 为什么值得学

- 这是营销自动化（MarTech）里最经典的「表单 → 分流 → 外部注册 → CRM 回写」四段式漏斗，换个皮就是：活动报名、课程试听、预约演示、白皮书下载分流
- 模板虽小（3 功能节点），但藏了一个高质量表达式（未来最近场计算）和三个生产级坑，教学密度很高
- 中国环境下 KlickTipp/Zoom 都可替身化：表单系统任意、Zoom 换腾讯会议/钉钉直播 API 或 mock，骨架完全复用

## 关键概念词汇

| 词 | 含义 |
|---|---|
| KlickTipp | 德国邮件营销/CRM SaaS（本模板作者用它；本课用免费替身） |
| Outbound webhook | KlickTipp 的出站钩子：联系人被打上指定 tag 时向外部 URL POST 数据 |
| Webinar occurrence | 循环研讨会的单场次；一个 webinar 可有多场，各有 start_time 和 status |
| join_url | Zoom 给每个报名者生成的专属参会链接 |
| tag/list | KlickTipp 的联系人分群机制，tag 触发营销活动 |