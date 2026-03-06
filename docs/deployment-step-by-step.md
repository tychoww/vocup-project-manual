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

### Toàn bộ triển khai (infra + GitLab CI/CD) nằm trong `deployment/`

**Có.** Toàn bộ quá trình triển khai (hạ tầng CloudFormation **và** CI/CD GitLab) nằm trong một thư mục **`deployment/`** ở root repo (cùng cấp với `backend/`, `frontend/`, `vocup-project-manual/`).

**Cấu trúc:**

```
EdTech/                          ← root repo
├── backend/
│   └── app-service/
├── frontend/
│   ├── edtech-home/
│   └── edtech-lms/
├── vocup-project-manual/
├── deployment/                  ← TOÀN BỘ TRIỂN KHAI (infra + CI/CD GitLab)
│   ├── README.md
│   ├── infra/                   ← CloudFormation (hạ tầng AWS)
│   │   ├── network.yaml
│   │   ├── rds.yaml
│   │   ├── secrets.yaml
│   │   ├── messaging.yaml
│   │   ├── ecs.yaml
│   │   ├── api-gateway.yaml
│   │   ├── frontend.yaml
│   │   ├── cloudfront.yaml
│   │   └── parameters/
│   │       ├── dev.json
│   │       └── ...
│   └── ci/                      ← GitLab CI/CD (pipeline)
│       └── .gitlab-ci.yml       (file pipeline; GitLab trỏ đường dẫn vào đây, không cần file ở root)
```

- **`deployment/infra/`** – Template CloudFormation và parameters. Deploy bằng AWS CLI hoặc từ pipeline.
- **`deployment/ci/`** – Cấu hình pipeline GitLab (build, test, SonarQube, deploy backend/frontend). Trong GitLab: **Settings → CI/CD → Default CI/CD configuration file** đặt là `deployment/ci/.gitlab-ci.yml` để không cần file `.gitlab-ci.yml` ở root.

**Lợi ích:** Một chỗ cho infra và CI/CD; cùng version với code; không cần repo riêng.

---

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

**Lưu ý – Nếu sau này đổi API Gateway sang ALB:** Không cần xoá hay tạo lại network. Chỉ thay stack api-gateway bằng stack ALB, cập nhật ECS và CloudFront. Chi tiết xem [deployment/README.md](../../deployment/README.md).

**Lệnh ví dụ (từ thư mục gốc repo; mỗi stack cần output từ stack trước):**

```bash
aws cloudformation deploy --template-file deployment/infra/network.yaml --stack-name edtech-dev-network --parameter-overrides ...
# Lần lượt: rds, secrets, messaging, ecs, api-gateway, frontend, cloudfront (đường dẫn deployment/infra/...)
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

Sau Bước 5, kiểm tra: mở yourdomain.com, yourdomain.com/lms và gọi thử API (login, health…). **CI/CD GitLab:** Toàn bộ pipeline nằm trong **`deployment/ci/.gitlab-ci.yml`**. Trong GitLab đặt **Settings → CI/CD → Default CI/CD configuration file** = `deployment/ci/.gitlab-ci.yml` (không cần file ở root). Khi pipeline chạy, các bước 4 và 5 được tự động hóa; lần đầu nên làm tay để quen luồng. Xem [deployment/README.md](../../deployment/README.md).
