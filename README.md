# PTIT S-Link — Phân tích luồng QR Check-in & Authentication

> Kiến trúc client-side của app **PTIT S-Link** (React Native), tập trung vào luồng quét mã QR điểm danh sự kiện và cơ chế xác thực (SSO/OAuth2).
>
> Được tổng hợp từ việc phân tích `index_android.js` (reverse engineering). Chỉ sử dụng cho mục đích học tập và nghiên cứu — không chứa logic gen mã điểm danh hay bất kỳ nội dung nào hỗ trợ gian lận điểm danh.

---

## Mục lục

- [1. Định dạng mã QR](#1-định-dạng-mã-qr)
- [2. Luồng xử lý check-in (client)](#2-luồng-xử-lý-check-in-client)
- [3. API endpoint](#3-api-endpoint)
- [4. Kiến trúc Authentication](#4-kiến-trúc-authentication)
- [5. Sơ đồ tổng thể](#5-sơ-đồ-tổng-thể)
- [Giới hạn phạm vi](#giới-hạn-phạm-vi)

---

## 1. Định dạng mã QR
<img width="720" height="1280" alt="image" src="https://github.com/user-attachments/assets/bd102f03-8544-422b-a9eb-bfceeb8e86f2" />


Mã QR điểm danh sự kiện có cấu trúc `PREFIX|module|action|data`:

```
PTIT|SU_KIEN|CHECK_IN|{"maDiemDanh":"153591","idSuKien":"6aa161869d2ed9b3aa03bf5d"}
```

| Phần | VD | Ý nghĩa |
|---|---|---|
| `module` | `SU_KIEN` | Nhóm chức năng: Sự kiện |
| `action` | `CHECK_IN` | Hành động điểm danh |
| `data` | JSON string | `maDiemDanh`, `idSuKien` |

Được parse bởi hàm `parseModularQR()` trong app.

## 2. Luồng xử lý check-in (client)

```text
Quét QR
 └─ parseModularQR(qrString)
     └─ handleModularQR → handleEventModule(action, data)
         └─ module === 'SU_KIEN' && action === 'CHECK_IN'
             └─ handleEventCheckIn(data)
                 ├─ JSON.parse(data) → { maDiemDanh, idSuKien }
                 ├─ getAttendEventUseCase().execute({ ma: maDiemDanh, loaiQR: 'Tham gia' })
                 ├─ AttendEventUseCase.execute → repository.attendEvent(payload)
                 └─ apiService.api().post('/slink/sv-su-kien/qr', payload)
```

> **Note:** `idSuKien` được parse từ QR nhưng không xuất hiện trong payload gửi lên ở đoạn xử lý này — chỉ `ma` (= `maDiemDanh`) và `loaiQR` được POST.

## 3. API endpoint

```http
POST https://gwdu.ptit.edu.vn/slink/sv-su-kien/qr
Authorization: Bearer <accessToken>
Content-Type: application/json

{
  "ma": "153591",
  "loaiQR": "Tham gia"
}
```

- Mình xác định được domain qua thực nghiệm: truy cập https://gwdu.ptit.edu.vn/slink/sv-su-kien/qr trực tiếp không kèm token → `401 Unauthorized`.
<img width="443" height="210" alt="image" src="https://github.com/user-attachments/assets/71a4e7c4-6d88-4bc3-9af7-2c27215e5842" />


- `API_URL` không hard-code trong bundle, được inject qua biến môi trường lúc build.
- Response xử lý theo 2 nhánh:
  - Có `surveyId` → điều hướng `KhaoSatFormScreen` (khảo sát sau sự kiện).
  - Không có → toast "Check-in thành công" → điều hướng `SuKienThamGiaScreen`.

## 4. Kiến trúc Authentication

Thư viện: [`react-native-app-auth`](https://github.com/FormidableLabs/react-native-app-auth) (native module `RNAppAuth`) — chuẩn **OAuth2 / OpenID Connect + PKCE**, kết nối **Keycloak** (nhận diện qua path `/protocol/openid-connect/...`).

### 4.1 Cấu hình SSO

```js
CONFIG_SSO = {
  issuer:      `${SSO_URL}`,
  clientId:    `${SUB_NAME}-connect`,
  redirectUrl: `${REDIREC_URL}`,
  serviceConfiguration: {
    authorizationEndpoint: `${SSO_URL}/protocol/openid-connect/auth`,
    tokenEndpoint:         `${SSO_URL}/protocol/openid-connect/token`,
    revocationEndpoint:    `${SSO_URL}/protocol/openid-connect/revoke`,
    endSessionEndpoint:    `${SSO_URL}/protocol/openid-connect/logout`,
  },
  scopes: ['openid', 'profile'],
}
```

### 4.2 Đăng nhập
`RNAppAuth.authorize(CONFIG_SSO)` → Keycloak login (WebView/Custom Tabs) → Authorization Code + PKCE → trả về `{ accessToken, refreshToken, idToken, tokenType, accessTokenExpirationDate }`.

### 4.3 Lưu trữ token
Singleton store giữ token + user info trong bộ nhớ, đồng thời persist xuống **AsyncStorage**:

| Storage key | Nội dung |
|---|---|
| `KEY_ACCESS_TOKEN` | Access token |
| `KEY_REFRESH_TOKEN` | Refresh token |
| `KEY_RESPONSE_SSO` | Toàn bộ response SSO |
| `KEY_USER_ME` / `KEY_USER_INFO` | Thông tin người dùng |
| `KEY_PERMISSION` / `KEY_ALL_PERMISSIONS` | Quyền hạn |

`initData()` đọc lại các key này khi mở app để auto-login nếu còn phiên hợp lệ.

### 4.4 Gắn token vào request
Mọi request qua `apiService.api()` tự động thêm:
```
Authorization: Bearer <accessToken>
```

### 4.5 Refresh token (interceptor)

```text
API trả 401 / TokenInvalid
 ├─ isRefreshing = true (tránh refresh trùng, request khác vào processQueue)
 ├─ RNAppAuth.refresh(CONFIG_SSO, { refreshToken })
 │    └─ POST tokenEndpoint (grant_type=refresh_token)
 ├─ Thành công → setAccessToken/setRefreshToken/setResponseSSO → retry request gốc
 └─ Thất bại   → logout() → xoá token → điều hướng LoginScreen
```

### 4.6 Tính chất của token

- **Access token**: JWT ngắn hạn, **sinh mới mỗi lần login/refresh** — không cố định theo tài khoản.
- **Refresh token**: thay mới sau mỗi lần refresh:
  ```js
  refreshToken = response.refreshToken ?? oldRefreshToken
  ```
- Refresh token hết hạn → bắt buộc đăng nhập lại.

## 5. Sơ đồ tổng thể

```text
┌─────────────┐   authorize (PKCE)    ┌──────────────┐
│ LoginScreen │ ────────────────────► │   Keycloak   │
└─────────────┘ ◄──────────────────── │  (SSO_URL)   │
      │         accessToken/refreshToken└──────────────┘
      ▼
┌────────────────────────────┐
│ AuthStore (in-memory)      │
│  + persist → AsyncStorage  │
└────────────────────────────┘
      │  Authorization: Bearer <accessToken>
      ▼
┌────────────────────────────┐   401 TokenInvalid   ┌──────────────┐
│ apiService.api()           │ ────────────────────►│  refresh()   │
│ (mọi API call của app)     │ ◄──────────────────── │  → Keycloak  │
└────────────────────────────┘   token mới → retry   └──────────────┘
      │
      ▼ (ví dụ: check-in QR)
POST /slink/sv-su-kien/qr  { ma, loaiQR }
```

## Giới hạn phạm vi

Tài liệu không bao gồm:
- Cách server sinh `maDiemDanh` hay chu kỳ/thuật toán sinh mã QR.
- Bất kỳ hướng dẫn nào để gửi request điểm danh giả mạo hoặc dò/đoán mã điểm danh của người khác.

---

*Repo này chỉ phục vụ mục đích tìm hiểu kiến trúc kỹ thuật (reverse engineering cho mục đích học tập). Việc sử dụng thông tin trong tài liệu để can thiệp trái phép vào hệ thống điểm danh hoặc dữ liệu của PTIT là trách nhiệm của người sử dụng và không được khuyến khích.*
