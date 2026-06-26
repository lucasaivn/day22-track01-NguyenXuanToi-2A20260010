# Case 02 - Sales Chat Copilot — Workbook Answers

---

## 1. Unit of Work

**Lát cắt được chọn:** Một lượt tin nhắn mới từ khách đi vào → AI đọc message + lịch sử hội thoại → phát hiện tín hiệu nhận diện (số điện thoại / email / mã đơn) → tra cứu CRM/OMS → sinh ra summary + gợi ý bước tiếp theo cho nhân viên sales.

**Vì sao đây là đơn vị đủ nhỏ để eval:** Mỗi lượt chat là một vòng xử lý độc lập — có input xác định (tin nhắn, lịch sử, metadata kênh) và output có thể đối chiếu (tín hiệu nào được phát hiện, lookup đúng hay sai, summary có grounded không). Rủi ro vận hành xảy ra ngay tại lượt này: nếu AI match sai hồ sơ hoặc không cảnh báo ambiguity, nhân viên sales có thể trả lời sai khách trong cùng cuộc hội thoại đó — không cần chờ đến bước sau.

---

## 2. Quality Question

**Câu hỏi chất lượng:** Copilot có phát hiện đúng tín hiệu, lookup đúng hồ sơ/đơn hàng, và biết dừng lại (cảnh báo) khi xuất hiện ambiguity hoặc mâu thuẫn giữa các nguồn dữ liệu — thay vì tự tin đẩy thông tin sai cho nhân viên không?

**Vì sao đây là câu hỏi đúng:** Nếu Copilot match sai hồ sơ, nhân viên sales sẽ gọi sai tên, trả lời sai trạng thái đơn, hoặc tệ hơn — lộ thông tin của người khác cho khách hiện tại. Không phải lỗi "câu văn chưa hay" mà là lỗi data integrity và privacy. Nếu Copilot không cảnh báo khi CRM và OMS mâu thuẫn, nhân viên tin vào thông tin sai và mất trust khách. Behavior bắt buộc: cảnh báo ambiguity. Behavior bị cấm: tự chốt một bản ghi khi có nhiều match.

---

## 3. Output Contract tối thiểu

```json
{
  "detected_signals": [
    { "type": "phone | email | order_id | customer_id", "value": "string" }
  ],
  "lookup_result": {
    "status": "found_single | found_multiple | not_found | not_attempted",
    "customer_profile": { "name": "string", "tier": "string" },
    "order": { "order_id": "string", "status": "string", "product": "string" }
  },
  "ambiguity_flag": "boolean",
  "ambiguity_reason": "string | null",
  "conversation_summary": "string",
  "suggested_next_step": "string",
  "draft_reply": "string | null",
  "data_conflict_warning": "boolean"
}
```

**Giải thích từng field:**

- `detected_signals` — core input cho lookup; phải log rõ loại và giá trị để eval kiểm tra AI có detect đúng tín hiệu không.
- `lookup_result.status` — quan trọng nhất: phân biệt "tìm thấy 1 bản ghi" vs "nhiều bản ghi" vs "không tìm thấy". `found_multiple` phải trigger ambiguity_flag.
- `lookup_result.customer_profile` và `order` — thông tin để render UI; chỉ hiển thị khi `status=found_single`.
- `ambiguity_flag` — cờ hành động: khi true, UI phải yêu cầu nhân viên chọn thủ công trước khi tiếp tục.
- `ambiguity_reason` — giải thích tại sao ambiguous; cần cho nhân viên và cho LLM judge eval quality.
- `conversation_summary` — phần AI tóm tắt hội thoại; subject của LLM judge eval.
- `suggested_next_step` — gợi ý hành động cho nhân viên; phải hợp lý và không act vượt quyền.
- `draft_reply` — nullable: chỉ sinh khi có đủ context; nhân viên phải xem lại trước khi gửi.
- `data_conflict_warning` — khi CRM và OMS mâu thuẫn (vd: CRM nói lead mới, OMS có đơn cũ).

**Không đưa vào contract:** toàn bộ lịch sử đơn hàng, raw CRM data, nội dung chat đầy đủ — đã có trong input, không cần lặp lại trong output.

---

## 4. Eval Decision Map

| Thành phần cần chấm | Code | LLM | Human | Expert | Lý do |
|---|:---:|:---:|:---:|:---:|---|
| Schema output hợp lệ (đúng field, đúng kiểu) | ✓ | | | | Deterministic — schema validator |
| detected_signals đúng type enum | ✓ | | | | Tập giá trị cố định: phone/email/order_id/customer_id |
| Khi found_multiple → ambiguity_flag phải true | ✓ | | | | Business rule tường minh, if-else |
| Khi not_found → không được có customer_profile | ✓ | | | | Nếu AI bịa hồ sơ khi không tìm thấy là lỗi nghiêm trọng |
| draft_reply không được gửi khi ambiguity_flag=true | ✓ | | | | Safety constraint: không được act khi chưa chắc danh tính |
| Khi CRM ≠ OMS → data_conflict_warning phải true | ✓ | | | | Logic so sánh hai nguồn, viết code được |
| Signal detection accuracy (phone/email/order_id) | | ✓ | | | Cần đọc hiểu text, đặc biệt số điện thoại viết tắt, email viết hoa/thường |
| conversation_summary có grounded và đủ ý không | | ✓ | | | Cần đánh giá độ đầy đủ và không hallucinate thông tin |
| suggested_next_step có hợp lý và không vượt quyền hạn không | | ✓ | | | "Mời mua thêm" sai thời điểm cần semantic judgment |
| Ambiguous case được xử lý đúng thái độ (không self-confident sai) | | | ✓ | | Sales ops team xác nhận behavior khi input mơ hồ |
| Calibration LLM judge với sales scenario thực tế | | | ✓ | | Cần người đã làm sales/CSKH xác nhận rubric trước khi scale |

**Không cần domain expert:** Case này thuộc ops/sales — không có kiến thức chuyên môn đặc thù. Human review từ sales ops team (người thực tế dùng Copilot) là đủ để calibrate và validate.

---

## 5. Kiểm tra tự động bằng code

- **Kiểm tra: `detected_signals[].type` phải nằm trong `{phone, email, order_id, customer_id}`**
  Vì sao nên giao cho code: Enum cố định, set membership check.

- **Kiểm tra: Khi `lookup_result.status = found_multiple` → `ambiguity_flag` phải là `true`**
  Vì sao nên giao cho code: Business rule tường minh — nhiều bản ghi match = phải cảnh báo. Không cần judgment.

- **Kiểm tra: Khi `lookup_result.status = not_found` → `lookup_result.customer_profile` và `order` phải null/empty**
  Vì sao nên giao cho code: Phát hiện hallucination cơ bản — AI không được bịa hồ sơ khi lookup fail.

- **Kiểm tra: Khi `ambiguity_flag = true` → `draft_reply` phải null**
  Vì sao nên giao cho code: Safety constraint tường minh. Nếu AI vừa cảnh báo ambiguity vừa đề xuất nháp trả lời → contradiction, nhân viên bị mislead.

- **Kiểm tra: `data_conflict_warning` phải true khi CRM.status ≠ OMS.status cho cùng khách**
  Vì sao nên giao cho code: So sánh giá trị hai field, hoàn toàn deterministic.

- **Kiểm tra: Khi input không có tín hiệu nhận diện nào → `lookup_result.status` phải là `not_attempted` (không phải `not_found`)**
  Vì sao nên giao cho code: Phân biệt "tìm không ra" vs "chưa thử tìm" — quan trọng cho UI và eval. Code check `detected_signals` có empty không.

- **Kiểm tra: Order ID trong output phải khớp với order ID từ input signal (không được tự sinh ID mới)**
  Vì sao nên giao cho code: String matching đơn giản — phát hiện fabrication.

- **Kiểm tra: `suggested_next_step` không được chứa các action cấm: "tạo đơn mới", "cập nhật hồ sơ", "gửi tin nhắn cho khách"**
  Vì sao nên giao cho code: Business rule tường minh — AI không được tự hành động. Keyword blocklist đủ để bắt.

---

## 6. Tiêu chí chấm bằng LLM

- **Tiêu chí: AI có phát hiện đúng tất cả tín hiệu nhận diện trong hội thoại không (recall)?**
  Vì sao code không bắt tốt: Số điện thoại có thể viết dạng "số điện thoại em là 09-09-12-34-56" hay "0909.123.456". Email có thể viết hoa/thường lẫn lộn. Regex cứng không cover hết format thực tế của người Việt nhắn.

- **Tiêu chí: `conversation_summary` có grounded — phản ánh đúng nội dung hội thoại, không thêm thông tin không có?**
  Vì sao code không bắt tốt: Hallucination trong summary không phải là vi phạm schema, cần đọc hiểu cả input lẫn output để so sánh.

- **Tiêu chí: `suggested_next_step` có hợp lý với ngữ cảnh hội thoại và trạng thái khách không?**
  Vì sao code không bắt tốt: "Mời mua thêm" ngay khi khách đang complain về đơn chưa giao là sai thời điểm — cần judgment, không thể check bằng rule.

- **Tiêu chí: Với input mơ hồ (Tình huống D: "xử lý case này với, gấp lắm"), AI có gợi ý hỏi thêm thay vì tự đoán không?**
  Vì sao code không bắt tốt: Cần đánh giá AI có "biết mình không biết" — không thể check bằng keyword hay schema.

- **Tiêu chí: `draft_reply` (nếu có) có phù hợp để nhân viên dùng — không lộ thông tin nhạy cảm, không sai trạng thái đơn?**
  Vì sao code không bắt tốt: Privacy leak trong draft reply đòi hỏi đọc hiểu ngữ nghĩa, không phải pattern matching.

---

## 7. Human / Expert Review

**Ai cần review:** Sales ops team — cụ thể là nhân viên CSKH/sales đang thực tế dùng Copilot hàng ngày.

**Review những case nào:**
- Tất cả case có `ambiguity_flag=true` trong pilot — xem AI có cảnh báo đúng không.
- Case `found_multiple` — xem UI có đủ để nhân viên chọn đúng không.
- Case có `data_conflict_warning=true` — xem mâu thuẫn được trình bày rõ không.
- 20% sample ngẫu nhiên để phát hiện pattern failure không bị code hay LLM judge bắt.
- Mọi case mà `draft_reply` được sinh ra — vì đây là nội dung sẽ đến tay khách.

**Có cần domain expert không:** Không áp dụng. Case này là quy trình bán hàng và CSKH — không có kiến thức chuyên ngành nào vượt quá năng lực của sales ops team. CRM taxonomy, quy trình xử lý đơn, và ranh giới quyền hạn AI đều là policy nội bộ do team tự định nghĩa và review được.

---

## 8. Release Gate

**Điều kiện để ship:**

1. **Code checks: 100% pass** — ambiguity flag, null safety khi not_found, draft_reply bị block khi ambiguous: không có exception.

2. **LLM judge: ≥92% signal detection recall** — Copilot bỏ sót tín hiệu nhận diện nguy hiểm hơn phân loại sai category. Ngưỡng cao hơn Case 01.

3. **0 trường hợp** AI tự chốt một bản ghi khi `found_multiple` — đây là lỗi privacy nghiêm trọng nhất.

4. **0 trường hợp** AI bịa hồ sơ hoặc đơn hàng khi `not_found` — fabrication là hard block.

5. **Human review: ≥88% đồng thuận** với sales ops team trên 25 sampled cases, bao gồm ít nhất 5 ambiguous case.

**Điều kiện cần human review bổ sung trước ship:**
- Nếu có bất kỳ case nào `draft_reply` được sinh khi context chưa đủ chắc chắn.
- Nếu `data_conflict_warning` không xuất hiện khi CRM và OMS rõ ràng mâu thuẫn.

---

## 5 Dataset Edge Cases

1. **Happy path — match rõ, một bản ghi duy nhất:**
   Khách gửi số điện thoại đúng format, CRM trả về 1 hồ sơ, OMS có 1 đơn gần nhất.
   *Dùng để bắt: Copilot không chạy đúng flow cơ bản — display đúng thông tin, suggest đúng bước tiếp.*

2. **Ambiguous lookup — một số điện thoại khớp nhiều hồ sơ:**
   Số 0901234567 gắn với 2 tài khoản do nhập trùng.
   *Dùng để bắt: AI tự chốn một bản ghi thay vì set ambiguity_flag=true và yêu cầu nhân viên chọn.*

3. **Missing information — khách hỏi mơ hồ không có tín hiệu:**
   "Chị ơi xử lý case này với, gấp lắm." — không có số, email, hay mã đơn.
   *Dùng để bắt: AI không được tự tra cứu mà phải gợi ý hỏi thêm; không được để lookup_status=not_found khi chưa thử.*

4. **Conflicting systems — CRM và OMS mâu thuẫn:**
   CRM nói khách là "lead mới chưa mua". OMS có lịch sử 3 đơn đã giao.
   *Dùng để bắt: data_conflict_warning phải true; AI không được tóm tắt như thể mọi thứ chắc chắn.*

5. **Regression case — mã đơn thuộc người khác:**
   Khách gửi mã đơn DH-48291 nhưng đơn đó thuộc customer_id khác.
   *Dùng để bắt: AI không được suy luận "đây chắc là đơn của người đang chat" — phải cảnh báo không match.*

---

## 9. Kế hoạch chạy thử và dự toán chi phí

**Model sử dụng:** `gpt-4o-mini`
Giá API thật từ OpenAI (tháng 6/2026):
- Input: $0.15 / 1M tokens
- Output: $0.60 / 1M tokens

**Quy mô pilot:**
- 80 cases (5 tình huống mẫu × 6 biến thể + 5 edge cases × 5 + 15 real chat logs từ ops team)
- 40 lần chạy / lặp lại (10 baseline + 30 iterate judge prompt theo failure patterns)

**Chi phí API (LLM judge):**
- Mỗi lần judge 1 case: ~800 input tokens (hội thoại + rubric dài hơn Case 01) + 350 output tokens
- Chi phí/case/run: (800 × 0.15 + 350 × 0.60) / 1,000,000 = $0.00033
- 80 cases × 40 runs = 3,200 lần chạy
- Tổng API cost: 3,200 × $0.00033 ≈ **$1.06**

**Giờ công người:**
- PM / thiết kế eval: 5h (rubric phức tạp hơn — nhiều tiêu chí liên quan đến privacy và ambiguity)
- Sales ops team / human label: 7h (label 80 cases, review ambiguous và conflict cases kỹ hơn)
- Kỹ thuật / setup + CRM mock: 3h (cần mock CRM/OMS data cho pilot)
- Tổng: **~15h người**

**Tổng chi phí pilot:** ~$2–3 (API) + giờ nội bộ
**Thời gian dự kiến:** 6–8 ngày làm việc

Giá API lấy từ platform.openai.com/pricing. Với 80 cases và 40 lần chạy, tổng API cost dưới $2 — rất nhỏ so với rủi ro nếu Copilot match sai hồ sơ trên production. Plan này đủ để trả lời: AI có biết dừng đúng chỗ khi ambiguous không, và có 0 case tự chốt sai hồ sơ không? Hai câu đó đủ để quyết định pilot thật với 5–10 nhân viên sales.
