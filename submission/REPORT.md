# K4-Track02-Day17 — Report cá nhân

Phần phân tích tối đa một trang, không tính output ở phần 5.
Định dạng tham chiếu và phạm vi tính trang: [SUBMISSION.md](../docs/SUBMISSION.md).

**Họ tên / MSSV:** Phạm Đình Duy / 2A202602913
**Repo:** https://github.com/Dzzuy/K4-Track02-Day17-PhamDinhDuy-2A202602913-DataPipelineEngineering
**Commit bài nộp:** 1944df52059f2933020dd18e22ffb9f4da5bb57d
**AI đã dùng và phạm vi hỗ trợ (hoặc không dùng):** Tôi dùng Codex đểđọc code, đề xuất sửa, tôi chỉnh sửa plan, duyệt và review lại code, chạy kiểm tra, review output, kiểm tra bằng chứng, soạn bản nháp REPORT, Codex viết và tôi đọc và chỉnh sửa lại.
**Nguồn tham khảo khác (nếu có):** README, hướng dẫn và code của repo đề bài.

## 1. Ba lỗi

Mỗi lỗi 4 dòng. Triệu chứng = thứ bạn *thấy* đầu tiên (check nào fail, số nào lạ,
checksum nào lệch) — không phải cách sửa.

| | Lỗi Silver | Lỗi late data | Lỗi xoá (CDC) |
|---|---|---|---|
| **Triệu chứng** | Verify 8/18: 24 hàng cho 12 ticket; T-91 có ba trạng thái. | u05 ngày 12/08 có (2 event, 1 click, 0 feedback down), thay vì (5, 3, 1); Gold lệch full recompute. | T-97 còn trong Silver, snapshot mới nhất và hai RAG chunk. |
| **Nguyên nhân gốc** | `silver.py` chỉ `INSERT`; lọc trùng trong batch không xử lý trùng giữa các ngày. | Lookback bằng 0 nên event đến muộn ba ngày không được tính lại. | Delete có `after=null` nhưng `staging.py` chỉ đọc khoá từ `after`. |
| **Cách sửa** (file, vài dòng) | `pipeline/silver.py`: `MERGE` theo `ticket_id`, chỉ nhận LSN mới hơn. | `pipeline/config.py`: đặt `LOOKBACK_DAYS=3` theo P99 Bronze; Gold nhóm theo event time. | `pipeline/staging.py`: lấy khoá từ `after` hoặc `before`, bỏ Kafka tombstone rỗng. |
| **Khái niệm trên slide** | Silver keyed upsert, LSN guard, idempotency. | Late data, event time, overwrite partition. | Log-based CDC, tombstone, xoá lan xuống Gold. |

## 2. Các con số

- P99 lateness đo từ Bronze: `3.00` ngày → `LOOKBACK_DAYS = 3`
- `submission/checksums.txt`: PASS — Gold checksum: `39e115c510ecdf526800eac227158a4f`
- `make parity`: PARITY

## 3. Lựa chọn công cụ / kỹ thuật (mỗi dòng một câu "vì sao")

- MERGE theo khoá cho `silver_tickets`, overwrite-partition cho `gold_feature_daily`: Ticket cần một trạng thái hiện tại; feature theo ngày cần tính lại các partition trong lookback.
- Tombstone thay vì xoá hẳn hàng trong Silver: Giữ LSN để replay cũ không làm ticket sống lại, nhưng xoá các cột cá nhân.
- Snapshot training dựng lại từ Bronze "as of" ngày đó, không sửa snapshot cũ: Version cũ cần tái lập được; dữ liệu mới tạo version mới.
- DuckDB (lite) / dbt (track dbt) cho bài toán cỡ này, chứ không phải Spark: Seed nhỏ chạy cục bộ là đủ; dbt kiểm tra MERGE và microbatch mà không thêm hạ tầng Spark.

## 4. Hai câu hỏi suy ngẫm

1. Snapshot `v2026-08-12`..`v2026-08-14` vẫn chứa văn bản của T-97 (đã bị xoá ngày
   08-15). "Snapshot bất biến" và "quyền được xoá dữ liệu" mâu thuẫn — bạn xử lý thế nào?
   Mình sẽ ngừng phục vụ, xoá hoặc dựng lại bản dữ liệu và bản sao chứa T-97, rồi kiểm tra còn ID/PII không. Giữ metadata version để audit, không giữ nội dung cá nhân chỉ vì snapshot bất biến. Lab mới xử lý snapshot gần nhất và RAG hiện hành.
2. Regex che được email và số điện thoại, nhưng tên "Nguyễn Văn An" vẫn còn. Bạn sẽ
   đặt chốt PII nào, ở tầng nào, và đo nó ra sao?
   Trước Silver/Gold, mình thêm nhận diện tên tiếng Việt bên cạnh regex email/điện thoại và duyệt ca không chắc. Đo recall, PII lọt qua, false positive và tỷ lệ quarantine trên tập gán nhãn.

## 5. Output (dán nguyên văn)

```text
$ make verify
=== verify.py — Day 17 pipeline contracts ===
  [OK ] Bronze  every daily batch landed as Parquet (7 days x 3 sources)
  [OK ] Bronze  re-landing a batch is a no-op (append-only, no duplicate file)
  [OK ] Bronze  Bronze keeps the raw truth: Kafka tombstone + redelivered events are still there
  [OK ] Silver  silver_tickets has exactly one row per ticket_id
  [OK ] Silver  T-91 shows its latest state: high / closed / bug
  [OK ] Silver  deleted ticket T-97 is a tombstone: is_deleted and no personal data left
  [OK ] Silver  no email / phone number survives past Bronze
  [OK ] Silver  silver_events has one row per event_id (Kafka redeliveries removed)
  [OK ] Silver  2 malformed events quarantined with a reason; the run did not halt
  [OK ] Gold    gold_feature_daily reconciles with a full recompute from Silver
  [OK ] Gold    u05's offline events of 08-12 (arrived 08-15) are counted on 08-12
  [OK ] Gold    LOOKBACK_DAYS covers measured P99 lateness (p99=3.00 days)
  [OK ] Gold    training set uses point-in-time priority (T-91 created as 'low')
  [OK ] Gold    late feedback creates a NEW snapshot version; the old one is untouched
  [OK ] Gold    latest training snapshot excludes the deleted ticket T-97
  [OK ] Gold    deletes propagate to the RAG index: no chunk of T-97
  [OK ] Gold    gold_doc_chunks: one row per chunk, and a re-run embeds 0 new chunks
  [OK ] Rerun   re-run 2026-08-12 three times -> Gold checksum identical to a fresh build

RESULT: 18/18 checks — ALL PASS
re-run checksums written to submission/checksums.txt

$ make test
..................................                                       [100%]
34 passed in 1.30s

$ make rerun3
# Lab 17 — re-run check for 2026-08-12

run                     gold_feature_daily    gold_training_set     gold_doc_chunks       gold (combined)
fresh build             8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f
re-run #1 of 2026-08-12 8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f
re-run #2 of 2026-08-12 8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f
re-run #3 of 2026-08-12 8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f

RESULT: PASS — 3 re-runs, identical checksums

$ make lateness
event lateness over 43 Bronze records (calendar days): p50=0.00 p95=2.90 p99=3.00 max=3
-> lookback must be >= ceil(p99) = 3 day(s); config.LOOKBACK_DAYS = 3

$ make dbt
cd dbt_project && DBT_PROFILES_DIR=. /home/celosia/Tom/VSCode/VinAI/K4-Track02-Day17-Data-Pipeline-Engineering/.venv/bin/dbt build --event-time-start 2026-08-10 --event-time-end 2026-08-17
02:33:54  Running with dbt=1.12.5
02:33:55  Registered adapter: duckdb=1.11.0
02:33:55  Found 5 models, 13 data tests, 2 sources, 502 macros, 1 unit test
02:33:55
02:33:55  Concurrency: 1 threads (target='dev')
02:33:55
02:33:55  1 of 19 START sql view model main.stg_events ................................... [RUN]
02:33:55  1 of 19 OK created sql view model main.stg_events .............................. [OK in 0.06s]
02:33:55  2 of 19 START sql view model main.stg_ticket_changes ........................... [RUN]
02:33:55  2 of 19 OK created sql view model main.stg_ticket_changes ...................... [OK in 0.03s]
02:33:55  3 of 19 START sql incremental model main.silver_events ......................... [RUN]
02:33:55  3 of 19 OK created sql incremental model main.silver_events .................... [OK in 0.09s]
02:33:55  4 of 19 START unit_test silver_tickets::silver_tickets_latest_change_wins_and_delete_is_tombstone  [RUN]
02:33:55  4 of 19 PASS silver_tickets::silver_tickets_latest_change_wins_and_delete_is_tombstone  [PASS in 0.09s]
02:33:55  8 of 19 START sql incremental model main.silver_tickets ........................ [RUN]
02:33:55  8 of 19 OK created sql incremental model main.silver_tickets ................... [OK in 0.10s]
02:33:55  5 of 19 START test not_null_silver_events_event_id ............................. [RUN]
02:33:55  5 of 19 PASS not_null_silver_events_event_id ................................... [PASS in 0.03s]
02:33:55  6 of 19 START test not_null_silver_events_user_id .............................. [RUN]
02:33:55  6 of 19 PASS not_null_silver_events_user_id .................................... [PASS in 0.01s]
02:33:55  7 of 19 START test unique_silver_events_event_id ............................... [RUN]
02:33:56  7 of 19 PASS unique_silver_events_event_id ..................................... [PASS in 0.01s]
02:33:56  9 of 19 START test accepted_values_silver_tickets_category__bug__billing__other  [RUN]
02:33:56  9 of 19 PASS accepted_values_silver_tickets_category__bug__billing__other ...... [PASS in 0.02s]
02:33:56  10 of 19 START test accepted_values_silver_tickets_priority__low__medium__high . [RUN]
02:33:56  10 of 19 PASS accepted_values_silver_tickets_priority__low__medium__high ....... [PASS in 0.02s]
02:33:56  11 of 19 START test accepted_values_silver_tickets_status__open__pending__closed  [RUN]
02:33:56  11 of 19 PASS accepted_values_silver_tickets_status__open__pending__closed ..... [PASS in 0.02s]
02:33:56  12 of 19 START test not_null_silver_tickets__lsn ............................... [RUN]
02:33:56  12 of 19 PASS not_null_silver_tickets__lsn ..................................... [PASS in 0.01s]
02:33:56  13 of 19 START test not_null_silver_tickets_is_deleted ......................... [RUN]
02:33:56  13 of 19 PASS not_null_silver_tickets_is_deleted ............................... [PASS in 0.01s]
02:33:56  14 of 19 START test not_null_silver_tickets_ticket_id .......................... [RUN]
02:33:56  14 of 19 PASS not_null_silver_tickets_ticket_id ................................ [PASS in 0.01s]
02:33:56  15 of 19 START test unique_silver_tickets_ticket_id ............................ [RUN]
02:33:56  15 of 19 PASS unique_silver_tickets_ticket_id .................................. [PASS in 0.01s]
02:33:56  16 of 19 START sql microbatch model main.gold_feature_daily .................... [RUN]
02:33:56  Batch 1 of 7 START batch 2026-08-10 of main.gold_feature_daily ....................... [RUN]
02:33:56  Batch 1 of 7 OK created batch 2026-08-10 of main.gold_feature_daily .................. [OK in 0.03s]
02:33:56  Batch 2 of 7 START batch 2026-08-11 of main.gold_feature_daily ....................... [RUN]
02:33:56  Batch 2 of 7 OK created batch 2026-08-11 of main.gold_feature_daily .................. [OK in 0.02s]
02:33:56  Batch 3 of 7 START batch 2026-08-12 of main.gold_feature_daily ....................... [RUN]
02:33:56  Batch 3 of 7 OK created batch 2026-08-12 of main.gold_feature_daily .................. [OK in 0.02s]
02:33:56  Batch 4 of 7 START batch 2026-08-13 of main.gold_feature_daily ....................... [RUN]
02:33:56  Batch 4 of 7 OK created batch 2026-08-13 of main.gold_feature_daily .................. [OK in 0.02s]
02:33:56  Batch 5 of 7 START batch 2026-08-14 of main.gold_feature_daily ....................... [RUN]
02:33:56  Batch 5 of 7 OK created batch 2026-08-14 of main.gold_feature_daily .................. [OK in 0.02s]
02:33:56  Batch 6 of 7 START batch 2026-08-15 of main.gold_feature_daily ....................... [RUN]
02:33:56  Batch 6 of 7 OK created batch 2026-08-15 of main.gold_feature_daily .................. [OK in 0.02s]
02:33:56  Batch 7 of 7 START batch 2026-08-16 of main.gold_feature_daily ....................... [RUN]
02:33:56  Batch 7 of 7 OK created batch 2026-08-16 of main.gold_feature_daily .................. [OK in 0.02s]
02:33:56  16 of 19 OK created sql microbatch model main.gold_feature_daily ............... [SUCCESS in 0.17s]
02:33:56  17 of 19 START test dbt_utils_free_unique_combination_gold_feature_daily_user_id__event_date  [RUN]
02:33:56  17 of 19 PASS dbt_utils_free_unique_combination_gold_feature_daily_user_id__event_date  [PASS in 0.02s]
02:33:56  18 of 19 START test not_null_gold_feature_daily_event_date ..................... [RUN]
02:33:56  18 of 19 PASS not_null_gold_feature_daily_event_date ........................... [PASS in 0.01s]
02:33:56  19 of 19 START test not_null_gold_feature_daily_user_id ........................ [RUN]
02:33:56  19 of 19 PASS not_null_gold_feature_daily_user_id .............................. [PASS in 0.01s]
02:33:56
02:33:56  Finished running 3 incremental models, 13 data tests, 1 unit test, 2 view models in 0 hours 0 minutes and 0.85 seconds (0.85s).
02:33:56
02:33:56  Completed successfully
02:33:56
02:33:56  Done. PASS=19 WARN=0 ERROR=0 SKIP=0 NO-OP=0 REUSED=0 TOTAL=19

$ make parity
=== parity: lite pipeline vs dbt ===
  [OK ] silver_tickets       lite 3c15dfd43701  dbt 3c15dfd43701
  [OK ] gold_feature_daily   lite 8630e04a61d1  dbt 8630e04a61d1
RESULT: PARITY — both implementations agree

$ make bonus-llm
=== bonus: LLM labelling of 11 live tickets ===
  cost estimate before running: ~484 tokens = $0.0010 per full run
  [OK ] first run labels every live ticket
  [OK ] re-run with same model + prompt makes 0 LLM calls
  [OK ] every Gold label is bug / billing / other
  [OK ] off-schema answers go to llm_label_quarantine
  [OK ] new prompt version re-labels on purpose
  [OK ] labels carry their prompt version
BONUS PASS
```

Nếu dùng PowerShell, ghi lệnh tương đương và output thực tế theo [SUBMISSION.md](../docs/SUBMISSION.md).
Nếu làm bonus, thêm output B1 hoặc đường dẫn bằng chứng B2 ở cuối phần này.

Bằng chứng bonus B2: [bonus/DESIGN.md](../bonus/DESIGN.md). Prototype `make flywheel` đã chạy thành công.
