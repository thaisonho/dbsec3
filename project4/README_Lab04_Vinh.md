# Vinh

Dự án áp dụng mã hóa dữ liệu cơ sở dữ liệu với phân quyền Admin/User.

- **Mật khẩu (MATKHAU):** Băm bằng **SHA-512** ở client, chỉ lưu bản băm xuống DB để xác thực.
- **Lương / Điểm thi:** Mã hóa bằng **RSA 2048**. Lưu dưới dạng Ciphertext Base64.
- **Khóa công khai (Public Key):** Lưu trực tiếp trong CSDL để người khác có thể dùng mã hóa dữ liệu gửi cho mình.
- **Khóa bí mật (Private Key): KHÔNG LƯU Ở ĐÂU CẢ.** Key được sinh ra động (on-the-fly) từ chính mật khẩu đăng nhập của người dùng. Cứ gõ đúng mật khẩu là hệ thống tự tính toán lại ra đúng Private Key cũ.

## Cấu trúc File

1. `01_create_db_lab04.sql` (Tạo DB)
2. `02_create_tables_lab04.sql` (Tạo bảng, có thêm cột VAITRO)
3. `03_create_procs_lab04.sql` (Các Stored Procedures)
4. `04_seed_and_tests_lab04.sql` (Xóa data cũ, generate data mẫu chuẩn)
5. `security_utils.py` (Thư viện xử lý mã hóa cốt lõi ở App Layer)

## Thứ tự chạy

1. Chạy `01`, `02`, `03` trong SQL Server.
2. Mở terminal, chạy file `security_utils.py`. Script này sẽ in ra các lệnh `EXEC` chứa dữ liệu đã được mã hóa chuẩn.
3. Copy toàn bộ output từ terminal đó, dán đè vào file `04_seed_and_tests_lab04.sql`.
4. Chạy file `04` trong SQL Server để hoàn tất nạp dữ liệu.

---

## `security_utils.py`

Thư viện sinh RSA Key bình thường sẽ dùng hàm random của máy tính, nên mỗi lần chạy sẽ ra một key khác nhau. Nếu quên lưu Private Key là mất trắng dữ liệu. Để giải quyết việc này, ta áp dụng cơ chế sinh key cố định (Deterministic Key Generation) kết hợp giữa **SHA-512** và **ChaCha20**:

1. **Tại sao dùng ChaCha20?**
   - Đây là một Stream Cipher (Mã hóa luồng) hiện đại của Google, cực kỳ tối ưu cho việc phun ra một luồng dữ liệu ngẫu nhiên liên tục.
   
2. **Hàm `get_deterministic_randfunc(password)` hoạt động ra sao?**: 
   - Lấy `password` gõ từ màn hình đăng nhập đem băm **SHA-512** ra 64 bytes.
   - Cắt 64 bytes này ra làm "chìa khóa" (Key & Nonce) nhét vào ChaCha20.
   - Lúc này, ChaCha20 biến thành một "máy bơm". Cứ đưa đúng password đó vào, máy bơm sẽ xịt ra một chuỗi byte y hệt nhau mọi lúc mọi nơi.
   
3. **`generate_deterministic_rsa_keypair(password)`**:
   - Thư viện RSA cần sự ngẫu nhiên để tìm số nguyên tố p và q. 
   - Thay vì để máy tính tự random, ta cắm ChaCha20 ở trên đút vào hàm sinh RSA. RSA sẽ hút luồng byte cố định đó để tạo key. 
   - **Kết quả:** Pass '123' sẽ luôn luôn tạo ra cặp RSA Key A. Pass '456' luôn tạo ra cặp RSA Key B. Không bao giờ bị mất Private Key nữa.

## Stored Procedures

### 1. Thêm nhân viên: `SP_INS_PUBLIC_ENCRYPT_NHANVIEN`
*Dùng để tạo mới user. Phải dùng Python/C# để chuẩn bị payload trước.*
- `@MANV`, `@HOTEN`, `@EMAIL`, `@TENDN`
- `@LUONG` : ciphertext RSA-2048 Base64, mã hóa bằng Public Key mới.
- `@MK` : SHA-512 hex string của mật khẩu.
- `@PUB` : public key RSA-2048 Base64 (DER/SPKI).
- `@VAITRO`: 'ADMIN' hoặc 'USER' (Mặc định là USER).

### 2. Đăng nhập: `SP_SEL_PUBLIC_ENCRYPT_NHANVIEN`
- `@TENDN` : Nhập tên đăng nhập hoặc mã nhân viên đều được.
- `@MK` : Mật khẩu đã hash SHA-512 từ client.
- **Trả về:** `MANV`, `HOTEN`, `EMAIL`, `LUONG` (Ciphertext - Client tự dùng pass đăng nhập tạo lại Private Key để giải mã), `VAITRO` (Để App biết đường ẩn/hiện nút Admin).

### 3. Cập nhật thông tin cơ bản: `SP_UPD_NHANVIEN_INFO`
*Dùng để đổi Tên, Email. Không đụng tới bảo mật.*
- `@ACTION_MANV`: Mã của người đang bấm nút Save (Dùng để check quyền).
- `@TARGET_MANV`: Mã của người bị sửa.
- `@NEW_HOTEN`, `@NEW_EMAIL`.
- **Logic:** ADMIN sửa được mọi người. USER chỉ sửa được chính `@TARGET_MANV` trùng với `@ACTION_MANV` của mình.

### 4. Đổi mật khẩu: `SP_CHANGE_PASSWORD_NHANVIEN`
*Yêu cầu bắt buộc phải gọi hàm `build_change_password_payload` ở Client trước để tính toán.*
- `@MANV`: Mã nhân viên.
- `@OLD_MK_HASH`: Hash mật khẩu cũ (Để DB xác thực lại).
- `@NEW_MK_HASH`: Hash mật khẩu mới.
- `@NEW_PUBKEY`: Public key sinh từ pass mới.
- `@NEW_LUONG`: Lương đã được mã hóa lại bằng pass mới.