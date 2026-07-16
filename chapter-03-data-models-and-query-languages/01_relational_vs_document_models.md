# Relational vs Document Models

---

## 1. Lịch sử ngắn gọn

| Thời kỳ                          | Mô hình                                                                               |
| -------------------------------- | ------------------------------------------------------------------------------------- |
| 1970s–1980s                      | Network model, Hierarchical model (đối thủ ban đầu)                                   |
| Từ 1970, thống trị từ giữa 1980s | **Relational Model** (Edgar Codd, 1970)                                               |
| Cuối 1980s–đầu 1990s             | Object databases (đến rồi đi)                                                         |
| Đầu 2000s                        | XML databases (chỉ niche adoption)                                                    |
| 2010s                            | **NoSQL** — bộ ý tưởng mới (data model, schema flexibility, scalability, open source) |
| Cùng thời                        | **NewSQL** — kết hợp scalability của NoSQL + transactional guarantee của relational   |

> **Di sản lâu dài của NoSQL:** Sự phổ biến của **Document Model** (thường dùng JSON), ban đầu được phổ biến bởi MongoDB, Couchbase — hiện nay hầu hết relational database cũng đã support JSON.

---

## 2. Object-Relational Mismatch

### Vấn đề cốt lõi

> Khi data lưu trong relational tables, cần một **translation layer** vụng về (awkward) giữa objects trong application code và model bảng/dòng/cột của database. Đây gọi là **impedance mismatch** (thuật ngữ mượn từ điện tử học).

### Object-Relational Mapping (ORM)

ORM frameworks (ActiveRecord, Hibernate) giảm boilerplate code cho translation layer, nhưng vẫn bị phê bình:

| Vấn đề của ORM                  | Chi tiết                                                                                  |
| ------------------------------- | ----------------------------------------------------------------------------------------- |
| Không ẩn hoàn toàn sự khác biệt | Developer vẫn phải nghĩ về CẢ HAI representation                                          |
| Chỉ dùng cho OLTP               | Data engineers vẫn cần làm việc với relational representation gốc cho analytics           |
| Hạn chế với hệ thống đa dạng    | ORM thường chỉ hỗ trợ relational OLTP, không hỗ trợ search engine, graph DB, NoSQL        |
| Schema generation kém hiệu quả  | Schema tự sinh có thể vụng về và không hiệu quả                                           |
| **N+1 Query Problem**           | Lấy N comments → thực hiện N+1 query (1 cho comments + N để lookup author) thay vì 1 join |

**Ưu điểm của ORM:**

- Giảm boilerplate cho translation (dữ liệu phù hợp relational model)
- Hỗ trợ caching kết quả query
- Hỗ trợ schema migrations

```
N+1 Query Problem minh họa:

Không có ORM (SQL thủ công):
   1 query duy nhất với JOIN → lấy N comments + author name

Với ORM (mặc định):
   1 query lấy N comments
   N query riêng để lookup author của TỪNG comment
   = N+1 query (CHẬM hơn join)
```

---

## 3. Document Model cho One-to-Many Relationships

### Ví dụ: LinkedIn Profile (Résumé)

**Cách Relational (Figure 3-1):** Tách thành nhiều bảng `users`, `positions`, `education`, `contact_info`, liên kết bằng foreign key.

**Cách Document (JSON):**

```json
{
  "user_id": 251,
  "first_name": "Barack",
  "last_name": "Obama",
  "positions": [
    { "job_title": "President", "organization": "United States of America" },
    { "job_title": "US Senator (D-IL)", "organization": "United States Senate" }
  ],
  "education": [
    { "school_name": "Harvard University", "start": 1988, "end": 1991 }
  ],
  "contact_info": { "website": "...", "x": "..." }
}
```

### Ưu điểm của Document Model ở đây

| Ưu điểm                     | Chi tiết                                                             |
| --------------------------- | -------------------------------------------------------------------- |
| **Giảm impedance mismatch** | JSON gần với object model trong code hơn                             |
| **Data locality tốt hơn**   | Tất cả info liên quan nằm CÙNG MỘT NƠI → query nhanh và đơn giản hơn |
| **Tree structure rõ ràng**  | One-to-many relationships thể hiện trực tiếp qua cấu trúc tree       |

> **Lưu ý quan trọng:** One-to-many relationship đôi khi gọi là **one-to-few** (résumé thường có ÍT position). Nếu có **số lượng items LỚN** (ví dụ: comments trên bài post của celebrity — có thể hàng nghìn), embedding tất cả vào 1 document có thể **quá nặng** → relational approach phù hợp hơn.

---

## 4. Normalization, Denormalization, và Joins

### Tại sao dùng ID thay vì String trực tiếp?

Ví dụ: `region_id` thay vì lưu trực tiếp "Washington, DC, United States"

**Lợi ích của Standardized List (Normalization):**

- Style và spelling thống nhất
- Tránh ambiguity (Washington DC vs Washington State?)
- Dễ update (chỉ cần sửa 1 nơi)
- Hỗ trợ localization (dịch theo ngôn ngữ viewer)
- Search tốt hơn (biết Washington DC thuộc East Coast)

### Normalized vs Denormalized

|              | Normalized                                             | Denormalized                                    |
| ------------ | ------------------------------------------------------ | ----------------------------------------------- |
| Cách lưu     | Chỉ lưu ID, thông tin "có nghĩa với người" lưu MỘT NƠI | Lưu trực tiếp text/value, LẶP LẠI ở nhiều nơi   |
| Viết (Write) | NHANH (chỉ 1 copy cần update)                          | CHẬM hơn (nhiều copy cần update)                |
| Đọc (Read)   | CHẬM hơn (cần JOIN)                                    | NHANH (không cần join)                          |
| Rủi ro       | —                                                      | Inconsistency nếu một số copy không được update |

```sql
-- Ví dụ JOIN để resolve ID thành tên dễ đọc
SELECT users.*, regions.region_name
FROM users
JOIN regions ON users.region_id = regions.id
WHERE users.id = 251;
```

### Document Databases và Joins

```
Document DB thường ASSOCIATED với denormalization vì:
   1. JSON model dễ thêm field denormalized
   2. Hỗ trợ JOIN yếu → normalization bất tiện

Một số document DB KHÔNG hỗ trợ join → phải tự làm trong app code
MongoDB: hỗ trợ join qua $lookup operator trong aggregation pipeline
```

### Nguyên tắc chung khi chọn Normalization/Denormalization

```
Normalized  → Thường nhanh hơn khi VIẾT, chậm hơn khi ĐỌC (cần join)
Denormalized → Thường nhanh hơn khi ĐỌC, đắt hơn khi VIẾT (nhiều copy update)
```

> Denormalization có thể coi là một dạng **Derived Data** (xem Chapter 1) — cần process để update các copy redundant.

**Lưu ý về Consistency:** Database có **atomic transactions** giúp dễ giữ consistency hơn, nhưng không phải mọi DB đều hỗ trợ atomicity across multiple documents. Có thể đảm bảo consistency qua **stream processing** (Chapter 12).

### Khi nào dùng Normalized vs Denormalized?

| Loại hệ thống            | Khuyến nghị                                                                                     |
| ------------------------ | ----------------------------------------------------------------------------------------------- |
| **OLTP**                 | Normalization thường TỐT HƠN (cả đọc và viết cần nhanh)                                         |
| **Analytics**            | Denormalization thường TỐT HƠN (update theo batch, đọc là ưu tiên chính)                        |
| **Scale nhỏ-trung bình** | Normalized thường TỐT NHẤT (không lo consistency giữa nhiều copy, cost của join chấp nhận được) |
| **Scale RẤT lớn**        | Cost của join có thể trở thành VẤN ĐỀ                                                           |

---

## 5. Case Study: Denormalization trong Social Network (X/Twitter)

> Liên kết với Case Study Chapter 2 (Home Timelines).

### Thực tế tại X (Twitter)

```
Materialized timeline KHÔNG lưu TOÀN BỘ text của post
Chỉ lưu: post ID, sender ID, một ít info phụ (repost/reply)

→ Tương đương kết quả PRECOMPUTE của query:
SELECT posts.id, posts.sender_id FROM posts
JOIN follows ON posts.sender_id = follows.followee_id
WHERE follows.follower_id = current_user
ORDER BY posts.timestamp DESC LIMIT 1000
```

### Hydrating IDs

```
Khi đọc timeline, service VẪN CẦN làm 2 join:
   1. Lookup post ID → lấy content thực, số like/reply
   2. Lookup sender ID → lấy username, profile picture

→ Gọi là "HYDRATING the IDs" — join thực hiện ở APPLICATION CODE
```

**Tại sao KHÔNG denormalize content/like-count vào timeline?**

```
Like count, profile picture THAY ĐỔI RẤT NHANH (nhiều lần/giây với post hot)
→ Denormalize sẽ làm:
   1. Storage cost TĂNG đáng kể
   2. Phải update timeline LIÊN TỤC để giữ data mới nhất
   → KHÔNG HỢP LÝ
```

> **Bài học quan trọng:** Việc cần JOIN khi đọc data **KHÔNG PHẢI** trở ngại cho hệ thống scalable, performance cao như nhiều người vẫn nghĩ. Hydrating IDs **dễ scale** vì nó **parallelize tốt**, và cost KHÔNG phụ thuộc vào số lượng follower/following.

→ **Kết luận:** Quyết định normalize/denormalize **không hiển nhiên**. Cách scalable nhất có thể là **MIX** — denormalize một số thứ, giữ normalized phần khác. Phải xem xét: tần suất thay đổi của data + cost đọc/viết (có thể bị chi phối bởi outlier như celebrity).

---

## 6. Many-to-One và Many-to-Many Relationships

### Phân loại quan hệ

| Loại quan hệ                 | Ví dụ                                                                          |
| ---------------------------- | ------------------------------------------------------------------------------ |
| **One-to-many (one-to-few)** | 1 résumé có nhiều positions                                                    |
| **Many-to-one**              | Nhiều người sống ở CÙNG 1 region                                               |
| **Many-to-many**             | 1 người làm ở NHIỀU công ty; 1 công ty có NHIỀU nhân viên (quá khứ + hiện tại) |

### Many-to-Many trong Relational Model

```
Dùng ASSOCIATIVE TABLE (join table):
   Mỗi position liên kết 1 user ID với 1 organization ID
```

### Many-to-Many trong Document Model

```json
{
  "user_id": 251,
  "positions": [
    { "start": 2009, "end": 2017, "job_title": "President", "org_id": 513 }
  ]
}
```

→ Link tới organizations/schools nên là **REFERENCE** đến document khác, không nhúng trực tiếp.

### Vấn đề: Query "cả hai hướng"

```
Many-to-many thường cần query 2 chiều:
   - Tìm TẤT CẢ organizations mà 1 người đã làm việc
   - Tìm TẤT CẢ người đã làm việc ở 1 organization

Giải pháp:
   DENORMALIZED: Lưu ID cả 2 phía (résumé chứa org IDs, org chứa résumé IDs)
                 → Rủi ro INCONSISTENCY
   NORMALIZED:   Lưu relationship CHỈ MỘT NƠI + dùng SECONDARY INDEXES
                 (Chapter 4) để query hiệu quả cả 2 chiều
```

---

## 7. Stars và Snowflakes: Schema cho Analytics

### Star Schema

```
                    dim_date
                       │
   dim_customer ── fact_sales ── dim_product
                       │
                   dim_store
```

> **Fact table** (vd: `fact_sales`) nằm ở TRUNG TÂM. Mỗi dòng đại diện cho 1 **EVENT** xảy ra ở 1 thời điểm cụ thể (ví dụ: 1 lần mua hàng).

**Đặc điểm:**

- Fact table có thể RẤT LỚN (petabytes data) vì capture từng event riêng lẻ
- Một số columns là **attributes** (giá bán, giá mua) — dùng tính profit margin
- Một số columns là **foreign keys** đến **dimension tables**

**Dimension tables** trả lời: ai, cái gì, ở đâu, khi nào, làm sao, tại sao của event.

```
Tên "STAR SCHEMA" xuất phát từ hình dạng:
   Fact table ở giữa, dimension tables xung quanh
   → giống các tia của một ngôi sao
```

### Snowflake Schema

```
Biến thể của Star Schema: dimension tables được CHIA NHỎ HƠN NỮA
   thành sub-dimensions

Ví dụ: dim_product → tham chiếu riêng dim_brand và dim_category
       (thay vì lưu brand/category là string trực tiếp trong dim_product)

→ Snowflake schema NORMALIZED HƠN star schema
→ NHƯNG star schema thường được ưa chuộng hơn vì ĐƠN GIẢN cho analyst
```

### One Big Table (OBT)

```
Đẩy denormalization ĐI XA HƠN: BỎ HẲN dimension tables
   → Gộp hết thông tin dimension vào fact table luôn
   → (Precompute JOIN giữa fact và dimension table)

Trade-off: TỐN nhiều storage hơn, NHƯNG có thể query NHANH HƠN
```

> **Tại sao OK để denormalize mạnh trong Analytics?** Data trong warehouse thường đại diện cho **log lịch sử KHÔNG THAY ĐỔI** (trừ sửa lỗi hiếm). Vấn đề **consistency** và **write overhead** của denormalization trong OLTP **KHÔNG ÁP DỤNG MẠNH** ở đây.

---

## 8. Khi nào dùng Model nào?

### Lập luận chính cho mỗi model

| Document Model                  | Relational Model                         |
| ------------------------------- | ---------------------------------------- |
| Schema flexibility              | Hỗ trợ JOIN tốt hơn                      |
| Performance tốt hơn (locality)  | Hỗ trợ many-to-one, many-to-many tốt hơn |
| Gần với object model trong code | —                                        |

### Quy tắc quyết định

```
Nếu data CÓ CẤU TRÚC document-like (tree of one-to-many,
   thường load CẢ tree một lúc)
   → Document Model phù hợp

"SHREDDING" (chia document thành nhiều bảng) có thể
   tạo ra schema vụng về, code phức tạp không cần thiết
```

**Hạn chế của Document Model:**

- Không thể tham chiếu trực tiếp item lồng nhau (vd: "item thứ 2 trong list positions của user 251")
- Relational tốt hơn nếu cần reference nested items trực tiếp

**Document Model tốt cho Reorderable Lists:**

```
Ví dụ: To-do list, issue tracker với drag-and-drop reorder
   → Document: dùng JSON array, order = thứ tự trong array (dễ!)
   → Relational: KHÔNG có cách chuẩn, phải dùng trick
        (sort theo integer column, linked list of IDs, fractional indexing)
```

### Schema-on-Read vs Schema-on-Write

|                             | **Document DB** (Schema-on-Read)         | **Relational DB** (Schema-on-Write) |
| --------------------------- | ---------------------------------------- | ----------------------------------- |
| Khi nào schema được áp dụng | Lúc ĐỌC (implicit, không enforce bởi DB) | Lúc VIẾT (explicit, DB enforce)     |
| Tương tự với                | Dynamic (runtime) type checking          | Static (compile-time) type checking |
| Thêm field mới              | Code app handle field cũ/mới khi đọc     | `ALTER TABLE` + migration           |

> **Lưu ý:** Gọi document DB là "schemaless" là **GÂY HIỂU LẦM** — code đọc data LUÔN giả định một structure nào đó, chỉ là **implicit schema** không được DB enforce.

**Ví dụ minh họa thay đổi format (split full name → first/last name):**

```javascript
// Document DB approach - handle cũ/mới trong code
if (user && user.name && !user.first_name) {
  user.first_name = user.name.split(" ")[0];
}
```

```sql
-- Relational DB approach - migration
ALTER TABLE users ADD COLUMN first_name text DEFAULT NULL;
UPDATE users SET first_name = split_part(name, ' ', 1); -- PostgreSQL
```

> **Lưu ý kỹ thuật:** `ALTER TABLE ADD COLUMN` với default value thường NHANH dù table lớn. Nhưng `UPDATE` toàn bộ rows thường CHẬM (rewrite mọi row). Có tools hỗ trợ migration KHÔNG downtime, nhưng vẫn **operationally challenging** với database lớn.

### Khi nào Schema-on-Read tốt hơn?

- Có NHIỀU loại object khác nhau, không tiện cho mỗi loại 1 table riêng
- Structure data được quyết định bởi **external system** ngoài kiểm soát của bạn, có thể thay đổi bất kỳ lúc nào

---

## 9. Data Locality cho Reads và Writes

```
Document lưu dạng 1 STRING LIÊN TỤC (JSON/XML/binary như BSON)

Nếu app thường cần TOÀN BỘ document (vd: render trang web)
   → Locality này cho LỢI THẾ PERFORMANCE

Nếu data trải trên NHIỀU table (như Figure 3-1)
   → Cần NHIỀU index lookup → nhiều disk seeks → CHẬM hơn
```

**Hạn chế của Locality:**

- Chỉ có lợi nếu cần **PHẦN LỚN** document cùng lúc
- DB thường phải load TOÀN BỘ document — lãng phí nếu chỉ cần MỘT PHẦN NHỎ
- Update document → thường phải REWRITE TOÀN BỘ document

→ **Khuyến nghị:** Giữ document KHÁ NHỎ, tránh update nhỏ thường xuyên.

**Locality KHÔNG chỉ có ở Document Model:**

- Google Spanner: cho phép "interleave" (nest) rows của 1 table TRONG parent table → có locality dù là relational
- Oracle: multi-table index cluster tables
- Bigtable/HBase/Accumulo: column families cho mục đích tương tự

---

## 10. Query Languages cho Documents

### Đa dạng hơn SQL

```
Relational DB → chủ yếu SQL
Document DB → ĐA DẠNG HƠN:
   - Một số chỉ key-value access theo primary key
   - Một số có secondary indexes
   - Một số có rich query language
```

| Loại Document DB | Query Language                                           |
| ---------------- | -------------------------------------------------------- |
| XML databases    | XQuery, XPath                                            |
| JSON (general)   | JSON Pointer, JSONPath                                   |
| MongoDB          | Aggregation Pipeline (`$lookup`, `$match`, `$group`,...) |

### Ví dụ Aggregation (cùng query, 2 cách viết)

**PostgreSQL (SQL):**

```sql
SELECT date_trunc('month', observation_timestamp) AS observation_month,
       sum(num_animals) AS total_animals
FROM observations
WHERE family = 'Sharks'
GROUP BY observation_month;
```

**MongoDB (Aggregation Pipeline):**

```javascript
db.observations.aggregate([
  { $match: { family: "Sharks" } },
  {
    $group: {
      _id: {
        year: { $year: "$observationTimestamp" },
        month: { $month: "$observationTimestamp" },
      },
      totalAnimals: { $sum: "$numAnimals" },
    },
  },
]);
```

> Aggregation pipeline ngang ngửa **subset của SQL** về khả năng biểu đạt, chỉ khác là dùng cú pháp JSON-based thay vì SQL English-sentence-style. Khác biệt chủ yếu là **sở thích** (matter of taste).

---

## 11. Sự Hội tụ (Convergence) giữa Document và Relational Databases

```
Ban đầu: 2 hướng đi RẤT KHÁC NHAU
Theo thời gian: NGÀY CÀNG GIỐNG NHAU

Relational DB → thêm: JSON type support, query operator, index trên JSON
Document DB (MongoDB, Couchbase, RethinkDB) → thêm: JOIN, secondary index,
                                                       declarative query language
```

> **Tin tốt cho developer:** Relational model và document model hoạt động TỐT NHẤT khi có thể KẾT HỢP cả hai trong CÙNG MỘT database. Relational-document hybrid là combination MẠNH MẼ.

**Lịch sử ít người biết:** Bản mô tả ban đầu của Codd về relational model (1970) đã cho phép thứ tương tự JSON — gọi là **"nonsimple domains"** — giá trị trong 1 row không cần là primitive type, có thể là nested relation (table). Khái niệm này tương đương JSON/XML support được thêm vào SQL hơn 30 năm sau.

---

## Tóm tắt phần này

```
Relational vs Document — KHÔNG CÓ ĐÁP ÁN TUYỆT ĐỐI:

  Document Model TỐT khi:
    + Data có cấu trúc tree (one-to-many), load cả cây 1 lúc
    + Cần locality cao
    + Schema cần FLEXIBLE (schema-on-read)
    + Cần list có thể REORDER

  Relational Model TỐT khi:
    + Many-to-many relationships phức tạp
    + Cần reference trực tiếp đến nested items
    + Cần schema NGHIÊM NGẶT (schema-on-write)

  Star/Snowflake/OBT:
    + Dùng cho ANALYTICS (data warehouse)
    + Denormalization OK vì data là log lịch sử, ít vấn đề consistency

  Xu hướng: HỘI TỤ — relational thêm JSON support,
            document DB thêm join/index/query language
```
