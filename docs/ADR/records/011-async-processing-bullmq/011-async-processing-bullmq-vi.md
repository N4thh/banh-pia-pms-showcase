# ADR-011: Asynchronous Processing với BullMQ

**Ngày viết:** 12/11/2026
**Trạng thái:** Đã áp dụng

## 1. Bối cảnh

Hệ thống có hai việc không nên chạy trong lúc khách chờ phản hồi đặt hàng.

**Thông báo Telegram:** Telegram là dịch vụ của bên thứ ba, mình không kiểm soát được tốc độ của nó. Mỗi lần gọi Telegram là một lần gọi qua internet; nếu bên đó lỗi hoặc chậm thì việc trả kết quả cho khách hàng cũng chậm theo. Nên việc này cần được chạy sau khi kết quả đã được trả cho khách.

**Hủy đơn chuyển khoản hết hạn:** việc này chỉ cần chạy sau 10 phút kể từ lúc tạo đơn nếu đơn chưa được thanh toán, nên cần một cơ chế hẹn giờ chạy sau, không làm được trong request tạo đơn.

Không có sự cố xảy ra trước đó. Tôi làm vì lo nhiều event cùng lúc gây nghẽn module thông báo, và vì muốn học thêm message queue.

## 2. Các phương án cân nhắc

| Phương án | Ưu điểm | Nhược điểm | Lý do chọn hoặc không chọn |
|---|---|---|---|
| A: Chuyển thẳng event sang module thông báo | Đơn giản, dễ hiểu | Tôi lo tất cả event nếu đến cùng một lúc đều chuyển về module thông báo có thể làm nghẽn và server trở nên quá tải | Không chọn: tôi muốn học thêm message queue và nghĩ dùng hàng đợi sẽ đảm bảo hơn |
| B: Dùng BullMQ, tạo các job thông báo và đơn hết hạn vào hàng đợi | Các event được xếp hàng, xử lý riêng, có hẹn giờ hủy sau 10 phút, có thể thử lại nếu nhận được lỗi | Chưa cân nhắc | Chọn: cùng lý do ở trên |

## 3. Quyết định

Hiện tại hệ thống có 2 hàng đợi: hàng đợi cho đơn hết hạn (`order-expiry-queue`) và hàng đợi cho thông báo (`notification_queue`).

Khi đơn chuyển khoản được tạo, hệ thống phát event `ORDER_CREATED`. Listener ở notification module bắt event này và thêm job `cancel-expired-order` vào `order-expiry-queue`. Sau 10 phút, processor của `order-expiry-queue` nhận job, kiểm tra trạng thái đơn, hủy đơn với lý do hết hạn và phát event `ORDER_CANCELLED_AUTO`. Với đơn CASH, listener không tạo job hủy đơn mà gọi Telegram trực tiếp, không qua hàng đợi.

Với hàng đợi thông báo: khi nhận event `PAYMENT_SUCCESS`, listener thêm job `order.paid` vào `notification_queue`; processor nhận job và gửi Telegram với nội dung đơn hàng đã thanh toán thành công. Với event `ORDER_CANCELLED_AUTO`, processor có sẵn nhánh thông báo hủy tự động nhưng chưa có nơi nào thêm job cho nhánh đó, và listener của event này chỉ ghi log, vì tôi không muốn thông báo bị nhiễu.

Thông số của `notification_queue`: `attempts: 3`, `backoff: { type: 'exponential', delay: 2000 }`, `removeOnComplete: true`.
Thông số của `order-expiry-queue`: `delay: 10 * 60 * 1000` (10 phút), `attempts: 5`, `backoff: { type: 'exponential', delay: 2000 }`, `removeOnComplete: true`.

Với hàng đợi thông báo, tôi cho thử lại nếu lỗi, tối đa 3 lần cho mỗi thông báo. Với hàng đợi đơn hết hạn, công việc được xử lý sau 10 phút và được thử lại tối đa 5 lần khi lỗi. Cả hai đều bị xóa khỏi Redis sau khi job hoàn thành, và thời gian chờ thử lại ban đầu là 2 giây, tăng dần sau mỗi lần thử lại.

Tôi chọn BullMQ vì tôi muốn các job được xử lý riêng, có thể hẹn giờ 10 phút và đặc biệt có thể thử lại nếu lỗi. Về sau, khi nhận thấy nhược điểm là nếu Redis sập thì các job cũng mất, tôi dùng kết hợp thêm cron quét mỗi phút: đơn `NEW` quá hạn sẽ bị hủy, để lấp khoảng trống khi Redis có vấn đề.

## 4. Đánh đổi & giới hạn

Một đơn nhiều món tạo nhiều job hủy đơn vì không có `jobId`. Job đầu tiên hủy đơn và nhả slot cho tất cả các món; các job sau thấy đơn đã `CANCELLED`, nhận `BadRequestException` và bị bỏ qua. Vì vậy slot không bị nhả hai lần, nhưng có job thừa.

Hai đường hủy khác nhau về khóa và transaction. Job BullMQ dùng khóa `FOR UPDATE` trên `Order`, rồi trên từng dòng `Availability`; hủy đơn, cập nhật `paymentLink` và nhả slot nằm trong một transaction. Cron không khóa `Order`, dùng `updateMany` với điều kiện `id` và `status: NEW`; hủy đơn và cập nhật `paymentLink` nằm trong một transaction, còn nhả slot diễn ra sau khi commit, ngoài transaction, mỗi lần nhả mở transaction riêng. Nếu nhả slot của cron lỗi, đơn đã là `CANCELLED` nhưng slot không được nhả, không thử lại và chỉ có log. Dù cron chạy mỗi phút, đơn đã `CANCELLED` thay vì `NEW` nên bị bỏ qua, slot không được nhả.

Do Redis không lưu bền, khi container bị tạo lại thì job đang chờ và khóa giữ chỗ cũ cũng biến mất. Do đã có cron quét mỗi phút, dù Redis có lỗi thì việc hủy đơn chuyển khoản hết hạn và nhả slot vẫn tiếp tục; phần mất đi là các job thông báo đang chờ.

Khi thêm job hủy đơn bị lỗi (ví dụ Redis lỗi), lỗi chỉ được ghi log. Đơn đã commit nhưng không có job hủy; chỉ cron mới hủy được đơn đó.

Processor thông báo bắt lỗi Telegram và chỉ ghi log, không ném lại. Vì vậy `attempts` của job thông báo không còn tác dụng, và tin không được gửi. Ngoài ra lời gọi Telegram không có timeout, nên nếu Telegram không phản hồi thì job có thể bị treo (suy ra từ code, chưa chạy thử).

Hệ thống chưa có giám sát cho job thất bại (không có Bull Board hay cảnh báo). Có test cho cron; không tìm thấy test cho listener và hai processor.

Nếu webhook thanh toán đến gần 10 phút hoặc sau 10 phút, khi cron hoặc worker đã chuyển đơn thành `CANCELLED`, webhook sẽ cập nhật đơn đã hủy trở lại thành `PROCESSING`, vi phạm ràng buộc về trạng thái cuối của đơn đã hủy (xem ADR-004).

## 5. Những điều tôi sẽ làm khác đi

Các hướng dưới đây chưa được thực hiện trong code.

1. **Job hủy đơn thừa.** Chỉ thêm một job cho mỗi đơn, thay vì mỗi món một job, và đặt `jobId` theo `orderId` để BullMQ không nhận job trùng. Thay đổi ở listener của `ORDER_CREATED`. Đây là ý tôi nghĩ ra sau khi đọc lại code, không phải điều tôi cân nhắc lúc viết.

2. **Cron nhả slot ngoài transaction.** Cho cron gọi lại cùng hàm `cancelOrderById` mà worker đang dùng, để hủy đơn, cập nhật `paymentLink` và nhả slot nằm trong một transaction. Khi đó nếu nhả slot lỗi thì cả transaction rollback, đơn vẫn là `NEW` và lần quét sau sẽ thử lại. Thay đổi ở `PaymentCronService`. Tôi vẫn giữ cả job BullMQ và cron: job hủy đúng mốc 10 phút và có event, còn cron là lưới an toàn khi Redis làm mất job. Bỏ cron thì không còn gì cứu được đơn khi job mất. Đây cũng là ý nghĩ ra sau.

3. **Redis không lưu bền.** Bật lưu dữ liệu cho Redis (`appendonly`) và gắn volume cho container Redis trong `docker-compose.production.yml`, để khởi động lại không làm mất job đang chờ. Cách làm giống hướng ở ADR-010.

4. **Lỗi khi thêm job chỉ có log.** Ở quy mô hiện tại tôi chấp nhận. Đơn chuyển khoản vẫn được cron hủy, nên rủi ro thấp. Tôi chỉ nâng mức log lên `error` và ghi kèm `orderId` để tìm lại được.

5. **Lỗi Telegram bị nuốt và không có timeout.** Trong processor, ném lại lỗi mạng hoặc lỗi từ phía Telegram để `attempts` có tác dụng, còn lỗi thiếu token hoặc chat ID thì không cần thử lại. Đặt timeout rõ ràng cho `HttpService` để một lời gọi treo không chiếm chỗ xử lý. Thay đổi ở `NotificationProcessor` và `HttpModule`. Đây là ý nghĩ ra sau.

6. **Giám sát và test.** Thêm test cho listener và processor hủy đơn, vì đây là phần quyết định số phận của đơn. Còn dashboard hay cảnh báo cho job thất bại thì tôi chấp nhận chưa làm ở quy mô một tiệm, 109 đơn cả mùa.

7. **Webhook ghi đè đơn đã hủy.** Hướng sửa và quyết định sửa hay chỉ ghi hiện trạng thuộc về ADR-004. Ở bài này tôi chỉ ghi nhận giới hạn.