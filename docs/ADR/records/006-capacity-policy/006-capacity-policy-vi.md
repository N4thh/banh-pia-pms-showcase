# ADR-006: Capacity Policy (Soft Limit & Hard Limit Alignment)

**Ngày viết:** 13/10/2026
**Trạng thái:** Đã áp dụng

---

## 1. Bối cảnh

**Bối cảnh nghiệp vụ:**
Trong mô hình kinh doanh của tiệm bánh, nhu cầu phân định giữa sản xuất tiêu chuẩn (Soft Limit) và công suất mở rộng tối đa (Hard Limit) xuất phát từ lý do sau: giúp dễ quản lý và theo dõi tình trạng, biết được có bao nhiêu đơn hàng vượt quá mức bánh thực tế, từ đó dùng thông tin này để điều chỉnh số lượng bánh cho những ngày sau.

Vùng đệm công suất mang lại hai giá trị:

- **Tăng trải nghiệm của người dùng:** Trong một ví dụ thực tế, khi khách hàng vô tình đặt vượt công suất bánh tiêu chuẩn, nếu không có các suất mở rộng thì hệ thống sẽ báo lỗi ngay lập tức dù chỉ dư một vài cái bánh.
- **Tối ưu hóa doanh thu:** Khi vận hành một tiệm bánh, các đơn đặt trước đôi lúc bị hủy vì một lý do nào đó, hậu quả là số bánh đã làm trong ngày bị dư. Một vùng đệm có thể giúp giảm hao hụt và tăng doanh thu cho quán.

**Kịch bản rủi ro nếu không có cơ chế quản lý công suất:**
Nếu hệ thống nhận đơn vượt quá khả năng sản xuất thực tế, hậu quả đối với vận hành là thợ làm bánh (mẹ tôi) sẽ phải làm việc với công suất cao hơn thường ngày, tạo nên trải nghiệm không tốt cho chính người đang quản lý tiệm bánh. Ngoài ra, chất lượng dịch vụ có thể đi xuống nếu chất lượng bánh và thời gian chăm sóc khách hàng bị giảm.

Nếu dừng nhận đơn quá sớm ngay khi vừa chạm ngưỡng tiêu chuẩn, tiệm bánh sẽ bỏ lỡ hai cơ hội. Thứ nhất là cơ hội biết được khả năng bán ra số lượng bánh cao hơn so với mức đã đề ra; tôi xem đây là chi phí cơ hội. Thứ hai là cơ hội có thêm nguồn doanh thu từ số bánh đó.

**Ràng buộc bất biến:**
Đơn hàng nằm trong khoảng công suất tiêu chuẩn sẽ được hệ thống kiểm tra xem có vượt mức hay không. Nếu không vượt, đơn được gắn mác "CONFIRMED": là đơn bình thường, đạt yêu cầu.

Đơn hàng vượt công suất tiêu chuẩn nhưng chưa vượt giới hạn tối đa được gắn mác "WAITLIST": là những đơn nằm ngoài dự tính. Những đơn này được kiểm tra xem có vượt giới hạn hay không (lớn hơn 103% công suất bình thường). Nếu vượt, hệ thống trả về lỗi Conflict Exception; nếu hợp lệ, đơn mang mác trạng thái tương ứng.

Ràng buộc tuyệt đối của hệ thống khi đơn hàng vượt quá giới hạn: tổng số lượng bánh đã đặt trong ngày hôm đó cộng với số lượng bánh khách muốn đặt không được vượt quá vùng đệm (103% tổng lượng bánh mà tiệm sẽ làm ra trong ngày). Nếu vượt, đơn đã vi phạm quy định của hệ thống và bị trả lỗi ngay; ở FE sẽ hiển thị số lượng bánh đặt bị vượt.

---

## 2. Các phương án cân nhắc

| Phương án | Ưu điểm | Nhược điểm | Vì sao không chọn |
|---|---|---|---|
| **A: Chỉ áp dụng một mức giới hạn cứng** | Đơn giản, logic dễ hiểu và không mất thời gian. | Mất đi những cơ hội mà tiệm bánh có thể có. Làm giảm trải nghiệm của người dùng khi gặp vấn đề. | Làm hệ thống quá cứng nhắc và mất đi những dữ liệu "quý giá". |
| **B: Phân tầng Soft Limit & Hard Limit có vùng đệm** | Tăng trải nghiệm người dùng, cũng như tăng một phần nhỏ doanh thu cho tiệm bánh. | Phải kiểm tra ở nhiều bước, sinh ra những ràng buộc mà hệ thống phải có để đảm bảo các biến số diễn ra trong tầm kiểm soát. | — (phương án được chọn) |

**Câu hỏi so sánh cơ chế quản lý công suất:**
Sự khác biệt giữa việc từ chối ngay lập tức khi chạm mốc tiêu chuẩn với cho phép nhận thêm trong vùng đệm có kiểm soát là gì?

Đó chính là sự "mềm mại" trong một hệ thống "cứng nhắc": nới lỏng những ràng buộc quá chặt chẽ mà các con số mang lại.

---

## 3. Quyết định

**Công thức & cấu hình công suất:**
Công thức tính vùng đệm (buffer limit) = Max Capacity × 1.03 (làm tròn lên số nguyên).

**Thứ tự kiểm tra & điều kiện chặn:**
Khi có đơn hàng mới, hệ thống tính tổng số lượng lũy kế (`newBooked`) = tổng lượng bánh đã đặt trong ngày + số lượng bánh trong đơn hàng mới.

Điều kiện kích hoạt việc ném ngoại lệ từ chối đơn hàng là khi tổng số lượng lũy kế > vùng đệm. Lúc đó hệ thống trả về phản hồi: "Hôm nay chúng tôi chưa thể xử lý thêm đơn hàng. Bạn vui lòng đặt sang ngày khác giúp chúng tôi nhé".

**Phân loại và thực tế:**
Các trạng thái "hàng chờ" (wait list) và "xác nhận" (confirmed) được sinh ra khi đơn đã vượt qua các bước ràng buộc ở trên. Nếu không vượt quá số lượng sẵn có thì là "xác nhận"; nếu nằm trong vùng đệm thì là "hàng chờ".

Nhưng hiện tại các trạng thái này chỉ mang ý nghĩa thông tin khi code; thực tế chưa có hàm nào phía sau sử dụng chúng để thực thi nghiệp vụ.

---

## 4. Đánh đổi & giới hạn

**Đánh đổi đã chấp nhận:**

- **Đánh đổi nghiệp vụ:** Việc cho phép nhận thêm đơn ở vùng đệm đem lại doanh thu nhưng vô hình chung có thể tạo áp lực cho thợ làm bánh (mẹ tôi). Tuy nhiên, vùng đệm thêm 3% là con số không quá lớn, có thể chấp nhận được để đổi lấy những cơ hội và ưu điểm mà nó mang lại.
- **Đánh đổi kỹ thuật:** Tôi chọn hardcode vùng đệm vì muốn tinh gọn và đơn giản hóa mọi thứ trong quá trình quản lý tiệm bánh cho mẹ tôi, nên tôi không muốn có thêm những bước điều chỉnh phần trăm của vùng đệm.

**Giới hạn / điểm chưa đồng bộ trong codebase hiện tại:**
Các trạng thái dùng để phân loại vùng đệm chưa được ghi xuống Database, nên việc xác định đơn nào thuộc vùng đệm chỉ có thể quan sát qua các con số. Trên thực tế, hệ thống đã thực hiện việc phân chia đó rồi nhưng không ghi nhận lại, khiến bước này bị dư thừa.

---

## 5. Những điều tôi sẽ làm khác đi

- **Thêm cột trạng thái đơn trong Database:** Bổ sung một cột dữ liệu vào Database để ghi nhận chính xác đơn nào thuộc vùng đệm, giúp minh bạch hóa dữ liệu vận hành.
- **Cấu hình linh hoạt:** Thay vì hardcode, tôi sẽ thêm một màn hình để sau này muốn thay đổi phần trăm của vùng đệm thì có thể thao tác ngay trên hệ thống, không phải tìm và đổi đúng dòng code.
- **Quy trình vận hành & giám sát:** Xây dựng cơ chế cảnh báo hoặc quy trình xử lý hàng chờ, ví dụ gửi thông báo khi số lượng đơn hàng đã đạt mức cho phép ở vùng đệm.