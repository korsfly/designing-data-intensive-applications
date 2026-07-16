# Graph-Like Data Models

---

## 1. Khi nào cần Graph Model?

```
Document Model    → tốt khi data CHỦ YẾU one-to-many (tree structure)
Relational Model  → xử lý được many-to-many ĐƠN GIẢN
GRAPH MODEL       → tốt khi many-to-many relationship RẤT PHỔ BIẾN
                     và connections trong data NGÀY CÀNG PHỨC TẠP
```

### Graph gồm 2 loại object

| Thành phần   | Tên gọi khác        |
| ------------ | ------------------- |
| **Vertices** | Nodes, Entities     |
| **Edges**    | Relationships, Arcs |

### Ví dụ điển hình về Graph Data

| Loại Graph            | Vertices             | Edges             |
| --------------------- | -------------------- | ----------------- |
| **Social graph**      | Người                | Quan hệ biết nhau |
| **Web graph**         | Trang web            | HTML links        |
| **Road/Rail network** | Junction (giao điểm) | Đường/đường sắt   |

**Thuật toán nổi tiếng hoạt động trên Graph:**

- Map navigation apps: tìm **shortest path** giữa 2 điểm
- **PageRank**: xác định độ phổ biến của web page (dùng cho search ranking)

### Hai cách biểu diễn Graph

| Cách                 | Mô tả                                                     | Phù hợp cho      |
| -------------------- | --------------------------------------------------------- | ---------------- |
| **Adjacency List**   | Mỗi vertex lưu ID của các vertex láng giềng (1 edge away) | Graph traversal  |
| **Adjacency Matrix** | Ma trận 2D, 0/1 đánh dấu có/không có edge                 | Machine Learning |

### Graph cho Heterogeneous Data (Không chỉ 1 loại object)

> Một use case mạnh mẽ của Graph là cung cấp **cách lưu trữ thống nhất** cho NHIỀU LOẠI object hoàn toàn khác nhau trong CÙNG MỘT database.

| Hệ thống thực tế                   | Vertices đa dạng                                                                                                          |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| **Facebook's Graph**               | Người, địa điểm, sự kiện, check-in, comments — edges: bạn bè, check-in tại đâu, comment trên post nào, tham gia event nào |
| **Search Engine Knowledge Graphs** | Tổ chức, người, địa điểm (từ crawl & phân tích web, hoặc structured data như Wikidata)                                    |

---

## 2. Hai mô hình Graph chính: Property Graph & Triple Store

```
PROPERTY GRAPH MODEL          TRIPLE STORE MODEL
(Neo4j, Memgraph, KùzuDB)  ≈  (Datomic, AllegroGraph, Blazegraph)

Tương đối GIỐNG NHAU về khả năng biểu đạt
Một số DB (vd: Amazon Neptune) hỗ trợ CẢ HAI
```

**Query languages sẽ tìm hiểu:** Cypher, SPARQL, Datalog, GraphQL + SQL support cho graph.

### Ví dụ xuyên suốt phần này

```
Lucy (từ Idaho) và Alain (từ Saint-Lô, Pháp)
   → Cưới nhau, sống ở London

Vertices: người + địa điểm
Edges: BORN_IN, LIVES_IN, MARRIED_TO, WITHIN (location hierarchy)
```

---

## 3. Property Graph Model

### Cấu trúc Vertex

| Thành phần        | Mô tả                                               |
| ----------------- | --------------------------------------------------- |
| Unique identifier | ID duy nhất                                         |
| Label             | String mô tả loại object (vd: "Person", "Location") |
| Outgoing edges    | Set các edge đi ra                                  |
| Incoming edges    | Set các edge đi vào                                 |
| Properties        | Key-value pairs                                     |

### Cấu trúc Edge

| Thành phần        | Mô tả           |
| ----------------- | --------------- |
| Unique identifier | ID duy nhất     |
| Tail vertex       | Vertex bắt đầu  |
| Head vertex       | Vertex kết thúc |
| Label             | Loại quan hệ    |
| Properties        | Key-value pairs |

### Biểu diễn bằng Relational Schema

```sql
CREATE TABLE vertices (
  vertex_id integer PRIMARY KEY,
  label     text,
  properties jsonb
);
CREATE TABLE edges (
  edge_id     integer PRIMARY KEY,
  tail_vertex integer REFERENCES vertices (vertex_id),
  head_vertex integer REFERENCES vertices (vertex_id),
  label       text,
  properties  jsonb
);
CREATE INDEX edges_tails ON edges (tail_vertex);
CREATE INDEX edges_heads ON edges (head_vertex);
```

### Đặc điểm quan trọng

```
1. BẤT KỲ vertex nào cũng có thể nối với BẤT KỲ vertex khác
   → KHÔNG có schema giới hạn loại nào được liên kết với loại nào

2. Từ 1 vertex, có thể tìm HIỆU QUẢ cả incoming VÀ outgoing edges
   → Traverse graph (đi theo chain of vertices) CẢ HAI HƯỚNG
   (vì vậy có index trên CẢ tail_vertex VÀ head_vertex)

3. Dùng LABEL khác nhau cho loại vertex/relationship khác nhau
   → Lưu NHIỀU loại thông tin trong 1 graph, vẫn giữ data model SẠCH
```

> **Bảng `edges` tương tự** join table many-to-many đã thấy ở phần trước, nhưng được **tổng quát hóa** để lưu NHIỀU loại relationship trong CÙNG 1 table.

### Hạn chế của Graph Model

> Một edge chỉ liên kết được **2 vertex**. Relational join table có thể biểu diễn quan hệ 3-way hoặc cao hơn (nhiều foreign key trên 1 row). Để biểu diễn trong graph: tạo thêm 1 vertex tương ứng với mỗi row của join table, hoặc dùng **hypergraph**.

### Tại sao Graph linh hoạt cho Data Modeling?

```
Ví dụ minh họa (Figure 3-6):
   - Cấu trúc khu vực KHÁC NHAU theo quốc gia
     (Pháp: département/région, US: county/state)
   - "Quirks" lịch sử: quốc gia trong quốc gia
   - Độ chi tiết KHÁC NHAU (Lucy hiện ở mức city,
     nơi sinh chỉ biết đến mức state)

→ Đây là điều RẤT KHÓ biểu diễn trong relational schema truyền thống
```

**Tính evolvability:** Có thể mở rộng graph dễ dàng — ví dụ thêm food allergy (vertex cho allergen, edge person→allergen), rồi link allergen với food chứa chất đó, để query "thức ăn nào an toàn cho mỗi người."

---

## 4. Cypher Query Language

> Cypher: ngôn ngữ query cho Property Graph, ban đầu tạo cho Neo4j, sau phát triển thành chuẩn mở **openCypher**.

**Hỗ trợ bởi:** Neo4j, Memgraph, KùzuDB, Amazon Neptune, Apache AGE (lưu trong PostgreSQL).

### Insert dữ liệu

```cypher
CREATE
  (namerica :Location {name:'North America', type:'continent'}),
  (usa      :Location {name:'United States', type:'country'}),
  (idaho    :Location {name:'Idaho', type:'state'}),
  (lucy     :Person   {name:'Lucy'}),
  (idaho) -[:WITHIN]-> (usa) -[:WITHIN]-> (namerica),
  (lucy)  -[:BORN_IN]-> (idaho)
```

> Tên ký hiệu như `usa`, `idaho` **KHÔNG được lưu trong database** — chỉ dùng NỘI BỘ trong query để tạo edges.

### Query: Tìm người di cư từ US sang Europe

```cypher
MATCH
  (person) -[:BORN_IN]-> () -[:WITHIN*0..]-> (:Location {name:'United States'}),
  (person) -[:LIVES_IN]-> () -[:WITHIN*0..]-> (:Location {name:'Europe'})
RETURN person.name
```

**Đọc hiểu:** Tìm vertex `person` thỏa MÃN CẢ HAI điều kiện:

1. Có outgoing `BORN_IN` edge → theo chain `WITHIN` edges → đến Location có name = "United States"
2. Có outgoing `LIVES_IN` edge → theo chain `WITHIN` edges → đến Location có name = "Europe"

**`*0..` nghĩa là gì?** "Theo edge WITHIN, ZERO HOẶC NHIỀU LẦN" — giống operator `*` trong regular expression.

### Hai cách thực thi Query

```
Cách 1: Bắt đầu từ MỌI PERSON → kiểm tra birthplace/residence của mỗi người
Cách 2: Bắt đầu từ 2 Location vertex (US, Europe) → đi NGƯỢC
        (nếu có index trên `name`, tìm US/Europe HIỆU QUẢ)
        → tìm các location con (states, regions, cities) qua incoming WITHIN
        → tìm người qua incoming BORN_IN/LIVES_IN
```

---

## 5. Graph Queries trong SQL

### Vấn đề chính: Số lượng JOIN không xác định trước

```
Trong relational query thông thường:
   BIẾT TRƯỚC cần bao nhiêu join

Trong graph query:
   CÓ THỂ cần traverse một SỐ LƯỢNG KHÁC NHAU edges
   trước khi tìm được vertex mong muốn
   → số join KHÔNG cố định trước
```

**Giải pháp SQL:** Recursive Common Table Expressions (`WITH RECURSIVE`)

```sql
WITH RECURSIVE
  in_usa(vertex_id) AS (
    SELECT vertex_id FROM vertices
    WHERE label = 'Location' AND properties->>'name' = 'United States'
    UNION
    SELECT edges.tail_vertex FROM edges
    JOIN in_usa ON edges.head_vertex = in_usa.vertex_id
    WHERE edges.label = 'within'
  ),
  in_europe(vertex_id) AS (
    SELECT vertex_id FROM vertices
    WHERE label = 'location' AND properties->>'name' = 'Europe'
    UNION
    SELECT edges.tail_vertex FROM edges
    JOIN in_europe ON edges.head_vertex = in_europe.vertex_id
    WHERE edges.label = 'within'
  ),
  born_in_usa(vertex_id) AS (
    SELECT edges.tail_vertex FROM edges
    JOIN in_usa ON edges.head_vertex = in_usa.vertex_id
    WHERE edges.label = 'born_in'
  ),
  lives_in_europe(vertex_id) AS (
    SELECT edges.tail_vertex FROM edges
    JOIN in_europe ON edges.head_vertex = in_europe.vertex_id
    WHERE edges.label = 'lives_in'
  )
SELECT vertices.properties->>'name'
FROM vertices
JOIN born_in_usa ON vertices.vertex_id = born_in_usa.vertex_id
JOIN lives_in_europe ON vertices.vertex_id = lives_in_europe.vertex_id;
```

> **So sánh ấn tượng:** Cypher query này CHỈ CẦN **4 dòng**. Phiên bản SQL tương đương cần **31 dòng**. Điều này cho thấy **lựa chọn đúng data model và query language** có thể tạo khác biệt LỚN.

**Ghi chú thêm:**

- Oracle có SQL extension riêng cho recursive query, gọi là "hierarchical"
- Ngôn ngữ graph khác: TigerGraph's GSQL, PGQL (Property Graph Query Language)
- **GQL** (Graph Query Language) ISO standard, dựa trên Cypher, công bố 2024 — chưa được adopt rộng rãi nhưng hy vọng tạo uniformity trong tương lai

---

## 6. Triple Stores và SPARQL

### Mô hình Triple

> Mọi thông tin lưu dưới dạng câu 3 phần: **(subject, predicate, object)**

```
Ví dụ: (Jim, likes, bananas)
   Subject = Jim
   Predicate (verb) = likes
   Object = bananas
```

> **Lưu ý kỹ thuật:** Một số DB cần lưu metadata thêm cho mỗi tuple — AWS Neptune dùng **quad** (4-tuple, thêm graph ID); Datomic dùng **5-tuple** (thêm transaction ID + boolean xóa). Vẫn gọi chung là "triple stores" vì giữ cấu trúc subject-predicate-object cơ bản.

### Object có thể là 2 loại

| Loại Object                        | Ý nghĩa                                 | Ví dụ                      |
| ---------------------------------- | --------------------------------------- | -------------------------- |
| Giá trị primitive (string, number) | Predicate+Object = Property (key+value) | `(lucy, birthYear, 1989)`  |
| Vertex khác trong graph            | Predicate = Edge label                  | `(lucy, marriedTo, alain)` |

### Turtle Format (subset của Notation3/N3)

```turtle
@prefix : <urn:example:>.
_:lucy   a :Person;
         :name "Lucy";
         :bornIn _:idaho.
_:idaho  a :Location; :name "Idaho";
         :type "state"; :within _:usa.
_:usa    a :Location; :name "United States"; :type "country"; :within _:namerica.
_:namerica a :Location; :name "North America"; :type "continent".
```

> `_:lucy` = tên vertex CHỈ DÙNG TRONG FILE NÀY (không có nghĩa ngoài context).

### The Semantic Web

```
RDF/Triple Stores xuất phát từ phong trào "Semantic Web" (đầu 2000s)
   Mục tiêu: trao đổi data INTERNET-WIDE, dạng machine-readable

KẾT QUẢ: Semantic Web (như tầm nhìn ban đầu) KHÔNG thành công
NHƯNG di sản còn lại:
   - JSON-LD (linked data standard)
   - Ontology trong biomedical science
   - Facebook's Open Graph protocol (dùng cho link unfurling)
   - Knowledge graphs (Wikidata)
   - Schema.org (structured data vocabularies)
```

### RDF Data Model

```
Turtle là MỘT CÁCH encode dữ liệu trong RDF (Resource Description Framework)
RDF cũng có thể encode qua XML (verbose hơn)
Tools như Apache Jena có thể convert TỰ ĐỘNG giữa các encoding
```

**Quirk của RDF:** Subject, predicate, object thường là **URI** (vd: `<http://my-company.com/namespace#within>` thay vì chỉ `WITHIN`).

> **Lý do thiết kế:** Cho phép kết hợp data của bạn với data của người khác — nếu họ định nghĩa "within" khác nghĩa, sẽ KHÔNG conflict vì predicate thực ra khác URI (namespace).

### SPARQL Query Language

> SPARQL (đọc "sparkle") = SPARQL Protocol and RDF Query Language. **Có TRƯỚC Cypher**, và Cypher's pattern matching được MƯỢN từ SPARQL → 2 ngôn ngữ TRÔNG RẤT GIỐNG NHAU.

```sparql
PREFIX : <urn:example:>
SELECT ?personName WHERE {
  ?person :name ?personName.
  ?person :bornIn / :within* / :name "United States".
  ?person :livesIn / :within* / :name "Europe".
}
```

**So sánh cú pháp Cypher vs SPARQL:**

```
(person) -[:BORN_IN]-> () -[:WITHIN*0..]-> (location)   # Cypher
?person :bornIn / :within* ?location.                     # SPARQL
```

> RDF KHÔNG phân biệt giữa property và edge — cả 2 đều dùng "predicate". Nên cú pháp match property GIỐNG cú pháp match relationship.

**Hỗ trợ bởi:** Amazon Neptune, AllegroGraph, Blazegraph, OpenLink Virtuoso, Apache Jena.

---

## 7. Datalog: Recursive Relational Queries

### Bối cảnh

```
Datalog: ngôn ngữ CỔ HƠN nhiều (xuất phát academic research 1980s)
Ít người biết đến trong giới software engineer
KHÔNG được hỗ trợ rộng rãi trong mainstream database
NHƯNG: rất EXPRESSIVE, đặc biệt mạnh cho QUERY PHỨC TẠP

Databases dùng Datalog: Datomic, LogicBlox, CozoDB, LinkedIn's LIquid

→ Dựa trên RELATIONAL data model (không phải graph)
   NHƯNG thảo luận ở đây vì RECURSIVE QUERIES trên graph
   là điểm MẠNH ĐẶC BIỆT của Datalog
```

### Facts trong Datalog

```
table(val1, val2, ...) nghĩa là: table chứa 1 dòng
   với column 1 = val1, column 2 = val2,...

Ví dụ:
location(2, "United States", "country").  /* US là 1 country, ID=2 */
within(2, 1).  /* US within North America (ID=1) */
person(100, "Lucy").
born_in(100, 3). /* Lucy (ID=100) sinh ở Idaho (ID=3) */
```

### Rules trong Datalog

```prolog
within_recursive(LocID, PlaceName) :- location(LocID, PlaceName, _). /* Rule 1 */
within_recursive(LocID, PlaceName) :- within(LocID, ViaID),          /* Rule 2 */
                                       within_recursive(ViaID, PlaceName).

migrated(PName, BornIn, LivingIn) :- person(PersonID, PName),         /* Rule 3 */
                                      born_in(PersonID, BornID),
                                      within_recursive(BornID, BornIn),
                                      lives_in(PersonID, LivingID),
                                      within_recursive(LivingID, LivingIn).

us_to_europe(Person) :- migrated(Person, "United States", "Europe").  /* Rule 4 */
```

### Cách hiểu Datalog

```
KHÁC với Cypher/SPARQL (nhảy thẳng vào SELECT)
Datalog làm TỪNG BƯỚC NHỎ:

   Định nghĩa RULES → tạo ra VIRTUAL TABLES (derived tables) từ facts gốc
   Virtual tables giống SQL VIEWS: KHÔNG lưu trong DB, nhưng query được
      như table chứa facts thường

Bên TRÁI :- = tên + cột của virtual table
Bên PHẢI :- = pattern cần match trong table khác (gồm cả virtual table khác)

Rule ÁP DỤNG khi tìm được match cho TẤT CẢ pattern bên phải :-
   → Như thể bên trái được THÊM vào database
```

### Ví dụ thực thi Rule (Recursive)

```
1. location(1, "North America", "continent") tồn tại
   → Rule 1 áp dụng → tạo within_recursive(1, "North America")

2. within(2, 1) tồn tại + within_recursive(1, "North America") (từ bước trước)
   → Rule 2 áp dụng → tạo within_recursive(2, "North America")

3. within(3, 2) tồn tại + within_recursive(2, "North America")
   → Rule 2 áp dụng → tạo within_recursive(3, "North America")

→ Bằng việc ÁP DỤNG LẶP LẠI rule 1 và 2,
  within_recursive cho biết MỌI location nằm trong North America
```

> **So sánh với Function/Recursion trong code:** Datalog rules giống như BREAK CODE THÀNH FUNCTIONS gọi nhau. Giống function có thể RECURSIVE, Datalog rules cũng có thể GỌI CHÍNH NÓ (Rule 2 tự gọi `within_recursive`) → cho phép GRAPH TRAVERSAL.

---

## Tóm tắt phần này

```
Graph Model phù hợp khi: many-to-many relationships PHỨC TẠP,
                          mọi thứ CÓ THỂ liên quan đến mọi thứ

2 mô hình chính:
┌──────────────────┬─────────────────────┬───────────────────────┐
│                   │ PROPERTY GRAPH      │ TRIPLE STORE          │
├──────────────────┼─────────────────────┼───────────────────────┤
│ Cấu trúc cơ bản   │ Vertex + Edge        │ (subject, predicate,  │
│                   │ (với label+property)│  object)              │
│ Query Language    │ Cypher               │ SPARQL                │
│ Database ví dụ     │ Neo4j, Memgraph     │ Datomic, AllegroGraph │
└──────────────────┴─────────────────────┴───────────────────────┘

DATALOG: tiếp cận RELATIONAL, mạnh về RECURSIVE QUERY
         dùng RULES (như function) thay vì SELECT trực tiếp

Bài học chính: Cùng 1 query — Cypher cần 4 dòng,
               SQL recursive CTE cần 31 dòng
               → Chọn đúng data model + query language = KHÁC BIỆT LỚN
```
