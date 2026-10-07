# 决策表

## DT-S6 补发可行性

按部件逐行评估，按规则 ID 顺序评估，命中即停止。

| 规则 ID | separately_shippable | get_part_availability | 其他条件 | 结果 | 去向 | reason_code |
|---|---|---|---|---|---|---|
| R01 | false | 任意 | — | `not_reshippable` | S6 选项：整件退货 / 补偿 | `PARTS_NOT_SEPARABLE` |
| R02 | 空值 | 任意 | — | 无法判定 | 升级 E3 | `PARTS_BOM_INCOMPLETE` |
| R03 | true | `discontinued` | — | `not_reshippable` | S6 选项：整件退货 / 补偿 | `PARTS_DISCONTINUED` |
| R04 | true | `backorder` | — | `backorder` | S6 选项：等待 / 补偿 | `PARTS_BACKORDER` |
| R05 | true | `available` | reship_qty ≤ `bom.parts[].qty` | `reshippable` | S7 | — |
| R06 | true | `available` | reship_qty > `bom.parts[].qty` | 无法判定 | 升级 E3 | `PARTS_QTY_EXCEEDS_BOM` |
| R07 | 其他组合 | 其他组合 | — | 无法判定 | 升级 E3 | `PARTS_UNDETERMINED` |

## 多部件汇总规则

一个会话中涉及多个部件时，先按部件逐个得出结果，再按以下规则汇总：

| 情况 | 处理 |
|---|---|
| 全部 `reshippable` | S7，一张清单列出全部部件 |
| 部分 `reshippable`，部分 `backorder` | 先让客户对 `backorder` 部件做选择，再进入 S7；清单中分别标注 |
| 部分 `reshippable`，部分 `not_reshippable` | 先说明 `not_reshippable` 部件的选项。客户选择整件退货 → 整单转 `return-refund-eligibility`，不创建补发申请；客户选择补偿 → 可补发部件进入 S7，不可补发部件转 `compensation-request` |
| 任一部件升级 | 整单升级，不拆分处理 |

整单升级的原因：同一订单拆成“部分自动补发 + 部分人工处理”，容易出现重复补发或客户收到矛盾的说明。
