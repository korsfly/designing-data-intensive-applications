# Schema Evolution

---

## 1. Vấn đề cốt lõi: Schema thay đổi theo thời gian

```
Schema LUÔN phải thay đổi = SCHEMA EVOLUTION

Câu hỏi: Làm sao thay đổi schema mà VẪN DUY TRÌ
   backward và forward compatibility?
```

---

## 2. Protocol Buffers — Schema Evolution qua Field Tags

### Field Tags là chìa khóa

```
Encoded record = concatenation của các encoded fields
Mỗi field: xác định bởi TAG NUMBER + datatype annotation

Nếu field value KHÔNG được set → OMIT khỏi encoded record
```

### Rule 1: Thêm Field Mới (Adding Fields)

```
CÓ THỂ thêm field mới, MIỄN LÀ gán TAG NUMBER MỚI

FORWARD COMPATIBILITY (code CŨ đọc data MỚI):
   Code cũ gặp tag number không biết → SKIP field đó
   Datatype annotation cho biết cần skip BAO NHIÊU BYTES
   → KHÔNG gây lỗi, không mất data (Figure 5-1 được giải quyết!)

BACKWARD COMPATIBILITY (code MỚI đọc data CŨ):
   Field mới chưa có trong data cũ → fill với DEFAULT VALUE
   (empty string nếu là string, 0 nếu là number)
```

### Rule 2: Xóa Field (Removing Fields)

```
GIỐNG thêm field nhưng ngược chiều compatibility

CÓ THỂ xóa field, NHƯNG:
   KHÔNG BAO GIỜ dùng lại tag number đó
   (Data cũ có thể vẫn chứa tag number đó → code mới phải ignore)

→ Tag numbers đã dùng trong quá khứ có thể RESERVE trong schema
  để đảm bảo chúng không bị dùng lại VÔ TÌNH
```

### Rule 3: Thay đổi Datatype

```
POSSIBLE với một số types — check documentation
NHƯNG CÓ RỦI RO:

Ví dụ: 32-bit integer → 64-bit integer
   Code MỚI đọc data CŨ (32-bit value) → OK (fill missing bits với 0s)
   Code CŨ đọc data MỚI (64-bit value) → VẤN ĐỀ
      Nếu 64-bit value KHÔNG VỪA 32 bits → TRUNCATED (mất data!)
```

### Rule quan trọng nhất

```
TAG NUMBER = ALIASED CHO FIELD NAME
   → CÓ THỂ đổi field name (encoded data không reference tới field name)
   → KHÔNG THỂ đổi field's TAG NUMBER
     (sẽ làm TẤT CẢ existing encoded data trở nên INVALID)
```

---

## 3. Avro — Schema Evolution qua Writer/Reader Schema

### Hai Schemas trong Avro Decoding

> **Protobuf/Thrift:** encoding và decoding có thể dùng DIFFERENT VERSIONS của cùng 1 schema.
> **Avro:** decoding cần ĐỒNG THỜI cả 2 schemas — writer's schema VÀ reader's schema.

```
WRITER'S SCHEMA:
   Version schema được COMPILE vào application khi ENCODE data
   (schema của phía ghi)

READER'S SCHEMA:
   Version schema mà APPLICATION CODE phía đọc MONG ĐỢI
   (có thể là OLDER hoặc NEWER version của cùng 1 schema)
```

### Avro Schema Resolution (Figure 5-6)

```
Nếu writer's schema VÀ reader's schema GIỐNG NHAU → decode dễ

Nếu KHÁC NHAU:
   Avro SO SÁNH 2 schemas và DỊCH data từ writer's sang reader's schema

Match fields: BY FIELD NAME (không phải by position!)
   → OK kể cả khi fields có THỨ TỰ KHÁC NHAU

Field trong writer's schema NHƯNG KHÔNG có trong reader's:
   → IGNORED (skipped)

Field trong reader's schema NHƯNG KHÔNG có trong writer's:
   → Filled với DEFAULT VALUE từ reader's schema
```

### Schema Evolution Rules trong Avro

```
FORWARD COMPATIBILITY: writer dùng NEWER schema → reader dùng OLDER schema
BACKWARD COMPATIBILITY: writer dùng OLDER schema → reader dùng NEWER schema

ĐỂ DUY TRÌ compatibility:
   CHỈ CÓ THỂ add/remove field NẾU FIELD ĐÓ CÓ DEFAULT VALUE

Ví dụ: thêm field có default value
   Reader với NEW schema đọc data của OLD writer (không có field đó)
   → Fill với default value = OK (backward compatible)

   Reader với OLD schema đọc data của NEW writer (có field đó)
   → Ignore field đó = OK (forward compatible)

VẤN ĐỀ NẾU VI PHẠM:
   Thêm field KHÔNG CÓ default value:
     → New readers KHÔNG THỂ đọc data cũ → PHÁ backward compatibility

   Xóa field KHÔNG CÓ default value:
     → Old readers KHÔNG THỂ đọc data mới → PHÁ forward compatibility
```

### Null trong Avro — Explicit Union

```
Trong nhiều ngôn ngữ, null = acceptable default
NHƯNG TRONG AVRO: null PHẢI khai báo tường minh

Dùng UNION TYPE để cho phép null:
   union { null, long, string } field;
   → field có thể là number, string, HOẶC null

Chỉ dùng null làm DEFAULT VALUE nếu null là
   FIRST BRANCH của union type

Tại sao?
   Verbose hơn, nhưng EXPLICIT VỀ những gì có thể và không thể null
   → Giúp PHÒNG NGỪA BUGS
   (như Tony Hoare nói: null references = "The Billion Dollar Mistake")
```

### Thay đổi khác trong Avro

| Thay đổi                             | Compatibility                                                |
| ------------------------------------ | ------------------------------------------------------------ |
| Thêm field có default                | Backward + Forward compatible                                |
| Xóa field có default                 | Backward + Forward compatible                                |
| Thêm field KHÔNG có default          | Phá backward compatibility                                   |
| Xóa field KHÔNG có default           | Phá forward compatibility                                    |
| Đổi datatype (nếu Avro convert được) | Possible                                                     |
| Đổi field name                       | Backward compatible (dùng aliases), KHÔNG forward compatible |
| Thêm branch vào union type           | Backward compatible, KHÔNG forward compatible                |

---

## 4. Avro: Làm sao Reader biết Writer's Schema?

### Vấn đề

```
Reader CẦN writer's schema để decode → Nhưng KHÔNG THỂ include
   toàn bộ schema vào MỖI RECORD (schema có thể lớn hơn data!)
```

### 3 Giải pháp tùy Context

#### Large file với lots of records

```
Ví dụ: Hadoop large file với hàng triệu records, tất cả cùng schema
→ Writer include schema MỘT LẦN ở ĐẦU FILE
→ Avro object container files dùng cách này
```

#### Database với individually written records

```
Different records có thể dùng DIFFERENT SCHEMAS
(written at different times, schema đã thay đổi)

Giải pháp đơn giản:
   Include VERSION NUMBER ở đầu mỗi encoded record
   Giữ danh sách schema versions trong database

Reader: fetch record → extract version number
        → fetch writer's schema của version đó từ DB
        → decode phần còn lại của record với schema đó

Ví dụ: Confluent Schema Registry cho Apache Kafka,
        LinkedIn's Espresso
```

#### Sending records over network connection

```
2 processes communicate qua bidirectional network connection:
   → NEGOTIATE schema version khi setup connection
   → Dùng schema đó cho LIFETIME của connection

Avro RPC protocol dùng cách này
```

### Lợi ích của Database Schema Versions

```
Cũng hữu ích cho documentation và checking compatibility:
   → Check xem schema change có BACKWARD/FORWARD COMPATIBLE không
     TRƯỚC KHI deploy
   → Số version: simple incrementing integer HOẶC hash của schema
```

---

## 5. Dynamically Generated Schemas — Lợi thế của Avro

### Vấn đề với Protobuf khi Schema Dynamic

```
Ví dụ: Dump relational database vào binary file

Với Protobuf: field tags phải được gán BY HAND
   Mỗi khi database schema thay đổi (add/remove column)
   → Admin phải MANUALLY UPDATE mapping từ column names → field tags
   → Khó automate vì schema generator cần CẨN THẬN
     không gán lại PREVIOUSLY USED tag numbers

→ Protobuf NOT DESIGNED cho dynamically generated schema
```

### Avro tốt hơn cho Dynamic Schemas

```
Ví dụ tương tự với Avro:
   Generate Avro schema TỰ ĐỘNG từ relational schema
   (mỗi table → 1 record schema, mỗi column → 1 field)

Nếu database schema THAY ĐỔI:
   → Generate Avro schema MỚI TỰ ĐỘNG từ updated database schema
   → Export data với schema mới
   → Data export process KHÔNG cần biết gì về schema change
     (chỉ cần generate schema mới mỗi lần chạy)

Reader thấy fields thay đổi, nhưng vì fields được identify BY NAME
   → Updated writer's schema vẫn match được với old reader's schema

→ Dynamic schema generation ĐƠN GIẢN HƠN NHIỀU với Avro
```

---

## 6. Merits of Schema-Based Binary Encoding

### So sánh với JSON Schema và XML Schema

```
Protobuf và Avro schema languages ĐƠN GIẢN HƠN nhiều so với
JSON Schema hay XML Schema:
   - Chỉ define fields và types
   - Không support complex validation rules
     ("string này phải match regex này",
      "integer này phải trong range 0-100")

→ Đơn giản hơn để implement và use
→ Đã có wide range of programming language support
```

### 4 Lợi ích chính

| Lợi ích                                            | Giải thích                                                                          |
| -------------------------------------------------- | ----------------------------------------------------------------------------------- |
| **Compact hơn**                                    | Omit field names → tiết kiệm space nhiều hơn binary JSON variants                   |
| **Schema là documentation**                        | Schema required cho decoding → LUÔN up-to-date (không như manually maintained docs) |
| **Kiểm tra compatibility trước deploy**            | Database of schemas → check forward/backward compatibility trước khi deploy change  |
| **Code generation cho statically-typed languages** | Generate code từ schema → type checking tại compile time                            |

### Schema Evolution = Flexibility của Schemaless + Better Guarantees

```
Schema evolution cho phép FLEXIBILITY TƯƠNG TỰ schema-on-read JSON databases
   (xem Chapter 3)

NHƯNG CÒN CHO:
   Better guarantees về data
   Better tooling (code generation, compatibility checks)
```

> **Practical advice:** Giữ số lượng concurrent schema formats ở MỨC TỐI THIỂU để giữ operations đơn giản.

---

## Tóm tắt phần này

```
PROTOCOL BUFFERS — Schema Evolution:
   Field tags là PERMANENT ALIASES cho fields
   Thêm field: tag mới → forward compatible (old code skip)
   Xóa field: KHÔNG dùng lại tag → backward compatible
   Đổi datatype: OK với một số pairs (cẩn thận truncation)

AVRO — Schema Evolution:
   Writer's schema + Reader's schema → Avro RESOLVE sự khác nhau
   Match by FIELD NAME (không phải position hay tag)
   Add/remove field: CHỈ OK nếu có DEFAULT VALUE
   Null: phải explicit dùng UNION TYPE
   Tốt hơn Protobuf cho DYNAMICALLY GENERATED SCHEMAS

MERITS OF SCHEMA-BASED ENCODING:
   Compact (omit field names) + Documentation + Compatibility check
   + Code generation (type safety)
```
