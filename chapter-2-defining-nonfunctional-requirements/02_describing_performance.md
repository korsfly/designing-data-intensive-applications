# Describing Performance

---

## 1. Hai loại metric chính

| Metric            | Định nghĩa                                                   | Đơn vị              |
| ----------------- | ------------------------------------------------------------ | ------------------- |
| **Response time** | Thời gian từ lúc user gửi request đến lúc nhận được response | seconds / ms / μs   |
| **Throughput**    | Số request/giây, hoặc lượng data/giây mà hệ thống xử lý được | "things per second" |

### Trong case study social network

| Loại          | Ví dụ                                                                       |
| ------------- | --------------------------------------------------------------------------- |
| Throughput    | "posts per second", "timeline writes per second"                            |
| Response time | "thời gian load home timeline", "thời gian post được deliver đến followers" |

---

## 2. Quan hệ giữa Throughput và Response Time

```
Response Time
     │                                    ╱
     │                                  ╱
     │                               ╱
     │                            ╱
     │                       ╱
     │              ╱  ╱
     │  ───────╱
     └──────────────────────────────────────► Throughput
                                          (gần đến max capacity)
```

**Nguyên nhân:** **Queueing**. Khi request đến trong lúc CPU đang xử lý request trước, request mới phải đợi. Khi throughput tiến gần max capacity của hardware, queueing delay tăng **rất nhanh**.

### Khi hệ thống Overloaded không thể tự hồi phục (Metastable Failure)

```
Queue dài → response time tăng cao → client TIMEOUT → client RESEND request
                                                              │
                                                              ▼
                                            Rate request TĂNG THÊM (retry storm)
                                                              │
                                                              ▼
                                              Hệ thống NGÀY CÀNG quá tải
```

> Đây gọi là **metastable failure** — hệ thống có thể vẫn ở trạng thái overload **dù load gốc đã giảm**, cần reboot/reset để khôi phục.

### Các kỹ thuật phòng ngừa

| Kỹ thuật                   | Mô tả                                                            |
| -------------------------- | ---------------------------------------------------------------- |
| **Exponential backoff**    | Tăng và randomize thời gian giữa các lần retry                   |
| **Circuit breaker**        | Tạm dừng gửi request đến service đang lỗi/timeout                |
| **Token bucket algorithm** | Giới hạn rate gửi request                                        |
| **Load shedding**          | Server tự phát hiện gần overload và **chủ động từ chối** request |
| **Backpressure**           | Server gửi response yêu cầu client **chậm lại**                  |

> **Ghi nhớ:** Response time là điều **user quan tâm nhất**. Throughput quyết định **lượng compute resources cần** (và do đó **chi phí**). Hệ thống được gọi là **scalable** nếu throughput tối đa có thể tăng đáng kể bằng cách thêm computing resources.

---

## 3. Latency và Response Time — Phân biệt chính xác

> **Lưu ý:** "Latency" và "response time" đôi khi dùng thay thế nhau, nhưng sách này dùng theo nghĩa **cụ thể**.

| Thuật ngữ           | Định nghĩa                                                                          |
| ------------------- | ----------------------------------------------------------------------------------- |
| **Response time**   | Tổng thời gian client thấy được — bao gồm TẤT CẢ delay trong hệ thống               |
| **Service time**    | Thời gian service **đang xử lý chủ động** request                                   |
| **Queueing delay**  | Thời gian chờ (chờ CPU available, chờ buffer gửi qua network,...)                   |
| **Latency**         | Thuật ngữ chung cho thời gian request **không được xử lý chủ động** (đang "latent") |
| **Network latency** | Thời gian cụ thể request/response di chuyển qua network                             |

### Sơ đồ minh họa (mô tả lại Figure 2-4)

```
Client                    Server
  │                          │
  │──── network latency ───►│
  │                          │  (queueing delay - chờ CPU)
  │                          │──┐
  │                          │  │ service time (xử lý thực)
  │                          │◄─┘
  │◄─── network latency ─────│
  │                          │

Response time = network latency (đi) + queueing delay + service time + network latency (về)
```

### Nguyên nhân gây biến động response time (random delays)

- Context switch sang background process
- Mất network packet → TCP retransmission
- Garbage collection pause
- Page fault → đọc từ disk
- Mechanical vibrations trong server rack (!)

### Head-of-Line Blocking

> Server chỉ xử lý được một số lượng nhỏ request song song (giới hạn bởi số CPU cores). Một vài request CHẬM có thể **chặn** việc xử lý các request khác phía sau trong queue — gọi là **head-of-line blocking**.

```
Queue: [Request CHẬM] [Request nhanh] [Request nhanh] [Request nhanh]
                │
                ▼
       Các request nhanh phía sau phải ĐỢI request chậm xử lý xong
       → Dù service time của chúng nhanh, response time vẫn CHẬM
```

> **Quan trọng:** Queueing delay **không phải** một phần của service time → cần đo response time **ở phía client**, không chỉ ở phía server.

---

## 4. Average, Median, và Percentiles

### Tại sao không dùng Average (Mean)?

- Mean = tổng tất cả response times / số lượng request
- Mean **hữu ích** để ước tính throughput limits
- Nhưng mean **KHÔNG tốt** để biết "trải nghiệm thông thường" — nó không cho biết bao nhiêu user thực sự trải qua delay đó

### Percentiles — Cách đo tốt hơn

| Percentile       | Ý nghĩa                                          |
| ---------------- | ------------------------------------------------ |
| **Median (p50)** | 50% requests nhanh hơn giá trị này, 50% chậm hơn |
| **p95**          | 95% requests nhanh hơn; 5% chậm hơn hoặc bằng    |
| **p99**          | 99% requests nhanh hơn                           |
| **p999 (p99.9)** | 99.9% requests nhanh hơn                         |

```
Sort response times từ nhanh → chậm:
[10ms, 15ms, 18ms, 20ms, ..., 200ms (median), ..., 1.5s (p95), ..., 5s (p999)]
```

### Tail Latencies

> **Tail latency:** Các percentile cao (p95, p99, p999) — quan trọng vì **ảnh hưởng trực tiếp** đến trải nghiệm user.

**Ví dụ Amazon:** Amazon dùng **p99.9** (chỉ 1/1000 requests) để định nghĩa response time requirement. Lý do: khách hàng có response time chậm nhất thường là khách hàng **có nhiều data nhất** (đã mua nhiều) — chính là **khách hàng giá trị nhất**.

> Amazon từng cân nhắc dùng p99.99 nhưng thấy quá đắt và lợi ích không nhiều — **giảm response time ở percentile rất cao rất khó** vì dễ bị ảnh hưởng bởi các yếu tố ngẫu nhiên nằm ngoài kiểm soát, và lợi ích giảm dần.

### Nghiên cứu về ảnh hưởng của Latency lên User Behavior

> **Lưu ý:** Các số liệu được trích dẫn nhiều **không đáng tin cậy hoàn toàn** hoặc có mâu thuẫn:

| Nghiên cứu    | Kết quả                                                                                                                                              |
| ------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| Google (2006) | Chậm từ 400ms→900ms làm giảm 20% traffic/revenue                                                                                                     |
| Google (2009) | Tăng 400ms latency chỉ giảm 0.6% số lượng search/ngày                                                                                                |
| Bing (2009)   | Tăng 2s load time giảm 4.3% ad revenue                                                                                                               |
| Akamai        | Tăng 100ms response time giảm tới 7% conversion rate — nhưng có vấn đề: trang load NHANH NHẤT (404 error) cũng có conversion thấp → kết quả nghi vấn |
| Yahoo         | Fast search có 20-30% nhiều click hơn khi khác biệt ≥1.25s, có kiểm soát chất lượng kết quả search                                                   |

→ **Kết luận:** Dữ liệu định lượng về ảnh hưởng latency lên hành vi user **khó lấy được đáng tin cậy**.

---

## 5. Sử dụng Response Time Metrics trong thực tế

### Tail Latency Amplification

```
End-user request cần GỌI NHIỀU backend service song song
   │
   ├──► Backend call 1 (nhanh)
   ├──► Backend call 2 (CHẬM ← chỉ cần 1 cái chậm)
   └──► Backend call 3 (nhanh)
        │
        ▼
   Toàn bộ end-user request phải ĐỢI call CHẬM NHẤT
```

> Dù chỉ một % nhỏ backend calls chậm, nếu một end-user request cần gọi NHIỀU backend calls, **xác suất gặp ít nhất 1 call chậm tăng lên** → tỷ lệ end-user request bị chậm cao hơn dự kiến. Hiện tượng này gọi là **tail latency amplification**.

### SLO và SLA

| Thuật ngữ                         | Định nghĩa                                                                                  |
| --------------------------------- | ------------------------------------------------------------------------------------------- |
| **SLO** (Service Level Objective) | Mục tiêu performance/availability (ví dụ: median <200ms, p99 <1s, 99.9% requests không lỗi) |
| **SLA** (Service Level Agreement) | Hợp đồng quy định **điều gì xảy ra** nếu SLO không đạt (ví dụ: refund cho customer)         |

> Trong thực tế, định nghĩa **availability metrics tốt** cho SLO/SLA **không đơn giản**.

---

## 6. Tính Percentile trong thực tế (Computing Percentiles)

### Vấn đề

Muốn hiển thị percentile trên monitoring dashboard, cần tính **liên tục** (ví dụ: rolling window 10 phút, tính lại mỗi phút).

| Cách tiếp cận                                 | Ưu/Nhược                                             |
| --------------------------------------------- | ---------------------------------------------------- |
| Lưu list tất cả response times, sort mỗi phút | Đơn giản nhưng **không hiệu quả** ở scale lớn        |
| Dùng algorithm approximation                  | Tính percentile gần đúng với chi phí CPU/memory thấp |

**Thư viện open source phổ biến:** HdrHistogram, t-digest, OpenHistogram, DDSketch

> **Cảnh báo quan trọng:** **Average các percentile lại với nhau là VÔ NGHĨA về mặt toán học** (ví dụ: trung bình p99 của nhiều máy không cho ra p99 đúng của tổng thể). Cách đúng để **aggregate** response time data là **cộng các histogram lại**.

---

## Tóm tắt phần này

```
Performance Metrics:
  Response Time  ←─ User quan tâm nhất
  Throughput     ←─ Quyết định cost / resource cần thiết

Response Time = Network latency + Queueing delay + Service time

Đo response time:
  ❌ Average/Mean (không phản ánh trải nghiệm thực)
  ✓ Percentiles (p50, p95, p99, p999) - đặc biệt tail latencies

Hiện tượng cần biết:
  - Head-of-line blocking
  - Tail latency amplification
  - Metastable failure (retry storm)

Công cụ kiểm soát:
  - Exponential backoff, Circuit breaker, Load shedding, Backpressure

Hợp đồng performance:
  - SLO (mục tiêu) → SLA (hợp đồng + hậu quả nếu không đạt)
```
