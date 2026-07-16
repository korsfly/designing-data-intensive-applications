# Dataflow Through Databases and Services

---

## 1. Ba Modes of Dataflow

```
Data flow từ process này sang process khác theo 3 cách chính:

1. DATABASES
   Process WRITE encode data → process READ decode data
   (có thể là cùng 1 process, nhưng ở thời điểm khác nhau)

2. SERVICE CALLS (REST / RPC)
   Client encode REQUEST → Server decode request, encode RESPONSE
   → Client decode response

3. ASYNC MESSAGE PASSING
   Sender encode MESSAGE → Receiver decode message
   (qua message broker ở giữa)
```

---

## 2. Dataflow Through Databases

### "Gửi Message cho Future Self"

```
Nếu chỉ 1 process access database:
   Process WRITE = process READ (chỉ ở thời điểm KHÁC)
   → Giống "gửi message cho FUTURE SELF"
   → Cần backward compatibility rõ ràng
     (future self cần đọc được data của past self)
```

### Nhiều Processes: Cần Cả Hai Chiều Compatibility

```
Thực tế: nhiều processes access database ĐỒNG THỜI
   (multiple instances, rolling upgrade → mix new + old code)

Một value trong DB có thể:
   Được VIẾT bởi code MỚI → được ĐỌC bởi code CŨ (forward compat!)
   Được VIẾT bởi code CŨ → được ĐỌC bởi code MỚI (backward compat!)

→ DATABASE CẦN CẢ 2 CHIỀU
```

### Vấn đề: Unknown Fields bị Mất (Figure 5-1)

```
Kịch bản nguy hiểm:
   1. Code MỚI viết record có field MỚI vào DB
   2. Code CŨ đọc record đó (không biết field mới)
   3. Code CŨ UPDATE record và GHI LẠI vào DB

Nếu code cũ decode record thành object model
   mà KHÔNG PRESERVE unknown fields:
   → Field mới BỊ MẤT khi ghi lại!

Giải pháp:
   Encoding format phải cho phép old code SKIP field mới
   (Protobuf làm được nhờ field tags + datatype annotation)
   Application code phải KHÔNG drop unknown fields khi update
```

### "Data Outlives Code"

> Một trong những insight quan trọng nhất của chương:

```
Khi deploy version MỚI của server-side application:
   Có thể REPLACE HOÀN TOÀN code cũ trong VÀI PHÚT

Database KHÔNG GIỐNG NHƯ VẬY:
   Data viết 5 NĂM TRƯỚC vẫn ở đó, encoding gốc
   TRỪ KHI bạn đã explicitly rewrite/migrate nó

→ DATA OUTLIVES CODE
```

### Schema Migrations — Chiến lược

```
REWRITE (MIGRATE) tất cả data sang new schema:
   Possible NHƯNG expensive trên large dataset
   → Hầu hết DB DEFER việc này, thực hiện ASYNC on best-effort basis

Ví dụ:
   LSM-tree storage: rewrite data với LATEST FORMAT trong lúc compaction
   Relational DB: ADD COLUMN với null default value
      → KHÔNG rewrite existing data
      → Khi đọc old rows: fill null cho columns MISSING từ encoded data on disk

→ Schema evolution cho phép DB TRÔNG NHƯ được encode với 1 schema duy nhất
  dù underlying storage chứa records encode với NHIỀU HISTORICAL VERSIONS

Complex schema changes (vd: single-valued → multivalued, split table):
   → VẪN cần data rewrite ở APPLICATION LEVEL
   → Maintaining forward/backward compat qua migrations: OPEN RESEARCH PROBLEM
```

### Archival Storage

```
Snapshot database cho backup / load vào data warehouse:
   Data dump thường được encode với LATEST SCHEMA
   (kể cả nếu original encoding là mix of schema versions)

→ Vì đang copy data anyway, encode consistently luôn

Data dump: write once, immutable sau đó
   → Phù hợp với Avro object container files
   → Cũng là cơ hội encode sang analytics-friendly columnar format (Parquet)
```

---

## 3. Dataflow Through Services: REST và RPC

### Web và Services

```
Arrangement phổ biến nhất: CLIENTS và SERVERS

Servers expose API qua network → Clients connect, make requests
API exposed by server = SERVICE

Web browser (client) → Web server:
   GET: download HTML, CSS, JS, images
   POST: submit data
   Standard protocols: HTTP, URLs, SSL/TLS, HTML,...
```

### Services khác Databases như thế nào?

```
DATABASES: cho phép arbitrary queries (SQL, etc.)
SERVICES: expose APPLICATION-SPECIFIC API
   Chỉ cho phép inputs/outputs được predetermined bởi business logic

→ Encapsulation: services IMPOSE restrictions trên clients
→ Clients KHÔNG thể tùy ý query underlying data
```

### Microservices và Evolvability

```
Design goal của microservices/SOA:
   Services INDEPENDENTLY DEPLOYABLE và EVOLVABLE
   Mỗi service owned by 1 team → team release thường xuyên
   KHÔNG cần coordinate với teams khác

→ Old và new versions của server/client sẽ chạy ĐỒNG THỜI
→ Data encoding PHẢI compatible across versions
→ Miễn là API compatible: teams tự do modify internal systems
```

---

## 4. Web Services: REST

### REST (Representational State Transfer)

```
Design philosophy xây trên PRINCIPLES OF HTTP
   Roy Thomas Fielding — PhD thesis 2000

REST emphasizes:
   Simple data formats
   URLs để identify resources
   HTTP features cho: cache control, authentication, content negotiation

API thiết kế theo REST principles → gọi là RESTFUL API
```

### Cách Clients "biết" API

```
Client cần biết: HTTP endpoint nào để query,
                 data format nào để gửi/nhận

Service developers thường dùng IDL để:
   Define và document API endpoints và data models
   Evolve chúng theo thời gian

2 IDL phổ biến nhất:
   OPENAPI (Swagger): cho web services dùng JSON
   PROTOCOL BUFFERS: cho gRPC services
```

### OpenAPI — Ví dụ

```yaml
openapi: 3.0.0
info:
  title: Ping, Pong
  version: 1.0.0
servers:
  - url: http://localhost:8080
paths:
  /ping:
    get:
      summary: Given a ping, returns a pong message
      responses:
        "200":
          description: A pong
          content:
            application/json:
              schema:
                type: object
                properties:
                  message:
                    type: string
                    example: Pong!
```

### Service Frameworks

```
Framework (Spring Boot, FastAPI, gRPC) cho phép developer
   TẬP TRUNG vào business logic của mỗi endpoint
   Framework lo: routing, metrics, caching, authentication,...

FastAPI (Python): viết server code, IDL được GENERATE TỰ ĐỘNG
gRPC: viết service definition TRƯỚC, server code scaffolding GENERATE

Cả hai: có thể generate CLIENT LIBRARIES + SDKs nhiều ngôn ngữ
IDL tools (Swagger,...): generate docs, verify schema compatibility,
                         GUI để test services
```

---

## 5. Vấn đề với Remote Procedure Calls (RPC)

### Lịch sử RPC

```
EJB (Enterprise JavaBeans): chỉ Java
Java RMI: chỉ Java
DCOM: chỉ Microsoft platforms
CORBA: excessively complex, không có backward/forward compat
SOAP / WS-*: aim for interoperability nhưng bị plagued by complexity

Tất cả đều dựa trên ý tưởng RPC (1970s):
   Làm cho request đến remote network service TRÔNG GIỐNG
   calling a function/method TRONG CÙNG PROCESS

→ Gọi là "LOCATION TRANSPARENCY" (abstraction)
```

### Tại sao RPC Fundamentally Flawed

> Network request KHÔNG GIỐNG local function call:

| Khía cạnh          | Local Function Call             | Network Request                                                       |
| ------------------ | ------------------------------- | --------------------------------------------------------------------- |
| **Predictability** | Predictable, chỉ fail do params | Unpredictable (network loss, remote slow/unavailable)                 |
| **Return values**  | Result HOẶC exception           | Result, exception, HOẶC TIMEOUT (không biết điều gì xảy ra)           |
| **Retry safety**   | An toàn để retry                | Retry có thể execute action NHIỀU LẦN (cần idempotency/deduplication) |
| **Latency**        | ~constant, predictable          | Wildly variable (< 1ms đến nhiều giây)                                |
| **Parameters**     | Efficient pointers trong memory | Phải encode thành bytes (object lớn → vấn đề)                         |
| **Language**       | Cùng language                   | Client và server có thể khác language → cần type translation          |

```
KHOẢNG CÁCH VÀ TIMEOUT:
   Network request → có thể timeout
   Sau timeout: KHÔNG BIẾT request đã được nhận chưa
   (xem thêm Chapter 9 — "The Trouble with Distributed Systems")

→ Không thể giả vờ network call = local function call
→ REST treat state transfer qua network như 1 PROCESS DISTINCT FROM function call
   (đây là 1 phần lý do REST được ưa thích)
```

---

## 6. Load Balancers, Service Discovery, Service Meshes

### Service Discovery Problem

```
Client cần biết ADDRESS của service muốn connect
Đơn giản nhất: configure IP address + port cố định
NHƯNG: nếu server down, move to new machine, or overloaded
   → Phải manually reconfigure client

→ Cần ĐỘNG HƠN: Service Discovery
```

### Các giải pháp Load Balancing + Service Discovery

#### Hardware Load Balancers

```
Specialized equipment trong datacenter
Client connect đến 1 host:port duy nhất
→ Load balancer route đến 1 trong nhiều servers
Detect failures → shift traffic sang server khác
```

#### Software Load Balancers (NGINX, HAProxy)

```
Behave tương tự hardware load balancers
NHƯNG: applications cài trên standard machine
→ Không cần specialized appliance
```

#### DNS

```
Cho phép nhiều IP address gắn với 1 domain name
Client resolve domain → pick 1 IP address

DRAWBACK: DNS designed cho propagate changes CHẬM + cache DNS entries
   Nếu servers start/stop/move thường xuyên
   → Clients có thể thấy STALE IP addresses không còn server
```

#### Service Discovery Systems (etcd, ZooKeeper)

```
Dùng CENTRALIZED REGISTRY thay vì DNS

Khi service instance MỚI start:
   → Register với discovery system (host + port + metadata)
   → Gửi HEARTBEAT định kỳ để báo vẫn available

Client muốn connect:
   → Query discovery system lấy list of available endpoints
   → Connect TRỰC TIẾP đến endpoint

So với DNS: hỗ trợ DYNAMIC ENVIRONMENT HƠN
   Services change thường xuyên → cập nhật nhanh
   Clients có thêm METADATA về service
   → Có thể đưa ra smarter load-balancing decisions

(Xem thêm "Coordination Services" — Chapter 10)
```

#### Service Meshes (Istio, Linkerd)

```
Sophisticated form: kết hợp software load balancers + service discovery

TOPOLOGY:
   Load balancer deployed as IN-PROCESS CLIENT LIBRARY
   hoặc as PROCESS / "SIDECAR" CONTAINER ở cả CLIENT và SERVER side

   Client app → Local load balancer → Server's load balancer
                                               → Local server process

ADVANTAGES:
   Clients và servers route qua LOCAL CONNECTIONS
   → Connection encryption hoàn toàn ở LOAD BALANCER LEVEL
   → Shielded clients/servers khỏi complexity của SSL certs + TLS

   Sophisticated OBSERVABILITY:
   → Track which services calling which (real-time)
   → Detect failures
   → Track traffic load

Phù hợp cho: DYNAMIC environment với orchestrator (Kubernetes)
Simpler deployments: software load balancers (NGINX, HAProxy) là đủ
```

---

## 7. Data Encoding và Evolution cho RPC

### Assumption quan trọng (khác với Databases)

```
Với dataflow qua databases:
   KHÔNG giả định được AI sẽ read data
   Cả 2 chiều compat đều cần

Với dataflow qua services:
   CÓ THỂ assume: ALL SERVERS UPDATED FIRST, ALL CLIENTS SECOND

→ Chỉ cần:
   BACKWARD COMPATIBILITY trên REQUESTS (server phải đọc được request cũ từ client cũ)
   FORWARD COMPATIBILITY trên RESPONSES (client cũ phải đọc được response mới từ server mới)
```

### Encoding Format quyết định Compatibility

```
gRPC (Protocol Buffers): evolve theo Protobuf compatibility rules
Avro RPC: evolve theo Avro compatibility rules
RESTful APIs (JSON):
   Thêm optional request params → backward compatible
   Thêm new fields vào response objects → forward compatible
```

### API Versioning — Không có chuẩn

```
Service compatibility khó hơn vì:
   RPC thường dùng cho cross-organizational communication
   → Service provider KHÔNG có control over clients
   → Clients KHÔNG bị force upgrade
   → Phải maintain compatibility RẤT LÂU (có thể vô thời hạn)
   → Nếu cần breaking change: maintain MULTIPLE VERSIONS song song

API versioning approaches:
   RESTful: version number TRONG URL (v1/v2) HOẶC HTTP Accept header
   API keys: store client's version selection server-side, update qua admin interface
   → Không có chuẩn chung
```

---

## Tóm tắt phần này

```
DATAFLOW QUA DATABASES:
   Cần CẢ 2 chiều compat (mix old/new code)
   Data outlives code → DB có thể có mixture of schema versions
   Schema migration: defer, best-effort, lazy
   Archival: encode với latest schema

DATAFLOW QUA SERVICES (REST/RPC):
   REST: HTTP verbs, URLs, resources, simple data formats (JSON)
   Clients cần biết API qua IDL: OpenAPI (JSON), Protobuf (gRPC)
   Service frameworks: Spring Boot, FastAPI, gRPC

RPC PROBLEMS:
   Network request KHÔNG phải local function call
   Timeout, retry (idempotency), variable latency, cross-language types

LOAD BALANCING + SERVICE DISCOVERY:
   Hardware LB → Software LB (NGINX) → DNS → Service Discovery (etcd/ZK) → Service Mesh (Istio)
   Service Mesh: sidecar pattern, encryption at LB level, observability

RPC EVOLUTION:
   Servers updated first → backward compat on requests, forward compat on responses
   API versioning: NO standard (URL versions, HTTP headers, API keys)
```
