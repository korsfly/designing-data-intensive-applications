# Data Systems, Law, and Society

---

## 1. Trách nhiệm vượt ra ngoài kinh doanh

Kiến trúc của data systems không chỉ bị ảnh hưởng bởi yêu cầu kỹ thuật và nhu cầu kinh doanh, mà còn bởi **nhu cầu con người** trong xã hội mà tổ chức phục vụ.

> **Quan điểm cốt lõi của chương:** Kỹ sư data systems ngày càng nhận ra rằng phục vụ nhu cầu của business thôi là không đủ — họ còn có **trách nhiệm với xã hội nói chung**.

---

## 2. Bối cảnh pháp lý

### Các quy định quan trọng

| Quy định           | Khu vực        | Nội dung chính                                                              |
| ------------------ | -------------- | --------------------------------------------------------------------------- |
| **GDPR** (từ 2018) | EU             | Cho cư dân nhiều nước châu Âu quyền kiểm soát lớn hơn đối với personal data |
| **CCPA**           | California, US | Quy định tương tự GDPR về privacy                                           |
| **EU AI Act**      | EU             | Hạn chế cách sử dụng personal data trong AI                                 |

### Tác động xã hội của hệ thống máy tính

Ngay cả ở những khu vực chưa có regulation trực tiếp, vẫn cần nhận thức về tác động:

- **Social media** thay đổi cách con người tiêu thụ tin tức → ảnh hưởng quan điểm chính trị → có thể ảnh hưởng kết quả bầu cử
- **Automated systems** ngày càng đưa ra quyết định ảnh hưởng sâu sắc đến cá nhân:
  - Ai được cấp vay (loan)
  - Ai được bảo hiểm (insurance coverage)
  - Ai được mời phỏng vấn việc làm
  - Ai bị nghi ngờ phạm tội

> Mọi người làm việc trên các hệ thống này đều chia sẻ trách nhiệm xem xét tác động đạo đức của quyết định mình đưa ra, và đảm bảo tuân thủ luật liên quan.

---

## 3. GDPR và "Right to be Forgotten"

### Vấn đề kỹ thuật từ yêu cầu pháp lý

GDPR cho cá nhân quyền yêu cầu **xóa data của họ** (right to be forgotten).

**Mâu thuẫn kỹ thuật:** Nhiều data systems dựa trên các cấu trúc **immutable** (bất biến) như **append-only logs** — đây là một thiết kế phổ biến trong sách này.

### Các câu hỏi kỹ thuật chưa có giải pháp đơn giản

```
Append-only Log (immutable)
   │
   ├── Làm sao xóa một record ở giữa file mà "đáng lẽ" bất biến?
   │
   └── Data đã lan vào derived datasets thì sao?
          │
          ├── Training data cho ML models
          ├── Caches
          ├── Materialized views
          └── Indexes
```

→ Việc trả lời các câu hỏi này tạo ra **engineering challenges mới**.

### Thiếu hướng dẫn cụ thể

- Hiện tại **không có guideline rõ ràng** về kiến trúc/technology nào được coi là "GDPR compliant"
- Luật **cố tình** không bắt buộc công nghệ cụ thể (vì công nghệ thay đổi nhanh)
- Luật chỉ đưa ra **nguyên tắc cấp cao** (high-level principles), cần được diễn giải (interpretation)

---

## 4. Cost-Benefit của việc lưu trữ data

### Chi phí lưu trữ không chỉ là tiền bill

```
Chi phí thực sự của lưu trữ data =
    Chi phí storage (S3 bill, v.v.)
  + Rủi ro liability (trách nhiệm pháp lý)
  + Rủi ro reputational damage (nếu data bị leak)
  + Rủi ro legal costs & fines (nếu không compliant)
```

### Rủi ro chính trị/an toàn của data nhạy cảm

Khi data có thể tiết lộ các hành vi bị criminalize ở một số khu vực:

| Ví dụ                                | Rủi ro                                           |
| ------------------------------------ | ------------------------------------------------ |
| Đồng tính (homosexuality)            | Bị criminalize ở một số nước Trung Đông/Châu Phi |
| Tìm kiếm dịch vụ phá thai (abortion) | Bị criminalize ở một số bang tại Mỹ              |

**Cách data có thể bị "lộ":**

- **Location data** (ví dụ: đi đến phòng khám phá thai)
- **IP address logs theo thời gian** → có thể suy ra vị trí gần đúng

> Lưu trữ loại data này tạo ra **rủi ro an toàn thực sự** cho người dùng.

---

## 5. Data Minimization

### Nguyên tắc

**Data minimization** (tiếng Đức: _Datensparsamkeit_) = chỉ lưu trữ data **thực sự cần thiết**.

### Đối lập với "Big Data Philosophy"

| Big Data Philosophy                                  | Data Minimization                          |
| ---------------------------------------------------- | ------------------------------------------ |
| Lưu nhiều data nhất có thể, "đầu cơ" (speculatively) | Chỉ lưu data cần thiết cho mục đích cụ thể |
| "Có thể sẽ hữu ích trong tương lai"                  | Phải có lý do rõ ràng để lưu               |

### Yêu cầu của GDPR (phù hợp với Data Minimization)

GDPR mandates:

1. Personal data chỉ được thu thập cho **mục đích cụ thể, rõ ràng** (specified, explicit purpose)
2. **Không** được dùng sau đó cho mục đích khác
3. **Không** được giữ lâu hơn mức cần thiết cho mục đích thu thập ban đầu

---

## 6. Tiêu chuẩn ngành (Business Compliance)

Ngoài luật pháp, ngành công nghiệp cũng có các tiêu chuẩn tự quản (self-regulation):

| Tiêu chuẩn                      | Áp dụng cho              | Đặc điểm                                                    |
| ------------------------------- | ------------------------ | ----------------------------------------------------------- |
| **PCI** (Payment Card Industry) | Công ty xử lý thanh toán | Yêu cầu strict compliance, audit thường xuyên từ bên thứ ba |
| **SOC Type 2**                  | Software vendors         | Buyer yêu cầu vendor tuân thủ, audit từ third-party         |

→ Xu hướng chung: **third-party audits** để verify compliance.

---

## 7. Nguyên tắc cân bằng

> **Kết luận chính của phần này:** Cần cân bằng giữa nhu cầu của business và quyền của những người mà data của họ đang được thu thập/xử lý.

```
        Business Needs                    Individual Rights
        (lợi nhuận, insight,       ⇄      (privacy, safety,
         hiệu quả vận hành)               kiểm soát data cá nhân)
```

Đây không phải vấn đề có giải pháp đơn giản — và sẽ được khai triển sâu hơn ở **Chapter 14** (đạo đức, bias, discrimination).

---

## Tóm tắt phần này

| Khái niệm                                | Ý nghĩa                                                |
| ---------------------------------------- | ------------------------------------------------------ |
| **GDPR / CCPA / EU AI Act**              | Khung pháp lý bảo vệ personal data                     |
| **Right to be forgotten**                | Thử thách kỹ thuật với immutable systems               |
| **Data minimization (Datensparsamkeit)** | Chỉ lưu data cần thiết, đối lập "big data" philosophy  |
| **Cost-benefit của storage**             | Phải tính cả risk pháp lý/reputational, không chỉ tiền |
| **PCI / SOC 2**                          | Tiêu chuẩn ngành, third-party audit                    |

**Liên kết:** Chapter 14 sẽ đi sâu hơn vào đạo đức và pháp lý, bao gồm bias và discrimination trong các hệ thống tự động.
