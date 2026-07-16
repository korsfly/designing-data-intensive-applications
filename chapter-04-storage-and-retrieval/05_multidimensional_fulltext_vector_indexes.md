# Multidimensional, Full-Text, và Vector Indexes

---

## 1. Giới hạn của B-Tree và LSM-Tree

```
B-tree và LSM-tree cho phép RANGE QUERY trên MỘT ATTRIBUTE

Ví dụ: index trên username → tìm tất cả names bắt đầu bằng 'L'

NHƯNG đôi khi search theo MỘT ATTRIBUTE là KHÔNG ĐỦ
→ Cần indexes PHỨC TẠP HƠN
```

---

## 2. Concatenated Index — Bước đầu tiên

```
MULTI-COLUMN INDEX PHỔ BIẾN NHẤT: Concatenated Index

Đơn giản COMBINE nhiều fields thành 1 key
   bằng cách APPEND column này vào column kia
   (index definition chỉ định THỨ TỰ fields)

Ví dụ: Phone book (lastname, firstname) → phone number

Concatenated index CÓ THỂ dùng để tìm:
   - Tất cả người có LASTNAME cụ thể
   - Tất cả người có LASTNAME + FIRSTNAME cụ thể

NHƯNG VÔ DỤNG nếu muốn tìm theo FIRSTNAME mà không biết LASTNAME
   (do thứ tự ưu tiên: lastname TRƯỚC trong sort order)
```

---

## 3. Multidimensional Indexes — Query Nhiều Cột Đồng Thời

### Use case điển hình: Geospatial Data

```
Restaurant search website:
   Database chứa latitude + longitude của mỗi restaurant
   User nhìn map → website cần tìm TẤT CẢ restaurant
   trong RECTANGULAR MAP AREA đang xem

→ Cần TWO-DIMENSIONAL RANGE QUERY:
   SELECT * FROM restaurants
   WHERE latitude > 51.4946 AND latitude < 51.5079
     AND longitude > -0.1162 AND longitude < -0.1004;
```

### Tại sao Concatenated Index KHÔNG đủ ở đây?

```
Concatenated index trên (latitude, longitude):
   CÓ THỂ cho: tất cả restaurants trong 1 RANGE OF LATITUDES
              (bất kể longitude)
   CÓ THỂ cho: tất cả restaurants trong 1 RANGE OF LONGITUDES
              (bất kể latitude)
   KHÔNG THỂ: cả 2 chiều ĐỒNG THỜI
```

### Giải pháp 1: Space-Filling Curve

```
Translate 2D location → 1 số duy nhất
   → Dùng B-tree index thông thường

(Kỹ thuật toán học — ít phổ biến trong thực tế)
```

### Giải pháp 2: Specialized Spatial Indexes

```
Chia KHÔNG GIAN sao cho điểm gần nhau được GOM VÀO CÙNG SUBTREE

Ví dụ phổ biến:
   R-TREES (Rectangle Trees)
   Bkd-TREES

PostGIS: implement geospatial indexes dưới dạng R-tree
         dùng PostgreSQL's Generalized Search Tree (GiST) facility

CÒN CÓ THỂ dùng: regularly spaced grids of triangles, squares, hexagons
   (ví dụ: Uber H3 — hexagonal hierarchical spatial index)
```

### Multidimensional Indexes KHÔNG CHỈ cho Geographic

```
Ứng dụng NÓI RỘNG HƠN:

E-commerce: 3D index trên (red, green, blue)
   → search products trong 1 RANGE OF COLORS

Weather database: 2D index trên (date, temperature)
   → tìm tất cả observations trong năm X
     có temperature giữa 25°C và 30°C

With 1D index:
   Scan TẤT CẢ records trong năm đó → filter by temperature
   HOẶC: scan TẤT CẢ records trong temp range → filter by year
   → MỘT chiều phải scan nhiều hơn cần thiết

With 2D index:
   Narrow results by CÙNG LÚC timestamp VÀ temperature
   → HIỆU QUẢ HƠN
```

---

## 4. Full-Text Search — Inverted Index

### Định nghĩa

```
Full-text search: search collection of TEXT DOCUMENTS
   (web pages, product descriptions,...)
   bằng KEYWORDS có thể xuất hiện BẤT KỲ ĐÂU trong text

Phức tạp:
   - Language-specific processing
     (Asian languages: không có spaces giữa words,
      cần model để detect word boundaries)
   - Matching SIMILAR (not identical) words
     (typos, grammatical forms, synonyms)
   → Vượt scope của sách này
```

### Full-text như Multidimensional Query

```
Về cốt lõi: full-text search = loại MULTIDIMENSIONAL QUERY

Mỗi word xuất hiện trong text = 1 DIMENSION
   Document chứa term x → value = 1 trong dimension x
   Document KHÔNG chứa term x → value = 0

Tìm docs đề cập "red apples" = query tìm:
   value = 1 trong dimension "red"
   VÀ value = 1 trong dimension "apples"

→ Số dimensions CÓ THỂ RẤT LỚN (mỗi unique word = 1 dimension)
```

### Inverted Index — Cấu trúc Dữ liệu Cốt lõi

```
INVERTED INDEX:
   Key: term (từ)
   Value: list of IDs của TẤT CẢ documents chứa term đó
          (gọi là "POSTINGS LIST")

Nếu document IDs là sequential numbers:
   Postings list → SPARSE BITMAP (bit thứ n = 1 nếu doc n chứa term)

Query "red apples":
   1. Load bitmap cho term "red"
   2. Load bitmap cho term "apples"
   3. Bitwise AND → bitmap chứa docs đề cập CẢ HAI
      (giống vectorized data warehouse query!)

   Dù bitmaps là run-length encoded → NHANH
```

### Ví dụ: Lucene (Elasticsearch, Solr)

```
LUCENE: full-text indexing engine được Elasticsearch và Solr dùng

Cách lưu: term → postings list trong SSTable-LIKE SORTED FILES
   Merge ở background dùng CÙNG LOG-STRUCTURED APPROACH
   như đã thấy ở phần OLTP storage!

PostgreSQL GIN index:
   Dùng postings lists để hỗ trợ full-text search
   VÀ indexing inside JSON documents
```

### N-gram Index

```
Thay vì split text thành WORDS:
   Tìm TẤT CẢ SUBSTRINGS độ dài n (gọi là n-GRAMS)

Ví dụ: TRIGRAMS (n=3) của "hello": hel, ell, llo

Build inverted index của tất cả trigrams
   → CÓ THỂ search arbitrary SUBSTRINGS dài ≥ 3 chars
   → Thậm chí cho phép REGULAR EXPRESSIONS trong search

Nhược điểm: index RẤT LỚN
```

### Fuzzy Matching cho Typos

```
Lucene: CÓ THỂ search words trong 1 EDIT DISTANCE nhất định
   (edit distance = 1: thêm/xóa/thay thế 1 chữ cái)

Implementation:
   Lưu set of terms như FINITE STATE AUTOMATON trên characters
   (giống TRIE)
   Transform thành LEVENSHTEIN AUTOMATON
   → Hỗ trợ efficient search trong given edit distance
```

---

## 5. Vector Embeddings — Semantic Search

### Từ Full-Text đến Semantic Search

```
Full-text search: chỉ khớp về TỪNG TỪ (lexical matching)
   "canceling your subscription" KHÔNG match với
   "how to close my account" hay "terminate contract"
   (hoàn toàn khác words, dù NGHĨA TƯƠNG ĐƯƠNG)

SEMANTIC SEARCH: hiểu CONCEPTS và INTENTIONS
   → Match về NGHĨA, không chỉ về từ ngữ

Use case quan trọng: RAG (Retrieval-Augmented Generation)
   → Tích hợp search results vào output của LLM
```

### Vector Embedding là gì?

```
EMBEDDING MODEL: translate TEXT DOCUMENT
   → VECTOR of floating-point values (vector embedding)
   (thường implemented bằng NEURAL NETWORKS)

Vector = ĐIỂM trong MULTIDIMENSIONAL SPACE
   Mỗi floating-point value = location dọc theo 1 DIMENSION'S AXIS

Embedding models tạo vectors GẦN NHAU khi input documents
   TƯƠNG ĐỒNG VỀ NGHĨA (semantically similar)

Ví dụ (3D — thực tế thường >1000 dimensions):
   Wikipedia page về agriculture: [0.38, 0.83, 0.41]
   Wikipedia page về vegetables: [0.36, 0.64, 0.67]  ← GẦN nhau
   Wikipedia page về star schemas: [0.85, 0.10, -0.52] ← XA hơn
```

### Lưu ý: Hai nghĩa của "Vector"

```
1. "Vectorized processing" (phần trên):
   Vector = BATCH OF BITS được xử lý với optimized code (bitwise ops)

2. "Vector embedding" (phần này):
   Vector = ARRAY OF FLOATING-POINT NUMBERS
            đại diện cho LOCATION trong multidimensional space

→ HOÀN TOÀN KHÁC NHAU, chỉ dùng CÙNG TỪ
```

### Multimodal Embedding

```
Ban đầu: Word2Vec, BERT, GPT → chỉ TEXT
Sau đó: models cho VIDEO, AUDIO, IMAGES
Gần đây: MULTIMODAL models → 1 model generate embeddings
   cho NHIỀU modalities (text + images cùng lúc)
```

### Semantic Search Flow

```
1. User nhập QUERY
2. Query + context (vd: user's location) → EMBEDDING MODEL
3. Embedding model → QUERY VECTOR EMBEDDING
4. Search engine tìm documents có vectors GẦN NHẤT với query vector
   → dùng VECTOR INDEX
```

### Distance Functions

```
Đo KHOẢNG CÁCH giữa 2 vectors để xác định sự tương đồng:

COSINE SIMILARITY: đo cosine của GÓC giữa 2 vectors
   → Xác định mức độ GẦN nhau

EUCLIDEAN DISTANCE: đo khoảng cách ĐƯỜNG THẲNG giữa 2 điểm trong space
```

### 3 Loại Vector Index

#### Flat Index

```
Lưu vector embeddings NGUYÊN VẸN trong index
Query: phải ĐỌC TỪNG VECTOR và đo khoảng cách với query vector

CHÍNH XÁC: cho kết quả EXACT NEAREST NEIGHBOR
NHƯNG CHẬM: đo khoảng cách với MỌI vector khi query lớn
```

#### Inverted File (IVF) Index

```
Cluster vector space thành PARTITIONS (gọi là CENTROIDS)
   → GIẢM số vectors cần compare

NHANH HƠN flat index
NHƯNG: chỉ cho APPROXIMATE RESULTS
   Query và document CÓ THỂ nằm ở PARTITION KHÁC NHAU
   dù chúng GẦN NHAU

PROBE PARAMETER: số partitions cần check
   Nhiều probes → CHÍNH XÁC HƠN nhưng CHẬM HƠN
```

#### HNSW (Hierarchical Navigable Small World) Index

```
Duy trì NHIỀU LAYERS của vector space (Figure 4-11)
   Mỗi layer = GRAPH:
      Node = vector
      Edge = proximity đến nearby vectors

QUERY PROCESS:
   1. Locate NEAREST VECTOR trong TOPMOST LAYER (ÍT nodes)
   2. Move xuống LAYER BÊN DƯỚI TẠI CÙNG NODE
   3. Follow edges trong layer MẬT ĐỘ HƠN
      → tìm vector GẦN HƠN với query vector
   4. Lặp lại cho đến LAST LAYER

Như IVF → APPROXIMATE results (không exact)

HIỆU SUẤT TỐT HỚN IVF trong nhiều trường hợp thực tế
```

### Hệ thống hỗ trợ Vector Index

```
Hầu hết POPULAR VECTOR DATABASES implement IVF VÀ HNSW:
   Facebook Faiss library: nhiều biến thể của cả 2
   PostgreSQL pgvector: hỗ trợ cả 2

Vector databases chuyên biệt: Weaviate, Qdrant, Pinecone, Chroma,...
```

---

## Tóm tắt phần này

```
CONCATENATED INDEX:
   Combine nhiều fields → 1 key
   Chỉ useful khi query theo PREFIX (thứ tự columns quan trọng)

MULTIDIMENSIONAL INDEXES (R-tree, Bkd-tree):
   Query NHIỀU ĐIỀU KIỆN ĐỒNG THỜI (lat+lng, date+temp,...)
   Giải pháp thực tế: R-tree (PostGIS), Bkd-tree, space-filling curves

FULL-TEXT SEARCH (Inverted Index):
   term → postings list (document IDs)
   Sparse bitmap + bitwise AND/OR → query NHANH
   Kỹ thuật nâng cao: n-gram (fuzzy/regex), Levenshtein automaton (typos)
   Ví dụ: Lucene (Elasticsearch/Solr), PostgreSQL GIN

VECTOR EMBEDDINGS (Semantic Search):
   Document → floating-point vector (location trong multidimensional space)
   Embedding model: Word2Vec, BERT, GPT,...; multimodal
   Vector Index types:
      Flat (exact, slow) → IVF (fast, approximate, needs probes)
      → HNSW (fast, approximate, multi-layer graph)
   Cosine similarity / Euclidean distance = khoảng cách giữa vectors
   Tools: Faiss, pgvector, specialized vector DBs
```
