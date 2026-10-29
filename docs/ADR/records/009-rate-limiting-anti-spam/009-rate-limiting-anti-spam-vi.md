# ADR-009: Rate Limiting & Anti-Spam

**Ngày viết:** 27/10/2026
**Trạng thái:** Áp dụng một phần

## 1. Bối cảnh

Những đường vào công khai của hệ thống gồm có:

- **Đặt hàng của khách:** đây là nhóm duy nhất có cơ chế bảo vệ. Rủi ro có thể xảy ra tại đường này là gửi lặp request, chiếm slot, tốn tài nguyên.
- **Tra cứu dữ liệu khách và đơn:** rủi ro là tra user theo số điện thoại liên tục để lấy thông tin khách hàng.
- **Thanh toán và đăng nhập:** tạo link thanh toán và đăng nhập admin. Rủi ro là gọi lặp để tạo link bên PayOS, thử mật khẩu liên tục.
- **Ghi dữ liệu công khai:** ghi log truy cập, giữ chỗ (hold). Rủi ro là ghi vào Redis hoặc DB không giới hạn.

Từ những rủi ro của việc gửi quá nhiều request về server, làm cho các slot bánh bị chiếm mất, gây tốn tài nguyên và có thể dẫn đến sai kết quả cuối cùng, nên tôi đã thực hiện phần giới hạn và chống hành động gửi yêu cầu liên tục.

ADR-005 đã nói về việc chống gửi đơn trùng nhờ Idempotency Key. Cơ chế đọc IP và tạo key trong ADR này đếm số request gửi từ cùng một IP trong một khoảng thời gian, để ngừng nhận request khi vượt ngưỡng.

## 2. Các phương án cân nhắc

Lúc viết code tôi không cân nhắc phương án nào khác, tôi làm trực tiếp ở tầng ứng dụng bằng guard và Redis.

## 3. Quyết định

Tôi đã thực hiện cơ chế đọc IP khi request được gửi vào, dùng IP đó để tạo Redis key. Kiểm tra xem key đã tồn tại hay chưa, nếu chưa thì đếm là 0; nếu bằng 0, tôi thực hiện lệnh `SET` của Redis với value = 1 và TTL = 60 giây. Mong muốn lúc đầu của tôi là chỉ cho phép 1 request duy nhất thực hiện được nghiệp vụ tạo đơn hàng, nên tôi đặt `MAX_REQUEST = 1`. Nếu vượt quá 1 thì trả lỗi 429. Nếu bằng 1 thì dùng lệnh `INCR` của Redis để tăng giá trị của key lên 1.

Khi nhận lỗi 429, FE hiển thị thông báo do BE trả về: "Bạn thao tác quá nhanh. Vui lòng đợi chúng tôi 1 phút rồi tiếp tục đặt nhé".

## 4. Đánh đổi & giới hạn

Khi các request trong 60 giây đến tuần tự, có 2 request qua được: request đầu tiên thực hiện `SET` và đặt TTL; request thứ 2 thấy request count không còn bằng 0 nên thực hiện `INCR`. Khi hàng loạt request đến cùng một lúc, tất cả đều đọc thấy giá trị 0 nên đều đi qua và đều thực hiện lệnh `SET`. So với giá trị `MAX_REQUEST = 1`, kết quả này không còn đúng, và đó là thiếu sót của tôi.

Khi key hết hạn giữa hai lệnh Redis (`GET` và `INCR`): sau khi `GET` thấy request count = 1 thì thực hiện `INCR`, lúc này key vừa hết hạn. `INCR` sẽ tạo một key mới có giá trị 1 nhưng không có TTL, nên key tồn tại mãi, trừ khi bị xóa tay.

Không xử lý được trường hợp nhiều khách dùng chung một IP.

Không có try/catch để xóa key khi hàm `createOrder` ném lỗi (thất bại, hết slot,...). Hậu quả: nếu người dùng gửi lại với cùng các trường `phone`, `items`, `receiveDate`, `paymentMethod` (nên băm ra cùng key), hệ thống vẫn trả lỗi 409 với nội dung "Đơn hàng của bạn đang được xử lý. Vui lòng đợi chúng tôi trong giây lát".

## 5. Những điều tôi sẽ làm khác đi

Xử lý tốt hơn ở phần khoảng giữa các cơ chế để không có key nào bị treo nữa bằng cách sử dụng try/catch. Đảm bảo chỉ một request được qua, thể hiện đúng tên hằng `MAX_REQUEST`.

Xử lý các tình huống dùng lệnh `DEL` để xóa key của request đã lỗi, bằng cách bắt lỗi mà BE trả về và thực hiện xóa ngay sau đó.

Tìm cách xử lý các trường hợp nhiều khách dùng chung một IP.