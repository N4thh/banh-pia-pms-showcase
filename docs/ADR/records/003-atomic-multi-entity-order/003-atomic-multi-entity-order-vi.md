## ADR-[003]: Atomic Multi-Entity Order Creation & Boundary Design

**Ngày viết:** [25/09/2026] **Trạng thái:** [Đã áp dụng]

---

### 1. Bối cảnh

1. **Bài toán này khác gì so với ADR-001 (chỉ 1 bảng Availability)?**
2. **Kịch bản cụ thể nào cho thấy nếu KHÔNG xử lý đúng, hệ thống sẽ hỏng ra sao?**
3. **Vì sao vấn đề này không thể giải quyết bằng cách ghi từng bảng một cách độc lập, kiểm tra lỗi thủ công sau mỗi bước?**

Trả lời:

1.

Khi bấm đặt hàng thì các bảng data liên quan như: order, address, order items, user, availability sẽ được tạo mới cùng lúc với những thông tin tương ứng.

Việc ghi vào nhiều bảng cùng lúc tại sao lại tạo ra những rủi ro? Bởi vì mức độ ảnh hưởng và phụ thuộc của các bảng với nhau tạo nên sự nhất quán về dữ liệu. Nếu một bảng nào thiếu hoặc không đúng về thông tin đều có thể gây ra sai lệch về dữ liệu của đơn hàng và khách hàng.

Ngoài ra còn cần đảm bảo kiểu dữ liệu vì có thể các bảng có kiểu dữ liệu khác nhau nên đòi hỏi tính chính xác cao hơn.

2.

Ví dụ thực tế nếu bảng Availability được ghi vào DB thành công nhưng bước tạo order ngay sau đó bị lỗi thì hậu quả là các slot bánh bị trừ nhưng thực tế đơn hàng chưa được tạo.

Đây là vi phạm tính nguyên tử của giao dịch.

Nếu lặp lại nhiều lần, hệ thống sẽ hiển thị sai là đã hết hàng dù chưa bán được, gây hao hụt doanh thu thực sự. Ngoài ra còn làm cho quản trị viên thấy bối rối và tác động xấu đến trải nghiệm người dùng của cả admin lẫn khách hàng.

3.

Trên thực tế việc tạo thủ công từng bảng vẫn có thể thực hiện được, nhưng chỉ phù hợp nếu tất cả các bước tạo luôn đảm bảo thành công.

Nếu một bước nào đó sai, chúng ta phải tự dọn dẹp các bản ghi mồ côi (orphan record) phát sinh và việc đó không hề tối ưu.

Cách ở đây là bọc tất cả việc tạo vào một transaction để đảm bảo rằng tất cả thông tin đều được ghi vào các bảng một cách đồng nhất và nhất quán.

---

### 2. Các phương án cân nhắc

| **Phương án** | **Ưu điểm** | **Nhược điểm** | **Vì sao không chọn** |
|--------------|-------------|---------------|----------------------|
| A: Thực hiện các câu lệnh riêng lẻ, ghi từng bảng độc lập | Dễ thực hiện | Nếu có bước nào không thành công sẽ tạo orphan record và phải dọn dẹp nếu muốn dữ liệu nhất quán | Phát sinh thêm bước dọn dẹp thủ công và rủi ro nếu dữ liệu sai lệch không được xử lý kịp thời. |
| B: Bọc toàn bộ DB transaction và Redis hold key vào một transaction | Đáp ứng đúng yêu cầu ban đầu (mọi thứ cùng thành công hoặc cùng thất bại), code trông gọn hơn | Không đảm bảo được phần Redis hold key sẽ hoạt động cùng với các tác vụ trong transaction | Vì Redis và DB transaction hoạt động ở hai tầng khác nhau. Việc bọc Redis vào transaction là vô nghĩa vì Redis không tham gia được cơ chế rollback của DB. |
| C: Dùng DB transaction để thực hiện chung các câu lệnh, tách Redis hold key ra ngoài | Đảm bảo được tất cả nhu cầu mà chúng ta đặt ra | Code phức tạp hơn | |

---

### 3. Quyết định

Tôi đã chọn phương án dùng DB transaction để gộp chung các câu lệnh ghi, đảm bảo các thao tác ghi vào các bảng liên quan cùng được ghi vào một lúc.

Chắc chắn rằng nếu một thao tác ghi vào bảng nào đó xảy ra lỗi thì tất cả thao tác đã thực hiện và chưa thực hiện sẽ được rollback, quá trình dừng lại ngay tại đó.

Đồng thời tôi để Redis hold key ở bên ngoài transaction bởi vì DB và Redis hoạt động ở hai tầng khác nhau. Vì vậy nếu transaction thành công nhưng Redis fail sau đó thì đơn hàng vẫn được tạo vì lỗi Redis không nên làm dừng transaction.

a) Vì sao tách Redis ra khỏi transaction:

Ở đây tôi xem DB là nguồn sự thật (source of truth) còn Redis chỉ là lớp hỗ trợ tạm thời. Vì vậy nếu Redis gặp lỗi thì dữ liệu nghiệp vụ chính vẫn đúng trong DB.

Ngoài ra Redis không có khả năng tham gia vào cơ chế rollback của DB transaction.

b) Vì sao dùng Redis để giữ hold thay vì query DB liên tục (lý do hiệu năng):

Chúng ta nên nhớ dữ liệu DB được đọc từ ổ đĩa (SSD/HDD), trong khi dữ liệu Redis nằm trong RAM. Vì vậy tốc độ truy cập khác nhau đáng kể.

Nếu hệ thống liên tục query DB để tính toán thời gian thanh toán còn lại của đơn hàng (10 phút) thì query cost sẽ lớn hơn rất nhiều, đặc biệt khi có nhiều người dùng đặt hàng cùng lúc.

Đây là lý do Redis được chọn làm lớp giữ trạng thái tạm thời ngay từ đầu và quyết định này không liên quan đến rollback.

---

### 4. Đánh đổi và giới hạn

- Ban đầu tôi từng định đặt Redis hold key bên trong transaction với kỳ vọng rằng nếu Redis fail thì transaction cũng rollback theo. Tuy nhiên sau khi đọc lại và tìm hiểu thêm về cách DB và Redis hoạt động, tôi nhận ra cách tiếp cận đó là sai và chuyển sang giải pháp hiện tại.
- Với đơn thanh toán chuyển khoản, hold slot chỉ tồn tại trong 10 phút. Để dọn các đơn quá hạn, tôi dùng cơ chế bù đắp: một cron job quét các đơn có status 'NEW' đã quá thời gian thanh toán, so sánh thời gian đặt hàng với thời gian hiện tại rồi chuyển sang 'CANCELLED' với lý do 'thanh toán hết hạn'.
- Trong lúc viết lại pseudocode cho hàm createOrder, điều lớn nhất tôi nhận ra là redis-hold-key không nên nằm trong try-catch dùng để xử lý lỗi Redis.

---

### 5. Những điều tôi sẽ làm khác đi

Nếu được làm lại, tôi sẽ hệ thống lại tất cả hậu quả có thể xảy ra để có thể xử lý và thông báo lỗi chi tiết hơn.

Điển hình là trường hợp Redis hold key bị lỗi nhưng DB transaction vẫn thành công. Điều này khiến tôi nhận ra rằng mình đã bỏ sót một lỗi tiềm ẩn.

Ngoài ra hàm createOrder hiện cũng đang khá dài và phức tạp. Nếu làm lại, tôi sẽ tổ chức nó tốt hơn vì ngay cả khi viết lại pseudocode, tôi cũng phải tự nhắc mình nhiều lần mới nhớ đủ các bước cần thiết.