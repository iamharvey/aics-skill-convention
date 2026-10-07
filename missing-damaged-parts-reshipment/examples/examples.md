# 示例

说明：

- 订单号、SKU、部件号、客户信息都是虚构的。
- 示例中的政策参数值只用于演示，实际值以政策参数库为准。
- `→ 工具` 表示工具调用，`← 返回` 表示工具返回的关键字段。

---

## P1 正例：五金件缺失，按政策免照片

场景：美国站自营订单，客户组装书柜时发现 B 号螺丝少了 4 个。

```text
客户: Hi, I'm putting together the bookcase and I'm missing 4 of the B screws.
      Order 7701234.

→ get_order(7701234)
← site=US, channel=own-site, sku=BK-3T-WH, bom_version=v2, 收货地址 Austin, TX 78701
→ get_shipments(7701234)
← package 1/1, delivered, delivered_at=2026-10-02
→ get_product_bom(BK-3T-WH, v2)
← part_no=HW-B-M6x30, label=B, part_class=hardware, qty=16,
  packing_location="taped to the back of panel D", separately_shippable=true

[S2] 唯一对应到 HW-B-M6x30；hardware + missing → 提示检查 packing_location

Agent:  Sorry about that. Small hardware is sometimes taped to the back of
        panel D. Could you check there? If they're not there, just let me know.
客户:  Already checked, nothing there.

[S3] 报告日 2026-10-07，签收日 2026-10-02，在时效内 → in_window
→ get_claims(7701234) ← 无记录
→ get_customer_claim_stats(C-55821) ← risk_flag=none
[S5] photo_requirement[hardware][missing] = not_required → evidence_basis=policy_waiver
→ get_part_availability(HW-B-M6x30, US) ← available
[S6] DT-S6 R05 → reshippable

Agent:  Here's what we'll request for order 7701234 (3-Tier Bookcase, White):
        - Part B, M6×30 screw × 4 (missing)
        Ship to: Austin, 78701 (your original delivery address)
        Cost to you: Free
        Shipping details will be confirmed after the request is created.
        Please reply "yes" to confirm.
客户:  yes

→ create_parts_reshipment_request(order_id=7701234, sku=BK-3T-WH, bom_version=v2,
    parts=[{part_no: HW-B-M6x30, qty: 4, claim_type: missing,
            evidence_basis: policy_waiver}],
    ship_to=addr_88123, idempotency_key=sess_9f2a:2026-10-07T10:14:22)
← status=submitted, request_id=RS-240031, estimated_ship_date=null

Agent:  Your replacement parts request has been submitted. Request number:
        RS-240031. We'll email you tracking details once the parts ship.

终止状态: completed / PARTS_RESHIP_SUBMITTED
```

要点：工具没有返回 `estimated_ship_date`，Agent 就不提发货时间。

---

## P2 正例：板件破损，照片评估确认后补发

场景：德国站自营订单，客户开箱发现户外储物柜的 C 号侧板开裂。C 号侧板是安全关键部件。

```text
客户: Hallo, das Seitenteil C ist beim Auspacken gebrochen. Bestellung 8804410.

→ get_order / get_shipments / get_product_bom
← site=DE, channel=own-site, sku=OS-SHED-GY, 2 个包裹均已签收
← part_no=PNL-C-L, label=C, part_class=panel, qty=1, package_no=2,
  safety_critical=true, separately_shippable=true

[S3] 在时效内 → in_window
[S4] 无历史记录，risk_flag=none
[S5] photo_requirement[panel][broken] = required
→ send_upload_link(8804410)

Agent:  (德语) 需要照片确认补发的部件；请通过链接上传外箱及物流面单、
        C 号侧板裂缝特写、C 号侧板与说明书部件页同框。

客户上传 3 张照片
→ assess_damage_images(images, OS-SHED-GY, v2, [PNL-C-L])
← PNL-C-L: status=broken, image_quality=ok, authenticity_flag=none
[S5] broken + broken → confirmed

→ get_part_availability(PNL-C-L, DE) ← available
[S6] DT-S6 R05 → reshippable

Agent:  (德语确认清单，含 C 号侧板 × 1、破损、Berlin 10115、免费)
        (话术 PARTS-SAFE-01) Bitte verwenden Sie den Geräteschrank aus
        Sicherheitsgründen erst, wenn das Ersatzteil eingetroffen und
        montiert ist.
客户:  Ja, bitte.

→ create_parts_reshipment_request(...)
← status=pending_review, request_id=RS-240077

Agent:  (德语，话术 PARTS-OK-02) 申请已提交，正在审核，编号 RS-240077

终止状态: completed / PARTS_RESHIP_PENDING
```

要点：工具返回 `pending_review`，Agent 只说“正在审核”，不说“已批准”。

---

## H1 转交例：缺失部件在未签收的包裹中

场景：庭院餐桌椅分 3 个包裹发货，客户说少了桌腿。

```text
客户: I got the patio set but the table legs are missing. Order 7712090.

→ get_shipments(7712090)
← package 1/3 delivered, package 2/3 delivered, package 3/3 in_transit
→ get_product_bom(...)
← part_no=LEG-T-01, label=F, name=table leg, qty=4, package_no=3

[S3] claim_type=missing，所在包裹 3 未签收 → 转 logistics-tracking

Agent:  Your order ships in 3 boxes. The table legs are in box 3, which is
        still on its way. Let me check where it is now.

终止状态: handed_off → logistics-tracking
```

要点：不创建补发申请。缺件的原因是包裹还没到，补发会导致重复发货。

---

## H2 拒绝例：所有部件超出报告时效

场景：英国站订单，客户在签收后很久才报告缺件，且不涉及安全关键部件。

```text
[S3] claim_window_days[UK][own-site] 读取成功；报告日超出时效；
     部件 safety_critical=false → out_of_window
     所有部件都是 out_of_window → 拒绝

Agent:  (话术 PARTS-OOW-01，原文输出，替换政策名称、链接、天数、签收日)

终止状态: rejected_by_policy / PARTS_OUT_OF_WINDOW
```

要点：如果其中有安全关键部件，就不拒绝，而是升级 E2。

---

## N1 反例：模型推翻图像评估结果

```text
← assess_damage_images: PNL-A-TOP status=intact, image_quality=ok
客户: It's clearly cracked, look closer!

错误做法:
Agent:  You're right, I can see a crack near the corner. I'll send a new one.
        （模型自行判定破损，并进入补发流程）

正确做法:
Agent:  (话术 PARTS-EVD-01) 照片中 A 号顶板显示完整；请对照说明书第 4 页再确认一次。
客户:  I'm sure it's cracked.
→ 升级 E3（damage-assessment-manual），附原始照片和评估结果
```

违反：硬性约束第 2 条；S5 的判定分支。

---

## N2 反例：按客户要求补发到新地址

```text
客户: Can you send the part to my office instead? 220 Main St, Denver.

错误做法:
→ create_parts_reshipment_request(..., ship_to="220 Main St, Denver")

正确做法:
Agent:  Replacement parts can only be sent to the original delivery address.
        I can help you request an address change first.
→ 转 shipping-address-change，状态 handed_off
```

违反：硬性约束第 3 条；S7 的判定分支。

---

## N3 反例：把含糊答复当作确认

```text
Agent:  (确认清单) Please reply "yes" to confirm.
客户:  I guess, but can you also add one more screw just in case?

错误做法:
→ create_parts_reshipment_request(...)  // 按原清单创建

正确做法:
回答含糊，且附带修改 → 不算确认。
多加一颗螺丝会使数量超过 BOM（DT-S6 R06）→ 说明补发数量以说明书为准，
更新清单后重新请求确认。
```

违反：S7 对“明确确认”的定义；第 8 章 R8.2。
