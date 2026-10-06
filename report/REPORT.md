# Báo cáo Lab: Self evolving Agentic

## 1. Thông tin nhóm và cấu hình

| Họ tên | Mã sinh viên | Phần đóng góp |
|---|---|---|
| Ngọ Doãn Ngọc|2A202602635 | All|

- Mô hình (tên deployment hoặc `LAB_MODEL`), nhiệt độ (`LAB_TEMPERATURE`), `recursion_limit`: `LAB_MODEL=google_genai:gemini-3.1-flash-lite` (Gemini API, gói miễn phí), `LAB_TEMPERATURE=0`, `recursion_limit=60` (mặc định của `lab.runner`). Lý do chọn mô hình: DeepSeek hết số dư (402); `gemini-3.8-flash` hết hạn mức 20 request/ngày của gói miễn phí (429); `gemini-3.5-flash` và `gemini-3.7-flash` liên tục trả 503 (quá tải); `gemini-3.6-flash` và `gemini-3.5-flash-lite` bỏ qua `temperature`. Mọi lần chạy tính điểm đều dùng cùng một mô hình `gemini-3.1-flash-lite`.
- Phiên bản Deep Agents (`pip show deepagents`), hệ điều hành, chạy trực tiếp hay trong Docker: `deepagents==0.7.21`, `langchain-google-genai==4.4.0`. Máy chủ Windows 11; mọi lần chạy tác tử và `pytest` chạy **trong Docker** (image `python:3.12-slim` dựng từ `Dockerfile` của lab, cộng thêm `langchain-google-genai`), vì trên Windows `LocalShellBackend` dùng `cmd.exe` thay cho `/bin/sh`. Lệnh: `docker run --rm --env-file .env -v <repo>:/lab lab-deepagents-gemini python -m lab.runner ...`. `verify_freeze.py`, `check_breakdown.py` chạy trên máy chủ (cần `git`).
- Số lần chạy tác vụ đã dùng / ngân sách: 18 lần chạy chính thức (3 điều kiện × 6 tác vụ) + 3 lần `skills-auto` trên tác vụ học ở Phần 3.4 + 2 lần gọi curator. Ngoài ra 6 lần chạy tác vụ học đầu tiên bị loại vì lỗi hạ tầng CRLF (xem Phụ lục) và 3 lần chạy thử `data-learn` lỗi API (429/503) không tính.
- Commit của tag `freeze`: (điền sau Phần 4.1)

## 2. Giả thuyết (commit TRƯỚC tag `freeze`, Phần 4.0)

Dự đoán chung: trên tác vụ đánh giá, **không điều kiện nào vượt rõ rệt `baseline`**; `skills-auto` có khả năng cao nhất nhưng chênh lệch dự kiến ≤ 1 check mỗi tác vụ, nằm trong mức nhiễu.

- H1 (subagents so với baseline): Điểm `subagents` trên tác vụ đánh giá **bằng `baseline` trong khoảng ±1 check mỗi tác vụ**, check quy ước vẫn ~0, nhưng token trung bình **cao hơn ít nhất 30%**. Căn cứ: ở tác vụ học, `subagent_calls` = 0/1/0 (chỉ giao việc 1 lần), điểm tổng bằng nhau (16/27 so với 16/27) còn token +48% (156.154 so với 105.862); lời giao việc không chứa quy ước Acme vì đề không có (mục 5). Tài liệu (Anthropic, multi-agent research system) ghi nhận đa tác tử tốn token hơn nhiều lần, lợi ích chủ yếu ở tác vụ cần song song hóa, không phải tác vụ tuần tự ngắn như ở đây.
- H2 (skills-auto so với baseline): `skills-auto` **cao hơn `baseline` tối đa 1–3 check quy ước trên tổng 3 tác vụ đánh giá** (chỉ ở họ `code`/`logs`, nơi skill chép đúng quy ước cũ), check kỹ thuật không đổi, và **không giúp check quy ước mới** của tác vụ đánh giá. Căn cứ: 9/11 lỗi ở tác vụ học là nhóm E, nên skill nhắm đúng chỗ; nhưng ở Phần 3.4 `skills_read` chỉ 1/3 (đọc nhầm skill) và check quy ước vẫn 0/9, nên khả năng skill được đọc và làm theo trên tác vụ mới thấp; họ `data` không có skill. SkillsBench ghi nhận skill do mô hình tự sinh trung bình không có lợi.
- H3 (tác vụ học so với tác vụ đánh giá): Mức tăng của `skills-auto` so với `baseline` trên tác vụ học **lớn hơn hoặc bằng** mức tăng trên tác vụ đánh giá (dấu hiệu quá khớp), vì skill chỉ chứa quy ước của tác vụ học còn mỗi tác vụ đánh giá thêm một quy ước mới mà skill không thể biết. SkillEvolBench ghi nhận lợi ích trên tác vụ học thường không chuyển sang tác vụ mới. Mọi chênh lệch ≤ 2 check cần được coi là nhiễu, vì mỗi cấu hình chỉ chạy một lần và mô hình cho kết quả khác nhau giữa hai lần chạy cùng điều kiện, kể cả với `temperature=0` (`data-learn` baseline: `north_q1_revenue` = 4155.87 ở lần chạy bị loại so với 2374.22 ở lần chính thức; `data-learn` skills-auto Phần 3.4 đạt cả hai check `north_q1_*` dù không có skill nào nói về ngày tháng).

## 3. Làm quen Deep Agents (Phần 0.3)

1. Tác tử mặc định có 9 công cụ: công cụ tệp `ls`, `read_file`, `write_file`, `edit_file`, `delete`, `glob`, `grep`; công cụ shell `execute`; công cụ subagent `task`. Công cụ cho phép chạy lệnh là `execute` (chỉ hoạt động khi backend cài đặt `SandboxBackendProtocol`, nếu không sẽ trả về lỗi).
2. Mô tả của `task` nói `general-purpose` là subagent đa dụng để nghiên cứu câu hỏi phức tạp, tìm tệp/nội dung và thực thi tác vụ nhiều bước, "has access to all tools as the main agent"; nên dùng khi tìm kiếm mà không chắc tìm đúng sau vài lần thử. Về ngữ cảnh: mỗi lần gọi là stateless, subagent "sees only the prompt you give it and returns a single final report", tức không thấy hội thoại hay lịch sử của tác tử chính, chỉ thấy prompt được truyền vào; báo cáo của nó cũng không hiển thị cho người dùng.
3. System prompt mặc định là chuỗi rỗng (`''`), hành vi được định hướng qua mô tả công cụ:
   - `task`: "Put full detail in the prompt and state exactly what it should return".
   - `execute`: "You MUST avoid using search commands like find and grep. Instead use the grep, glob tools to search."

## 4. Đường cơ sở và phân loại lỗi (Phần 2.2)

Nguồn: `results/baseline/{code,data,logs}-learn/run.json` và `trace.md`. Điểm baseline: `code-learn` 7/10, `data-learn` 3/8, `logs-learn` 6/9 (tổng 16/27).

| Tác vụ | Check thất bại | Nhóm lỗi (A-G) | Bằng chứng (trích ngắn từ `detail` hoặc vết) |
|---|---|---|---|
| code-learn | `rule_type_hints` | E | `RULE: every public function ... has type annotations on all parameters and on the return value.` Đề chỉ nói "Acme Python team conventions", không nêu quy ước. |
| code-learn | `rule_regression_tests` | E | `RULE: add tests/test_regressions.py with one test function per bug you fixed (at least 3)`. Vết: tác tử tạo tệp test riêng rồi `delete`, không có `test_regressions.py`. |
| code-learn | `rule_changelog` | E | `RULE: record each fix in CHANGELOG.md under the heading '## Unreleased' as a bullet '- fix(<function name>): ...'`. Vết không có lệnh sửa `CHANGELOG.md`. |
| data-learn | `north_q1_revenue` | D | `wrong value (got 2374.22)`. Vết (`analyze.py`): `dateutil.parser.parse(date_str).replace(tzinfo=None)` bỏ offset thay vì đổi sang UTC, và `dateutil` đọc `DD/MM/YYYY` (ví dụ `09/02/2024`) theo kiểu tháng trước nên nhánh `strptime('%d/%m/%Y')` không bao giờ chạy, trái với README ("Three formats occur ... DD/MM/YYYY"). |
| data-learn | `north_q1_orders` | D (+B) | `wrong value (got 9)`. Cùng nguyên nhân ngày tháng; tác tử chỉ đọc lại `answer.json` rồi kết thúc, không đối chiếu vài dòng mẫu với quy tắc ngày của README (B). |
| data-learn | `rule_money_in_cents` | E | `RULE: money values in answer.json are integer cents (1606.67 USD is written 160667).` Đề ghi `north_q1_revenue (number)`. |
| data-learn | `rule_meta_block` | E | `RULE: answer.json has an object meta = {"source": ..., "rows_in": ..., "rows_used": ...}`. |
| data-learn | `rule_clean_csv` | E | `RULE: write workspace/clean.csv with the header order_id,timestamp_utc,region,amount_cents ...`. Không có trong đề. |
| logs-learn | `rule_service_names` | E | `RULE: service names in the output are lower-case with '-' replaced by '_' (payment-service -> payment_service).` |
| logs-learn | `rule_sorted_errors` | E | `RULE: errors is sorted by service, then by timestamp_utc, ascending.` |
| logs-learn | `rule_schema_header` | E | `RULE: the top-level object has "schema_version": 2 and "generated_by": "log-triage".` |
| code-learn | (không phải check) | G | Lần chạy dừng do `GraphRecursionError: Recursion limit of 60 reached` sau 30 tool call, `final_message` rỗng. Tác tử sửa xong mã (mọi check kỹ thuật đạt) nhưng tiếp tục viết, chạy rồi xóa tệp test phụ cho tới khi hết bước. |

Nhận xét:

- **Nhóm E chiếm đa số: 9/11 check thất bại** (100% check `rule_`, 0/9 đạt). Còn lại 2/11 là nhóm D (xử lý định dạng ngày/múi giờ) ở `data-learn`.
- **Bằng chứng phủ định cho A–D** (`python scripts/check_breakdown.py`): check kỹ thuật baseline đạt **16/18**, check quy ước đạt **0/9**. A (bỏ qua đặc tả): ở cả 3 tác vụ, lệnh đầu tiên sau `ls` là `read_file workspace/README.md` (data, logs) hoặc đọc test và mã nguồn (code); `low_stock_follows_docstring` và `csv_quoting_follows_docstring` đều đạt nên docstring đã được đọc. B: `visible_suite_passes` đạt, tác tử có chạy lại test. C (vá triệu chứng): `parse_price_all_formats` và `other_caller_fixed` đều đạt, tức đã sửa hàm dùng chung. F: không có câu trả lời cuối nào nói sai về tệp đã tạo (`code-learn` không có câu trả lời cuối vì hết bước).
- **Skill có phòng ngừa được nhóm E không?** Có, về nguyên tắc: quy ước E không thể suy ra từ đề, chỉ có trong `detail` của bot đánh giá; một skill chép lại chính xác quy ước đó là cách duy nhất để tác tử biết. Nhóm D cũng có thể được nhắc bằng một checklist "chuẩn hóa ngày về UTC trước khi lọc", nhưng mức độ giúp phụ thuộc vào việc tác tử có làm theo đúng không.

## 5. Điều kiện `subagents` (Phần 2.3)

- **Các subagent đã định nghĩa** (`src/lab/subagents.py`): `explorer` (chỉ đọc: README, docstring, test, mẫu dữ liệu; báo cáo đặc tả và các bất thường dữ liệu, không sửa tệp), `implementer` (thực hiện thay đổi, sửa nguyên nhân gốc, chạy test/script để kiểm chứng, báo cáo đúng tệp đã sửa), `reviewer` (kiểm tra độc lập kết quả theo từng yêu cầu và trường hợp biên, báo PASS/FAIL, không sửa). Lý do: tách ba giai đoạn hay hỏng nhất theo phân loại lỗi (đọc đặc tả → A/D; thực hiện → C; kiểm chứng → B/F). `description` của mỗi subagent viết dạng "Use BEFORE/AFTER ... Send it ALL the task rules ..." để tác tử chính biết lúc nào gọi và phải truyền gì.
- **`subagent_calls`**: `code-learn` 0, `data-learn` 1 (`implementer`), `logs-learn` 0. Hai tác vụ không giao việc: tác tử chính (mô hình nhẹ `flash-lite`) đi thẳng vào vòng làm-chạy (logs: `ls → read_file ×2 → write_file → execute → read_file ×3`, giống hệt baseline). Logs là việc viết một script duy nhất nên tác tử coi là "trivial"; code-learn là vòng sửa-chạy test ngắn mà tác tử tự làm. `SUBAGENTS_NOTE` chỉ khuyến khích, mô hình không bắt buộc làm theo; đây là kết quả hợp lệ.
- **Thông tin thiếu khi giao việc** (`data-learn`, trace): lời giao việc cho `implementer` liệt kê đủ 5 khóa và các bất thường từ README (trùng `order_id`, `-999`, chuẩn hóa vùng, ba định dạng ngày) nhưng **thiếu** (1) "date-only nghĩa là 00:00 UTC" và "đổi timestamp có offset sang UTC", (2) câu "plus whatever the Acme reporting conventions require" và thông tin có bot đánh giá. Báo cáo của subagent chỉ nêu kết quả (`north_q1_revenue: 3189.59`, `north_q1_orders: 10`). Tác tử chính **có kiểm tra** trước khi dùng: đọc lại `answer.json`, đọc 10 dòng CSV, tự tính lại bằng script `csv` thuần (pandas không có) rồi ghi lại `answer.json`. Kết quả: `north_q1_orders` đạt (baseline không đạt), `north_q1_revenue` vẫn sai, 3 check `rule_` vẫn sai vì quy ước không có trong đề lẫn lời giao việc.
- **Token và thời gian** (subagents so với baseline): `code-learn` 209.330 so với 195.652 token (+7%), 225,6 s so với 170,0 s; `data-learn` **200.821 so với 65.910 (×3,05)**, 189,2 s so với 97,5 s; `logs-learn` 58.312 so với 56.026 (+4%), 39,1 s so với 57,8 s. Trung bình 156.154 so với 105.862 token/lần (+48%). Điểm tổng tác vụ học không đổi: 16/27 so với 16/27 (code 6 so với 7, data 4 so với 3, logs 6 so với 6). Ở `code-learn` (subagents), tác tử chính lặp lại cùng một lệnh `edit_file` không đổi gì (`old_string == new_string`) trên tệp `tests/test_extra.py` cho tới khi hết `recursion_limit`; tệp test phụ đó còn lỗi nên `visible_suite_passes` thất bại (`1 failed, 8 passed`), đây là lỗi nhóm G (lặp), không liên quan tới subagent.

## 6. Self-evolving: skill do curator sinh (Phần 3)

- **Số lần chạy curator: 2** (lần đầu + 1 lần chạy lại; còn 1 lần chạy lại không dùng). Nguồn: `baseline` của 3 tác vụ học (11 check thất bại, phần cuối 6000 ký tự của mỗi vết). Không lần nào đọc dữ liệu tác vụ đánh giá.
- **Lần 1** sinh 3 skill hợp lệ về định dạng: `enforce-rule-compliance`, `implement-regression-testing`, `verify-data-integrity` (bản lưu ở `report/curator-run1/` làm bằng chứng). **Cả 3 bị xóa**, lý do:
  1. `enforce-rule-compliance` bước 1 hướng dẫn sai: "extract all "RULE:" requirements from the prompt", trong khi quy ước Acme **không có** trong đề, nên tác tử sẽ không tìm thấy gì; skill cũng không nêu tên tệp bắt buộc `clean.csv`.
  2. `verify-data-integrity` phát biểu quy ước thiếu ("Replace hyphens with underscores" nhưng bỏ phần chữ thường của `rule_service_names`) và dạng "verify" thay vì mệnh lệnh làm.
  3. Cả 3 skill có thẻ `<body>`/`</body>` thừa: do prompt curator của nhóm dùng placeholder `<body>` và mô hình chép nguyên văn.
  Trước khi chạy lại, nhóm sửa **prompt curator** (không sửa skill): nói rõ quy ước không có trong đề nên skill phải tự phát biểu đầy đủ, và đổi placeholder thành `<numbered instructions>`.
- **Lần 2** sinh 3 khối; `validate_skill` (có sẵn) **từ chối** `data-processing-standards` với lý do `mentions evaluation material: orders`. Từ "orders" xuất phát từ chính `detail` của tác vụ học ("number of distinct orders with a known amount"), nên đây là dương tính giả của bộ lọc rò rỉ; nhóm **không** chỉnh prompt để né từ này, vì như vậy là dùng thông tin về tác vụ đánh giá để định hướng curator. Hệ quả: họ `data` không có skill. Giữ lại 2 skill bên dưới, không sửa tay.

| Skill | Tổng quát hay riêng cho tác vụ học? | Đúng hay sai (nêu chỗ sai nếu có) | Độ dài, `description` và `skills_read` ở Phần 3.4 |
|---|---|---|---|
| `code-quality-and-compliance` | Tổng quát cho họ `code`: không nêu tên hàm, tệp hay gói của tác vụ học (`inventory`, `parse_price` không xuất hiện); chỉ nêu tên do quy ước yêu cầu (`tests/test_regressions.py`, `CHANGELOG.md`, `## Unreleased`). | Đúng và đủ: khớp từng chữ với 3 `detail` (`rule_type_hints`, `rule_regression_tests`, `rule_changelog`). Không có hướng dẫn gây hại; bước 3 "Ensure all tests pass by running pytest" còn hỗ trợ check kỹ thuật. | 5 dòng, gọn. `description`: "Use when writing or modifying code to ensure it meets project-wide standards and documentation requirements." Bắt đầu bằng "Use when", phạm vi rộng hợp lý. `skills_read` ở `code-learn` = **0**: không được đọc, 3 check quy ước vẫn trượt. |
| `log-analysis-compliance` | Tổng quát cho họ `logs`: không nêu tên tệp log (`app.log`) hay tên service của dữ liệu; ví dụ `payment_service` lấy từ chính `detail`. | Đúng và đủ: khớp 3 `detail` (`rule_service_names` gồm cả chữ thường và `-`→`_`, `rule_sorted_errors`, `rule_schema_header`). Không có hướng dẫn gây hại. | 4 dòng. `description`: "Use when parsing log files to ensure the output JSON structure and naming conventions are correct." Nêu đúng tình huống kích hoạt. `skills_read` ở `logs-learn` = **0** (vết: `ls → README → app.log → write_file ...`, không đọc skill). Ngược lại ở `data-learn` tác tử đọc nhầm skill này (`skills_read=1`) rồi bỏ qua vì không liên quan. |

Nhận xét Phần 3.4: điểm `skills-auto` trên tác vụ học là code 7/10, data 5/8, logs 6/9 (tổng 18/27 so với 16/27 của baseline), nhưng **check quy ước vẫn 0/9**: skill không được đọc ở đúng tác vụ, nên chênh lệch +2 nằm ở check kỹ thuật `north_q1_*` của `data-learn`, nơi không có skill nào nói về ngày tháng. Chênh lệch đó do nhiễu của mô hình, không phải do skill. Hướng dẫn gợi ý chạy lại curator khi `skills_read = 0`, nhưng ở đây `description` đã đúng tình huống; nguyên nhân là mô hình nhẹ bỏ qua câu "As your FIRST action, read the SKILL.md ..." trong `SKILLS_NOTE`. Chạy lại curator khó sửa được điều này và có thể làm mất 2 skill đúng, nên nhóm dừng ở lần 2.

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
