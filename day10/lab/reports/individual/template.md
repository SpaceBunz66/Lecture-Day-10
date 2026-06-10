# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** [Nguyễn Thái Hoàng]
**Vai trò:** Cleaning / Expectation / Retrieval Evaluation
**Ngày nộp:** [10/06/2026]
**Run ID chính:** `2026-06-10T07-45Z`
**File phụ trách:** `transform/cleaning_rules.py`, `quality/expectations.py`, `artifacts/eval/grading_run.jsonl`

---

## 1. Tôi phụ trách phần nào?

Trong Lab Day 10, tôi phụ trách chính phần làm sạch dữ liệu, bổ sung expectation suite và kiểm tra kết quả retrieval sau khi pipeline được sửa. Cụ thể, tôi chỉnh sửa `transform/cleaning_rules.py` để mở rộng `ALLOWED_DOC_IDS`, thêm `access_control_sop`, xử lý conflict version của HR leave policy, loại bỏ nội dung `Ticket P2` nằm sai trong tài liệu `sla_p1_2026`, và cải thiện chunk escalation P1 để retrieval trả lời đúng câu hỏi về auto escalation sau `10 phút`. Ngoài ra, tôi bổ sung các kiểm tra mới trong `quality/expectations.py`, ví dụ kiểm tra `access_control_sop` phải tồn tại sau clean, không còn `Ticket P2` trong SLA P1, và phải có chunk escalation P1 chứa `10 phút`. Bằng chứng chính là pipeline chạy thành công với log `PIPELINE_OK`, `raw_records=247`, `cleaned_records=44`, `quarantine_records=203`.

---

## 2. Một quyết định kỹ thuật

Một quyết định kỹ thuật quan trọng của tôi là phân loại các lỗi dữ liệu ảnh hưởng trực tiếp đến grading thành expectation mức `halt`, thay vì chỉ để `warn`. Ví dụ, nếu `hr_leave_policy` vẫn còn nội dung cũ `10 ngày phép năm`, pipeline phải dừng vì dữ liệu này có thể khiến hệ thống trả lời sai chính sách HR 2026. Tương tự, nếu `access_control_sop` không xuất hiện trong cleaned data, câu hỏi về `Level 4 Admin Access` sẽ không thể trả lời đúng. Tôi cũng chọn quarantine nội dung `Ticket P2` trong document `sla_p1_2026`, vì chunk này gây nhiễu cho câu hỏi P1 escalation và làm retrieval lấy nhầm thông tin `90 phút` thay vì `10 phút`. Các expectation này giúp pipeline không chỉ chạy được, mà còn bảo vệ chất lượng dữ liệu trước khi embed vào vector store.

---

## 3. Một lỗi hoặc anomaly đã xử lý

Lỗi đầu tiên tôi xử lý là pipeline bị halt ở expectation `hr_leave_no_stale_10d_annual`. Log ban đầu cho thấy `violations=2`, nghĩa là sau khi clean vẫn còn 2 dòng HR cũ chứa thông tin `10 ngày phép năm`. Tôi sửa rule trong `cleaning_rules.py` để quarantine các chunk `hr_leave_policy` có ngày hiệu lực trước `2026-01-01` hoặc có marker nội dung cũ như `10 ngày phép năm` và `bản HR 2025`. Sau đó expectation này chuyển sang `OK` với `violations=0`. Lỗi tiếp theo là grading câu `gq_d10_06` và `gq_d10_10`. Câu 10 fail vì `access_control_sop` chưa được retrieve đúng, nên tôi thêm document này vào allowlist và rebuild Chroma index. Câu 6 fail vì retrieval bị nhiễu bởi chunk `Ticket P2`, nên tôi thêm rule quarantine P2 khỏi SLA P1 và làm rõ chunk escalation P1 chứa `10 phút`.

---

## 4. Bằng chứng trước / sau

Trước khi sửa, pipeline halt tại HR expectation:

`expectation[hr_leave_no_stale_10d_annual] FAIL (halt) :: violations=2`

Sau khi sửa, pipeline chạy thành công:

`run_id=2026-06-10T07-45Z`
`raw_records=247`
`cleaned_records=44`
`quarantine_records=203`
`expectation[hr_leave_no_stale_10d_annual] OK (halt) :: violations=0`
`PIPELINE_OK`

Kết quả grading cuối cùng cũng đạt đủ 10/10. Các câu quan trọng đã pass: `gq_d10_06` có `contains_expected=true`, `hits_forbidden=false`, `top1_doc_matches=true`; `gq_d10_10` có `top1_doc_id=access_control_sop`, `contains_expected=true`, `hits_forbidden=false`, `top1_doc_matches=true`.

---

## 5. Cải tiến tiếp theo

Nếu có thêm 2 giờ, tôi sẽ cải tiến rule versioning để không hard-code. Thay vào đó, cutoff date và danh sách allowed documents nên được đọc từ `contracts/data_contract.yaml` hoặc biến môi trường. Cách này giúp pipeline dễ bảo trì hơn khi chính sách đổi version trong tương lai.
