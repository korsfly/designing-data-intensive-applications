# Reliability and Fault Tolerance

---

## 1. Reliability là gì?

### Kỳ vọng trực quan về một hệ thống "reliable"

- Application thực hiện đúng chức năng mà user mong đợi
- Application chịu được lỗi của user hoặc cách sử dụng bất ngờ
- Performance đủ tốt cho use case dưới load/data volume kỳ vọng
- Hệ thống ngăn chặn truy cập trái phép và lạm dụng

> **Định nghĩa Reliability:** "Tiếp tục hoạt động đúng, **ngay cả khi có sự cố xảy ra**" (continuing to work correctly, even when things go wrong).

---

## 2. Phân biệt Fault và Failure

| Thuật ngữ   | Định nghĩa                                                                                                        |
| ----------- | ----------------------------------------------------------------------------------------------------------------- |
| **Fault**   | Một **bộ phận cụ thể** của hệ thống ngừng hoạt động đúng (1 disk hỏng, 1 máy crash, 1 external service bị outage) |
| **Failure** | **TOÀN BỘ hệ thống** ngừng cung cấp dịch vụ cần thiết cho user (không đạt SLO)                                    |

### Tính tương đối của Fault/Failure

```
Hệ thống chỉ có 1 hard drive:
   Hard drive hỏng → Đó là FAULT của disk, NHƯNG cũng là FAILURE của hệ thống
   (vì hệ thống = chỉ 1 disk đó)

Hệ thống có NHIỀU hard drive (redundancy):
   1 hard drive hỏng → Chỉ là FAULT (từ góc nhìn hệ thống lớn)
   Hệ thống vẫn hoạt động nhờ có bản copy ở disk khác → KHÔNG PHẢI failure
```

→ **Bài học:** Fault và Failure là **cùng một sự việc**, chỉ khác **cấp độ quan sát** (level).

---

## 3. Fault Tolerance

> **Fault-tolerant system:** Hệ thống tiếp tục cung cấp dịch vụ cần thiết cho user **dù có một số fault xảy ra**.

> **Single Point of Failure (SPOF):** Nếu hệ thống KHÔNG thể chịu được lỗi ở một bộ phận nào đó, bộ phận đó được gọi là SPOF — vì fault ở đó sẽ leo thang thành failure của toàn hệ thống.

### Ví dụ từ Case Study (Chapter 2, Section 1)

```
Trong quá trình fan-out cập nhật materialized timeline:
   Máy đang xử lý CRASH → cần máy KHÁC tiếp quản
   → KHÔNG được miss post nào, KHÔNG được duplicate post nào
   → Gọi là "exactly-once semantics" (chi tiết ở Chapter 12)
```

### Giới hạn của Fault Tolerance

> Fault tolerance **luôn có giới hạn** — chỉ chịu được **một số lượng/loại fault nhất định**.

```
Ví dụ: Hệ thống có thể chịu được TỐI ĐA 2 hard disks hỏng đồng thời
       Hệ thống có thể chịu được TỐI ĐA 1/3 nodes crash

KHÔNG THỂ chịu được "bất kỳ số lượng fault nào":
   Nếu TẤT CẢ nodes crash → không thể làm gì được
   Nếu Trái Đất bị hố đen nuốt → cần hosting trong không gian (đùa, nhưng minh họa giới hạn thực tế)
```

### Fault Injection và Chaos Engineering

> **Fault injection:** Chủ động **kích hoạt fault** (ví dụ: random kill process không cảnh báo) để **kiểm tra** khả năng fault tolerance.

**Lý do hợp lý (counterintuitive):**

- Nhiều bug nghiêm trọng thực ra do **xử lý lỗi kém** (poor error handling)
- Chủ động gây fault → đảm bảo cơ chế fault-tolerance được **thực hành liên tục**
- Tăng độ tin tưởng rằng fault sẽ được xử lý đúng khi xảy ra **tự nhiên**

> **Chaos Engineering:** Một discipline nhằm tăng độ tin tưởng vào fault-tolerance mechanisms qua các thí nghiệm như chủ động gây fault.

### Khi nào nên Phòng ngừa (Prevention) hơn là Chịu đựng (Tolerance)?

```
Thông thường: ƯU TIÊN tolerating faults > preventing faults

NGOẠI LỆ: Security
   - Nếu attacker đã compromised hệ thống và lấy được sensitive data
   - → KHÔNG THỂ "undo" sự việc này → cần PREVENTION, không phải cure
```

---

## 4. Hardware Faults

### Số liệu thực tế về tỷ lệ hỏng hóc hardware

| Component                | Tỷ lệ hỏng                                                                           |
| ------------------------ | ------------------------------------------------------------------------------------ |
| **Magnetic hard drives** | 2%–5%/năm → cluster 10,000 disks: ~1 disk hỏng/ngày                                  |
| **SSDs**                 | 0.5%–1%/năm hỏng hoàn toàn; lỗi không sửa được xảy ra ~1 lần/năm/drive (cao hơn HDD) |
| **CPU core lỗi**         | ~1/1000 máy có CPU core tính SAI kết quả (do lỗi sản xuất)                           |
| **RAM corruption**       | Ngay cả với ECC memory, >1% máy gặp lỗi không sửa được/năm → thường gây crash        |
| **Cả datacenter**        | Có thể mất hoàn toàn (mất điện, lỗi network, cháy, lũ, động đất, **solar storm**)    |

> **Lưu ý:** Các sự kiện này hiếm với hệ thống nhỏ, nhưng với hệ thống **lớn**, hardware fault trở thành **một phần của vận hành bình thường**.

### Chịu đựng Hardware Faults qua Redundancy

| Kỹ thuật                      | Mô tả                                                   |
| ----------------------------- | ------------------------------------------------------- |
| **RAID**                      | Trải data trên nhiều disks → 1 disk hỏng không mất data |
| **Dual power supplies**       | Server có 2 nguồn điện                                  |
| **Hot-swappable CPUs**        | Thay CPU khi máy vẫn chạy                               |
| **Backup power (datacenter)** | Pin, máy phát điện diesel                               |

> **Redundancy hiệu quả nhất khi faults là ĐỘC LẬP** (independent) — nhưng thực tế có **correlation đáng kể** giữa các failures (ví dụ: cùng 1 rack, cùng datacenter).

### Từ Single Machine Redundancy đến Distributed Systems

```
Hardware Redundancy (trong 1 máy)
   → Tăng uptime của 1 máy
   → NHƯNG: vẫn có thể mất toàn bộ rack/datacenter

Distributed System (nhiều máy/datacenter)
   → Có thể chịu được MẤT TOÀN BỘ 1 datacenter
   → Cloud providers dùng "AVAILABILITY ZONES" để biết resource nào physically cùng vị trí
     (resource cùng chỗ → khả năng hỏng cùng lúc cao hơn)
```

**Lợi ích vận hành (Rolling Upgrade):**

```
Single-server system: cần PLANNED DOWNTIME để reboot (vd: patch OS)
Multi-node fault-tolerant: PATCH TỪNG NODE một, không ảnh hưởng users
   → gọi là ROLLING UPGRADE (chi tiết Chapter 5)
```

---

## 5. Software Faults

### Khác biệt căn bản với Hardware Faults

> **Hardware faults** thường **yếu tương quan** (weakly correlated) — disk này hỏng không có nghĩa disk khác cũng hỏng ngay.

> **Software faults** thường **TƯƠNG QUAN RẤT MẠNH** — vì nhiều node chạy CÙNG software → cùng bug.

### Các ví dụ thực tế về Software Faults

| Loại lỗi                           | Ví dụ                                                                       |
| ---------------------------------- | --------------------------------------------------------------------------- |
| **Bug đồng thời trên nhiều node**  | Leap second bug (30/6/2012) làm nhiều Java app trên Linux bị hang đồng loạt |
| **Firmware bug theo thời gian**    | Mọi SSD model nhất định hỏng đúng sau 32,768 giờ hoạt động (~4 năm)         |
| **Runaway process**                | Process dùng hết CPU/memory/disk/network/threads chung                      |
| **Service dependency degradation** | Service phụ thuộc bị chậm/unresponsive/trả response sai                     |
| **Emergent behavior**              | Tương tác giữa nhiều hệ thống tạo ra hành vi không xảy ra khi test riêng lẻ |
| **Cascading failures**             | Lỗi ở 1 component làm component khác bị overload, lan rộng                  |

### Tại sao Software Bugs khó dự đoán?

> Bug thường **"ngủ yên"** (dormant) lâu, chỉ bị kích hoạt bởi **tình huống bất thường**. Lúc đó mới phát hiện software đã **giả định sai** về môi trường — giả định ĐÚNG hầu hết thời gian nhưng cuối cùng SAI vì lý do nào đó.

### Giải pháp (không có "quick fix")

- Suy nghĩ kỹ về assumptions và interactions trong hệ thống
- Testing kỹ lưỡng
- Đảm bảo process isolation
- Cho phép process crash và restart
- Tránh feedback loop (vd: retry storm — xem phần Performance)
- Đo lường, monitor, phân tích hành vi hệ thống trong production

---

## 6. Humans và Reliability

### Con người: Nguồn gốc của nhiều lỗi

> Một nghiên cứu cho thấy: **configuration changes của operators** là nguyên nhân hàng đầu gây outage tại các internet service lớn, trong khi hardware fault chỉ chiếm **10%–25%** trường hợp.

### Tại sao đổ lỗi "Human Error" là phản tác dụng?

```
"Human Error" KHÔNG PHẢI nguyên nhân thực sự
   → Nó là TRIỆU CHỨNG của vấn đề trong hệ thống socio-technical
   → Con người đang CỐ GẮNG LÀM TỐT nhất công việc của họ
```

> Hệ thống phức tạp thường có **emergent behavior** — tương tác bất ngờ giữa các component dẫn đến lỗi.

### Kỹ thuật giảm thiểu tác động của Human Mistakes

| Kỹ thuật                                          | Mục đích                                                |
| ------------------------------------------------- | ------------------------------------------------------- |
| Thorough testing (handwritten + property testing) | Phát hiện lỗi trước production                          |
| Rollback mechanisms                               | Hoàn tác config change nhanh                            |
| Gradual rollouts                                  | Giảm impact khi deploy code mới                         |
| Detailed monitoring                               | Phát hiện sớm                                           |
| Observability tools                               | Debug nhanh trong production                            |
| Well-designed interfaces                          | "Encourage the right thing, discourage the wrong thing" |

### Vấn đề ưu tiên kinh doanh

```
Đầu tư vào resilience cần TIỀN và THỜI GIAN
   vs.
Đầu tư vào TÍNH NĂNG MỚI (revenue-generating)

→ Nhiều tổ chức chọn FEATURES hơn TESTING (hợp lý về kinh doanh)
→ Khi lỗi xảy ra: KHÔNG nên đổ lỗi cá nhân — vấn đề là ƯU TIÊN của tổ chức
```

### Blameless Postmortems

> **Văn hóa Blameless Postmortem:** Sau incident, người liên quan được khuyến khích chia sẻ **đầy đủ chi tiết** về việc đã xảy ra, **không sợ bị phạt** — để tổ chức học được cách phòng tránh vấn đề tương tự trong tương lai.

```
Postmortem KHÔNG nên kết luận đơn giản:
   ❌ "Bob nên cẩn thận hơn khi deploy"
   ❌ "Phải viết lại backend bằng Haskell"

NÊN:
   ✓ Học chi tiết cách hệ thống socio-technical hoạt động
     từ góc nhìn người làm việc với nó MỖI NGÀY
   ✓ Cải thiện dựa trên feedback đó
```

---

## 7. Reliability quan trọng đến mức nào?

### Không chỉ áp dụng cho hệ thống "critical"

> Reliability không chỉ dành cho nhà máy điện hạt nhân hay air traffic control — ứng dụng đời thường cũng cần hoạt động đáng tin cậy.

### Ví dụ tác động thực tế

| Ví dụ                                           | Tác động                                                      |
| ----------------------------------------------- | ------------------------------------------------------------- |
| Bug trong business application                  | Mất productivity, rủi ro pháp lý nếu số liệu báo cáo sai      |
| Outage ecommerce site                           | Mất doanh thu, tổn hại reputation                             |
| **Mất dữ liệu vĩnh viễn** (ví dụ: ảnh gia đình) | **Tai họa**, không thể chấp nhận dù chỉ outage tạm thời là OK |

### Case study: Post Office Horizon Scandal (Anh)

> Từ 1999–2019, **hàng trăm người** quản lý chi nhánh Post Office tại Anh bị **kết tội trộm cắp/lừa đảo** vì software kế toán báo sai thiếu hụt. Cuối cùng phát hiện ra **nhiều lỗi này do bug software**, dẫn đến việc nhiều bản án bị **lật lại**.

**Nguyên nhân pháp lý sâu xa:**

```
Luật pháp Anh GIẢ ĐỊNH computers hoạt động ĐÚNG
   (và do đó evidence từ computer được coi là reliable)
   TRỪ KHI có bằng chứng ngược lại

→ Có thể là MIỄN OAN SAI lớn nhất trong lịch sử pháp lý Anh
→ Nhiều người: phá sản, thậm chí TỰ TỬ vì bị kết án sai
```

> **Bài học:** "Software có thể có bug" không phải lời an ủi cho người bị **kết án oan, phá sản, hoặc tự tử** vì hệ thống không đáng tin cậy.

### Khi nào có thể đánh đổi Reliability?

```
Một số tình huống có thể HY SINH reliability để giảm chi phí phát triển
   (ví dụ: prototype cho thị trường chưa được kiểm chứng)

→ NHƯNG: phải Ý THỨC RÕ khi đang "cắt góc" (cutting corners)
         và hậu quả tiềm ẩn
```

---

## Tóm tắt phần này

```
Fault vs Failure:
  Fault = lỗi ở MỘT bộ phận (relative đến cấp hệ thống quan sát)
  Failure = TOÀN BỘ hệ thống ngừng hoạt động đúng

Fault Tolerance:
  Luôn có GIỚI HẠN (tolerate được N fault, không phải vô hạn)
  Fault Injection / Chaos Engineering giúp TĂNG độ tin cậy

Hardware Faults:
  Yếu tương quan → Redundancy (RAID, dual PSU,...) hiệu quả
  Ở SCALE LỚN: hardware fault là "bình thường", cần distributed system

Software Faults:
  TƯƠNG QUAN MẠNH (cùng bug trên nhiều node)
  Khó dự đoán, không có "quick fix"

Human & Reliability:
  "Human error" là TRIỆU CHỨNG, không phải NGUYÊN NHÂN
  Blameless Postmortem → học hỏi, không đổ lỗi

Reliability quan trọng:
  Mất data vĩnh viễn = TAI HỌA
  Post Office Horizon Scandal = ví dụ thực tế về hậu quả nghiêm trọng
```
