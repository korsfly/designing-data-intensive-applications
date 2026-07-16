# Encoding Formats

---

## 1. Hai cách biểu diễn Data trong Programs

```
TRONG MEMORY:
   Objects, structs, lists, arrays, hash tables, trees,...
   Tối ưu cho CPU access và manipulation (thường dùng POINTERS)

KHI GỬI QUA NETWORK / GHI VÀO FILE:
   Cần encode thành SELF-CONTAINED SEQUENCE OF BYTES
   (ví dụ: JSON document)
   POINTER không còn ý nghĩa với process khác
   → Format thường TRÔNG RẤT KHÁC với in-memory structures
```

### Thuật ngữ quan trọng

| In → Out          | Tên gọi                                                   |
| ----------------- | --------------------------------------------------------- |
| In-memory → bytes | **Encoding** (= Serialization = Marshaling)               |
| Bytes → In-memory | **Decoding** (= Parsing = Deserialization = Unmarshaling) |

> **Lưu ý:** "Serialization" còn được dùng trong context của TRANSACTIONS (Chapter 8) với nghĩa hoàn toàn khác. Sách này dùng "encoding" để tránh nhầm lẫn.

### Khi nào KHÔNG cần Encoding?

```
Database vận hành trực tiếp trên compressed data đã load từ disk
   (Vectorized/JIT query execution — Chapter 4)

Zero-copy data formats: dùng được ở RUNTIME và trên DISK/NETWORK
   mà KHÔNG cần bước conversion tường minh
   Ví dụ: Cap'n Proto, FlatBuffers
```

---

## 2. Language-Specific Formats — Không nên dùng

```
Java: java.io.Serializable
Python: pickle
Ruby: Marshal
Thư viện third-party: Kryo (Java),...
```

### 4 Vấn đề Nghiêm trọng

| Vấn đề                         | Chi tiết                                                                                                                  |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------- |
| **Language lock-in**           | Encoding gắn với NGÔN NGỮ cụ thể → khó đọc từ ngôn ngữ khác → LOCK vào ngôn ngữ hiện tại, NGĂN tích hợp với hệ thống khác |
| **Security risk**              | Decoding cần INSTANTIATE ARBITRARY CLASSES → attacker có thể inject malicious byte sequence → EXECUTE ARBITRARY CODE      |
| **Versioning là afterthought** | Thường BỎ QUA forward/backward compatibility                                                                              |
| **Efficiency kém**             | Java built-in serialization nổi tiếng vì BAD PERFORMANCE và BLOATED encoding                                              |

> **Khuyến nghị:** Chỉ dùng language-specific encoding cho mục đích rất TRANSIENT (tạm thời). KHÔNG dùng cho bất kỳ thứ gì persistent hoặc cross-service.

---

## 3. JSON, XML, CSV — Widely Used nhưng Có Vấn Đề

### Điểm chung

```
Textual formats → somewhat human-readable
JSON và XML: WIDELY KNOWN và WIDELY SUPPORTED
CSV: popular nhưng chỉ hỗ trợ TABULAR data (no nesting)
```

### XML — Bị chỉ trích

```
Too verbose và unnecessarily complicated
```

### CSV — Vague

```
KHÔNG có schema → application tự define ý nghĩa của rows/columns
Schema change (add row/column) → phải handle MANUALLY
Format vague: value chứa COMMA hay NEWLINE thì sao?
   (escaping rules đã được spec, nhưng nhiều parser implement KHÔNG ĐÚNG)
```

### JSON — Phổ biến nhất nhưng vẫn có vấn đề

| Vấn đề                                | Chi tiết                                                                                                      |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| **Ambiguity về numbers**              | Không phân biệt integer và floating-point; không specify precision                                            |
| **Large number problem**              | Integers > 2⁵³ không thể represent exactly trong IEEE 754 double-precision float → parse sai trong JavaScript |
| **X (Twitter) workaround**            | JSON trả về post ID 2 LẦN: 1 lần là JSON number, 1 lần là decimal string (để JS app không parse sai!)         |
| **Không có binary strings**           | JSON và XML hỗ trợ Unicode strings NHƯNG không hỗ trợ binary (sequences of bytes không có char encoding)      |
| **Base64 workaround**                 | Encode binary data thành text bằng Base64 → HACKY, tăng size ~33%                                             |
| **XML Schema / JSON Schema phức tạp** | Powerful nhưng complex; app không dùng schema phải hardcode encoding/decoding logic                           |

### Khi nào JSON/XML/CSV là đủ tốt?

```
Đặc biệt HỮU ÍCH cho DATA INTERCHANGE giữa CÁC TỔ CHỨC:
   Miễn là các bên đồng ý về format → không cần quá đẹp hay hiệu quả

Khó khăn nhất thường là: khiến CÁC TỔ CHỨC ĐỒNG Ý với nhau
   → Điều đó OUTWEIGH hầu hết concern khác
```

---

## 4. JSON Schema — Tiêu chuẩn hiện đại

### Được sử dụng rộng rãi

```
Web services: phần của OpenAPI (Swagger) specification
Schema registries: Confluent Schema Registry, Red Hat Apicurio Registry
Databases: PostgreSQL (pg_jsonschema), MongoDB ($jsonSchema)
```

### Đặc điểm

```
Standard primitive types: string, number, integer, object, array, boolean, null

Validation spec tách biệt: overlay constraints lên fields
   (ví dụ: port field với min=1, max=65535)

Open vs Closed content models:
   OPEN (additionalProperties: true — DEFAULT):
      Cho phép fields KHÔNG ĐỊNH NGHĨA trong schema tồn tại với BẤT KỲ datatype
      → Schema định nghĩa "cái gì KHÔNG được phép" hơn là "cái gì được phép"

   CLOSED (additionalProperties: false):
      Chỉ cho phép fields được ĐỊNH NGHĨA EXPLICITLY
```

```json
// Ví dụ: JSON Schema cho map từ integer keys → string values
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "patternProperties": {
    "^[0-9]+$": { "type": "string" }
  },
  "additionalProperties": false
}
```

### Hạn chế

```
JSON không có map/dictionary type với INTEGER keys
   (JSON objects LUÔN dùng strings làm keys)
   → Phải dùng patternProperties để constrain → phức tạp

JSON Schema hỗ trợ: conditional if/else logic, named types,
   references to remote schemas,... → RẤT POWERFUL nhưng PHỨC TẠP

Remote schema resolution, conditional rules,
schema evolution forward/backward compatible: ALL CHALLENGING
```

---

## 5. Binary Encodings của JSON

### Tại sao tồn tại?

```
JSON ít verbose hơn XML, nhưng VẪN dùng NHIỀU space so với binary formats
→ Dẫn đến nhiều binary encoding cho JSON:
   MessagePack, CBOR, BSON, BJSON, UBJSON, BISON, Hessian, Smile,...
→ Cho XML: WBXML, Fast Infoset
```

### Minh họa: MessagePack (Figure 5-2)

Record gốc (JSON):

```json
{
  "userName": "Martin",
  "favoriteNumber": 1337,
  "interests": ["daydreaming", "hacking"]
}
```

```
Textual JSON (whitespace removed): 81 bytes
MessagePack binary:                66 bytes

→ Tiết kiệm nhỏ (~18%), KHÔNG đáng kể
→ Phải đánh đổi: MẤT human-readability
```

### Vấn đề cốt lõi của binary JSON

```
Vì KHÔNG CÓ SCHEMA, vẫn phải include TẤT CẢ field NAMES
   trong encoded data (userName, favoriteNumber, interests,...)

→ KHÔNG tiết kiệm được nhiều so với textual JSON
→ Không có binary JSON nào được ADOPTED RỘNG RÃI như textual versions
```

---

## 6. Protocol Buffers — Binary Schema-Driven Encoding

### Nguồn gốc

```
Developed at GOOGLE
Tương tự Apache THRIFT (originally from Facebook)
```

### Schema Definition (IDL — Interface Definition Language)

```protobuf
syntax = "proto3";
message Person {
  string user_name       = 1;
  int64  favorite_number = 2;
  repeated string interests = 3;
}
```

```
Code generation tool → generate classes implementing schema
   trong MANY PROGRAMMING LANGUAGES

Application code gọi generated code để ENCODE / DECODE records
   phù hợp với schema
```

### Encoding (Figure 5-3)

```
Encoding record trên:  33 bytes (so với 66 bytes của MessagePack!)

Không có FIELD NAMES trong encoded data
Thay vào đó: FIELD TAGS (numbers: 1, 2, 3)
   → Compact way của "field nào đang được đề cập"
   → Không cần spell out full field name

Packing field type + tag number vào 1 BYTE

Variable-length integers:
   1337 → encoded trong 2 bytes
   (top bit mỗi byte = có còn bytes tiếp theo không;
    7 least significant bits = data)

   Số từ -64 đến 63 → 1 byte
   Số từ -8192 đến 8191 → 2 bytes
   Lớn hơn → nhiều bytes hơn
```

### repeated modifier

```
Protocol Buffers KHÔNG có explicit list/array datatype

"repeated" modifier trên field → field đó là LIST of values
   Trong binary encoding: list elements = REPEATED OCCURRENCES
                          của cùng 1 field tag trong cùng 1 record
```

---

## 7. Avro — Alternative Binary Format

### Nguồn gốc

```
Apache Avro: bắt đầu 2009 như subproject của HADOOP
(Protobuf không phù hợp cho Hadoop's use cases)
```

### Hai schema languages

```
1. Avro IDL: cho human editing
2. JSON representation: cho machine-readable
```

### Schema Example

**Avro IDL:**

```avro
record Person {
  string                    userName;
  union { null, long }      favoriteNumber = null;
  array<string>             interests;
}
```

**JSON representation:**

```json
{
  "type": "record",
  "name": "Person",
  "fields": [
    { "name": "userName", "type": "string" },
    { "name": "favoriteNumber", "type": ["null", "long"], "default": null },
    { "name": "interests", "type": { "type": "array", "items": "string" } }
  ]
}
```

### Encoding — Compact nhất: 32 bytes!

```
KHÔNG CÓ field names, KHÔNG CÓ field tags trong encoded data
KHÔNG có gì identify fields hay datatypes

Encoding = CHỈ concatenate các values lại với nhau

String: length prefix + UTF-8 bytes (không có marker kết thúc)
Integer: variable-length encoding

→ Compact NHẤT trong tất cả encodings đã xem!
```

### Cái bẫy: Đọc phụ thuộc HOÀN TOÀN vào Schema

```
Để parse binary data:
   Đi qua fields THEO THỨ TỰ xuất hiện trong SCHEMA
   Dùng schema để determine DATATYPE của mỗi field

→ Nếu schema của READER và WRITER KHÁC NHAU → DATA DECODED SAI
→ Cần mechanism để resolve sự khác nhau này (xem Section 2)
```

---

## Tóm tắt phần này

```
KHÔNG NÊN DÙNG: Language-specific formats
   (lock-in, security, versioning, efficiency)

OK CHO INTEROP: JSON / XML / CSV
   Phù hợp cho data exchange giữa tổ chức
   Nhưng: vague về datatypes, không hỗ trợ binary natively

COMPACT + SCHEMA-DRIVEN: Protocol Buffers (Protobuf) và Avro
   Protobuf: field tags (numbers) thay cho field names → compact
   Avro: không cả tags, chỉ concatenate values → compact nhất
   Cả 2: cần schema cho encoding/decoding

So sánh size cho cùng 1 record:
   JSON (text):      81 bytes
   MessagePack:      66 bytes
   Protocol Buffers: 33 bytes
   Avro:             32 bytes
```
