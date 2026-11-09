# ADR-010: JWT Rotation & Redis Blacklist

**Ngày viết:** 02/11/2026
**Trạng thái:** Đã áp dụng

## 1. Bối cảnh

Hệ thống có những chức năng và thông tin chỉ admin mới được dùng và xem, ví dụ: thêm slot bánh, xem tất cả đơn hàng, xem chi tiết doanh thu, đổi trạng thái đơn, xem báo cáo doanh thu theo một hoặc nhiều tuần.

Thông tin của cửa hàng và của khách hàng là mục tiêu chính cần bảo vệ. Ngoài ra cần đảm bảo chỉ đúng người có vai trò admin mới thực hiện được các tác vụ quan trọng, vì vậy hệ thống cần xác thực và phân quyền cho người truy cập.

Admin đăng nhập bằng tài khoản và mật khẩu. Mật khẩu được lưu dạng hash trong DB; khi đăng nhập, hệ thống dùng hàm `compare` của `bcrypt` để so mật khẩu nhập vào với hash, và tài khoản cũng phải khớp. Nếu khớp, hệ thống cấp token. Các request admin sau đó được kiểm tra bằng token (xem mục 3), không so mật khẩu lại.

## 2. Các phương án cân nhắc

Lúc viết code tôi không cân nhắc phương án nào khác. Tôi dùng JWT để xác minh danh tính truy cập, lưu ở `localStorage`, kết hợp với blacklist lưu ở Redis.

## 3. Quyết định

Hệ thống phát ra Access Token (AT) và Refresh Token (RT). AT có thời hạn 15 phút, RT có thời hạn 7 ngày. AT dùng như "tấm vé" để được thực hiện các tác vụ của admin. RT dùng để xin AT mới khi AT hết hạn.

Một request admin được kiểm tra theo thứ tự sau: endpoint có phải là `public` hay không; nếu không, xác minh chữ ký và thời hạn của AT; sau đó kiểm tra `jti` của AT có nằm trong blacklist hay không; cuối cùng `AdminGuard` kiểm tra endpoint có yêu cầu admin hay không (`IS_ADMIN_KEY`) và `role` có phải là `ADMIN` hay không. Nếu tất cả thỏa thì hàm nghiệp vụ mới được chạy.

Khi đăng xuất, AT và RT (nếu có) được thêm vào blacklist, lưu ở Redis với khóa `blacklist:${jti}` và TTL bằng thời gian còn lại của token. Khi làm mới token, hệ thống cấp cặp AT và RT mới, và RT cũ được thêm vào blacklist.

FE lưu token ở `localStorage`. Khi nhận lỗi `401`, FE thử làm mới token bằng RT; nếu thất bại thì xóa token khỏi `localStorage`.

## 4. Đánh đổi & giới hạn

Sau khi làm mới token, tôi chỉ thêm RT cũ vào blacklist, chưa thêm AT cũ. Hậu quả là AT cũ nếu còn thời hạn thì vẫn dùng được để truy cập, tối đa 15 phút.

Nếu 2 yêu cầu làm mới cùng một RT đến gần như cùng lúc, và token còn thời hạn, thì cả 2 đều đọc thấy token chưa nằm trong blacklist nên đều được qua. Hậu quả là cả 2 đều tạo cặp token mới, dư một cặp.

Khi RT đã dùng bị gửi lại, hệ thống kiểm tra thấy `jti` nằm trong blacklist thì báo lỗi ngay. Việc này có thể do có người dùng lại RT cũ (RT có thể đã bị lộ), nhưng cũng có thể do hai yêu cầu đồng thời hoặc client gửi lại. Với code hiện tại hệ thống chưa phân biệt được các trường hợp này.

Khi Redis không kết nối được lúc kiểm tra quyền hoặc lúc làm mới token, request không đi tiếp được vì không đọc được blacklist. Do chưa có try/catch cho trường hợp này nên lỗi không được xử lý theo ý tôi muốn. Phản hồi cuối cùng mà client nhận chưa được kiểm thử.

Dữ liệu blacklist không được lưu bền: file Compose không khai báo volume hay cấu hình lưu dữ liệu cho Redis. Suy ra từ cấu hình (chưa kiểm chứng): nếu Redis mất dữ liệu thì các token đã đăng xuất dùng lại được tới khi hết hạn (AT tối đa 15 phút, RT tối đa 7 ngày).

Khi RT hết hạn hoặc sai chữ ký được gửi tới endpoint làm mới, `verifyAsync` nằm ngoài try/catch và code không chuyển lỗi thành `UnauthorizedException`. Suy ra từ code (chưa kiểm chứng): client nhận `500` thay vì `401`.

Token lưu ở `localStorage` và repo chưa có cấu hình CSP hay header bảo mật. Nếu trang admin bị chèn script độc hại, script có thể đọc được token.

Về đăng nhập và phân quyền: route đăng nhập không giới hạn số lần thử mật khẩu; `MinLength(6)` của `AdminDto` chưa được thực thi vì không có `ValidationPipe` trên route này; và `AdminGuard` cho qua khi endpoint thiếu `IS_ADMIN_KEY`, nên quyền phụ thuộc vào việc từng route có được gắn cờ hay không.

Trong lúc rà soát phân quyền, tôi phát hiện ba route cakes và `/availability/book` không có guard hiệu lực: đặt guard ở service không bảo vệ các route. Tôi đã chuyển guard sang các route bị ảnh hưởng. Ban đầu, 12 trong 14 ca test thất bại; sau khi sửa, cả 14 ca đều thành công.

Test cho `AuthService`, `AuthController` và `RedisService` vẫn chỉ có `should be defined`. `JwtBlacklistGruard` được kiểm thử với handler public, request thiếu token (`401`), sai role (`403`) và token đã bị blacklist (`401`). Vẫn chưa có test cho đăng nhập, làm mới, đăng xuất hay lỗi Redis.

## 5. Những điều tôi sẽ làm khác đi

Vấn đề đặt guard được nêu ở mục 4 đã được xử lý. Các hướng dưới đây chưa được thực hiện trong code.

1. **AT cũ còn dùng được sau khi làm mới.** Gửi kèm AT trong request làm mới, hoặc đọc `jti` của AT từ header `Authorization`, rồi thêm `jti` đó vào blacklist cùng lúc với RT cũ. Thay đổi ở endpoint `/auth/refresh` và `AuthService`. Chi phí là thêm một lần ghi Redis cho mỗi lần làm mới.

2. **Hai yêu cầu làm mới đồng thời.** Gộp bước đọc và bước ghi blacklist thành một lệnh duy nhất ở Redis (`SET ... EX ... NX`); nếu lệnh báo key đã tồn tại thì từ chối request. Thay đổi ở `RedisService` và hàm `refreshToken`. Đây là ý tôi nghĩ ra sau khi đọc lại code, không phải điều tôi cân nhắc lúc viết.

3. **RT đã dùng bị gửi lại.** Ghi log cảnh báo khi gặp tình huống này để có dữ liệu phân biệt nguyên nhân. Chưa thu hồi toàn bộ chuỗi token tự động, vì với một tài khoản admin và một tiệm, việc khóa nhầm do hai yêu cầu đồng thời gây phiền hơn lợi ích. Thay đổi ở `AuthService`.

4. **Blacklist không lưu bền.** Bật lưu dữ liệu cho Redis (`appendonly`) và gắn volume cho container Redis trong `docker-compose.production.yml`, để khởi động lại không làm mất danh sách thu hồi.

5. **Redis lỗi.** Với các route admin, giữ hướng chặn request khi không đọc được blacklist, vì các route này chứa dữ liệu khách và chỉ có một admin nên chấp nhận gián đoạn ngắn. Bổ sung try/catch để trả lỗi rõ ràng (ví dụ `503`) thay vì lỗi không kiểm soát. Thay đổi ở `JwtBlacklistGruard` và `AuthService`. Quyết định này khác hướng "cho qua" ở ADR-009, vì mức nhạy cảm của dữ liệu khác nhau.

6. **Mã lỗi khi RT hết hạn hoặc sai chữ ký.** Bọc `verifyAsync` trong try/catch và chuyển lỗi thành `UnauthorizedException` để client nhận `401`. Thay đổi ở hàm `refreshToken`. Trước đó tôi sẽ chạy thử trên môi trường local để xác nhận mã lỗi thực tế.

7. **Token ở `localStorage`.** Ở quy mô hiện tại (một tài khoản admin, truy cập từ thiết bị của chủ tiệm), tôi chấp nhận. Nếu mở rộng, tôi sẽ thêm CSP ở Nginx và cân nhắc lưu RT trong cookie `HttpOnly`. Đây là ý tôi nghĩ ra sau khi đọc lại code.