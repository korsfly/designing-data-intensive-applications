# DDIA 2nd Edition — Chapter 2: Defining Nonfunctional Requirements

> **Sách:** Designing Data-Intensive Applications, 2nd Edition  
> **Tác giả:** Martin Kleppmann & Chris Riccomini  
> **Chương:** 2 — Defining Nonfunctional Requirements

---

## Mục tiêu chương

Nếu Chapter 1 nói về **functional trade-offs** (loại hệ thống nào để chọn), Chapter 2 đi sâu vào **nonfunctional requirements** — những thuộc tính "ngầm" nhưng cực kỳ quan trọng của một hệ thống: performance, reliability, scalability, maintainability.

> Epigraph mở đầu chương:
> _"The Internet was done so well that most people think of it as a natural resource like the Pacific Ocean, rather than something that was man-made. When was the last time a technology with a scale like that was so error-free?"_
> — Alan Kay

---

## Cấu trúc tài liệu này

| File                                        | Nội dung                                                                |
| ------------------------------------------- | ----------------------------------------------------------------------- |
| `01_Case_Study_Social_Network_Timelines.md` | Case study: home timeline của social network — fan-out, materialization |
| `02_Describing_Performance.md`              | Response time, throughput, latency, percentiles, SLO/SLA                |
| `03_Reliability_and_Fault_Tolerance.md`     | Fault vs Failure, hardware/software faults, human reliability           |
| `04_Scalability.md`                         | Load, shared-memory/disk/nothing architectures, nguyên tắc scalability  |
| `05_Maintainability.md`                     | Operability, Simplicity, Evolvability                                   |
| `06_Summary_and_Key_Concepts.md`            | Tổng hợp toàn chương, glossary, mindmap, câu hỏi ôn tập                 |

---

## Big Picture — 4 Nonfunctional Requirements Chính

```
┌──────────────────────────────────────────────────────┐
│           NONFUNCTIONAL REQUIREMENTS                 │
│                                                       │
│  1. PERFORMANCE   — đo bằng response time, throughput │
│                                                       │
│  2. RELIABILITY   — tiếp tục hoạt động đúng khi có lỗi│
│                                                       │
│  3. SCALABILITY   — khả năng đối phó với tăng load    │
│                                                       │
│  4. MAINTAINABILITY                                   │
│       ├─ Operability  (dễ vận hành)                   │
│       ├─ Simplicity   (dễ hiểu)                       │
│       └─ Evolvability (dễ thay đổi)                   │
└──────────────────────────────────────────────────────┘
```

---

## Case Study xuyên suốt chương

Chương sử dụng một ví dụ thực tế để minh họa các khái niệm: **xây dựng home timeline cho social network kiểu X (Twitter)**.

- 500 triệu posts/ngày (~5,800 posts/giây trung bình, đỉnh 150,000/giây)
- Trung bình mỗi user follow 200 người, có 200 followers
- Vấn đề: làm sao trả về timeline nhanh mà không quá tải hệ thống?

→ Giải pháp: **materialization** (precompute) + **fan-out** khi viết, dùng cache khi đọc.

---

## Các chương liên quan

- **Chapter 1:** Trade-Offs in Data Systems Architecture (nền tảng)
- **Chapter 4:** Storage and Retrieval (materialized views, indexes)
- **Chapter 6, 10:** Kỹ thuật fault tolerance cụ thể (Replication, Consensus)
- **Chapter 7:** Sharding (operations: automatic vs manual rebalancing)
- **Chapter 9:** The Trouble with Distributed Systems (timeouts, delays)
- **Chapter 12:** Stream Processing (exactly-once semantics)
