# Summary & Key Concepts — Chapter 14

## Tóm tắt chương 14

Đây là kết thúc của cuốn sách. Sau khi khảo sát rất nhiều kiến trúc, chương cuối
cùng "lùi lại một bước" để xem xét **khía cạnh đạo đức** của việc xây dựng ứng dụng
đậm chất dữ liệu:

- Dữ liệu **có thể làm điều tốt**, nhưng cũng có thể gây **hại đáng kể**: ra quyết
  định ảnh hưởng nghiêm trọng đến cuộc sống con người mà khó kháng cáo, dẫn đến
  phân biệt đối xử và bóc lột, bình thường hóa giám sát, và phơi bày thông tin thân
  mật.
- Ta còn đối mặt rủi ro **rò rỉ dữ liệu (data breach)**, và có thể phát hiện rằng
  một cách sử dụng dữ liệu với ý định tốt lại có **hệ quả ngoài ý muốn**.
- Với tác động lớn mà phần mềm và dữ liệu có lên thế giới, là kỹ sư, ta phải nhớ
  rằng ta mang **trách nhiệm** hướng tới loại thế giới ta muốn sống trong đó — một
  thế giới đối xử với con người bằng **sự nhân văn và tôn trọng**. Hãy cùng nhau
  hướng tới mục tiêu đó.

## Tóm lược hành trình toàn cuốn sách (Chương 1–14)

| Chương                                         | Nội dung chính                                                                                                                                                                 |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **1. Trade-Offs in Data Systems Architecture** | So sánh hệ thống operational vs analytical, cloud vs self-hosting, distributed vs single-node, cân bằng nhu cầu doanh nghiệp và người dùng                                     |
| **2. Defining Nonfunctional Requirements**     | Định nghĩa performance, reliability, scalability, maintainability                                                                                                              |
| **3. Data Models and Query Languages**         | Mô hình relational, document, graph, event sourcing, DataFrames; các ngôn ngữ query: SQL, Cypher, SPARQL, Datalog, GraphQL                                                     |
| **4. Storage and Retrieval**                   | Storage engine cho OLTP (LSM-tree, B-tree) và analytics (column-oriented); index cho information retrieval (full-text, vector search)                                          |
| **5. Encoding and Evolution**                  | Mã hóa dữ liệu thành byte, hỗ trợ tiến hóa khi yêu cầu thay đổi; các cách dữ liệu chảy giữa tiến trình: qua database, service call, workflow engine, event-driven architecture |
| **6. Replication**                             | Đánh đổi giữa single-leader, multi-leader, leaderless replication; mô hình consistency (read-after-write); sync engine cho làm việc offline                                    |
| **7. Sharding**                                | Chiến lược rebalancing, request routing, secondary indexing                                                                                                                    |
| **8. Transactions**                            | Durability, các mức isolation (read committed, snapshot isolation, serializable), atomicity trong distributed transaction                                                      |
| **9. The Trouble with Distributed Systems**    | Các vấn đề nền tảng: lỗi/độ trễ mạng, sai lệch đồng hồ, process pause, crash — và vì sao cả một cái lock cũng khó cài đúng                                                     |
| **10. Consistency and Consensus**              | Các dạng consensus, mô hình consistency (linearizability)                                                                                                                      |
| **11. Batch Processing**                       | Từ chuỗi công cụ Unix đơn giản đến batch processor phân tán quy mô lớn dùng distributed filesystem/object store                                                                |
| **12. Stream Processing**                      | Tổng quát hóa batch thành stream; message broker, CDC, fault tolerance, streaming join                                                                                         |
| **13. A Philosophy of Streaming Systems**      | Triết lý tích hợp hệ thống dữ liệu khác biệt, tiến hóa hệ thống, mở rộng ứng dụng dễ dàng hơn                                                                                  |
| **14. Doing the Right Thing**                  | Khía cạnh đạo đức: predictive analytics, bias, giám sát, quyền riêng tư, trách nhiệm xã hội của kỹ sư                                                                          |

## Bảng khái niệm chính (Key Concepts) — Chương 14

| Khái niệm                                              | Giải thích ngắn gọn                                                                                                                                               |
| ------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Predictive analytics**                               | Dùng phân tích dữ liệu để dự đoán hành vi/kết quả tương lai (tái phạm, vỡ nợ, khiếu nại bảo hiểm...); ảnh hưởng trực tiếp đến cuộc sống cá nhân.                  |
| **Algorithmic prison**                                 | Tình trạng một người bị loại trừ có hệ thống khỏi việc làm, bảo hiểm, tín dụng... do bị thuật toán gắn nhãn "rủi ro," khó kháng cáo.                              |
| **Bias and discrimination**                            | Thiên kiến trong dữ liệu đầu vào (thường phản ánh bất công lịch sử) bị thuật toán **học và khuếch đại** trong kết quả đầu ra.                                     |
| **Proxy variable**                                     | Đặc điểm dữ liệu (như mã bưu điện, IP) tương quan chặt với đặc điểm được pháp luật bảo vệ (chủng tộc...), dù không trực tiếp là đặc điểm đó.                      |
| **Accountability (trách nhiệm giải trình)**            | Yêu cầu con người/tổ chức phải chịu trách nhiệm và giải thích được quyết định của thuật toán, không thể "đổ lỗi cho thuật toán."                                  |
| **Feedback loop (vòng lặp phản hồi)**                  | Cơ chế tự củng cố khiến bất lợi ban đầu (nhỏ) trở thành vòng xoáy tiêu cực kéo dài (ví dụ: điểm tín dụng ↔ thất nghiệp).                                          |
| **Systems thinking**                                   | Cách tiếp cận suy nghĩ về toàn bộ hệ thống (bao gồm cả con người tương tác với nó), không chỉ phần mềm, để dự đoán hệ quả.                                        |
| **Data exhaust (khí thải dữ liệu)**                    | Cách gọi (thường sai lệch) dữ liệu hành vi là "chất thải" vô giá trị được thu thập như sản phẩm phụ.                                                              |
| **Surveillance (giám sát)**                            | Việc theo dõi hành vi vượt xa mục đích phục vụ người dùng, chủ yếu phục vụ lợi ích của bên thứ ba (như nhà quảng cáo).                                            |
| **Meaningful consent (đồng thuận thực chất)**          | Sự đồng ý chỉ có giá trị khi người dùng hiểu rõ hệ quả và có lựa chọn thực sự — điều hiếm khi đúng với chính sách quyền riêng tư hiện tại.                        |
| **GDPR**                                               | Quy định bảo vệ dữ liệu của EU: yêu cầu đồng thuận tự nguyện, cụ thể, minh bạch; nguyên tắc tối giản hóa dữ liệu (data minimization).                             |
| **Privacy as a decision right**                        | Quyền riêng tư = quyền **quyết định** tiết lộ gì cho ai, không phải "giữ bí mật tuyệt đối."                                                                       |
| **Data as a toxic asset (dữ liệu là tài sản độc hại)** | Quan điểm: dữ liệu cá nhân có giá trị nhưng cũng là rủi ro lớn (rò rỉ, lạm dụng, bị chính phủ cưỡng ép giao nộp).                                                 |
| **Data broker (nhà môi giới dữ liệu)**                 | Tổ chức mua, tổng hợp, phân tích, bán lại dữ liệu cá nhân, thường phục vụ mục đích marketing.                                                                     |
| **"Knowledge is power"**                               | Nguyên lý: khả năng giám sát người khác mà tránh bị giám sát là một dạng quyền lực quan trọng — cơ sở cho lo ngại về quyền lực tập trung ở các công ty công nghệ. |
| **So sánh với Cách mạng Công nghiệp**                  | Ẩn dụ: dữ liệu là "ô nhiễm" của kỷ nguyên thông tin, cần "quy định môi trường" tương tự như từng cần cho ô nhiễm công nghiệp.                                     |
| **Data minimization (tối giản hóa dữ liệu)**           | Nguyên tắc chỉ thu thập/giữ dữ liệu cần thiết cho mục đích cụ thể — đối lập với triết lý "thu thập càng nhiều càng tốt" của big data.                             |
| **Tragedy of the commons**                             | Ẩn dụ: nếu không ai chủ động bảo vệ quyền riêng tư như tài sản chung, nó sẽ bị hủy hoại bởi lợi ích cá nhân/tổ chức khai thác nó.                                 |
| **Self-regulation (tự điều chỉnh)**                    | Kêu gọi ngành công nghệ chủ động thay đổi văn hóa, thực hành thu thập/xử lý dữ liệu để xây dựng niềm tin, thay vì chỉ chờ luật pháp.                              |

## Ghi chú liên hệ

- Chương 14 nối tiếp Chương 13 về mặt tư duy: sau khi bàn kỹ thuật để hệ thống dữ
  liệu _hoạt động đúng_ (correctness, integrity), chương cuối bàn về việc hệ thống
  đó _nên được dùng đúng đắn_ về mặt đạo đức — hai khái niệm "đúng" bổ sung cho nhau.
- Ý tưởng "derived data" và các bộ dữ liệu tổng hợp toàn user base (Chương 12–13)
  chính là loại dữ liệu khiến người dùng **không thể hiểu hay kiểm soát** hệ quả
  đầy đủ của việc chia sẻ dữ liệu — cầu nối trực tiếp giữa kỹ thuật và đạo đức.
- Đây cũng là lời kết cho toàn bộ cuốn sách: từ các đánh đổi kỹ thuật ở Chương 1
  đến trách nhiệm xã hội ở Chương 14, khép lại vòng tròn "reliable, scalable,
  maintainable — và có đạo đức."
