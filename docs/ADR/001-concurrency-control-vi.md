# ADR-[001]: Concurrency Control

**Ngày viết:** [07/09/2026]  
**Trạng thái:** [Đã áp dụngi]

---

## 1. Context (Bối cảnh)

Làm sao đảm bảo dữ liệu hệ thống vẫn chính xác khi nhiều client cùng đọc/ghi gần như đồng thời? Tại vì chỉ cách một vài ms khi cả 2 client A và B cùng bấm đặt tại một thời điểm gần như đồng nhất thì dữ liệu đọc vào có thể tạo ra sự khác biệt.

Ví dụ thực tế khi hệ thống còn 1 slot bánh duy nhất và client A là người đến trước hệ thống vẫn hiển thị còn hàng vì chưa cập nhật kịp. Nhưng tại thời điểm client B vào bấm đặt bánh thì lúc này request từ client A vẫn chưa giải quyết xong, hậu quả là client B vẫn được phép đặt bánh và tạo nên tình huống double booking - cả hai client đều nhận được thông báo đặt bánh thành công, cả hai đều bị trừ tiền, nhưng thực tế kho chỉ đủ cho một người.

Trả lời:

- Chúng ta cần đảm bảo tại một thời điểm chỉ có một người được đọc và ghi vào dữ liệu của hệ thống. Nên tôi đã quyết định sử dụng pessimistic locking cho tình huống này.
- Đồng thời kết hợp với over-booking tạo thêm một bufferLimit = maxQuantity + 3% * maxQuantity (softLimit) - tổng số lượng bánh mà admin nhập vào mỗi ngày. Để nới lỏng độ cứng nhắc của hệ thống, đồng thời tăng trải nghiệm người dùng đồng thời có thể tạo ra một khoảng doanh thu nhỏ từ 3% này. Vì sao là 3%, vì đây là mức rủi ro over-sell tối đa mà tiệm bánh vẫn xử lý bù được bằng tay.
- Chống hoàn toàn over-booking nếu số lượng đặt bánh > bufferLimit sẽ bị reject ngay

---

## 2. Options Considered

| Phương án | Ưu điểm | Nhược điểm | Vì sao không chọn |
|------------|----------|----------|----------|
| A: Atomic Update Statement | Không cần thêm cơ chế lock, dùng UPDATE thường có điều kiện | Không thể gộp điều kiện kiểm tra bufferLimit vào 1 câu lệnh atomic duy nhất một cách an toàn - với bài toán trên ta cần đảm bảo với số lượng khách đặt phải <= bufferLimit thì multiple updates không đáp ứng được điều đó | Không giải quyết được tình huống bài toán |
| B: Optimistic locking | Không client nào phải đợi. Tăng trải nghiệm người dùng | Khi Client bắt đầu ghi vào dữ liệu của hệ thống, dữ liệu không đảm bảo tại thời điểm đó là đúng. | Chỉ phù hợp cho các chức năng nào cần việc đọc và ít hoặc không ghi cho nên không phù hợp với bài toán |
| C: Pessimistic locking | Tạo ra hàng chờ - client A đang bắt đầu phiên thì client B phải chờ đến lượt. Làm giảm trải nghiệm người dùng | Đảm bảo dữ liệu khi client bắt đầu đọc và đến ghi vẫn bảo toàn khối lượng. Khi client B bắt đầu vào thì sẽ đọc dữ liệu mới sau khi được client A đọc và ghi | — |

---

## 3. Decision (Quyết định)

Đã chọn: Bọc thao tác đọc–kiểm tra–ghi trong một transaction và sử dụng Pessimistic locking - Select FOR UPDATE

- Đảm bảo tính nhất quán dữ liệu khi có nhiều client đọc/ghi đồng thời, rất phù hợp cho bài toán đảm bảo an toàn khối lượng như trên.
- Vì độ chênh thời gian giữa A và B chỉ tính bằng mili-giây, thời gian B phải chờ trong hàng đợi là không đáng kể.

```mermaid
sequenceDiagram
...
```

---

## 4. Trade-offs & Limitations (Đánh đổi và giới hạn)

- Đánh đổi chính là độ trễ nhỏ (vài ms đến vài trăm ms) ở phía client B để đổi lấy đảm bảo tuyệt đối không double-booking — chấp nhận được vì tần suất trùng thời điểm đặt bánh là hiếm.
- Với điều kiện mà tiệm bánh có 2 loại bánh trở lên thì cách triển khai trên sẽ gãy hoàn toàn bởi vì mỗi phiên đều được bọc trong một transaction - đảm bảo các thao tác bên trong đều chạy thì nó mới được thực thi. Nhưng với việc bảo toàn khối lượng cho 2 hoặc nhiều loại bánh - có khối lượng khác nhau, sẽ có thể tạo nên một vòng chờ vô hạn request này chờ request kia.
- Tại sao không cải thiện bài toán trên bằng sort lại id theo thứ tự tăng hoặc giảm dần để đảm bảo các request tại cùng thời điểm chỉ đang xếp hàng chờ cùng một slot bánh xử lý xong. Vì tiệm bánh của mẹ tôi hiện chỉ bán một loại bánh, tôi quyết định không triển khai giải pháp ordered-locking này để tiết kiệm thời gian và tránh over-engineering. Tôi sẽ cân nhắc áp dụng nếu sau này tiệm bán thêm loại bánh khác.

---

## 5. What I'd do differently

Nếu làm lại, tôi sẽ viết integration test giả lập 2 request đồng thời (dùng Promise.all gọi 2 lần API cùng lúc) để tự động verify race condition đã thực sự được chặn, thay vì chỉ tin vào lý thuyết của SELECT FOR UPDATE.