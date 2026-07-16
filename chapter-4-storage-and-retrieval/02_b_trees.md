# B-Trees

---

## 1. Triết lý thiết kế KHÁC HOÀN TOÀN với Log-Structured

```
Log-structured indexes (SSTable/LSM):
   Phá database thành SEGMENT KÍCH THƯỚC THAY ĐỔI
   (thường vài MB+), viết MỘT LẦN, sau đó IMMUTABLE

B-TREES:
   Phá database thành BLOCK/PAGE KÍCH THƯỚC CỐ ĐỊNH
   CÓ THỂ OVERWRITE 1 PAGE TẠI CHỖ (in place)
```

### Lịch sử

> Giới thiệu năm **1970**, được gọi là **"ubiquitous"** (phổ biến khắp nơi) chưa đầy 10 năm sau. B-Tree vẫn là index implementation **CHUẨN** trong hầu hết relational databases, và nhiều nonrelational databases cũng dùng.

### Điểm chung với SSTable

```
GIỐNG SSTable: giữ key-value pairs SORTED THEO KEY
   → cho phép lookup hiệu quả + range query
```

→ Đến đây là HẾT điểm giống nhau.

---

## 2. Cấu trúc B-Tree

### Page (Block)

```
Page (block) — kích thước CỐ ĐỊNH
   Truyền thống: 4 KiB
   PostgreSQL hiện nay: 8 KiB
   MySQL mặc định: 16 KiB

Mỗi page có PAGE NUMBER → cho phép page này REFERENCE page khác
   (giống POINTER, nhưng trên DISK thay vì memory)

Nếu tất cả page lưu CÙNG 1 file:
   page_number × page_size = byte offset trong file
```

### Cấu trúc Tree

```
1 page = ROOT của B-tree
   → MỌI lookup BẮT ĐẦU từ đây

Page chứa NHIỀU key + reference đến CHILD pages
Mỗi child chịu trách nhiệm cho 1 KHOẢNG (range) key LIÊN TỤC
Key GIỮA các reference = BOUNDARY của các range đó

(Cấu trúc này đôi khi gọi là B+ TREE, nhưng sách KHÔNG phân biệt
 với các B-tree variant khác)
```

### Ví dụ: Tìm key 251

```
ROOT PAGE
   │
   └─► page chứa range 200-300
            │
            └─► page chứa range 250-270
                     │
                     └─► LEAF PAGE (chứa individual keys)
                          (chứa value INLINE hoặc reference đến page chứa value)
```

### Branching Factor

> **Branching Factor:** Số lượng reference đến child page TRONG 1 page của B-tree.

```
Trong ví dụ trên: branching factor = 6

Thực tế: phụ thuộc vào KHÔNG GIAN cần để lưu page reference + range boundary
         Thường: VÀI TRĂM
```

---

## 3. Update và Insert

### Update giá trị HIỆN CÓ

```
Tìm LEAF PAGE chứa key đó
→ OVERWRITE page đó trên disk với phiên bản chứa value MỚI
```

### Insert key MỚI

```
Tìm page có RANGE bao gồm key mới → THÊM vào page đó

Nếu KHÔNG ĐỦ SPACE trong page:
   → SPLIT page đó thành 2 page NỬA ĐẦY (half-full)
   → UPDATE parent page để phản ánh subdivision MỚI
```

### Ví dụ Split: Insert key 334

```
Page cho range 333-345 ĐÃ ĐẦY

→ Split thành:
   Page A: range 333-337 (chứa key MỚI 334)
   Page B: range 337-345

→ UPDATE parent page: thêm reference đến CẢ HAI children,
                       với BOUNDARY = 337 giữa chúng

Nếu PARENT page KHÔNG ĐỦ SPACE cho reference mới
   → PARENT cũng cần split
   → Có thể LAN TRUYỀN lên tới ROOT của tree
   → Khi ROOT bị split: TẠO ROOT MỚI ở trên (tree TĂNG 1 LEVEL)

(Xóa key — có thể cần MERGE nodes — phức tạp hơn)
```

### Đảm bảo Tree luôn Balanced

```
B-tree với n key LUÔN có depth = O(log n)

Hầu hết database FIT trong B-tree CHỈ 3-4 LEVEL
   → KHÔNG cần follow nhiều page reference

Ví dụ: B-tree 4-level, page 4 KiB, branching factor 500
   → Lưu được TỚI 250 TB DATA
```

---

## 4. Làm cho B-Tree Reliable: Write-Ahead Log (WAL)

### Vấn đề: Overwrite Page là RỦI RO

```
Write operation CƠ BẢN của B-tree: OVERWRITE 1 page trên disk
   với data MỚI

GIẢ ĐỊNH: overwrite KHÔNG đổi VỊ TRÍ của page
   (mọi reference đến page đó VẪN ĐÚNG sau khi overwrite)

KHÁC HẲN log-structured indexes (LSM-tree):
   CHỈ append vào file (cuối cùng xóa file cũ),
   KHÔNG BAO GIỜ modify file TẠI CHỖ
```

### Rủi ro khi Overwrite NHIỀU Page (vd: Page Split)

```
Crash GIỮA CHỪNG (chỉ MỘT SỐ page đã được viết):
   → CORRUPTED TREE (vd: orphan page — KHÔNG phải child của parent nào)

Hardware KHÔNG THỂ atomic write CẢ 1 page:
   → PARTIALLY WRITTEN PAGE (gọi là "TORN PAGE")
```

### Giải pháp: Write-Ahead Log (WAL)

```
B-tree implementation thường có THÊM cấu trúc trên disk: WAL

WAL = file APPEND-ONLY
   MỖI thay đổi B-tree PHẢI ghi vào WAL TRƯỚC KHI áp dụng vào page

Khi database RESTART sau crash:
   → DÙNG WAL để khôi phục B-tree về TRẠNG THÁI NHẤT QUÁN

(Tương đương trong filesystem: gọi là JOURNALING)
```

### Buffer để cải thiện Performance

```
B-tree implementation thường KHÔNG ghi NGAY mỗi page modified vào disk
   → BUFFER các B-tree page trong MEMORY 1 thời gian

WAL đảm bảo data KHÔNG MẤT nếu crash
   (miễn là đã write vào WAL + flush bằng fsync system call
    → data DURABLE, database recover được sau crash)
```

---

## 5. Các Biến thể của B-Tree

### Copy-on-Write Scheme

```
Thay vì overwrite page + duy trì WAL:
   1 số database (như LMDB) dùng COPY-ON-WRITE

Page bị modify → viết vào VỊ TRÍ KHÁC
   Tạo PHIÊN BẢN MỚI của parent pages, TRỎ đến vị trí mới

→ CŨNG hữu ích cho CONCURRENCY CONTROL
  (xem "Snapshot Isolation and Repeatable Read" — Chapter 8)
```

### Tiết kiệm Space: Abbreviated Keys

```
KHÔNG lưu TOÀN BỘ key — chỉ lưu ĐỦ THÔNG TIN để làm BOUNDARY
   (đặc biệt ở INTERIOR pages của tree)

→ Pack ĐƯỢC NHIỀU key hơn vào 1 page
→ TĂNG branching factor → ÍT level hơn
```

### Sequential Layout cho Scan nhanh hơn

```
1 số B-tree implementation TRY làm cho LEAF PAGE xuất hiện
THEO THỨ TỰ TUẦN TỰ trên disk
   → GIẢM số disk seeks khi scan

NHƯNG: duy trì thứ tự này KHÓ khi tree GROWS
```

### Sibling Pointers

```
THÊM pointer: mỗi LEAF PAGE có reference đến
   sibling page BÊN TRÁI và BÊN PHẢI

→ Cho phép SCAN key theo thứ tự MÀ KHÔNG cần
  nhảy NGƯỢC lên parent page
```

---

## 6. So sánh B-Trees và LSM-Trees

### Nguyên tắc Thumb

```
LSM-trees:  TỐT HƠN cho WRITE-HEAVY applications
B-trees:    NHANH HƠN cho READS

NHƯNG: benchmark THƯỜNG nhạy với CHI TIẾT của workload
   → CẦN test với workload CỤ THỂ của bạn để so sánh ĐÚNG

KHÔNG PHẢI lựa chọn either/or NGHIÊM NGẶT:
   1 số storage engine BLEND đặc điểm của CẢ HAI approach
   (vd: nhiều B-tree + merge theo kiểu LSM)
```

### Read Performance

```
B-TREE:
   Lookup key = đọc 1 PAGE TẠI MỖI LEVEL
   Số level THƯỜNG NHỎ → READ NHANH, performance DỰ ĐOÁN ĐƯỢC

LSM STORAGE:
   Read THƯỜNG phải check NHIỀU SSTable ở các stage compaction khác nhau
   Bloom filters GIÚP GIẢM số disk I/O cần thiết

→ Cái nào NHANH HƠN: PHỤ THUỘC chi tiết storage engine + workload
```

### Range Query

```
B-TREE: ĐƠN GIẢN và NHANH (tận dụng cấu trúc SORTED của tree)

LSM STORAGE: CŨNG tận dụng được sort của SSTable
   NHƯNG: cần scan TẤT CẢ segment SONG SONG, combine kết quả
   Bloom filters KHÔNG GIÚP cho range query
      (phải tính hash của MỌI key có thể trong range — KHÔNG THỰC TẾ)

→ Range query ĐẮT HƠN point query trong LSM approach
```

### Latency Spike khi Write Throughput Cao

```
LSM storage: HIGH write throughput → memtable ĐẦY NHANH
   Nếu data KHÔNG được write ra disk ĐỦ NHANH
   (compaction process KHÔNG theo kịp incoming write)
   → LATENCY SPIKE

Nhiều storage engine (vd: RocksDB) áp dụng BACKPRESSURE:
   TẠM DỪNG TẤT CẢ read/write cho đến khi memtable
   được write ra disk
```

### Read Throughput trên SSD hiện đại

```
SSD hiện đại (đặc biệt NVMe — kết nối qua PCIe nhanh hơn SATA)
   CÓ THỂ thực hiện NHIỀU read request ĐỘC LẬP SONG SONG

CẢ LSM-trees VÀ B-trees ĐỀU có thể cung cấp read throughput cao
   NHƯNG storage engine CẦN được thiết kế CẨN THẬN
   để TẬN DỤNG parallelism này
```

---

## 7. Sequential vs Random Writes

### B-Tree: Random Writes

```
App write key SCATTERED khắp key space
   → Disk operation CŨNG scattered RANDOM
   (page cần overwrite có thể nằm BẤT KỲ ĐÂU trên disk)
```

### LSM-Tree: Sequential Writes

```
Log-structured storage engine: viết TOÀN BỘ SEGMENT FILE 1 lần
   (write memtable HOẶC compact existing segments)
   LỚN HƠN NHIỀU so với 1 page của B-tree
```

### Tại sao Sequential nhanh hơn Random?

| Loại Disk               | Lý do                                                                                                                                        |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| **HDD (spinning disk)** | Random write phải DI CHUYỂN CƠ HỌC disk head đến vị trí mới, chờ đúng phần platter — MẤT VÀI MILLISECOND (rất lâu trong computing timescale) |
| **SSD (NVMe/flash)**    | KHÔNG bị giới hạn cơ học, NHƯNG vẫn có throughput sequential CAO HƠN random                                                                  |

### Tại sao SSD vẫn chậm hơn với Random Write? (Garbage Collection)

```
Flash memory:
   ĐỌC/VIẾT theo PAGE (thường 4 KiB)
   CHỈ XÓA theo BLOCK (thường 512 KiB)

1 block có thể chứa CẢ data VALID + data KHÔNG CẦN NỮA
Trước khi erase block: controller phải DI CHUYỂN page VALID
   sang block khác → gọi là GARBAGE COLLECTION (GC)

SEQUENTIAL WRITE: viết chunk LỚN 1 lần
   → THƯỜNG 1 block 512 KiB thuộc về 1 FILE
   → Khi file bị xóa: CẢ block erase được, KHÔNG cần GC

RANDOM WRITE: block thường chứa MIX page valid/invalid
   → Garbage collector phải làm NHIỀU VIỆC HƠN trước khi erase
   → BĂNG THÔNG GC này KHÔNG dùng được cho application
   → THÊM write do GC → MÒN flash NHANH HƠN
```

> **Kết luận:** Random write LÀM MÒN SSD NHANH HƠN sequential write.

---

## 8. Write Amplification

### Định nghĩa

> **Write Amplification:** Tỷ lệ giữa **tổng số byte ghi vào disk** trong workload, chia cho **số byte cần ghi nếu chỉ dùng append-only log KHÔNG INDEX**.

### Với LSM-Trees

```
1 value được viết NHIỀU LẦN:
   1. Viết vào LOG (cho durability)
   2. Viết LẠI khi memtable được write ra disk
   3. Viết LẠI mỗi lần key-value pair là PHẦN của 1 compaction

(Có thể GIẢM overhead bằng cách lưu value RIÊNG khỏi key,
 chỉ compact SSTable chứa key + reference đến value)
```

### Với B-Trees

```
PHẢI viết MỖI piece of data ÍT NHẤT 2 LẦN:
   1. Vào WRITE-AHEAD LOG
   2. Vào TREE PAGE chính nó

THÊM NỮA: đôi khi PHẢI viết TOÀN BỘ page,
   dù CHỈ vài byte trong page đó thay đổi
   (để đảm bảo B-tree recover ĐÚNG sau crash/power failure)
```

### Ảnh hưởng của Write Amplification

```
Write-heavy application: BOTTLENECK có thể là TỐC ĐỘ
   database GHI VÀO DISK

→ Write amplification CÀNG CAO
  → SỐ WRITE/GIÂY có thể handle TRONG disk bandwidth sẵn có
    CÀNG THẤP

LSM-trees THƯỜNG có write amplification THẤP HƠN B-trees
   (KHÔNG cần viết TOÀN BỘ page, CÓ THỂ nén chunk của SSTable)
   → Lý do KHÁC khiến LSM phù hợp cho write-heavy workload

CŨNG ảnh hưởng đến WEAR của SSD:
   Write amplification THẤP HƠN → SSD mòn CHẬM HƠN
```

> **Lưu ý đo lường:** Khi đo write throughput, cần CHẠY THÍ NGHIỆM ĐỦ LÂU để effect của write amplification RÕ RÀNG (LSM-tree RỖNG chưa có compaction → toàn bộ bandwidth dành cho write mới; khi database LỚN DẦN → write phải SHARE bandwidth với compaction).

---

## 9. Disk Space Usage

### B-Trees: Fragmentation

```
B-tree có thể FRAGMENT theo thời gian
   Vd: xóa NHIỀU key → file database chứa NHIỀU page KHÔNG dùng nữa

Insert MỚI có thể dùng FREE page đó
   NHƯNG KHÔNG DỄ trả lại cho OS
   (vì NẰM GIỮA file) → VẪN chiếm disk space

→ Cần BACKGROUND PROCESS di chuyển page để TỐI ƯU
  (vd: VACUUM process trong PostgreSQL)
```

### LSM-Trees: Ít Fragmentation Hơn

```
COMPACTION process ĐỊNH KỲ rewrite data files
   → SSTable KHÔNG có page với SPACE KHÔNG DÙNG

THÊM NỮA: block key-value pairs trong SSTable NÉN TỐT HƠN
   → file trên disk THƯỜNG NHỎ HƠN so với B-tree

Key/value bị OVERWRITE VẪN chiếm space cho đến khi
   COMPACTION xóa chúng — nhưng overhead THẤP với LEVELED compaction
   (Size-tiered compaction dùng NHIỀU disk space hơn,
    đặc biệt TẠM THỜI trong lúc compaction)
```

### Vấn đề Xóa Data Vĩnh Viễn (Compliance)

```
Có NHIỀU COPY của data trên disk = VẤN ĐỀ
   khi cần XÓA data và CHẮC CHẮN nó ĐÃ BỊ XÓA THẬT
   (vd: tuân thủ quy định bảo vệ data)

Hầu hết LSM storage: record bị xóa CÓ THỂ VẪN TỒN TẠI
   ở các LEVEL CAO HƠN cho đến khi TOMBSTONE
   được PROPAGATE qua TẤT CẢ compaction level
   → CÓ THỂ MẤT THỜI GIAN DÀI

(Specialist storage engine design CÓ THỂ propagate deletion NHANH HƠN)
```

### Ưu điểm của Immutable SSTable: Snapshot

```
Tính chất IMMUTABLE của SSTable RẤT HỮU ÍCH
   khi muốn TAKE SNAPSHOT của database tại 1 thời điểm
   (cho backup, hoặc tạo copy database để test)

CÁCH LÀM:
   Write out memtable + ghi nhớ segment file nào TỒN TẠI lúc đó
   MIỄN LÀ KHÔNG XÓA file thuộc snapshot
   → KHÔNG CẦN COPY thực sự các file đó

Trong B-tree (page bị OVERWRITE):
   Take snapshot HIỆU QUẢ KHÓ HƠN NHIỀU
```

---

## Tóm tắt phần này

```
B-TREE:
  - Page kích thước CỐ ĐỊNH (4-16 KiB), OVERWRITE TẠI CHỖ
  - Tree depth = O(log n), THƯỜNG 3-4 level
  - Reliability: cần WRITE-AHEAD LOG (WAL) cho crash recovery
  - Variants: copy-on-write, abbreviated keys, sibling pointers

SO SÁNH B-TREE vs LSM-TREE:
┌─────────────────┬─────────────────┬──────────────────┐
│                  │ B-TREE          │ LSM-TREE          │
├─────────────────┼─────────────────┼──────────────────┤
│ Read             │ NHANH, dự đoán  │ Cần check NHIỀU   │
│                  │ được            │ SSTable (Bloom    │
│                  │                 │ filter giúp)       │
│ Write pattern    │ RANDOM          │ SEQUENTIAL        │
│ Write throughput │ THẤP HƠN        │ CAO HƠN           │
│ Range query      │ ĐƠN GIẢN, NHANH │ Cần scan nhiều    │
│                  │                 │ segment           │
│ Write            │ CAO (≥2x: WAL   │ THẤP HƠN (thường) │
│ Amplification    │ + page)         │                   │
│ Fragmentation    │ CÓ (cần vacuum) │ ÍT (compaction tự │
│                  │                 │ dọn)              │
│ Snapshot         │ KHÓ             │ DỄ (immutable)    │
└─────────────────┴─────────────────┴──────────────────┘

QUY TẮC THUMB: LSM = WRITE-HEAVY, B-Tree = READ-HEAVY
              (NHƯNG luôn TEST với workload thực tế của bạn)
```
