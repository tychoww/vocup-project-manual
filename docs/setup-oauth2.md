# Hướng dẫn setup OAuth 2.0 (Google, GitHub)

Tài liệu này mô tả cách cấu hình đăng nhập OAuth 2.0 cho ứng dụng EdTech LMS (Google và GitHub).

---

## 1. Tổng quan luồng OAuth

```
[User] → [Frontend Login] → [Backend /oauth/{provider}] → [Google/GitHub Consent]
                                                                    ↓
[User] ← [Frontend OAuth Callback] ← [Backend redirect #tokens] ← [Callback]
```

1. User click "Đăng nhập với Google/GitHub" trên trang login
2. Frontend redirect sang backend: `GET /api/v1/auth/oauth/google` hoặc `.../github`
3. Backend redirect sang trang consent của Google/GitHub
4. User đăng nhập và đồng ý quyền truy cập
5. Google/GitHub redirect về backend: `GET /api/v1/auth/oauth/google/callback?code=...&state=...`
6. Backend đổi `code` lấy access token, lấy thông tin user, tạo/đăng nhập user
7. Backend redirect về frontend kèm tokens trong URL fragment: `#access_token=...&refresh_token=...`
8. Frontend lưu tokens và chuyển user vào app

---

## 2. Google OAuth

### 2.1 Tạo OAuth Client (Google Cloud Console)

1. Vào [Google Cloud Console](https://console.cloud.google.com/)
2. Tạo project mới hoặc chọn project có sẵn
3. Vào **APIs & Services** → **Credentials** → **Create Credentials** → **OAuth client ID**
4. Nếu chưa có OAuth consent screen:
   - **APIs & Services** → **OAuth consent screen** → chọn **External** (hoặc Internal cho G Suite)
   - Điền App name, User support email, Developer contact
   - Thêm scope: `email`, `profile`, `openid` (hoặc để mặc định)
5. Tạo OAuth client ID:
   - **Application type**: Web application
   - **Name**: ví dụ `EdTech LMS`
   - **Authorized redirect URIs**: thêm các URI callback (xem bên dưới)

### 2.2 Authorized redirect URIs (Google)

Redirect URI phải trỏ tới **backend** (không phải frontend):

| Môi trường | Redirect URI |
|------------|--------------|
| Local | `http://localhost:51080/api/v1/auth/oauth/google/callback` |
| Cloudflare Tunnel | `https://{backend-tunnel}.trycloudflare.com/api/v1/auth/oauth/google/callback` |
| Production | `https://api.yourdomain.com/api/v1/auth/oauth/google/callback` |

Lưu ý: URI phải **khớp chính xác** (kể cả `/` cuối hay không, http vs https).

### 2.3 Lấy Client ID và Client Secret

Sau khi tạo, copy **Client ID** và **Client secret** để cấu hình ở bước sau.

---

## 3. GitHub OAuth

### 3.1 Tạo OAuth App (GitHub)

1. Vào [GitHub](https://github.com/) → **Settings** → **Developer settings** → **OAuth Apps** → **New OAuth App**
2. Điền thông tin:
   - **Application name**: EdTech LMS
   - **Homepage URL**: `http://localhost:5173` (local) hoặc URL frontend của bạn
   - **Authorization callback URL**: trỏ tới **backend** (xem bảng dưới)

### 3.2 Authorization callback URL (GitHub)

| Môi trường | Callback URL |
|------------|--------------|
| Local | `http://localhost:51080/api/v1/auth/oauth/github/callback` |
| Cloudflare Tunnel | `https://{backend-tunnel}.trycloudflare.com/api/v1/auth/oauth/github/callback` |
| Production | `https://api.yourdomain.com/api/v1/auth/oauth/github/callback` |

### 3.3 Lấy Client ID và Client Secret

Sau khi tạo, copy **Client ID** và tạo **Client secret** mới (Generate a new client secret).

---

## 4. Cấu hình Backend

### 4.1 Biến môi trường (Environment Variables)

Có thể dùng file `.env` hoặc export trực tiếp:

```bash
# Google OAuth
GOOGLE_CLIENT_ID=your-google-client-id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=GOCSPX-xxxxxxxxxxxxxxxxxxxxxxxx

# GitHub OAuth
GITHUB_CLIENT_ID=Ov23lixxxxxxxx
GITHUB_CLIENT_SECRET=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Redirect URI (backend nhận callback từ Google/GitHub)
GOOGLE_REDIRECT_URI=http://localhost:51080/api/v1/auth/oauth/google/callback
GITHUB_REDIRECT_URI=http://localhost:51080/api/v1/auth/oauth/github/callback

# Frontend redirect (sau khi login, backend chuyển user về đâu)
AUTH_FRONTEND_REDIRECT_URI=http://localhost:5173/auth/oauth/callback
```

### 4.2 application.yaml (default)

File `backend/app-service/src/main/resources/application.yaml`:

```yaml
app:
  auth:
    frontend-redirect-uri: ${AUTH_FRONTEND_REDIRECT_URI:http://localhost:5173/auth/oauth/callback}
    google:
      client-id: ${GOOGLE_CLIENT_ID:}
      client-secret: ${GOOGLE_CLIENT_SECRET:}
      redirect-uri: ${GOOGLE_REDIRECT_URI:http://localhost:51080/api/v1/auth/oauth/google/callback}
    github:
      client-id: ${GITHUB_CLIENT_ID:}
      client-secret: ${GITHUB_CLIENT_SECRET:}
      redirect-uri: ${GITHUB_REDIRECT_URI:http://localhost:51080/api/v1/auth/oauth/github/callback}
```

Giá trị trong `${VAR:default}` ưu tiên biến môi trường, fallback về default.

### 4.3 CORS

Đảm bảo `allowed-origins` bao gồm domain frontend:

```yaml
app:
  cors:
    allowed-origins: ${APP_CORS_ALLOWED_ORIGINS:http://localhost:5173}
```

Khi dùng Cloudflare Tunnel, thêm domain tunnel vào:  
`http://localhost:5173,https://chemistry-anatomy-have-spoke.trycloudflare.com`

---

## 5. Cấu hình Frontend

### 5.1 API Base URL

File `frontend/edtech-lms/.env`:

```bash
# Local
VITE_API_BASE_URL=http://localhost:51080

# Cloudflare Tunnel (thay URL backend tunnel)
VITE_API_BASE_URL=https://aimed-metal-segment-bizarre.trycloudflare.com
```

Frontend dùng `VITE_API_BASE_URL` để build URL OAuth:  
`{API_BASE_URL}/api/v1/auth/oauth/google` và `/api/v1/auth/oauth/github`

### 5.2 OAuth Callback Route

Frontend có route `/auth/oauth/callback` để nhận redirect từ backend (kèm tokens trong fragment).

Cần set `AUTH_FRONTEND_REDIRECT_URI` trên backend trỏ đúng URL callback frontend:

- Local: `http://localhost:5173/auth/oauth/callback`
- Cloudflare: `https://{frontend-tunnel}.trycloudflare.com/auth/oauth/callback`

---

## 6. Local Development

### 6.1 Chạy local (không tunnel)

1. Backend: `http://localhost:51080`
2. Frontend: `http://localhost:5173`

Cấu hình:

- `GOOGLE_REDIRECT_URI=http://localhost:51080/api/v1/auth/oauth/google/callback`
- `GITHUB_REDIRECT_URI=http://localhost:51080/api/v1/auth/oauth/github/callback`
- `AUTH_FRONTEND_REDIRECT_URI=http://localhost:5173/auth/oauth/callback`
- `VITE_API_BASE_URL=http://localhost:51080`

### 6.2 Chạy với Cloudflare Tunnel (chia sẻ public)

Xem `cloudflared-public-app.txt` để chạy tunnel.

**Frontend tunnel** (ví dụ `chemistry-anatomy-have-spoke.trycloudflare.com`):
```bash
cloudflared tunnel --url http://localhost:5173
```

**Backend tunnel** (ví dụ `aimed-metal-segment-bizarre.trycloudflare.com`):
```bash
cloudflared tunnel --url http://localhost:51080
```

Cấu hình:

- `GOOGLE_REDIRECT_URI=https://aimed-metal-segment-bizarre.trycloudflare.com/api/v1/auth/oauth/google/callback`
- `GITHUB_REDIRECT_URI=https://aimed-metal-segment-bizarre.trycloudflare.com/api/v1/auth/oauth/github/callback`
- `AUTH_FRONTEND_REDIRECT_URI=https://chemistry-anatomy-have-spoke.trycloudflare.com/auth/oauth/callback`
- `VITE_API_BASE_URL=https://aimed-metal-segment-bizarre.trycloudflare.com`
- `APP_CORS_ALLOWED_ORIGINS=http://localhost:5173,https://chemistry-anatomy-have-spoke.trycloudflare.com`

**Lưu ý:** URL tunnel thay đổi mỗi lần chạy `cloudflared` (trừ khi dùng named tunnel). Cần cập nhật lại trong Google/GitHub OAuth settings và env mỗi lần tunnel mới.

---

## 7. Production

### 7.1 Redirect URI cố định

Dùng domain cố định cho backend, ví dụ `https://api.edtech.com`:

- Google: thêm `https://api.edtech.com/api/v1/auth/oauth/google/callback`
- GitHub: `https://api.edtech.com/api/v1/auth/oauth/github/callback`
- `AUTH_FRONTEND_REDIRECT_URI=https://app.edtech.com/auth/oauth/callback`

### 7.2 Bảo mật

- Không commit `client-secret` vào git
- Dùng biến môi trường hoặc secrets manager
- JWT secret: set `APP_AUTH_JWT_SECRET` mạnh và bảo mật

---

## 8. Xử lý lỗi thường gặp

| Lỗi | Nguyên nhân | Cách xử lý |
|-----|-------------|------------|
| `redirect_uri_mismatch` (Google) | Redirect URI trong request khác với URI đăng ký trong Google Console | Kiểm tra `GOOGLE_REDIRECT_URI` và cấu hình trong Google Console |
| `Application with identifier ... was not found` (GitHub) | Client ID sai hoặc app bị xóa | Kiểm tra `GITHUB_CLIENT_ID` và OAuth App trên GitHub |
| CORS error | Frontend domain chưa có trong `allowed-origins` | Thêm domain frontend vào `APP_CORS_ALLOWED_ORIGINS` |
| Sau login không nhận được tokens | `AUTH_FRONTEND_REDIRECT_URI` sai hoặc frontend không parse fragment | Kiểm tra redirect URI và component `OAuthCallback.tsx` |
| `invalid_grant` / `code already used` | Code OAuth chỉ dùng 1 lần, user refresh hoặc quay lại | User cần thực hiện đăng nhập OAuth lại từ đầu |

---

## 9. Tóm tắt biến môi trường

| Biến | Mô tả | Ví dụ |
|------|-------|-------|
| `GOOGLE_CLIENT_ID` | Google OAuth Client ID | `xxx.apps.googleusercontent.com` |
| `GOOGLE_CLIENT_SECRET` | Google OAuth Client Secret | `GOCSPX-xxx` |
| `GOOGLE_REDIRECT_URI` | URL backend nhận callback Google | `http://localhost:51080/api/v1/auth/oauth/google/callback` |
| `GITHUB_CLIENT_ID` | GitHub OAuth App Client ID | `Ov23li...` |
| `GITHUB_CLIENT_SECRET` | GitHub OAuth App Client Secret | `xxx` |
| `GITHUB_REDIRECT_URI` | URL backend nhận callback GitHub | `http://localhost:51080/api/v1/auth/oauth/github/callback` |
| `AUTH_FRONTEND_REDIRECT_URI` | URL frontend sau khi login (nhận tokens) | `http://localhost:5173/auth/oauth/callback` |
| `VITE_API_BASE_URL` | URL backend cho frontend | `http://localhost:51080` |
