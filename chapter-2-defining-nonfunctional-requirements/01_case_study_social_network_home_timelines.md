# Case Study: Social Network Home Timelines

---

## 1. Bối cảnh bài toán

Giả sử ta phải xây dựng một social network kiểu X (trước đây là Twitter), nơi user có thể post message và follow user khác.

### Các số liệu giả định

| Chỉ số                          | Giá trị                                                       |
| ------------------------------- | ------------------------------------------------------------- |
| Tổng số posts/ngày              | 500 triệu                                                     |
| Posts/giây (trung bình)         | 5,800                                                         |
| Posts/giây (đỉnh điểm)          | lên đến 150,000                                               |
| Số người follow trung bình/user | 200                                                           |
| Số followers trung bình/user    | 200                                                           |
| Outlier                         | Celebrity có thể có >100 triệu followers (ví dụ Barack Obama) |

---

## 2. Mô hình hóa Users, Posts, Follows

Dữ liệu được lưu trong relational database với 3 bảng chính:

- **users**
- **posts**
- **follows** (quan hệ follow giữa users)

### Yêu cầu đọc chính: Home Timeline

Home timeline = hiển thị các posts gần đây của những người mà user đang follow.

```sql
SELECT posts.*, users.* FROM posts
JOIN follows ON posts.sender_id = follows.followee_id
JOIN users
ON posts.sender_id = users.id
WHERE follows.follower_id = current_user
ORDER BY posts.timestamp DESC
LIMIT 1000
```

**Cách hoạt động:** Database tìm tất cả người mà `current_user` follow, lấy post gần đây của họ, sort theo timestamp, lấy 1000 post mới nhất.

---

## 3. Vấn đề của cách tiếp cận Naive (Polling)

### Polling là gì?

Client lặp lại query trên **mỗi 5 giây** trong khi user đang online, để đảm bảo follower thấy post mới trong vòng 5 giây.

### Tại sao đây là vấn đề lớn?

```
Giả sử: 10 triệu users online đồng thời
→ Chạy query 2 triệu lần/giây (nếu poll mỗi 5s)

Mỗi query cần fetch posts từ 200 người follow
→ 2,000,000 × 200 = 400,000,000 lookups/giây
```

→ **400 triệu lookups/giây** là một con số khổng lồ, đặc biệt với users follow hàng chục nghìn account thì query này rất nặng.

---

## 4. Giải pháp: Push + Materialization

### Hai cải tiến chính

1. **Push thay vì Poll:** Server **chủ động đẩy** (push) post mới đến followers đang online, thay vì để client liên tục hỏi.
2. **Materialization (Precompute):** Tính trước kết quả của query, lưu vào cache để serve nhanh.

### Cách hoạt động

```
Với mỗi user, lưu một data structure: "home timeline" của họ
(giống một mailbox)

Khi user X post bài mới:
   → Tìm tất cả followers của X
   → Insert bài viết mới vào home timeline của TỪNG follower
   → (giống delivering message vào mailbox)

Khi user đăng nhập:
   → Chỉ cần lấy home timeline đã được precompute (từ cache)
```

### Fan-out

> **Fan-out:** Khi một request đầu vào dẫn đến **nhiều downstream requests**, ta gọi tỷ số tăng lên này là fan-out factor.

```
                 ┌──► Follower 1's timeline
   1 New Post ───┼──► Follower 2's timeline
   (fan-out)     ├──► Follower 3's timeline
                 └──► ... (200 followers trung bình)
```

### Hiệu quả so sánh

| Cách tiếp cận                             | Số lượng work/giây                                 |
| ----------------------------------------- | -------------------------------------------------- |
| **Polling** (lookups theo sender)         | 400 triệu lookups/giây                             |
| **Push + Fan-out** (writes theo timeline) | ~1 triệu writes/giây (5,800 posts/s × fan-out 200) |

→ Push + Materialization **tiết kiệm đáng kể** so với polling.

### Ưu điểm bổ sung

- Khi spike traffic xảy ra: có thể **enqueue** việc deliver timeline, chấp nhận delay nhỏ tạm thời
- Timeline vẫn load nhanh vì luôn được serve từ **cache**

---

## 5. Materialized View — Khái niệm tổng quát

> **Materialization:** Quy trình **precompute và update** kết quả của một query.  
> **Materialized view:** Kết quả được precompute đó (ví dụ: timeline cache).

```
Materialized View đánh đổi:
   + Đọc (READ) nhanh hơn nhiều
   − Ghi (WRITE) phải làm nhiều việc hơn (update view khi data thay đổi)
```

---

## 6. Các trường hợp đặc biệt (Edge Cases)

### Trường hợp 1: User follow rất nhiều account hoạt động nhiều

```
User follow hàng nghìn account, các account này post liên tục
→ Rate viết vào materialized timeline của user đó rất cao
→ Giải pháp: User không đọc hết timeline đâu, nên có thể
   DROP một số writes, chỉ hiển thị SAMPLE bài viết
```

### Trường hợp 2: Celebrity Account (Triệu followers)

```
Celebrity post 1 bài → cần insert vào TRIỆU timeline của followers
→ KHÔNG ĐƯỢC drop writes này (quan trọng)

Giải pháp: Xử lý celebrity post RIÊNG BIỆT
   - Lưu celebrity posts ở một nơi riêng
   - KHÔNG fan-out ngay vào triệu timeline
   - Merge celebrity posts vào timeline khi user ĐỌC (read-time merge)
```

```
                    ┌── Regular posts ──► Fan-out write vào mỗi timeline
Post mới ───────────┤
                    └── Celebrity posts ──► Lưu riêng, merge lúc READ
```

> Dù có optimization này, việc xử lý celebrity trên social network vẫn cần **rất nhiều infrastructure**.

---

## Tóm tắt phần này

```
Naive Polling:
  Query lặp lại liên tục → 400 triệu lookups/giây (KHÔNG SCALE)

Push + Materialization:
  Fan-out khi WRITE → ~1 triệu writes/giây (SCALE TỐT HƠN)
  Trade-off: WRITE nặng hơn, nhưng READ rất nhanh (từ cache)

Edge cases cần xử lý đặc biệt:
  - User follow nhiều: có thể DROP writes (lossy timeline OK)
  - Celebrity: KHÔNG được drop, xử lý riêng, merge lúc đọc
```

**Bài học chính:** Case study này minh họa cụ thể cho khái niệm **throughput**, **fan-out**, và **materialized view** sẽ được dùng xuyên suốt phần còn lại của chương 2 và cả cuốn sách.
