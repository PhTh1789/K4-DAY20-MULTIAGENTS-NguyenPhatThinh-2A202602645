# Báo cáo Lab: Self evolving Agentic

> Sao chép tệp này thành `report/REPORT.md` (đã làm ở Phần 0) và điền dần qua các Phần của lab. Xóa các dòng hướng dẫn dạng trích dẫn (bắt đầu bằng `>`). Văn phong kỹ thuật, ngắn gọn, mọi nhận định đi kèm số liệu hoặc bằng chứng. Trong buổi học: điền mục 1 đến 7 (bản nháp). Sau buổi học: hoàn thiện mục 8 đến 10.

## 1. Thông tin nhóm và cấu hình

| Họ tên            | Mã sinh viên | Phần đóng góp |
| ----------------- | ------------ | ------------- |
| Nguyễn Phát Thịnh | 2A292602645  | Toàn bộ       |

- Mô hình (tên deployment hoặc `LAB_MODEL`), nhiệt độ (`LAB_TEMPERATURE`), `recursion_limit`: `gpt-6-luna` (qua OpenAI official API), nhiệt độ `1`, `recursion_limit = 60`
- Phiên bản Deep Agents (`pip show deepagents`), hệ điều hành, chạy trực tiếp hay trong Docker: Deep Agents `0.7.21`, Windows (chạy trực tiếp trong môi trường ảo `.venv`)
- Số lần chạy tác vụ đã dùng / ngân sách: 0 / 18
- Commit của tag `freeze`: (chưa tạo)

## 2. Giả thuyết (commit TRƯỚC tag `freeze`, Phần 4.0)

- H1 (subagents so với baseline): Trên tác vụ đánh giá, điều kiện `subagents` dự đoán đạt điểm tương đương hoặc chênh lệch không đáng kể so với `baseline` nhưng tiêu tốn lượng token gấp 1.8 đến 2.5 lần. Căn cứ từ kết quả tác vụ học: cả baseline và subagents đều đạt 17/18 check kỹ thuật và 0/9 check quy ước vì subagents được cô lập ngữ cảnh và không tự tạo thêm tri thức thủ tục mới (phù hợp với nhận định trong nghiên cứu multi-agent của Anthropic về chi phí token tăng vọt).
- H2 (skills-auto so với baseline): Trên tác vụ đánh giá, điều kiện `skills-auto` sẽ đạt điểm cao hơn rõ rệt so với `baseline` ở các check quy ước dùng chung đã được đúc kết từ tác vụ học (như tiền tệ integer cents, schema metadata, sort order), nhưng sẽ không giải quyết được các quy ước hoàn toàn mới của tác vụ đánh giá. Căn cứ từ nghiên cứu SkillsBench và SkillEvolBench: kỹ năng ngữ cảnh giúp chuyển giao tri thức thủ tục nhưng gặp hiện tượng giảm hiệu quả khi gặp các quy ước chưa từng xuất hiện.
- H3 (tác vụ học so với tác vụ đánh giá): Điểm số trung bình trên tác vụ học sẽ cao hơn tác vụ đánh giá ở điều kiện `skills-auto` (chênh lệch khoảng 15-25 điểm phần trăm), trong khi ở điều kiện `baseline` và `subagents` thì điểm số giữa 2 tập tương đương nhau. Căn cứ: Tác vụ đánh giá bổ sung thêm các quy ước mới của Acme mà tập học không có, do đó skill tự sinh chỉ giúp đạt một phần các quy ước tái sử dụng.

## 3. Làm quen Deep Agents (Phần 0.3)

1. Tác tử mặc định có 9 công cụ: `ls`, `read_file`, `write_file`, `edit_file`, `delete`, `glob`, `grep`, `execute`, `task`. Trong đó, công cụ cho phép chạy lệnh shell trên hệ điều hành là `execute`.
2. Mô tả của công cụ `task` nêu về subagent `general-purpose`: Đây là tác tử đa năng dùng để nghiên cứu câu hỏi phức tạp, tìm kiếm tệp/nội dung và thực hiện các tác vụ nhiều bước; nó có toàn quyền truy cập các công cụ như tác tử chính. Về ngữ cảnh: Mỗi lần gọi là phi trạng thái (stateless) theo mặc định, subagent chỉ nhìn thấy những gì được truyền trong prompt giao việc và trả về một báo cáo kết thúc duy nhất (không thấy lịch sử ngữ cảnh của tác tử chính trừ khi được chỉ dẫn tường minh).
3. Hướng dẫn hành vi:
   - Từ mô tả công cụ `task`: *"Put full detail in the prompt and state exactly what it should return"* (và *"The agent's report is not shown to the user; relay a summary yourself"*).
   - Từ mô tả công cụ `execute`: *"You MUST avoid using search commands like find and grep. Instead use the grep, glob tools to search. Use read_file rather than cat/head/tail."*

## 4. Đường cơ sở và phân loại lỗi (Phần 2.2)

| Tác vụ | Check thất bại | Nhóm lỗi (A-G) | Bằng chứng (trích ngắn từ `detail` hoặc vết) |
| ------ | -------------- | -------------- | -------------------------------------------- |
| `code-learn` | `tests_not_modified` | B (Không kiểm chứng) | `"the original files in tests/ must not be modified (new test files are allowed)"` |
| `code-learn` | `rule_type_hints` | E (Vi phạm quy ước tổ chức) | `"RULE: every public function... has type annotations on all parameters and on the return value."` |
| `code-learn` | `rule_regression_tests` | E (Vi phạm quy ước tổ chức) | `"RULE: add tests/test_regressions.py with one test function per bug you fixed (at least 3)"` |
| `code-learn` | `rule_changelog` | E (Vi phạm quy ước tổ chức) | `"RULE: record each fix in CHANGELOG.md under the heading '## Unreleased' as a bullet '- fix(<function name>): <short description>'"` |
| `data-learn` | `rule_money_in_cents` | E (Vi phạm quy ước tổ chức) | `"RULE: money values in answer.json are integer cents (1606.67 USD is written 160667)."` |
| `data-learn` | `rule_meta_block` | E (Vi phạm quy ước tổ chức) | `"RULE: answer.json has an object meta = {\"source\": <input file name>, \"rows_in\": ..., \"rows_used\": ...}."` |
| `data-learn` | `rule_clean_csv` | E (Vi phạm quy ước tổ chức) | `"RULE: write workspace/clean.csv with the header order_id,timestamp_utc,region,amount_cents; ..."` |
| `logs-learn` | `rule_service_names` | E (Vi phạm quy ước tổ chức) | `"RULE: service names in the output are lower-case with '-' replaced by '_' (payment-service -> payment_service)."` |
| `logs-learn` | `rule_sorted_errors` | E (Vi phạm quy ước tổ chức) | `"RULE: errors is sorted by service, then by timestamp_utc, ascending."` |
| `logs-learn` | `rule_schema_header` | E (Vi phạm quy ước tổ chức) | `"RULE: the top-level object has 'schema_version': 2 and 'generated_by': 'log-triage'."` |

Nhận xét:
- Nhóm lỗi **E (Vi phạm quy ước tổ chức Acme)** chiếm đa số áp đảo (9/10 check thất bại). Các quy ước này không xuất hiện trong đề bài của tác vụ mà là quy ước ngầm định của tổ chức.
- **Bằng chứng phủ định cho các nhóm A đến D**: Số check kỹ thuật đạt của mô hình là **17/18** (đạt 94.4% theo `scripts/check_breakdown.py`), chứng minh mô hình hiểu đề và giải quyết bài toán nghiệp vụ xuất sắc.
- **Khả năng phòng ngừa của Skill**: Hoàn toàn có thể phòng ngừa bằng skill. Bộ tuyển chọn (curator) có thể đọc các nhận xét vi phạm này để tự sinh ra các bản hướng dẫn thủ tục (procedural guidelines) chứa các quy ước tổ chức Acme.

## 5. Điều kiện `subagents` (Phần 2.3)

- Các subagent đã định nghĩa:
  + `explorer`: Chuyên khảo sát, đọc README, code, cấu trúc thư mục, schema dữ liệu và báo cáo khách quan không sửa file.
  + `implementer`: Chuyên viết code, sửa file và thực thi lệnh/test theo chỉ dẫn.
  + `reviewer`: Chuyên kiểm tra độc lập sản phẩm đầu ra đối chiếu với yêu cầu đề bài.
- `subagent_calls` ở từng tác vụ và nhận xét:
  + `code-learn`: 3 lần gọi (`explorer`, `implementer`, `reviewer`). Tác tử chính phân rã quy trình thành 3 pha rất bài bản.
  + `data-learn`: 1 lần gọi (`explorer` để phân tích schema tệp dữ liệu).
  + `logs-learn`: 2 lần gọi (`implementer` để xử lý và trích xuất log).
- Thông tin giao việc: Lời giao việc truyền tải đầy đủ ngữ cảnh nhiệm vụ và đường dẫn file; nhờ `PATHS_NOTE` được tự động chèn vào `system_prompt` của từng subagent, các subagent không gặp lỗi đường dẫn ảo.
- Ảnh hưởng đến token và thời gian: Chi phí token tăng gấp hơn **2.1 lần** (trung bình 151,979 tokens ở `subagents` so với 71,914 tokens ở `baseline`). Thời gian thực thi tăng tương ứng. Điểm số chưa cải thiện (vẫn đạt 17/18 kỹ thuật và 0/9 quy ước) do subagent chưa được trang bị tri thức về quy ước Acme.

## 6. Self-evolving: skill do curator sinh (Phần 3)

- Số lần chạy curator, số skill bị xóa và lý do: Chạy curator 1 lần duy nhất; sinh thành công 3 skill; 0 skill bị xóa vì cả 3 đều đạt chuẩn `validate_skill`, xúc tích (11-12 dòng), tính khái quát cao và không rò rỉ dữ liệu đánh giá.

| Skill | Tổng quát hay riêng cho tác vụ học? | Đúng hay sai (nêu chỗ sai nếu có) | Độ dài, `description` và `skills_read` ở Phần 3.4 |
| ----- | ----------------------------------- | --------------------------------- | ------------------------------------------------- |
| `repository-bug-fix-workflow` | Tổng quát cho mọi bài toán sửa bug có ràng buộc về regression test, type hints và changelog. | Đúng: Hướng dẫn giữ nguyên test gốc, thêm type annotations, tạo file test regression và ghi nhận changelog chuẩn. | 11 dòng; *"Use when fixing bugs in an existing package that has repository rules for tests, typing, or changelog entries."*; `skills_read` = 1 (`code-learn`). |
| `structured-data-deliverables` | Tổng quát cho việc chuyển đổi dữ liệu bảng thành JSON/CSV chuẩn hóa có schema và quy tắc tiền tệ. | Đúng: Hướng dẫn chuyển đổi tiền tệ sang integer minor units (cents), tách biệt đếm dòng và xuất file CSV chuẩn. | 12 dòng; *"Use when transforming tabular input into JSON or CSV deliverables with specified schemas, deduplication, or money and timestamp rules."*; `skills_read` = 2 (`data-learn`). |
| `normalized-log-outputs` | Tổng quát cho việc bóc tách log thành báo cáo lỗi JSON có sắp xếp và chuẩn hóa tên service. | Đúng: Hướng dẫn chuẩn hóa tên service, chuyển múi giờ trước khi sort và khai báo `schema_version`. | 11 dòng; *"Use when parsing logs into structured error reports with canonical fields, metadata, or ordering requirements."*; `skills_read` = 1 (`logs-learn`). |

## 7. Kết quả so sánh (Phần 4.3, 4.4)

> Dán nội dung `report/table.md` và kết quả `python scripts/check_breakdown.py`. Nêu các lần chạy có `error` hoặc `skills_modified = true` (nếu có) và cách xử lý.

```text
(dán bảng ở đây)
```

## 8. Phân tích

> Trả lời từng câu bằng số liệu từ mục 7 và bằng chứng từ vết. Kết quả âm hoặc không có khác biệt vẫn hợp lệ nếu được phân tích tốt.

1. So với `baseline`, điều kiện nào cải thiện điểm tác vụ **học**? Điều kiện nào cải thiện điểm tác vụ **đánh giá**? Có điều kiện nào cải thiện tác vụ học nhưng không cải thiện tác vụ đánh giá? Nếu có, đó là dấu hiệu gì?
2. Tách điểm thành check kỹ thuật và check quy ước (`rule_`). Skill do curator sinh giúp nhóm check nào? Check quy ước **mới** của tác vụ đánh giá có được skill giúp không, và vì sao?
3. Dựa vào vết và `skills_read`, giải thích một check mà skill giúp đạt và một check mà skill không giúp (skill chưa được đọc, đọc nhưng không làm theo, skill thiếu hoặc sai).
4. Chi phí: so sánh số token trung bình giữa các điều kiện. Điều kiện nào có hiệu quả tốt nhất theo điểm trên mỗi token? Đa tác tử có đáng chi phí trong thí nghiệm này không?
5. Có dấu hiệu rò rỉ dữ liệu hoặc quá khớp nào trong skill sinh ra không? Nhóm đã phòng tránh như thế nào?
6. Nhiễu: so sánh điểm tác vụ học của cùng bộ skill ở Phần 3.4 (đã sao lưu) và sau đóng băng. Chênh lệch bao nhiêu? Nó cho biết điều gì về độ tin cậy của các chênh lệch trong bảng ở mục 7?

## 9. Hạn chế và tính hợp lệ

> Nêu ít nhất 3 hạn chế và ảnh hưởng của từng hạn chế đến kết luận (ví dụ: chỉ 3 tác vụ mỗi vai trò, mỗi cấu hình chạy một lần, nhiễu của mô hình, tác vụ do giảng viên thiết kế sẵn quy ước, chỉ một mô hình).

1.
2.
3.

## 10. Kết luận

> Tối đa 5 câu. Chỉ khẳng định điều số liệu hỗ trợ. Nêu một đề xuất cải tiến tiếp theo.

## Phụ lục

- Lệnh đã chạy (theo thứ tự):
- Thử thách mở rộng (nếu có): hướng chọn, kết quả, nhận xét.
- Ghi chú khác:
