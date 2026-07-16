# Summary & Key Concepts — Chapter 1

---

## 1. Tổng hợp toàn chương

Chủ đề xuyên suốt chương 1 là **trade-offs**: hầu hết các câu hỏi thiết kế hệ thống dữ liệu **không có một câu trả lời đúng duy nhất**, mà có nhiều lựa chọn, mỗi lựa chọn có ưu/nhược điểm riêng tùy vào ngữ cảnh.

### Bốn trade-off lớn đã khám phá

```
1. OPERATIONAL vs ANALYTICAL
   └─ Khác nhau về: loại data, access pattern, đối tượng phục vụ
   └─ Kết nối qua: ETL/ELT → Data Warehouse / Data Lake
   └─ Chương liên quan: Chapter 4 (internal data layout khác nhau)

2. CLOUD vs SELF-HOSTING
   └─ Phụ thuộc vào: tình huống cụ thể của bạn (skill, load pattern)
   └─ Cloud native: thay đổi cách kiến trúc hệ thống (separation storage/compute)

3. DISTRIBUTED vs SINGLE-NODE
   └─ Cloud systems về bản chất là distributed
   └─ Nguyên tắc: không vội distribute nếu single-node đủ dùng
   └─ Chương liên quan: Chapter 9 (chi tiết các vấn đề distributed systems)

4. BUSINESS NEEDS vs INDIVIDUAL RIGHTS (Law & Society)
   └─ Privacy regulations (GDPR) định hình kiến trúc hệ thống
   └─ Chương liên quan: Chapter 14 (đạo đức, bias, discrimination)
```

---

## 2. Bảng thuật ngữ (Glossary)

### Operational / Analytical

| Thuật ngữ                                             | Định nghĩa                                                         |
| ----------------------------------------------------- | ------------------------------------------------------------------ |
| **OLTP** (Online Transaction Processing)              | Hệ thống xử lý giao dịch, point query, low-latency reads/writes    |
| **OLAP** (Online Analytical Processing)               | Hệ thống phân tích, aggregate query trên lượng lớn data            |
| **HTAP** (Hybrid Transactional/Analytical Processing) | Hệ thống cố gắng kết hợp OLTP + OLAP không cần ETL                 |
| **Point query**                                       | Truy vấn tìm một số ít record theo key                             |
| **Business Intelligence (BI)**                        | Report giúp management ra quyết định                               |
| **Data Warehouse**                                    | Database riêng cho analytics, chứa copy read-only từ OLTP          |
| **Data Lake**                                         | Repository chứa raw data đa dạng format, không áp đặt schema       |
| **ETL** (Extract-Transform-Load)                      | Quy trình lấy data từ OLTP, transform, load vào warehouse          |
| **ELT**                                               | Giống ETL nhưng transform sau khi load                             |
| **Reverse ETL**                                       | Đưa data/kết quả từ analytical system ngược vào operational system |
| **Feature engineering**                               | Transform data thành vector/matrix cho ML model                    |
| **System of Record**                                  | Nguồn dữ liệu authoritative, ghi đầu tiên                          |
| **Derived Data**                                      | Data được tính toán/biến đổi từ data khác, có thể tái tạo          |
| **Data silos**                                        | Vấn đề data nằm rải rác, khó kết hợp                               |
| **Sushi Principle**                                   | "Raw data is better" — giữ data ở dạng gốc trong data lake         |

### Cloud / Self-Hosting

| Thuật ngữ                              | Định nghĩa                                                            |
| -------------------------------------- | --------------------------------------------------------------------- |
| **IaaS** (Infrastructure as a Service) | Thuê VM trên cloud, tự quản lý software                               |
| **SaaS** (Software as a Service)       | Dùng software hoàn toàn qua vendor, không tự deploy                   |
| **Cloud Native**                       | Kiến trúc thiết kế riêng để tận dụng cloud services                   |
| **Object Storage**                     | Dịch vụ lưu file lớn, ẩn physical machine (S3, Azure Blob)            |
| **Disaggregation**                     | Tách storage và compute thành các service độc lập                     |
| **Multitenancy**                       | Nhiều khách hàng share cùng hardware/service                          |
| **RAID**                               | Redundant Array of Independent Disks — chống mất data khi 1 disk fail |
| **DevOps**                             | Triết lý tích hợp dev + ops thành một team                            |
| **SRE** (Site Reliability Engineering) | Phiên bản DevOps của Google                                           |
| **Metered billing**                    | Trả tiền theo tài nguyên thực dùng, không cần plan trước              |
| **Vendor lock-in**                     | Rủi ro khó migrate khỏi 1 vendor cloud                                |

### Distributed / Single-Node

| Thuật ngữ                               | Định nghĩa                                                                  |
| --------------------------------------- | --------------------------------------------------------------------------- |
| **Node**                                | Một process tham gia trong distributed system                               |
| **Fault tolerance**                     | Khả năng hệ thống tiếp tục hoạt động khi có lỗi                             |
| **Observability**                       | Kỹ thuật thu thập & query data để debug distributed systems                 |
| **Microservices**                       | Kiến trúc chia app thành nhiều service độc lập, mỗi service 1 team          |
| **SOA** (Service-Oriented Architecture) | Tiền thân của microservices                                                 |
| **Serverless / FaaS**                   | Function as a Service — cloud tự quản lý infra, billing theo execution time |
| **HPC** (High-Performance Computing)    | Supercomputing — dùng cho scientific computing                              |
| **RDMA**                                | Remote Direct Memory Access — kỹ thuật network tốc độ cao cho HPC           |
| **Bisection bandwidth**                 | Đo hiệu suất tổng thể của network topology                                  |
| **Distributed transaction**             | Kỹ thuật đảm bảo consistency qua nhiều services/databases                   |

### Law & Society

| Thuật ngữ                                | Định nghĩa                                                      |
| ---------------------------------------- | --------------------------------------------------------------- |
| **GDPR**                                 | General Data Protection Regulation — luật bảo vệ data EU        |
| **CCPA**                                 | California Consumer Privacy Act                                 |
| **Right to be forgotten**                | Quyền yêu cầu xóa data cá nhân                                  |
| **Data minimization (Datensparsamkeit)** | Nguyên tắc chỉ lưu data cần thiết                               |
| **PCI**                                  | Payment Card Industry standard                                  |
| **SOC Type 2**                           | Service Organization Control — tiêu chuẩn compliance cho vendor |

---

## 3. Mindmap khái niệm

```
                        TRADE-OFFS IN DATA SYSTEMS ARCHITECTURE
                                       │
        ┌──────────────────┬──────────┴──────────┬──────────────────┐
        │                  │                     │                  │
   OPERATIONAL vs      CLOUD vs              DISTRIBUTED vs     LAW & SOCIETY
   ANALYTICAL          SELF-HOSTING          SINGLE-NODE
        │                  │                     │                  │
   ┌────┴────┐       ┌─────┴─────┐         ┌─────┴─────┐      ┌──────┴──────┐
   │         │       │           │         │           │      │             │
  OLTP    OLAP    Pros/Cons   Cloud      Khi nào      Micro-  GDPR      Data
   │         │     của Cloud  Native    cần distri-  services  /        Minimi-
   │         │       │        Architect  buted?         /      CCPA     zation
   │         │       │           │         │         Serverless  │         │
Point     Aggre-  Control    Storage/   Fault tol.,    │      Right     Cost-
query     gate    vs Conve-  Compute    Scalability,   │      to be     Benefit
          query   nience     separation  Latency       │      forgotten Analysis
   │         │       │           │         │            │        │
   └────┬────┘       │           │         └─────┬──────┘        │
        │            │           │               │               │
  Data Warehouse  DevOps/SRE  Multitenancy   Nhược điểm:     PCI / SOC 2
  Data Lake                                  Network fails,
  ETL/ELT                                    Latency,
  System of Record /                         Observability,
  Derived Data                               Consistency
```

---

## 4. Câu hỏi tự kiểm tra (Self-Check Questions)

Dùng để ôn tập kiến thức chương 1:

1. **Sự khác biệt chính** giữa OLTP và OLAP là gì? Cho ví dụ access pattern của mỗi loại.
2. Tại sao **data warehouse** lại tách biệt khỏi OLTP database, dù SQL có thể chạy trên cả hai?
3. **Data lake** khác **data warehouse** như thế nào? Tại sao data scientists thường thích data lake hơn?
4. Phân biệt **system of record** và **derived data** — cho 3 ví dụ derived data.
5. Khi nào thì **self-hosting** rẻ hơn cloud? Khi nào **cloud** có lợi thế rõ rệt?
6. **Cloud native architecture** khác gì so với chỉ chạy software truyền thống trên VM cloud?
7. Tại sao **separation of storage and compute** lại quan trọng trong cloud native systems?
8. Liệt kê **5 lý do** để chọn distributed system thay vì single-node.
9. **Microservices** giải quyết vấn đề gì? Tại sao nó "primarily a technical solution to a people problem"?
10. **Serverless** khác gì so với dùng VM truyền thống về mặt billing và quản lý?
11. **HPC (supercomputing)** khác cloud computing như thế nào về cách xử lý node failure?
12. **Right to be forgotten** (GDPR) tạo ra thử thách kỹ thuật gì với append-only logs?
13. **Data minimization** đối lập với triết lý "big data" như thế nào?

---

## 5. Liên kết tới các chương sau

| Chapter 1 đề cập                      | Sẽ được mở rộng ở                                    |
| ------------------------------------- | ---------------------------------------------------- |
| OLTP/OLAP data layout khác nhau       | **Chapter 4** — Storage and Retrieval                |
| Stars/Snowflake schema cho analytics  | **Chapter 3** — Data Models                          |
| Transaction (định nghĩa loose)        | **Chapter 8** — Transactions                         |
| Distributed transactions              | **Chapter 8**                                        |
| Vấn đề network, clock, faults         | **Chapter 9** — The Trouble with Distributed Systems |
| Data pipelines                        | **Chapter 11** — Batch Processing                    |
| Stream processing                     | **Chapter 12** — Stream Processing                   |
| Reliability, Fault Tolerance          | **Chapter 2** — Defining Nonfunctional Requirements  |
| Scalability                           | **Chapter 2**                                        |
| Ethics, bias, discrimination, privacy | **Chapter 14** — Doing the Right Thing               |

---

## 6. Trích dẫn mở đầu chương (Epigraph)

> _"Computing is pop culture. […] Pop culture holds a disdain for history. Pop culture is all about identity and feeling like you're participating. It has nothing to do with cooperation, the past or the future—it's living in the present. I think the same is true of most people who write code for money. They have no idea where [their culture came from]."_
> — **Alan Kay**, in interview with _Dr. Dobb's Journal_ (2012)

→ Lời nhắc nhở rằng hiểu **nguyên tắc nền tảng** (foundational principles) quan trọng hơn việc chạy theo "buzzwords" và công nghệ mới nhất — đây cũng chính là triết lý xuyên suốt cuốn sách.
