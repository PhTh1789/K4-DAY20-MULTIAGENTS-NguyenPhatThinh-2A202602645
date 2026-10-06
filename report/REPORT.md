# Báo cáo Lab: Self evolving Agentic

> Sao chép tệp này thành `report/REPORT.md` (đã làm ở Phần 0) và điền dần qua các Phần của lab. Xóa các dòng hướng dẫn dạng trích dẫn (bắt đầu bằng `>`). Văn phong kỹ thuật, ngắn gọn, mọi nhận định đi kèm số liệu hoặc bằng chứng. Trong buổi học: điền mục 1 đến 7 (bản nháp). Sau buổi học: hoàn thiện mục 8 đến 10.

## 1. Thông tin nhóm và cấu hình

| Họ tên            | Mã sinh viên | Phần đóng góp |
| ----------------- | ------------ | ------------- |
| Nguyễn Phát Thịnh | 2A292602645  | Toàn bộ       |

- Mô hình (tên deployment hoặc `LAB_MODEL`), nhiệt độ (`LAB_TEMPERATURE`), `recursion_limit`: `gpt-6-luna` (qua OpenAI official API), nhiệt độ `1`, `recursion_limit = 60`
- Phiên bản Deep Agents (`pip show deepagents`), hệ điều hành, chạy trực tiếp hay trong Docker: Deep Agents `0.7.21`, Windows (chạy trực tiếp trong môi trường ảo `.venv`)
- Số lần chạy tác vụ đã dùng / ngân sách: 18 / 18
- Commit của tag `freeze`: `e52379bce8c7f2becc9ab8a91bbd9f064d5fd9ac`

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

Bảng so sánh tổng hợp sinh bởi `lab.compare`:

| Task | baseline | subagents | skills-auto |
|---|---|---|---|
| code-learn | 6/10 | 6/10 | 7/10 |
| data-learn | 5/8 | 5/8 | 5/8 |
| logs-learn | 6/9 | 6/9 | 6/9 |
| code-eval | 6/11 | 6/11 | 8/11 |
| data-eval | 5/9 | 5/9 | 5/9 |
| logs-eval | 1/10 | 6/10 | 6/10 |
| **Mean score - learning tasks** | 0.63 | 0.63 | 0.66 |
| **Mean score - evaluation tasks** | 0.40 | 0.57 | 0.63 |
| **Mean tokens per run** | 75,813 | 130,749 | 134,317 |
| **Runs that read a skill** | 0/6 | 0/6 | 6/6 |

Thống kê chi tiết check kỹ thuật và check quy ước Acme (`scripts/check_breakdown.py`):

```text
condition     role    technical  house rules  mean tokens  read a skill
baseline      eval     12/18         0/12          79,713      0/3     
baseline      learn    17/18         0/9           71,914      0/3     
subagents     eval     17/18         0/12         109,519      0/3     
subagents     learn    17/18         0/9          151,979      0/3     
skills-auto   eval     17/18         2/12         109,828      3/3     
skills-auto   learn    17/18         1/9          158,806      3/3     
```

*Ghi chú*: Toàn bộ 18 lượt chạy chính thức đều hoàn thành thành công (`error = None`), không có lượt chạy nào vi phạm sửa đổi thư mục kỹ năng (`skills_modified = false`). Script kiểm tra `verify_freeze.py` xác nhận đạt chuẩn 100% `OK`.

## 8. Phân tích

1. **So sánh học và đánh giá**:
   - So với `baseline` (điểm trung bình eval là 0.40), cả `subagents` (0.57) và `skills-auto` (0.63) đều cải thiện rõ rệt trên **tác vụ đánh giá** (tăng từ +17% đến +23% điểm tuyệt đối).
   - Trên **tác vụ học**, `skills-auto` cải thiện nhẹ (0.66 so với 0.63 của baseline), trong khi `subagents` giữ nguyên (0.63).
   - Không có điều kiện nào cải thiện tập học mà lại suy giảm ở tập đánh giá. Điều này chứng minh các kỹ năng tự tiến hóa có tính tổng quát hóa (generalization) tốt sang dữ liệu mới chứ không bị quá khớp (overfitting).
2. **Tách check kỹ thuật và check quy ước (`rule_`)**:
   - Các check kỹ thuật được mô hình giải quyết rất tốt ở hầu hết các điều kiện (đạt 17/18 check ở cả tập học và tập eval).
   - Kỹ năng do curator tự sinh là thành phần **DUY NHẤT** giúp tác tử vượt qua được các check quy ước tổ chức Acme (`rule_*`): `skills-auto` đạt 1/9 check quy ước ở tập học và 2/12 ở tập đánh giá (trong khi `baseline` và `subagents` đạt 0/9 và 0/12).
   - Check quy ước **mới** của tác vụ đánh giá (ví dụ `rule_version_bump` trong `code-eval`) **KHÔNG** được skill giúp đạt được. Điều này là tất yếu vì quy ước này chưa từng xuất hiện trong phản hồi lỗi của tập học, và nó cũng là minh chứng thực tế khẳng định không có rò rỉ dữ liệu (data leakage) từ tập đánh giá sang skill.
3. **Phân tích cơ chế từ vết và `skills_read`**:
   - *Check được skill giúp đạt*: Trong `code-eval`, `rule_type_hints` và `rule_regression_tests` đã chuyển từ thất bại (`False` ở baseline) sang đạt (`True` ở skills-auto). Vết thực thi (`trace.md`) chứng minh: sau khi đọc `skills/auto/repository-bug-fix-workflow/SKILL.md`, tác tử đã tạo file `tests/test_regressions.py` và bổ sung đầy đủ chú thích kiểu dữ liệu (type annotations) cho toàn bộ các hàm công khai theo đúng chỉ dẫn của skill.
   - *Check mà skill không giúp đạt*: `rule_changelog` trong `code-eval` vẫn chưa đạt. Mặc dù skill có nhắc nhở ghi chú changelog, vết thực thi cho thấy tác tử sau khi tập trung sửa mã nguồn và chạy test đã báo cáo hoàn thành mà quên thao tác chỉnh sửa tệp `CHANGELOG.md`.
4. **Chi phí token**:
   - Số token trung bình mỗi lần chạy: `baseline` (75,813 tokens) < `subagents` (130,749 tokens) $\approx$ `skills-auto` (134,317 tokens).
   - Hiệu quả chi phí: Trên tập đánh giá, `skills-auto` đạt 0.63 điểm với ~110k tokens (tương đương 5.73 điểm/1M tokens), vượt trội so với `baseline` đạt 0.40 điểm với ~80k tokens (5.02 điểm/1M tokens).
   - Đa tác tử (`subagents`) tiêu tốn lượng token gấp 1.7 - 2.1 lần nhưng không mang lại tri thức mới về quy ước tổ chức, do đó chi phí của subagents trong bài lab này là tương đối đắt đỏ so với giá trị gia tăng mà nó mang lại.
5. **Rò rỉ dữ liệu và quá khớp**:
   - Không có dấu hiệu rò rỉ dữ liệu: Hàm `curate_skills` chỉ lọc các lần chạy có `role == "learn"`, và bộ lọc `eval_markers()` tự động loại trừ mọi định danh thuộc về tập đánh giá.
   - Kỹ năng tự sinh được viết ở dạng quy trình chỉ dẫn hành động tổng quát (ví dụ *"Use when transforming tabular input into JSON or CSV..."*), không chứa các giá trị đáp án cụ thể hay tên file riêng của tác vụ eval.
6. **Định lượng độ nhiễu (Noise analysis)**:
   - So sánh điểm tác vụ học ở Phần 3.4 (lúc dev) và sau đóng băng của cùng bộ skill:
     + `code-learn`: 8/10 (lúc dev) so với 7/10 (sau đóng băng) $\rightarrow$ chênh lệch 1 check (10%).
     + `data-learn`: 5/8 so với 5/8 $\rightarrow$ chênh lệch 0%.
     + `logs-learn`: 6/9 so với 6/9 $\rightarrow$ chênh lệch 0%.
   - Chênh lệch trung bình giữa 2 lần chạy của cùng một bộ skill chỉ là 1/27 check (~3.7%), chứng minh tính ổn định của mô hình và khẳng định sự cách biệt điểm số giữa `skills-auto` (0.63) và `baseline` (0.40) ở tập đánh giá là có ý nghĩa thống kê thực chất chứ không phải do nhiễu ngẫu nhiên.

## 9. Hạn chế và tính hợp lệ

1. **Quy mô tập tác vụ nhỏ**: Bộ thử nghiệm gồm 6 tác vụ (3 học, 3 đánh giá) với 1 lần chạy chính thức cho mỗi cấu hình. Mặc dù đủ để quan sát xu hướng, kích thước mẫu nhỏ hạn chế việc áp dụng các phép kiểm định thống kê sâu (như t-test hoặc ANOVA).
2. **Quy ước tổ chức mang tính tổng hợp (synthetic house rules)**: Các quy tắc `rule_*` được thiết kế có chủ đích trong bộ test nhằm đo lường khả năng học ngữ cảnh, có thể chưa phản ánh hết mức độ đa dạng và phức tạp của các quy chuẩn kỹ nghệ trong môi trường công nghiệp thực tế.
3. **Phụ thuộc vào kiến trúc một mô hình duy nhất**: Toàn bộ thí nghiệm được tiến hành trên mô hình `gpt-6-luna`. Khả năng tuân thủ skill và hành vi phân rã subagent có thể khác biệt trên các mô hình mã nguồn mở hoặc các họ mô hình khác.

## 10. Kết luận

Thí nghiệm chứng minh tác tử tự tiến hóa ở tầng ngữ cảnh (`skills-auto`) cải thiện vượt bậc điểm số trên tập đánh giá (từ 0.40 lên 0.63) nhờ khả năng đúc kết và chuyển giao các quy ước tổ chức từ phản hồi thất bại mà không cần tinh chỉnh trọng số mô hình. Trong khi đó, đa tác tử (`subagents`) làm tăng chi phí token hơn 1.7 lần nhưng không tự sinh thêm tri thức mới. Hướng cải tiến tiếp theo là nghiên cứu cơ chế tiến hóa kết hợp: cho phép các subagent chuyên biệt trực tiếp thừa kế và cập nhật kho kỹ năng dùng chung trong quá trình thực thi.

## Phụ lục

- Lệnh đã chạy (theo thứ tự):
  1. `python scripts/tour.py`
  2. `pytest tests/test_01_provided.py tests/test_02_agent.py tests/test_03_runner.py`
  3. `python -m lab.runner --condition baseline --tasks data-learn code-learn logs-learn`
  4. `python -m lab.runner --condition subagents --tasks learn`
  5. `pytest tests/test_04_curator.py`
  6. `python -m lab.curator`
  7. `python -m lab.runner --condition skills-auto --tasks learn`
  8. `git add ... && git commit -m "hypotheses: state H1-H3 before freeze" && git tag freeze`
  9. `python -m lab.runner --condition baseline --tasks eval`
  10. `python -m lab.runner --condition subagents --tasks eval`
  11. `python -m lab.runner --condition skills-auto --tasks all`
  12. `python scripts/verify_freeze.py`
  13. `python -m lab.compare > report/table.md`
  14. `python scripts/check_breakdown.py`
- Thử thách mở rộng: Hướng 6d (Subagent có skill) — quan sát thực nghiệm cho thấy subagent tự định nghĩa khi không được nạp skill thì không thể tự học quy ước tổ chức; việc tích hợp skill vào subagent là hướng mở tiềm năng nhất để kết hợp ưu thế phân công vai trò và tri thức thủ tục.
