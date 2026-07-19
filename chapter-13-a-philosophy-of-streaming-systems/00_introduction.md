# Chương 13: Triết Lý Về Các Hệ Thống Streaming

### (A Philosophy of Streaming Systems)

**Sách:** _Designing Data-Intensive Applications_, ấn bản thứ 2 — Martin Kleppmann & Chris Riccomini (O'Reilly, 2026)

> "Nếu một vật được sinh ra để phục vụ một mục đích khác, thì mục đích tối hậu của nó
> không thể chỉ là bảo toàn sự tồn tại của chính nó. Một thuyền trưởng không xem việc
> giữ gìn con tàu là mục đích cuối cùng, vì con tàu vốn được tạo ra để phục vụ việc điều hướng."
> — Thomas Aquinas, _Summa Theologica_

---

## Bối cảnh và mục tiêu chương

Chương 2 đặt ra mục tiêu xây dựng hệ thống **đáng tin cậy (reliable), có thể mở rộng
(scalable) và dễ bảo trì (maintainable)**. Xuyên suốt cuốn sách, các chủ đề này lặp
đi lặp lại: thuật toán chịu lỗi (Chương 6–9), sharding để mở rộng quy mô (Chương 7),
cơ chế tiến hóa schema và trừu tượng hóa (Chương 4, 11, 12).

Chương 13 **gộp tất cả các ý tưởng đó lại**, xây dựng tiếp trên kiến trúc
streaming/event-driven của Chương 12, để hình thành một **triết lý phát triển ứng
dụng** đáp ứng những mục tiêu trên. Đây là chương mang tính quan điểm cá nhân nhiều
hơn — nó đào sâu vào **một** triết lý cụ thể thay vì so sánh nhiều cách tiếp cận
như các chương trước.

## Ba trụ cột chính của chương

Nội dung chương được chia thành 3 phần lớn (mỗi phần được tóm tắt trong file riêng):

| File                           | Chủ đề                                                                                                                                                                                                |
| ------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `01-Data-Intergration.md`      | **Data Integration** — vì sao không có công cụ nào giải quyết mọi bài toán, và cách dùng batch/stream để kết hợp nhiều hệ thống dữ liệu chuyên biệt                                                   |
| `02-Unbundling-Databases.md`   | **Unbundling Databases** — nhìn cơ sở dữ liệu như tập hợp các "tính năng" (index, materialized view, replication...) có thể tháo rời và ghép lại bằng dataflow; thiết kế ứng dụng xoay quanh dataflow |
| `03-Aiming-For-Correctness.md` | **Aiming for Correctness** — end-to-end argument, exactly-once, ràng buộc uniqueness, timeliness vs. integrity, coordination-avoiding systems, và kiểm toán (auditing)                                |

## Ý tưởng cốt lõi xuyên suốt chương

1. **Không có công cụ vạn năng.** Mỗi hệ thống dữ liệu được thiết kế tối ưu cho một
   số kiểu truy cập nhất định. Ứng dụng thực tế luôn phải phối hợp nhiều công cụ
   chuyên biệt (database, search index, cache, data warehouse, ML system...).
2. **Dữ liệu dẫn xuất (derived data) qua log sự kiện** là cách kết nối các hệ thống
   này một cách đáng tin cậy hơn so với distributed transaction (2PC/XA) — vì nó
   tránh khuếch đại lỗi và cho phép các thành phần phát triển độc lập.
3. **"Thao rời" (unbundling) cơ sở dữ liệu**: những gì database làm nội bộ (duy trì
   index, materialized view...) về bản chất giống hệt việc build hệ thống dữ liệu
   dẫn xuất bằng stream processor — chỉ khác là chạy trên nhiều máy, nhiều đội ngũ.
4. **Đúng đắn (correctness) không nhất thiết cần distributed transaction.** Bằng
   cách dùng request ID xuyên suốt (end-to-end argument), hàm dẫn xuất tất định, và
   xử lý ràng buộc bất đồng bộ, có thể đạt được **integrity** mạnh mà không cần
   coordination đồng bộ toàn hệ thống — đánh đổi lấy hiệu năng và khả năng chịu lỗi
   tốt hơn nhiều.
5. **Tách biệt timeliness và integrity**: vi phạm timeliness (dữ liệu cũ tạm thời)
   có thể tự sửa theo thời gian; vi phạm integrity (mất/hỏng dữ liệu) là vĩnh viễn
   trừ khi sửa chữa chủ động. Phần lớn ứng dụng cần integrity hơn timeliness.

---

_Xem chi tiết từng phần trong các file `01`, `02`, `03`._
