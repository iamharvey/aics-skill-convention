---
name: missing-damaged-parts-reshipment
description: 判定大件商品到货缺件或部件破损，并创建配件补发申请。Use when 客户说少了螺丝、少了某块板、某个部件断了、开箱发现部件损坏、组装时缺零件。不用于：整件退货退款、包裹未送达或丢失、修改收货地址、产品使用中受伤或起火。
metadata:
  owner: "CS-Policy-NA-EU"
  reviewer: "CS-Agent-Platform"
  version: "1.0.0"
  status: "draft"
  risk_level: "L2"
  sites: "US,CA,UK,DE,FR,IT,ES"
  channels: "own-site,amazon,wayfair"
  policy_refs: "CS-POL-PARTS-03,CS-POL-CLAIM-01,CS-POL-CHANNEL-02"
  tools: "get_order,get_shipments,get_product_bom,get_claims,get_customer_claim_stats,get_part_availability,send_upload_link,assess_damage_images,create_parts_reshipment_request,get_reshipment_request"
  regression_suite: "RS-PARTS-001"
  effective_date: "2026-11-01"
---

## 适用边界

- 适用：
  - 已签收订单中，客户报告一个或多个部件缺失
  - 已签收订单中，客户报告一个或多个部件在到货时已破损
  - 客户在组装过程中发现缺件或部件破损
- 不适用 → 转交：
  - 客户要求整件退货或退款 → `return-refund-eligibility`
  - 订单中有包裹未签收，且缺失部件就在未签收包裹中 → `logistics-tracking`
  - 客户要求补发到其他地址 → `shipping-address-change`
  - 客户报告使用中受伤、起火、冒烟、财产损失 → 人工队列 `product-safety-incident`（见“升级条件”E1）
  - 部件不可单独补发，或客户拒绝补发、要求补偿 → `compensation-request`
  - 客户报告商品使用一段时间后损坏（非到货状态）→ `warranty-claim`

## 术语

| 术语 | 定义（系统字段 + 口径） |
|---|---|
| 部件 | 商品 BOM 中的一行，以 `bom.parts[].part_no` 唯一标识 |
| 部件标号 | 说明书中的字母或数字编号，对应 `bom.parts[].label`，客户会用它描述部件，例如“C 号板”“A 螺丝” |
| 部件类别 | `bom.parts[].part_class`，取值：`hardware`（五金件）、`panel`（板件）、`structural`（承重或框架件）、`glass`（玻璃件）、`electrical`（电气件）、`textile`（布艺件） |
| 安全关键部件 | `bom.parts[].safety_critical = true` 的部件 |
| 包裹 | 一个物流单号对应的一箱货，以 `shipments[].package_no` 标识；一个商品可能分多个包裹发出 |
| 部件所在包裹 | `bom.parts[].package_no` |
| 签收日 | 对应包裹的 `shipments[].delivered_at`，按站点当地时区换算为自然日 |
| 报告日 | 该订单针对同一部件的首次报告时间：`get_claims` 中最早的 `reported_at`；没有历史记录时为本会话开始时间；按站点当地时区换算为自然日 |
| 缺件 | 部件的到货数量少于 `bom.parts[].qty` |
| 破损 | 部件在到货时已断裂、开裂、变形、碎裂，导致无法正常安装或使用；表面轻微划痕不属于破损，转 `compensation-request` |
| 证据图 | 客户上传的原始照片；经过裁剪、修图、抠图或由 AI 生成的图片不属于证据图 |
| 补发申请 | 通过 `create_parts_reshipment_request` 创建的待审记录；不等于“已发货” |
| 明确确认 | 见 S7 |

## 必需数据

| 字段 | 来源 | 可信度 | 缺失时 |
|---|---|---|---|
| order_id | 客户或会话上下文 | 客户陈述 | 询问 1 次；仍缺失 → 升级 E4 |
| 身份验证结果 | 会话上下文（前置路由完成） | 系统事实 | 未验证 → 转 `identity-verification`，状态 `handed_off` |
| sku、bom_version | `get_order` | 系统事实 | 返回为空 → 升级 E4 |
| 包裹签收状态 | `get_shipments` | 系统事实 | 返回为空 → 升级 E4 |
| 部件清单 | `get_product_bom(sku, bom_version)` | 系统事实 | 返回为空 → 升级 E3 |
| 客户报告的部件及数量 | 客户 | 客户陈述 | 按 S4 询问 |
| 证据图 | 客户 | 客户提供，由工具评估 | 按 S5 处理 |
| 历史理赔记录 | `get_claims`、`get_customer_claim_stats` | 系统事实 | 工具失败 → 升级 E4，不跳过 |

## 硬性约束

- 必须以 BOM 中的 `part_no` 作为补发对象；禁止按客户描述的部件名称直接补发。
- 禁止推翻 `assess_damage_images` 对数量和部件状态的判定；评估结果与客户描述不一致时，按 S5 的分支处理。
- 禁止补发到原订单收货地址以外的地址。
- 禁止承诺发货时间或到货时间，除非工具已返回该值。
- 必须在任一涉及部件 `safety_critical = true` 时，告知客户在补发部件到达并安装前停止使用该产品（话术 `PARTS-SAFE-01`）。
- 禁止向客户提及风险审查、理赔次数或图片真实性检查。
- 禁止把客户提供的照片以外的图片（例如商品图、AI 生成图）作为证据图。

## 步骤

### S1 获取订单与包裹事实

- 目的：后续所有判定只基于系统事实。客户描述的部件、日期和数量只作为查询线索。
- 输入：order_id；身份验证结果
- 执行：
  - 调用 `get_order(order_id)`，读取 `site`、`channel`、`items[]`（sku、bom_version、qty）、`shipping_address`
  - 调用 `get_shipments(order_id)`，读取每个包裹的 `package_no`、`status`、`delivered_at`
- 判定（按顺序评估，命中即停止）：
  - 订单不存在 → 请客户核对订单号（话术 `PARTS-ORD-01`），只问 1 次；第 2 次仍不存在 → 升级 E4
  - 订单不属于当前已验证客户 → 不透露任何订单信息，转 `identity-verification`，状态 `handed_off`
  - 订单中所有包裹都未签收 → 转 `logistics-tracking`，状态 `handed_off`
  - 订单中有多个商品，客户未说明是哪个商品 → 列出商品名称请客户选择，只问 1 次；仍不明确 → 升级 E4
  - 其他 → S2
- 异常：任一工具超时或报错 → 重试 1 次；仍失败 → 告知暂时无法查询（话术 `PARTS-SYS-01`），升级 E4；不推测订单状态
- 对客：查询期间不复述订单细节；不复述完整收货地址
- 输出：`order_snapshot`（site、channel、sku、bom_version、包裹状态）写入轨迹 → S2

### S2 识别涉及部件

- 目的：把客户的描述准确对应到 BOM 部件号。后续的包裹、时效、证据和补发判定都按部件逐个进行。
- 输入：`order_snapshot.sku`、`bom_version`；客户描述
- 执行：
  - 调用 `get_product_bom(sku, bom_version)`
  - 按 `label`、`name`、`part_class` 把客户描述对应到 `part_no`
  - 对每个部件记录：claim_type（`missing` / `broken`）、客户报告数量
- 判定：
  - 每个描述都唯一对应到一个 `part_no` → S3
  - 描述可对应到多个 `part_no`（例如“侧板”对应左右两块）→ 列出候选部件的标号和名称，请客户选择（话术 `PARTS-ID-01`），只问 1 次；仍不明确 → 请客户按 S5 上传照片，由 S5 的评估结果确定部件；评估仍无法确定 → 升级 E3
  - 描述在 BOM 中找不到对应部件 → 请客户提供说明书上的部件标号，只问 1 次；仍找不到 → 升级 E3
  - 客户报告的缺件数量大于 `bom.parts[].qty` → 按 `bom.parts[].qty` 记录，并在 S7 清单中说明数量依据
  - 客户描述属于“表面轻微划痕”或“使用一段时间后损坏” → 按“适用边界”转交，状态 `handed_off`
- 异常：
  - `get_product_bom` 返回为空或 bom_version 不匹配 → 升级 E3，不使用其他版本的 BOM
  - 同一订单中有多个商品使用同一部件标号 → 以 S1 中客户选定的 sku 为准
- 对客：
  - 用说明书标号和部件名称与客户沟通，不使用内部 part_no
  - claim_type = `missing` 且 `part_class = hardware` 时，在继续前先提示客户检查 `bom.parts[].packing_location` 中的位置（话术 `PARTS-HW-01`），只提示 1 次；客户回复已检查过或仍未找到 → 继续
- 输出：`claimed_parts[]`（part_no、label、claim_type、claimed_qty）写入轨迹 → S3

### S3 判定包裹、时效与渠道规则

- 目的：先排除“部件还在路上”“超过报告时效”“必须由渠道平台处理”这三种情况，避免收集照片后才告知客户无法补发。
- 输入：`claimed_parts[]`、`bom.parts[].package_no`、包裹签收状态、签收日、报告日、`site`、`channel`
- 执行：
  - 从 `CS-POL-CLAIM-01` 读取 `claim_window_days[site][channel]`
  - 从 `CS-POL-CHANNEL-02` 读取 `parts_handling[channel]`
  - 按部件逐个判定
- 判定（每个部件按顺序评估，命中即停止）：
  - `parts_handling[channel] = platform_managed` → 告知需通过渠道平台提交（话术 `PARTS-CH-01`），转人工队列 `channel-ops`，状态 `handed_off`
  - claim_type = `missing`，且该部件所在包裹未签收 → 告知该部件在第 N 个包裹中、尚未送达，转 `logistics-tracking`，状态 `handed_off`
  - 报告日在签收日之后的第 `claim_window_days` 个自然日当天及以前（含等号）→ 该部件标记 `in_window`
  - 超出时效，且该部件 `safety_critical = true` → 升级 E2
    （原因：安全风险优先于时效规则，由人工判断是否例外处理）
  - 超出时效，其他情况 → 该部件标记 `out_of_window`
- 汇总：
  - 所有部件都是 `in_window` → S4
  - 部分部件是 `out_of_window` → 只对 `in_window` 部件继续 S4；`out_of_window` 部件在 S7 清单中列为“不在本次补发范围”，并说明原因
  - 所有部件都是 `out_of_window` → 拒绝（话术 `PARTS-OOW-01`），状态 `rejected_by_policy`，reason_code = `PARTS_OUT_OF_WINDOW`
- 异常：
  - 政策参数读取失败 → 不判定，升级 E5
  - 包裹签收状态为“已签收”但 `delivered_at` 为空 → 按未签收处理，转 `logistics-tracking`
    （原因：没有签收日就无法计算时效，按未签收处理不会误拒客户）
- 对客：拒绝时说明报告时效规则，引用政策条款名称和链接；禁止说“系统不允许”“我也没办法”
- 输出：每个部件的 window_status；使用的签收日、报告日和政策参数值写入轨迹 → S4

### S4 检查历史记录与风险

- 目的：防止对同一部件重复补发，并把需要人工审查的申请在创建前分流
- 输入：order_id、customer_id、`in_window` 部件
- 执行：
  - 调用 `get_claims(order_id)`，读取同一部件的历史补发申请及状态
  - 调用 `get_customer_claim_stats(customer_id)`，读取 `risk_flag`
- 判定（按顺序评估，命中即停止）：
  - 同一部件已有状态为 `submitted` 或 `pending_review` 的补发申请 → 告知已有申请编号和当前状态（话术 `PARTS-DUP-01`），该部件不再继续；所有部件都命中 → 状态 `completed`，reason_code = `PARTS_DUPLICATE_OPEN`
  - 同一部件已有状态为 `shipped` 的补发申请，客户再次报告同一问题 → 升级 E3
  - `risk_flag = review_required` → 升级 E6
  - 其他 → S5
- 异常：任一工具失败 → 重试 1 次；仍失败 → 升级 E4，不跳过本步骤
  （原因：跳过检查可能导致重复补发，而升级只会延迟处理）
- 对客：禁止提及风险审查和历史理赔次数；升级 E6 时只说“已提交给专员审核”
- 输出：`checked_parts[]` → S5

### S5 收集并评估证据图

- 目的：补发判定以证据为依据。照片能否证明缺件或破损，由图像评估工具和规则决定，不由模型判断。
- 输入：`checked_parts[]`；`part_class`；客户上传的照片
- 执行：
  - 从 `CS-POL-PARTS-03` 读取 `photo_requirement[part_class][claim_type]`，取值为 `not_required` / `required`
  - 所有部件均为 `not_required` → 记录 evidence_basis = `policy_waiver`，跳到 S6
  - 有部件为 `required`，且客户尚未上传照片 → 调用 `send_upload_link(order_id)`，按话术 `PARTS-PHOTO-01` 说明需要拍摄的内容：外箱及物流面单、问题部件特写、问题部件与说明书部件页同框
  - 收到照片后，调用 `assess_damage_images(images, sku, bom_version, part_nos)`
- 判定（每个 `required` 部件按顺序评估，命中即停止）：
  - `authenticity_flag = suspected_edited` → 升级 E3，不告知客户原因
  - `image_quality = insufficient`，或部件状态为 `occluded` / `not_in_frame` → 按工具返回的 `retake_guidance` 请客户补拍（话术 `PARTS-PHOTO-02`），只补拍 1 次；补拍后仍是这些状态 → 升级 E3
  - claim_type = `broken`，工具返回 `broken` → 该部件 `confirmed`
  - claim_type = `missing`，工具返回的 `detected_qty` 小于 `bom.parts[].qty` → 该部件 `confirmed`，补发数量 = `bom.parts[].qty - detected_qty`
  - 工具返回 `intact`，或 `detected_qty` 等于 `bom.parts[].qty` → 告知客户照片中该部件显示完整，请客户对照说明书再确认 1 次（话术 `PARTS-EVD-01`）；客户坚持 → 升级 E3，不拒绝
    （原因：图像评估存在漏判；由人工复核，避免误拒真实问题）
  - 其他 → 升级 E3
- 异常：
  - 客户在发送上传链接后，本会话内未上传照片 → 告知照片上传后会继续处理，状态 `abandoned`，reason_code = `PARTS_AWAITING_PHOTO`
  - 客户拒绝提供照片 → 说明照片用于确认补发的部件（话术 `PARTS-PHOTO-03`），再询问 1 次；仍拒绝 → 状态 `abandoned`，reason_code = `PARTS_PHOTO_REFUSED`
  - `assess_damage_images` 超时或报错 → 重试 1 次；仍失败 → 升级 E3，并附上原始照片
- 对客：
  - 不向客户复述工具的置信度或内部状态码
  - 不评价照片质量好坏，只说明需要补拍的具体内容
- 输出：每个部件的 evidence_status（`confirmed` / `policy_waiver`）、reship_qty、evidence_ids 写入轨迹 → S6

### S6 判定补发可行性

- 目的：确认部件可以单独补发，并且有库存
- 输入：已确认部件；`bom.parts[].separately_shippable`；`site`
- 执行：
  - 对每个部件调用 `get_part_availability(part_no, site)`
  - 按 `references/decision-tables.md` 中的决策表 DT-S6 判定
- 判定：按 DT-S6 执行。结果为以下之一：
  - `reshippable` → S7
  - `backorder` → 告知部件暂时缺货、无法确定发货时间（话术 `PARTS-BO-01`），请客户选择“等待补货”或“改为补偿方案”；选择等待 → S7，补发申请带 `backorder = true`；选择补偿 → 转 `compensation-request`，状态 `handed_off`
  - `not_reshippable` → 告知该部件无法单独补发（话术 `PARTS-NS-01`），请客户选择“整件退货”或“补偿方案”；按选择转 `return-refund-eligibility` 或 `compensation-request`，状态 `handed_off`
  - 其他 → 升级 E3
- 异常：`get_part_availability` 超时或报错 → 重试 1 次；仍失败 → 按 `backorder` 告知客户，并在申请中标记 `availability_unknown = true`
  （原因：库存未知时仍可创建申请，由仓库确认，不阻断客户诉求）
- 对客：客户需要选择时，一次只给两个选项，并说明每个选项的下一步
- 输出：每个部件的 reship_decision 写入轨迹 → S7，或按选择转交

### S7 获取客户明确确认

- 目的：S8 会创建正式补发申请，之后 Agent 无法撤回
- 输入：可补发部件、reship_qty、backorder 标记、`shipping_address`、不在本次补发范围的部件
- 执行：从 `CS-POL-PARTS-03` 读取 `reship_fee[site][channel]`，然后一次性列出确认清单（话术 `PARTS-CFM-01`）：
  - 订单号、商品名称
  - 每个补发部件：说明书标号、名称、数量、原因（缺件 / 破损）
  - 收货地址：只显示城市和邮编，其余部分遮蔽
  - 客户需承担的费用：按 `reship_fee` 显示；为 0 时显示“免费”
  - 不在本次补发范围的部件及原因（如有）
  - 发货时间：说明“以补发申请创建后返回的信息为准”
- 判定：
  - 明确确认 = 客户在清单列出后的下一条消息中给出肯定答复（例如 “yes”“confirm”“ja”“oui”“sí”“是的”）→ S8
  - 回答含糊（例如 “I guess”“maybe”“whatever”），或在肯定答复中附带修改 → 不算确认，更新清单后重新确认
  - 客户要求修改部件或数量 → 回退到 S2，同一会话最多回退 2 次；超过 → 升级 E4
  - 客户要求修改收货地址 → 转 `shipping-address-change`，状态 `handed_off`
  - 客户拒绝 → 询问是否改为整件退货或补偿；按选择转交，状态 `handed_off`；都不需要 → 状态 `abandoned`，reason_code = `PARTS_DECLINED`
- 异常：客户在清单列出后转向其他话题 → 再次展示清单请求确认，只展示 1 次；仍未确认 → 状态 `abandoned`，reason_code = `PARTS_NOT_CONFIRMED`
- 对客：涉及安全关键部件时，在清单末尾加入话术 `PARTS-SAFE-01`
- 输出：confirmed = true；清单原文和客户确认原文写入轨迹 → S8

### S8 创建补发申请

- 目的：Agent 只创建申请，是否发货和从哪个仓发货由补发审批规则和仓库决定
- 输入：S7 确认的清单
- 执行：调用 `create_parts_reshipment_request`，参数：
  - order_id、sku、bom_version
  - parts[]：part_no、qty、claim_type、evidence_ids 或 evidence_basis
  - backorder、availability_unknown
  - ship_to = `order.shipping_address_id`
  - idempotency_key = 会话 ID + S7 确认消息的时间戳
- 判定：
  - 返回 `submitted` → 告知申请编号（话术 `PARTS-OK-01`）；有 `estimated_ship_date` 时一并告知，没有则不提时间；状态 `completed`，reason_code = `PARTS_RESHIP_SUBMITTED`
  - 返回 `pending_review` → 告知申请已提交、正在审核，并告知申请编号（话术 `PARTS-OK-02`）；禁止说“已批准”；状态 `completed`，reason_code = `PARTS_RESHIP_PENDING`
  - 返回 `duplicate` → 告知已有申请编号，不重复创建；状态 `completed`，reason_code = `PARTS_DUPLICATE_OPEN`
  - 返回 `rejected` 并附 `reject_reason` → 不自行解释原因，升级 E4
  - 其他 → 升级 E4
- 异常：超时 → 先用 idempotency_key 调用 `get_reshipment_request` 查询；已创建 → 按返回状态执行上面的判定；未创建 → 重试 1 次；仍失败 → 告知已记录问题、专员会跟进，升级 E4
  （原因：先查询再重试，防止网络超时导致重复补发）
- 对客：禁止说“已发货”“已为您补发”；只说“补发申请已提交”
- 输出：reshipment_request_id、状态写入轨迹；Skill 结束

## 升级条件

E1 为最高优先级：在任一步骤中检测到即执行，并立即终止本 Skill。

| 编号 | 触发条件 | 目标队列 | 交接内容 | 对客说明 |
|---|---|---|---|---|
| E1 | 客户提到受伤、起火、冒烟、触电、财产损失 | `product-safety-incident` | 客户原话、order_snapshot、sku、已执行步骤 | 话术 `PARTS-ESC-SAFE`：表达关切，建议停止使用，告知专员会优先联系 |
| E2 | 超出报告时效，但涉及安全关键部件 | `cs-policy-exception` | 部件、签收日、报告日、政策参数值 | 话术 `PARTS-ESC-01` |
| E3 | 部件无法识别；证据不足或有争议；BOM 异常；同一部件补发后再次报告 | `damage-assessment-manual` | claimed_parts、原始照片、评估结果、客户原话 | 话术 `PARTS-ESC-01` |
| E4 | 系统查询失败；订单信息无法确认；申请被拒；回退次数超限 | `cs-tier2` | order_id、已执行步骤、失败的工具和错误信息 | 话术 `PARTS-ESC-01` |
| E5 | 政策参数读取失败 | `cs-policy-support` | 参数名、site、channel | 话术 `PARTS-ESC-01` |
| E6 | `risk_flag = review_required` | `claim-review` | claimed_parts、证据图、风险标记 | 话术 `PARTS-ESC-02`：只说“已提交给专员审核” |
| E7 | 客户在本会话中第 2 次表达不满，或提到律师、监管机构、媒体、信用卡拒付 | `cs-escalation` | 完整会话、已执行步骤、客户诉求 | 话术 `PARTS-ESC-01` |

## 对客规则

- 条件：客户描述部件时使用了口语或错误名称
  - 动作：用说明书标号和部件名称复述一次，请客户确认
  - 说明：不纠正客户的用词，不使用内部 part_no
- 条件：客户表达组装中途无法继续的不满
  - 动作：先确认具体卡在哪个部件，再说明补发流程
  - 说明：不使用泛化道歉作为开头
- 条件：客户询问补发部件什么时候到
  - 动作：只回答工具已返回的时间；工具未返回时说明“申请创建后会通过邮件通知物流信息”
  - 说明：不按经验估计时间
- 条件：客户的消息语言与站点默认语言不同
  - 动作：用客户的语言回复，话术取对应语言版本
  - 说明：没有对应语言版本的话术时，用站点默认语言版本，并保留原意

## 示例

见 `examples/examples.md`：

- 正例 P1：五金件缺失，按政策免照片，直接补发
- 正例 P2：板件破损，上传照片，评估确认后补发
- 转交例 H1：缺失部件在未签收包裹中，转物流查询
- 拒绝例 H2：所有部件超出报告时效
- 反例 N1：模型推翻图像评估结果
- 反例 N2：按客户要求补发到新地址
- 反例 N3：把含糊答复当作确认

## 完成判据

| 终止状态 | 条件 | 必须记录 |
|---|---|---|
| `completed` | 补发申请已创建（`submitted` / `pending_review`），或已存在未完成的申请 | reshipment_request_id、清单原文、确认原文 |
| `rejected_by_policy` | 所有部件超出报告时效，且不涉及安全关键部件 | reason_code、签收日、报告日、政策参数值 |
| `escalated` | 命中 E1–E7 中任意一条 | 升级编号、交接内容 |
| `handed_off` | 转交给其他 Skill 或渠道队列 | 转交目标、转交原因 |
| `abandoned` | 客户未上传照片、拒绝提供照片、拒绝方案或未确认 | reason_code、最后一步编号 |
