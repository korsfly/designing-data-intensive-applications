# Data Storage for Analytics

---

## 1. Tại sao Analytics cần Storage Engine KHÁC?

```
Bề ngoài: Data Warehouse và relational OLTP database TRÔNG GIỐNG NHAU
   (cả 2 đều có SQL query interface)

NHƯNG: Internals KHÁC HOÀN TOÀN vì:
   OLTP: tối ưu cho NHIỀU requests nhỏ, mỗi request đọc/ghi
         SỐ ÍT records, cần response TIME NHANH
   Analytics: tối ưu cho query PHỨC TẠP scan qua
              LƯỢNG LỚN records

→ Nhiều database vendor TẬP TRUNG hỗ trợ MỘT trong 2 workload,
  không phải cả hai
```

---

## 2. Cloud Data Warehouses

### Truyền thống (On-Premises)

```
Teradata, Vertica, SAP HANA → Commercial license, on-premises
(Cũng cung cấp cloud-based solution)
```

### Cloud-Native Data Warehouses

```
Khi customers CHUYỂN SANG CLOUD:
   Google BigQuery, Amazon Redshift, Snowflake
   → Tận dụng cloud infrastructure:
      - OBJECT STORAGE (S3, GCS,...)
      - SERVERLESS COMPUTATION
      - DISAGGREGATED storage & compute (Chapter 1)

Lợi ích:
   - Integrate TỐT HƠN với cloud services khác
   - Automatic log ingestion từ nhiều nguồn
   - ELASTIC hơn: điều chỉnh compute/storage RIÊNG BIỆT theo nhu cầu
```

### Open Source Data Warehouses (Data Lake era)

```
Apache Hive, Trino, Apache Spark
→ Khi data warehouse chuyển sang Data Lake trên object storage,
  open source warehouse BẮT ĐẦU TỪ DỒN thành CÁC PHẦN RIÊNG:
```

| Component          | Vai trò                                                                                                                   | Ví dụ                                                                 |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| **Query Engine**   | Parse SQL, optimize thành execution plan, execute (thường parallel + distributed)                                         | Trino, Apache DataFusion, Presto (có thể dùng Spark/Flink để execute) |
| **Storage Format** | Cách encode rows thành bytes trong file, thường lưu trên object storage                                                   | Parquet, ORC, Lance, Nimble                                           |
| **Table Format**   | Quản lý IMMUTABLE files: định nghĩa set files nào = 1 table + schema; hỗ trợ insert/delete, time travel, GC, transactions | Apache Iceberg, Databricks Delta format                               |
| **Data Catalog**   | Định nghĩa tables nào = 1 database; create/rename/drop tables; chạy như standalone service (REST interface)               | Snowflake Polaris, Databricks Unity Catalog, Apache Iceberg Catalog   |

> Trước đây, tất cả component trên được TÍCH HỢP trong 1 hệ thống như Hive. Tách biệt chúng cho phép **data discovery và data governance** access catalog metadata độc lập.

---

## 3. Column-Oriented Storage — Nguyên tắc cốt lõi

### Vấn đề với Row-Oriented Storage cho Analytics

```
Fact table trong analytics: thường >100 cột
Typical analytics query: chỉ cần 4-5 cột

Ví dụ query (Example 4-1):
   SELECT weekday, category, SUM(quantity)
   FROM fact_sales
   JOIN dim_date ON date_key
   JOIN dim_product ON product_sk
   WHERE year = 2024
   AND category IN ('Fresh fruit', 'Candy')
   GROUP BY weekday, category;

→ Chỉ cần 3 cột của fact_sales: date_key, product_sk, quantity
→ Bỏ qua TẤT CẢ cột KHÁC (100+ cột)
```

### Row-Oriented vs Column-Oriented

```
ROW-ORIENTED (OLTP thông thường):
   TẤT CẢ values của 1 ROW lưu CẠNh NHAU
   (Document DB: toàn bộ document = 1 contiguous sequence of bytes)

   → Để xử lý query trên, phải:
     Load TẤT CẢ rows (100+ attributes MỖI ROW) vào memory
     Parse, filter → CHẬM, TỐN I/O

COLUMN-ORIENTED (Columnar):
   TẤT CẢ values của 1 CỘT lưu CẠNH NHAU
   → Query chỉ cần đọc và parse CÁC CỘT DÙNG TRONG QUERY
   → TIẾT KIỆM ĐÁNG KỂ I/O và CPU
```

### Ví dụ minh họa (Figure 4-7)

```
Fact table: date_key | product_sk | store_sk | quantity | net_price | ...

ROW-ORIENTED lưu:
   [row 1: date=20240101, prod=31, store=5, qty=3, price=2.5, ...]
   [row 2: date=20240101, prod=68, store=2, qty=1, price=7.0, ...]
   ...

COLUMN-ORIENTED lưu:
   date_key file:    [20240101, 20240101, 20240102, ...]
   product_sk file:  [31, 68, 69, 31, ...]
   store_sk file:    [5, 2, 3, 5, ...]
   quantity file:    [3, 1, 2, 4, ...]
   net_price file:   [2.5, 7.0, 1.2, ...]

Query chỉ cần: date_key, product_sk, quantity
   → Chỉ load 3 files, BỎ QUA tất cả file còn lại
```

### Khôi phục 1 Row từ Columnar Storage

```
Column storage: mỗi column lưu rows THEO CÙNG THỨ TỰ

Để khôi phục ROW KE N:
   Lấy item THỨ N từ MỖI column file
   → Ghép lại = ROW THỨ N

(Đây là LÝ DO columnar storage phải giữ thứ tự nhất quán)
```

### Thực tế: Không lưu 1 Column file KHỔNG LỒ

```
Columnar storage engine KHÔNG lưu TOÀN BỘ column (có thể hàng tỉ rows)
   trong 1 file duy nhất

→ Break table thành BLOCKS của hàng nghìn đến hàng triệu rows
   Trong mỗi block: lưu values của TỪNG COLUMN riêng

Vì nhiều query giới hạn theo DATE RANGE:
   → Thường mỗi block chứa rows cho 1 TIMESTAMP RANGE cụ thể
   → Query chỉ cần load columns cần thiết,
     TRONG CÁC BLOCK OVERLAP với date range của query
```

### Ai dùng Columnar Storage?

```
Gần như TẤT CẢ analytical databases ngày nay:
   Cloud-scale: Snowflake, BigQuery, Azure Synapse
   Single-node embedded: DuckDB
   Product analytics: Apache Pinot, Apache Druid
   Storage formats: Parquet, ORC, Lance, Nimble
   In-memory analytics: Apache Arrow, Pandas/NumPy
   Time-series: InfluxDB IOx, TimescaleDB
```

> **CẢNH BÁO nhầm lẫn thuật ngữ:** Đừng nhầm **column-oriented databases** (lưu values của cùng column gần nhau) với **wide-column / column-family model** (Bigtable, HBase, Accumulo — lưu TẤT CẢ values của 1 ROW gần nhau → thực ra là ROW-ORIENTED, dù tên gọi có "column").

---

## 4. Column Compression

### Tại sao Column Storage nén tốt?

```
Sequence của values TRONG 1 column:
   → Thường có NHIỀU REPETITION (trùng lặp)
   → Dấu hiệu TỐT cho compression

Nhiều kỹ thuật compression khác nhau tùy data trong column
Kỹ thuật HIỆU QUẢ ĐẶC BIỆT trong data warehouse: BITMAP ENCODING
```

### Bitmap Encoding (Figure 4-8)

```
Số DISTINCT VALUES trong 1 column thường NHỎ so với số rows
   Ví dụ: retailer có HÀNG TỈ sales transaction,
          nhưng CHỈ 100,000 sản phẩm KHÁC NHAU

→ Với column có n distinct values:
   Tạo n BITMAPS RIÊNG BIỆT (1 bitmap / distinct value)
   MỖI bitmap: 1 BIT / ROW
      Bit = 1: row này CÓ value đó
      Bit = 0: row này KHÔNG CÓ value đó
```

### Run-Length Encoding

```
Các bitmap thường SPARSE (nhiều 0)

→ Dùng RUN-LENGTH ENCODING: đếm số lượng 0s hoặc 1s LIÊN TIẾP,
  lưu SỐ ĐẾM thay vì từng bit

Kỹ thuật như ROARING BITMAPS:
   Switch GIỮa 2 representation bitmap,
   dùng cái NÉN NHẤT tại mỗi phần
   → Encoding HIỆU QUẢ ĐÁNG KỂ
```

### Bitmap Indexes cho Query Analytics

```
Bitmap index PHÙ HỢP HOÀN HẢO cho query phổ biến trong data warehouse:

WHERE product_sk IN (31, 68, 69):
   Load 3 bitmaps (product_sk=31, 68, 69)
   → Tính bitwise OR → NHANH (CPU bitwise operation)

WHERE product_sk = 30 AND store_sk = 3:
   Load bitmap product_sk=30, bitmap store_sk=3
   → Tính bitwise AND
   → Vì cả 2 columns lưu rows CÙNG THỨ TỰ:
     bit thứ k trong column A = CÙNG ROW với bit thứ k trong column B
```

> **Ứng dụng ngoài analytics:** Bitmap cũng dùng cho GRAPH QUERIES — ví dụ tìm tất cả users của social network được follow bởi user X VÀ cũng follow user Y.

---

## 5. Sort Order trong Column Storage

### Thứ tự mặc định: Insert Order

```
Đơn giản nhất: lưu rows THEO THỨ TỰ INSERT
   (insert row mới = append vào CUỐI mỗi column)
```

### Imposed Sort Order: Indexing Mechanism

```
CÓ THỂ áp đặt THỨ TỰ, giống SSTables trước đó
→ Dùng như INDEXING MECHANISM

Database admin CHỌN columns để SORT TABLE
   (dựa trên knowledge về common queries)

Ví dụ:
   Query thường target DATE RANGE (last month)
   → Đặt date_key làm FIRST SORT KEY
   → Query chỉ cần scan rows của THÁNG VỪA RỒI,
     thay vì scan TẤT CẢ rows

   Nếu date_key là first sort key:
   → product_sk có thể làm SECOND SORT KEY
   → TẤT CẢ sales cùng product trong cùng ngày
     được GOM LẠI trong storage
   → Hữu ích cho query group/filter sales by product
     trong date range nhất định
```

### Sort Order giúp Compression tốt hơn

```
PRIMARY SORT COLUMN: nếu ÍT distinct values
   → Sau khi sort: CHUỖI DÀI của cùng 1 value lặp lại
   → Run-length encoding NÉN XUỐNG chỉ còn VÀI KiB
     dù table có hàng TỈ rows

SECONDARY, TERTIARY sort key: JUMBLED hơn
   → Không có runs dài → NÉN KÉM HƠN

→ NHƯNG: có first few columns SORTED = WIN TỔNG THỂ
```

---

## 6. Writing vào Column-Oriented Storage

### Vấn đề với Columnar Storage cho Writes

```
Column-oriented storage + compression + sorting:
   → READ queries NHANH HƠN RẤT NHIỀU

NHƯNG writes TRONG data warehouse: thường là BULK IMPORTS (ETL)

Viết 1 INDIVIDUAL ROW GIỮA sorted table:
   → RẤT KÉM HIỆU QUẢ: phải REWRITE TẤT CẢ compressed columns
                         từ VỊ TRÍ INSERT trở đi
```

### Giải pháp: Log-Structured Approach + Batch Writes

```
BULK WRITE: amortize (phân bổ đều) cost của rewrite → HỢP LÝ

1 số columnar storage engine dùng LOG-STRUCTURED APPROACH:
   1. TẤT CẢ writes → TRƯỚC HẾT vào ROW-ORIENTED, sorted,
      IN-MEMORY store
   2. Khi đủ writes tích lũy → MERGE với column-encoded files
      trên disk, write thành NEW files theo bulk

OLD files KHÔNG THAY ĐỔI (immutable) + new files write một lần
→ OBJECT STORAGE phù hợp để lưu các files này

QUERIES cần examine CẢ:
   - Column data trên DISK (lâu hơn)
   - Recent writes trong MEMORY

Query execution engine HÀN DATA từ 2 nguồn, ẨN SỰ KHÁC BIỆT này với user
   → Analyst thấy data insert/update/delete NGAY LẬP TỨC trong subsequent queries

Databases làm điều này: Snowflake, Vertica, Apache Pinot, Apache Druid,...
```

---

## 7. Query Execution: Compilation và Vectorization

### Vấn đề: CPU time cho Analytics Queries

```
Analytics query phức tạp scan HÀNG TRIỆU rows:
   KHÔNG CHỈ lo về lượng data đọc từ disk
   CÒN lo về CPU TIME cần để execute complex operators

Query plan: nhiều STAGES (operators), có thể distributed
            trên nhiều máy cho parallel execution

Query engine cần:
   - Tìm rows có value trong 1 SET nhất định (phần của JOIN)
   - Check điều kiện (vd: giá trị > 15)
   - Look at NHIỀU columns cùng row (vd: product = "bananas" VÀ store = X)
```

### Cách đơn giản nhất: Interpreter

```
Iterator qua TỪNG ROW, check data structure biểu diễn query
để biết cần làm comparison/calculation GÌ trên column NÀO

→ QUÁ CHẬM cho nhiều analytics purpose
```

### Approach 1: Query Compilation (JIT Compilation)

```
Query engine lấy SQL query → GENERATE CODE để execute nó
   Code iterate qua rows 1 by 1, look at values trong columns,
   perform comparisons/calculations, copy values vào output buffer
   nếu conditions satisfied

Compile generated code → MACHINE CODE
   (thường dùng existing compiler như LLVM)
   Run trên column-encoded data đã load vào memory

Tương tự: JIT compilation trong JVM và runtimes tương tự
```

### Approach 2: Vectorized Processing

```
KHÔNG compile query — INTERPRET, nhưng làm NHANH bằng cách:
   Xử lý NHIỀU VALUES từ 1 column CÙNG LÚC (batch)
   thay vì iterate qua rows 1 by 1

FIXED SET of predefined operators build sẵn trong database
   → Truyền arguments vào, nhận về BATCH OF RESULTS

Ví dụ:
   Truyền column product_sk + ID "bananas"
   → Equality operator trả về BITMAP (1 bit/value trong column,
     = 1 nếu match)

   Truyền column store_sk + store ID X
   → Equality operator trả về BITMAP khác

   Bitwise AND hai bitmaps
   → BITMAP kết quả: 1 = tất cả sales của bananas tại store X
```

### Tại sao cả 2 approach NHANH trên Modern CPUs

```
Cả Query Compilation VÀ Vectorized Processing đều đạt performance tốt nhờ:

   1. Prefer SEQUENTIAL MEMORY ACCESS over random access
      → Giảm cache miss

   2. Làm phần lớn work trong TIGHT INNER LOOPS
      (số lượng instruction nhỏ, không có function call)
      → CPU instruction processing pipeline BẬN và hiệu quả
      → Tránh branch misprediction

   3. Tận dụng PARALLELISM:
      - Multiple THREADS
      - SIMD instructions (Single-Instruction, Multiple-Data)

   4. Operate DIRECTLY trên COMPRESSED DATA
      (không decode ra in-memory representation riêng)
      → Tiết kiệm memory allocation và copying cost
```

---

## 8. Materialized Views và Data Cubes

### Nhắc lại Materialized Views

```
Materialized View (xem lại từ Chapter 2, 3):
   Object giống table, nội dung = KẾT QUẢ của 1 QUERY
   COPY THỰC SỰ được VIẾT RA DISK

Virtual View: chỉ là SHORTCUT để viết query
   SQL engine EXPAND ra underlying query ON THE FLY

Khi underlying data THAY ĐỔI:
   Materialized view cần ĐƯỢC UPDATE tương ứng
   (1 số DB tự động; system như Materialize chuyên về việc này)
   → Write: NHIỀU CÔNG VIỆC HƠN
   → Read: CÓ THỂ CẢI THIỆN performance đáng kể
     (nếu cùng 1 query được thực hiện NHIỀU LẦN)
```

### Materialized Aggregates — Dạng Đặc Biệt

```
Data warehouse query THƯỜNG dùng AGGREGATE FUNCTIONS:
   COUNT, SUM, AVG, MIN, MAX trong SQL

Nếu CÙNG aggregates được NHIỀU QUERY dùng:
   → CHẾ BIẾN qua raw data mỗi lần = LÃNG PHÍ

→ Cache một số COUNTS/SUMS phổ biến nhất: MATERIALIZED AGGREGATES

DATA CUBE (OLAP Cube):
   Tạo GRID of aggregates GROUPED BY các dimensions KHÁC NHAU
```

### Data Cube — Ví dụ 2 chiều (Figure 4-10)

```
Giả sử fact table có 2 foreign key: date_key, product_sk

→ Tạo bảng 2 chiều:
   Trục X: dates (date_key)
   Trục Y: products (product_sk)
   Mỗi CELL: SUM(net_price) của tất cả facts với date+product đó

Áp dụng CÙNG aggregate dọc theo rows/columns:
   → Summary GIẢM BỚT 1 chiều:
     - Sales by product BẤT KỂ ngày nào
     - Sales by date BẤT KỂ product nào
```

### Thực tế: Nhiều Dimensions hơn

```
Fact tables thường có >2 dimensions
Ví dụ: 5 dimensions: date, product, store, promotion, customer

→ 5-DIMENSIONAL HYPERCUBE
   (Khó hình dung, nhưng NGUYÊN TẮC TƯƠNG TỰ)

Mỗi cell: sales cho COMBINATION cụ thể của date-product-store-promotion-customer
   → Có thể REPEATEDLY SUMMARIZED dọc theo TỪNG DIMENSION
```

### Ưu và Nhược điểm của Data Cube

|               | Ưu điểm                       | Nhược điểm                   |
| ------------- | ----------------------------- | ---------------------------- |
| **Data Cube** | QUERY RẤT NHANH (precomputed) | KHÔNG LINH HOẠT như raw data |

```
Ví dụ nhược điểm:
   KHÔNG thể tính proportion sales từ items >$100
   (vì price KHÔNG phải 1 dimension trong data cube)

→ Hầu hết data warehouses: giữ càng nhiều RAW DATA càng tốt
   Chỉ dùng data cube như PERFORMANCE BOOST cho một số query cụ thể
```

---

## Tóm tắt phần này

```
ANALYTICS STORAGE KHÁC OLTP vì:
   Query: scan NHIỀU rows, CHỈ CẦN vài cột
   Write: bulk import qua ETL, KHÔNG phải individual rows

CLOUD DATA WAREHOUSE:
   Traditional on-premises → Cloud-native (BigQuery, Redshift, Snowflake)
   Open source: tách thành components riêng:
      Query Engine / Storage Format / Table Format / Data Catalog

COLUMN-ORIENTED STORAGE:
   Lưu values của cùng COLUMN gần nhau (thay vì cùng ROW)
   → Load CHỈ columns cần thiết → TIẾT KIỆM I/O đáng kể

COLUMN COMPRESSION:
   Bitmap Encoding + Run-Length Encoding
   → Bitwise OR/AND cho query cực NHANH

SORT ORDER:
   Admin chọn SORT KEY phù hợp với common queries
   → First sort key: compression TỐT NHẤT (long runs)

WRITING TO COLUMNAR:
   Log-structured approach + batch writes
   → In-memory row store → batch merge vào columnar files trên disk

QUERY EXECUTION:
   Query Compilation (JIT) vs Vectorized Processing
   → Cả 2 tận dụng modern CPU:
     sequential access, tight inner loops, SIMD, operate on compressed data

MATERIALIZED VIEWS / DATA CUBES:
   Precompute aggregates → query NHANH hơn
   NHƯNG kém linh hoạt hơn raw data
   → Dùng như performance boost, KHÔNG thay thế raw data
```
