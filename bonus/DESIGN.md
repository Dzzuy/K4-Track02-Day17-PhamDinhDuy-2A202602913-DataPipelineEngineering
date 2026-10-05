# Bonus B2 — Từ hội thoại chatbot đến dữ liệu đánh giá đáng tin

**Phạm Đình Duy — 2A202602913**

## Bài toán tôi chọn

Mình muốn thiết kế pipeline dữ liệu cho một chatbot hỗ trợ khách hàng của sản phẩm SaaS ở Việt Nam. Bot trả lời chuyện đăng nhập, thanh toán và sử dụng tài khoản. Những câu khó sẽ được chuyển cho nhân viên. Sau mỗi lần đổi prompt hoặc model, nhóm sản phẩm cần trả lời hai câu hỏi: bot có tốt hơn không, và những ca nào có thể dùng để cải thiện bot ở lần tiếp theo?

Dữ liệu trông có vẻ sẵn: trace của agent, feedback của khách và ticket của nhân viên. Nhưng một lượt được hệ thống ghi `success` chỉ nói rằng chương trình chạy xong; nó không bảo đảm câu trả lời đúng. Khách cũng thường bỏ đi mà không bấm 👍 hay 👎. Feedback có thể đến sau vài ngày, ticket có thể được sửa hoặc xoá, và một câu hỏi có thể được diễn đạt theo nhiều kiểu tiếng Việt. Vì vậy, mình muốn làm rõ đường đi của dữ liệu trước khi chọn công cụ. Mục tiêu 24 giờ cho báo cáo chất lượng và các ngưỡng bên dưới là giả định thiết kế, chưa phải số liệu từ sản phẩm đang vận hành.

## Đường đi dự kiến

```text
Agent traces ─┐
Feedback ─────┼──> Bronze (payload + thời điểm + ID nguồn)
Ticket CDC ───┘                   │
                                ▼
                    Silver (chuẩn hoá, PII gate, quarantine)
                                │
                    ghép dữ liệu đúng thời điểm xảy ra
                                │
                    ┌───────────┴───────────┐
                    ▼                       ▼
             Eval holdout đã duyệt     Ứng viên dữ liệu train
                    │                       │
                    └──── kiểm tra trùng ───┘
                                │
                    Manifest, version, checksum
                                │
                       Đánh giá trước release
```

Một sổ theo dõi yêu cầu xoá (`erasure ledger`) đi qua cả Bronze, Silver, hai bộ dữ liệu Gold và các bản export. Nó không chứa lại nội dung khách hàng, nhưng phải đủ ID để tìm và thu hồi bản sao liên quan.

## Sáu quyết định cần chốt

### 1. Nhận dữ liệu ở hình dạng nào?

Mình sẽ yêu cầu trace có `trace_id`, `session_id`, thời điểm sự kiện, thời điểm ingest, model và prompt version. Feedback nối với trace bằng ID, còn ticket CDC giữ thao tác và LSN. Bronze giữ payload gốc kèm schema version; Silver mới chuyển chúng thành các bảng span, lượt hội thoại, feedback và ticket. Bản ghi thiếu ID hoặc timestamp hợp lệ sẽ vào quarantine, kèm lý do để người phụ trách nguồn dữ liệu kiểm tra.

Có thể ép cả ba nguồn vào một schema ngay lúc nhận để truy vấn dễ hơn. Mình không chọn cách đó: khi agent thêm công cụ mới hoặc format trace thay đổi, những trường chưa biết rất dễ biến mất. Giữ raw payload tốn chỗ và đòi hỏi phân quyền chặt hơn, nhưng cho phép sửa parser rồi chạy lại. Với dữ liệu khách hàng thật, “Bronze bất biến” không thể là lý do giữ PII mãi mãi; yêu cầu xoá phải xử lý cả bản gốc theo chính sách lưu giữ.

### 2. Cần streaming không?

Tập eval và ứng viên train có thể cập nhật mỗi đêm. Pipeline sẽ lấy dữ liệu theo event time và tính lại một cửa sổ lookback dựa trên độ trễ feedback đo được, giống cách lab xử lý event đến muộn. Dashboard báo lỗi nghiêm trọng có thể đọc log gần thời gian thực, nhưng đó là luồng vận hành riêng.

Streaming toàn bộ flywheel cho kết quả nhanh hơn, đổi lại phải quản lý state, duplicate và watermark. Trong khi đó, feedback và nhãn do nhân viên duyệt không xuất hiện ngay sau khi bot trả lời. Với mục tiêu báo cáo trong 24 giờ, batch ít thành phần hơn và dễ chạy lại hơn. Nếu sau này cần cảnh báo trong vài phút, mình chỉ đưa phần cảnh báo sang streaming; không chuyển cả pipeline huấn luyện khi chưa có lợi ích đo được.

### 3. Khi nào một hội thoại đủ tin để dùng?

Ở Silver, mình kiểm tra ID, timestamp, quan hệ span cha–con, phiên bản model/prompt và giá trị feedback. Mình theo dõi tỷ lệ quarantine theo từng nguồn và schema version; nếu tăng đột ngột, dataset của ngày đó chưa được xuất bản và người sở hữu connector phải xem lại. Eval holdout chỉ chứa những ca có câu trả lời tham chiếu được người duyệt xác nhận. Các ca khác là ứng viên train, có nguồn gốc và mức tin cậy rõ ràng.

Lấy mọi `status=ok` hoặc lượt 👍 làm nhãn đúng sẽ rẻ và có nhiều mẫu. Mình không tin hai tín hiệu đó đến mức ấy: bot có thể trả lời sai nhưng vẫn chạy thành công, còn khách không phải lúc nào cũng phản hồi. Duyệt tay tất cả lại quá tốn công. Mình sẽ ưu tiên duyệt holdout và các ca rủi ro cao, đồng thời lấy một mẫu ngẫu nhiên để biết chất lượng chung. Tạm đặt ngưỡng dừng xuất bản khi hơn 1% bản ghi lỗi contract hoặc còn bất kỳ PII nào chưa xử lý; ngưỡng 1% phải được hiệu chỉnh sau khi có số liệu thật.

### 4. Làm sao tránh tự “học thuộc đề thi”?

Mình chốt eval holdout trước khi tạo train set và lưu danh sách mẫu trong manifest. Prompt trùng sau chuẩn hoá sẽ bị loại khỏi train. Sau đó cần tìm cả câu diễn đạt lại bằng n-gram hoặc embedding, rồi cho người xem các cặp không chắc chắn. Khi ghép ticket, feedback và tài liệu, mình lấy trạng thái *as of* lúc bot đã trả lời; thông tin đến sau không được giả vờ là đã có ở thời điểm phục vụ.

Cách dễ làm nhất là tạo hết preference pairs rồi chia ngẫu nhiên train/eval. Mình bác bỏ cách này vì hai câu cùng ý, hoặc hai lần hỏi trong cùng ticket, có thể rơi vào hai phía. Điểm eval lúc đó cao nhưng không phản ánh năng lực với câu hỏi mới. Chia theo nhóm ticket/ý định và theo thời gian làm tập train nhỏ hơn, đổi lại phép đánh giá đáng tin hơn. Kiểm tra trùng gần cũng có thể loại nhầm mẫu; mình sẽ đo tỷ lệ đó trên một tập được duyệt tay.

### 5. Chạy lại và xoá dữ liệu như thế nào?

Mỗi batch có ID; span và event được ghi theo khoá, ticket theo khoá cùng LSN. Mỗi phiên bản dataset có manifest ghi ngày dữ liệu, checksum đầu vào, commit code, schema và danh sách mẫu. Chạy lại cùng batch phải ra cùng checksum. Khi có yêu cầu xoá, erasure ledger giúp tìm mẫu trong eval/train còn đang dùng, cache và bản export. Sau khi xử lý, pipeline truy vấn lại để xác nhận không còn ID hoặc nội dung cần xoá trong các bản đang phục vụ.

Snapshot bất biến rất tiện để tái lập thí nghiệm, nhưng không được đứng trên quyền xoá. Mình giữ lịch sử *metadata* và đánh dấu manifest cũ là `revoked`; bản dữ liệu chứa PII phải được huỷ hoặc dựng lại. Xoá cứng ticket mà không giữ dấu thứ tự thay đổi cũng nguy hiểm: replay một batch cũ có thể làm ticket xuất hiện lại. Một tombstone chỉ giữ ID và LSN cho phép chống hồi sinh mà không giữ nội dung nhạy cảm.

### 6. Khi dữ liệu tăng 10 hoặc 100 lần, cái gì hỏng trước?

Mình sẽ đo số file Bronze, kích thước partition, thời gian ghép trace–feedback, số ca phải duyệt tay và số lượt gọi model. Nếu file nhỏ làm scan chậm ở 10 lần tải, gộp file và phân vùng theo ngày/sản phẩm. Nếu ở 100 lần tải việc quét lại toàn bộ lịch sử quá đắt, duy trì bảng đã dedup theo khoá và chỉ tính lại các ngày trong lookback. Đây là thứ tự xử lý dựa trên phép đo, không phải lời hứa rằng một công cụ cụ thể sẽ chịu được 100 lần tải.

Mình dự đoán chi phí nhân viên duyệt nhãn và LLM labelling sẽ đáng chú ý hơn tiền lưu các bảng nhỏ. Vì vậy, mình chọn mẫu ưu tiên ở những ca bot không chắc hoặc vừa gây lỗi, cache nhãn theo hash nội dung + model + prompt version, và ước lượng chi phí trước mỗi batch. Chọn mẫu kiểu này dễ làm tập train thiên về ca khó; manifest phải ghi cách lấy mẫu và vẫn giữ một mẫu ngẫu nhiên để đo chất lượng chung. Mình chưa đưa Spark, Kafka Streams hoặc vector database vào bản đầu tiên. DuckDB/dbt và batch theo ngày đủ đơn giản để kiểm chứng; chỉ thay công cụ khi một nút thắt cụ thể đã được đo.

## Thử nhanh bằng code có sẵn trong lab

`make flywheel` chạy các module trong `extensions/` để biến trace mẫu thành eval set và preference pairs. Lần chạy hiện tại có 21 spans từ 8 traces, tạo 2 mẫu eval và 3 cặp preference thô; sau bước loại trùng còn 1 cặp. Ví dụ ASOF JOIN cũng chỉ ra 2 hàng bị rò giá trị tương lai nếu lấy giá trị mới nhất. Prototype này giúp kiểm tra hai quyết định về decontamination và point-in-time, nhưng chưa xử lý câu tiếng Việt diễn đạt lại hoặc yêu cầu xoá thật. Nếu xây tiếp, mình sẽ thêm fixture cho hai trường hợp đó và test rằng dữ liệu bị xoá không còn trong bất kỳ manifest đang hoạt động nào.
