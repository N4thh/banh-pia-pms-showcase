# ADR-004: Order Cancellation & Concurrency Control

**Ngày viết:** 01/10/2026
**Trạng thái:** Đã áp dụng

---

## 1. Bối cảnh

**Câu hỏi 1.1:** Trong hệ thống Bánh Pía PMS, có 3 tác nhân độc lập nào cùng có khả năng tác động/chuyển đổi trạng thái của một đơn hàng? Kịch bản tranh chấp xấu nhất giữa Admin và BullMQ delayed worker tại cùng 1 mili-giây là gì?

**Câu hỏi 1.2:** Ràng buộc bất biến về Trạng thái cuối của đơn hàng quy định điều gì? Nếu một Webhook thanh toán gửi về trễ sau khi Cron/Admin đã hủy đơn, hệ thống phải xử lý thế nào để không làm "sống lại" đơn hàng?

**Câu hỏi 1.3:** Ràng buộc bất biến về Xả slot công suất bánh quy định số lần tối đa slot được phép xả cho 1 đơn hàng bị hủy là bao nhiêu? Nếu bị xả 2 lần thì hậu quả kinh doanh là gì?

**Trả lời:**

**1.1:** Trong hệ thống PMS hiện tại có 3 tác nhân độc lập tác động vào việc chuyển đổi trạng thái của một đơn hàng, đó chính là: Admin, BullMQ delayed worker (xử lý các đơn hàng quá hạn) và Webhook từ PayOS. Kịch bản tệ nhất là tranh chấp giữa Admin và worker tại cùng một mili giây — và đây không chỉ là rủi ro lý thuyết: khi rà lại code (xem mục 4.3), tôi xác nhận ban đầu Worker tự cập nhật trạng thái đơn mà không qua cùng cơ chế khóa với `cancelOrderById`, khiến cả 2 có thể cùng đọc trạng thái 'NEW' và cùng xả slot, dẫn đến overbooking do slot bị hoàn trả thừa.

**1.2:** Hệ thống PMS có một ràng buộc bất biến về trạng thái cuối của đơn hàng: đó chính là nếu đơn hàng đang là "CANCELLED" hoặc "COMPLETED" thì sẽ không được quay trở lại bất kì trạng thái nào trước đó. Nếu Webhook thanh toán gửi về trễ sau khi Cron/Admin đã hủy đơn thì hệ thống sẽ kiểm tra: nếu đơn đã nằm trong 1 trong 2 trạng thái trên, trả về lỗi rằng đơn đã được hủy/hoàn thành trước đó.

**1.3:** Hệ thống PMS hiện tại chỉ cho phép xả slot đúng 1 lần cho mỗi đơn hàng bị hủy. Như đã nêu ở mục 1.1, nếu xả 2 lần sẽ tạo dư slot so với thực tế, tạo nên overbooking — sai lệch nghiêm trọng về an toàn khối lượng.

---

### 2. Các phương án cân nhắc

| Phương án | Ưu điểm | Nhược điểm | Vì sao không chọn |
|---|---|---|---|
| **A: Lần lượt kiểm tra các trạng thái đơn hàng rồi gọi release-hold-slot và update đơn hàng lần lượt mà không bọc trong transaction** | Dễ thực hiện | Nếu một trong 2 bước thất bại thì tạo nên việc sai lệch về thông tin | Không đảm bảo tính an toàn hệ thống đặt ra, dễ tạo lỗ hổng nhất quán dữ liệu đơn hàng. |
| **B: Sử dụng UpdateMany, nếu DB trả count = 1 thì được phép gọi release-hold-slot** | Các luồng sau không phải chờ, có thể hoạt động cùng lúc. | UpdateMany chỉ trả về số lượng bản ghi đã cập nhật (dạng number), không trả về chi tiết đơn hàng. Nhưng chúng ta cần order items để có thể gọi vào release hold slot, nên phải gọi riêng một hàm find cho mục này. Ngoài ra còn mất khả năng phản hồi lỗi chi tiết cho User/Admin. | Phát sinh thêm nhiều bước phụ, và cuối cùng vẫn phải bọc trong transaction — không tiết kiệm được độ phức tạp. |
| **C: Sử dụng pessimistic locking và bọc 2 thao tác vào trong một transaction** | Đảm bảo được 2 thao tác ra đúng yêu cầu. Trả về các items trong order, không phải gọi kiểm tra một lần nữa. Trả lỗi chi tiết cho User/Admin. | Thực hiện phức tạp. Phải đảm bảo transaction được truyền đúng qua các service con (tx propagation). | — |

---

### 3. Quyết định

**3.1:** Sử dụng transaction parent và truyền database client giữa các service. Nếu không truyền tx, service con (release hold slot) dù tự mở transaction riêng vẫn sẽ tạo một DB connection khác, tách biệt khỏi transaction chính. Truyền tx xuyên suốt đảm bảo toàn bộ `cancelOrderById` chạy chung một connection, cùng một ranh giới transaction.

**3.2:** Dùng `SET LOCAL lock_timeout` 3 giây để tránh một `SELECT FOR UPDATE` chờ lấy khóa quá lâu, làm các luồng khác bị treo vô thời hạn và cạn kiệt connection pool.

**3.3:** Ở tầng SQL, hàm xả slot dùng công thức `GREATEST(0, expression)` để đảm bảo số lượng slot không bao giờ âm.

**3.4:** Worker (BullMQ) khi tự động hủy đơn quá hạn sẽ gọi lại chính `cancelOrderById` với `adminId = null` để biểu thị hệ thống tự hủy, đảm bảo cả admin và worker đi qua cùng một ranh giới transaction và cơ chế khóa.

---

### 4. Đánh đổi và giới hạn

**4.1 (Đánh đổi hiệu năng):** Giải pháp được chọn chính là sử dụng pessimistic locking trong vòng 3s để đảm bảo được tính an toàn. Nhưng thay vào đó sẽ đánh đổi rằng các thao tác phía sau sẽ phải chờ cho đến lượt, điều đó làm giảm hiệu năng của hệ thống. Nhưng 3 giây là thời gian chờ tối đa để lấy khóa, không phải thời gian mỗi thao tác thực tế mất. Thực tế một thao tác chỉ tốn khoảng mili giây, và với lưu lượng người dùng hiện tại (vừa và nhỏ), hàng chờ phát sinh không đáng kể.

**4.2 (Khóa dư thừa có chủ đích trên Availability):** Về mặt lý thuyết, câu lệnh UPDATE xả slot đã mang tính atomic. Nhưng tôi vẫn lựa chọn sử dụng `SELECT FOR UPDATE` thủ công trên bảng `Availability` để đồng nhất về code base, để bất kỳ ai (kể cả tôi sau này) đọc lại code đều thấy rõ chủ đích của quyết định này, không hiểu nhầm là sót logic.

**4.3:** Ban đầu, worker xử lý đơn quá hạn (BullMQ) tự cập nhật đơn trực tiếp, không sử dụng cùng cơ chế với hàm `cancelOrderById` của booking service — tạo ra một race condition tiềm ẩn: cả worker và admin có thể cùng đọc trạng thái 'NEW' rồi cùng xả slot. Sau khi rà lại, tôi sửa để worker dùng chung hàm `cancelOrderById` với admin, đảm bảo đồng nhất cơ chế khóa và giải quyết triệt để vấn đề trên.

---

### 5. Những điều tôi sẽ làm khác đi

**5.1:** Hệ thống của tôi đã có sẵn BullMQ + Redis với worker đã viết sẵn nhánh `order.cancelled` để báo các đơn bị hủy do quá hạn. Nhưng vì thông báo này khiến các đơn mới của mẹ tôi bị trôi mất trong luồng tin nhắn, tôi đã quyết định không bật tính năng này. Nếu lần sau làm lại tôi sẽ thiết kế lại phần nhận thông báo cho mẹ tôi một cách trực quan hơn, để mẹ tôi vẫn nhận được thông báo khi có đơn bị hủy, mà không bị lẫn với thông báo đơn hàng mới.