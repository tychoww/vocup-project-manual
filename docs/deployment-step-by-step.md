# Triển khai từng bước (Deployment Step-by-Step)

Tài liệu này nối với [system-design.md](system-design.md). Thứ tự thực hiện: **Chuẩn bị → Domain & Mail → Hạ tầng AWS → DNS → Backend → Frontend**.

---

## Trước khi bắt đầu – Đầu tiên bạn cần làm gì

**Bước 0: Chuẩn bị (làm trước mọi thứ)**

| # | Việc cần làm | Chi tiết |
|---|----------------|----------|
| 1 | **Tài khoản AWS** | Đăng ký [AWS](https://aws.amazon.com/), bật MFA. Chọn region triển khai (ví dụ **ap-southeast-1** Singapore). |
| 2 | **Cài đặt công cụ local** | [AWS CLI](https://aws.amazon.com/cli/) (v2), [Docker](https://docs.docker.com/get-docker/). Cấu hình `aws configure` (Access Key, Secret, region). |
| 3 | **Tài khoản Cloudflare** | Đăng ký [Cloudflare](https://dash.cloudflare.com/sign-up) (dùng cho domain + DNS). |
| 4 | **Quyết định tên miền** | Chọn tên miền sẽ dùng (ví dụ `yourdomain.com`). Sẽ mua/transfer tại Cloudflare ở Bước 1. |
| 5 | **Mã nguồn sẵn sàng** | Repo có `backend/app-service`, `frontend/edtech-home`, `frontend/edtech-lms`. Backend build được (`mvn package`), frontend build được (`npm run build`). |

Sau khi xong Bước 0, làm lần lượt từ **Bước 1** trở đi.

---

## Bước 1: Domain (web) và Mail

**Mục tiêu:** Có tên miền và (tùy chọn) email @yourdomain.com.

| # | Việc cần làm | Ghi chú |
|---|----------------|----------|
| 1.1 | **Mua hoặc chuyển domain tại Cloudflare** | [Cloudflare Registrar](https://www.cloudflare.com/products/registrar/) – mua mới hoặc transfer domain về. DNS sẽ do Cloudflare quản lý. |
| 1.2 | **(Tùy chọn) Zoho Mail** | Nếu cần email kiểu hello@yourdomain.com: đăng ký [Zoho Mail](https://www.zoho.com/mail/), thêm domain. Bản ghi MX (và SPF/DKIM) sẽ cấu hình ở Bước 4 khi đã có DNS tại Cloudflare. |

**Kết quả:** Bạn có domain (ví dụ `yourdomain.com`) và có thể đăng nhập Cloudflare Dashboard → DNS. Chưa cần trỏ A/CNAME đi đâu; trỏ sau khi có CloudFront và API Gateway (Bước 4).

---

## Bước 2: Hạ tầng AWS (CloudFormation)

**Mục tiêu:** Tạo VPC, RDS, ECS, API Gateway, S3, CloudFront, SNS/SQS/Lambda (email) theo thứ tự trong [system-design.md](system-design.md) mục 2.2.2.

**Điều kiện:** Đã có thư mục `infrastructure/` với các file YAML (network, rds, secrets, messaging, ecs, api-gateway, frontend, cloudfront). Nếu chưa có, cần viết hoặc copy template trước.

**Thứ tự deploy stack (bắt buộc):**

| Thứ tự | Stack | Nội dung chính |
|--------|--------|-----------------|
| 1 | **network** | VPC, Subnets (public/private), Security Groups, VPC Link. |
| 2 | **rds** | RDS PostgreSQL (t4g.micro, 20 GB gp3) trong Private Subnet. |
| 3 | **secrets** | Secrets Manager (DB, JWT, OAuth, SNS topic ARN…). |
| 4 | **messaging** | SNS topic, SQS, Lambda gửi email qua SES. Output: SNS Topic ARN. |
| 5 | **ecs** | ECR, ECS Cluster, Cloud Map, Task Definition, ECS Service (Fargate Spot). |
| 6 | **api-gateway** | HTTP API, route `/api/*`, VPC Link → Cloud Map. |
| 7 | **frontend** | S3 buckets (edtechhome, edtechlms), bucket policy cho CloudFront. |
| 8 | **cloudfront** | Distribution: origins S3 + API Gateway, behaviors `/`, `/lms/*`, `/api/*`. |

**Lệnh ví dụ (sau mỗi stack cần output cho stack sau):**

```bash
aws cloudformation deploy --template-file infrastructure/network.yaml --stack-name edtech-dev-network --parameter-overrides ...
# Lần lượt: rds, secrets, messaging, ecs, api-gateway, frontend, cloudfront
```

**Sau Bước 2:** Ghi lại **CloudFront distribution URL** (ví dụ `d1234abcd.cloudfront.net`) và **API Gateway URL** (nếu dùng custom domain thì sau Bước 4 mới có api.yourdomain.com).

---

## Bước 3: Cấu hình DNS tại Cloudflare

**Mục tiêu:** Trỏ domain về CloudFront và (nếu dùng) subdomain api về API.

| # | Việc cần làm | Ghi chú |
|---|----------------|----------|
| 3.1 | **Trang web (root / www)** | Trong Cloudflare DNS: thêm bản ghi **CNAME** `www` → `xxxx.cloudfront.net` (distribution CloudFront). Root `@` có thể dùng CNAME flatten hoặc A (IP) theo hướng dẫn Cloudflare. |
| 3.2 | **API (nếu dùng subdomain)** | CNAME `api` → URL API Gateway custom domain (đã gắn certificate ACM). Hoặc dùng chung domain và path `/api/*` qua CloudFront (không cần bản ghi api riêng). |
| 3.3 | **(Đã dùng Zoho ở Bước 1.2)** | Thêm bản ghi **MX** (và SPF/DKIM nếu Zoho yêu cầu) theo hướng dẫn Zoho Mail. |

**Kết quả:** User gõ yourdomain.com → Cloudflare DNS → CloudFront → S3 / API Gateway.

---

## Bước 4: Deploy Backend (app-service)

**Mục tiêu:** Build image từ `backend/app-service`, đẩy lên ECR, cập nhật ECS Service để chạy container mới.

| # | Việc cần làm | Ghi chú |
|---|----------------|----------|
| 4.1 | **Build JAR** | Trong `backend/app-service`: `mvn clean package -DskipTests` (hoặc chạy test). Ra file `target/app-service.jar`. |
| 4.2 | **Build Docker image** | `docker build -t edtech-app-service .` (Dockerfile copy JAR, Corretto 21). |
| 4.3 | **Tag và push ECR** | Login ECR, tag image theo ECR URI, `docker push`. |
| 4.4 | **Cập nhật ECS** | Cập nhật Task Definition (image mới) hoặc force new deployment ECS Service. ECS sẽ kéo image mới và Rolling Update. |

**Lưu ý:** Task Definition cần biến môi trường: DB URL (từ Secrets), `AWS_SNS_EMAIL_TRANSACTIONAL_TOPIC_ARN` (từ stack messaging). Kiểm tra trong AWS Console ECS → Task Definition.

**Kết quả:** API `/api/*` trả về từ ECS (qua API Gateway / CloudFront).

---

## Bước 5: Deploy Frontend (edtech-home + edtech-lms)

**Mục tiêu:** Build và upload trang marketing + LMS lên S3, invalidation CloudFront.

| # | Việc cần làm | Ghi chú |
|---|----------------|----------|
| 5.1 | **edtech-home** | Trong `frontend/edtech-home`: `npm ci && npm run build`. Upload output (ví dụ `out/` hoặc `.next/static` + HTML) lên S3 bucket **edtechhome**. Invalidation CloudFront path `/` (và path marketing nếu có). |
| 5.2 | **edtech-lms** | Trong `frontend/edtech-lms`: `npm ci && npm run build`. Upload thư mục `dist/` lên S3 bucket **edtechlms**. Invalidation CloudFront path `/lms/*`. |

**Cấu hình API base URL:** Ở edtech-lms, cấu hình base URL API trỏ tới domain (ví dụ `https://api.yourdomain.com` hoặc `https://yourdomain.com/api`).

**Kết quả:** Truy cập yourdomain.com → trang marketing; yourdomain.com/lms → ứng dụng LMS; API do backend (Bước 4) phục vụ.

---

## Tóm tắt thứ tự

| Bước | Nội dung |
|------|----------|
| **0** | Chuẩn bị: AWS, Cloudflare, công cụ, tên miền, mã nguồn build được. |
| **1** | Domain (Cloudflare) + (tùy chọn) Zoho Mail. |
| **2** | Deploy CloudFormation: network → rds → secrets → messaging → ecs → api-gateway → frontend → cloudfront. |
| **3** | DNS Cloudflare: trỏ domain và (nếu dùng) api, MX cho Zoho. |
| **4** | Deploy backend: build JAR → Docker → ECR → ECS. |
| **5** | Deploy frontend: build edtech-home + edtech-lms → upload S3 → invalidation CloudFront. |

Sau Bước 5, kiểm tra: mở yourdomain.com, yourdomain.com/lms và gọi thử API (login, health…). Nếu dùng CI/CD (GitLab/CodePipeline), các bước 4 và 5 sẽ được tự động hóa khi push code; lần đầu vẫn nên làm tay theo các bước trên để quen luồng.
