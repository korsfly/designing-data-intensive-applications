# GraphQL và Event Sourcing/CQRS

---

## Phần A: GraphQL

### 1. GraphQL khác gì với các Graph Query Language khác?

> GraphQL **HẠN CHẾ HƠN NHIỀU** so với Cypher/SPARQL/Datalog đã thấy ở phần trước. Mục đích KHÔNG phải để query graph database phức tạp — mà để **client software** (mobile app, JavaScript frontend) request 1 JSON document có cấu trúc CỤ THỂ, chứa đúng field cần để render UI.

```
GraphQL = OLTP query language CHO CLIENT-SERVER COMMUNICATION
   KHÔNG phải dành cho graph database traversal phức tạp
```

### 2. Ưu điểm chính

> GraphQL cho phép developer THAY ĐỔI NHANH query trong client code **MÀ KHÔNG cần thay đổi server-side API**.

```
Client cần field MỚI → chỉ cần SỬA QUERY ở client
   KHÔNG cần backend deploy API version mới
```

### 3. Chi phí và Hạn chế

| Vấn đề                                        | Chi tiết                                                                     |
| --------------------------------------------- | ---------------------------------------------------------------------------- |
| **Cần tooling chuyển đổi**                    | Tổ chức cần convert GraphQL query → request đến internal service (REST/gRPC) |
| **Authorization phức tạp hơn**                | Cần kiểm soát field-level access                                             |
| **Rate limiting khó hơn**                     | Query có thể có độ phức tạp khác nhau rất nhiều                              |
| **KHÔNG cho phép recursive query**            | Khác Cypher/SPARQL/SQL/Datalog — vì query đến từ **untrusted sources**       |
| **KHÔNG cho phép arbitrary search condition** | Trừ khi service owner CHỦ ĐỘNG cung cấp search functionality                 |

> **Lý do hạn chế:** GraphQL query đến từ **client không tin cậy** (untrusted). Nếu cho phép query đắt đỏ tùy ý → user (vô tình hoặc cố ý) có thể gây **denial-of-service** trên server.

### 4. Ví dụ: Group Chat Application (Discord/Slack-like)

```graphql
query ChatApp {
  channels {
    name
    recentMessages(latest: 50) {
      timestamp
      content
      sender {
        fullName
        imageUrl
      }
      replyTo {
        content
        sender {
          fullName
        }
      }
    }
  }
}
```

**Response tương ứng (JSON mirror cấu trúc query):**

```json
{
  "data": {
    "channels": [
      {
        "name": "#general",
        "recentMessages": [
          {
            "timestamp": 1693143014,
            "content": "Hey! How are y'all doing?",
            "sender": { "fullName": "Aaliyah", "imageUrl": "https://..." },
            "replyTo": null
          },
          {
            "timestamp": 1693143024,
            "content": "Great! And you?",
            "sender": { "fullName": "Caleb", "imageUrl": "https://..." },
            "replyTo": {
              "content": "Hey! How are y'all doing?",
              "sender": { "fullName": "Aaliyah" }
            }
          }
        ]
      }
    ]
  }
}
```

### 5. Đặc điểm quan trọng của Response

```
Response = JSON document MIRROR cấu trúc của query
   → Chứa ĐÚNG attributes đã request, KHÔNG HƠN KHÔNG KÉM

Lợi ích:
   Server KHÔNG cần biết client cần attribute gì để render UI
   Client TỰ request đúng cái cần
   → Nếu UI thêm hiển thị profile picture cho replyTo.sender,
     client chỉ cần THÊM imageUrl vào query, KHÔNG cần sửa server
```

### 6. Duplication trong Response — Trade-off chủ ý

```
sender.fullName + imageUrl LẶP LẠI trong mỗi message
   (dù cùng 1 user gửi nhiều message)

replyTo.content + sender LẶP LẠI
   (thay vì chỉ trả message ID gốc)

→ GraphQL CHỦ Ý chấp nhận response LỚN HƠN
  để ĐƠN GIẢN HÓA việc render UI ở client
  (client không cần thêm round-trip để resolve ID)
```

### 7. Implementation phía Server

```
Server's database có thể lưu data NORMALIZED HƠN
   (vd: message lưu user ID của sender, ID của message được reply)

Khi nhận GraphQL query → server RESOLVE các ID đó (join nội bộ)

QUAN TRỌNG: CHỈ những join đã ĐƯỢC KHAI BÁO RÕ trong GraphQL schema
            mới có thể được client request
```

> **Lưu ý thú vị:** Mặc dù response GraphQL TRÔNG giống response từ document database, và tên có "graph" — GraphQL có thể implement trên **BẤT KỲ loại database nào**: relational, document, hoặc graph thực sự.

---

## Phần B: Event Sourcing và CQRS

### 1. Động lực: Một Representation Không Đủ

```
Tất cả model đã thấy (JSON, relational rows, graph vertices/edges):
   DATA ĐƯỢC QUERY THEO CÙNG FORM như khi VIẾT

NHƯNG với application PHỨC TẠP:
   1 representation DUY NHẤT có thể KHÔNG ĐỦ
   để thỏa mãn TẤT CẢ cách query/present data cần thiết

→ Ý tưởng: VIẾT data theo 1 form, sau đó DERIVE
   các representation TỐI ƯU cho các loại READ khác nhau
```

> Liên kết với **Systems of Record and Derived Data** (Chapter 1) và **ETL** (Data Warehousing).

### 2. Event Log — Cách viết Data tối ưu nhất

> Có thể cách đơn giản, NHANH, và **EXPRESSIVE NHẤT** để viết data là: **Event Log**.

```
Mỗi lần muốn viết data:
   → Encode thành self-contained string (vd: JSON)
   → Bao gồm TIMESTAMP
   → APPEND vào sequence of events

Events trong log: IMMUTABLE
   KHÔNG BAO GIỜ thay đổi/xóa
   CHỈ append THÊM events (có thể "supersede" event cũ)

Event có thể chứa BẤT KỲ properties nào (arbitrary)
```

### 3. Ví dụ: Conference Management System

```
Domain phức tạp:
   - Cá nhân đăng ký + trả qua thẻ
   - Công ty đặt ghế HÀNG LOẠT, trả qua invoice,
     gán ghế cho người sau
   - Ghế dành riêng cho speaker, sponsor, volunteer
   - Reservation có thể bị CANCEL
   - Organizer có thể đổi CAPACITY (chuyển phòng khác)

→ Tính số ghế còn lại trở thành QUERY PHỨC TẠP
```

### 4. Kiến trúc Event Sourcing

```
              ┌──► Materialized View 1: Booking status info
              │
Event Log ────┼──► Materialized View 2: Charts cho organizer dashboard
(source of    │
 truth)       └──► Materialized View 3: File cho máy in (badge attendees)
```

```
MỖI thay đổi state (organizer mở registration, attendee đặt/hủy)
   → TRƯỚC HẾT được lưu thành EVENT

MỖI khi event được append vào log
   → CÁC materialized views được UPDATE để phản ánh effect của event đó
```

### 5. Định nghĩa thuật ngữ

| Thuật ngữ                                           | Định nghĩa                                                                                    |
| --------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| **Event Sourcing**                                  | Dùng events làm SOURCE OF TRUTH, expressing mọi state change thành EVENT                      |
| **CQRS** (Command Query Responsibility Segregation) | Nguyên tắc giữ READ-OPTIMIZED representations RIÊNG, derive từ WRITE-OPTIMIZED representation |
| **Command**                                         | Request từ user, cần được VALIDATE trước khi trở thành Event                                  |
| **Materialized View / Projection / Read Model**     | Representation được derive từ event log, tối ưu cho 1 loại read cụ thể                        |

> Thuật ngữ này xuất phát từ cộng đồng **DDD** (Domain-Driven Design), tuy ý tưởng tương tự đã tồn tại lâu (vd: **state machine replication**).

### 6. Quy trình Command → Event

```
User Request → COMMAND
                  │
                  ▼
            VALIDATE (vd: có đủ ghế trống?)
                  │
                  ▼ (nếu hợp lệ)
              EVENT (fact đã xảy ra)
                  │
                  ▼
            Append vào Event Log
                  │
                  ▼
       Materialized View consumers KHÔNG được REJECT event
       (event log CHỈ chứa VALID events)
```

> **Quy ước đặt tên Event:** Nên dùng **THỜI QUÁ KHỨ** (vd: "the seats were booked") vì event = record của 1 fact ĐÃ XẢY RA. Dù user sau đó hủy reservation, fact "họ TỪNG có booking" vẫn ĐÚNG; cancellation là 1 EVENT KHÁC được thêm SAU.

### 7. So sánh với Star Schema Fact Table

|            | **Event Sourcing**                                           | **Star Schema Fact Table**              |
| ---------- | ------------------------------------------------------------ | --------------------------------------- |
| Loại event | NHIỀU loại, mỗi loại có PROPERTIES khác nhau                 | TẤT CẢ rows có CÙNG set columns         |
| Thứ tự     | RẤT QUAN TRỌNG (booking → cancel phải đúng thứ tự)           | KHÔNG quan trọng (unordered collection) |
| Điểm chung | Cả hai là collection của các SỰ KIỆN ĐÃ XẢY RA trong quá khứ |

### 8. Ưu điểm của Event Sourcing/CQRS

| Ưu điểm                               | Giải thích                                                                                                      |
| ------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| **Communicate intent rõ ràng**        | "booking was canceled" DỄ HIỂU hơn liệt kê chi tiết các thay đổi row/column                                     |
| **Reproducibility**                   | Materialized view = derive REPRODUCIBLE từ event log → có thể XÓA và TÍNH LẠI                                   |
| **Dễ debug**                          | Có thể RERUN view maintenance code nhiều lần, kiểm tra hành vi                                                  |
| **Nhiều materialized view khác nhau** | Mỗi view tối ưu cho 1 loại query cụ thể, có thể dùng data model khác nhau, denormalized cho tốc độ              |
| **View tạm thời (chỉ memory)**        | OK nếu có thể recompute từ event log khi service restart                                                        |
| **Dễ thêm features mới**              | Build NEW materialized view từ event log CŨ rất dễ; thêm event type mới hoặc property mới mà KHÔNG sửa event cũ |
| **Chain behaviors mới**               | Vd: attendee hủy → ghế tự động offer cho người trong waiting list                                               |
| **Sửa lỗi bằng Event mới**            | Event sai → viết EVENT XÓA để đảo ngược; downstream views tự động incorporate                                   |
| **Audit log**                         | Hữu ích cho ngành CÓ QUY ĐỊNH cần auditability                                                                  |
| **Throughput cao hơn**                | Event log có sequential access pattern → absorb burst tốt, downstream catch up theo pace riêng                  |

### 9. Nhược điểm của Event Sourcing/CQRS

| Nhược điểm                            | Chi tiết                                                                                                                                                                                        |
| ------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **External information cần cẩn thận** | Vd: tỷ giá ngoại tệ thay đổi theo thời gian → CẦN lưu exchange rate TRONG event (hoặc query historical rate theo timestamp xác định) để event processing DETERMINISTIC                          |
| **Immutability + GDPR conflict**      | Right to be forgotten (xóa data cá nhân) khó với event immutable. Giải pháp: per-user log (dễ xóa), hoặc **crypto-shredding** (mã hóa rồi xóa key) — nhưng làm khó việc recompute derived state |
| **Side effects khi reprocess**        | Vd: KHÔNG muốn gửi lại confirmation email mỗi lần rebuild materialized view                                                                                                                     |

### 10. Hệ thống hỗ trợ Event Sourcing

| Loại                                    | Ví dụ                                                                |
| --------------------------------------- | -------------------------------------------------------------------- |
| Database thiết kế riêng cho pattern này | EventStoreDB, MartenDB (dựa trên PostgreSQL), Axon Framework         |
| Message brokers (lưu event log)         | Apache Kafka + stream processors (giữ materialized views up-to-date) |

> **Yêu cầu quan trọng nhất:** Event storage system PHẢI đảm bảo **TẤT CẢ materialized views xử lý events theo ĐÚNG THỨ TỰ** như chúng xuất hiện trong log. Trong **distributed system**, điều này **KHÔNG dễ** đạt được (xem Chapter 10 — Consistency and Consensus).

---

## Tóm tắt phần này

```
GRAPHQL:
  - Query language CHO CLIENT-SERVER (OLTP), KHÔNG phải graph traversal phức tạp
  - Client chỉ định CHÍNH XÁC field cần → response MIRROR query
  - HẠN CHẾ: không recursive query, không arbitrary search
    (vì query đến từ untrusted client)
  - Implement được trên BẤT KỲ database nào (relational/document/graph)

EVENT SOURCING + CQRS:
  - VIẾT data dưới dạng EVENT LOG (immutable, append-only)
  - ĐỌC data qua MATERIALIZED VIEWS (derived, tối ưu riêng cho mỗi nhu cầu)
  - Ưu điểm: reproducibility, audit log, dễ thêm feature mới,
             throughput cao, communicate intent rõ ràng
  - Nhược điểm: external info cần cẩn thận (determinism),
                GDPR conflict (immutability), side effects khi reprocess
  - YÊU CẦU CỐT LÕI: process events ĐÚNG THỨ TỰ
                      (khó trong distributed system — xem Chapter 10)
```
