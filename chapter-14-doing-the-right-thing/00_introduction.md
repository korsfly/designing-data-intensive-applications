# Chương 14: Làm Điều Đúng Đắn (Doing the Right Thing)

### Chương cuối cùng của sách

**Sách:** _Designing Data-Intensive Applications_, ấn bản thứ 2 — Martin Kleppmann & Chris Riccomini (O'Reilly, 2026)

> "Nuôi các hệ thống AI bằng cái đẹp, cái xấu, và sự tàn nhẫn của thế giới, nhưng lại
> kỳ vọng nó chỉ phản ánh cái đẹp — đó là một ảo tưởng."
> — Vinay Uday Prabhu & Abeba Birhane, _"Large Datasets: A Pyrrhic Win for Computer
> Vision?"_ (2020)

---

## Bối cảnh

Đây là chương cuối cùng của cuốn sách. Sau khi đã khảo sát rất nhiều kiến trúc hệ
thống dữ liệu, so sánh ưu/nhược điểm, và bàn kỹ thuật xây dựng ứng dụng đáng tin
cậy — sách "lùi lại một bước" để bàn về **phần bị bỏ sót**: **trách nhiệm đạo đức**
của người xây dựng hệ thống dữ liệu.

Luận điểm mở đầu: **mọi hệ thống được xây cho một mục đích; mọi hành động đều có hệ
quả — cả chủ ý lẫn ngoài ý muốn.** Mục đích có thể đơn giản là kiếm tiền, nhưng hệ
quả có thể rất sâu rộng. Kỹ sư xây dựng hệ thống có **trách nhiệm** cân nhắc kỹ các
hệ quả đó và đảm bảo quyết định của mình không gây hại.

Dữ liệu thường được nói tới như một thứ trừu tượng, nhưng nhiều bộ dữ liệu thực ra
là **về con người**: hành vi, sở thích, danh tính của họ — cần được đối xử với **sự
nhân văn và tôn trọng**. Có các bộ quy tắc như _ACM Code of Ethics and Professional
Conduct_, nhưng hiếm khi được thảo luận, áp dụng, và thực thi nghiêm túc trong thực
tế — dẫn đến thái độ xuề xòa với privacy và hệ quả tiêu cực tiềm ẩn của sản phẩm.

**Công nghệ tự nó không tốt hay xấu** — điều quan trọng là nó được dùng như thế nào
và ảnh hưởng đến con người ra sao (giống một công cụ tìm kiếm hay một khẩu súng).
Trách nhiệm đạo đức thuộc về chúng ta; kỹ sư phần mềm không thể chỉ tập trung vào
công nghệ mà bỏ qua hệ quả của nó.

Khác với phần lớn nội dung máy tính, các khái niệm cốt lõi của đạo đức **không cố
định hay xác định rõ ràng** — chúng đòi hỏi diễn giải, có thể mang tính chủ quan.
Đạo đức không phải là việc đi qua một checklist để "tick" đủ ô; đó là một **quá
trình phản tư liên tục, mang tính đối thoại** với những người liên quan, có trách
nhiệm giải trình với kết quả.

## Cấu trúc chương

| File                             | Chủ đề                                                                                                                                                                                                                   |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `01-Predictive-Analytics.md`     | **Predictive Analytics** — thiên kiến & phân biệt đối xử, trách nhiệm giải trình, vòng lặp phản hồi (feedback loop)                                                                                                      |
| `02-Privacy-And-Tracking.md`     | **Privacy and Tracking** — giám sát (surveillance), sự đồng thuận & tự do lựa chọn, quyền riêng tư và việc sử dụng dữ liệu, dữ liệu như tài sản & quyền lực, bài học từ Cách mạng Công nghiệp, luật pháp & tự điều chỉnh |
| `03-Summary-And-Key-Concepts.md` | Tóm tắt chương 14, tóm lược toàn bộ hành trình cuốn sách (Chương 1–14), và bảng khái niệm chính                                                                                                                          |

## Ý tưởng cốt lõi xuyên suốt chương

1. **Predictive analytics** (dự đoán ai đáng tin, ai rủi ro...) có thể tạo ra
   "nhà tù thuật toán" — loại người ta khỏi việc làm, bảo hiểm, tín dụng... một cách
   hệ thống, khó kháng cáo, dựa trên khuôn mẫu học được từ dữ liệu quá khứ vốn đã
   mang thiên kiến.
2. **Ai chịu trách nhiệm khi thuật toán sai?** — Trách nhiệm không thể đổ hết cho
   "thuật toán"; con người vẫn phải giải trình.
3. **Vòng lặp phản hồi tự củng cố (feedback loop)** có thể khiến bất lợi ban đầu nhỏ
   trở thành vòng xoáy đi xuống (ví dụ: điểm tín dụng ↔ thất nghiệp).
4. **Theo dõi hành vi (tracking)** phục vụ lợi ích người dùng ở mức độ nào đó, nhưng
   khi mô hình kinh doanh dựa vào quảng cáo, việc theo dõi vượt xa lợi ích người
   dùng, trở thành **giám sát (surveillance)**.
5. **Sự đồng thuận** của người dùng với việc bị theo dõi thường không thực chất — vì
   thiếu hiểu biết, thiếu lựa chọn thay thế thực tế, và điều khoản dịch vụ do bên
   cung cấp áp đặt một chiều.
6. **Quyền riêng tư** không phải là "giữ bí mật mọi thứ" mà là **quyền quyết định**
   tiết lộ gì, cho ai, trong hoàn cảnh nào — quyền này đang bị chuyển giao ồ ạt từ cá
   nhân sang doanh nghiệp.
7. Dữ liệu cá nhân là **tài sản có giá trị** nhưng cũng là **"tài sản độc hại"** —
   càng thu thập nhiều, rủi ro rò rỉ/lạm dụng càng lớn, kể cả trong tương lai chính
   trị bất định.
8. So sánh với **Cách mạng Công nghiệp**: công nghệ mang lại tăng trưởng nhưng cũng
   gây hại nghiêm trọng nếu không được kiểm soát — cần "quy định môi trường" cho dữ
   liệu giống như từng cần cho ô nhiễm công nghiệp.
9. Luật (như **GDPR**) có thể giúp nhưng có giới hạn — cần **thay đổi văn hóa** trong
   ngành công nghệ: tôn trọng người dùng như con người, tự điều chỉnh, tối giản hóa
   thu thập dữ liệu, và xóa dữ liệu khi không còn cần thiết.

---

_Xem chi tiết từng phần trong các file `01`, `02`, `03`._
