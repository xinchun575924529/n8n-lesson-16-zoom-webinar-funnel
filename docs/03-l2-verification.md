# 03 · 三层验证报告（L2 全绿 29/29）

验证环境：本机 n8n 2.33.3 CLI（executeWorkflowTrigger 驱动），2026-10-20。

## A 单元级 · 12/12 ✅（L2Test841600001）

| 套件 | 断言 | 内容 |
|---|---|---|
| occurrence-expr | 5/5 | ①循环 webinar 取未来最近场（deleted+过去场被滤）②无 occurrences 回退 start_time ③全部过期/deleted 回退 ④仅 deleted 未来场回退 ⑤垃圾日期→NaN 静默（坑#2 复现） |
| switch-out0 | 2/2 | 真 Switch v3.2 原参数：精确德语值入 A 分支；strict+caseSensitive 生效 |
| switch-out1 | 1/1 | 精确值入 B 分支 |
| switch-silent-drop | 1/1 | 4 进 2 出：小写变体+未知值被静默丢弃（坑#1 复现） |
| zoom-jsonbody | 3/3 | 键名/umlaut（Klünter）透传；缺 first_name→undefined 静默（坑#3 复现） |

> 首轮 10/12：两处失败为测试自身问题（预期写成次近场而非最近场；silent-drop 断言受分支单跑视角限制，改 `$('节点').all()` 跨节点统计），修正后全绿——模板逻辑零缺陷。

## B mock 全链 · 10/10 ✅（L2Mock841600001）

保留模板**全节点名全连线**，外部依赖替换：KlickTipp Trigger→Code 投喂 3 场景；Zoom OAuth2 httpRequest→mock 响应；KlickTipp 写回→Code 仿真（字段+tag+list 语义一致）。

| 场景 | 结果 |
|---|---|
| S1 新手场 | 5/5：email 透传、join_url→field219991、Unix 秒=未来最近场（occurrences 含 deleted+过去场）、tagId/listId 正确 |
| S2 专家场 | 4/4：Unix 秒=回退单场 start_time（无 occurrences 分支）、tagId 正确 |
| S3 未匹配 | 1/1：3 进 2 出，静默丢弃实锤 |

## C 免费替身链真跑 · 7/7 ✅（L2Cstm84160001）

**真实成分**，零付费凭证：

- httpRequest 节点（模板原结构、原 jsonBody 表达式）**真实 POST** 到本地 mock Zoom HTTP 服务（python，127.0.0.1:18416），真实 201 响应含 join_url/occurrences
- 写回节点用 fs **真实落盘** `contacts.json` 联系人档案
- Verify 节点**从落盘文件回读**验证（真 E2E）

核验记录：A 场 `join_url=…89062978982?tk=real-mock`、`field219990=1792951200`（=2026-10-25T18:00Z 未来最近场）；B 场 `field219990=1793037600`（=2026-10-26T18:00Z 单场回退）——全对。

## 结论

模板逻辑本身零缺陷；三个生产级坑（静默丢弃/NaN/缺字段守卫）全部复现并在 custom 版修复。