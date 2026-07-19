# Phần 1 — Phân Tích Dự Đoán (Predictive Analytics)

Predictive analytics là lý do chính khiến người ta hào hứng với big data và AI —
nhưng cũng là mảng đầy **rủi ro đạo đức**. Dùng phân tích dữ liệu để dự báo thời
tiết hay sự lây lan dịch bệnh là một chuyện; dùng nó để dự đoán **liệu một phạm
nhân có tái phạm, người vay có vỡ nợ, khách hàng bảo hiểm có khiếu nại tốn kém hay
không** lại là chuyện khác — vì những dự đoán này **ảnh hưởng trực tiếp đến cuộc
sống của từng cá nhân**.

Các tổ chức (mạng thanh toán, ngân hàng, hãng bay, nhà tuyển dụng) tự nhiên muốn
**thận trọng**: chi phí bỏ lỡ cơ hội kinh doanh thấp, nhưng chi phí một khoản vay
xấu hoặc một nhân viên có vấn đề lại cao hơn nhiều — nên khi nghi ngờ, họ thường
chọn **"nói không."**

### "Nhà tù thuật toán" (algorithmic prison)

Khi ra quyết định tự động ngày càng phổ biến, một người bị thuật toán gắn nhãn "rủi
ro" (dù đúng hay sai) có thể hứng chịu **hàng loạt quyết định "không"** — bị loại
trừ có hệ thống khỏi việc làm, đi lại bằng máy bay, bảo hiểm, thuê nhà, dịch vụ tài
chính... Đây là sự hạn chế tự do cá nhân lớn đến mức được gọi là **"algorithmic
prison."** Trong khi hệ thống tư pháp hình sự ở các nước tôn trọng nhân quyền giả
định **vô tội cho đến khi chứng minh có tội**, hệ thống tự động có thể loại một
người khỏi xã hội **một cách hệ thống và tùy tiện, không cần bằng chứng, và khó
kháng cáo**.

## 1.1 Thiên kiến và Phân biệt đối xử (Bias and Discrimination)

Quyết định của thuật toán không nhất thiết tốt hơn hay tệ hơn quyết định của con
người — ai cũng có thiên kiến, kể cả khi cố gắng chống lại nó, và thực hành phân
biệt đối xử có thể trở thành **thể chế hóa về mặt văn hóa**. Có hy vọng rằng dựa
quyết định trên dữ liệu (thay vì đánh giá chủ quan, bản năng của con người) có thể
công bằng hơn, cho cơ hội tốt hơn với những người thường bị bỏ qua/bất lợi trong hệ
thống truyền thống.

Nhưng khi phát triển hệ thống predictive analytics/AI, ta **không chỉ tự động hóa**
quyết định của con người bằng cách viết luật lệ tường minh — ta để cho **các quy
tắc được suy ra từ dữ liệu**. Vấn đề: các mẫu hình học được **mờ đục (opaque)** —
dù dữ liệu chỉ ra tương quan, ta có thể không biết **vì sao**. Nếu input mang thiên
kiến hệ thống, hệ thống rất có thể sẽ **học và khuếch đại** thiên kiến đó trong output.

Nhiều nước có luật chống phân biệt đối xử cấm đối xử khác biệt dựa trên đặc điểm
được bảo vệ (chủng tộc, tuổi, giới tính, khuynh hướng tính dục, khuyết tật, tín
ngưỡng). Nhưng các đặc điểm khác của dữ liệu người dùng có thể **tương quan chặt**
với các đặc điểm được bảo vệ — ví dụ ở khu dân cư phân biệt chủng tộc, **mã bưu
điện hoặc thậm chí địa chỉ IP** có thể dự đoán khá tốt chủng tộc. Vì vậy tin rằng
thuật toán có thể nhận dữ liệu thiên kiến làm đầu vào rồi cho ra kết quả **công
bằng, vô tư** nghe có vẻ phi lý — nhưng niềm tin này lại thường ngầm ẩn trong tuyên
bố của những người ủng hộ ra quyết định dựa trên dữ liệu — thái độ này từng bị châm
biếm là **"machine learning giống như rửa tiền cho thiên kiến."**

**Predictive analytics chỉ đơn thuần ngoại suy từ quá khứ** — nếu quá khứ mang tính
phân biệt đối xử, hệ thống sẽ **mã hóa và khuếch đại** sự phân biệt đó. Muốn tương
lai tốt hơn quá khứ, cần **trí tưởng tượng đạo đức (moral imagination)** — điều chỉ
con người mới có thể cung cấp. **Dữ liệu và mô hình nên là công cụ của chúng ta,
không phải chủ nhân của chúng ta.**

## 1.2 Trách nhiệm và Giải trình (Responsibility and Accountability)

Ra quyết định tự động đặt ra câu hỏi về **trách nhiệm và giải trình**. Nếu con
người mắc lỗi, họ có thể bị quy trách nhiệm, và người bị ảnh hưởng có thể kháng
cáo. Thuật toán cũng mắc lỗi — nhưng **ai chịu trách nhiệm khi nó sai**? Xe tự lái
gây tai nạn — ai chịu trách nhiệm? Thuật toán chấm điểm tín dụng phân biệt đối xử
hệ thống theo chủng tộc/tôn giáo — có cách nào khắc phục? Nếu quyết định của hệ
thống ML bị xét xử lại trước tòa, bạn có thể giải thích cho thẩm phán **thuật toán
đã ra quyết định như thế nào** không? **Con người không được phép trốn tránh trách
nhiệm bằng cách đổ lỗi cho thuật toán.**

**Điểm tín dụng truyền thống** — dù gây khó khăn nếu thấp — ít nhất dựa trên các
sự kiện **liên quan trực tiếp** đến lịch sử vay của một người, và sai sót trong hồ
sơ _có thể_ được sửa (dù các cơ quan tín dụng thường không làm điều này dễ dàng).
Ngược lại, các thuật toán chấm điểm dựa trên machine learning thường dùng **phạm
vi input rộng hơn nhiều, mờ đục hơn nhiều**, khiến việc hiểu vì sao một quyết định
cụ thể được đưa ra, và liệu ai đó có bị đối xử bất công/phân biệt hay không, trở
nên khó khăn hơn.

Điểm tín dụng tóm tắt câu hỏi **"Bạn đã hành xử như thế nào trong quá khứ?"**, còn
predictive analytics thường hoạt động dựa trên câu hỏi **"Ai giống bạn, và người
giống bạn đã hành xử như thế nào trong quá khứ?"** — việc so sánh hành vi với người
khác ngầm chứa **rập khuôn hóa (stereotyping)** con người, ví dụ dựa trên nơi họ
sống (một proxy sát với chủng tộc và tầng lớp kinh tế-xã hội). Điều gì xảy ra với
người bị xếp nhầm nhóm ("wrong bucket")? Hơn nữa, nếu một quyết định sai vì dữ liệu
lỗi, việc kháng cáo gần như **bất khả thi**.

Phần lớn dữ liệu mang tính **thống kê** — nghĩa là dù phân phối xác suất tổng thể
đúng, **từng trường hợp cá nhân vẫn có thể sai**. Ví dụ: nếu tuổi thọ trung bình ở
nước bạn là 80, không có nghĩa bạn được kỳ vọng qua đời đúng sinh nhật thứ 80. Từ
trung bình và phân phối xác suất, không thể nói nhiều về tuổi thọ của **một cá
nhân cụ thể**. Tương tự, output của hệ thống dự đoán mang tính xác suất, và hoàn
toàn có thể **sai với từng trường hợp cụ thể**.

**Niềm tin mù quáng vào sự tối thượng của dữ liệu** trong việc ra quyết định không
chỉ ảo tưởng mà còn **thực sự nguy hiểm**. Khi ra quyết định dựa trên dữ liệu ngày
càng phổ biến, ta cần tìm cách: tránh củng cố thiên kiến sẵn có; làm thuật toán
**giải trình được và minh bạch**; và sửa chữa khi chúng chắc chắn sẽ mắc lỗi.

Ta cũng cần tìm cách hiện thực hóa **tiềm năng tích cực** của dữ liệu và ngăn nó bị
dùng để hại người. Ví dụ: phân tích có thể tiết lộ đặc điểm tài chính/xã hội của
cuộc sống con người — một mặt, có thể dùng để **tập trung hỗ trợ** cho người cần
nhất; mặt khác, đôi khi bị các doanh nghiệp trục lợi dùng để **nhận diện người dễ
tổn thương** và bán cho họ sản phẩm rủi ro như khoản vay lãi cao hay bằng cấp đại
học vô giá trị.

## 1.3 Vòng lặp phản hồi (Feedback Loops)

Ngay cả với ứng dụng dự đoán ít tác động trực tiếp đến người hơn (như hệ thống gợi
ý), vẫn có vấn đề khó phải đối mặt. Khi dịch vụ giỏi dự đoán nội dung người dùng
muốn xem, chúng có thể chỉ cho người dùng thấy **những quan điểm họ đã đồng ý sẵn**
→ dẫn đến **buồng vọng âm (echo chamber)**, nơi khuôn mẫu, thông tin sai lệch, và
sự phân cực có thể sinh sôi. Tác động của echo chamber mạng xã hội lên các chiến
dịch bầu cử đã được ghi nhận rõ.

Khi predictive analytics ảnh hưởng đến cuộc sống con người, vấn đề đặc biệt nghiêm
trọng nảy sinh từ **vòng lặp phản hồi tự củng cố (self-reinforcing feedback loops)**:

> **Ví dụ**: Nhà tuyển dụng dùng điểm tín dụng để đánh giá ứng viên. Bạn có thể là
> người làm việc tốt với điểm tín dụng tốt, nhưng bất ngờ gặp khó khăn tài chính vì
> một biến cố ngoài tầm kiểm soát. Khi bạn trễ hạn thanh toán, điểm tín dụng giảm →
> bạn khó tìm việc hơn → thất nghiệp đẩy bạn vào nghèo đói → điểm tín dụng càng tệ
> hơn → càng khó tìm việc hơn. Một **vòng xoáy đi xuống** do các giả định độc hại,
> được ngụy trang bởi vẻ ngoài chặt chẽ toán học và dữ liệu.

> **Ví dụ khác**: các nhà kinh tế học phát hiện khi trạm xăng ở Đức áp dụng định
> giá thuật toán, cạnh tranh giảm và giá cho người tiêu dùng tăng lên — vì các
> thuật toán **học cách thông đồng** với nhau.

Ta không phải lúc nào cũng dự đoán được khi nào vòng lặp phản hồi như vậy sẽ xảy
ra. Tuy nhiên, nhiều hệ quả có thể được dự đoán bằng cách **suy nghĩ về toàn bộ hệ
thống** (không chỉ phần được lập trình hóa, mà cả con người tương tác với nó) — cách
tiếp cận gọi là **systems thinking**. Ta có thể cố hiểu hệ thống phân tích dữ liệu
phản ứng thế nào với các hành vi/cấu trúc/đặc điểm khác nhau. Nó có **củng cố và
khuếch đại** sự khác biệt sẵn có giữa người với người (làm người giàu càng giàu,
người nghèo càng nghèo) hay nó cố gắng **chống lại bất công**? Ngay cả với ý định
tốt nhất, ta vẫn phải cảnh giác với khả năng có **hệ quả ngoài ý muốn**.
