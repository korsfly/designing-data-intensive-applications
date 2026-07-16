# DDIA 2nd Edition — Chapter 1: Trade-Offs in Data Systems Architecture

> **Sách:** Designing Data-Intensive Applications, 2nd Edition  
> **Tác giả:** Martin Kleppmann & Chris Riccomini  
> **Chương:** 1 — Trade-Offs in Data Systems Architecture

---

## Mục tiêu chương

Chương 1 giới thiệu các **trade-off** (đánh đổi) cốt lõi trong thiết kế hệ thống dữ liệu. Không có câu trả lời đúng duy nhất — mỗi lựa chọn đều có ưu và nhược điểm riêng. Chương này đặt nền móng cho toàn bộ cuốn sách bằng cách định nghĩa thuật ngữ và khung tư duy.

---

## Cấu trúc tài liệu này

| File                               | Nội dung                                                   |
| ---------------------------------- | ---------------------------------------------------------- |
| `01_Operational_vs_Analytical.md`  | OLTP vs OLAP, Data Warehouse, Data Lake, Systems of Record |
| `02_Cloud_vs_Self_Hosting.md`      | Ưu/nhược điểm Cloud, Cloud Native Architecture, DevOps/SRE |
| `03_Distributed_vs_Single_Node.md` | Khi nào dùng distributed, Microservices, Serverless, HPC   |
| `04_Data_Law_and_Society.md`       | GDPR, đạo đức dữ liệu, data minimization                   |
| `05_Summary_and_Key_Concepts.md`   | Tổng hợp toàn chương, bảng thuật ngữ, mindmap khái niệm    |

---

## Big Picture — 4 Trade-Off Chính

```
┌─────────────────────────────────────────────────┐
│         TRADE-OFFS IN DATA SYSTEMS              │
│                                                 │
│  1. Operational  ←────────→  Analytical         │
│     (OLTP)                   (OLAP)             │
│                                                 │
│  2. Cloud        ←────────→  Self-Hosting       │
│                                                 │
│  3. Distributed  ←────────→  Single-Node        │
│                                                 │
│  4. Business     ←────────→  Society/Law        │
│     Needs                    (GDPR, ethics)     │
└─────────────────────────────────────────────────┘
```

---

## Các chương liên quan trong sách

- **Chapter 02:** Reliability, Scalability, Maintainability (Nonfunctional Requirements)
- **Chapter 03:** Data Models & Query Languages
- **Chapter 04:** Storage & Retrieval (OLTP vs OLAP internals)
- **Chapter 08:** Transactions (ACID, distributed transactions)
- **Chapter 09:** Problems with Distributed Systems
- **Chapter 14:** Ethics & Law (GDPR, bias, privacy)
