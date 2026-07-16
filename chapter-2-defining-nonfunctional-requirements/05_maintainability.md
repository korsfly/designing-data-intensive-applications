# Maintainability

---

## 1. Tại sao Maintainability quan trọng?

> Software **không hao mòn** vật lý như máy móc — nhưng requirements thay đổi liên tục, môi trường (dependencies, platform) thay đổi, và luôn có bug cần fix.

### Sự thật về chi phí Software

> **Đa số chi phí của software KHÔNG nằm ở phát triển ban đầu, mà ở MAINTENANCE liên tục:**

- Fix bugs
- Giữ hệ thống vận hành
- Điều tra failures
- Thích ứng với platform mới
- Sửa đổi cho use case mới
- Trả "technical debt"
- Thêm feature mới

### Maintenance phức tạp với Legacy Systems

```
Hệ thống chạy thành công LÂU DÀI
   → Có thể dùng technology LẠC HẬU (mainframes, COBOL,...)
   → Institutional knowledge VỀ THIẾT KẾ có thể đã MẤT (người rời tổ chức)
   → Phải SỬA lỗi của người khác

→ Maintenance là VẤN ĐỀ CON NGƯỜI, không kém gì vấn đề kỹ thuật
```

> **Insight quan trọng:** Mọi hệ thống ta xây dựng hôm nay, nếu **đủ giá trị để tồn tại lâu**, MỘT NGÀY sẽ trở thành **legacy system**. Để giảm đau khổ cho thế hệ tương lai phải maintain code của ta, ta nên **thiết kế với maintenance trong đầu** từ bây giờ.

---

## 2. Ba nguyên tắc chính của Maintainability

```
┌─────────────────────────────────────────────┐
│              MAINTAINABILITY                 │
│                                               │
│  1. OPERABILITY                              │
│     → Dễ cho tổ chức GIỮ hệ thống chạy trơn  │
│                                               │
│  2. SIMPLICITY                               │
│     → Dễ cho engineer MỚI HIỂU hệ thống      │
│                                               │
│  3. EVOLVABILITY                             │
│     → Dễ cho engineer THAY ĐỔI hệ thống      │
│       trong tương lai                        │
└─────────────────────────────────────────────┘
```

---

## 3. Operability: Làm cho cuộc sống Operations dễ dàng hơn

### Trích dẫn quan trọng

> _"Good operations can often work around the limitations of bad (or incomplete) software, but good software cannot run reliably with bad operations."_
> — Jay Kreps

→ **Operations TỐT** quan trọng **hơn cả software tốt**.

### Automation: Lưỡi dao 2 chiều

```
Hệ thống LỚN (hàng nghìn máy):
   Manual maintenance = QUÁ ĐẮT
   → AUTOMATION là CẦN THIẾT

NHƯNG: Automation có 2 mặt:
   ✓ Giảm công việc thủ công lặp lại
   ✗ LUÔN có edge case cần manual intervention (failure scenarios hiếm)
   ✗ Edge case KHÔNG xử lý tự động được THƯỜNG LÀ phức tạp nhất
     → Cần TEAM OPERATIONS GIỎI HƠN để xử lý chúng
   ✗ Hệ thống automated bị lỗi THƯỜNG KHÓ troubleshoot hơn
     hệ thống dựa vào operator thao tác thủ công
```

> **Kết luận:** "More automation" KHÔNG LUÔN tốt hơn cho operability. Cần tìm **sweet spot** tùy theo application/organization cụ thể.

### Data Systems giúp Operability tốt như thế nào?

| Cách hệ thống hỗ trợ Operations | Chi tiết                                                 |
| ------------------------------- | -------------------------------------------------------- |
| **Monitoring & Observability**  | Theo dõi key metrics, hiểu runtime behavior              |
| **Không phụ thuộc máy đơn lẻ**  | Cho phép maintenance 1 máy mà hệ thống vẫn chạy          |
| **Documentation tốt**           | Operational model dễ hiểu ("Nếu tôi làm X, Y sẽ xảy ra") |
| **Good default behavior**       | NHƯNG vẫn cho admin tự do override khi cần               |
| **Self-healing**                | Nhưng vẫn giữ manual control cho admin khi cần           |
| **Hành vi dự đoán được**        | Giảm thiểu bất ngờ (predictable behavior)                |

---

## 4. Simplicity: Quản lý Complexity

### Vấn đề "Big Ball of Mud"

```
Project nhỏ: code có thể ĐẸP, ĐƠN GIẢN, EXPRESSIVE
Project LỚN: thường trở nên RẤT PHỨC TẠP, KHÓ HIỂU
   → Gọi là "Big Ball of Mud"
```

**Hệ quả của Complexity:**

- Làm chậm MỌI người làm việc trên hệ thống
- Tăng chi phí maintenance
- Tăng RỦI RO bug khi thay đổi code
- Hidden assumptions, unintended consequences, unexpected interactions
  → DỄ BỊ BỎ QUA hơn

> **Kết luận:** Giảm complexity → cải thiện ĐÁNG KỂ maintainability → **Simplicity nên là mục tiêu chính** khi xây hệ thống.

### Simplicity là một khái niệm CHỦ QUAN

```
KHÔNG có chuẩn objective cho "đơn giản"

Ví dụ:
   Hệ thống A: implementation PHỨC TẠP, ẩn sau interface ĐƠN GIẢN
   Hệ thống B: implementation ĐƠN GIẢN, expose nhiều internal detail

→ Cái nào "đơn giản hơn"? KHÔNG CÓ câu trả lời dứt khoát
```

### Essential vs Accidental Complexity

| Loại complexity | Định nghĩa                                      |
| --------------- | ----------------------------------------------- |
| **Essential**   | Vốn có trong **problem domain** của application |
| **Accidental**  | Sinh ra CHỈ VÌ **hạn chế của tooling**          |

> **Lưu ý:** Phân biệt này **cũng có flaw** — biên giới giữa essential và accidental **dịch chuyển** khi tooling phát triển.

### Abstraction — Công cụ tốt nhất để quản lý Complexity

```
Abstraction TỐT:
   ✓ Ẩn nhiều implementation detail sau giao diện SẠCH, DỄ HIỂU
   ✓ Dùng được cho NHIỀU loại application (reuse)
   ✓ Reuse hiệu quả hơn reimplement nhiều lần
   ✓ Quality improvement ở abstraction → benefit MỌI application dùng nó
```

**Ví dụ về Abstraction:**

- **High-level programming languages** → ẩn machine code, CPU registers, system calls
- **SQL** → ẩn on-disk/in-memory data structures, concurrent requests, crash inconsistencies

```
Dùng programming language CẤP CAO:
   VẪN dùng machine code BÊN DƯỚI
   NHƯNG không cần TRỰC TIẾP nghĩ về nó
   (vì abstraction đã "giải phóng" ta khỏi việc đó)
```

> **Lưu ý:** Sách này nói về **general-purpose abstractions** (database transactions, indexes, event logs) — chứ KHÔNG nói về application-specific abstractions như Design Patterns hay Domain-Driven Design (DDD). Nếu muốn dùng DDD, có thể XÂY trên nền tảng từ sách này.

---

## 5. Evolvability: Làm cho Thay đổi dễ dàng

### Requirements LUÔN thay đổi

```
Requirements KHÔNG BAO GIỜ "đứng yên mãi mãi"
   Nguyên nhân thay đổi:
   - Học được facts mới
   - Use cases mới phát sinh không lường trước
   - Business priorities thay đổi
   - Users yêu cầu feature mới
   - Platform mới thay thế platform cũ
   - Legal/regulatory requirements thay đổi
   - Growth ép buộc architectural changes
```

### Agile & Evolvability

```
Agile (organizational process) → framework để thích ứng với thay đổi
TDD, Refactoring → technical tools hỗ trợ Agile

Sách này: tập trung vào AGILITY ở LEVEL HỆ THỐNG
   (gồm nhiều application/service với đặc điểm khác nhau)
   → dùng thuật ngữ riêng: EVOLVABILITY
```

> **Quan hệ với Simplicity:** Hệ thống **loosely coupled, simple** thường dễ thay đổi HƠN hệ thống **tightly coupled, complex**.

### Irreversibility — Kẻ thù của Evolvability

```
IRREVERSIBILITY = hành động KHÔNG THỂ hoàn tác
   → là yếu tố CHÍNH làm thay đổi KHÓ trong hệ thống lớn

Ví dụ: Migrate từ database A → database B
   Nếu KHÔNG THỂ switch back nếu B có vấn đề
      → Stakes RẤT CAO
   Nếu CÓ THỂ dễ dàng quay lại A
      → Stakes thấp hơn nhiều

→ Hành động IRREVERSIBLE cần được thực hiện RẤT CẨN THẬN
→ MINIMIZE irreversibility → cải thiện FLEXIBILITY
```

---

## Tóm tắt phần này

```
Maintainability = phần lớn CHI PHÍ thực sự của software

3 nguyên tắc:
┌─────────────┬──────────────────────────────────────────┐
│ Operability │ Dễ vận hành: monitoring, docs, defaults,  │
│             │ self-healing, predictable behavior        │
├─────────────┼──────────────────────────────────────────┤
│ Simplicity  │ Giảm complexity bằng ABSTRACTION tốt      │
│             │ (essential vs accidental complexity)      │
├─────────────┼──────────────────────────────────────────┤
│ Evolvability│ Dễ thay đổi: loosely coupled, simple,     │
│             │ MINIMIZE irreversibility                  │
└─────────────┴──────────────────────────────────────────┘

Insight: Automation = lưỡi dao 2 chiều, không phải luôn tốt
Insight: Mọi system thành công LÂU DÀI sẽ trở thành LEGACY SYSTEM
```
