# Case 01 - Support Ticket Triage — Workbook Answers

---

## 1. Unit of Work

**Lát cắt được chọn:** Một ticket đi vào hệ thống → AI đọc nội dung, tiêu đề, và loại khách hàng → sinh ra bộ nhãn triage gồm category, urgency, route, requires_human flag, và reason summary.

**Vì sao đây là đơn vị đủ nhỏ để eval:** Mỗi ticket là một giao dịch độc lập, có input rõ ràng (ticket_id, subject, message, customer_tier) và output xác định (tập nhãn có thể đối chiếu với ground truth). Không cần chờ cả pipeline hỗ trợ — chỉ cần 1 ticket là có thể chạy 1 eval. Nếu sai ở lát cắt này, hậu quả vận hành ngay lập tức: ticket đến sai team, trễ xử lý, hoặc bỏ sót escalation cần thiết cho khách doanh nghiệp.

---

## 2. Quality Question

**Câu hỏi chất lượng:** AI có gán đúng route và escalation flag để ticket không bị xử lý sai hàng hoặc bỏ sót trường hợp cần người thật nhảy vào không — đặc biệt với khách enterprise có dấu hiệu "blocking work"?

**Vì sao đây là câu hỏi đúng:** Nếu AI fail ở đây, hậu quả không phải là "câu trả lời nghe chưa hay" mà là ticket đi sai team (billing đến product_team), khách enterprise bị xếp hàng thường thay vì ưu tiên, hoặc cờ `requires_human=false` khiến không ai nhảy vào kịp thời. Đây là những failure gây mất SLA và mất trust khách hàng — không phải vấn đề aesthetic.

---

## 3. Output Contract tối thiểu

```json
{
  "ticket_id": "string",
  "category": "Technical | Billing | Feature Request | Unknown",
  "urgency": "low | medium | high | critical",
  "requires_human": "boolean",
  "route_to": "technical_support | billing_ops | product_team | human_escalation",
  "reason_summary": "string",
  "confidence": "float [0.0 - 1.0]"
}
```

**Giải thích từng field:**

- `ticket_id` — để nối eval result với input gốc, bắt buộc cho tracing và debugging.
- `category` — quyết định routing logic; field quan trọng nhất, drive hầu hết business rules.
- `urgency` — xác định hàng ưu tiên; kết hợp với `customer_tier` để trigger escalation.
- `requires_human` — cờ hành động trực tiếp; nếu sai thì không có người nhảy vào dù ticket cần.
- `route_to` — xác định team nhận ticket; sai team = ticket nằm sai hàng đợi.
- `reason_summary` — bằng chứng để human agent review nhanh và để LLM judge kiểm tra grounding; không được bịa thông tin.
- `confidence` — cho phép triage thêm bước human review khi AI không chắc; cần để đặt threshold gate.

**Không đưa vào contract:** tên khách, nội dung ticket gốc (đã có trong input), timestamp — những field này không thay đổi routing hay eval logic.

---

## 4. Eval Decision Map

| Thành phần cần chấm | Code | LLM | Human | Expert | Lý do |
|---|:---:|:---:|:---:|:---:|---|
| Schema hợp lệ (đúng field, đúng kiểu dữ liệu) | ✓ | | | | Deterministic — kiểm tra bằng schema validator, không cần hiểu ngữ nghĩa |
| Category nằm trong allowed enum | ✓ | | | | Tập giá trị cố định, check bằng set membership |
| confidence trong khoảng [0, 1] | ✓ | | | | Numeric bound check, hoàn toàn deterministic |
| Rule: enterprise + (high/critical) → requires_human=true | ✓ | | | | Business rule tường minh, viết thành if-else được |
| Rule: category=Billing → route_to ≠ product_team | ✓ | | | | Routing constraint cụ thể, không cần hiểu ngữ nghĩa |
| Rule: "blocking"/"locked out"/"account disabled" → urgency ≠ low | ✓ | | | | Keyword rule đủ rõ để code bắt |
| reason_summary không rỗng | ✓ | | | | Kiểm tra độ dài chuỗi |
| Category có khớp nội dung ticket không | | ✓ | | | Cần đọc hiểu ngữ nghĩa ticket — "thanh toán lỗi" → Billing không thể check bằng regex |
| Urgency có hợp lý với ngữ cảnh đầy đủ không | | ✓ | | | "URGENT" trong subject không tự động là critical; cần đọc hiểu cả message |
| reason_summary có grounded trong nội dung ticket không | | ✓ | | | Phát hiện hallucination — AI có bịa thêm thông tin không có trong input |
| Ticket mơ hồ (low-info) được xử lý thích hợp | | | ✓ | | Cần ops agent phán xét "Unknown với confidence thấp" có đúng không |
| Calibration threshold có hợp lý không | | | ✓ | | Support ops team xác nhận ngưỡng 90% / 0 lỗi escalation trước khi ship |

**Không cần domain expert:** Ticket triage là nghiệp vụ vận hành — support ops team (L1/L2) biết rõ taxonomy, escalation policy, và SLA hơn bất kỳ chuyên gia domain nào. Không có kiến thức chuyên môn đặc thù (y tế, pháp lý...) cần xác nhận riêng.

---

## 5. Kiểm tra tự động bằng code

- **Kiểm tra: `category` phải nằm trong `{Technical, Billing, Feature Request, Unknown}`**
  Vì sao nên giao cho code: Tập giá trị cố định, so sánh string đơn giản. Nếu AI hallucinate ra "Payment Issue" hay "Login Problem" thay vì enum chuẩn thì downstream system vỡ.

- **Kiểm tra: `urgency` phải nằm trong `{low, medium, high, critical}`**
  Vì sao nên giao cho code: Tương tự — enum chuẩn, set membership check.

- **Kiểm tra: `route_to` phải nằm trong `{technical_support, billing_ops, product_team, human_escalation}`**
  Vì sao nên giao cho code: Route sai enum = ticket không vào được hàng đợi nào.

- **Kiểm tra: `confidence` phải là float trong [0.0, 1.0]**
  Vì sao nên giao cho code: Numeric bound check thuần túy.

- **Kiểm tra: `requires_human` phải là boolean (true/false)**
  Vì sao nên giao cho code: Type check — string "true" hay số 1 đều không hợp lệ.

- **Kiểm tra: Nếu `customer_tier=enterprise` AND `urgency` trong `{high, critical}` → `requires_human` PHẢI là `true`**
  Vì sao nên giao cho code: Business rule tường minh từ đề bài. Đây là rule không thể thương lượng — sai là escalation bị bỏ sót. Viết được bằng if-else.

- **Kiểm tra: Nếu `category=Billing` → `route_to` KHÔNG được là `product_team`**
  Vì sao nên giao cho code: Routing constraint explicit, không cần ngữ nghĩa.

- **Kiểm tra: Nếu message chứa "blocking", "locked out", hoặc "account disabled" → `urgency` KHÔNG được là `low`**
  Vì sao nên giao cho code: Keyword matching đủ rõ. Những tín hiệu này là business-defined red flags.

- **Kiểm tra: `reason_summary` không được rỗng hoặc dưới 10 ký tự**
  Vì sao nên giao cho code: Nếu rỗng, human agent không có gì để review. Độ dài tối thiểu là proxy kiểm tra AI có thực sự điền.

- **Kiểm tra: Tất cả ticket ID được nhắc trong `reason_summary` phải khớp với `ticket_id` trong input**
  Vì sao nên giao cho code: Phát hiện fabrication đơn giản — AI không được tự bịa ra ticket ID khác trong lý do.

---

## 6. Tiêu chí chấm bằng LLM

- **Tiêu chí: Category có phản ánh đúng bản chất vấn đề trong ticket không?**
  Vì sao code không bắt tốt: Ticket "password reset không vào được" có thể là Technical hoặc Billing (nếu account bị khóa vì thanh toán). Cần đọc hiểu ngữ cảnh để phân biệt — keyword match sẽ không đủ với các edge case.

- **Tiêu chí: Urgency có phù hợp với mức độ impact thực tế được mô tả trong message không?**
  Vì sao code không bắt tốt: "URGENT" trong subject không tự động là critical. Ticket "gấp" từ SME có thể chỉ là medium, trong khi ticket không có từ "urgent" nhưng mô tả "toàn bộ team bị block" thì là high. Cần đọc hiểu ngữ cảnh tổng thể.

- **Tiêu chí: `reason_summary` có grounded — chỉ dựa trên thông tin có trong ticket, không bịa thêm?**
  Vì sao code không bắt tốt: Phát hiện hallucination phức tạp hơn string matching. AI có thể suy luận "khách đã bị charge hai lần" dù ticket không đề cập — code không biết đó là suy luận hay fact.

- **Tiêu chí: Với ticket mơ hồ (low-info như "Help, please help asap"), AI có đặt Unknown/confidence thấp thay vì gán category tự tin không?**
  Vì sao code không bắt tốt: Code không biết ticket có đủ thông tin để phân loại không. Cần hiểu mức độ ambiguity của input.

---

## 7. Human / Expert Review

**Ai cần review:** Support ops team — cụ thể là L1/L2 agents có kinh nghiệm xử lý ticket hàng ngày.

**Review những case nào:**
- Ticket có `confidence < 0.6` — AI không chắc, cần người xác nhận.
- Ticket được gán `category = Unknown` — cần kiểm tra AI có đúng khi bỏ không gán không.
- 20% sample ngẫu nhiên sau mỗi prompt change để phát hiện regression không bị code eval bắt.
- Tất cả case fail 2+ LLM judge criteria — cụm failure thường là failure mode cụ thể cần fix.

**Có cần domain expert không:** Không. Ticket triage là quy trình vận hành nội bộ — không có kiến thức chuyên môn đặc thù nào vượt quá năng lực của support ops team. Taxonomy (Technical/Billing/Feature Request) và escalation rules đều do ops team và PM định nghĩa, không cần bác sĩ, luật sư, hay chuyên gia ngành. Human review từ ops team là đủ.

*Không áp dụng: 7A và 7B.*

---

## 8. Release Gate

**Điều kiện để ship:**

1. **Code checks: 100% pass** — bất kỳ violation nào về schema, enum, business rules (enterprise+critical→requires_human, billing→không vào product_team) là hard block. Không thương lượng.

2. **LLM judge: ≥90% category accuracy** trên calibration set 75 cases — đo bằng precision/recall theo từng category, không chỉ overall accuracy.

3. **LLM judge: 0 trường hợp** enterprise + urgency=high/critical nhưng requires_human=false lọt qua — đây là lỗi nghiêm trọng nhất, threshold là zero.

4. **Latency: <2s** per ticket — triage chậm làm toàn bộ hàng đợi bị nghẽn.

5. **Human review: ≥85% đồng thuận** với ops team trên 20 case sampled (ưu tiên edge cases: low-info, ambiguous, high-risk).

**Điều kiện cần human review trước khi ship:**
- Pass rate của LLM judge nằm trong 85–90% — gần ngưỡng, cần ops team xác nhận trước khi chấp nhận.
- Có bất kỳ failure mode mới nào xuất hiện chưa từng thấy trong seed cases.

---

## 5 Dataset Edge Cases

1. **Happy path — match rõ, enterprise, billing critical:**
   Subject: "URGENT: payment failed and account locked". Customer_tier: enterprise.
   Kỳ vọng: category=Billing, urgency=critical, requires_human=true, route=billing_ops.
   *Dùng để bắt: AI không gán đúng category Billing hoặc quên set requires_human khi đủ điều kiện.*

2. **Ambiguous input — thiếu ngữ cảnh:**
   Subject: "Help". Message: "Please help asap". Customer_tier: standard.
   Kỳ vọng: category=Unknown, confidence thấp (< 0.4), không được tự gán urgency=high chỉ vì "asap".
   *Dùng để bắt: AI overconfident với input rỗng — failure mode phổ biến nhất.*

3. **Missing information — subject misleading:**
   Subject: "Question about billing". Message: "How do I export my data to CSV?"
   Kỳ vọng: AI phải đọc message, không phải subject — category=Feature Request, không phải Billing.
   *Dùng để bắt: AI shortcut bằng keyword trên subject mà không đọc message.*

4. **High-risk / escalation — non-enterprise nhưng có blocking signal:**
   Subject: "Account completely locked, cannot work". Customer_tier: standard (không phải enterprise).
   Kỳ vọng: urgency=high (vì "cannot work" + "locked"), nhưng requires_human có thể là false theo business rule (rule chỉ apply với enterprise). Kiểm tra AI không confuse business rule.
   *Dùng để bắt: AI áp dụng sai requires_human rule cho non-enterprise.*

5. **Regression case — ticket từng bị classify sai sau prompt update:**
   Subject: "API rate limit exceeded". Message: "We're hitting rate limits on your API, blocking our production system."
   Kỳ vọng: category=Technical, urgency=high (production blocked), route=technical_support.
   *Dùng để bắt: Sau mỗi prompt change, case này không được tụt về medium hay route sang product_team.*

---

## 9. Kế hoạch chạy thử và dự toán chi phí

**Giả định model sử dụng:** `gpt-4o-mini`
Giá API thật từ OpenAI pricing page (tháng 6/2026):
- Input: $0.15 / 1M tokens
- Output: $0.60 / 1M tokens

**Quy mô pilot:**
- 75 cases (3 seed × 5 biến thể + 5 edge cases × 5 + 20 random real tickets)
- 40 lần chạy / lặp lại (10 lần baseline + 30 lần iterate prompt judge)

**Chi phí API (LLM judge):**
- Mỗi lần judge 1 case: ~600 input tokens (ticket + rubric) + 300 output tokens
- Chi phí/case/run: (600 × 0.15 + 300 × 0.60) / 1,000,000 = $0.00027
- 75 cases × 40 runs = 3,000 lần chạy
- Tổng API cost: 3,000 × $0.00027 ≈ **$0.81**

**Giờ công người:**
- PM / thiết kế eval: 4h (viết rubric, tạo calibration set, review decision map)
- Ops team / human label: 5h (gắn nhãn 75 cases, review 20 samples sau mỗi prompt iteration)
- Kỹ thuật / setup: 2h (script eval, kết nối API, báo cáo)
- Tổng: **~11h người**

**Tổng chi phí pilot:** <$5 (API cost + không kể giờ công nội bộ)
**Thời gian dự kiến:** 5–7 ngày làm việc

**Kết luận:** Với ~$1 API và ~11h người, pilot đủ để trả lời: AI có đạt ≥90% category accuracy và 0 lỗi escalation không? Nếu pass → đề xuất ship. Nếu fail → có đủ failure log để sửa prompt có hệ thống thay vì đoán mò.
