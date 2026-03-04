# Setup Email Notification (SES + SNS + SQS + Lambda)

Luồng: App → SNS → SQS → Lambda → SES (gửi mail verify email, quên mật khẩu). Dùng Gmail cá nhân làm địa chỉ gửi (verified trong SES).

---

## 1. Verify Gmail trong Amazon SES

- Vào **AWS Console** → **Amazon SES** (region **ap-southeast-1** – Singapore).
- Menu trái: **Verified identities** → **Create identity**.
- Chọn **Email address**.
- Nhập Gmail (vd: `nguyenchaudz1998@gmail.com`) → **Create identity**.
- Mở hộp thư Gmail → mở mail từ AWS → bấm link xác thực. Trạng thái identity chuyển **Verified**.

---

## 2. Tạo SNS topic (nếu chưa có)

- **Amazon SNS** → **Topics** → **Create topic**.
- **Type**: Standard. **Name**: `edtech-email-transactional`.
- **Create topic** → copy **Topic ARN** (vd: `arn:aws:sns:ap-southeast-1:682761214020:edtech-email-transactional`).

---

## 3. Tạo SQS queue (nếu chưa có)

- **Amazon SQS** → **Create queue**.
- **Type**: Standard. **Name**: `edtech-email-transactional-queue`.
- Các mục khác mặc định → **Create queue**.
- Vào queue vừa tạo → **Access policy** → **Edit**.
- Dùng policy cho phép SNS gửi vào queue (AWS có mẫu “Allow SNS to send to SQS”). **Save**.
- Copy **Queue ARN** (vd: `arn:aws:sqs:ap-southeast-1:682761214020:edtech-email-transactional-queue`).

---

## 4. Subscribe SQS vào SNS

- **Amazon SNS** → **Topics** → chọn topic `edtech-email-transactional`.
- **Subscriptions** → **Create subscription**.
- **Topic ARN**: đã chọn. **Protocol**: **Amazon SQS**. **Endpoint**: dán **Queue ARN** của bước 3.
- (Nên bật) **Enable raw message delivery** → **Create subscription**.

---

## 5. Tạo Lambda và cấu hình

- **AWS Lambda** → **Create function**.
- **Name**: `edtech-email-transactional` (thống nhất với topic/queue; rõ là email giao dịch: verify, forgot password).
- **Runtime**: Python 3.11 (hoặc 3.12). **Architecture**: arm64 (Graviton2, rẻ hơn và thường nhanh hơn x86_64).
- **Create function**.
- **Code**: dán/upload code từ `backend/lambda-email-transactional/lambda_function.py`, **Deploy**.
- **Configuration** → **Environment variables** → **Edit**:
  - `SENDER_EMAIL` = Gmail đã verify (vd: `nguyenchaudz1998@gmail.com`).
  - `APP_NAME` = `EdTech` (hoặc tên app).
- (Nếu code đã hardcode `SENDER_EMAIL` và `APP_NAME` trong file thì có thể bỏ qua env.)
- **Configuration** → **Permissions**: role của Lambda cần quyền SES và SQS (xem bước 6).

---

## 6. Gắn quyền SQS + SES cho role Lambda (làm trước khi tạo trigger)

**Lưu ý:** Đây là cấp quyền cho **Execution role** (role Lambda dùng khi chạy), **không phải** màn hình "Add permissions" với Principal / Source ARN / Action `lambda:InvokeFunction` (màn hình đó thuộc bước 7 – resource-based policy).

**Execution role** cần quyền đọc SQS và gửi SES. Nếu chưa cấp, khi thêm SQS trigger sẽ báo: *"The function execution role does not have permissions to call ReceiveMessage on SQS"*.

1. Lambda → **Configuration** → **Permissions**.
2. Ở phần **Execution role**, bấm vào **Role name** (link mở sang IAM).
3. Trong IAM (trang role): **Add permissions** (nút dropdown) → chọn **Attach policies**.
4. Trên màn hình **Attach policies**:
   - Ô **Filter policies** / tìm kiếm: gõ `AmazonSES` → tick chọn **AmazonSESFullAccess**.
   - Gõ tiếp `AmazonSQS` (hoặc xóa filter rồi gõ) → tick chọn **AmazonSQSFullAccess**.
   - Hoặc trong ô search gõ từng tên: `SES`, tick **AmazonSESFullAccess**; sau đó gõ `SQS`, tick **AmazonSQSFullAccess**.
5. Bấm **Add permissions** (nút cam ở dưới) để gắn hai policy vào role.
6. Sau đó quay lại Lambda và thực hiện **bước 7** (gắn SQS trigger).

---

## 7. Gắn SQS làm trigger cho Lambda

- Vào **Lambda** → chọn function `edtech-email-transactional` → **Configuration** → **Triggers** → **Add trigger**.
- Chọn **SQS** → chọn queue `edtech-email-transactional-queue` → **Add** (AWS có thể tự thêm permission cho SQS gọi Lambda).

Nếu cần **thêm permission thủ công** (màn hình "Add permissions" / resource-based policy), điền lần lượt:

| Ô / Field | Điền / Chọn |
|-----------|-------------|
| **Permission type** | Chọn radio **AWS service** (không chọn AWS account hay Function URL). |
| **Service** | Dropdown: chọn **SQS**. |
| **Statement ID** | Gõ ví dụ: `SQS-invoke-edtech-email-transactional` (hoặc `my-custom-id-0001`). |
| **Principal** | Gõ: `sqs.amazonaws.com`. |
| **Source ARN** | Gõ ARN queue, ví dụ: `arn:aws:sqs:ap-southeast-1:682761214020:edtech-email-transactional-queue` (thay `682761214020` bằng Account ID của bạn). Lấy ARN ở SQS → chọn queue → phần Details. |
| **Action** | Dropdown "Choose an action to allow": chọn **lambda:InvokeFunction**. Chỉ cần một action này, không thêm action khác. |
| | Bấm **Save** (nút cam). |

---

## 8. Cấu hình app-service (backend)

Trong `application.yaml` (hoặc biến môi trường):

- `aws.region`: `ap-southeast-1`
- `aws.sns.email-transactional-topic-arn`: **Topic ARN** từ bước 2
- Nếu chạy local: set `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` (hoặc dùng default profile)

Đảm bảo app-service có quyền **SNS:Publish** lên topic đó (IAM user/role dùng khi chạy app).

---

## 9. Kiểm tra

- Gọi API đăng ký (hoặc quên mật khẩu) với email **đã verify trong SES** (trong sandbox SES chỉ gửi được tới email đã verify).
- Kiểm tra **SQS** → queue có message (rồi biến mất khi Lambda xử lý).
- **CloudWatch** → Log group của Lambda: xem log “Sent email type=…”.
- Hộp thư (Gmail người nhận) → có mail verify/forgot password, From = Gmail đã verify của bạn.

---

## Xử lý lỗi thường gặp

**"Couldn't configure trigger for queue" / "The function execution role does not have permissions to call ReceiveMessage on SQS"**

- Nguyên nhân: Execution role của Lambda chưa có quyền SQS (ReceiveMessage, DeleteMessage, GetQueueAttributes).
- Cách xử lý: Làm đủ **bước 6** (cấp quyền): IAM → role của Lambda → **Attach policies** → gắn **AmazonSQSFullAccess** (hoặc policy tối thiểu có `sqs:ReceiveMessage`, `sqs:DeleteMessage`, `sqs:GetQueueAttributes`). Sau đó thử lại **bước 7** (Trigger AWS Lambda function → Save).
