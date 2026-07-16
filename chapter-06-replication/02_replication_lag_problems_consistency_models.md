# Replication Lag Problems và Consistency Models

---

## 1. Eventual Consistency và Replication Lag

### Read Scaling Architecture

```
Leader-based replication: TẤT CẢ writes → single leader
NHƯNG: Read-only queries → BẤT KỲ replica nào

Workload nhiều reads, ít writes (phổ biến với online services):
   Tạo NHIỀU followers → distribute read requests
   → Giảm load trên leader
   → Serve reads từ geographically nearby replicas

→ Thực tế ĐỒI HỎI ASYNCHRONOUS replication
   (Synchronous với tất cả followers: 1 node fail → toàn hệ thống không viết được)
```

### Vấn đề: Stale Reads từ Asynchronous Follower

```
Đọc từ async follower → CÓ THỂ thấy OUTDATED DATA
(follower chưa kịp apply writes mới nhất từ leader)

Nếu chạy cùng query trên leader và follower cùng lúc:
   → Kết quả KHÁC NHAU

EVENTUAL CONSISTENCY:
   Nếu dừng viết vào database và chờ → followers eventually catch up
   Inconsistency = TEMPORARY STATE

Replication lag = thời gian từ khi write xảy ra trên leader
   đến khi reflected trên follower
   Bình thường: fraction of a second
   Nếu system near capacity hoặc network problems: NHIỀU PHÚT
```

> **"Eventually"** là cố tình mơ hồ — không có giới hạn về lag. Đây không chỉ là vấn đề lý thuyết mà là vấn đề THỰC TẾ cho applications.

---

## 2. Regions và Availability Zones

### Định nghĩa

```
REGION: Một hoặc nhiều datacenters trong CÙNG GEOGRAPHIC LOCATION
   (cloud providers sử dụng khái niệm này)

AVAILABILITY ZONE (ZONE): Mỗi datacenter riêng biệt trong cùng region
   Cùng region → kết nối bởi VERY HIGH-SPEED NETWORK
   Latency đủ thấp để distributed systems chạy qua nhiều zones như cùng 1 zone

MULTI-ZONE: Survive zonal outage (1 zone down)
   NHƯNG không protect khỏi REGIONAL OUTAGE (tất cả zones down)

MULTI-REGION: Survive regional outage
   NHƯNG: Higher latency, lower throughput, increased networking costs
```

---

## 3. Ba Anomaly của Replication Lag

### Anomaly 1: Read-After-Write Consistency

#### Vấn đề

```
User SUBMIT data → read lại ngay sau → KHÔNG THẤY DATA VỪA SUBMIT

Lý do: Write đi đến leader, nhưng read có thể đi đến async follower
   chưa nhận write đó

→ User thấy như data của họ ĐÃ BỊ MẤT
```

#### Giải pháp: Read-Your-Writes Consistency

> Còn gọi: **read-your-writes consistency**. Đảm bảo user luôn thấy updates mà **chính họ** đã submit. Không đảm bảo gì về updates của người dùng khác.

**Các kỹ thuật:**

| Kỹ thuật                    | Mô tả                                                                                                                                                                  |
| --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Route theo loại data**    | Đọc data user CÓ THỂ đã modify → read từ leader. Ví dụ: luôn đọc own profile từ leader, other profiles từ follower                                                     |
| **Time-based routing**      | Trong 1 phút sau khi user làm write, đọc từ leader. Monitor replication lag, không query follower đang behind > 1 phút                                                 |
| **Timestamp-based**         | Client nhớ timestamp của write gần nhất. System đảm bảo replica phục vụ read đó reflect updates đến ít nhất timestamp đó. Nếu không: route sang replica khác hoặc wait |
| **Multi-region complexity** | Request cần serve bởi leader → phải route đến region chứa leader                                                                                                       |

**Cross-device read-after-write:**

```
User enter info trên device A → xem trên device B → phải thấy info đó

Challenges:
   Timestamp của last update KHÔNG dễ share giữa devices → cần CENTRALIZE metadata
   Different devices có thể route đến DIFFERENT REGIONS
   → Nếu dùng leader read: phải route TẤT CẢ devices cùng user đến cùng region
```

---

### Anomaly 2: Monotonic Reads

#### Vấn đề

```
User thấy TIME GO BACKWARD:
   First read từ replica ÍT LAG → thấy comment user khác vừa post
   Second read từ replica NHIỀU LAG → KHÔNG thấy comment đó nữa

→ Comment vừa "appear" rồi "disappear"
→ Rất confusing cho user
```

#### Giải pháp: Monotonic Reads

> **Monotonic Reads:** Đảm bảo nếu 1 user make several reads theo sequence, họ KHÔNG thấy time go backward (không đọc data cũ hơn sau khi đã đọc data mới hơn).

```
Không phải strong consistency, nhưng MẠNH HƠN eventual consistency

Cách đạt được:
   Đảm bảo mỗi user LUÔN đọc từ CÙNG 1 REPLICA
   (các users khác nhau có thể đọc từ các replicas khác nhau)

   Ví dụ: chọn replica dựa trên HASH của user ID (không random)

   Nhược điểm: Nếu replica đó fail → phải reroute sang replica khác
```

---

### Anomaly 3: Consistent Prefix Reads

#### Vấn đề

```
Causal ordering bị vi phạm → người quan sát thấy EFFECT TRƯỚC CAUSE:

Mr. Poons: "How far into the future can you see, Mrs. Cake?"
Mrs. Cake: "About 10 seconds usually, Mr. Poons."

Nếu reply của Mrs. Cake replicated NHANH HƠN câu hỏi của Mr. Poons:
Observer thấy:
   Mrs. Cake: "About 10 seconds usually, Mr. Poons."  [TRƯỚC]
   Mr. Poons: "How far into the future can you see?"  [SAU]

→ Mrs. Cake trả lời trước khi được hỏi! (như có ESP)
```

#### Giải pháp: Consistent Prefix Reads

> **Consistent Prefix Reads:** Đảm bảo nếu một sequence of writes xảy ra theo thứ tự nhất định, bất kỳ ai đọc writes đó cũng thấy chúng theo CÙNG THỨ TỰ.

```
Đặc biệt là vấn đề với SHARDED databases:
   Different shards operate independently → không có GLOBAL ORDERING

Solution: Writes có causal relationship → viết vào CÙNG SHARD
   (nhưng không phải lúc nào cũng có thể làm hiệu quả)

Formal approach: CAUSAL DEPENDENCY TRACKING
   (algorithm để track happens-before relation — xem phần 5)
```

---

## 4. Solutions for Replication Lag

### Trade-offs và Quyết định kiến trúc

```
Khi working với eventually consistent system:
   Hỏi: "Nếu replication lag tăng lên vài phút hoặc vài giờ,
         app behave như thế nào?"

   Nếu OK → tốt
   Nếu BAD EXPERIENCE → cần STRONGER GUARANTEE
```

### Option 1: Handle trong Application Code

```
Application có thể provide STRONGER GUARANTEE hơn underlying DB
   (vd: đọc từ leader cho certain reads)

NHƯNG: Complex và dễ mắc lỗi
```

### Option 2: Chọn Database có Strong Consistency (Đơn giản nhất)

```
Chọn DB cung cấp:
   STRONG CONSISTENCY GUARANTEE (như linearizability — Chapter 10)
   ACID TRANSACTIONS (Chapter 8)

→ Application developer có thể IGNORE challenges từ replication
   → Treat DB như SINGLE NODE

TRƯỚC ĐÂY (early 2010s): Quan điểm NoSQL cho rằng strong consistency
   limited scalability → large-scale systems phải embrace eventual consistency

NGÀY NAY (NewSQL): Nhiều databases cung cấp cả strong consistency/transactions
   VÀ fault tolerance/high availability/scalability
   → NewSQL trend: không phải về SQL specifically, mà về new approaches
     to scalable transaction management

VẪN CÒN LÝ DO dùng weaker consistency:
   Stronger resilience với network interruptions
   Lower overhead so với transactional systems
```

---

## Tóm tắt phần này

```
EVENTUAL CONSISTENCY:
   Asynchronous replication → followers có thể STALE
   Replication lag: bình thường fraction of second;
   có thể nhiều phút khi problems

3 ANOMALIES CỦA REPLICATION LAG:

   1. READ-AFTER-WRITE CONSISTENCY:
      Vấn đề: User không thấy data mình vừa submit
      Giải pháp: Route based on data type / timestamp / time window

   2. MONOTONIC READS:
      Vấn đề: User thấy data "disappear" → time appears to go backward
      Giải pháp: Route cùng user luôn đến cùng replica (hash of user ID)

   3. CONSISTENT PREFIX READS:
      Vấn đề: Observer thấy effect trước cause (causality violation)
      Giải pháp: Causal writes → cùng shard; causal dependency tracking

GIẢI PHÁP:
   Application-level (phức tạp) HOẶC
   Database với strong consistency (NewSQL, linearizability, ACID)
   HOẶC chấp nhận eventual consistency (nếu app cho phép)
```
