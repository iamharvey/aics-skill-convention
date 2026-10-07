# 对客话术（English）

使用说明：

- 带“原文输出”标记的话术，必须逐字输出，只替换 `{变量}`。
- 带“要点”标记的话术，只需传达列出的要点，措辞可以自然调整。
- 其他语言版本放在 `scripts-de.md`、`scripts-fr.md`、`scripts-it.md`、`scripts-es.md`，话术 ID 保持一致。
- 本文件中带“原文输出”标记的话术，须经各站点本地合规确认后才能上线。

| 话术 ID | 类型 | 内容 |
|---|---|---|
| PARTS-ORD-01 | 要点 | 没有找到这个订单号；请客户核对确认邮件中的订单号 |
| PARTS-SYS-01 | 要点 | 暂时无法查询订单；已转给专员，专员会通过当前渠道回复 |
| PARTS-ID-01 | 要点 | 列出候选部件的说明书标号和名称；请客户指出是哪一个 |
| PARTS-HW-01 | 要点 | 小五金件有时会贴在 `{packing_location}`；请客户先检查这个位置，如果仍然没有找到，告诉我们即可 |
| PARTS-CH-01 | 原文输出 | Because this order was placed on {channel}, parts requests need to be handled through {channel}. Our team will follow up with you there. |
| PARTS-OOW-01 | 原文输出 | Under our {policy_name} ({policy_link}), missing or damaged parts need to be reported within {claim_window_days} days of delivery. This item was delivered on {delivered_date}, so we're unable to send replacement parts under this policy. |
| PARTS-DUP-01 | 要点 | 这个部件已经有一个补发申请，编号 `{request_id}`，当前状态 `{status}`；不需要重新提交 |
| PARTS-PHOTO-01 | 要点 | 需要照片确认补发的部件；通过链接上传：外箱及物流面单、问题部件特写、问题部件与说明书部件页同框 |
| PARTS-PHOTO-02 | 要点 | 按 `{retake_guidance}` 说明需要补拍的具体内容；不评价照片好坏 |
| PARTS-PHOTO-03 | 要点 | 照片用于确认要补发的具体部件和数量，确保寄出的部件正确 |
| PARTS-EVD-01 | 要点 | 照片中 `{label}` 部件显示完整；请客户对照说明书第 `{manual_page}` 页再确认一次 |
| PARTS-BO-01 | 要点 | 该部件暂时缺货，现在无法确定发货时间；请客户选择等待补货或改为补偿方案 |
| PARTS-NS-01 | 要点 | 该部件无法单独寄出；请客户选择整件退货或补偿方案 |
| PARTS-CFM-01 | 原文输出 | Here's what we'll request for order {order_id} ({product_name}):<br>{parts_list}<br>Ship to: {city}, {postcode} (your original delivery address)<br>Cost to you: {reship_fee_display}<br>{excluded_parts_section}<br>Shipping details will be confirmed after the request is created.<br>Please reply "yes" to confirm. |
| PARTS-SAFE-01 | 原文输出 | For your safety, please don't use the {product_name} until the replacement part has arrived and been installed. |
| PARTS-OK-01 | 原文输出 | Your replacement parts request has been submitted. Request number: {request_id}. {estimated_ship_sentence}We'll email you tracking details once the parts ship. |
| PARTS-OK-02 | 原文输出 | Your replacement parts request has been submitted and is being reviewed. Request number: {request_id}. We'll email you once the review is complete. |
| PARTS-ESC-01 | 要点 | 已转给专员；专员会通过当前渠道回复；不承诺回复时间 |
| PARTS-ESC-02 | 原文输出 | I've passed your request to a specialist for review. They'll get back to you through this channel. |
| PARTS-ESC-SAFE | 原文输出 | I'm sorry to hear this. Please stop using the product and keep it away from people and pets. I've escalated this to our product safety team as a priority, and they will contact you directly. |
