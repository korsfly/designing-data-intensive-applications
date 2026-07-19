# Phần 2 — Quyền Riêng Tư & Theo Dõi (Privacy and Tracking)

Ngoài vấn đề của predictive analytics (dùng dữ liệu để ra quyết định tự động), còn
có vấn đề đạo đức ở chính **việc thu thập dữ liệu**. Mối quan hệ giữa tổ chức thu
thập dữ liệu và người bị thu thập dữ liệu là gì?

Khi hệ thống chỉ lưu dữ liệu người dùng **chủ động nhập vào** vì họ muốn hệ thống
xử lý nó theo cách nào đó, hệ thống đang **phục vụ** người dùng — người dùng là
khách hàng. Nhưng khi hoạt động của người dùng bị **theo dõi và ghi lại như một hệ
quả phụ (side effect)** của việc họ làm việc khác, mối quan hệ trở nên mập mờ hơn —
dịch vụ không còn chỉ làm những gì người dùng yêu cầu; nó có **lợi ích riêng**, có
thể xung đột với lợi ích người dùng.

Theo dõi dữ liệu hành vi ngày càng quan trọng với nhiều tính năng hướng người dùng:
theo dõi kết quả tìm kiếm nào được click giúp cải thiện xếp hạng; gợi ý ("người
thích X cũng thích Y") giúp người dùng khám phá điều thú vị; A/B test và phân tích
user flow giúp cải thiện UI. Các tính năng này cần một mức độ theo dõi hành vi nhất
định, và **người dùng được hưởng lợi** từ chúng.

Tuy nhiên, tùy mô hình kinh doanh, việc theo dõi thường **không dừng lại ở đó**. Nếu
dịch vụ được tài trợ bằng quảng cáo, **nhà quảng cáo mới là khách hàng thực sự**, và
lợi ích của người dùng trở thành thứ yếu. Dữ liệu theo dõi trở nên chi tiết hơn,
phân tích sâu rộng hơn, và dữ liệu được giữ lại lâu dài để xây **hồ sơ chi tiết** về
từng người phục vụ mục đích marketing. Lúc này, mối quan hệ giữa công ty và người
dùng bắt đầu trông rất khác — có thể mô tả chính xác hơn bằng một từ mang hàm ý u
ám hơn: **giám sát (surveillance)**.

## 2.1 Giám sát (Surveillance)

**Thí nghiệm tư duy**: thử thay từ "dữ liệu" (data) bằng "giám sát" (surveillance)
trong các cụm từ quen thuộc, xem chúng còn nghe hay không: _"Trong tổ chức hướng
giám sát của chúng tôi, chúng tôi thu thập các luồng giám sát thời gian thực và lưu
vào kho giám sát. Các nhà khoa học giám sát của chúng tôi dùng phân tích nâng cao và
xử lý giám sát để rút ra insight mới."_

Trong nỗ lực để phần mềm "nuốt chửng thế giới," chúng ta đã xây dựng **hạ tầng giám
sát hàng loạt lớn nhất từng thấy**. Ta đang nhanh chóng tiến tới thế giới mà **mọi
không gian có người ở đều chứa ít nhất một micro kết nối internet** — dưới dạng
smartphone, smart TV, thiết bị trợ lý giọng nói, máy theo dõi em bé, thậm chí đồ
chơi trẻ em dùng nhận diện giọng nói qua cloud. Nhiều thiết bị trong số này có **hồ
sơ bảo mật rất tệ**.

Điều mới so với trước đây: số hóa đã khiến việc thu thập **lượng lớn dữ liệu về con
người** trở nên dễ dàng. Giám sát vị trí và di chuyển, quan hệ xã hội và giao tiếp,
mua sắm và thanh toán, dữ liệu sức khỏe — gần như **không thể tránh khỏi**. Một tổ
chức giám sát có thể biết về một người **nhiều hơn** cả người đó biết về chính mình
— ví dụ phát hiện bệnh tật hoặc vấn đề kinh tế trước khi người đó nhận ra.

Ngay cả những chế độ toàn trị, đàn áp nhất trong lịch sử cũng chỉ **mơ ước** có thể
đặt micro trong mọi phòng và buộc mọi người luôn mang theo thiết bị theo dõi vị
trí/di chuyển. Vậy mà lợi ích ta nhận được từ công nghệ số lớn đến mức ta **tự
nguyện chấp nhận** trạng thái giám sát toàn diện này — khác biệt chỉ là dữ liệu
được thu thập bởi **doanh nghiệp** để cung cấp dịch vụ, thay vì cơ quan chính phủ
tìm cách kiểm soát.

Không phải mọi thu thập dữ liệu đều đủ điều kiện gọi là giám sát, nhưng xem xét nó
dưới góc độ này giúp hiểu rõ hơn mối quan hệ của ta với bên thu thập dữ liệu. Vì
sao ta có vẻ vui vẻ chấp nhận bị giám sát bởi doanh nghiệp? Có lẽ vì cảm thấy "mình
không có gì để giấu" — nói cách khác, hoàn toàn phù hợp với cấu trúc quyền lực hiện
tại, không phải nhóm thiểu số bị gạt ra lề, không phải lo bị đàn áp. **Không phải
ai cũng may mắn như vậy.** Hoặc có lẽ vì mục đích có vẻ vô hại — không phải cưỡng
ép công khai, chỉ là gợi ý tốt hơn và marketing cá nhân hóa hơn. Nhưng kết hợp với
predictive analytics ở phần trước, ranh giới đó trở nên **mờ nhạt hơn nhiều**.

Ta đã thấy dữ liệu hành vi lái xe (theo dõi bởi xe hơi mà không có sự đồng thuận
của tài xế) ảnh hưởng đến phí bảo hiểm; bảo hiểm y tế phụ thuộc vào việc đeo thiết
bị theo dõi thể chất. Khi giám sát được dùng để ra quyết định ảnh hưởng đến những
khía cạnh quan trọng của cuộc sống (bảo hiểm, việc làm), nó bắt đầu trông **ít vô
hại hơn**. Phân tích dữ liệu còn có thể tiết lộ những điều **xâm phạm đáng ngạc
nhiên** — ví dụ cảm biến chuyển động trong smartwatch/thiết bị theo dõi thể chất có
thể được dùng để **suy ra những gì bạn đang gõ** (như mật khẩu) với độ chính xác
khá cao. Độ chính xác cảm biến và thuật toán phân tích chỉ ngày càng tốt hơn.

## 2.2 Sự đồng thuận và tự do lựa chọn (Consent and Freedom of Choice)

Có thể lập luận rằng người dùng **tự nguyện chọn** sử dụng dịch vụ theo dõi hoạt
động của họ, đồng ý điều khoản dịch vụ và chính sách quyền riêng tư. Nhưng lập luận
này có vấn đề:

**Thứ nhất**, cần hỏi _vì sao_ việc theo dõi là cần thiết. Một số hình thức theo
dõi trực tiếp cải thiện tính năng cho người dùng (ví dụ: theo dõi click-through
rate cải thiện xếp hạng tìm kiếm). Nhưng khi theo dõi tương tác để **gợi ý nội
dung** hoặc xây **hồ sơ người dùng cho quảng cáo**, có thực sự vì lợi ích người
dùng hay chỉ vì quảng cáo trả tiền cho dịch vụ?

**Thứ hai**, hầu hết người dùng **hiểu biết rất ít** về dữ liệu họ đang cung cấp,
cách nó được lưu giữ và xử lý — và hầu hết chính sách quyền riêng tư **che giấu**
nhiều hơn là làm rõ. Không hiểu chuyện gì xảy ra với dữ liệu của mình, người dùng
**không thể đưa ra sự đồng thuận có ý nghĩa**. Hơn nữa, dữ liệu của một người dùng
thường tiết lộ thông tin về **những người khác** không phải là người dùng dịch vụ
và chưa từng đồng ý điều khoản nào. Các bộ dữ liệu dẫn xuất (đã bàn ở các chương
trước) — nơi dữ liệu toàn bộ user base kết hợp với theo dõi hành vi và nguồn dữ
liệu bên ngoài — chính là loại dữ liệu mà người dùng **không thể hiểu một cách có
ý nghĩa**.

Hơn nữa, dữ liệu được trích xuất từ người dùng qua một **quy trình một chiều**,
không phải mối quan hệ có đi có lại thực sự hay trao đổi giá trị công bằng. Không
có đối thoại, không có lựa chọn cho người dùng thương lượng họ cung cấp bao nhiêu
dữ liệu để đổi lấy dịch vụ gì — mối quan hệ **bất đối xứng, một chiều**; điều khoản
do dịch vụ đặt ra, không phải do người dùng.

**GDPR (Liên minh châu Âu)** yêu cầu sự đồng thuận phải **"tự nguyện, cụ thể, được
thông tin đầy đủ, và không mập mờ"**, và người dùng phải có khả năng **"từ chối
hoặc rút lại đồng thuận mà không chịu thiệt hại"** — nếu không, không được xem là
"tự nguyện." Mọi yêu cầu đồng thuận phải viết bằng "ngôn ngữ dễ hiểu, rõ ràng, dễ
tiếp cận," và **"im lặng, ô đã tick sẵn, hoặc không hành động không cấu thành đồng
thuận."**

Đồng thuận không phải cơ sở duy nhất để xử lý dữ liệu cá nhân hợp pháp theo GDPR —
còn có các cơ sở khác như tuân thủ luật khác, hoặc bảo vệ tính mạng ai đó. Ngoài ra,
cơ sở **"lợi ích chính đáng"** cho phép một số cách sử dụng dữ liệu nhất định (ví
dụ: phòng chống gian lận). Dù vậy, đồng thuận vẫn là cơ sở phổ biến nhất cho việc
xử lý dữ liệu cá nhân trong các dịch vụ internet.

Có thể lập luận rằng người dùng không đồng thuận giám sát chỉ cần **không dùng dịch
vụ**. Nhưng lựa chọn này **không hề tự do**. Nếu một dịch vụ phổ biến đến mức được
**"coi là thiết yếu cho sự tham gia xã hội cơ bản,"** thì không thể kỳ vọng người ta
từ chối sử dụng nó — việc dùng nó **trên thực tế là bắt buộc**. Ví dụ, ở hầu hết
cộng đồng phương Tây, mang theo smartphone, dùng mạng xã hội để giao tiếp, dùng
Google để tìm thông tin đã trở thành **chuẩn mực**. Đặc biệt khi dịch vụ có hiệu
ứng mạng lưới, có **chi phí xã hội** khi ai đó chọn không dùng nó.

Từ chối dùng dịch vụ vì chính sách theo dõi **nói dễ hơn làm**. Các nền tảng này
được thiết kế đặc biệt để **giữ chân người dùng**, nhiều dùng cơ chế game và chiến
thuật phổ biến trong cờ bạc để khiến người dùng quay lại. Ngay cả khi vượt qua được
điều đó, từ chối tham gia chỉ là lựa chọn khả thi với **số ít người có đặc quyền**
đủ thời gian và hiểu biết để đọc hiểu chính sách quyền riêng tư, và đủ khả năng
chấp nhận bỏ lỡ cơ hội xã hội/nghề nghiệp nếu không tham gia dịch vụ. Với người
**kém đặc quyền hơn**, không có **tự do lựa chọn thực chất** — giám sát trở nên
**không thể tránh khỏi**.

## 2.3 Quyền riêng tư và Việc sử dụng dữ liệu (Privacy and Use of Data)

Đôi khi có người khẳng định **"quyền riêng tư đã chết"** với lý do một số người
dùng sẵn sàng đăng đủ thứ về cuộc sống lên mạng xã hội, đôi khi tầm thường, đôi khi
rất riêng tư. Tuy nhiên, khẳng định này **sai**, dựa trên hiểu lầm về từ _privacy_.

**Có quyền riêng tư không đồng nghĩa với giữ bí mật mọi thứ; nó nghĩa là có tự do
lựa chọn** tiết lộ gì cho ai, công khai gì, giữ bí mật gì. Quyền riêng tư là **quyền
quyết định** — cho phép mỗi người quyết định vị trí của họ trên phổ giữa bí mật và
minh bạch, tùy hoàn cảnh. Đó là khía cạnh quan trọng của **tự do và quyền tự quyết**
của một người.

Ví dụ: người mắc bệnh hiếm có thể rất vui lòng chia sẻ dữ liệu y tế riêng tư với
nhà nghiên cứu nếu điều đó giúp phát triển phương pháp điều trị. Nhưng người này
phải có **lựa chọn** ai được truy cập dữ liệu và với mục đích gì. Nếu thông tin về
bệnh của họ có thể cản trở khả năng tiếp cận bảo hiểm y tế hay việc làm, người này
sẽ **thận trọng hơn nhiều** về việc chia sẻ.

Khi dữ liệu bị trích xuất từ người dùng qua hạ tầng giám sát, quyền riêng tư
**không nhất thiết bị xói mòn mà bị chuyển giao** cho bên thu thập dữ liệu. Các
công ty thu thập dữ liệu về cơ bản nói: _"Hãy tin chúng tôi sẽ làm điều đúng với
dữ liệu của bạn"_ — nghĩa là quyền quyết định tiết lộ/giữ bí mật đã **chuyển từ cá
nhân sang công ty**.

Đến lượt mình, các công ty chọn **giữ kín** phần lớn kết quả giám sát này, vì tiết
lộ ra sẽ bị coi là "rợn người" (creepy) và tổn hại mô hình kinh doanh (vốn dựa vào
việc biết nhiều về người dùng hơn công ty khác). Thông tin thân mật về người dùng
chỉ được tiết lộ **gián tiếp** — ví dụ dưới dạng công cụ nhắm mục tiêu quảng cáo cho
nhóm người cụ thể (như những người mắc một bệnh cụ thể).

Ngay cả khi từng người dùng cụ thể không thể bị nhận diện lại từ nhóm mục tiêu của
một quảng cáo, họ vẫn **mất quyền tự quyết** về việc tiết lộ thông tin thân mật.
**Không phải người dùng quyết định** điều gì được tiết lộ cho ai dựa trên sở thích
cá nhân — mà là **công ty** thực thi quyền riêng tư đó với mục tiêu **tối đa hóa
lợi nhuận**.

Nhiều công ty muốn tránh bị coi là "rợn người," né tránh câu hỏi việc thu thập dữ
liệu của họ xâm phạm đến mức nào, thay vào đó tập trung **quản lý nhận thức** của
người dùng. Và ngay cả nhận thức này thường được quản lý kém — ví dụ: một điều gì
đó có thể đúng sự thật, nhưng nếu gợi lại ký ức đau buồn, người dùng có thể không
muốn bị nhắc nhở về nó. Với bất kỳ loại dữ liệu nào, ta nên lường trước khả năng nó
sai, không mong muốn, hoặc không phù hợp theo cách nào đó, và cần xây **cơ chế xử
lý các thất bại đó**. Điều gì "không mong muốn" hay "không phù hợp" tất nhiên tùy
thuộc phán đoán của con người; thuật toán vô cảm với những khái niệm này trừ khi ta
lập trình rõ ràng để tôn trọng nhu cầu con người. Là kỹ sư xây các hệ thống này, ta
phải **khiêm tốn**, chấp nhận và lên kế hoạch cho những thiếu sót như vậy.

Cài đặt quyền riêng tư cho phép người dùng kiểm soát dữ liệu nào người khác thấy
được là điểm khởi đầu để trao lại một phần quyền kiểm soát cho người dùng. Tuy
nhiên, bất kể cài đặt thế nào, **bản thân dịch vụ vẫn có toàn quyền truy cập dữ
liệu**, tự do sử dụng nó theo bất kỳ cách nào được phép bởi chính sách quyền riêng
tư. Ngay cả khi dịch vụ hứa không bán dữ liệu cho bên thứ ba, họ thường tự cấp cho
mình quyền **không giới hạn** xử lý và phân tích dữ liệu nội bộ, thường vượt xa
những gì hiển thị công khai cho người dùng.

Kiểu **chuyển giao quy mô lớn quyền riêng tư** từ cá nhân sang doanh nghiệp này là
**chưa từng có tiền lệ trong lịch sử**. Giám sát luôn tồn tại, nhưng trước đây nó
tốn kém và thủ công, không phải quy mô lớn và tự động. Quan hệ tin cậy luôn tồn tại
(ví dụ giữa bệnh nhân và bác sĩ, giữa bị cáo và luật sư), nhưng trong các trường
hợp này, việc sử dụng dữ liệu bị điều chỉnh chặt chẽ bởi các ràng buộc đạo đức,
pháp lý và quy định. Dịch vụ internet đã khiến việc **thu thập lượng khổng lồ thông
tin nhạy cảm mà không có sự đồng thuận thực chất**, và sử dụng nó ở **quy mô lớn**
mà người dùng không hiểu chuyện gì đang xảy ra với dữ liệu riêng tư của họ, trở nên
dễ dàng hơn nhiều.

## 2.4 Dữ liệu như Tài sản và Quyền lực (Data as Assets and Power)

Vì dữ liệu hành vi là sản phẩm phụ của việc người dùng tương tác với dịch vụ, đôi
khi nó được gọi là **"khí thải dữ liệu" (data exhaust)** — ngụ ý dữ liệu là chất
thải vô giá trị. Nhìn theo cách này, phân tích hành vi và dự đoán có thể được xem
như một hình thức **tái chế**, trích xuất giá trị từ thứ lẽ ra bị vứt bỏ.

Nhưng **nhìn theo hướng ngược lại mới chính xác hơn**. Về mặt kinh tế, nếu quảng
cáo nhắm mục tiêu là thứ trả tiền cho dịch vụ, hoạt động của người dùng tạo ra dữ
liệu hành vi có thể được coi là một hình thức **lao động**. Thậm chí có thể lập
luận xa hơn rằng ứng dụng mà người dùng tương tác chỉ đơn thuần là **phương tiện dụ
dỗ** người dùng đưa ngày càng nhiều thông tin cá nhân vào hạ tầng giám sát. Sự sáng
tạo đáng yêu của con người và các mối quan hệ xã hội thường được thể hiện qua dịch
vụ trực tuyến bị **cỗ máy trích xuất dữ liệu khai thác một cách hoài nghi**.

Dữ liệu cá nhân là **tài sản có giá trị**, minh chứng bởi sự tồn tại của các **nhà
môi giới dữ liệu (data broker)** hoạt động trong bí mật, mua, tổng hợp, phân tích,
và bán lại dữ liệu cá nhân, chủ yếu cho mục đích marketing. Startup được định giá
theo số lượng người dùng, hay "eyeballs" — tức là **khả năng giám sát** của họ.

Vì dữ liệu có giá trị, nhiều bên muốn có nó. Công ty đương nhiên muốn — đó là lý do
họ thu thập nó ngay từ đầu. Nhưng **chính phủ cũng muốn**, và có thể tìm cách có
được nó qua giao dịch bí mật, cưỡng ép, buộc pháp lý, hoặc đơn giản là trộm cắp. Khi
công ty phá sản, dữ liệu cá nhân họ thu thập là một trong những **tài sản được bán
đi**. Và vì dữ liệu khó bảo mật, các vụ **rò rỉ (breach)** xảy ra thường xuyên đến
mức đáng lo ngại.

Những quan sát này khiến các nhà phê bình gọi dữ liệu không chỉ là tài sản, mà là
**"tài sản độc hại"** hoặc ít nhất là **"vật liệu nguy hiểm."** Có lẽ dữ liệu không
phải vàng mới hay dầu mỏ mới, mà là **uranium mới**. Ngay cả khi ta nghĩ mình có
khả năng ngăn chặn lạm dụng dữ liệu, mỗi khi thu thập nó, ta cần **cân bằng lợi ích
với rủi ro** dữ liệu rơi vào tay kẻ xấu. Hệ thống máy tính có thể bị tội phạm hoặc
tình báo nước ngoài thù địch xâm nhập, dữ liệu có thể bị rò rỉ bởi người trong nội
bộ, công ty có thể rơi vào tay ban lãnh đạo vô đạo đức không chia sẻ giá trị của ta,
hoặc quốc gia có thể bị một chế độ không ngần ngại **buộc ta giao nộp dữ liệu** tiếp quản.

Như quan sát đó gợi ý, khi thu thập dữ liệu, ta cần cân nhắc **không chỉ môi trường
chính trị hiện tại, mà mọi chính phủ có thể có trong tương lai**. Không có gì đảm
bảo mọi chính phủ được bầu trong tương lai sẽ tôn trọng nhân quyền và tự do dân sự
— như Bruce Schneier nhận xét: **"Việc lắp đặt các công nghệ có thể một ngày nào đó
tạo điều kiện cho một nhà nước cảnh sát là vệ sinh công dân kém."**

**"Kiến thức là quyền lực"** — câu ngạn ngữ cũ. Và hơn nữa, **"Giám sát người khác
trong khi tránh né bị giám sát chính mình là một trong những hình thức quyền lực
quan trọng nhất."** Đây là lý do các chính phủ toàn trị muốn giám sát: nó cho họ
**quyền lực kiểm soát dân số**. Dù các công ty công nghệ ngày nay không công khai
tìm kiếm quyền lực chính trị, dữ liệu và kiến thức họ đã tích lũy — phần lớn một
cách âm thầm, ngoài sự giám sát công khai — vẫn cho họ **rất nhiều quyền lực** đối
với cuộc sống của chúng ta.

## 2.5 Nhớ về Cách mạng Công nghiệp (Remembering the Industrial Revolution)

Dữ liệu là đặc điểm định hình **kỷ nguyên thông tin**. Internet, lưu trữ và xử lý dữ
liệu, tự động hóa bằng phần mềm đang tác động lớn đến kinh tế toàn cầu và xã hội
loài người. Khi đời sống và tổ chức xã hội hàng ngày của ta đã bị công nghệ thông
tin thay đổi, và có thể tiếp tục thay đổi mạnh mẽ trong những thập kỷ tới, so sánh
với **Cách mạng Công nghiệp** là điều tự nhiên.

Cách mạng Công nghiệp ra đời từ những tiến bộ lớn về công nghệ và nông nghiệp, mang
lại tăng trưởng kinh tế bền vững và cải thiện đáng kể mức sống về lâu dài — nhưng
cũng đi kèm những **vấn đề lớn**. Ô nhiễm không khí (do khói và quá trình hóa học)
và nước (từ chất thải công nghiệp và sinh hoạt) rất tồi tệ. Chủ nhà máy sống xa hoa,
trong khi công nhân đô thị thường sống trong nhà ở chật chội, mất vệ sinh, làm việc
giờ dài trong điều kiện khắc nghiệt. **Lao động trẻ em** phổ biến, bao gồm công việc
nguy hiểm, lương thấp trong hầm mỏ.

Phải mất rất lâu trước khi các biện pháp bảo vệ được thiết lập: quy định bảo vệ môi
trường, quy trình an toàn nơi làm việc, luật cấm lao động trẻ em, kiểm tra sức khỏe
thực phẩm. Chắc chắn chi phí kinh doanh tăng lên khi nhà máy không còn được phép đổ
chất thải xuống sông, bán thực phẩm nhiễm bẩn, hay bóc lột công nhân. Nhưng **toàn
xã hội được hưởng lợi rất lớn** từ các quy định này, và ít ai trong chúng ta muốn
quay lại thời trước đó.

Giống như Cách mạng Công nghiệp có mặt tối cần được quản lý, quá trình chuyển đổi
sang kỷ nguyên thông tin của ta có những vấn đề lớn cần đối mặt và giải quyết. Việc
thu thập và sử dụng dữ liệu là một trong những vấn đề đó. Theo lời Bruce Schneier:

> _"Dữ liệu là vấn đề ô nhiễm của kỷ nguyên thông tin, và bảo vệ quyền riêng tư là
> thách thức môi trường. Gần như mọi máy tính đều sản sinh ra thông tin. Nó tồn
> đọng lại, thối rữa dần. Cách chúng ta xử lý nó — cách ta chứa nó và cách ta xử lý
> nó — là trung tâm của sức khỏe nền kinh tế thông tin của chúng ta. Giống như hôm
> nay ta nhìn lại những thập kỷ đầu của thời đại công nghiệp và tự hỏi làm sao tổ
> tiên ta có thể phớt lờ ô nhiễm trong cơn vội vã xây dựng thế giới công nghiệp,
> con cháu ta sẽ nhìn lại chúng ta trong những thập kỷ đầu của thời đại thông tin và
> phán xét ta về cách ta đối mặt với thách thức của việc thu thập và lạm dụng dữ
> liệu. Chúng ta nên cố gắng làm cho họ tự hào."_

## 2.6 Luật pháp và Tự điều chỉnh (Legislation and Self-Regulation)

Luật bảo vệ dữ liệu có thể giúp bảo tồn quyền của cá nhân. Ví dụ, **GDPR** quy định
dữ liệu cá nhân phải **"được thu thập cho các mục đích cụ thể, rõ ràng và hợp pháp,
và không được xử lý thêm theo cách không tương thích với các mục đích đó,"** và
phải **"đầy đủ, liên quan, và giới hạn trong những gì cần thiết"** cho mục đích xử lý.

Tuy nhiên, nguyên tắc **tối giản hóa dữ liệu (data minimization)** này đi ngược trực
tiếp với triết lý của big data — vốn là **tối đa hóa** thu thập dữ liệu, kết hợp dữ
liệu thu thập được với các bộ dữ liệu khác, và thử nghiệm/khám phá để tạo ra insight
mới. Khám phá nghĩa là dùng dữ liệu cho **mục đích không lường trước** — điều mà
GDPR nói là ngược lại với mục đích "cụ thể và rõ ràng" mà dữ liệu phải được thu
thập cho. Dù quy định này có tác động phần nào đến ngành quảng cáo trực tuyến, nó
đã được **thực thi yếu**, và dường như không dẫn đến nhiều thay đổi văn hóa/thực
hành trong toàn ngành công nghệ rộng lớn hơn.

Các công ty thu thập nhiều dữ liệu về con người nhìn chung phản đối quy định vì
xem đó là **gánh nặng và cản trở đổi mới**. Ở một mức độ nào đó, sự phản đối này có
cơ sở. Ví dụ: chia sẻ dữ liệu y tế tạo ra rủi ro rõ ràng cho quyền riêng tư nhưng
cũng có tiềm năng lớn: bao nhiêu ca tử vong có thể được ngăn chặn nếu phân tích dữ
liệu giúp chẩn đoán tốt hơn hoặc tìm phương pháp điều trị tốt hơn? Quy định quá
mức có thể ngăn cản những đột phá như vậy. Khó **cân bằng** giữa cơ hội tiềm năng và
rủi ro.

Về cơ bản, ta cần một **sự thay đổi văn hóa** trong ngành công nghệ đối với dữ liệu
cá nhân. Ta nên **ngừng xem người dùng như những chỉ số cần tối ưu hóa**, và nhớ
rằng họ là **con người** xứng đáng được tôn trọng, có nhân phẩm, và quyền tự quyết.
Ta nên **tự điều chỉnh** thực hành thu thập và xử lý dữ liệu của mình để xây dựng
và duy trì **niềm tin** của những người phụ thuộc vào phần mềm của ta. Và ta nên tự
mình có trách nhiệm **giáo dục người dùng cuối** về cách dữ liệu của họ được sử
dụng, thay vì để họ mù mờ.

Ta nên cho phép mỗi cá nhân duy trì quyền riêng tư của họ (tức là quyền kiểm soát
dữ liệu của chính mình) và không đánh cắp quyền kiểm soát đó khỏi họ thông qua giám
sát. **Quyền kiểm soát dữ liệu cá nhân của chúng ta giống như môi trường tự nhiên
của một công viên quốc gia**: nếu ta không chủ động bảo vệ và chăm sóc nó, nó sẽ bị
phá hủy. Đó sẽ là **"bi kịch của tài sản chung" (tragedy of the commons)**, và tất
cả chúng ta sẽ tệ hơn vì điều đó. **Giám sát phổ quát không phải là điều tất yếu.
Chúng ta vẫn có thể ngăn chặn nó.**

Bước đầu tiên: ta không nên giữ dữ liệu mãi mãi, mà nên **xóa nó ngay khi không còn
cần thiết**, và **tối giản hóa** những gì thu thập ngay từ đầu. Dữ liệu bạn không
có là dữ liệu **không thể bị rò rỉ, đánh cắp, hoặc bị chính phủ buộc phải giao
nộp**. Nhìn chung, cần thay đổi văn hóa và thái độ. Là những người làm trong lĩnh
vực công nghệ, **nếu chúng ta không cân nhắc tác động xã hội của công việc mình,
chúng ta đang không làm tròn công việc của mình.**
