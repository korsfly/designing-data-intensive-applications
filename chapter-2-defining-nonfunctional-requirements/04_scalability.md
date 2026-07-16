# Scalability

---

## 1. Scalability là gì?

> **Scalability:** Khả năng của hệ thống để **đối phó với load tăng lên**.

### Phản bác câu nói phổ biến

> _"Bạn không phải Google hay Amazon. Đừng lo về scale, cứ dùng relational database."_

**Khi nào câu này ĐÚNG:**

```
Nếu bạn xây dựng product MỚI, ít users, ở startup
   → Mục tiêu kỹ thuật quan trọng nhất: GIỮ hệ thống ĐƠN GIẢN và LINH HOẠT
   → Dễ thay đổi feature khi hiểu thêm về nhu cầu khách hàng
   → Lo lắng về scale GIẢ ĐỊNH trong tương lai = COUNTERPRODUCTIVE
   → Đầu tư scalability sớm: TỐT NHẤT là lãng phí, TỆ NHẤT là khóa bạn vào
     thiết kế KHÔNG LINH HOẠT
```

### Scalability KHÔNG phải nhãn một chiều

> Nói "X có scalable" hay "Y không scale" là **VÔ NGHĨA**. Thay vào đó cần hỏi:

- Nếu hệ thống tăng theo cách CỤ THỂ nào, ta có lựa chọn gì để đối phó?
- Làm sao thêm computing resources để xử lý load thêm?
- Dựa trên dự báo tăng trưởng hiện tại, KHI NÀO hệ thống hiện tại sẽ hit limit?

> **Nguyên tắc:** Khi product thành công và load tăng, bạn sẽ học được **bottleneck nằm ở đâu** và **chiều nào cần scale**. Lúc đó mới là thời điểm lo về scalability techniques.

---

## 2. Hiểu về Load (Understanding Load)

### Bước 1: Hiểu Load hiện tại

```
Cần hiểu RÕ load hiện tại TRƯỚC khi trả lời câu hỏi tăng trưởng
   ("Nếu load gấp đôi thì sao?")
```

**Các loại metric:**

- **Throughput:** requests/giây, GB data mới/ngày, số checkout/giờ
- **Peak của biến động:** số user online đồng thời (case study social network)

### Đặc điểm thống kê khác ảnh hưởng đến yêu cầu Scalability

| Yếu tố                        | Ví dụ                         |
| ----------------------------- | ----------------------------- |
| Tỷ lệ đọc/ghi                 | Read/write ratio của database |
| Hit rate của cache            | % request được serve từ cache |
| Số lượng data items/user      | Followers trong case study    |
| Average case hay extreme case | Tùy thuộc application cụ thể  |

### Hai cách phân tích khi Load tăng

```
Cách 1: GIỮ resources cố định (CPU, RAM, network bandwidth,...)
        → Tăng load theo cách nào đó → Performance bị ảnh hưởng RA SAO?

Cách 2: GIỮ performance cố định
        → Tăng load theo cách nào đó → Cần tăng resources THÊM BAO NHIÊU?
```

> **Mục tiêu thường gặp:** Giữ performance trong yêu cầu SLA, đồng thời **minimize cost** vận hành.

### Linear Scalability

```
Tăng GẤP ĐÔI resources → xử lý được GẤP ĐÔI load, performance GIỮ NGUYÊN
   = LINEAR SCALABILITY (tốt!)

Hiếm khi: GẤP ĐÔI load chỉ cần TĂNG ÍT HƠN gấp đôi resources
   (nhờ economies of scale hoặc phân bổ peak load tốt hơn)

NHIỀU KHẢ NĂNG HƠN: Cost tăng NHANH HƠN linear
   (vd: xử lý 1 write request có thể tốn nhiều công hơn nếu data lớn hơn,
   dù size của request giống nhau)
```

---

## 3. Shared-Memory, Shared-Disk, và Shared-Nothing Architectures

### Cách đơn giản nhất: Vertical Scaling (Scaling Up)

> Chuyển sang máy mạnh hơn (nhiều CPU cores, RAM, disk hơn).

```
Shared-Memory Architecture:
   - Dùng nhiều processes/threads TRÊN 1 máy
   - Tất cả threads CÙNG process truy cập CÙNG RAM
   - Vấn đề: Chi phí tăng NHANH HƠN linear
     (máy 2x resources thường đắt HƠN 2x giá máy bình thường)
   - Bottleneck: máy mạnh hơn KHÔNG chắc xử lý được gấp đôi load thực tế
```

### Shared-Disk Architecture

```
Nhiều máy (CPU + RAM độc lập) NHƯNG chia sẻ MỘT mảng disk
   qua network nhanh (NAS - Network Attached Storage, hoặc SAN)

→ Truyền thống dùng cho on-premises data warehousing
→ Hạn chế: CONTENTION và OVERHEAD của LOCKING giới hạn scalability
```

### Shared-Nothing Architecture (Horizontal Scaling / Scaling Out)

```
Distributed system: NHIỀU NODES
   Mỗi node có CPU + RAM + Disk RIÊNG
   Coordination giữa nodes: ở SOFTWARE LEVEL, qua network thông thường
```

**Ưu điểm:**

- Có khả năng scale **linear**
- Có thể dùng hardware với **price/performance ratio tốt nhất** (đặc biệt trên cloud)
- Dễ điều chỉnh resources theo load tăng/giảm
- Fault tolerance tốt hơn (phân bố trên nhiều datacenter/region)

**Nhược điểm:**

- Cần **explicit sharding** (Chapter 7)
- Gánh chịu **toàn bộ complexity** của distributed systems (Chapter 9)

### Mô hình lai trong Cloud Native (Separation of Storage and Compute)

```
Một số cloud native database:
   - Dùng SEPARATE SERVICE cho storage và transaction execution
   - Nhiều compute nodes CHIA SẺ truy cập đến cùng storage service

   → Giống Shared-Disk NHƯNG TRÁNH được vấn đề scalability cũ
   → Storage service expose API CHUYÊN BIỆT (không phải filesystem/block device chung)
     được thiết kế riêng cho nhu cầu của database
```

### Tổng hợp 3 kiến trúc

| Kiến trúc                       | Compute                  | Storage                    | Coordination                  |
| ------------------------------- | ------------------------ | -------------------------- | ----------------------------- |
| **Shared-Memory** (Vertical)    | 1 máy, nhiều threads     | Cùng RAM                   | Trong process                 |
| **Shared-Disk**                 | Nhiều máy, CPU/RAM riêng | Disk array CHUNG (NAS/SAN) | Qua network nhanh, có locking |
| **Shared-Nothing** (Horizontal) | Nhiều node độc lập       | Disk riêng mỗi node        | Software level, qua network   |

---

## 4. Nguyên tắc cho Scalability

### Không có "Magic Scaling Sauce"

> Kiến trúc của hệ thống ở **large scale** thường **rất specific** với application — **không có** kiến trúc scalable chung cho mọi trường hợp.

```
Ví dụ minh họa: Cùng throughput data (100 MB/giây), nhưng kiến trúc khác nhau hoàn toàn:
   - Hệ thống A: 100,000 request/giây, mỗi request 1 kB
   - Hệ thống B: 3 request/phút, mỗi request 2 GB
```

### Phải Re-think Architecture mỗi Order of Magnitude

```
Kiến trúc PHÙ HỢP cho 1 mức load → KHÔNG CHẮC xử lý được 10× load đó

→ Service đang phát triển NHANH: thường phải RETHINK architecture
  mỗi khi load tăng 1 ORDER OF MAGNITUDE (10x)

→ KHÔNG NÊN plan scaling needs xa hơn 1 order of magnitude vào tương lai
  (vì nhu cầu application thường thay đổi)
```

### Nguyên tắc 1: Chia nhỏ hệ thống thành các thành phần độc lập

```
Nguyên tắc nền tảng của:
   - Microservices
   - Sharding (Chapter 7)
   - Stream Processing (Chapter 12)
   - Shared-Nothing Architectures

Thách thức: Biết VẠCH RANH GIỚI ở đâu
   (cái gì nên đi CÙNG NHAU, cái gì nên TÁCH RA)
```

### Nguyên tắc 2: Đừng làm phức tạp hơn cần thiết

```
Nếu single-machine database làm được việc → ƯU TIÊN nó
   hơn distributed setup phức tạp

Autoscaling: "cool" nhưng nếu load PREDICTABLE,
   manually scaled system có thể ÍT BẤT NGỜ VẬN HÀNH hơn

Hệ thống 5 services ĐƠN GIẢN HƠN hệ thống 50 services

→ Good architectures = MIX THỰC TẾ (pragmatic) của nhiều approaches
```

---

## Tóm tắt phần này

```
Scalability = khả năng đối phó với LOAD TĂNG (không phải nhãn 1 chiều)

Quy trình tư duy:
  1. Hiểu LOAD hiện tại (throughput, peak, read/write ratio,...)
  2. Phân tích: Giữ resource cố định → performance thế nào?
                Giữ performance cố định → cần resource thêm bao nhiêu?
  3. Mục tiêu: Linear scalability (lý tưởng), trong giới hạn SLA, minimize cost

3 kiến trúc cơ bản:
  Shared-Memory (Vertical)  → Đơn giản, chi phí tăng nhanh hơn linear
  Shared-Disk               → Hạn chế bởi locking/contention
  Shared-Nothing (Horizontal) → Scale tốt nhất, nhưng phức tạp (sharding, distributed)

Nguyên tắc:
  - KHÔNG có magic scaling sauce — kiến trúc rất specific với use case
  - Rethink architecture mỗi 10x load
  - Chia nhỏ thành components độc lập
  - Đừng phức tạp hóa khi không cần thiết
```
