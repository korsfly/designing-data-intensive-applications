# Log-Structured Storage

---

## 1. Database Đơn Giản Nhất Thế Giới

```bash
#!/bin/bash
db_set () {
  echo "$1,$2" >> database
}
db_get () {
  grep "^$1," database | sed -e "s/^$1,//" | tail -n 1
}
```

### Cách hoạt động

```
db_set key value → APPEND "key,value" vào cuối file "database"
db_get key       → Tìm dòng CUỐI CÙNG khớp key, trả về value
```

```bash
$ db_set 12 '{"name":"London","attractions":["Big Ben","London Eye"]}'
$ db_set 42 '{"name":"San Francisco","attractions":["Golden Gate Bridge"]}'
$ db_get 42
{"name":"San Francisco","attractions":["Golden Gate Bridge"]}
```

> **Lưu ý quan trọng:** Update KHÔNG ghi đè giá trị cũ. Mỗi `db_set` chỉ APPEND. Để có giá trị MỚI NHẤT, cần dùng `tail -n 1` để lấy dòng CUỐI khớp key đó.

### Định nghĩa Log

> Trong sách này, **"log"** dùng theo nghĩa TỔNG QUÁT: một **append-only sequence of records trên disk** — KHÔNG nhất thiết là human-readable (có thể là binary, chỉ dùng nội bộ bởi database).

(Khác với "application logs" — text mô tả việc gì đang xảy ra trong app.)

---

## 2. Performance: Tốt cho Write, Tệ cho Read

```
db_set: NHANH vì append vào file LUÔN hiệu quả

db_get: CHẬM nếu có NHIỀU record
   → Mỗi lần lookup phải SCAN TOÀN BỘ file từ đầu đến cuối
   → Độ phức tạp: O(n) — n GẤP ĐÔI thì lookup LÂU GẤP ĐÔI
```

→ Cần **INDEX** để tìm value theo key HIỆU QUẢ.

### Định nghĩa Index

> Index = cấu trúc dữ liệu PHỤ, được **derive** từ primary data. Tổ chức data theo CÁCH CỤ THỂ (vd: sort theo key) để TÌM data NHANH HƠN. Nếu cần search nhiều cách khác nhau → có thể cần NHIỀU index trên các phần khác nhau của data.

---

## 3. Hash Index — Bước cải tiến đầu tiên

### Ý tưởng

```
Giữ HASH MAP trong memory:
   key → byte offset (vị trí) trong log file
        nơi value MỚI NHẤT của key đó được tìm thấy

Mỗi lần append key-value mới → UPDATE hash map (offset mới)
Mỗi lần lookup → dùng hash map tìm offset → SEEK đến đó → đọc value
```

> Nếu phần data đó đã có trong **filesystem cache**, read **KHÔNG CẦN** disk I/O nào cả.

### Vấn đề của Hash Index

| Vấn đề                               | Chi tiết                                                                                         |
| ------------------------------------ | ------------------------------------------------------------------------------------------------ |
| **Không giải phóng disk space**      | Old entries bị overwrite KHÔNG được dọn → có thể HẾT disk                                        |
| **Hash map không persist**           | Phải REBUILD khi restart database (scan toàn bộ log) → restart CHẬM nếu data lớn                 |
| **Hash table phải fit trong memory** | On-disk hash map KHÓ làm performant (random access I/O nhiều, khó grow, hash collision phức tạp) |
| **Range query KHÔNG hiệu quả**       | Không thể scan keys từ 10000-19999 dễ dàng — phải lookup TỪNG key riêng lẻ                       |

---

## 4. SSTable (Sorted String Table) — Giải pháp cho Range Query

### Đặc điểm

```
SSTable: lưu key-value pairs, NHƯNG:
   1. SORTED THEO KEY
   2. Mỗi key CHỈ XUẤT HIỆN MỘT LẦN trong file
```

### Sparse Index

```
KHÔNG cần giữ TẤT CẢ key trong memory

Group key-value pairs thành BLOCKS (vài KB)
Lưu KEY ĐẦU TIÊN của mỗi block vào index
   → Index này CHỈ lưu MỘT SỐ key → gọi là SPARSE index

Sparse index lưu ở phần RIÊNG của SSTable
   (dùng immutable B-tree, trie, hoặc cấu trúc tương tự)
```

### Ví dụ tìm kiếm

```
Block 1: bắt đầu từ "handbag"
Block 2: bắt đầu từ "handsome"

Tìm "handiwork" (không có trong sparse index):
   Vì data ĐÃ SORTED → "handiwork" PHẢI nằm giữa "handbag" và "handsome"
   → Seek đến offset của "handbag" → scan file TỪ ĐÓ
     cho đến khi tìm thấy "handiwork" (hoặc không tìm thấy)

Block vài KB → scan RẤT NHANH
```

> **Compression:** Mỗi block có thể được COMPRESS thêm — vừa tiết kiệm disk space, vừa giảm I/O bandwidth (đánh đổi 1 ít CPU time).

---

## 5. Xây dựng và Merge SSTables

### Vấn đề: Viết SSTable khó hơn Append-only Log

```
KHÔNG THỂ chỉ append vào cuối (file sẽ KHÔNG còn sorted)
Nếu phải REWRITE TOÀN BỘ SSTable mỗi lần insert giữa file
   → Write QUÁ ĐẮT
```

### Giải pháp: Log-Structured Approach (kết hợp Log + Sorted File)

```
1. Write mới → thêm vào IN-MEMORY ordered map data structure
   (red-black tree, skip list, hoặc trie)
   → Gọi là MEMTABLE
   (Cho phép insert key BẤT KỲ THỨ TỰ, lookup hiệu quả,
    đọc lại theo thứ tự sorted)

2. Khi memtable LỚN HƠN ngưỡng (thường vài MB)
   → Write OUT thành SSTable file (sorted) trên disk
   → Gọi là SEGMENT mới nhất, lưu RIÊNG cùng các segment cũ hơn
   → Mỗi segment có INDEX RIÊNG
   → Database TIẾP TỤC ghi vào memtable instance MỚI
     trong khi segment cũ đang được write ra disk

3. Đọc value của 1 key:
   → Tìm trong MEMTABLE trước
   → Nếu không có, tìm SEGMENT mới nhất trên disk
   → Tiếp tục tìm các segment CŨ HƠN cho đến khi tìm thấy
     hoặc đến segment CŨ NHẤT
   → Nếu không tìm thấy ở đâu cả → key KHÔNG TỒN TẠI

4. Định kỳ chạy MERGING & COMPACTION ở background
   → Gộp các segment file, BỎ giá trị bị overwrite/xóa
```

### Merging giống Mergesort

```
Đọc các input file SONG SONG, mỗi lần nhìn key ĐẦU TIÊN của mỗi file
Copy KEY NHỎ NHẤT (theo sort order) sang output file
Lặp lại

Nếu CÙNG 1 key xuất hiện ở NHIỀU file → giữ giá trị MỚI NHẤT

→ Tạo ra MERGED SEGMENT FILE mới (cũng sorted, mỗi key 1 value)
→ Dùng MEMORY TỐI THIỂU (chỉ cần iterate qua SSTables 1 key tại 1 lúc)
```

### Bảo vệ Memtable khỏi Crash: Write-Ahead Log riêng

```
Storage engine giữ 1 LOG RIÊNG trên disk
   Mỗi write được APPEND NGAY LẬP TỨC vào log này

Log này KHÔNG sorted theo key
   (chỉ dùng để RESTORE memtable sau crash)

Mỗi khi memtable được write ra SSTable
   → Phần TƯƠNG ỨNG của log có thể bị XÓA
```

### Xóa Key: Tombstone

```
Để xóa 1 key → append TOMBSTONE (deletion record đặc biệt)

Khi merge log segments:
   Tombstone báo cho process merge BỎ mọi giá trị cũ của key đó

Khi tombstone đã merge vào segment CŨ NHẤT → có thể DROP nó
```

---

## 6. LSM-Tree — Tên gọi chung

```
Thuật toán này = nền tảng cho: RocksDB, Cassandra, ScyllaDB, HBase
   (đều lấy cảm hứng từ Google's BIGTABLE paper
    — giới thiệu thuật ngữ "SSTable" và "memtable")

Tên gốc: Log-Structured Merge-Tree (LSM-tree), công bố 1996
   (dựa trên nghiên cứu TRƯỚC ĐÓ về log-structured filesystems)

→ Storage engines dựa trên nguyên tắc merge+compact sorted files
  thường gọi là "LSM STORAGE ENGINES"
```

### Đặc điểm Immutability

```
Segment file: viết MỘT LẦN (write out memtable HOẶC merge segment cũ)
   SAU ĐÓ TRỞ THÀNH IMMUTABLE

Merging/compaction → có thể chạy ở BACKGROUND THREAD
   Trong lúc merge: VẪN serve read bằng input segments
   Khi merge XONG: chuyển read request sang segment MỚI,
                   rồi XÓA input segment files
```

### Lưu trữ trên Object Storage

```
Segment files KHÔNG NHẤT THIẾT phải lưu local disk
   → PHÙ HỢP để viết lên OBJECT STORAGE

Ví dụ: SlateDB, Delta Lake dùng approach này
```

### Crash Recovery đơn giản hơn

```
Immutable segment files → ĐƠN GIẢN HÓA crash recovery
   Crash khi write memtable/merge → XÓA SSTable chưa hoàn thành,
                                      bắt đầu LẠI từ đầu

Log persist write vào memtable có thể chứa record KHÔNG HOÀN CHỈNH
   (crash giữa chừng, disk đầy)
   → Phát hiện qua CHECKSUM, BỎ entry corrupted/incomplete
   → (Chi tiết về durability/crash recovery: Chapter 8)
```

---

## 7. Bloom Filters — Tăng tốc Read

### Vấn đề

```
Với LSM storage: đọc key CŨ (update lâu rồi)
   hoặc đọc key KHÔNG TỒN TẠI
   → CHẬM vì cần check NHIỀU SEGMENT FILES
```

### Giải pháp: Bloom Filter

> **Bloom filter:** Cách kiểm tra NHANH NHƯNG XẤP XỈ (approximate) xem 1 key có xuất hiện trong 1 SSTable hay không.

### Cách hoạt động

```
Bloom filter = bitmap N bit

Với mỗi key trong SSTable:
   Tính HASH FUNCTION → tạo SET các số → dùng làm INDEX vào bitmap
   Set các bit TƯƠNG ỨNG = 1

Ví dụ: key "handbag" → hash thành (2, 9, 4)
   → Set bit thứ 2, 9, 4 = 1
```

### Kiểm tra key

```
Tính hash của key cần check → check các bit tại index TƯƠNG ỨNG

Ví dụ: key "handheld" → hash thành (6, 11, 2)
   Bit 2 = 1, Bit 6 = 0, Bit 11 = 0
   → MỘT bit = 0 → key CHẮC CHẮN KHÔNG có trong SSTable
```

### Kết luận từ kiểm tra

| Kết quả check        | Ý nghĩa                                                                       |
| -------------------- | ----------------------------------------------------------------------------- |
| Có ÍT NHẤT 1 bit = 0 | Key CHẮC CHẮN KHÔNG có trong SSTable                                          |
| TẤT CẢ bit = 1       | Key CÓ THỂ có (nhưng có thể là FALSE POSITIVE — trùng ngẫu nhiên do key khác) |

> **False Positive:** Trường hợp filter báo "có" nhưng thực tế KHÔNG CÓ key đó.

### Tham số Bloom Filter

```
Quy tắc thumb: 10 BIT bloom filter / KEY trong SSTable
   → False-positive probability ~1%

Cứ THÊM 5 bit/key → false-positive probability GIẢM 10 LẦN
```

### Tại sao False Positive KHÔNG VẤN ĐỀ trong LSM context?

```
Filter nói KHÔNG có → SKIP SSTable đó (CHẮC CHẮN đúng)

Filter nói CÓ → check sparse index + decode block
   Nếu là FALSE POSITIVE → chỉ làm THỪA 1 chút công việc
                            (KHÔNG hại gì), tiếp tục tìm segment cũ hơn
```

---

## 8. Compaction Strategies

### Size-Tiered Compaction

```
SSTable MỚI và NHỎ → merge dần thành SSTable CŨ và LỚN HƠN

Ví dụ: bốn SSTable 256 MB → compact thành MỘT SSTable ~898 MB
   (không phải 1024 MB vì có deletion, overwrite, TTL expiration)

Ưu điểm: Handle WRITE THROUGHPUT RẤT CAO
   (đa số data CHỈ rewrite vài lần qua merge tuần tự lớn)

Nhược điểm: SSTable chứa data CŨ có thể RẤT LỚN,
   merge cần NHIỀU disk space tạm thời
```

### Leveled Compaction

```
GIỮ SSTable size CỐ ĐỊNH, nhóm thành các "LEVEL" tăng dần
   (L0, L1, L2,...)

L0: data MỚI NHẤT
Level > L0: SSTable PARTITIONED THEO KEY RANGE
   (vd: L1 có 2 SSTable: a-m và n-z)

Mỗi level có SIZE LIMIT RIÊNG, level SAU lớn hơn level TRƯỚC

Khi 1 level VƯỢT max size:
   → 1+ SSTable từ level i MERGE vào level i+1, XÓA khỏi level i

Ưu điểm: COMPACTION incremental hơn, ÍT disk space hơn size-tiered
         READ HIỆU QUẢ HƠN (đọc ÍT SSTable hơn để check key)
```

### So sánh và lựa chọn

| Workload                                   | Compaction strategy phù hợp |
| ------------------------------------------ | --------------------------- |
| Mostly WRITE, ÍT READ                      | Size-tiered                 |
| Workload DOMINATED bởi READ                | Leveled                     |
| Write FREQUENT số ít key + RARE số lớn key | Leveled                     |

> Hầu hết LSM-tree implementation cung cấp ĐA DẠNG compaction strategy cho các workload khác nhau.

---

## Tóm tắt phần này

```
Tiến hóa từ Database đơn giản → LSM-Tree:

  Append-only Log    → Write NHANH, Read O(n) CHẬM
        │
        ▼
  + Hash Index        → Read NHANH (memory), NHƯNG: không persist,
                         không range query, phải fit memory
        │
        ▼
  + SSTable (sorted)  → Sparse index, range query OK,
                         NHƯNG: write giữa file khó
        │
        ▼
  + Memtable + Merge  → LSM-TREE: combine memtable (in-memory)
                         + SSTable (immutable, sorted) + background merge
        │
        ▼
  + Bloom Filter       → Tăng tốc check "key có tồn tại không"
        │
        ▼
  + Compaction Strategy → Size-tiered (write-heavy) vs
                          Leveled (read-heavy)

Hệ thống dùng LSM: RocksDB, Cassandra, ScyllaDB, HBase, Lucene
```
