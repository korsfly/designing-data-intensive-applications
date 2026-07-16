# DataFrames, Matrices, và Arrays

---

## 1. Bối cảnh: Data Model cho Analytics/Science, không phải OLTP

```
Các model đã thấy (relational, document, graph):
   Dùng cho CẢ transaction processing VÀ analytics

DataFrames + Multidimensional Arrays (matrices):
   CHỦ YẾU xuất hiện trong context ANALYTICAL/SCIENTIFIC
   HIẾM khi dùng trong OLTP systems
```

### Hệ thống hỗ trợ DataFrame

| Ngôn ngữ/Library    | Mô tả                                            |
| ------------------- | ------------------------------------------------ |
| **R**               | Ngôn ngữ thống kê, có DataFrame native           |
| **Pandas** (Python) | Library phổ biến nhất cho DataFrame trong Python |
| **Apache Spark**    | Distributed DataFrame cho big data               |
| **ArcticDB**        | DataFrame cho dữ liệu tài chính                  |
| **Dask**            | Parallel computing, DataFrame API                |

**Use cases chính:**

- Chuẩn bị data để train ML models
- Data exploration
- Statistical data analysis
- Data visualization

---

## 2. DataFrame trông giống gì?

```
Ở GÓC NHÌN ĐẦU TIÊN:
   DataFrame ≈ Table trong relational database
              ≈ Spreadsheet
```

### Operators tương tự Relational

| Operation                  | Tương đương relational              |
| -------------------------- | ----------------------------------- |
| Apply function cho mọi row | Map/transform                       |
| Filter rows theo condition | `WHERE`                             |
| Group + aggregate          | `GROUP BY`                          |
| Join 2 DataFrame theo key  | `JOIN` (DataFrame gọi là **merge**) |

### Khác biệt quan trọng: Cách Manipulate

```
SQL: ngôn ngữ DECLARATIVE — viết 1 query hoàn chỉnh

DataFrame: MANIPULATE qua MỘT LOẠT COMMANDS
           (sửa structure/content TỪNG BƯỚC)

→ Khớp với workflow THỰC TẾ của data scientist:
   "WRANGLE" data DẦN DẦN để tìm câu trả lời cho câu hỏi đang hỏi
```

**Đặc điểm vận hành:**

- Thường thao tác trên **bản copy riêng** của dataset (thường trên máy local của data scientist)
- Kết quả cuối có thể được SHARE với người khác

---

## 3. DataFrame vs Relational Model — Khác biệt sâu hơn

> DataFrame APIs cung cấp **đa dạng operation HƠN HẲN** so với relational database, và data model thường được dùng theo cách RẤT KHÁC với relational data modeling truyền thống.

### Use case tiêu biểu: Transform Relational → Matrix

```
Một use case PHỔ BIẾN của DataFrame:
   Transform data từ relational-like representation
   → MATRIX hoặc multidimensional array representation
   (đây là FORM mà nhiều thuật toán ML cần cho input)
```

### Ví dụ minh họa: Movie Ratings

```
BÊN TRÁI (Relational table):
   user_id | movie_id | rating
   --------|----------|-------
     1     |    101   |   5
     1     |    102   |   3
     2     |    101   |   4

BÊN PHẢI (Matrix — giống pivot table):
            movie_101  movie_102  movie_103 ...
   user_1       5          3          -
   user_2       4          -          -
   ...
```

```
Matrix này THƯA (SPARSE) — nhiều cặp user-movie KHÔNG có data
   → KHÔNG VẤN ĐỀ — DataFrame và library hỗ trợ sparse array
     (vd: NumPy cho Python) xử lý DỄ DÀNG

Matrix có thể có HÀNG NGHÌN cột
   → KHÔNG phù hợp tốt cho relational database
   → DataFrame xử lý TỰ NHIÊN
```

---

## 4. Chuyển đổi Non-numerical Data thành Numbers

> Matrix CHỈ chứa số. Cần kỹ thuật chuyển dữ liệu KHÔNG phải số thành số trong matrix.

### Kỹ thuật 1: Scaling

```
Dates → scale thành floating-point number trong range phù hợp
```

### Kỹ thuật 2: One-Hot Encoding

```
Cho columns chỉ nhận 1 trong SỐ ÍT giá trị cố định
   (vd: genre của movie: comedy/drama/horror...)

→ Tạo 1 COLUMN cho MỖI giá trị có thể
→ Với mỗi row (movie), đặt:
     1 ở column TƯƠNG ỨNG với genre
     0 ở các column KHÁC

Movie    comedy  drama  horror
Movie A     1       0      0
Movie B     0       1      0
```

> One-hot encoding cũng generalize tốt cho movie thuộc **NHIỀU genre** (đặt 1 ở nhiều cột tương ứng).

---

## 5. Ứng dụng: Linear Algebra cho ML

```
Sau khi data ở dạng MATRIX SỐ
   → Áp dụng được LINEAR ALGEBRA OPERATIONS
   → Đây là NỀN TẢNG của nhiều thuật toán ML

Ví dụ: Data trong Figure 3-9 có thể là PHẦN của 1 hệ thống
       RECOMMEND MOVIE mà user có thể thích
```

> **Tính linh hoạt của DataFrame:** Cho phép data EVOLVE DẦN DẦN từ relational form → matrix representation, trong khi vẫn cho data scientist **TOÀN QUYỀN KIỂM SOÁT** representation phù hợp nhất cho mục tiêu phân tích/training model.

---

## 6. Các Database/Use Case chuyên biệt khác

### Array Databases

```
Database CHUYÊN BIỆT cho lưu trữ multidimensional array của số:
   Ví dụ: TileDB

Thường dùng cho SCIENTIFIC DATASETS:
   - Geospatial measurements (raster data trên grid đều)
   - Medical imaging
   - Quan sát từ kính thiên văn (astronomical telescopes)
```

### DataFrame trong Tài chính (Finance)

```
DataFrame được dùng RỘNG RÃI trong ngành tài chính
   để biểu diễn TIME-SERIES DATA
   (giá tài sản, giao dịch theo thời gian)
```

### DataFrame trong Batch Processing Frameworks

```
Do PHỔ BIẾN với data scientist
   → DataFrames được THÊM vào các batch processing framework:
        Apache Spark
        Apache Flink

→ Sẽ quay lại chủ đề này ở Chapter 11 (Batch Processing)
```

---

## Tóm tắt phần này

```
DataFrames:
  - Data model dành cho ANALYTICAL/SCIENTIFIC context, ÍT dùng trong OLTP
  - TRÔNG giống table/spreadsheet, NHƯNG manipulate qua SERIES OF COMMANDS
    (không phải 1 declarative query hoàn chỉnh như SQL)
  - Use case quan trọng: transform relational data → MATRIX cho ML

Matrices/Arrays:
  - Chỉ chứa SỐ
  - Cần encoding: scaling (dates), one-hot encoding (categorical)
  - Nền tảng cho LINEAR ALGEBRA operations trong ML

Hệ thống liên quan:
  - R, Pandas, Spark, ArcticDB, Dask (DataFrame)
  - TileDB (Array Database, cho scientific data)
  - Spark/Flink cũng có DataFrame support (Chapter 11)
```
