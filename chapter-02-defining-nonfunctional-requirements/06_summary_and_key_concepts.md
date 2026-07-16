# Summary & Key Concepts — Chapter 2

---

## 1. Tổng hợp toàn chương

Chương 2 chuyển từ **trade-offs kiến trúc cấp cao** (Chapter 1) sang **nonfunctional requirements** cụ thể: làm sao đo lường và đạt được Performance, Reliability, Scalability, và Maintainability.

```
Chương 2 dùng MỘT case study xuyên suốt:
   Social Network Home Timelines
        │
        ├── Minh họa THROUGHPUT (posts/giây, fan-out writes/giây)
        ├── Minh họa MATERIALIZED VIEW (timeline cache)
        ├── Minh họa FAULT TOLERANCE (exactly-once delivery khi crash)
        └── Minh họa SCALABILITY (edge cases: celebrity, power users)
```

---

## 2. Bốn trụ cột Nonfunctional Requirements

```
1. PERFORMANCE
   └─ Response time (latency thực tế user thấy)
   └─ Throughput (capacity hệ thống)
   └─ Đo bằng PERCENTILES, không phải AVERAGE
   └─ SLO/SLA định nghĩa mục tiêu cụ thể

2. RELIABILITY
   └─ Fault (1 bộ phận lỗi) vs Failure (toàn hệ thống lỗi)
   └─ Fault Tolerance có GIỚI HẠN
   └─ Hardware faults: yếu tương quan → Redundancy hiệu quả
   └─ Software faults: tương quan MẠNH → khó dự đoán
   └─ Human errors: TRIỆU CHỨNG của vấn đề hệ thống, không phải NGUYÊN NHÂN

3. SCALABILITY
   └─ Khả năng đối phó LOAD TĂNG
   └─ KHÔNG có magic scaling sauce
   └─ 3 kiến trúc: Shared-Memory / Shared-Disk / Shared-Nothing
   └─ Nguyên tắc: chia nhỏ components độc lập + đừng phức tạp hóa thừa

4. MAINTAINABILITY
   └─ Operability (vận hành dễ)
   └─ Simplicity (hiểu dễ, qua ABSTRACTION)
   └─ Evolvability (thay đổi dễ, MINIMIZE irreversibility)
```

---

## 3. Bảng thuật ngữ (Glossary)

### Performance

| Thuật ngữ                            | Định nghĩa                                                     |
| ------------------------------------ | -------------------------------------------------------------- |
| **Response time**                    | Tổng thời gian từ request đến response, bao gồm mọi delay      |
| **Throughput**                       | Số request hoặc data xử lý/giây                                |
| **Service time**                     | Thời gian server XỬ LÝ CHỦ ĐỘNG request                        |
| **Queueing delay**                   | Thời gian chờ trước khi được xử lý                             |
| **Latency**                          | Thuật ngữ chung cho thời gian "không được xử lý chủ động"      |
| **Jitter**                           | Biến động trong network delay                                  |
| **Percentile (p50, p95, p99, p999)** | Điểm cắt trong distribution response time đã sort              |
| **Tail latency**                     | Response time ở percentile cao (ảnh hưởng UX)                  |
| **Tail latency amplification**       | Hiện tượng % request chậm tăng khi cần gọi nhiều backend calls |
| **Head-of-line blocking**            | Request chậm chặn request khác phía sau trong queue            |
| **Metastable failure**               | Hệ thống vẫn overload dù load gốc giảm, cần reset              |
| **Retry storm**                      | Client liên tục retry làm tăng load, làm trầm trọng vấn đề     |
| **Exponential backoff**              | Kỹ thuật tăng/randomize thời gian giữa các lần retry           |
| **Circuit breaker**                  | Tạm dừng gửi request đến service đang lỗi                      |
| **Load shedding**                    | Server chủ động từ chối request khi gần overload               |
| **Backpressure**                     | Server yêu cầu client chậm lại                                 |
| **SLO** (Service Level Objective)    | Mục tiêu performance/availability                              |
| **SLA** (Service Level Agreement)    | Hợp đồng + hậu quả nếu không đạt SLO                           |

### Reliability

| Thuật ngữ                          | Định nghĩa                                                    |
| ---------------------------------- | ------------------------------------------------------------- |
| **Fault**                          | Lỗi ở MỘT bộ phận của hệ thống                                |
| **Failure**                        | TOÀN BỘ hệ thống ngừng cung cấp dịch vụ cần thiết             |
| **Fault tolerance**                | Khả năng tiếp tục hoạt động dù có fault                       |
| **SPOF** (Single Point of Failure) | Bộ phận mà nếu lỗi sẽ làm TOÀN HỆ THỐNG fail                  |
| **Fault injection**                | Chủ động kích hoạt fault để test                              |
| **Chaos engineering**              | Discipline tăng confidence vào fault-tolerance qua thí nghiệm |
| **RAID**                           | Redundant Array of Independent Disks                          |
| **Rolling upgrade**                | Patch từng node một, không downtime                           |
| **Cascading failure**              | Lỗi 1 component lan ra component khác                         |
| **Blameless postmortem**           | Văn hóa phân tích incident không đổ lỗi cá nhân               |
| **Sociotechnical system**          | Hệ thống kết hợp con người + công nghệ                        |

### Scalability

| Thuật ngữ                            | Định nghĩa                                                  |
| ------------------------------------ | ----------------------------------------------------------- |
| **Scalability**                      | Khả năng đối phó với load tăng                              |
| **Linear scalability**               | Gấp đôi resource = gấp đôi capacity, performance giữ nguyên |
| **Vertical scaling (scaling up)**    | Chuyển sang máy mạnh hơn                                    |
| **Horizontal scaling (scaling out)** | Thêm nhiều máy                                              |
| **Shared-memory architecture**       | Nhiều threads/processes share RAM trên 1 máy                |
| **Shared-disk architecture**         | Nhiều máy share disk array qua network                      |
| **Shared-nothing architecture**      | Nhiều node độc lập hoàn toàn, coordination ở software level |
| **NAS / SAN**                        | Network Attached Storage / Storage Area Network             |

### Maintainability

| Thuật ngữ                         | Định nghĩa                                             |
| --------------------------------- | ------------------------------------------------------ |
| **Operability**                   | Dễ vận hành, giữ hệ thống chạy trơn                    |
| **Simplicity**                    | Dễ hiểu cho engineer mới                               |
| **Evolvability**                  | Dễ thay đổi trong tương lai                            |
| **Big ball of mud**               | Hệ thống phức tạp, khó hiểu, khó maintain              |
| **Essential complexity**          | Complexity vốn có trong problem domain                 |
| **Accidental complexity**         | Complexity sinh ra từ hạn chế tooling                  |
| **Abstraction**                   | Ẩn implementation detail sau interface sạch            |
| **Irreversibility**               | Hành động không thể hoàn tác — kẻ thù của evolvability |
| **TDD** (Test-Driven Development) | Kỹ thuật Agile hỗ trợ thay đổi an toàn                 |

---

## 4. Mindmap khái niệm

```
                     NONFUNCTIONAL REQUIREMENTS
                                │
      ┌──────────────┬─────────┴─────────┬──────────────┐
      │              │                   │              │
 PERFORMANCE    RELIABILITY        SCALABILITY    MAINTAINABILITY
      │              │                   │              │
  ┌───┴───┐      ┌───┴────┐         ┌────┴────┐    ┌─────┴─────┐
  │       │      │        │         │         │    │     │     │
Response Through-  Fault   Fault   Hiểu    Kiến  Opera- Simpli- Evolva-
 Time    put     vs Fail  Toler-   Load    trúc   bility  city   bility
  │       │      ance      ance     │       │      │      │      │
Percen-  SLO/   (limit)     │    Throughput Shared- Moni- Abstrac-Irrever
tiles    SLA              Hard/   Read/Write Mem/  toring  -tion  sibility
  │       │              Software  ratio   Disk/   │      │      │
Tail    Capacity            │       │      Nothing Docs  Essen-  Loosely
latency  cần          Human Error          │       │     tial/   coupled
  │     thiết              │          Nguyên tắc:  Default Acci-
Queueing               Blameless    chia nhỏ,      Self-  dental
delay                  Postmortem   đơn giản hóa    heal
```

---

## 5. Câu hỏi tự kiểm tra (Self-Check Questions)

1. Phân biệt **response time** và **throughput**. Cho ví dụ từ case study social network.
2. Tại sao **percentile** (đặc biệt p99, p999) lại tốt hơn **average** để đo "trải nghiệm thông thường" của user?
3. Giải thích hiện tượng **metastable failure** và **retry storm**. Kỹ thuật nào giúp ngăn chặn?
4. Phân biệt **fault** và **failure**. Cho ví dụ minh họa tính tương đối giữa 2 khái niệm này.
5. Tại sao **software faults** thường khó xử lý hơn **hardware faults**?
6. Tại sao "**human error**" được coi là **triệu chứng**, không phải **nguyên nhân**, của vấn đề hệ thống?
7. Thế nào là **blameless postmortem**? Mục đích của nó là gì?
8. Giải thích sự khác biệt giữa **shared-memory**, **shared-disk**, và **shared-nothing** architecture.
9. Vì sao **không có "magic scaling sauce"** cho mọi hệ thống?
10. Liệt kê 3 nguyên tắc của **maintainability**, và giải thích từng nguyên tắc với ví dụ.
11. **Abstraction** giúp giảm complexity như thế nào? Cho ví dụ (high-level programming language, SQL).
12. Tại sao **irreversibility** là kẻ thù của **evolvability**? Cho ví dụ.
13. Kể lại case study **Post Office Horizon Scandal** và bài học rút ra về reliability.

---

## 6. Liên kết tới các chương khác

| Chapter 2 đề cập                     | Mở rộng ở                                                         |
| ------------------------------------ | ----------------------------------------------------------------- |
| Materialized views (timeline cache)  | **Chapter 4** — Storage and Retrieval                             |
| Exactly-once semantics               | **Chapter 12** — Stream Processing                                |
| Rolling upgrades                     | **Chapter 5** — Encoding and Evolution                            |
| Sharding                             | **Chapter 7** — Sharding                                          |
| Fault tolerance techniques cụ thể    | **Chapter 6, 10** — Replication, Consensus                        |
| Operations automatic vs manual       | **Chapter 7** — "Operations: Automatic Versus Manual Rebalancing" |
| Timeouts, unbounded delays (network) | **Chapter 9** — The Trouble with Distributed Systems              |
| Observability chi tiết               | **Chapter 9** — "Problems with Distributed Systems"               |

---

## 7. Trích dẫn mở đầu chương (Epigraph)

> _"The Internet was done so well that most people think of it as a natural resource like the Pacific Ocean, rather than something that was man-made. When was the last time a technology with a scale like that was so error-free?"_
> — **Alan Kay**, in interview with _Dr. Dobb's Journal_ (2012)

→ Một lời nhắc nhở về **tầm vóc thành tựu kỹ thuật** khi một hệ thống đạt được reliability ở mức "tự nhiên" như Internet — và đó chính là **mục tiêu** mà các nguyên tắc trong chương này hướng đến.
