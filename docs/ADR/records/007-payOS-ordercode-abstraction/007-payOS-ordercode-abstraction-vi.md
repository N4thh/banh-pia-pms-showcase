# ADR-007: PayOS OrderCode Abstraction

**Ngày viết:** 20/10/2026
**Trạng thái:** Đã áp dụng

---

## 1. Bối cảnh

**Bối cảnh nghiệp vụ:**
Từ lúc khách chọn chuyển khoản đến lúc đơn được ghi nhận đã thanh toán, có một bước gửi yêu cầu kèm mã (`payosOrderCode`) sang PayOS để tạo link thanh toán. Ở bước đó, bảng `PaymentLink` được ghi mới và hệ thống dùng mã `orderId` để nhận diện đơn.

Hệ thống của tôi và PayOS mỗi bên nhận diện đơn bằng một mã khác nhau: hệ thống của tôi dùng `orderId`, còn PayOS dùng `payosOrderCode`.

**Kịch bản rủi ro nếu không có cơ chế riêng cho mã gửi PayOS:**

- Mã nội bộ và mã gửi PayOS phục vụ hai mục đích khác nhau, nên tách riêng để code rõ ràng, tránh nhầm lẫn.
- Mã gửi PayOS được sinh ngẫu nhiên, nên người ngoài chỉ có mã này thì không biết được nó ứng với đơn nào.

**Ràng buộc bất biến:**

- Mỗi đơn chỉ có tối đa một bản ghi `PaymentLink`, vì ràng buộc `@unique` ở database đảm bảo điều đó.
- Mã gửi PayOS phải là số nguyên và duy nhất; phía database của chúng tôi cũng yêu cầu như vậy.

---

## 2. Các phương án cân nhắc

| Phương án | Ưu điểm | Nhược điểm | Vì sao không chọn |
|---|---|---|---|
| **A: Sử dụng `orderId` làm `payosOrderCode`** | Tiện dụng, có thể dùng ngay | Người ngoài đọc mã thanh toán có thể biết được nó tương ứng với đơn nào | Chưa đúng yêu cầu ban đầu |
| **B: Tách biệt rõ ràng `orderId` và `payosOrderCode`** | Dù người ngoài có được `payosOrderCode` cũng không biết nó thuộc đơn nào | Phải thiết kế riêng các ràng buộc bất biến và cách tạo mã | — (phương án được chọn) |

---

## 3. Quyết định

Tôi chọn tách biệt hoàn toàn `orderId` và `payosOrderCode`. Mã gửi sang PayOS được sinh ngẫu nhiên từ 100.000.000 đến 999.999.999. Mỗi đơn hàng chỉ có duy nhất một mã PayOS, và mã này được lưu trong bảng `PaymentLink`.

Lý do là công dụng của từng trường dữ liệu sẽ rõ ràng, mỗi trường phục vụ đúng mục đích của nó. Ngoài ra, nếu người ngoài có được mã PayOS thì cũng không biết mã đó dùng cho đơn hàng nào.

**Luồng khi hàm tạo link thanh toán được gọi:**

- Nếu đơn chưa có link, hàm tạo link mới.
- Nếu đơn đã có link, hàm trả lại các dữ liệu như URL, QR code, ... để FE sử dụng và chuyển đến trang tương ứng với trạng thái của đơn hàng.

**Thứ tự các bước:**

1. Tìm đơn và kiểm tra đơn có tồn tại hay không.
2. Kiểm tra địa chỉ gốc của FE đã được thiết lập hay chưa.
3. Kiểm tra link đã tồn tại hay chưa. Nếu đã có thì trả lại link cũ; nếu chưa thì tiến hành tạo link: sinh `payosOrderCode` ngẫu nhiên từ 100.000.000 đến 999.999.999.
4. Từ bước này dùng `try/catch`: gọi PayOS để tạo link thanh toán; nếu thành công thì ghi mới các dữ liệu PayOS trả về vào bảng `PaymentLink`.
5. Nếu một trong hai bước này bị lỗi thì trả lỗi 500.

**Khi PayOS gửi webhook:**
Hệ thống dùng trường `orderId` trong dữ liệu webhook để tìm ra đơn, thông qua bảng `Order`.

---

## 4. Đánh đổi & giới hạn

**Đánh đổi đã chấp nhận:**

- **Đánh đổi kỹ thuật:** Phải thiết kế riêng các ràng buộc bất biến và cách tạo mã.
- **Đánh đổi nghiệp vụ:** Không có đánh đổi rõ rệt.

**Giới hạn / điểm chưa đồng bộ trong codebase hiện tại:**

- Hiện tại hệ thống chỉ hủy link thanh toán ở phía hệ thống, chưa hủy ở phía PayOS, nên nếu khách vẫn thanh toán vào link đó thì hệ thống sẽ xử lý sai, dù đơn đã được thanh toán.
- Việc sinh ngẫu nhiên từ 100.000.000 đến 999.999.999 vẫn có khả năng trùng mã, và hệ thống chưa có phần kiểm tra việc này.

---

## 5. Những điều tôi sẽ làm khác đi

- Tôi sẽ làm luồng hủy link thanh toán tốt hơn, hủy cả hai phía: hệ thống và PayOS, để có sự đồng bộ và ngăn trường hợp khách vẫn thanh toán vào link riêng của PayOS.
- Tôi sẽ kiểm tra trùng lặp của `payosOrderCode` để đảm bảo mỗi link được sinh ra là duy nhất.