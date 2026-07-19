# Phần 2 — Thao Rời Cơ Sở Dữ Liệu (Unbundling Databases)

## 2.1 Database, batch/stream processor, hệ điều hành — cùng bản chất

Ở mức trừu tượng, database, batch/stream processor, và hệ điều hành đều làm cùng
một việc: **lưu trữ dữ liệu và cho phép xử lý/truy vấn nó**. Database lưu theo mô
hình dữ liệu (row, document, vertex...); filesystem của OS lưu theo file — nhưng cả
hai về cốt lõi đều là hệ thống "quản lý thông tin".

**Unix và relational database có triết lý rất khác nhau:**

- Unix: cung cấp trừu tượng hóa **cấp thấp, logic nhưng gần phần cứng** (pipe, file
  = chuỗi byte).
- Relational database: cung cấp trừu tượng hóa **cấp cao**, che giấu độ phức tạp của
  cấu trúc dữ liệu trên đĩa, concurrency, crash recovery (SQL, transaction).
- Cả hai triết lý đều tồn tại song song từ đầu thập niên 1970 đến nay, chưa phân
  thắng bại. Phong trào **NoSQL** có thể xem là mang tinh thần Unix (trừu tượng cấp
  thấp) vào lĩnh vực OLTP phân tán.

## 2.2 Kết hợp các công nghệ lưu trữ dữ liệu

Các tính năng của database — secondary index, materialized view, replication log,
full-text search index — có sự **tương đồng rõ rệt** với các hệ thống dữ liệu dẫn
xuất mà người ta xây bằng batch/stream processor.

**Tạo index (`CREATE INDEX`)**: database phải quét snapshot nhất quán của bảng, sắp
xếp giá trị, ghi ra index, xử lý backlog ghi phát sinh trong lúc snapshot, rồi duy
trì index cập nhật liên tục — quá trình này **rất giống** việc thiết lập follower
replica mới, và giống việc bootstrap CDC (initial snapshot).

### "Meta-database của mọi thứ"

Nhìn theo góc độ này, **toàn bộ dataflow trong một tổ chức trông giống một database
khổng lồ**. Mỗi tiến trình batch/stream/ETL vận chuyển dữ liệu từ nơi này sang nơi
khác chính là đang đóng vai trò "subsystem" giữ index/materialized view luôn cập
nhật bên trong một database. Batch/stream processor giống các cài đặt phức tạp của
trigger, stored procedure, thuật toán duy trì materialized view; các hệ thống dữ
liệu dẫn xuất giống các "loại index" khác nhau — nhưng chạy trên nhiều máy, do nhiều
đội khác nhau vận hành, thay vì gói gọn trong một sản phẩm database tích hợp.

Hai hướng để ghép các công cụ lưu trữ/xử lý khác nhau thành hệ thống thống nhất:

| Hướng tiếp cận                                    | Vấn đề giải quyết                                                       | Ví dụ                                                       |
| ------------------------------------------------- | ----------------------------------------------------------------------- | ----------------------------------------------------------- |
| **Federated database / polystore** (hợp nhất đọc) | Cung cấp giao diện query thống nhất trên nhiều storage engine khác nhau | PostgreSQL foreign data wrapper, Trino, Hoptimator, Xorq    |
| **Unbundled database** (hợp nhất ghi)             | Đảm bảo mọi thay đổi dữ liệu đến đúng nơi, kể cả khi có lỗi             | CDC + event log để đồng bộ ghi giữa các công nghệ khác nhau |

- Federation đi theo truyền thống relational: hệ thống tích hợp đơn nhất, ngôn ngữ
  query cấp cao, ngữ nghĩa gọn gàng, nhưng cài đặt bên trong phức tạp.
- Unbundling đi theo truyền thống Unix: công cụ nhỏ làm tốt một việc, giao tiếp qua
  API cấp thấp thống nhất (pipe), kết hợp bằng ngôn ngữ cấp cao hơn (shell).

## 2.3 Làm cho unbundling hoạt động

Vấn đề khó hơn là **đồng bộ ghi** giữa nhiều hệ thống lưu trữ (đọc thì tương đối dễ
xử lý bằng ánh xạ mô hình dữ liệu). Cách truyền thống là distributed transaction
xuyên hệ thống dị chất (heterogeneous) — có vấn đề như đã bàn ở các chương trước.
**Log sự kiện có thứ tự + ghi idempotent** là cách tiếp cận thực tế và mạnh mẽ hơn
nhiều khi dữ liệu vượt ranh giới công nghệ khác nhau.

**Lợi ích của tích hợp dựa trên log — "loose coupling":**

- **Ở cấp hệ thống**: event stream bất đồng bộ giúp hệ thống tổng thể chịu được
  outage/suy giảm hiệu năng của từng thành phần riêng lẻ. Consumer chậm/lỗi → log
  đệm message lại, producer và consumer khác vẫn chạy bình thường; consumer lỗi bắt
  kịp sau khi được sửa, không mất dữ liệu, lỗi được cô lập. Ngược lại, tương tác
  đồng bộ của distributed transaction dễ khiến lỗi cục bộ leo thang thành lỗi diện rộng.
- **Ở cấp con người**: unbundling cho phép các thành phần/dịch vụ được phát triển,
  cải tiến, bảo trì **độc lập bởi các đội khác nhau**. Event log cung cấp interface
  đủ mạnh để giữ tính nhất quán chặt (nhờ durability + ordering), nhưng cũng đủ tổng
  quát để áp dụng cho hầu hết loại dữ liệu.

### Hệ thống unbundled vs. tích hợp

- Unbundling **không thay thế** database truyền thống — chúng vẫn cần thiết để duy
  trì state trong stream processor và phục vụ query cho output của batch/stream.
- Vận hành nhiều mảnh hạ tầng riêng lẻ tốn công sức (learning curve, cấu hình, vận
  hành riêng cho từng công cụ) — nên **càng ít moving part càng tốt**. Một sản phẩm
  tích hợp duy nhất thường đạt hiệu năng tốt/ổn định hơn cho loại workload nó được
  thiết kế, so với việc tự ghép nhiều công cụ. Xây dựng cho scale không cần thiết là
  **tối ưu hóa sớm (premature optimization)** lãng phí.
- Mục tiêu của unbundling **không phải cạnh tranh hiệu năng** với từng database đơn
  lẻ trên một workload cụ thể, mà là cho phép **kết hợp nhiều database** để đạt hiệu
  năng tốt trên phạm vi workload rộng hơn nhiều so với một phần mềm đơn lẻ có thể
  làm được — tức là **về bề rộng, không phải chiều sâu**. Nếu một công nghệ duy nhất
  đáp ứng đủ nhu cầu, tốt nhất hãy dùng nó thay vì tự xây lại từ linh kiện cấp thấp.
- Công cụ để ghép hệ thống dữ liệu ngày càng tốt hơn: **Debezium** (trích xuất
  change stream từ nhiều loại database), **giao thức Kafka** (đang trở thành chuẩn
  de facto cho event stream), engine **incremental view maintenance** (tiền tính
  toán và cập nhật cache cho query phức tạp).

## 2.4 Thiết kế ứng dụng xoay quanh Dataflow

Ý tưởng cập nhật dữ liệu dẫn xuất khi dữ liệu gốc thay đổi không mới — **bảng tính
(spreadsheet)** đã có khả năng dataflow programming mạnh mẽ từ 1979 (VisiCalc): đặt
công thức vào một ô, mọi input thay đổi thì kết quả tự tính lại. Đây chính xác là
điều ta muốn ở cấp hệ thống dữ liệu: khi record thay đổi, index/cached view/aggregation
liên quan **tự động cập nhật**, không cần lo chi tiết kỹ thuật.

Khác biệt so với spreadsheet: hệ thống dữ liệu hiện đại cần chịu lỗi, mở rộng quy
mô, lưu trữ bền vững, và tích hợp nhiều công nghệ khác nhau do nhiều nhóm viết theo
thời gian — không thể kỳ vọng mọi phần mềm dùng chung một ngôn ngữ/framework/công cụ.

### Application code như một "hàm dẫn xuất" (derivation function)

Khi một dataset được dẫn xuất từ dataset khác, nó đi qua một **hàm biến đổi**:

- Secondary index: hàm biến đổi đơn giản — chọn giá trị cột được index, sắp xếp theo đó.
- Full-text search index: qua nhiều hàm NLP (nhận diện ngôn ngữ, phân đoạn từ,
  stemming/lemmatization, sửa chính tả, nhận diện từ đồng nghĩa) → cấu trúc dữ liệu
  tra cứu hiệu quả (inverted index).
- Mô hình ML: dẫn xuất từ training data qua hàm feature extraction + phân tích thống kê.
- Cache: thường chứa aggregation ở dạng sẽ hiển thị trên UI — cần hiểu field nào
  được UI dùng.

Với hàm dẫn xuất "cookie-cutter" phổ biến (như secondary index), database đã tích
hợp sẵn (`CREATE INDEX`). Nhưng khi cần code tùy biến cho phần đặc thù ứng dụng (như
feature engineering trong ML), đây là chỗ nhiều database gặp khó — trigger/stored
procedure/UDF thường chỉ là "afterthought" trong thiết kế database.

### Tách biệt application code và state

Database về lý thuyết có thể là môi trường triển khai code tùy ý (như OS), nhưng
thực tế không phù hợp: quản lý dependency/package, version control, rolling upgrade,
evolvability, monitoring, gọi network service, tích hợp hệ thống ngoài — đều không
phải thế mạnh của database. Ngược lại, công cụ triển khai/quản lý cluster (Kubernetes,
Docker, Mesos, YARN...) chuyên làm tốt việc chạy application code.

Xu hướng hiện tại: **stateless service** (request nào cũng có thể route đến server
bất kỳ, server "quên" mọi thứ sau khi trả response) + state riêng nằm trong database.
→ Đùa vui trong cộng đồng lập trình hàm: _"Chúng tôi tin vào sự tách biệt giữa Church
và state"_ (chơi chữ church/state — Alonzo Church, cha đẻ lambda calculus, không có
mutable state).

**Hạn chế**: hầu hết ngôn ngữ lập trình không cho phép "subscribe" vào thay đổi của
một biến mutable — chỉ đọc định kỳ (polling). Database kế thừa cách tiếp cận thụ
động này: muốn biết dữ liệu đổi hay chưa, thường chỉ có thể poll lặp lại query.
**Subscribe vào thay đổi** mới chỉ bắt đầu xuất hiện như một tính năng.

### Dataflow: sự tương tác giữa thay đổi state và application code

Thay vì xem database là biến thụ động bị application thao túng, ta nghĩ nhiều hơn
về **sự cộng tác giữa state, thay đổi state, và code xử lý chúng** — application
code phản ứng với thay đổi state ở một nơi bằng cách kích hoạt thay đổi state ở nơi
khác. Ý tưởng này đã thấy ở CDC, actor model, trigger, incremental view maintenance.
Unbundling database nghĩa là áp dụng ý tưởng này để tạo dataset dẫn xuất **bên ngoài**
database chính: cache, search index, ML, hệ thống phân tích — dùng stream processing
và messaging system.

Yêu cầu để duy trì dữ liệu dẫn xuất đúng đắn (log-based message broker đáp ứng được):

- **Thứ tự** thay đổi state phải được giữ (nếu nhiều view dẫn xuất từ cùng event
  log, chúng phải xử lý theo cùng thứ tự để giữ nhất quán với nhau).
- **Chịu lỗi**: mất một message duy nhất khiến dataset dẫn xuất lệch vĩnh viễn khỏi
  nguồn — cả delivery lẫn xử lý phải reliable.

→ Yêu cầu này khắt khe nhưng **rẻ và bền vững hơn nhiều** so với distributed
transaction. Stream processor hiện đại có thể đáp ứng ở quy mô lớn, cho phép chạy
application code như stream operator — giống các tool Unix nối bằng pipe, các
stream operator có thể kết hợp để xây hệ thống lớn xoay quanh dataflow.

### Stream processor và service (so với microservices)

Kiến trúc microservices phổ biến hiện nay: chia chức năng thành các service giao
tiếp qua request/response đồng bộ (REST API). Ưu điểm chính là **khả năng mở rộng
tổ chức** nhờ loose coupling — các đội làm việc độc lập trên từng service.

Ghép các stream operator thành hệ thống dataflow có nhiều điểm tương đồng với
microservices, nhưng cơ chế giao tiếp khác hẳn: **message stream một chiều, bất
đồng bộ** thay vì request/response đồng bộ.

**Ví dụ so sánh — chuyển đổi ngoại tệ khi thanh toán:**

- **Cách microservices**: code xử lý mua hàng query một exchange-rate service/DB để
  lấy tỷ giá hiện tại → mỗi lần cần đều gọi network request.
- **Cách dataflow**: code xử lý mua hàng subscribe stream cập nhật tỷ giá từ trước,
  lưu tỷ giá hiện tại vào database cục bộ; khi xử lý mua hàng, chỉ cần query DB cục bộ.

→ Cách dataflow **nhanh hơn** (query network → query local database) và **bền vững
hơn** trước lỗi của service khác — "request mạng nhanh và tin cậy nhất chính là
không có request mạng nào cả!" Về bản chất, đây là **stream join** giữa event mua
hàng và event cập nhật tỷ giá. Join này **phụ thuộc thời gian**: nếu reprocess sau
này, cần biết tỷ giá lịch sử tại thời điểm mua (xem lại "Time dependence of joins").

## 2.5 Quan sát trạng thái dẫn xuất (Observing Derived State)

Hệ thống dataflow tạo ra dataset dẫn xuất (search index, materialized view, mô hình
dự đoán...) và giữ chúng cập nhật — gọi quá trình này là **write path**. Còn khi
serve request người dùng, đọc từ dataset dẫn xuất để trả về response — đó là
**read path**.

- **Write path**: phần công việc được **tính toán trước (precompute), thực hiện
  ngay lập tức (eager)** khi dữ liệu đến, bất kể có ai hỏi hay chưa. ~ eager evaluation.
- **Read path**: phần chỉ thực hiện **khi có người hỏi**. ~ lazy evaluation.
- Dataset dẫn xuất là nơi 2 con đường này **gặp nhau** — thể hiện đánh đổi giữa
  lượng công việc làm lúc ghi và lượng công việc làm lúc đọc.

### Materialized view và caching

Ví dụ full-text search index: write path cập nhật index; read path tìm từ khóa
trong index. Không có index → mọi query phải scan toàn bộ document (như `grep`) →
tốn kém khi dữ liệu lớn (ít việc ở write path, rất nhiều việc ở read path).

Ngược lại, có thể tưởng tượng **tính trước kết quả cho mọi query có thể có** — read
path gần như không việc gì, nhưng số query có thể là vô hạn (hoặc theo cấp số nhân
theo số từ trong corpus) → không khả thi.

**Trung dung**: tính trước kết quả cho một tập query phổ biến cố định — đây chính
là **cache** (hoặc gọi là materialized view, vì cần cập nhật khi có document mới
liên quan). → Index, cache, materialized view thực chất chỉ là **dịch chuyển ranh
giới giữa write path và read path**, cho phép làm nhiều việc hơn ở write path
(precompute) để tiết kiệm công sức ở read path. (Đây cũng chính là ví dụ "Case
Study: Social Network Home Timelines" ở Chương 2 — ranh giới này được vẽ khác nhau
cho celebrity so với người dùng thường.)

### Client trạng thái, hoạt động offline được

Trước đây trình duyệt web là **stateless client** — chỉ hữu ích khi có kết nối
internet. Nay single-page app (React, Elm...) và mobile app lưu nhiều state cục bộ,
cho phép làm việc offline, sync nền khi có mạng (xem lại "Sync Engines and
Local-First Software"). Có thể xem **state trên thiết bị như một cache của state
trên server**: pixel trên màn hình là materialized view của model object trong app;
model object là bản sao (replica) cục bộ của state trên datacenter từ xa.

### Đẩy thay đổi state đến client

Web truyền thống: trình duyệt chỉ đọc dữ liệu tại một thời điểm, không biết dữ liệu
đổi cho đến khi reload (RSS cũng chỉ là một dạng polling cơ bản). Giao thức mới
(**Server-Sent Events**, **WebSocket**) cho phép giữ kết nối mở, server **chủ động
đẩy** thay đổi đến client. → Về bản chất, đây là **mở rộng write path đến tận end
user**. Client khởi tạo lần đầu vẫn cần read path lấy state ban đầu, sau đó dựa vào
stream cập nhật liên tục từ server. Khi client offline, dùng cơ chế tương tự
**consumer offset** của message broker (đã bàn ở "Consumer offsets") để không bỏ
lỡ message khi kết nối lại.

### End-to-end event stream

Công cụ xây UI trạng thái (React, Elm) đã có khả năng tự cập nhật UI khi state đổi.
Có thể mở rộng mô hình này để server đẩy sự kiện thay đổi state thẳng vào pipeline
sự kiện phía client — tạo thành **write path xuyên suốt từ đầu đến cuối**: từ tương
tác trên một thiết bị → qua event log và các hệ dữ liệu dẫn xuất/stream processor →
đến tận UI trên thiết bị khác, có thể với độ trễ **dưới 1 giây**. Ứng dụng nhắn tin,
game online đã có kiến trúc "thời gian thực" kiểu này. Thách thức: giả định
stateless client + request/response đã ăn sâu vào database, thư viện, framework,
giao thức — cần **tư duy lại theo hướng publish/subscribe dataflow**.

### Đọc (reads) cũng là sự kiện

Có thể xem cả **request đọc** như một luồng sự kiện, gửi qua cùng stream processor
với luồng ghi — processor phản hồi sự kiện đọc bằng cách phát kết quả ra output
stream. Khi cả đọc và ghi đều là event đi qua cùng stream operator, thực chất đây là
**stream-table join** giữa luồng query đọc và database. Một request đọc một lần đi
qua join operator rồi bị quên ngay; một request subscribe là **join bền vững** với
sự kiện quá khứ và tương lai ở phía bên kia. Ghi log các request đọc còn giúp theo
dõi quan hệ nhân quả và nguồn gốc dữ liệu (data provenance) — ví dụ tái hiện những
gì người dùng đã thấy trước khi ra quyết định mua hàng.

### Xử lý dữ liệu đa shard

Với query chỉ chạm 1 shard, gửi qua stream có thể là quá mức cần thiết. Nhưng ý
tưởng này mở ra khả năng **thực thi phân tán các query phức tạp** cần kết hợp dữ
liệu từ nhiều shard, tận dụng hạ tầng định tuyến/sharding/join sẵn có của stream
processor (ví dụ: Storm's distributed RPC để đếm số người đã thấy một URL trên
mạng xã hội; đánh giá rủi ro gian lận dựa trên reputation score của IP/email/địa
chỉ, mỗi loại đều được shard riêng).
