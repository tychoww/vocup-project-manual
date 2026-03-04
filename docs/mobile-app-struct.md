# Cấu trúc ứng dụng mobile EdTech (edtech-mobile)

Tài liệu mô tả kiến trúc và cấu trúc thư mục của app React Native (Expo) đồng bộ phong cách với bản web edtech-lms, dễ mở rộng và bảo trì.

---

## 1. Tổng quan

- **Công nghệ:** React Native + Expo (SDK ~52), Expo Router (file-based routing).
- **API:** Axios (đồng bộ với edtech-lms). **Style:** NativeWind (Tailwind CSS cho RN), theme màu trong `tailwind.config.js`. **Icons:** `lucide-react-native` (cùng bộ với web `lucide-react`).
- **Mục tiêu:** Giao diện và màu sắc đồng điệu với web; một bộ theme dùng chung.
- **Vị trí mã nguồn:** `frontend/edtech-mobile/`.

---

## 2. Cấu trúc thư mục

```
frontend/edtech-mobile/
├── app/                        # Expo Router – entry & routes
│   ├── _layout.tsx             # Root layout, ThemeProvider, Stack
│   ├── index.tsx               # Entry: redirect auth / (app)
│   ├── (auth)/                 # Nhóm màn hình chưa đăng nhập
│   │   ├── _layout.tsx         # Stack auth (ẩn header)
│   │   └── login.tsx           # Đăng nhập
│   └── (app)/                  # Nhóm màn hình đã đăng nhập
│       ├── _layout.tsx         # Tabs (Trang chủ, Cá nhân, …)
│       ├── index.tsx           # Trang chủ
│       └── profile.tsx          # Cá nhân
├── src/
│   ├── theme/                  # Design system đồng bộ web
│   │   ├── colors.ts           # palette, darkPalette (brand #127278, …)
│   │   ├── spacing.ts         # radius, spacing, shadows
│   │   ├── typography.ts      # fontFamily (DM Sans), fontSize, fontWeight
│   │   ├── ThemeContext.tsx   # useTheme(), light/dark
│   │   └── index.ts
│   ├── components/
│   │   ├── icons/             # Re-export lucide-react-native (đồng bộ web)
│   │   │   └── index.tsx
│   │   └── ui/                # Button, Input, Card (dùng theme)
│   │       ├── Button.tsx
│   │       ├── Input.tsx
│   │       ├── Card.tsx
│   │       └── index.ts
│   ├── services/
│   │   ├── api.ts             # Axios instance, setAuthToken, getApiErrorCode/Message
│   │   └── auth.ts            # login(), clearAuthToken() – gọi API backend
│   ├── hooks/
│   │   └── useAuth.ts         # isLoggedIn, user, login, logout (mở rộng)
│   ├── constants/
│   │   └── index.ts           # AUTH_STORAGE_KEYS, …
│   └── css/                   # CSS đồng bộ edtech-lms (src/css/_core/...)
│       └── _core/
│           ├── globals.css    # Entry: @tailwind base/components/utilities + import
│           ├── app.css        # App-specific (card-auth, card-hover)
│           ├── layouts/
│           │   └── container.css
│           ├── override/
│           │   └── reboot.css
│           └── pages/
│               ├── auth.css
│               └── app.css
├── global.css                 # Entry NativeWind: @import './src/css/_core/globals.css'
├── tailwind.config.js         # Màu đồng bộ web; content gồm src/css/**/*.css
├── nativewind-env.d.ts        # TypeScript cho className
├── app.json
├── package.json
├── tsconfig.json              # paths: "@/*" -> "src/*"
├── babel.config.js            # alias @, nativewind/babel, reanimated
└── metro.config.js            # withNativeWind(..., { input: './global.css' })
```

---

## 3. Theme – đồng bộ với web

Nguồn tham chiếu: `frontend/edtech-lms/src/css/_core/globals.css`.

| Token / nhóm | Mô tả | Giá trị ví dụ (light) |
|--------------|--------|------------------------|
| **Brand / Primary** | Màu chính toàn app | `#127278` |
| **Secondary** | Nút phụ, accent | `#2db5bc` |
| **Background / Foreground** | Nền, chữ chính | `#ffffff`, `#1c2536` |
| **Muted** | Nền/phụ nhạt | `#f5f5f5`, `rgba(90,106,133,0.75)` |
| **Border / Input** | Viền, ô nhập | `#dfe5ef`, `#e5e7eb` |
| **Light variants** | Nền accent nhạt (vd. nền login) | `lightPrimary` ≈ 12% primary |
| **Status** | success, warning, error, info | `#13deb9`, `#f6b51e`, `#ef4444`, `#8754ec` |
| **Radius** | Bo góc | sm: 4, md: 7, base: 10, tw: 12 (px) |
| **Spacing** | padding, gap, margin | 15, 30, … (px) |
| **Typography** | Font chính | DM Sans (có thể load qua expo-font) |

- **Dark mode:** `darkPalette` trong `src/theme/colors.ts` tương ứng biến `.dark` trên web.
- Component dùng `useTheme()` từ `ThemeContext` để lấy `colors`, `isDark` và dùng `theme.radius`, `theme.spacing` khi cần.

---

## 4. Navigation (Expo Router)

- **Entry:** `app/index.tsx` – kiểm tra đăng nhập (sẽ dùng token/AsyncStorage), redirect:
  - Chưa đăng nhập → `/(auth)/login`
  - Đã đăng nhập → `/(app)`
- **Auth:** Nhóm `(auth)`: Stack, không header. Hiện có `login`; có thể thêm `register`, `forgot-password`.
- **App:** Nhóm `(app)`: Bottom Tabs. Tab active dùng `colors.primary`, inactive dùng `mutedForeground`; header và tab bar dùng `colors.background`, `colors.card`, `colors.border`.

Mở rộng: thêm file trong `app/(auth)/` hoặc `app/(app)/`, hoặc tạo nhóm mới (vd. `(app)/(courses)/`) với `_layout.tsx` riêng.

---

## 5. UI components

- **Button:** variants `primary` | `secondary` | `outline` | `ghost` | `destructive`; loading, disabled; dùng màu theme và `shadows.button`.
- **Input:** label, error, border/placeholder màu từ theme.
- **Card:** nền card, border, shadow; padding tùy chọn.

Tất cả đọc màu từ `useTheme()` để tự đồng bộ light/dark và brand.

---

## 6. API (Axios) & Services

- **api.ts:** Dùng **axios** (đồng bộ với edtech-lms). `getApiInstance()`, `apiRequest()`, `setAuthToken(token)`, `getApiErrorCode(err)`, `getApiErrorMessage(err)`.
- **auth.ts:** `login(payload)` gọi `apiRequest('/api/auth/login', { method: 'POST', data: payload })`, sau đó `setAuthToken(data.accessToken)`. Re-export `getApiErrorCode`, `getApiErrorMessage` để màn hình bắt lỗi giống web.

Mở rộng: lưu token/user vào AsyncStorage (keys trong `constants`), dùng trong `useAuth` và `app/index.tsx` để quyết định redirect; thêm `refreshToken`, intercept 401.

---

## 7. Styling – NativeWind (Tailwind)

- **Tailwind:** NativeWind 4 + `tailwindcss` 3.x. Cấu trúc CSS giống edtech-lms: entry `global.css` → `src/css/_core/globals.css` (@tailwind base/components/utilities + @import app.css, layouts/container.css, override/reboot.css, pages/auth.css, pages/app.css). Import `global.css` trong `app/_layout.tsx`; Metro: `withNativeWind(..., { input: './global.css' })`; Babel: preset `nativewind/babel`.
- **tailwind.config.js:** `theme.extend.colors` đồng bộ web (primary, secondary, background, muted, border, …). Component có thể dùng `className="flex-1 bg-primary"` (cần dùng component hỗ trợ NativeWind hoặc styled).
- **Theme JS:** `src/theme/` vẫn dùng cho StyleSheet và `useTheme()`; màu trong Tailwind config trùng với theme để hai cách dùng nhất quán.

---

## 8. Icons – đồng bộ với web (Lucide)

- **Web (edtech-lms):** `lucide-react`, `@tabler/icons-react`, `react-icons`.
- **Mobile:** Chỉ dùng **lucide-react-native** (cùng bộ Lucide, cùng tên icon). Re-export trong `src/components/icons/index.tsx` (ChevronDown, Check, Home, User, Mail, Lock, …) để import từ `@/components/icons` giống cách web dùng `lucide-react`.
- Tab bar và màn hình dùng icon từ `@/components/icons`; thêm icon mới khi cần thì bổ sung vào file re-export.

---

## 9. Quy ước mở rộng / bảo trì

- **Màu / font:** Chỉ sửa trong `src/theme/`. Cần đồng bộ web thì đối chiếu lại `edtech-lms/src/css/_core/globals.css`.
- **Màn mới:** Thêm file route trong `app/`; component màn đặt trong `app/...` hoặc `src/screens/` (tùy team).
- **API:** Endpoint mới gọi qua `apiRequest`; auth gắn header `Authorization: Bearer <token>` từ storage hoặc `useAuth`.
- **i18n:** Có thể dùng chung key với web, đặt file ngôn ngữ trong `src/locales` hoặc repo dùng chung.

---

## 10. Chạy dự án

```bash
cd frontend/edtech-mobile
npm install
npx expo start --port 19006 --clear
```

Sau đó mở bằng simulator/Expo Go. Cấu hình API (nếu cần): `app.config.js` hoặc env `EXPO_PUBLIC_API_URL` trỏ tới backend edtech.
