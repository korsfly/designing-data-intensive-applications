# Phần 1 — Tích Hợp Dữ Liệu (Data Integration)

## 1.1 Vấn đề: không có "một giải pháp đúng"

Với bất kỳ bài toán nào ("tôi muốn lưu và tra cứu lại dữ liệu"), luôn có **nhiều
cách giải quyết**, mỗi cách có ưu/nhược điểm riêng (ví dụ: storage engine — log-
structured vs B-tree vs columnar; replication — single-leader vs multi-leader vs
leaderless). Một phần mềm cụ thể thường chỉ chọn _một_ hướng tiếp cận — cố "ôm đồm"
quá nhiều use case thường dẫn đến chất lượng kém hơn so với công cụ chuyên biệt.

→ Vì vậy, ứng dụng phức tạp trong thực tế **luôn phải ghép nhiều công cụ** lại với
nhau: OLTP database + full-text search index + cache + data warehouse + hệ thống ML

- hệ thống gửi thông báo...

## 1.2 Kết hợp công cụ chuyên biệt bằng cách dẫn xuất dữ liệu (deriving data)

### Suy luận về dòng chảy dữ liệu (dataflow)

Khi nhiều bản sao của cùng dữ liệu tồn tại ở nhiều hệ thống lưu trữ, cần biết rõ:
đâu là nơi ghi đầu tiên (system of record), và các bản sao khác được **dẫn xuất**
từ đó như thế nào.

- Nếu **Change Data Capture (CDC)** là con đường _duy nhất_ để cập nhật index tìm
  kiếm, thì index chắc chắn nhất quán với hệ thống gốc (trừ bug phần mềm).
- Nếu ứng dụng ghi trực tiếp vào cả database lẫn search index, hai client ghi xung
  đột có thể khiến hai hệ thống xử lý theo thứ tự khác nhau → **không đồng bộ vĩnh
  viễn**, vì không ai "chịu trách nhiệm" quyết định thứ tự.
- Giải pháp: dồn toàn bộ input qua **một hệ thống quyết định thứ tự tổng thể** (total
  order), sau đó các dữ liệu dẫn xuất khác được xử lý theo đúng thứ tự đó — đây là
  ứng dụng của **state machine replication**. Dùng CDC hay event-sourcing log không
  quan trọng bằng nguyên tắc "quyết định một thứ tự chung".
- Cập nhật hệ thống dữ liệu dẫn xuất dựa trên event log thường có thể làm **tất
  định (deterministic) và idempotent**, giúp phục hồi lỗi dễ dàng.

### Dữ liệu dẫn xuất so với distributed transaction

- **Distributed transaction (2PC)** đạt mục tiêu tương tự bằng atomic commit
  protocol; **hệ thống log-based** đạt correctness bằng retry tất định + idempotence.
- Khác biệt lớn nhất: transaction hỗ trợ "đọc ngay giá trị vừa ghi" (read-your-writes);
  hệ thống dẫn xuất thường cập nhật **bất đồng bộ** nên không đảm bảo dữ liệu mới nhất.
- XA có khả năng chịu lỗi và hiệu năng kém (xem lại "Distributed Transactions Across
  Different Systems"), khó được áp dụng rộng rãi. → **Log-based derived data hiện là
  hướng tiếp cận khả thi nhất** để tích hợp nhiều hệ thống dữ liệu khác nhau.

### Giới hạn của total ordering

Xây dựng một event log có thứ tự tổng thể là khả thi với hệ thống nhỏ, nhưng gặp
giới hạn khi mở rộng:

- Log phải qua một leader duy nhất quyết định thứ tự → khi throughput vượt quá một
  máy, phải shard log, và thứ tự giữa 2 shard là **không xác định**.
- Hệ thống đa vùng địa lý (multi-datacenter) thường có leader riêng ở mỗi vùng vì
  network delay khiến đồng bộ liên vùng không hiệu quả → thứ tự event từ 2 vùng khác
  nhau **không xác định**.
- Kiến trúc microservices: mỗi service quản lý state riêng, không chia sẻ trạng thái
  bền vững → sự kiện từ hai service khác nhau không có thứ tự xác định.
- Ứng dụng có client-side state (cập nhật ngay khi người dùng nhập, kể cả offline)
  → client và server rất dễ thấy sự kiện theo thứ tự khác nhau.

Về mặt hình thức, việc quyết định thứ tự tổng thể của sự kiện gọi là **total order
broadcast**, tương đương với **consensus** — hầu hết thuật toán consensus chỉ thiết
kế cho thông lượng một node, không chia việc sắp thứ tự cho nhiều node.

### Sắp thứ tự sự kiện để nắm bắt quan hệ nhân quả (causality)

- Nếu 2 sự kiện không có liên hệ nhân quả, thiếu total order không phải vấn đề lớn.
- Ví dụ kinh điển: mạng xã hội — người dùng A hủy kết bạn với B, sau đó gửi tin nhắn
  phàn nàn về B tới bạn bè còn lại. Nếu hệ thống lưu "trạng thái bạn bè" và "tin nhắn"
  ở 2 nơi khác nhau, sự phụ thuộc nhân quả giữa 2 sự kiện có thể bị mất → dịch vụ
  thông báo có thể gửi nhầm thông báo tin nhắn đó cho B.
- Đây thực chất là vấn đề **join phụ thuộc thời gian** (time dependence of joins).
  Chưa có lời giải đơn giản, nhưng có vài hướng gợi ý:
  1. **Logical timestamp** cho phép sắp thứ tự tổng thể mà không cần coordination,
     nhưng recipient vẫn phải xử lý sự kiện đến sai thứ tự, và cần thêm metadata.
  2. Ghi lại sự kiện thể hiện trạng thái hệ thống mà người dùng thấy trước khi quyết
     định, gán ID duy nhất cho sự kiện đó — các sự kiện sau có thể tham chiếu ID này
     để ghi nhận quan hệ nhân quả.
  3. **Thuật toán giải quyết xung đột** (conflict resolution) giúp xử lý sự kiện đến
     sai thứ tự khi duy trì state, nhưng không giúp được nếu hành động có side effect
     bên ngoài (như gửi thông báo).

## 1.3 Batch và Stream Processing

Mục tiêu của tích hợp dữ liệu là đưa dữ liệu về đúng dạng, đúng nơi. Batch và stream
processor là công cụ thực hiện việc đó — output của chúng là **dữ liệu dẫn xuất**
(search index, materialized view, gợi ý, metric tổng hợp...). Khác biệt căn bản: batch
xử lý dataset kích thước hữu hạn, stream xử lý dataset không giới hạn (unbounded).

### Duy trì trạng thái dẫn xuất

- Batch processing mang tính chất **hàm thuần túy (pure function)**: đầu ra chỉ phụ
  thuộc đầu vào, không side effect, input bất biến, output append-only. Stream
  processing tương tự nhưng có thêm state được quản lý, chịu lỗi.
- Nguyên tắc hàm tất định + input/output rõ ràng không chỉ tốt cho fault tolerance
  mà còn giúp **dễ suy luận về dataflow** trong toàn tổ chức.
- Về lý thuyết có thể duy trì dữ liệu dẫn xuất **đồng bộ** (như database cập nhật
  secondary index cùng transaction), nhưng **bất đồng bộ mới là điều làm hệ thống
  event-log-based trở nên robust**: lỗi ở một phần được cô lập cục bộ, trong khi
  distributed transaction sẽ abort toàn bộ nếu một participant lỗi → khuếch đại lỗi.

### Reprocessing dữ liệu để tiến hóa ứng dụng

- Stream processing: phản ánh thay đổi input vào view dẫn xuất với độ trễ thấp.
- Batch processing: **reprocess** lượng lớn dữ liệu lịch sử tích lũy để tạo view mới.
- Reprocessing là cơ chế tốt để tiến hóa hệ thống hỗ trợ tính năng mới/yêu cầu thay
  đổi — không chỉ giới hạn ở thêm field tùy chọn, mà có thể **tái cấu trúc hoàn toàn**
  mô hình dữ liệu.
- **Ví dụ tương tự ngoài đời**: chuyển đổi khổ đường ray tàu hỏa ở Anh thế kỷ 19 —
  chuyển dần bằng cách thêm ray thứ 3 (dual gauge), cho cả 2 loại tàu chạy song song,
  rồi mới bỏ ray cũ. → **View dẫn xuất cho phép tiến hóa dần dần**: giữ song song
  schema cũ và mới, dịch chuyển dần user sang view mới, có thể **rollback dễ dàng**
  ở bất kỳ giai đoạn nào nếu có sự cố — giảm rủi ro giúp đi nhanh hơn, tự tin hơn.

### Hợp nhất batch và stream processing

- **Lambda architecture** (đề xuất ban đầu) có nhiều vấn đề, hiện không còn phổ biến.
- Xu hướng hiện tại: **Kappa architecture** — cùng một hệ thống xử lý cả batch (dữ
  liệu lịch sử) lẫn stream (dữ liệu mới).
- Yêu cầu để hợp nhất:
  1. Khả năng **replay sự kiện lịch sử** qua cùng engine xử lý stream mới (log-based
     broker có thể replay message; một số stream processor đọc từ distributed
     filesystem/object storage).
  2. **Exactly-once semantics** cho stream processor — output giống như không có lỗi
     xảy ra, dù thực tế có lỗi (phải loại bỏ output một phần của task lỗi).
  3. Công cụ **windowing theo event time** (không phải processing time) — vì
     processing time vô nghĩa khi reprocess dữ liệu cũ. Ví dụ: Apache Beam cung cấp
     API cho việc này, chạy trên Apache Flink hoặc Google Cloud Dataflow.
