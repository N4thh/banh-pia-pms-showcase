# ADR-003: Atomic Multi-Entity Order Creation & Boundary Design

**Ngày viết:** 25/09/2026  
**Trạng thái:** chưa xong

---

## 1. Context

**Câu hỏi dẫn dắt:** Bài toán này khác gì so với ADR-001 (chỉ 1 bảng Availability)? Kịch bản cụ thể nào cho thấy nếu KHÔNG xử lý đúng, hệ thống sẽ hỏng ra sao? Vì sao vấn đề này không thể giải quyết bằng cách ghi từng bảng một cách độc lập, kiểm tra lỗi thủ công sau mỗi bước?

**Phân tích & Trả lời:**
1. Khi bấm đặt hàng thì các bảng data liên quan như: `Order`, `Address`, `OrderItem` sẽ được tạo mới cùng lúc với những thông tin tương ứng. Việc ghi vào nhiều bảng cùng lúc tại sao lại tạo ra những rủi ro? Bởi vì mức độ ảnh hưởng và phụ thuộc của các bảng với nhau tạo nên sự nhất quán về dữ liệu. Nếu một bảng nào thiếu, không đúng về thông tin đều có thể gây ra sai lệch về dữ liệu của đơn hàng - khách hàng. Ngoài ra còn cần đảm bảo kiểu dữ liệu vì có thể các bảng có kiểu dữ liệu khác nhau nên đòi hỏi tính chính xác cao hơn.
2. **Ví dụ thực tế:** Nếu bảng `Availability` được ghi vào DB thành công nhưng bước tạo `Order` ngay sau đó bị lỗi, hậu quả là các slot bánh bị trừ nhưng thực tế đơn hàng chưa được tạo — tạo nên sự sai lệch về tính an toàn nguyên tử (Partial Commit). Nếu tình trạng này được nhân lên nhiều lần sẽ tạo nên sự hao hụt về tiền bạc (khách thấy hết bánh) và làm cho quản trị viên thấy bối rối, tác động xấu đến trải nghiệm người dùng cả admin lẫn client.
3. Trên thực tế, việc tạo thủ công từng bảng là có thể được nhưng nó chỉ phù hợp nếu tất cả các bước tạo luôn đảm bảo là thành công. Nếu một bước nào đó sai, chúng ta phải dọn dẹp lại (orphan record) một lần nữa và việc đó không hề tối ưu. Cách ở đây là bọc tất cả việc tạo trên vào một transaction để đảm bảo rằng tất cả thông tin đều được ghi vào bảng đồng nhất và nhất quán.

---

## 2. Options Considered

### Phương án A: Thực hiện các câu lệnh riêng lẻ, ghi từng bảng độc lập
* **Ưu điểm:** Dễ thực hiện.
* **Nhược điểm:** Nếu có bước nào không thành công sẽ tạo orphan record và phải dọn dẹp nó nếu muốn data nhất quán.
* **Vì sao không chọn:** Tạo thêm bước dọn dẹp cho hệ thống và rủi ro lớn nếu các thông tin sai lệch không được dọn dẹp triệt để.

### Phương án B: Bọc toàn bộ DB transaction và Redis hold key vào một transaction
* **Ưu điểm:** Đảm bảo bài toán lúc đầu yêu cầu và code nhìn trực quan hơn.
* **Nhược điểm:** Không đảm bảo được phần Redis hold key sẽ hoạt động cùng với các tác vụ trong transaction. Bởi vì Redis và DB transaction hoạt động ở 2 tầng khác nhau. Việc bọc Redis vào transaction không có nghĩa, không đảm bảo được mong muốn là "nếu 1 cái fail thì tất cả cùng fail".
* **Vì sao không chọn:** Không mang lại tính nguyên tố cross-system thật sự và dễ gây hiểu nhầm về ranh giới an toàn của transaction.

### Phương án C: Dùng DB transaction để thực hiện chung các câu lệnh DB, tách Redis hold key ra ngoài
* **Ưu điểm:** Đảm bảo được tất cả nhu cầu về an toàn dữ liệu 100% trong DB.
* **Nhược điểm:** Code phức tạp hơn.
* **Vì sao chọn:** Đáp ứng đúng ranh giới của từng hệ thống, tôn trọng PostgreSQL là Single Source of Truth.

---

## 3. Decision

Tôi đã chọn **Phương án C**: Dùng DB transaction (`$transaction`) để thực hiện chung các câu lệnh, đảm bảo các thao tác ghi vào các bảng liên quan cùng được ghi vào một lúc. Chắc chắn rằng nếu một thao tác ghi vào bảng nào đó xảy ra lỗi thì tất cả thao tác đã thực hiện và chưa thực hiện sẽ được rollback hoàn toàn.

Đồng thời, để câu lệnh Redis hold key ở BÊN NGOÀI transaction bởi vì DB và Redis hoạt động ở 2 tầng khác nhau. Nếu transaction thành công nhưng bước Redis fail thì đơn hàng vẫn được tạo bình thường trong DB. 

Ở đây, DB là nguồn sự thật (Single Source of Truth) và Redis là lớp hỗ trợ tạm thời nên việc dù Redis có fail thì hệ thống vẫn đọc được thông tin thật từ DB. Tuy nhiên, dữ liệu DB lưu trên đĩa cứng (SSD) còn Redis nằm trên RAM nên tốc độ truy cập rất khác nhau. Việc hệ thống phải liên tục gọi vào DB để xem thời gian đặt hàng nhằm tính lại thời gian còn lại thanh toán (10 phút) là rất lớn. Nếu mỗi giây gọi vào DB 1 lần cho 1 user thì cần 600 lượt query; với $N$ user cùng lúc sẽ là $600 \times N$ lượt query — tạo sức nặng lớn cho server và không đảm bảo thời gian phản hồi chính xác cho client. Vì vậy, Redis hold key đóng vai trò là lớp Cache/Timer tra cứu nhanh $O(1)$ phía trước.

---

## 4. Trade-offs & Limitations

* **Hiểu nhầm ban đầu về ranh giới Redis:** Bản thân từng đặt `setHold` bên trong transaction với mong muốn nếu Redis fail thì DB transaction cũng rollback. Nhưng sau khi tìm hiểu lại về kiến trúc tầng của DB và Redis, tôi đã nhận ra điểm chưa chính xác này và tách Redis ra ngoài.
* **Thời hạn thanh toán 10 phút:** Đối với các đơn chọn chuyển khoản (`BANK_TRANSFER`), đơn hàng chỉ được giữ trong 10 phút. Để giải quyết việc dọn dẹp các đơn quá hạn khi thiếu Redis key, tôi sử dụng cơ chế bù đắp (Reconciliation) là **Cron Job** quét các đơn có trạng thái `NEW`. Định kỳ Cron Job quét DB, so sánh `createdAt` với thời gian hiện tại, nếu quá 10 phút sẽ chuyển trạng thái từ `NEW` sang `CANCELLED` và trả lại slot bánh.
* **Phát hiện khi Pseudocode:** Việc nhận ra sự khác biệt của Redis hold key là phát hiện lớn nhất của tôi trong quá trình thử sức pseudo lại hàm `createOrder`.

---

## 5. What I'd do differently

* **Hệ thống hóa kịch bản lỗi:** Nếu được làm lại, tôi sẽ hệ thống hóa tất cả hậu quả có thể diễn ra để xử lý và thông báo lỗi chi tiết hơn. Đặc biệt là việc nếu Redis hold key fail nhưng DB transaction vẫn báo "xanh" và hoạt động bình thường, khiến người viết không nhận ra điểm chưa tối ưu nếu không test kỹ.
* **Tái cấu trúc hàm `createOrder`:** Hàm `createOrder` hiện tại khá dài và chứa nhiều logic (user, address, price, slot, status, event, redis). Nếu làm lại, tôi sẽ tách nhỏ và mô-đun hóa tốt hơn để giảm độ phức tạp và dễ bảo trì hơn.
