# Operational vs Analytical Systems

---

## 1. Bối cảnh: Ai làm việc với dữ liệu?

Trong một tổ chức, có nhiều nhóm người khác nhau làm việc với dữ liệu:

| Vai trò                 | Công việc                                                               |
| ----------------------- | ----------------------------------------------------------------------- |
| **Backend Engineers**   | Xây dựng services phục vụ user, đọc/ghi dữ liệu theo request            |
| **Business Analysts**   | Tạo report, phân tích hoạt động tổ chức (BI)                            |
| **Data Scientists**     | Tìm insight mới, xây dựng ML/AI (recommendations, spam filter,...)      |
| **Data Engineers**      | Tích hợp operational và analytical systems, quản lý data infrastructure |
| **Analytics Engineers** | Mô hình hóa và transform data cho analysts và data scientists           |

---

## 2. Hai loại hệ thống chính

### Operational Systems (OLTP — Online Transaction Processing)

- Chứa dữ liệu được **tạo ra** bởi users và backend services
- Application code **đọc và ghi** dữ liệu theo hành động của user
- Phục vụ external users trực tiếp hoặc gián tiếp

### Analytical Systems (OLAP — Online Analytical Processing)

- Chứa **bản sao read-only** của dữ liệu từ operational systems
- Tối ưu cho các truy vấn analytics (aggregation, scanning lượng lớn records)
- Phục vụ internal analysts và data scientists

---

## 3. So sánh OLTP và OLAP

| Đặc điểm             | Operational (OLTP)                   | Analytical (OLAP)                   |
| -------------------- | ------------------------------------ | ----------------------------------- |
| **Pattern đọc**      | Point query (lấy record theo key)    | Aggregate trên lượng lớn records    |
| **Pattern ghi**      | Create / Update / Delete từng record | Bulk import (ETL) hoặc event stream |
| **User điển hình**   | End user của web/mobile app          | Internal analyst (ra quyết định)    |
| **Loại query**       | Fixed, predefined trong app code     | Ad-hoc, tự do viết SQL              |
| **Số lượng query**   | Nhiều query nhỏ                      | Ít query nhưng phức tạp             |
| **Dữ liệu đại diện** | Trạng thái hiện tại (current state)  | Lịch sử sự kiện theo thời gian      |
| **Dataset size**     | Gigabytes → Terabytes                | Terabytes → Petabytes               |

> **Lưu ý:** "Online" trong OLAP có nghĩa là analysts query **interactively** (khám phá dữ liệu), không phải chỉ chạy report định sẵn.

---

## 4. Product Analytics / Real-Time Analytics

Ngoài OLTP và OLAP truyền thống, có một dạng hệ thống hybrid:

- **Ví dụ:** Apache Pinot, Apache Druid, ClickHouse
- Phân tích dữ liệu theo kiểu OLAP nhưng **nhúng vào user-facing products**
- Ingest data **real-time**, tối ưu cho **low-latency query**
- Khác với OLAP truyền thống (ingest theo batch, tối ưu throughput)

---

## 5. Data Warehouse

### Tại sao cần Data Warehouse?

Vấn đề khi để analyst query thẳng vào OLTP systems:

1. **Data silos:** Data nằm rải rác ở nhiều operational systems khác nhau
2. **Schema không phù hợp:** Schema tốt cho OLTP thường không tốt cho analytics
3. **Performance impact:** Query analytics nặng sẽ ảnh hưởng OLTP performance
4. **Security/Compliance:** OLTP systems thường nằm trong network riêng

### ETL — Extract, Transform, Load

```
Operational DB 1 ──┐
Operational DB 2 ──┼──► ETL Process ──► Data Warehouse ──► BI Tools
SaaS APIs (CRM,...) ──┘                                    (Tableau, Looker,...)
```

- **Extract:** Lấy data từ OLTP (periodic dump hoặc continuous stream)
- **Transform:** Chuyển đổi sang schema phù hợp cho analytics, làm sạch data
- **Load:** Đẩy vào Data Warehouse

Đôi khi thứ tự đảo ngược: **ELT** (transform sau khi đã load vào warehouse).

**ETL connectors phổ biến:** Fivetran, Singer, Airbyte

### HTAP — Hybrid Transactional/Analytical Processing

- Mục tiêu: kết hợp OLTP + analytics trong một hệ thống, không cần ETL
- Tuy nhiên: nhiều HTAP system thực chất là OLTP + analytical system riêng biệt ẩn phía sau
- **Không thay thế** data warehouse — vẫn hữu ích khi một app vừa cần OLAP query vừa cần OLTP latency thấp (ví dụ: fraud detection)

---

## 6. Data Lake

### Tại sao Data Warehouse chưa đủ?

Data scientists thường cần:

- **Feature engineering:** Transform data thành vectors/matrices cho ML (khó dùng SQL)
- **NLP/Computer Vision:** Làm việc với text, images — không phải relational data
- Dùng Python (Pandas, scikit-learn), R, Spark — không phải SQL

### Data Lake là gì?

- **Centralized data repository** chứa mọi loại data có thể hữu ích cho analytics
- Lấy data từ operational systems qua ETL
- **Không áp đặt** schema, format, hay data model cụ thể
- Có thể chứa: database records (Avro, Parquet), text, images, videos, sensor data, feature vectors,...
- Thường **rẻ hơn** relational storage (dùng object store như S3)

### Kiến trúc hiện đại

```
Operational Systems
       │
       ▼ ETL
   Data Lake (raw data)
       │
       ├──► Data Warehouse (relational, BI)
       │
       └──► Data Science workloads (Python, Spark, ML)
```

> **Sushi Principle:** "Raw data is better" — data lake chứa raw data, để mỗi consumer tự transform theo nhu cầu.

### Xu hướng mới hơn (DataOps)

- **DataOps Manifesto:** Chú trọng quản lý và vận hành analytical pipelines
- **Governance, Privacy, Compliance:** GDPR, CCPA ảnh hưởng đến thiết kế pipelines
- **Stream processing:** Thay vì chạy analytics theo batch (hàng ngày), xử lý events theo stream (seconds) — xem Chapter 12
- **Reverse ETL:** Đưa output của analytical systems (ví dụ ML model) vào operational systems (ví dụ recommendations). Tools: TFX, Kubeflow, MLflow

---

## 7. Systems of Record và Derived Data

Phân biệt quan trọng để hiểu **luồng data** trong hệ thống:

### Systems of Record (Source of Truth)

- Chứa **phiên bản authoritative** của data
- Data mới được **ghi vào đây đầu tiên**
- Mỗi fact được lưu **đúng một lần** (normalized)
- Nếu có conflict, value ở đây là **đúng theo định nghĩa**

### Derived Data Systems

- Data là kết quả của việc **transform hoặc xử lý** data từ nguồn khác
- Có thể tái tạo nếu mất (từ source gốc)
- Ví dụ: cache, indexes, materialized views, denormalized values, trained ML models

```
                  ┌── Derived: Index (tìm kiếm nhanh)
System of Record ─┤── Derived: Cache (đọc nhanh)
                  └── Derived: Analytical system (OLAP)
```

> **Quan trọng:** Phân biệt này phụ thuộc vào **cách bạn dùng** tool, không phải tool là gì. Một database có thể là cả hai tùy ngữ cảnh.

### Thách thức: Propagating Updates

Khi data trong system of record thay đổi → derived data cần được cập nhật.  
Nhiều databases không hỗ trợ tốt việc này. Chapter 11 sẽ thảo luận **data pipelines** như một giải pháp.

---

## Tóm tắt phần này

```
OLTP ──ETL──► OLAP (Data Warehouse)
               │
               └──(raw)──► Data Lake ──► ML / Data Science

System of Record ──────────► Derived Data Systems
(authoritative)               (cache, index, analytics)
```

**Key insight:** Operational và analytical systems tách nhau vì chúng phục vụ **đối tượng khác nhau**, có **access pattern khác nhau**, và cần **schema/layout khác nhau**. Không có one-size-fits-all solution.
