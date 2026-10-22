# ADR-008: Pricing & Money Integrity

**Ngày viết:** 22/10/2026
**Trạng thái:** Đã áp dụng

---

## 1. Bối cảnh

**Bối cảnh nghiệp vụ:**
Tổng tiền của một đơn là tổng giá trị của các món hàng:

`tổng = Σ ((giá gốc + số quả trứng muối × 5.000) × số lượng)`

Từ lúc khách chọn bánh đến lúc PayOS xác nhận đã thanh toán, số tiền đi qua ba bước: bỏ sản phẩm vào giỏ hàng, xác nhận thông tin đơn hàng và thanh toán.

- Ở bước giỏ hàng, FE là nơi giữ và hiển thị tổng số tiền của sản phẩm và đơn hàng mà khách phải trả. Thông tin được lưu trong localStorage, và các sản phẩm trong giỏ được lưu trong vòng 30 phút.
- Ở bước xác nhận thông tin, FE dùng lại các thông tin đang lưu trong localStorage để cho khách thấy tổng số tiền cần chi trả.
- Ở bước thanh toán, trang tải số tiền của đơn đã được tạo ở BE; số tiền này được lưu trong bảng `Order`.

**Kịch bản rủi ro nếu tiền bị sai lệch:**
Nếu số tiền sai lệch ở bất kỳ khâu nào, số tiền khách đã xác nhận và số tiền khách phải trả có thể khác nhau.

**Ràng buộc bất biến:**

- Nơi quyết định số tiền cuối cùng của một đơn là số liệu mà BE đã lưu trong database.
- Số tiền của đơn, số tiền gửi sang PayOS và số tiền trong webhook phải đồng nhất với nhau.

---

## 2. Các phương án cân nhắc

| Phương án | Ưu điểm | Nhược điểm | Vì sao không chọn |
|---|---|---|---|
| **A: Lưu bằng `Int`** | Tiền bánh không có số lẻ | Nếu sau này áp dụng thêm thuế, các khoản tính theo phần trăm sẽ rất phiền phức | Không chắc chắn |
| **B: Lưu bằng `Decimal`** | Chắc chắn hơn | Phải cấu hình đúng: Prisma trả `Decimal` dưới dạng đối tượng chứ không phải số, nên phải dùng `Number()` mỗi khi tính toán. Nếu không khai báo độ chính xác, Postgres tạo dữ liệu với 30 số lẻ | — (phương án được chọn) |

---

## 3. Quyết định

**Cách tính giá:**

- **Giá một cái bánh:** BE lấy `basePrice` của bánh từ database, rồi cộng 5.000 đồng cho mỗi quả trứng muối: `unitPrice = basePrice + eggCount × 5.000`. Hàm nằm trong `pricing.constants.ts`.
- **Tổng tiền của một đơn hàng:** trong bước `createOrder`, BE lặp qua các món, tính giá từng bánh rồi cộng `unitPrice × quantity` vào tổng: `totalMoney = Σ (unitPrice × quantity)`.

**Cách lưu và kiểu dữ liệu:**
Tiền được lưu ở các bảng và cột sau: `Order.totalMoney`, `Cake.basePrice`, `OrderItem.priceAtPurchase`, `PaymentTransaction.amount`. Việc chuyển kiểu và làm tròn nằm trong các hàm sử dụng những dữ liệu này. Tôi chọn cách này vì nó rõ ràng, rành mạch hơn nếu sau này áp dụng thuế.

**Số tiền gửi PayOS và đối chiếu webhook:**

- Số tiền gửi sang PayOS được lấy từ database, chuyển thành số bằng `Number()` và làm tròn bằng `Math.round()`.
- Khi webhook về, số tiền được so sánh với cột `totalMoney` trong bảng `Order`. Cả hai được làm tròn bằng `Math.round()` và so sánh bằng `!==`. Khi không khớp, BE ghi log lỗi và trả về `success: false` kèm message "Webhook received but amount does not match".

---

## 4. Đánh đổi & giới hạn

**Đánh đổi đã chấp nhận:**

- **Đánh đổi kỹ thuật:** Tôi đánh đổi việc phải viết chính xác từng câu lệnh gọi đến các giá trị. Vì Prisma `Decimal` trả về đối tượng nên mỗi lần dùng phải dùng `Number()`. Cột tiền không khai báo độ chính xác nên Postgres tạo ra dữ liệu với 30 số lẻ. Nhưng để chắc chắn về các con số dùng cho thuế, cũng như dễ làm tròn, tôi chọn dùng `Decimal` thay vì `Int`.
- **Đánh đổi nghiệp vụ:** Tôi không đánh đổi nhiều vì tôi cố gắng hiển thị các con số tròn nhất cho mẹ tôi.

**Giới hạn / điểm chưa đồng bộ trong codebase hiện tại:**

- Khi giá hiển thị ở FE khác với giá tính ở BE, khách thấy giá ở FE nhưng phải trả giá do BE tính.
- Khi admin đổi giá, các sản phẩm trong giỏ hàng không tự cập nhật lại giá. Đến lúc thanh toán, khách sẽ thấy giá hoàn toàn mới, vì BE vừa lấy giá mà admin mới đổi để tính.
- Khi xảy ra những sự cố quan trọng như số tiền trong webhook và số tiền trong hệ thống lệch nhau, hệ thống chưa có phần thông báo cho admin biết là đã xuất hiện lỗi.

---

## 5. Những điều tôi sẽ làm khác đi

- Tôi sẽ làm thêm một bước so sánh giá giữa FE và BE, có thể như sau: FE gửi kèm số tiền khách đã thấy, BE tự tính lại; nếu lệch thì từ chối và trả giá mới để khách xem lại.
- Khi khách vào trang xác nhận đơn hàng, tôi sẽ lấy lại giá bánh mới nhất và thông báo cho khách nếu giá có thay đổi.
- Hệ thống đã có thông báo Telegram sẵn. Lý do tôi không làm phần thông báo lỗi là vì sợ mẹ tôi nhận quá nhiều thông báo và bị rối. Nếu làm lại, tôi sẽ tách thành hai phòng chat Telegram khác nhau, và thông báo lỗi chỉ gửi cho admin, người quản lý hệ thống, để chỉ admin nắm thông tin này.