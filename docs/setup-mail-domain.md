# Hướng dẫn thiết lập Zoho Mail cho domain vocup.net

## Phần 1: Khởi tạo và Xác minh Tên miền

### Welcome to Zoho Mail (Thêm domain)

- Ở ô **Provide your existing domain name**: xóa chữ `www.` và điền **vocup.net**.
- **Organization name**: Điền tên tổ chức (VD: Genix Tech).
- **Industry Type**: Chọn ngành nghề (IT Services / Education) rồi nhấn tiếp tục.

### Chọn gói cước (Pricing)

- Tại màn hình hiện số lượng User và Add-ons, điền **1** vào **No. of Users**.
- **Lưu ý**: Nếu màn hình báo phí $15, tìm và chọn chuyển sang **Forever Free Plan** để Subtotal về **$0.00**.

### Verify Your Domain Ownership (Xác minh sở hữu)

1. Tại màn hình này, chọn nút **Configure manually** (Cấu hình thủ công).
2. Zoho cung cấp một bảng chứa mã TXT. Copy mã ở cột **Value**, ví dụ: `zoho-verification=zb05123065.zmverify.zoho.com`.
3. Mở tab **Cloudflare** → **DNS** → **Records** của **vocup.net**.
4. Bấm **Add record**:
   - **Type**: TXT  
   - **Name**: `@`  
   - **Content**: dán mã Zoho vừa copy  
   - Nhấn **Save**.
5. Quay lại tab Zoho, nhấn **Verify TXT Record**. Màn hình hiện tích xanh lá báo hiệu thành công.

---

## Phần 2: Thiết lập Tài khoản

### Tạo Super Administrator

- Tại ô **Your login email address**, điền tên user mong muốn và nhấn **Create**.
- Ví dụ: tạo hòm thư **support@vocup.net** cho Genix Tech.

### Setup Users & Setup Groups

- **Setup Users**: Hệ thống hiển thị user vừa tạo (VD: support@vocup.net). Nhấn **Proceed to Setup Groups** ở góc dưới cùng.
- **Setup Groups**: Nếu chưa có nhu cầu tạo nhóm, bỏ qua bằng cách nhấn **Proceed to DNS Mapping**.

---

## Phần 3: Cấu hình DNS Mapping (Thông nòng Gửi/Nhận)

### Lấy thông số từ Zoho

- Tại màn hình **DNS Mapping**, chọn **Configure manually**.
- Zoho hiện bảng gồm **5 bản ghi** cần cấu hình: 3 MX, 1 SPF, 1 DKIM.

### Thêm bản ghi vào Cloudflare

**Quan trọng**: Tất cả bản ghi đều để **DNS only** (đám mây xám), **không** bật Proxy (màu cam).

| # | Type | Name | Nội dung |
|---|------|------|----------|
| 1 | MX | `@` | Mail Server: `mx.zoho.com` — Priority: **10** |
| 2 | MX | `@` | Mail Server: `mx2.zoho.com` — Priority: **20** |
| 3 | MX | `@` | Mail Server: `mx3.zoho.com` — Priority: **50** |
| 4 | TXT (SPF) | `@` | `v=spf1 include:zohomail.com ~all` |
| 5 | TXT (DKIM) | `zmail._domainkey` | Dán toàn bộ chuỗi từ Zoho (bắt đầu bằng `v=DKIM1; k=rsa; p=MIGf...`) |

- Trong Cloudflare: **Add record** lần lượt cho từng bản ghi theo bảng trên.

### Xác nhận DNS (Verify all records)

1. Quay lại Zoho, cuộn xuống và nhấn **Verify all records**.
2. Chờ các dấu chấm than đỏ ở cột **Status** chuyển thành tích xanh.
3. Nhấn **Proceed to Email Migration**.

---

## Phần 4: Về đích

### Data Migration (Chuyển dữ liệu)

- Vì là hòm thư mới, không có dữ liệu cũ: **không** bấm "Start data migration".
- Nhấn **Proceed to Go Mobile** ở góc dưới.

### Go Mobile (Thông số kết nối app)

- Màn hình hiển thị thông số **IMAP / POP / SMTP** để đăng nhập trên ứng dụng bên thứ ba (Outlook, Apple Mail, v.v.).
- Nhấn **Proceed to Setup Completion** để kết thúc quá trình cài đặt.
