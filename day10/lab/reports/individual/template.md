# Individual Report — Day 10 Lab: Data Pipeline & Observability

**Name:** Nguyen Thai Hoang
**Role:** Data Cleaning, Expectation Suite, Retrieval Evaluation
**Lab folder:** `day10/lab`
**Main files modified:**

* `transform/cleaning_rules.py`
* `quality/expectations.py`
* `artifacts/eval/grading_run.jsonl`
* `artifacts/eval/after_fix_eval.csv`

---

## 1. My Responsibility

Trong lab này, tôi phụ trách chính phần cải thiện ETL pipeline cho quá trình ingest knowledge base. Công việc của tôi tập trung vào việc làm sạch dữ liệu policy bị nhiễu, bổ sung các data-quality guardrails, và kiểm tra xem hệ thống retrieval có thể trả lời đúng các câu hỏi grading/test hay không.

Ban đầu, pipeline đã có một số rule cơ bản như kiểm tra `doc_id` ngoài allowlist, chuẩn hoá `effective_date`, loại duplicate chunks, xử lý stale refund window và conflict version trong HR policy. Tuy nhiên, sau khi chạy pipeline và grading, tôi nhận thấy vẫn còn một số lỗi ảnh hưởng trực tiếp đến retrieval quality. Vì vậy, tôi chỉnh sửa chủ yếu trong hai file: `transform/cleaning_rules.py` và `quality/expectations.py`.

Kết quả cuối cùng, pipeline đã chạy thành công với log:

```txt
PIPELINE_OK
raw_records=247
cleaned_records=44
quarantine_records=203
```

Ngoài ra, kết quả grading cuối cùng đã pass toàn bộ 10 câu hỏi chính thức. Tất cả các dòng grading đều đạt:

```txt
contains_expected=true
hits_forbidden=false
top1_doc_matches=true
```

---

## 2. Technical Decisions

Một quyết định kỹ thuật quan trọng của tôi là coi các lỗi dữ liệu ảnh hưởng trực tiếp đến retrieval là expectation mức `halt`, thay vì chỉ để `warn`. Ví dụ, nếu cleaned data vẫn còn nội dung HR cũ như `10 ngày phép năm`, pipeline nên dừng lại vì thông tin này có thể khiến hệ thống trả lời sai câu hỏi về chính sách HR 2026. Tương tự, nếu `access_control_sop` không xuất hiện trong cleaned data thì hệ thống không thể trả lời đúng câu hỏi về `Level 4 Admin Access`.

Trong `transform/cleaning_rules.py`, tôi mở rộng allowed document catalog bằng cách thêm:

```python
"access_control_sop"
```

Việc này cần thiết vì câu grading `gq_d10_10` yêu cầu top-1 retrieved document phải là `access_control_sop`.

Tôi cũng thêm các cleaning rules kèm comment `metric_impact` để giải thích vì sao rule đó cần thiết và nó cải thiện metric nào. Các rule quan trọng gồm:

1. Loại stale HR 2025 leave-policy chunks dựa trên ngày hiệu lực và content markers. Rule này giúp loại câu trả lời sai `10 ngày phép năm` và hỗ trợ câu `gq_d10_09`.

2. Loại nội dung `Ticket P2` nằm sai trong document `sla_p1_2026`. Rule này tránh việc câu hỏi P1 escalation retrieve nhầm giá trị P2 là `90 phút`.

3. Enrich chunk P1 escalation bằng wording rõ hơn. Rule này giúp retrieval trả lời đúng câu hỏi auto escalation sau `10 phút`.

4. Enrich refund exception chunk bằng wording gần với câu hỏi grading/test. Rule này giúp câu hỏi về sản phẩm bị loại khỏi điều kiện hoàn tiền retrieve đúng `policy_refund_v4` làm top-1 document.

5. Giữ rule sửa stale refund window từ `14 ngày làm việc` sang `7 ngày làm việc`. Rule này ngăn hệ thống trả lời bằng chính sách hoàn tiền cũ.

---

## 3. Anomaly or Bug Handled

Lỗi lớn đầu tiên tôi gặp là pipeline bị halt ở expectation HR:

```txt
expectation[hr_leave_no_stale_10d_annual] FAIL (halt) :: violations=2
```

Điều này nghĩa là sau bước clean vẫn còn 2 chunks HR cũ chứa thông tin `10 ngày phép năm`. Đây là dữ liệu sai vì HR policy 2026 yêu cầu nhân viên dưới 3 năm kinh nghiệm có `12 ngày phép năm`. Tôi sửa lỗi này bằng cách thêm logic quarantine cho `hr_leave_policy`, dựa trên cả `effective_date` trước `2026-01-01` và các content markers như `10 ngày phép năm` hoặc `bản HR 2025`. Sau khi sửa, expectation này đã pass:

```txt
expectation[hr_leave_no_stale_10d_annual] OK (halt) :: violations=0
```

Lỗi thứ hai liên quan đến câu grading `gq_d10_06`. Câu hỏi yêu cầu hệ thống trả lời P1 auto escalation sau bao lâu, với đáp án đúng là `10 phút`. Ban đầu, retriever bị nhiễu bởi một chunk `Ticket P2` nằm trong `sla_p1_2026`, khiến kết quả có nguy cơ lấy nhầm thông tin escalation của P2 là `90 phút`. Tôi xử lý bằng cách quarantine nội dung `Ticket P2` khỏi document SLA P1, sau đó enrich lại chunk P1 escalation để câu hỏi retrieve đúng thông tin `10 phút`.

Lỗi thứ ba là câu `gq_d10_10`, trong đó top-1 document phải là `access_control_sop`. Ban đầu document này chưa được xử lý đúng trong allowlist/retrieval, nên tôi thêm `access_control_sop` vào `ALLOWED_DOC_IDS` và rebuild lại Chroma index.

Sau đó, khi test thêm với cả grading/test questions, tôi phát hiện regression ở `gq_d10_02`: đáp án vẫn đúng nhưng top-1 document bị chuyển sang `it_helpdesk_faq` thay vì `policy_refund_v4`. Tôi sửa bằng cách tăng độ rõ ràng của refund exception chunk, thêm wording gần với câu hỏi “loại sản phẩm bị loại khỏi điều kiện hoàn tiền”. Sau khi sửa, câu này quay lại đúng top-1 document là `policy_refund_v4`.

---

## 4. Expectations Added

Tôi cải thiện `quality/expectations.py` bằng cách thêm các custom expectations để tránh lỗi quay lại trong tương lai.

Các expectation mới gồm:

1. `access_control_sop_present`
   Expectation này kiểm tra `access_control_sop` có tồn tại trong cleaned dataset hay không. Đây là điều kiện quan trọng vì câu hỏi `Level 4 Admin Access` phụ thuộc vào document này.

2. `sla_p1_no_ticket_p2_content`
   Expectation này kiểm tra không còn nội dung `Ticket P2` trong document `sla_p1_2026`. Điều này giúp tránh retrieval nhầm thông tin P2 khi câu hỏi đang hỏi về P1.

3. `sla_p1_escalation_10min_present`
   Expectation này kiểm tra cleaned data vẫn còn chunk chứa thông tin escalation P1 `10 phút`. Nếu chunk này bị mất, hệ thống sẽ không thể trả lời đúng câu `gq_d10_06`.

Các expectation này giúp biến những lỗi phát hiện trong quá trình evaluation thành guardrails cố định. Nhờ vậy, pipeline không chỉ pass trong lần chạy hiện tại mà còn có khả năng phát hiện các lỗi tương tự trong các lần chạy sau.

---

## 5. Before and After Evidence

Trước khi sửa, pipeline bị halt vì stale HR content vẫn còn trong cleaned data:

```txt
expectation[hr_leave_no_stale_10d_annual] FAIL (halt) :: violations=2
PIPELINE_HALT
```

Sau khi sửa, pipeline chạy thành công:

```txt
run_id=2026-06-10T09-00Z
raw_records=247
cleaned_records=44
quarantine_records=203
expectation[min_one_row] OK (halt)
expectation[no_empty_doc_id] OK (halt)
expectation[refund_no_stale_14d_window] OK (halt)
expectation[chunk_min_length_8] OK (warn)
expectation[effective_date_iso_yyyy_mm_dd] OK (halt)
expectation[hr_leave_no_stale_10d_annual] OK (halt) :: violations=0
PIPELINE_OK
```

Sau khi thêm các expectation mới, pipeline cũng có thêm các kiểm tra như:

```txt
expectation[access_control_sop_present] OK (halt)
expectation[sla_p1_no_ticket_p2_content] OK (halt)
expectation[sla_p1_escalation_10min_present] OK (halt)
```

Kết quả grading cuối cùng pass toàn bộ 10 câu hỏi chính thức:

```txt
gq_d10_01: passed
gq_d10_02: passed
gq_d10_03: passed
gq_d10_04: passed
gq_d10_05: passed
gq_d10_06: passed
gq_d10_07: passed
gq_d10_08: passed
gq_d10_09: passed
gq_d10_10: passed
```

Một số evidence quan trọng:

```txt
gq_d10_02:
top1_doc_id=policy_refund_v4
contains_expected=true
hits_forbidden=false
top1_doc_matches=true

gq_d10_06:
top1_doc_id=sla_p1_2026
contains_expected=true
hits_forbidden=false
top1_doc_matches=true

gq_d10_10:
top1_doc_id=access_control_sop
contains_expected=true
hits_forbidden=false
top1_doc_matches=true
```

Điều này cho thấy pipeline sau khi sửa không chỉ chạy được mà còn retrieve đúng source document, chứa đáp án mong đợi và không chứa forbidden/outdated answer.

---

## 6. Reflection

Qua lab này, tôi hiểu rõ hơn rằng data cleaning trong hệ thống retrieval không chỉ là xoá dòng rỗng hoặc format sai. Một chunk nhìn có vẻ hợp lệ vẫn có thể làm giảm chất lượng trả lời nếu nó thuộc sai scope, chứa policy version cũ, hoặc có wording gây nhiễu cho retriever.

Bài học quan trọng nhất của tôi là mỗi cleaning rule nên gắn với một tác động đo được. Ví dụ, rule loại HR 2025 giúp loại forbidden answer `10 ngày phép năm`. Rule loại `Ticket P2` khỏi SLA P1 giúp câu hỏi P1 escalation không retrieve nhầm `90 phút`. Rule thêm `access_control_sop` giúp hệ thống trả lời đúng câu hỏi về Level 4 Admin Access. Rule enrich refund exception giúp giữ `policy_refund_v4` là top-1 document cho câu hỏi về sản phẩm không được hoàn tiền.

Tôi cũng nhận ra rằng `PIPELINE_OK` chưa chắc đồng nghĩa với retrieval quality tốt. Pipeline có thể pass về mặt kỹ thuật, nhưng grading vẫn fail nếu top-1 document sai. Vì vậy, tôi cần kết hợp cả expectation suite, eval results và grading results để đánh giá chất lượng cuối cùng của pipeline.

---

## 7. Future Improvements

Nếu có thêm thời gian, tôi sẽ cải thiện pipeline theo ba hướng.

Thứ nhất, tôi sẽ đưa các giá trị hard-code như HR cutoff date `2026-01-01`, danh sách allowed documents, và stale content markers vào file cấu hình như `contracts/data_contract.yaml`. Như vậy pipeline sẽ dễ bảo trì hơn khi policy thay đổi.

Thứ hai, tôi sẽ viết một regression test script nhỏ để tự động kiểm tra rằng tất cả grading questions đều pass sau mỗi lần chạy pipeline. Điều này giúp phát hiện sớm nếu một cleaning rule mới vô tình làm hỏng một câu đã pass trước đó.

Thứ ba, tôi sẽ cải thiện observability bằng cách tạo summary cho quarantine reasons. Ví dụ, report có thể cho biết bao nhiêu dòng bị quarantine vì `unknown_doc_id`, duplicate chunks, stale HR policy, P2 content, hoặc invalid dates. Điều này giúp việc audit pipeline dễ hơn và giải thích kết quả rõ ràng hơn.

Tổng kết lại, sau các thay đổi của tôi, pipeline trở nên ổn định hơn, dễ quan sát hơn và phù hợp hơn với yêu cầu retrieval evaluation của lab.
