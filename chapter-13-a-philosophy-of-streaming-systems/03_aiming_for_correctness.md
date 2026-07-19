# Phần 3 — Hướng Tới Sự Đúng Đắn (Aiming for Correctness)

## 3.1 Vì sao stateful system cần cẩn trọng hơn

Với service stateless chỉ đọc dữ liệu, lỗi không nghiêm trọng — sửa bug rồi restart
là xong. Database và các hệ thống **stateful** thì khác: chúng được thiết kế để
**nhớ mãi mãi**, nên nếu có gì sai, hậu quả cũng có thể kéo dài mãi mãi.

Suốt ~40 năm, thuộc tính transaction (atomicity, isolation, durability) là công cụ
chính để xây ứng dụng đúng đắn. Nhưng nền tảng này **yếu hơn vẻ ngoài** — ví dụ sự
rối rắm của các mức cô lập yếu (weak isolation level). Nhiều hệ thống hiện đại đã bỏ
transaction hoàn toàn để đổi lấy hiệu năng/khả năng mở rộng tốt hơn, nhưng ngữ nghĩa
lại lộn xộn hơn. **Jepsen experiments** của Kyle Kingsbury từng chỉ ra khoảng cách rõ
rệt giữa những gì sản phẩm database _hứa hẹn_ và hành vi _thực tế_ khi có sự cố mạng.

Nếu ứng dụng chấp nhận được thi thoảng mất/hỏng dữ liệu không đoán trước, mọi chuyện
đơn giản hơn. Nếu cần đảm bảo mạnh, **serializability + atomic commit** là cách tiếp
cận đã được kiểm chứng — nhưng có cái giá: thường chỉ hoạt động tốt trong **một
datacenter** (loại trừ kiến trúc phân tán địa lý), giới hạn khả năng mở rộng và chịu
lỗi. Chương này khám phá **cách khác để nghĩ về correctness** trong kiến trúc dataflow.

## 3.2 The End-to-End Argument for Databases

Dùng data system có thuộc tính an toàn mạnh (như serializable transaction) **không
đảm bảo** ứng dụng không mất/hỏng dữ liệu — ví dụ nếu ứng dụng có bug ghi sai hoặc
xóa nhầm dữ liệu, transaction serializable không cứu được. Đây là lý do ủng hộ dữ
liệu **bất biến, append-only** — dễ khôi phục hơn khi code lỗi vì không thể phá hủy
dữ liệu tốt.

### Exactly-once execution

"Exactly-once" (hay "effectively-once") nghĩa là: nếu xử lý message gặp lỗi, hoặc bỏ
cuộc (mất dữ liệu) hoặc thử lại (rủi ro xử lý 2 lần). Xử lý 2 lần = **hỏng dữ liệu**
(tính phí khách 2 lần, cộng dồn counter 2 lần...). Cách hiệu quả nhất: làm cho thao
tác **idempotent** — nhưng với thao tác vốn không idempotent, cần công sức: lưu thêm
metadata (tập ID thao tác đã áp dụng), đảm bảo fencing khi failover.

### Duplicate suppression

Pattern loại bỏ trùng lặp xuất hiện ở nhiều nơi:

- **TCP** dùng sequence number để sắp thứ tự, phát hiện gói mất/trùng — nhưng chỉ
  hoạt động **trong phạm vi một kết nối TCP**.
- Ví dụ: transaction chuyển tiền (Example 13-1) **không idempotent**. Nếu client mất
  kết nối sau khi gửi `COMMIT` nhưng trước khi nhận phản hồi, họ không biết transaction
  đã commit hay abort. Retry ngoài phạm vi TCP dedup → có thể chuyển **gấp đôi** số
  tiền dự định.
- **2PC** phá vỡ ánh xạ 1-1 giữa kết nối TCP và transaction (coordinator phải kết
  nối lại sau lỗi mạng để quyết định commit/abort) — nhưng vẫn **chưa đủ** đảm bảo
  thực thi đúng một lần.
- Vấn đề còn tồn tại ở tầng **giữa thiết bị người dùng và application server**: ví
  dụ browser gửi POST qua mạng cellular yếu, mất tín hiệu trước khi nhận response →
  người dùng thấy lỗi, retry thủ công ("Bạn có chắc muốn gửi lại form?") → từ góc
  nhìn web server, đây là **request riêng biệt**; từ góc nhìn database, là
  **transaction riêng biệt**. Cơ chế dedup thông thường **không giúp được**.

### Định danh duy nhất cho request

Để idempotent qua nhiều chặng mạng, không đủ chỉ dựa vào cơ chế transaction của
database — cần xét **toàn bộ luồng request từ đầu đến cuối (end-to-end)**. Giải
pháp: sinh **unique ID** (UUID hoặc hash các trường form) cho mỗi request ngay từ
client, truyền xuyên suốt đến database, và enforce **uniqueness constraint** trên
cột request ID (Example 13-2) — nếu ID đã tồn tại, `INSERT` fail, transaction abort,
tránh áp dụng 2 lần. Relational database có thể duy trì uniqueness constraint đúng
đắn ngay cả ở mức cô lập yếu (khác với check-then-insert ở tầng ứng dụng, dễ lỗi
dưới isolation không serializable — xem "Write Skew and Phantoms"). Bảng `requests`
này còn đóng vai trò như **event log**, hữu ích cho event sourcing/CDC.

### Nguyên lý End-to-End Argument

Được Saltzer, Reed và Clark phát biểu năm 1984:

> _"Chức năng đang xét chỉ có thể được cài đặt đầy đủ và chính xác với sự hiểu biết
> và hỗ trợ của ứng dụng đứng ở hai đầu mút của hệ thống liên lạc. Do đó, cung cấp
> chức năng đó như một tính năng của bản thân hệ thống liên lạc là không thể (dù
> một phiên bản không đầy đủ đôi khi có ích như một cải thiện hiệu năng)."_

Áp dụng vào ví dụ trên: chức năng là **loại bỏ trùng lặp**. TCP loại trùng ở cấp kết
nối, một số stream processor có "exactly-once" ở cấp xử lý message — nhưng **không
đủ** ngăn người dùng gửi trùng request nếu request đầu timeout. Chỉ giải pháp
**end-to-end** (transaction ID xuyên suốt từ client đến database) mới giải quyết
triệt để.

Nguyên lý này còn áp dụng cho:

- **Kiểm tra tính toàn vẹn dữ liệu**: checksum ở Ethernet/TCP/TLS phát hiện hỏng gói
  tin trên mạng, nhưng không phát hiện hỏng do bug phần mềm ở 2 đầu, hay hỏng trên
  đĩa lưu trữ → cần **checksum end-to-end**.
- **Mã hóa**: mật khẩu WiFi bảo vệ khỏi nghe lén trên WiFi, không bảo vệ khỏi kẻ tấn
  công trên internet; TLS/SSL bảo vệ khỏi tấn công mạng, không bảo vệ khỏi server bị
  xâm nhập → chỉ **end-to-end encryption** mới bảo vệ toàn diện.

Các cơ chế cấp thấp (TCP dedup, checksum Ethernet, mã hóa WiFi) vẫn **hữu ích** dù
không tự đủ — chúng giảm xác suất gặp vấn đề ở tầng cao hơn.

### Áp dụng end-to-end thinking vào data system

Sử dụng data system với thuộc tính an toàn mạnh **không đảm bảo** ứng dụng miễn
nhiễm mất/hỏng dữ liệu — bản thân ứng dụng phải có biện pháp end-to-end (như dedup
request). Điều đáng tiếc: cơ chế fault-tolerance cấp cao rất khó làm đúng, và ta vẫn
**chưa tìm ra abstraction chuẩn** để đóng gói nó cho application code khỏi phải lo.
Transaction từng được xem là abstraction hữu ích — thu gọn nhiều vấn đề (ghi đồng
thời, vi phạm ràng buộc, crash, gián đoạn mạng, hỏng đĩa) về 2 kết quả: commit hoặc
abort. Nhưng transaction **tốn kém**, đặc biệt qua nhiều công nghệ lưu trữ dị chất —
khi từ chối dùng distributed transaction, ta buộc phải tự cài lại cơ chế
fault-tolerance ở tầng ứng dụng, dễ sai sót (dẫn đến mất/hỏng dữ liệu).

## 3.3 Enforcing Constraints (Thực thi ràng buộc)

### Uniqueness constraint cần consensus

Trong hệ phân tán, thực thi uniqueness constraint **cần consensus** (Chương 10) —
nếu nhiều request cùng giá trị đến đồng thời, hệ thống phải quyết định request nào
được chấp nhận, các request khác bị từ chối.

Cách phổ biến nhất: **một leader duy nhất** ra mọi quyết định. Có thể scale bằng
**shard theo giá trị cần unique** (ví dụ shard theo hash username). Tuy nhiên,
**multi-leader bất đồng bộ bị loại trừ**, vì các leader khác nhau có thể chấp nhận
ghi xung đột đồng thời → giá trị không còn duy nhất. Nếu muốn từ chối ngay lập tức
write vi phạm ràng buộc, **coordination đồng bộ là không thể tránh**.

### Uniqueness trong log-based messaging

Shared log đảm bảo mọi consumer thấy message theo cùng thứ tự (total order broadcast
= consensus). Cách tương tự có thể enforce uniqueness constraint:

1. Mỗi request cho một username được mã hóa thành message, append vào shard xác
   định bởi hash(username).
2. Stream processor đọc tuần tự các request trong log, dùng local DB để theo dõi
   username nào đã được lấy. Username còn trống → ghi nhận đã lấy, phát message
   thành công. Username đã lấy → phát message từ chối.
3. Client theo dõi output stream, chờ message thành công/từ chối tương ứng.

Đây chính là cách xây dựng consensus qua shared log (Chương 10) — mở rộng dễ dàng
bằng cách tăng số shard, vì mỗi shard xử lý độc lập. Nguyên tắc cơ bản: **write có
thể xung đột phải route đến cùng shard và xử lý tuần tự** — áp dụng được cho nhiều
loại ràng buộc, không chỉ uniqueness.

### Xử lý request đa shard (Multishard request processing)

Ví dụ chuyển tiền (Example 13-2) có 3 shard tiềm năng: request ID, tài khoản người
nhận, tài khoản người gửi. Cách truyền thống cần **atomic commit xuyên 3 shard**,
buộc request vào một total order với mọi transaction khác trên các shard đó →
throughput giảm.

**Cách thay thế không cần cross-shard transaction** (Figure 13-2), dùng sharded log

- stream processor:

1. Request chuyển tiền được gán unique ID, append vào log shard theo tài khoản nguồn.
2. Stream processor đọc log request, duy trì DB local về trạng thái tài khoản nguồn
   - ID request đã xử lý. Nếu đủ tiền: cập nhật local state (giữ chỗ số tiền), phát
     sự kiện đến log shard nguồn (outgoing), log shard đích (incoming), log shard phí
     (incoming) — kèm request ID gốc.
3. Sự kiện outgoing quay lại processor tài khoản nguồn — nhận diện qua request ID là
   khoản đã "giữ chỗ" trước đó, thực thi thanh toán, bỏ qua trùng lặp theo request ID.
4. Log shard đích và phí được consume bởi task xử lý độc lập, cập nhật state, dedup
   theo request ID.

Yêu cầu duy nhất: sự kiện của mỗi tài khoản xử lý **đúng thứ tự log** với
**at-least-once semantics**, và stream processor **tất định**. Nếu processor crash
giữa chừng, sau khi phục hồi sẽ xử lý lại request (do at-least-once), ra **cùng
quyết định** (do tất định) → cùng output message, cùng request ID → consumer phía
sau tự dedup nếu trùng.

**Atomicity ở đây không đến từ transaction**, mà từ việc ghi request event ban đầu
vào log nguồn là hành động **atomic**. Một khi event đó vào log, mọi event downstream
sẽ được ghi (có thể trễ do crash/recovery, có thể trùng lặp, nhưng cuối cùng cũng
xảy ra). Với **exactly-once semantics**, việc này dễ hơn vì đảm bảo local state luôn
nhất quán với tập message đã xử lý.

→ Bằng cách chia transaction đa shard thành nhiều giai đoạn qua các shard khác nhau
và dùng request ID end-to-end, đạt **cùng thuộc tính đúng đắn** (mỗi request áp dụng
đúng một lần cho cả người trả và người nhận), kể cả khi có lỗi, **không cần** atomic
commit protocol.

## 3.4 Timeliness và Integrity

Nhiều hệ transaction có tính chất: ngay khi transaction commit, kết quả **hiển thị
ngay** cho transaction khác — hình thức hóa thành **strict serializability**. Điều
này **không đúng** khi thao tác được "thao rời" qua nhiều giai đoạn stream processor
— consumer của log bất đồng bộ theo thiết kế. Nhưng vẫn có thể chờ message xuất hiện
trên output stream (như user chờ event "outgoing payment"/"payment declined").

Thuật ngữ "consistency" gộp chung 2 yêu cầu nên tách riêng:

|                      | Timeliness                                                          | Integrity                                                                      |
| -------------------- | ------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| **Định nghĩa**       | Người dùng thấy hệ thống ở trạng thái cập nhật (up-to-date)         | Không có hỏng dữ liệu — không mất, không mâu thuẫn/sai lệch                    |
| **Ví dụ vi phạm**    | Đọc từ bản sao lag (replication lag)                                | Index thiếu record, tổng sao kê không khớp giao dịch, tiền "biến mất"          |
| **Bản chất vi phạm** | **Tạm thời** — tự sửa khi chờ và thử lại                            | **Vĩnh viễn** — chờ và thử lại không tự sửa; cần kiểm tra và sửa chữa chủ động |
| **Liên hệ CAP**      | "Consistency" trong CAP = linearizability, một dạng timeliness mạnh | ACID "consistency" thường hiểu là integrity đặc thù ứng dụng                   |

**Khẩu hiệu**: vi phạm timeliness được chấp nhận dưới eventual consistency; vi phạm
integrity gây ra tình trạng **không nhất quán vĩnh viễn**. Với hầu hết ứng dụng,
**integrity quan trọng hơn timeliness nhiều**. Ví dụ sao kê thẻ tín dụng: giao dịch
chưa hiện trong 24h là bình thường (vi phạm timeliness chấp nhận được); nhưng tổng
số dư không khớp tổng giao dịch, hoặc tiền bị trừ khách nhưng không trả merchant
(tiền "biến mất") — đó là **vi phạm integrity, rất tệ**.

### Correctness của hệ dataflow

ACID transaction thường đảm bảo cả timeliness (linearizability) lẫn integrity
(atomic commit) — nên phân biệt 2 khái niệm này ít quan trọng nếu chỉ nhìn từ góc độ
ACID. Nhưng hệ **event-based dataflow** lại **tách rời** timeliness và integrity: xử
lý stream bất đồng bộ **không đảm bảo timeliness** trừ khi chủ động chờ. Ngược lại,
**integrity là trung tâm** của streaming system — exactly-once/effectively-once
semantics chính là cơ chế bảo toàn integrity. Reliable stream processing đạt được
integrity **mà không cần** distributed transaction/atomic commit, nhờ kết hợp:

- Biểu diễn nội dung ghi như **một message duy nhất**, dễ ghi atomic (khớp với
  event sourcing).
- **Dẫn xuất mọi cập nhật khác** từ message đó qua hàm dẫn xuất tất định (giống
  stored procedure).
- **Truyền request ID client-generated** qua mọi tầng xử lý — cho phép dedup end-
  to-end, idempotence.
- **Message bất biến**, cho phép reprocess dữ liệu dẫn xuất định kỳ — dễ khôi phục
  khi có bug.

### Ràng buộc diễn giải lỏng (Loosely interpreted constraints)

Enforce uniqueness constraint truyền thống cần consensus (dồn qua một node) — không
thể tránh nếu muốn dạng ràng buộc "cứng" truyền thống. Nhưng **nhiều ứng dụng thực
tế chấp nhận vi phạm ràng buộc "cứng"** một cách có chủ đích:

- Bán hàng vượt tồn kho → đặt thêm hàng, xin lỗi, tặng giảm giá (tương tự việc xe
  nâng làm hỏng hàng trong kho — quy trình xin lỗi vốn đã cần có).
- Hãng bay/khách sạn **overbook** có chủ đích, dự trù một phần khách hủy/lỡ chuyến —
  đền bù (hoàn tiền, nâng hạng, phòng khách sạn lân cận) khi cầu vượt cung.
- Rút tiền vượt số dư → ngân hàng tính phí thấu chi, giới hạn rút/ngày để giới hạn
  rủi ro.
- Hệ thống tích hợp dữ liệu xuyên tổ chức (như thanh toán liên ngân hàng) luôn phát
  sinh không nhất quán, cần cơ chế sửa chữa.

→ Đây gọi là **compensating transaction** — thay đổi để sửa lỗi sau. Chi phí "xin
lỗi" (tiền hoặc uy tín) thường thấp: không thể "unsend" email nhưng có thể gửi email
đính chính; charge thẻ 2 lần có thể refund (chi phí chỉ là phí xử lý + có thể khách
than phiền).

Nếu chi phí xin lỗi chấp nhận được, mô hình truyền thống "kiểm tra mọi ràng buộc
trước khi ghi" là **quá hạn chế không cần thiết** — có thể ghi optimistic trước, rồi
kiểm tra ràng buộc sau đó. (Vẫn nên validate trước với hành động **không thể phục
hồi/tốn kém**.) Các ứng dụng này vẫn cần **integrity** (không được mất reservation,
tiền không được "biến mất") nhưng **không cần timeliness** cho việc thực thi ràng
buộc — có thể vá lỗi sau (giống conflict resolution ở "Dealing with Conflicting Writes").

### Hệ thống tránh coordination (Coordination-avoiding data systems)

Hai quan sát then chốt:

1. Hệ dataflow có thể duy trì integrity trên dữ liệu dẫn xuất **mà không cần** atomic
   commit, linearizability, hay synchronous cross-shard coordination.
2. Dù uniqueness constraint chặt cần timeliness + coordination, nhiều ứng dụng chấp
   nhận ràng buộc lỏng, có thể tạm vi phạm rồi sửa sau, miễn integrity được giữ.

→ Kết hợp lại: hệ dataflow có thể phục vụ **quản lý dữ liệu cho nhiều ứng dụng mà
không cần coordination**, vẫn giữ integrity mạnh. Hệ thống như vậy — **"coordination-
avoiding"** — có hiệu năng và khả năng chịu lỗi tốt hơn hệ cần synchronous
coordination. Ví dụ: hệ đa datacenter, multi-leader, replicate bất đồng bộ liên
vùng — mỗi datacenter hoạt động độc lập, không cần coordination liên vùng đồng bộ →
timeliness yếu (không linearizable nếu không có coordination) nhưng integrity vẫn mạnh.

Serializable transaction vẫn hữu ích khi duy trì derived state, nhưng chạy trong
**phạm vi nhỏ** nơi nó hoạt động tốt — không cần distributed/heterogeneous
transaction (XA) trên toàn hệ thống. Coordination đồng bộ vẫn có thể được đưa vào
nơi cần thiết (ví dụ enforce ràng buộc chặt trước hành động không thể phục hồi),
nhưng không cần toàn bộ ứng dụng trả giá cho coordination nếu chỉ một phần nhỏ cần nó.

**Cách nhìn khác**: coordination và ràng buộc chặt giảm số lần phải "xin lỗi" vì
không nhất quán, nhưng cũng giảm hiệu năng/tính sẵn sàng → tăng số lần phải "xin
lỗi" vì **outage**. Không thể giảm số lần xin lỗi về 0 — cần tìm **điểm cân bằng
tốt nhất** giữa quá nhiều không nhất quán và quá nhiều vấn đề về availability.

## 3.5 Trust, but Verify (Tin tưởng, nhưng vẫn kiểm chứng)

Mọi thảo luận về correctness/integrity/fault tolerance đều dựa trên giả định — gọi
là **system model** (xem "System Model and Reality"): process có thể crash, máy có
thể mất điện đột ngột, mạng có thể trễ/mất gói bất kỳ. Ta cũng thường giả định dữ
liệu ghi đĩa sau `fsync` không mất, dữ liệu trong RAM không hỏng, phép nhân CPU luôn
đúng. Những giả định này hợp lý vì đúng **hầu hết thời gian**, nhưng thực tế là vấn
đề **xác suất**: một số điều dễ xảy ra hơn, số khác hiếm hơn. Ở quy mô đủ lớn, **cả
điều rất hiếm cũng xảy ra**.

### Duy trì integrity trước bug phần mềm

Ngoài lỗi phần cứng, luôn có rủi ro **bug phần mềm** — không được phát hiện bởi
checksum mạng/RAM/filesystem. Ngay cả database phổ biến, được kiểm nghiệm nhiều năm
cũng từng có bug: MySQL từng lỗi duy trì uniqueness constraint; PostgreSQL serializable
isolation từng có write skew anomaly trong quá khứ. Với application code, phải giả
định **nhiều bug hơn nữa** — vì không được review/test kỹ như database code, nhiều
ứng dụng còn không dùng đúng tính năng integrity mà database cung cấp (foreign key,
uniqueness constraint). "ACID consistency" giả định transaction **không có bug** —
nếu ứng dụng dùng database sai cách (ví dụ dùng weak isolation level không an toàn),
integrity không thể đảm bảo.

### Đừng chỉ tin những gì họ hứa

Vì phần cứng lẫn phần mềm không luôn hoàn hảo, hỏng dữ liệu **là điều không thể
tránh sớm muộn**. Cần có cách **phát hiện** khi dữ liệu bị hỏng để sửa và truy tìm
nguồn gốc lỗi — gọi là **auditing** (kiểm toán). Auditing không chỉ dành cho ứng
dụng tài chính, nhưng quan trọng đặc biệt ở tài chính vì ai cũng biết lỗi có thể
xảy ra và cần khả năng phát hiện/sửa.

Hệ thống trưởng thành thường tính đến khả năng "điều khó xảy ra" và quản lý rủi ro
đó. Ví dụ: HDFS, Amazon S3 **không tin tưởng tuyệt đối** vào đĩa — chạy background
process liên tục đọc lại file, so sánh với replica khác, di chuyển file giữa đĩa để
giảm rủi ro **hỏng dữ liệu âm thầm (silent corruption)**. Muốn chắc dữ liệu còn đó,
phải **đọc và kiểm tra**. Tương tự, cần **thử phục hồi từ backup định kỳ** — nếu
không, có thể phát hiện backup hỏng khi đã quá muộn. **Đừng chỉ tin tưởng mù quáng.**

→ Hiện chưa nhiều hệ thống có cách tiếp cận **"trust, but verify"** kiểu tự kiểm
toán liên tục — đa số giả định correctness guarantee là tuyệt đối, không có phương
án cho khả năng hỏng dữ liệu hiếm gặp. Tương lai có thể xuất hiện nhiều hệ **tự kiểm
chứng (self-validating/self-auditing)** hơn.

### Thiết kế cho khả năng kiểm toán (auditability)

Nếu một transaction thay đổi nhiều object trong database, nguyên do đằng sau thường
**khó xác định** sau này — dù có transaction log, các insert/update/delete ở nhiều
bảng không nhất thiết cho biết **vì sao** thay đổi được thực hiện; lời gọi application
logic quyết định thay đổi đó là **thoáng qua, không thể tái tạo**.

Ngược lại, **hệ event-based** cung cấp khả năng kiểm toán tốt hơn. Trong event
sourcing: input người dùng là **một sự kiện bất biến**, mọi cập nhật state được dẫn
xuất từ sự kiện đó — dẫn xuất có thể làm **tất định và tái lặp**: chạy lại cùng log
sự kiện qua cùng phiên bản code dẫn xuất sẽ ra cùng kết quả cập nhật state.

Việc rõ ràng về dataflow giúp **nguồn gốc dữ liệu (provenance)** minh bạch hơn nhiều,
khiến kiểm tra integrity khả thi hơn: dùng hash để kiểm tra event log không bị hỏng;
với derived state, có thể **chạy lại** batch/stream processor đã dẫn xuất nó từ event
log để kiểm tra ra cùng kết quả, hoặc chạy **derivation dự phòng song song**.
Dataflow tất định, rõ ràng cũng giúp debug/trace dễ hơn, tạo khả năng **time-travel
debugging** — tái tạo chính xác hoàn cảnh dẫn đến sự kiện bất ngờ.

### End-to-end argument (lần nữa)

Vì không thể hoàn toàn tin tưởng mọi thành phần hệ thống miễn nhiễm lỗi/bug, cần
**định kỳ kiểm tra integrity dữ liệu**. Nếu không kiểm tra, sẽ không biết có hỏng
dữ liệu cho đến khi quá muộn, gây thiệt hại downstream — khi đó việc truy vết sẽ khó
và tốn kém hơn nhiều.

Kiểm tra integrity nên làm theo cách **end-to-end**: càng bao gồm nhiều hệ thống
trong một lần kiểm tra, càng ít cơ hội cho hỏng dữ liệu "lọt lưới" ở giai đoạn nào
đó. Nếu kiểm tra được toàn bộ pipeline dữ liệu dẫn xuất đúng đắn end-to-end, thì mọi
đĩa, mạng, dịch vụ, thuật toán trên đường đi đều **ngầm được kiểm tra**. Kiểm tra
integrity liên tục end-to-end giúp **tăng sự tự tin** vào correctness của hệ thống —
qua đó cho phép **đi nhanh hơn** (giống automated testing): tăng khả năng phát hiện
sớm bug, giảm rủi ro khi thay đổi hệ thống hoặc dùng công nghệ lưu trữ mới.

### Công cụ cho hệ thống dữ liệu có thể kiểm toán

Hiện chưa nhiều hệ dữ liệu coi auditability là ưu tiên hàng đầu. Một số ứng dụng tự
cài audit mechanism (log mọi thay đổi vào bảng audit riêng), nhưng đảm bảo integrity
của chính audit log + database state vẫn khó. Transaction log có thể được "tamper-
proof" bằng ký định kỳ với hardware security module — nhưng không đảm bảo **đúng
transaction** đã được đưa vào log ngay từ đầu.

**Blockchain** (Bitcoin, Ethereum) là log append-only chia sẻ với cryptographic
consistency check; giao dịch là sự kiện, smart contract về cơ bản là stream processor.
Khác biệt với consensus protocol ở Chương 10: blockchain có **Byzantine fault
tolerance** — vẫn hoạt động dù một số node có dữ liệu bị hỏng, vì replica liên tục
kiểm tra chéo integrity lẫn nhau. Với hầu hết ứng dụng, blockchain **quá tốn overhead**
để hữu ích, nhưng một số công cụ mật mã của nó có thể dùng nhẹ hơn:

- **Merkle tree**: cây hash chứng minh hiệu quả một record có nằm trong dataset hay
  không (và vài việc khác).
- **Certificate Transparency**: dùng append-only log kiểm chứng mật mã + Merkle tree
  để kiểm tra tính hợp lệ của chứng chỉ TLS/SSL, tránh cần consensus protocol bằng
  cách dùng **một leader duy nhất mỗi log**.

→ Thuật toán kiểm tra integrity/kiểm toán (như certificate transparency, distributed
ledger) có thể trở nên phổ biến hơn trong tương lai — cần thêm nghiên cứu để mở rộng
quy mô tương đương hệ không có kiểm toán mật mã và giảm chi phí hiệu năng, nhưng
hướng đi này rất đáng chú ý.

---
