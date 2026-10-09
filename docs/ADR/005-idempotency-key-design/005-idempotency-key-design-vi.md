# ADR-005: Idempotency Key Design

**Ngày viết:** 08/10/2026
**Trạng thái:** Đã áp dụng

---

## 1. Bối cảnh

**Bối cảnh nghiệp vụ:**
Khách hàng có thể bấm nút gửi yêu cầu tạo đơn nhiều lần, hoặc sự cố mạng có thể khiến máy chủ nhận request trùng lặp.

**Kịch bản rủi ro nếu không có cơ chế phòng thủ:**
Nếu các request trùng lặp lọt vào hệ thống, sai lệch dữ liệu xấu nhất xảy ra ở các bảng Order, OrderItem và Availability: hệ thống tạo thêm đơn hàng "ma" và trừ slot bánh tương ứng. Khi đó, nếu khách thấy cửa hàng đã hết bánh thì doanh thu bị sụt giảm. Ngoài ra, nếu thợ làm bánh nhận phải đơn hàng "ma" này, họ sẽ làm dư bánh so với thực tế.

**Ràng buộc bất biến:**
Với cùng một Idempotency Key (cùng một ý định đặt hàng), API phải đảm bảo chỉ thực hiện một lần và trả về kết quả của lần đó.

Khi người dùng làm mới trang hoặc gửi lại request với key cũ, Guard chỉ kiểm tra trạng thái của key trong Redis. Nếu key đang là `PENDING`, request bị từ chối với mã `409`. Nếu key là `SUCCESS:<orderId>`, Guard trả `409` kèm mã đơn để frontend tải đơn và điều hướng theo phương thức thanh toán.

**Xác thực trạng thái (Đã áp dụng và Giả định):**
Trong hệ thống Bánh Pía PMS đã có cơ chế chống trùng lặp bằng Idempotency Key được băm từ các trường: `phone`, `items`, `receiveDate`, `paymentMethod`. Mỗi tổ hợp thông tin chỉ tạo được một đơn khi key còn hiệu lực. Việc băm các trường này thành key cũng cho phép người dùng đặt thêm đơn khác trong thời gian key còn tồn tại, với điều kiện đơn mới không trùng các trường thông tin của đơn cũ. Cách này xử lý được hầu hết các đơn, nhưng chỉ phù hợp với lưu lượng vừa và nhỏ của hệ thống hiện tại.

Nếu hệ thống mở rộng thì cách này là chưa đủ: cần xử lý các trường hợp còn lại, và nếu Redis sập thì key cũng phải được lưu vào database để làm "nguồn sự thật".

---

## 2. Các phương án cân nhắc

| Phương án | Ưu điểm | Nhược điểm | Vì sao không chọn |
|---|---|---|---|
| **A: FE tạo một uuid mỗi khi người dùng gửi request tạo đơn** | Dễ thực hiện, hiệu năng nhẹ | Chỉ chặn được đơn trùng khi người dùng không làm mới trang. Nếu làm mới trang, FE sẽ tạo uuid mới nên vấn đề không được giải quyết. | Thiếu an toàn, nhiều trường hợp trùng vẫn lọt qua. |
| **B: Băm các trường `phone`, `items`, `receiveDate`, `paymentMethod` làm Idempotency Key, kết hợp Redis GET, SET để kiểm tra trạng thái của key** | Nếu người dùng làm mới trang, mỗi đơn hàng vẫn có một key riêng biệt nên chống được đơn trùng. Ngoài ra, trong thời gian TTL 10 phút, người dùng vẫn có thể đặt thêm đơn khác. | Không xử lý được trường hợp trong 10 phút đó người dùng đặt lại một đơn giống hoàn toàn các trường thông tin của đơn cũ. | — (phương án được chọn) |

---

## 3. Quyết định

Tôi chọn phương án băm các trường thông tin của đơn hàng (`phone`, `items`, `receiveDate`, `paymentMethod`) làm Idempotency Key với TTL 10 phút, kết hợp Redis GET, SET để truy cập nhanh trạng thái của key.

FE dùng các trường thông tin trên làm payload và băm bằng SHA-256 để ra hash. Key được gửi lên backend qua header `x-idempotency-key` khi gọi API `POST /booking/create`. Guard ở backend lấy key từ header, tạo thành Redis key rồi đọc trạng thái của key. Nếu Guard trả `409` kèm mã đơn, FE điều hướng theo phương thức thanh toán thay vì hiện lỗi, để người dùng có trải nghiệm tốt hơn. Nếu trạng thái là `PENDING`, hệ thống báo "Đang xử lý".

---

## 4. Đánh đổi & giới hạn

**Đánh đổi đã chấp nhận:**
Để đạt được tính an toàn chống trùng lặp, hệ thống chấp nhận độ phức tạp code cao hơn, và mỗi request tạo đơn phải thêm các lệnh Redis (chưa đo độ trễ).

**Giới hạn thực tế trong codebase hiện tại:**

- Hệ thống chưa xử lý được trường hợp gặp lỗi hoặc thất bại giữa chừng. Hiện tại chỉ có lỗi bên trong transaction là được xử lý, ví dụ không tìm thấy bánh hoặc thiếu địa chỉ giao hàng thì transaction được rollback. Tuy nhiên, Guard đã ghi `PENDING` trước khi gọi vào service, và chưa có phần nào trong hệ thống dọn key khi gặp lỗi.
- Hai request đến cùng lúc đều GET và thấy chưa có key, rồi cùng đi tiếp — chính là tình huống bấm đúp nút. Vì vậy, chỉ dùng GET và SET trên Redis là chưa đủ.
- Vì key không được lưu vào database (unique), nếu Redis gặp lỗi thì hệ thống không có nguồn nào để kiểm tra lại key, kể cả khi Redis đã phục hồi.
- Hệ thống hiện tại chỉ phù hợp với lưu lượng khách vừa và nhỏ, ít biến số. Nếu mở rộng, các trường hợp đặt đơn trùng thông tin, Redis sập hoặc server sập sẽ là lỗ hổng lớn đối với doanh thu và trải nghiệm khách hàng.

---

## 5. Những điều tôi sẽ làm khác đi

Các ý dưới đây nảy ra sau khi rà lại hệ thống, không nằm trong các phương án tôi đã cân nhắc lúc viết code.

Sau khi viết lại pseudocode dựa trên codebase hiện tại và nghiên cứu thêm về các giới hạn của hệ thống, nếu được làm lại tôi sẽ làm như sau:

- Không băm các trường thông tin của đơn hàng làm Idempotency Key nữa. Thay vào đó, FE sẽ tạo uuid khi phiên mua hàng bắt đầu (khi khách ở trang điền thông tin thanh toán). Nhờ vậy, dù khách làm mới trang, họ vẫn ở trong cùng phiên mua hàng và dùng lại cùng key, trừ khi thoát trang thanh toán.
- Thêm các hàm xóa key cho những trường hợp cần thiết, ví dụ khi hệ thống gặp lỗi.
- Dùng `SET key value NX EX 600` trong Redis thay cho GET, SET cơ bản như hiện tại.
- Kết hợp lưu key vào database với ràng buộc unique, để dù Redis có sập thì database vẫn là nguồn sự thật, đóng vai trò phao cứu sinh khi hệ thống gặp vấn đề.