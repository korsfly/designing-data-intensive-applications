# Cloud vs Self-Hosting

---

## 1. Khung tư duy: Build hay Buy?

**Nguyên tắc chung:** Những gì là **core competency / competitive advantage** của tổ chức → làm in-house. Những gì là non-core, routine → để vendor làm.

Ví dụ cực đoan: Hầu hết công ty không tự sản xuất CPU — họ mua từ nhà sản xuất chip.

### Hai chiều quyết định

1. **Ai viết software?** (bạn hay vendor?)
2. **Ai deploy và vận hành?** (bạn hay vendor?)

```
Bespoke in-house ◄────────────────────────────► Cloud SaaS
(tự viết, tự chạy)                              (vendor viết, vendor chạy)
                    ▲
                    │ Self-hosted open source
                    │ (vendor viết, bạn chạy)
                    │ — ví dụ: download MySQL, cài trên server của mình
```

---

## 2. Ưu và Nhược điểm của Cloud Services

### Ưu điểm

| Ưu điểm                                      | Khi nào đặc biệt có giá trị                                    |
| -------------------------------------------- | -------------------------------------------------------------- |
| **Tiết kiệm thời gian, tăng tốc độ**         | Khi bạn không có kinh nghiệm deploy/operate hệ thống cần thiết |
| **Elastic scaling**                          | Khi load **biến động nhiều** (ví dụ analytical workloads)      |
| **Operational expertise**                    | Provider có kinh nghiệm vận hành từ nhiều khách hàng           |
| **Không cần capacity planning truyền thống** | Metered billing: chỉ trả cho tài nguyên dùng thực              |

> **Ví dụ analytics:** Query lớn cần nhiều compute trong vài giây, sau đó tài nguyên nằm chờ. Cloud cho phép trả tiền theo query, không phải theo machine.

### Nhược điểm / Rủi ro

| Nhược điểm                | Giải thích                                               |
| ------------------------- | -------------------------------------------------------- |
| **Mất kiểm soát**         | Không implement được feature thiếu; không debug sâu được |
| **Service downtime**      | Chỉ có thể chờ vendor fix                                |
| **Khó debug**             | Không có access vào server logs, OS metrics              |
| **Vendor lock-in**        | Nhiều cloud services không có standard APIs; khó migrate |
| **Geopolitical risk**     | Sanctions có thể làm mất quyền truy cập dịch vụ          |
| **Security & compliance** | Phải tin tưởng cloud provider bảo vệ data                |

### Khi nào self-hosting có thể rẻ hơn?

- Bạn **đã có kinh nghiệm** deploy & operate hệ thống đó
- Load **predictable** (không biến động nhiều) → provisioning hiệu quả
- Scale đủ lớn để justify đầu tư hardware

---

## 3. Cloud Native System Architecture

### Cloud Native là gì?

Kiến trúc được **thiết kế từ đầu** để tận dụng cloud services, không chỉ chạy software cũ trên cloud VM.

**Ưu điểm đã được chứng minh:**

- Performance tốt hơn trên cùng hardware
- Recovery from failures nhanh hơn
- Scale computing resources nhanh theo load
- Hỗ trợ datasets lớn hơn

### So sánh: Self-hosted vs Cloud Native

| Category             | Self-hosted                 | Cloud Native                                           |
| -------------------- | --------------------------- | ------------------------------------------------------ |
| **Operational/OLTP** | MySQL, PostgreSQL, MongoDB  | AWS Aurora, Azure SQL Hyperscale, Google Cloud Spanner |
| **Analytical/OLAP**  | Teradata, ClickHouse, Spark | Snowflake, Google BigQuery, Azure Synapse Analytics    |

### Layering of Cloud Services

Cloud native không chỉ dùng VM như traditional computing. Nó **xây dựng trên các cloud services cấp thấp hơn**:

```
Ứng dụng của bạn
      │
      ├── Snowflake (analytical DB)
      │         └── Amazon S3 (object storage)
      │
      └── Dịch vụ khác
               └── Các cloud primitives (compute, network, storage)
```

**Nguyên tắc:** Abstractions cấp cao hơn phù hợp với use case cụ thể → dùng nếu match. Nếu không match → tự build từ lower-level components.

---

## 4. Separation of Storage and Compute

### Traditional Computing

- Storage (disk) và Compute (CPU/RAM) **trên cùng một máy**
- RAID để redundancy cho disks
- Mất machine = mất access vào data

### Cloud: Tách Storage và Compute

**Vấn đề với local disks trong cloud:**

- Nếu instance fail hoặc bị replace → local disk mất theo
- Virtual disks (EBS, Azure managed disks) = cloud service giả lập disk qua network → overhead + nhạy cảm với network glitches

**Giải pháp cloud native:**

- **Object storage** (S3, Azure Blob, Cloudflare R2): file lớn (KB → GB), ẩn physical machines, tự động replicate, không lo hết disk
- **Dedicated storage services:** Tối ưu cho workload cụ thể

```
Traditional:  [CPU + RAM + Disk]  ← tất cả trên 1 máy

Cloud Native: [Compute nodes]  ←──network──►  [Storage services (S3,...)]
              (có thể scale riêng)             (có thể scale riêng)
```

**Disaggregation:** Phân tách trách nhiệm storage và compute → mỗi phần có thể scale độc lập.

**Multitenancy:** Nhiều customers share cùng hardware → tận dụng tài nguyên tốt hơn, nhưng cần engineering cẩn thận để tránh "noisy neighbor" ảnh hưởng performance/security.

---

## 5. Operations in the Cloud Era

### Lịch sử: Từ DBA/SysAdmin đến DevOps/SRE

| Era          | Vai trò                         | Tập trung                                                   |
| ------------ | ------------------------------- | ----------------------------------------------------------- |
| Truyền thống | DBA, SysAdmin                   | Quản lý từng machine (disk space, provisioning, OS patches) |
| DevOps       | Dev + Ops shared responsibility | Automation, CI/CD, reliability                              |
| Cloud era    | SRE (Google) + Cloud Operations | Service-level thinking, cost optimization                   |

### DevOps/SRE Philosophy

- **Automation first:** Repeatable processes, không làm manual
- **Ephemeral VMs:** Không dựa vào long-running servers
- **Frequent updates:** Deploy thường xuyên
- **Learning from incidents:** Post-mortems
- **Knowledge preservation:** Dù người đến người đi

### Operations trong thời Cloud

Thay vì quản lý machines → quản lý **services**:

| Traditional                              | Cloud Era                                     |
| ---------------------------------------- | --------------------------------------------- |
| Capacity planning (mua bao nhiêu disks?) | Financial planning (chi bao nhiêu cho cloud?) |
| Performance optimization                 | Cost optimization                             |
| Provisioning machines                    | Choosing the right service                    |
| Moving services between machines         | Integrating services with each other          |

**Thách thức mới:**

- Thiếu standards cho service integration → nhiều manual effort
- Vẫn cần tự quản lý: application security, service interactions, monitoring, troubleshooting
- Cloud resource limits/quotas: phải biết và plan trước khi hit limits

> **Kết luận:** Cloud thay đổi _loại_ operations, không _giảm_ nhu cầu operations.

---

## Tóm tắt phần này

```
Cloud vs Self-Hosting Trade-offs:

                  ┌─ Cost: phụ thuộc vào skills + predictability of load
                  ├─ Control: Self-hosting thắng
                  ├─ Scalability: Cloud thắng (elastic)
Cloud Service ────┤─ Expertise: Cloud provider có lợi thế
                  ├─ Vendor lock-in: Rủi ro
                  └─ Compliance: Phức tạp hơn

Cloud Native Architecture:
- Tách storage và compute
- Xây trên các cloud primitives (S3, etc.)
- Multitenancy
- Disaggregation

DevOps/SRE:
- Automation, ephemeral, frequent updates
- Operations = service management, not machine management
```
