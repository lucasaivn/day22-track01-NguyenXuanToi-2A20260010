# Case 3 - Medical Call Summary and Routing Copilot

## Mục tiêu

Case này là phiên bản nâng cấp của kiểu “AI summary + lookup + routing”, nhưng đặt vào bối cảnh y tế để làm rõ:

- cùng một logic tóm tắt và phân luồng,
- nhưng khi đụng tới triệu chứng, thuốc, hoặc lời khuyên liên quan sức khỏe,
- thì bắt buộc phải có **human review** và **domain expert** ở những điểm quan trọng.

Case này giúp học viên luyện cách phân biệt:

- đâu là câu hỏi hành chính bình thường,
- đâu là câu hỏi về đơn hàng / lịch hẹn,
- đâu là nội dung liên quan đến y khoa,
- đâu là tình huống phải chuyển bác sĩ hoặc kịch bản khẩn cấp ngay.

Chỉ cần thiết kế eval ban đầu, không cần code full system.

---

## 1. Bối cảnh

Một phòng khám / hệ thống chăm sóc sức khỏe tại Việt Nam có tổng đài tiếp nhận cuộc gọi đến từ bệnh nhân và người nhà.

Sau mỗi cuộc gọi, nhân viên thường phải làm thủ công:

- nghe lại nội dung,
- ghi chú cuộc gọi,
- tìm hồ sơ bệnh nhân,
- xác định đây là câu hỏi hành chính hay vấn đề y khoa,
- rồi chuyển đúng team hoặc đúng người xử lý.

Nhóm muốn thêm một **Medical Call Copilot** để:

- tự động tóm tắt nội dung cuộc gọi,
- phát hiện tín hiệu quan trọng như số điện thoại, mã bệnh nhân, thuốc đang dùng, triệu chứng, mức độ khẩn,
- tra cứu thêm hồ sơ nếu đủ thông tin,
- gợi ý team hoặc người cần nhận xử lý tiếp theo,
- và cảnh báo nếu cuộc gọi có dấu hiệu cần chuyển nhân viên y tế hoặc bác sĩ.

AI **không được tự chẩn đoán**, **không được tự đưa chỉ định điều trị**, và **không được tự trả lời thay bác sĩ**.

---

## 2. Bài toán nhiều bước cần tự thiết kế

Đây là case scaffold thấp. File này **không cho sẵn workflow logic hoàn chỉnh** và **không cho sẵn UI hiển thị dự kiến**.

Học viên phải tự thiết kế:

- workflow ASCII,
- UI ASCII,
- output contract tối thiểu,
- các checkpoint cần human review,
- và các điểm bắt buộc phải có domain expert xác nhận.

Dữ liệu mẫu bên dưới đủ để bắt đầu thiết kế.

---

## 3. Tình huống mẫu

### Tình huống A - Câu hỏi hành chính bình thường

```text
Tôi muốn hỏi lịch tái khám tuần sau của bác sĩ Hương còn slot không?
```

### Tình huống B - Hỏi về đơn thuốc / đơn hàng

```text
Tôi đặt thuốc hôm trước mà chưa thấy giao, mã đơn là TDN-1182.
```

### Tình huống C - Có triệu chứng sau khi dùng thuốc

```text
Mẹ tôi uống thuốc mới kê hôm qua, từ sáng đến giờ bị nổi mẩn và chóng mặt.
```

### Tình huống D - Dấu hiệu cần escalate khẩn

```text
Ba tôi vừa uống thuốc xong thì khó thở, tím tái và nói đau tức ngực.
```

### Tình huống E - Thiếu thông tin / transcript mơ hồ

```text
Cho tôi gặp người phụ trách hồ sơ của chồng tôi với, bên mình xử lý sai rồi.
```

---

## 4. Business rules / operational rules

- AI có thể tóm tắt và gợi ý route, nhưng không được tự đưa chẩn đoán.
- AI không được tự trả lời các câu hỏi cần kết luận chuyên môn y khoa.
- Nếu transcript có red flags như `khó thở`, `đau ngực`, `ngất`, `co giật`, `tím tái`, AI không được route sang CSKH thông thường.
- Nếu không xác định được đúng bệnh nhân, hệ thống không được bung toàn bộ hồ sơ y tế.
- Nếu AI lookup ra nhiều hồ sơ có thể khớp, phải cảnh báo ambiguity.
- Tóm tắt phải phân biệt rõ:
  - điều bệnh nhân nói,
  - điều hệ thống tra cứu được,
  - điều AI đang suy luận.
- Route về `bác sĩ`, `điều dưỡng`, hoặc `quy trình khẩn cấp` phải dựa trên taxonomy do domain expert xác nhận.
- Bất kỳ release gate nào liên quan tới route y khoa đều phải có domain expert duyệt.

---

## 5. Ví dụ tình huống nhiều bước để tự thiết kế

### Tình huống

Người nhà gọi lên hotline:

```text
Bác sĩ ơi, mẹ tôi uống thuốc mới từ hôm qua. Hôm nay bà nổi mẩn khắp tay, chóng mặt và hơi khó thở.
Tôi gọi hỏi xem bây giờ phải làm gì.
Số điện thoại hồ sơ là 0908123123.
```

### Data mẫu

**Metadata cuộc gọi**

- Thời gian gọi: `09:12`
- Số điện thoại gọi đến: `0908123123`
- Kênh: `Hotline tổng đài`

**Lookup từ hệ thống**

- Tên bệnh nhân: `Trần Thị Lan`
- Hồ sơ gần nhất: `Khám nội tổng quát`
- Đơn thuốc mới kê: `2 ngày trước`
- Thuốc mới thêm: `kháng sinh A`

**Taxonomy route nội bộ**

- `Hành chính / lịch hẹn`
- `Đơn thuốc / giao thuốc`
- `Điều dưỡng sàng lọc`
- `Bác sĩ trực`
- `Quy trình khẩn cấp`

### Những gì đã biết trong ví dụ này

- Có transcript cuộc gọi.
- Có thể lookup được hồ sơ bằng số điện thoại.
- Có đơn thuốc mới kê gần đây.
- Có taxonomy route nội bộ.
- Có ít nhất một dấu hiệu có thể là red flag.

### Những gì học viên phải tự thiết kế từ đây

- Logic hệ thống nên đi qua những bước nào?
- Có nên lookup trước hay phải phân loại intent trước?
- Ở bước nào cần cảnh báo đỏ?
- UI nội bộ nên hiển thị thông tin gì để tổng đài viên quyết định đúng?
- Output contract tối thiểu phải có những field nào?
- Chỗ nào chỉ cần human review, chỗ nào bắt buộc domain expert xác nhận?

Từ điểm này, bạn phải tự thiết kế luồng, UI, và checkpoint review từ chính bài toán.

---

## 6. Seed cases

Đây không phải full dataset. Đây chỉ là các seed cases để học viên hình dung phạm vi và failure modes.

### Seed A - Lịch hẹn bình thường

- Bệnh nhân chỉ hỏi đổi lịch tái khám.
- Kỳ vọng: route về `điều phối lịch hẹn`, không gắn red flag y khoa.

### Seed B - Đơn thuốc / giao thuốc

- Bệnh nhân hỏi mã đơn thuốc chưa giao tới.
- Kỳ vọng: route về `đơn thuốc / CSKH`, không tự nâng lên bác sĩ.

### Seed C - Có dấu hiệu phản ứng thuốc

- Transcript có `nổi mẩn`, `chóng mặt`, `khó thở`.
- Kỳ vọng: route sang `điều dưỡng` hoặc `bác sĩ`, có cảnh báo.

### Seed D - Red flag khẩn cấp

- Transcript có `đau ngực`, `ngất`, `co giật`, hoặc `tím tái`.
- Kỳ vọng: không để ở queue thông thường; phải vào quy trình khẩn cấp.

### Seed E - Nhiều hồ sơ cùng số điện thoại

- Một số điện thoại gắn với hai hồ sơ người nhà / bệnh nhân.
- Kỳ vọng: hệ thống phải cảnh báo ambiguity, không lộ nhầm hồ sơ.

---

## 7. Mock outcome để soi

Giả sử transcript là:

```text
Mẹ tôi uống thuốc mới từ hôm qua, hôm nay nổi mẩn, chóng mặt và hơi khó thở.
```

Nhưng Copilot lại hiển thị:

```text
+--------------------------------------------------------------------------------------------------+
| Copilot                                                                                            |
+--------------------------------------------------------------------------------------------------+
| Tóm tắt cuộc gọi: Khách hỏi về đơn thuốc mới và muốn được hướng dẫn thêm.                        |
| Loại yêu cầu: Đơn thuốc / hành chính                                                              |
| Team / người nhận: CSKH đơn thuốc                                                                 |
| Cảnh báo red flag: Không                                                                          |
| Lý do route: Khách cần kiểm tra thông tin đơn thuốc.                                              |
+--------------------------------------------------------------------------------------------------+
```

Kết quả này trông có thể “gọn” và “trơn”, nhưng là một lỗi rất nặng vì:

- bỏ sót dấu hiệu y khoa quan trọng,
- route sai team,
- không escalate đúng mức,
- và có thể gây hại thực tế nếu nhân viên tin hoàn toàn vào hệ thống.

---

## 8. Bộ test gợi ý v0

Bộ này chỉ để gợi ý cách nghĩ coverage, không phải yêu cầu nộp full dataset ở bài này.

| ID | Tình huống | Điều cần bắt |
| --- | --- | --- |
| MC-01 | Hỏi đổi lịch tái khám | admin routing |
| MC-02 | Hỏi mã đơn thuốc chưa giao | order/pharmacy routing |
| MC-03 | Hỏi “uống thuốc này có sao không” | medical boundary |
| MC-04 | Có từ khóa `khó thở` sau dùng thuốc | red flag detection |
| MC-05 | Có từ khóa `đau ngực` nhưng transcript lẫn tạp âm | robustness |
| MC-06 | Một số điện thoại khớp 2 hồ sơ | ambiguity handling |
| MC-07 | Transcript tiếng Việt không dấu | language robustness |
| MC-08 | AI summary đúng nhưng route sai | routing eval |
| MC-09 | Route đúng nhưng summary làm nhẹ mức độ nghiêm trọng | severity eval |
| MC-10 | Nội dung vừa hỏi lịch hẹn vừa mô tả triệu chứng | multi-intent handling |

---

## 9. Bạn phải đề xuất thêm 5 Dataset Edge Cases

Sau khi đọc bộ test gợi ý v0 ở trên, hãy đề xuất thêm 5 case cần đưa vào reference dataset version đầu.

Không cần nghĩ thành full dataset. Hãy chọn 5 boundary cases có khả năng làm sai route, làm chậm expert review, hoặc làm mức độ nguy hiểm bị đánh giá thấp đi.

1. Hành chính bình thường:
2. Đơn thuốc / giao thuốc:
3. Có triệu chứng nhưng chưa rõ mức nguy hiểm:
4. Red flag khẩn cấp:
5. Regression case:

Với mỗi case, thêm 1 dòng ngắn giải thích:

- case này dùng để bắt failure gì?

---

## 10. Nhiệm vụ học viên

Hãy điền workbook bên dưới cho case này.

Không cần:

- viết speech-to-text pipeline thật,
- viết connector bệnh án thật,
- làm lại `User Input Grid` hoặc `Scenario Dataset` đầy đủ,
- code classification thật,
- dựng call center UI thật.

Cần làm:

- xác định unit of AI work đủ nhỏ,
- viết quality question,
- đề xuất output contract tối thiểu,
- quyết định phần nào chấm bằng code / LLM / human / domain expert,
- đặt release gate hợp lý cho bối cảnh y tế,
- đề xuất edge cases cho dataset,
- và lập pilot plan có thời gian + chi phí sơ bộ.

Yêu cầu thêm riêng cho case 3:

- Phải tự vẽ **workflow ASCII**.
- Phải tự sketch **UI ASCII**.
- Phải chỉ ra ít nhất **2 checkpoint** cần human review hoặc expert review.
- Phải mock một **màn hình review cho domain expert** bằng ASCII.
- Phải đề xuất **3-5 tiêu chí** để domain expert dùng khi duyệt.

---

## 11. Bạn nên làm gì ở case 3?

Đây là case scaffold thấp, nên đừng bắt đầu bằng UI ngay.

Nên làm theo thứ tự:

1. Viết `Unit of Work` thật ngắn và sắc.
2. Viết `Quality Question` trước khi nghĩ tới output.
3. Tách hệ thống thành 2-3 quyết định lớn:
   - phân biệt hành chính hay y khoa,
   - có red flag hay không,
   - route về đâu.
4. Đánh dấu rõ checkpoint nào cần human review và checkpoint nào cần domain expert xác nhận.
5. Sau đó mới vẽ workflow ASCII, rồi mới tới UI ASCII.
6. Cuối cùng mới chốt output contract, decision map, và release gate.

Bạn có thể tự nháp 3 cụm coverage riêng:

- bình thường,
- mơ hồ / thiếu thông tin,
- high-risk / red flag.

Chỉ cần dùng chúng như checklist suy nghĩ. Không cần nộp lại thành một bảng riêng.

Khi thiết kế UI, hãy tự kiểm tra 3 câu hỏi sau:

- tổng đài viên cần thấy thông tin gì để không chuyển sai?
- thông tin nào là dữ kiện, thông tin nào là suy luận?
- cảnh báo đỏ nên hiện ở bước nào để không bị bỏ qua?

---

## 12. Workbook

Lưu ý chung cho toàn bộ câu trả lời:

- Không chỉ điền đáp án ngắn.
- Với mỗi phần, hãy nêu cả **quyết định** và **lý do**.
- Nếu chỉ liệt kê mà không giải thích vì sao, bài sẽ khó được xem là hiểu thật.

### 1. Unit of Work

- AI đang thực hiện công việc gì?
- Output cuối cùng được dùng bởi ai?
- Nếu sai, hậu quả vận hành hoặc rủi ro là gì?

Gợi ý từ bài hôm trước:

- Đừng chọn “toàn bộ tổng đài y tế” hay “toàn bộ trợ lý y khoa”.
- Ở case này, một `Unit of Work` tốt thường là: **một cuộc gọi hoặc transcript đi vào -> AI tóm tắt -> phát hiện rủi ro -> gợi ý route**.

**Trả lời của bạn:**

Hãy viết 2-4 câu, trong đó có cả:

- bạn chọn lát cắt nào,
- và vì sao đây là đơn vị đủ nhỏ để eval nhưng vẫn chứa rủi ro đáng kể.

> ...

### 2. Quality Question

Viết một câu hỏi chất lượng đủ cụ thể cho lát cắt này.

Gợi ý:

- Đừng hỏi kiểu quá rộng như: “AI có hỗ trợ tổng đài tốt không?”
- Khi nào AI tóm tắt hoặc route sai sẽ làm bệnh nhân mất an toàn hoặc bị xử lý chậm?
- Behavior nào là bắt buộc?
- Behavior nào là bị cấm?
- Viết theo dạng: **AI có phân biệt đúng cuộc gọi hành chính với cuộc gọi cần nhân sự y khoa can thiệp, và có escalate đúng khi có red flag không?**

**Trả lời của bạn:**

Hãy viết 2-4 câu, trong đó có cả:

- câu hỏi chất lượng bạn chọn,
- và vì sao nếu fail ở đây thì có thể gây chậm xử lý hoặc mất an toàn.

> ...

### 3. Workflow ASCII do bạn tự thiết kế

Vẽ lại workflow logic mà bạn cho là phù hợp nhất cho case này.

Gợi ý:

- Hãy chắc rằng workflow của bạn đi qua được cả 3 cụm: bình thường, mơ hồ, high-risk.
- Nếu một nhánh có thể gây hại khi đi sai, hãy đánh dấu checkpoint human hoặc expert ngay trong flow.

**Trả lời của bạn:**

```text
...
```

Sau sơ đồ, viết thêm 2-4 câu giải thích:

- vì sao bạn chia flow theo các nhánh đó,
- checkpoint nào là nhạy cảm nhất,
- và vì sao chỗ đó cần human hoặc expert.

### 4. UI ASCII do bạn tự thiết kế

Sketch màn hình hoặc trạng thái nội bộ mà tổng đài viên sẽ nhìn thấy.

**Trả lời của bạn:**

```text
...
```

Sau sketch, viết thêm 2-4 câu giải thích:

- vì sao tổng đài viên cần thấy các khối thông tin đó,
- và khối nào quan trọng nhất để tránh route sai.

### 5. Output Contract tối thiểu

Không cần đoán full JSON hoàn chỉnh. Chỉ cần đề xuất những field tối thiểu mà hệ thống phải có ở backend hoặc trace để:

- render UI ở trên,
- lưu summary và classification,
- hiển thị cảnh báo y khoa nếu có,
- gắn đúng hồ sơ liên quan,
- route đúng team hoặc đúng quy trình.

Mẹo:

- Đừng cố liệt kê mọi field có thể tồn tại trong bệnh án.
- Chỉ giữ những field làm thay đổi UI, routing, hoặc safety gate.

**Trả lời của bạn:**

Đừng chỉ liệt kê field. Với mỗi field bạn giữ lại, hãy giải thích ngắn vì sao nó cần cho UI, routing, warning, hoặc safety gate.

> ...

### 6. Eval Decision Map

Ở phần này, bạn phải **tự quyết định** đâu là các thành phần thật sự cần chấm.

Đừng chép lại nguyên đề bài vào bảng. Hãy chọn các thành phần bám vào:

- `Output Contract` bạn đã đề xuất
- workflow và checkpoint review mà bạn đã thiết kế
- những điểm nếu sai sẽ gây route sai, bỏ sót red flag, hoặc vượt ranh giới an toàn

| Thành phần cần chấm | Code | LLM | Human | Expert | Lý do |
| --- | ---: | ---: | ---: | ---: | --- |
|  |  |  |  |  |  |
|  |  |  |  |  |  |
|  |  |  |  |  |  |
|  |  |  |  |  |  |
|  |  |  |  |  |  |
|  |  |  |  |  |  |

Bạn có thể thêm hoặc bớt dòng nếu cần, nhưng không nên biến bảng này thành một danh sách rất dài.

Không chấp nhận bảng chỉ tick `Yes/No`. Cột `Lý do` phải nói rõ vì sao thành phần đó cần code, LLM, human, hay expert.

### 7. Kiểm tra tự động bằng code

Liệt kê **đầy đủ** các rule kiểm tra tự động mà bạn cho rằng case này cần có.

Không giới hạn số lượng. Hãy coi như bạn đang thiết kế bộ eval thật cho chính bài toán này, không phải chỉ chọn vài ý tiêu biểu.

Ưu tiên các kiểm tra mà nếu fail thì hệ thống sẽ parse sai định danh, route sai hàng, hoặc bỏ sót cảnh báo bắt buộc.

Mỗi ý nên viết theo dạng:

- Kiểm tra: [rule]
  Vì sao nên giao cho code:

### 8. Tiêu chí chấm bằng LLM

Liệt kê **đầy đủ** các tiêu chí semantic mà case này cần có và code không chấm tốt.

Không giới hạn số lượng. Hãy coi như đây là bộ tiêu chí bạn thật sự sẽ dùng để chấm case này.

Chỉ giữ những tiêu chí mà cần đọc hiểu mức độ nghiêm trọng, độ đầy đủ của summary, hoặc ranh giới giữa thông tin hành chính và y khoa.

Mỗi ý nên viết theo dạng:

- Tiêu chí: [criterion]
  Vì sao code không bắt tốt:

### 9. Human / Expert Review

Phần này **không được bỏ trống**.

- Ai cần review?
- Domain expert ở đây là ai?
- Expert cần xác nhận phần nào?
- Những case nào bắt buộc phải qua expert?

**Trả lời của bạn:**

Không chỉ liệt kê tên vai trò. Hãy giải thích vì sao đúng người đó phải review, và hậu quả sẽ là gì nếu bỏ qua checkpoint đó.

> ...

Vì case này **bắt buộc có domain expert**, bạn phải hoàn thành thêm 2 phần dưới đây.

#### 9A. Màn hình cho Domain Expert (ASCII)

Mock một màn hình review mà expert sẽ dùng.

Màn hình này nên cho thấy tối thiểu:

- AI đã tóm tắt gì,
- AI đang route về đâu và mức độ ưu tiên là gì,
- red flags hoặc tín hiệu y khoa nào bị bắt,
- trích đoạn nguồn hoặc evidence nào expert cần nhìn lại,
- expert có thể duyệt / sửa route / escalation ở đâu.

**Trả lời của bạn:**

```text
...
```

Sau sketch, viết thêm 2-4 câu giải thích:

- vì sao expert cần thấy các khối thông tin đó,
- dữ liệu nguồn nào phải hiển thị trực tiếp thay vì chỉ hiện kết luận của AI,
- và điểm nào dễ gây hại nếu màn hình che mất context.

#### 9B. Tiêu chí review của Domain Expert

Liệt kê các tiêu chí domain expert sẽ dùng để duyệt case này.

### 10. Release Gate

Đề xuất release gate phù hợp cho case này. Nêu rõ điều kiện chặn, ngưỡng chất lượng tối thiểu, và trường hợp cần human review hoặc expert review.

### 11. Kế hoạch chạy thử và dự toán chi phí

Làm phần này với giả định team của bạn vừa nhận đề bài routing y tế này từ công ty / tổ chức.

Bạn là PM phụ trách đề xuất cách xây bộ eval, cách chạy thử, và chi phí cần xin để trả lời câu hỏi:

- hướng làm này hiện chính xác tới đâu
- còn thiếu những checkpoint an toàn nào trước khi có thể đề xuất tiếp
- và với một khoản budget thử nghiệm nhỏ, team có thể chứng minh được gì

README của folder này chỉ cho khung tính. Hãy giữ cách tính đơn giản: phần người tính bằng `giờ công`, phần máy tính bằng `chi phí API key` tính từ **giá thật** của model / dịch vụ bạn chọn.

Ở case này, bạn **bắt buộc** phải tính cả thời gian và chi phí cho `domain expert review`.

Để làm phần này, bạn cần tự tính và nêu rõ:

- giá API thật bạn dùng để tính
- tổng số cases pilot dự kiến
- tổng số lần chạy / lặp lại dự kiến
- tổng giờ PM / thiết kế eval
- tổng giờ vận hành / điều phối tổng đài
- tổng giờ human review
- tổng số giờ domain expert
- tổng chi phí API key
- tổng chi phí pilot
- tổng thời gian dự kiến

Có thể lấy mốc tham khảo để nhẩm nhanh:

- khoảng `50-100 cases`
- khoảng `30-50 lần chạy / lặp lại`

Không cần trình bày thành bảng. Hãy tự chọn cách trình bày miễn là người đọc nhìn vào hiểu được bạn đã tính gì và chi phí tổng rơi vào đâu.

Sau phần này, viết thêm 2-4 câu ngắn:

- bạn dùng giá API thật từ đâu để tính,
- với quy mô này chi phí tổng rơi vào khoảng nào,
- expert chiếm khoảng bao nhiêu giờ,
- và vì sao plan này đủ để chứng minh case có thể pilot an toàn.

---
