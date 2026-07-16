# Distributed vs Single-Node Systems

---

## 1. Distributed System là gì?

**Distributed system:** Hệ thống gồm **nhiều machines giao tiếp qua network**.  
Mỗi process tham gia gọi là một **node**.

---

## 2. Khi nào cần Distributed System?

### Các lý do chính

| Lý do                                   | Giải thích                                                                         |
| --------------------------------------- | ---------------------------------------------------------------------------------- |
| **Inherent distribution**               | Nhiều user trên nhiều thiết bị → bắt buộc phải distributed qua network             |
| **Cloud services interoperability**     | Data ở service A, xử lý ở service B → phải truyền qua network                      |
| **Fault tolerance / High availability** | Redundancy: khi một node fail, node khác tiếp quản                                 |
| **Scalability**                         | Data/compute vượt quá capacity của một máy                                         |
| **Latency**                             | Đặt server gần người dùng về địa lý → giảm round-trip time                         |
| **Elasticity**                          | Scale up/down theo demand, trả tiền theo usage                                     |
| **Specialized hardware**                | Object store dùng nhiều disk ít CPU; analytics dùng nhiều CPU/RAM; ML dùng GPU     |
| **Legal compliance**                    | Data residency laws: một số quốc gia yêu cầu data của công dân phải lưu trong nước |
| **Sustainability**                      | Chạy workload ở nơi/thời điểm có điện tái tạo, giảm carbon footprint               |

---

## 3. Nhược điểm của Distributed Systems

> **Nguyên tắc quan trọng:** Đừng vội chuyển sang distributed nếu single-node đủ dùng.

### Complexity

| Vấn đề                    | Chi tiết                                                                                                   |
| ------------------------- | ---------------------------------------------------------------------------------------------------------- |
| **Network failures**      | Request qua network có thể timeout, không biết server đã nhận chưa → retry không phải lúc nào cũng an toàn |
| **Latency**               | Network call chậm hơn nhiều so với function call trong cùng process                                        |
| **"More nodes ≠ Faster"** | Trong một số trường hợp, single-threaded program trên 1 máy có thể nhanh hơn cluster 100+ CPU cores        |
| **Troubleshooting khó**   | System slow → nguyên nhân ở đâu? Cần observability tools                                                   |
| **Data consistency**      | Khi mỗi service có database riêng → maintaining consistency cross-service là trách nhiệm của application   |

### Observability

Để debug distributed systems, cần thu thập và query data về execution:

- **High-level metrics** (latency, error rate, throughput)
- **Individual events/traces**
- **Tools:** OpenTelemetry, Zipkin, Jaeger

### Distributed Transactions

- Kỹ thuật đảm bảo consistency cross-service (xem Chapter 8)
- Thường **không dùng** trong microservices vì:
  - Đi ngược lại mục tiêu independence của services
  - Nhiều databases không support

---

## 4. Khi nào Single-Node là đủ?

Xu hướng gần đây: CPUs, RAM, disks ngày càng **lớn hơn, nhanh hơn, rẻ hơn**.

**Single-node databases mạnh:** DuckDB, SQLite, KùzuDB  
→ Nhiều workloads có thể chạy tốt trên một node (xem thêm Chapter 4).

> **Rule of thumb:** Bắt đầu với single-node. Chỉ scale out khi thực sự cần.

---

## 5. Microservices và Serverless

### Microservices Architecture

**Gốc rễ:** Service-Oriented Architecture (SOA) → refined thành Microservices.

**Định nghĩa:** Mỗi service có một **mục đích rõ ràng**, expose API qua network, và có **một team** chịu trách nhiệm.

#### Ưu điểm

- Update độc lập → giảm coordination giữa teams
- Assign hardware resources phù hợp cho từng service
- Ẩn implementation detail → tự do thay đổi internals
- Mỗi service có database riêng (không share database → tránh coupling)

#### Nhược điểm

| Vấn đề                      | Chi tiết                                                                        |
| --------------------------- | ------------------------------------------------------------------------------- |
| **Testing phức tạp**        | Phải run tất cả dependent services để test một service                          |
| **Infrastructure overhead** | Mỗi service cần: deploy pipeline, load balancing, logging, monitoring, alerting |
| **API evolution khó**       | Client expect API có fields nhất định; thêm/xóa field có thể break clients      |
| **Phát hiện lỗi muộn**      | Breaking changes thường chỉ thấy khi deploy lên staging/production              |

**API standards giảm vấn đề evolution:** OpenAPI, gRPC (xem Chapter 5)

> **Quan trọng:** Microservices về cơ bản là giải pháp **kỹ thuật cho vấn đề con người** — cho phép các teams tiến độc lập không cần coordinate. Với team nhỏ, overhead của microservices thường không đáng.

#### Tại sao không share database giữa services?

```
Service A ──► Database AB ◄── Service B
                  ▲
              Database structure trở thành
              phần của public API → khó thay đổi
              + queries của A có thể ảnh hưởng
              performance của B
```

### Serverless / Function-as-a-Service (FaaS)

| Khía cạnh                  | Serverless                                                                                 |
| -------------------------- | ------------------------------------------------------------------------------------------ |
| **Quản lý infrastructure** | Cloud provider tự động allocate/free resources theo incoming requests                      |
| **Billing model**          | Trả theo thời gian code chạy thực, không phải theo instance time                           |
| **Tương tự**               | Cloud storage → metered billing cho disks; Serverless → metered billing cho code execution |
| **Hạn chế**                | Time limit trên function execution; slow cold starts; hạn chế runtime environments         |

**Serverless trong data systems:** BigQuery, Kafka offerings → autoscale + billing by usage → cũng gọi là "serverless".

---

## 6. Cloud Computing vs Supercomputing (HPC)

### So sánh

| Khía cạnh            | Cloud Computing                                              | Supercomputing (HPC)                                        |
| -------------------- | ------------------------------------------------------------ | ----------------------------------------------------------- |
| **Use cases**        | Online services, business data, serving users                | Scientific computing (weather, climate, molecular dynamics) |
| **Failure handling** | Tiếp tục chạy với redundancy                                 | Stop cluster, repair node, restart từ checkpoint            |
| **Network**          | IP/Ethernet, Clos topologies, high bisection bandwidth       | Specialized (multidimensional meshes, toruses), RDMA        |
| **Trust model**      | Mutually untrusting organizations → VM isolation, encryption | High trust among users                                      |
| **Geography**        | Multi-region                                                 | Tất cả nodes gần nhau                                       |
| **Availability**     | Continually available                                        | Batch jobs, can tolerate downtime                           |

**Khi nào overlap:** Large-scale analytical systems đôi khi dùng techniques của HPC.

---

## Tóm tắt phần này

```
Distributed System:
  + Fault tolerance, scalability, latency, elasticity
  + Multi-user inherent distribution
  - Complexity, network failures, harder debugging
  - Data consistency challenges

Microservices:
  + Team independence
  + Independent scaling & deployment
  - Infrastructure overhead
  - Testing complexity
  - API versioning challenges

Serverless:
  + Metered billing, auto-scaling
  - Cold starts, execution limits

Rule: Single-node first. Distribute only when needed.
```

**Chapters liên quan:**

- Chapter 9: Chi tiết về vấn đề của distributed systems (network, clocks, consensus)
- Chapter 8: Distributed transactions
- Chapter 2: Scalability, reliability
