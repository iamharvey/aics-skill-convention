# 回归用例集 RS-PARTS-001

判定方式：

- D = 确定性断言（路径、工具调用及参数、终止状态、reason_code）
- H = 人工评审（对客表达）
- 所有用例都必须通过 D 断言；标注 H 的用例另外需要人工评审通过

| case_id | step_ref | 场景 | 关键前置数据 | 期望路径 | 期望终止状态 / reason_code | 对客检查点 | 判定 |
|---|---|---|---|---|---|---|---|
| PT-001 | S1.not_found | 订单号错误两次 | get_order 两次返回空 | S1 → E4 | escalated | 只询问 1 次 | D |
| PT-002 | S1.owner_mismatch | 订单不属于当前客户 | owner ≠ 已验证客户 | S1 | handed_off → identity-verification | 不出现任何订单信息 | D |
| PT-003 | S1.none_delivered | 所有包裹未签收 | 全部 in_transit | S1 | handed_off → logistics-tracking | — | D |
| PT-004 | S1.tool_fail | get_order 两次超时 | — | S1 → E4 | escalated | 不推测订单状态 | D |
| PT-005 | S2.ambiguous | 客户说“侧板”，BOM 中有左右两块 | 2 个候选 | S2 询问 → 客户选定 → S3 | — | 列出标号和名称，不出现 part_no | D+H |
| PT-006 | S2.not_in_bom | 客户描述的部件不在 BOM 中 | — | S2 → E3 | escalated | 只询问 1 次 | D |
| PT-007 | S2.qty_over_bom | 客户说少了 20 个，BOM qty=16 | — | 按 16 记录 | — | 清单说明数量依据 | D |
| PT-008 | S2.hw_location | 五金件缺失 | packing_location 有值 | 提示 1 次后继续 | — | 只提示 1 次 | D+H |
| PT-009 | S2.cosmetic | 客户只报告表面划痕 | — | S2 | handed_off → compensation-request | — | D |
| PT-010 | S2.bom_empty | BOM 返回为空 | — | S2 → E3 | escalated | — | D |
| PT-011 | S3.platform | Amazon 渠道，platform_managed | — | S3 | handed_off → channel-ops | 话术 PARTS-CH-01 原文 | D |
| PT-012 | S3.pkg_in_transit | 缺失部件在未签收的包裹 3 中 | package_no=3 in_transit | S3 | handed_off → logistics-tracking | 告知在第 3 个包裹中 | D |
| PT-013 | S3.window_before | 报告日 = 签收日 + 时效 − 1 | — | S3 → S4 | — | — | D |
| PT-014 | S3.window_equal | 报告日 = 签收日 + 时效 | — | S3 → S4 | — | — | D |
| PT-015 | S3.window_after | 报告日 = 签收日 + 时效 + 1，非安全件 | — | S3 | rejected_by_policy / PARTS_OUT_OF_WINDOW | 话术 PARTS-OOW-01 原文 | D |
| PT-016 | S3.oow_safety | 超出时效，安全关键部件 | safety_critical=true | S3 → E2 | escalated | 不出现拒绝表述 | D |
| PT-017 | S3.partial_oow | 2 个部件，1 个超出时效 | — | 只对 in_window 部件继续 | completed | 清单列出不在本次范围的部件及原因 | D+H |
| PT-018 | S3.timezone | 签收在 UTC 当天、站点当地时间前一天 | site=US | 按当地自然日计算 | — | — | D |
| PT-019 | S3.earlier_report | 客户 3 周前首次报告，今天再次联系 | get_claims 有早期 reported_at | 报告日取早期时间 | — | — | D |
| PT-020 | S3.delivered_no_date | 状态已签收但 delivered_at 为空 | — | S3 | handed_off → logistics-tracking | — | D |
| PT-021 | S3.policy_fail | 政策参数读取失败 | — | S3 → E5 | escalated | — | D |
| PT-022 | S4.dup_open | 同一部件已有 submitted 申请 | — | S4 | completed / PARTS_DUPLICATE_OPEN | 告知已有申请编号 | D |
| PT-023 | S4.repeat_shipped | 同一部件已补发过，再次报告 | status=shipped | S4 → E3 | escalated | — | D |
| PT-024 | S4.risk | risk_flag=review_required | — | S4 → E6 | escalated | 不提及风险审查或理赔次数 | D+H |
| PT-025 | S4.tool_fail | get_claims 两次失败 | — | S4 → E4 | escalated | 不跳过本步骤 | D |
| PT-026 | S5.waiver | 五金件缺失，免照片 | not_required | S5 → S6 | — | 不要求客户上传照片 | D |
| PT-027 | S5.broken_ok | 板件破损，评估确认 | status=broken | S5 → S6 | — | — | D |
| PT-028 | S5.missing_count | 客户报告缺件，评估 detected_qty < qty | detected=14, qty=16 | reship_qty=2 | — | — | D |
| PT-029 | S5.retake_ok | 首次照片模糊，补拍后确认 | insufficient → broken | S5 补拍 1 次 → S6 | — | 只说明补拍内容，不评价照片 | D+H |
| PT-030 | S5.retake_fail | 补拍后仍被遮挡 | occluded × 2 | S5 → E3 | escalated | — | D |
| PT-031 | S5.intact_dispute | 评估为 intact，客户坚持 | status=intact | 确认 1 次 → E3 | escalated | 不出现拒绝表述 | D |
| PT-032 | S5.edited | 照片疑似修图 | suspected_edited | S5 → E3 | escalated | 不告知原因 | D+H |
| PT-033 | S5.no_upload | 发送上传链接后未上传 | — | S5 | abandoned / PARTS_AWAITING_PHOTO | — | D |
| PT-034 | S5.refuse | 两次拒绝提供照片 | — | S5 | abandoned / PARTS_PHOTO_REFUSED | 说明照片用途 | D |
| PT-035 | S6.not_separable | 部件不可单独补发 | DT-S6 R01 | 客户选整件退货 | handed_off → return-refund-eligibility | 一次只给两个选项 | D+H |
| PT-036 | S6.backorder_wait | 缺货，客户选择等待 | DT-S6 R04 | S6 → S7 → S8 | completed | 申请带 backorder=true；不承诺时间 | D |
| PT-037 | S6.backorder_comp | 缺货，客户选择补偿 | DT-S6 R04 | S6 | handed_off → compensation-request | — | D |
| PT-038 | S6.discontinued | 部件停产 | DT-S6 R03 | 选项：整件退货 / 补偿 | handed_off | — | D |
| PT-039 | S6.mixed | 1 个可补发，1 个不可补发，客户选整件退货 | — | 整单转交，不创建补发申请 | handed_off → return-refund-eligibility | — | D |
| PT-040 | S6.avail_fail | 库存查询失败 | — | 按 backorder 告知 | completed | 申请带 availability_unknown=true | D |
| PT-041 | S7.confirm | 客户答复 “ja” | — | S7 → S8 | completed | 清单含遮蔽地址和费用 | D+H |
| PT-042 | S7.vague | 客户答复 “I guess” | — | 重新确认 | — | 未调用 create_parts_reshipment_request | D |
| PT-043 | S7.modify | 客户要求多加一个部件 | — | 回退 S2 | — | — | D |
| PT-044 | S7.backtrack_limit | 第 3 次要求修改 | — | S7 → E4 | escalated | — | D |
| PT-045 | S7.new_address | 客户要求补发到办公室 | — | S7 | handed_off → shipping-address-change | 未调用创建申请工具 | D |
| PT-046 | S7.safety_note | 涉及安全关键部件 | safety_critical=true | S7 | — | 清单末尾含 PARTS-SAFE-01 原文 | D |
| PT-047 | S8.submitted | 创建成功，无预计时间 | estimated_ship_date=null | S8 | completed / PARTS_RESHIP_SUBMITTED | 不出现发货或到货时间 | D |
| PT-048 | S8.pending | 返回 pending_review | — | S8 | completed / PARTS_RESHIP_PENDING | 不出现“approved” | D |
| PT-049 | S8.timeout_created | 创建超时，查询发现已创建 | get_reshipment_request 有记录 | 不重试 | completed | — | D |
| PT-050 | S8.timeout_not_created | 创建超时，查询未创建，重试成功 | — | 重试 1 次 | completed | create 调用次数 = 2，idempotency_key 相同 | D |
| PT-051 | S8.rejected | 返回 rejected | — | S8 → E4 | escalated | 不自行解释拒绝原因 | D |
| PT-052 | E1.injury | S5 中客户提到组装时被玻璃划伤 | — | 立即 E1 | escalated | 话术 PARTS-ESC-SAFE 原文；未继续补发流程 | D+H |
| PT-053 | E7.anger | 客户第 2 次表达不满 | — | E7 | escalated | — | D |
| PT-054 | E7.chargeback | 客户提到要申请信用卡拒付 | — | E7 | escalated | — | D |
| PT-055 | lang | 法国站客户用英语沟通 | site=FR | — | — | 用英语回复，话术取 en 版本 | H |
