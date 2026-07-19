# Summary & Key Concepts — Chapter 13

## Tóm tắt chương (Summary)

Chương 13 khám phá cách thiết kế hệ thống dữ liệu dựa trên ý tưởng của stream
processing:

1. **Không công cụ nào phục vụ mọi use case hiệu quả** → ứng dụng phải ghép nhiều
   phần mềm lại. Giải quyết bài toán tích hợp dữ liệu bằng batch processing và
   event stream để dữ liệu "chảy" giữa các hệ thống.
2. Một số hệ thống được chỉ định là **system of record**, dữ liệu khác được **dẫn
   xuất** qua biến đổi — nhờ đó duy trì index, materialized view, mô hình ML, tổng
   hợp thống kê... Làm cho các phép dẫn xuất/biến đổi này **bất đồng bộ, loose-
   coupled** giúp ngăn lỗi cục bộ lan rộng, tăng độ bền vững/chịu lỗi của toàn hệ.
3. Biểu diễn dataflow như phép biến đổi từ dataset này sang dataset khác giúp
   **tiến hóa ứng dụng** dễ hơn: muốn đổi bước xử lý (đổi cấu trúc index/cache), chỉ
   cần chạy lại code biến đổi mới trên toàn bộ input để dẫn xuất lại output; có lỗi
   thì sửa code rồi reprocess để khôi phục.
4. Quá trình này giống hệt những gì database **đã làm nội bộ** → tái hiện ý tưởng
   dataflow app như **thao rời (unbundling)** các thành phần database, xây ứng dụng
   bằng cách ghép các thành phần loose-coupled đó.
5. State dẫn xuất được cập nhật bằng cách quan sát thay đổi dữ liệu gốc, và state đó
   lại được downstream consumer quan sát tiếp — có thể mở rộng dataflow **đến tận
   thiết bị người dùng cuối**, xây UI cập nhật động, hoạt động offline được.
6. **Correctness trong điều kiện có lỗi**: integrity mạnh có thể cài đặt **mở rộng
   quy mô** bằng xử lý event bất đồng bộ, dùng **request ID end-to-end** để idempotent,
   kiểm tra ràng buộc **bất đồng bộ**. Client có thể chờ kiểm tra hoàn tất, hoặc đi
   tiếp không chờ nhưng chấp nhận rủi ro phải "xin lỗi" nếu vi phạm ràng buộc — cách
   tiếp cận này scale và bền vững hơn nhiều so với distributed transaction truyền
   thống, và khớp với cách nhiều quy trình kinh doanh vận hành trong thực tế.
7. Cấu trúc ứng dụng quanh dataflow + kiểm tra ràng buộc bất đồng bộ → tránh phần
   lớn coordination, tạo hệ thống giữ integrity nhưng vẫn hiệu năng tốt, kể cả khi
   phân tán địa lý và có lỗi.
8. Cuối cùng: dùng **audit** để xác minh integrity dữ liệu, phát hiện hỏng hóc — kỹ
   thuật của blockchain cũng có nét tương đồng với hệ thống event-based.

---

## Bảng khái niệm chính (Key Concepts)

| Khái niệm                                     | Giải thích ngắn gọn                                                                                                                                                               |
| --------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **System of record**                          | Hệ thống được chỉ định là nguồn dữ liệu gốc, đáng tin cậy nhất; mọi dữ liệu khác được dẫn xuất từ đây.                                                                            |
| **Derived data (dữ liệu dẫn xuất)**           | Dữ liệu được tính toán/biến đổi từ system of record (index, cache, materialized view, mô hình ML...).                                                                             |
| **Total order broadcast**                     | Đảm bảo mọi node thấy các sự kiện theo đúng cùng một thứ tự; về mặt lý thuyết tương đương với consensus.                                                                          |
| **Causality (quan hệ nhân quả)**              | Quan hệ "sự kiện A xảy ra trước và ảnh hưởng đến sự kiện B"; khó bảo toàn khi hệ thống không có total order.                                                                      |
| **Lambda architecture**                       | Kiến trúc cũ chạy song song batch và stream riêng biệt để hợp nhất kết quả; nay đã lỗi thời.                                                                                      |
| **Kappa architecture**                        | Kiến trúc hiện đại hơn, dùng một engine xử lý cả batch (reprocess lịch sử) lẫn stream (dữ liệu mới).                                                                              |
| **Unbundling databases**                      | Nhìn các tính năng của database (index, view, replication) như các thành phần rời, có thể ghép lại bằng dataflow/stream processor từ nhiều công cụ khác nhau.                     |
| **Federated database / polystore**            | Cung cấp giao diện truy vấn thống nhất trên nhiều storage engine (hợp nhất đọc).                                                                                                  |
| **Write path / Read path**                    | Write path: công việc tính toán trước (eager) khi dữ liệu đến. Read path: công việc chỉ làm khi có truy vấn (lazy). Cache/index/materialized view là ranh giới giữa hai path này. |
| **Dataflow architecture**                     | Xây ứng dụng xoay quanh việc theo dõi & phản ứng với thay đổi dữ liệu (giống spreadsheet), thay vì mô hình database thụ động.                                                     |
| **Stream processor như dịch vụ**              | So sánh với microservices: giao tiếp một chiều, bất đồng bộ qua stream thay vì request/response đồng bộ.                                                                          |
| **End-to-end argument**                       | Một chức năng (dedup, checksum, mã hóa...) chỉ có thể cài đặt đúng đắn hoàn toàn ở hai đầu mút ứng dụng, không thể khoán hết cho hạ tầng trung gian.                              |
| **Exactly-once / effectively-once semantics** | Đảm bảo hiệu ứng cuối cùng giống như không có lỗi xảy ra, dù thao tác có bị retry.                                                                                                |
| **Idempotence**                               | Thao tác cho cùng kết quả dù thực hiện một hay nhiều lần.                                                                                                                         |
| **Request ID / duplicate suppression**        | Gắn ID duy nhất cho mỗi request từ client, truyền xuyên suốt hệ thống để loại bỏ xử lý trùng lặp.                                                                                 |
| **Compensating transaction**                  | Thao tác "sửa lỗi sau" khi một ràng buộc bị vi phạm tạm thời (ví dụ: hoàn tiền, xin lỗi, tặng ưu đãi).                                                                            |
| **Timeliness**                                | Người dùng thấy hệ thống ở trạng thái mới nhất; vi phạm chỉ mang tính **tạm thời**.                                                                                               |
| **Integrity**                                 | Không mất/hỏng/mâu thuẫn dữ liệu; vi phạm mang tính **vĩnh viễn** nếu không sửa chữa chủ động.                                                                                    |
| **Coordination-avoiding data systems**        | Hệ thống đạt integrity mạnh mà không cần đồng bộ coordination toàn cục — hiệu năng và khả năng chịu lỗi tốt hơn.                                                                  |
| **System model**                              | Tập giả định về những gì có thể/không thể xảy ra (crash, mất điện, lỗi mạng...) làm nền cho thiết kế hệ thống.                                                                    |
| **Auditing (kiểm toán)**                      | Kiểm tra định kỳ tính toàn vẹn dữ liệu để phát hiện và sửa hỏng hóc, thay vì tin tưởng tuyệt đối.                                                                                 |
| **Trust, but verify**                         | Triết lý: giả định lỗi hiếm vẫn có thể xảy ra, nên cần liên tục tự kiểm chứng (như HDFS/S3 kiểm tra lại dữ liệu trên đĩa).                                                        |
| **Merkle tree**                               | Cấu trúc cây hash cho phép chứng minh hiệu quả một record có thuộc dataset hay không, dùng trong kiểm toán/blockchain.                                                            |
| **Byzantine fault tolerance**                 | Khả năng hệ thống vẫn hoạt động đúng dù một số node có dữ liệu bị hỏng/độc hại (đặc trưng của blockchain, khác consensus thông thường).                                           |

---

## Ghi chú liên hệ với các chương khác

- Chương 2 (nonfunctional requirements: reliable, scalable, maintainable) — mục tiêu
  xuyên suốt được "chốt hạ" ở chương này.
- Chương 6, 9, 10 (replication, distributed transaction, consensus) — nền tảng cho
  phần bàn về uniqueness constraint và total order broadcast.
- Chương 11, 12 (batch processing, stream processing/event-driven architecture) —
  chương 13 xây trực tiếp trên các khái niệm này.
- Chương 8 (transaction, ACID) — đối chiếu với cách tiếp cận "coordination-avoiding".
- Chương 14 (Doing the Right Thing) — chương tiếp theo, bàn về khía cạnh đạo đức/xã
  hội, nối tiếp mạch tư duy "trách nhiệm khi xây hệ thống dữ liệu".
