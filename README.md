# Phân tích kỹ thuật: Luồng QR Check-in & Cơ chế Auth trong app PTIT S-Link

*Tài liệu tổng hợp từ việc phân tích file bundle `index_android.js` (React Native, Hermes bytecode dạng dịch ngược).*

---

## 1. Định dạng mã QR điểm danh sự kiện

```
PTIT|SU_KIEN|CHECK_IN|{"maDiemDanh":"153591","idSuKien":"6aa161869d2ed9b3aa03bf5d"}
```

Cấu trúc chung do `parseModularQR()` xử lý: `PREFIX|module|action|data`

| Phần | Giá trị ví dụ | Ý nghĩa |
|---|---|---|
| module | `SU_KIEN` | Nhóm chức năng: Sự kiện |
| action | `CHECK_IN` | Hành động: điểm danh check-in |
| data | JSON string | Chứa `maDiemDanh` (mã điểm danh) và `idSuKien` (id sự kiện) |

---

## 2. Luồng xử lý phía client (app)

```
Quét QR
  → parseModularQR(qrString)
  → handleModularQR → handleEventModule(action, data)
  → module === 'SU_KIEN' && action === 'CHECK_IN'
      → handleEventCheckIn(data)
          → JSON.parse(data) => { maDiemDanh, idSuKien }
          → getAttendEventUseCase().execute({
                ma: maDiemDanh,
                loaiQR: 'Tham gia'
            })
          → AttendEventUseCase.execute → repository.attendEvent(payload)
          → apiService.api().post('/slink/sv-su-kien/qr', payload)
```

**Lưu ý:** `idSuKien` được parse ra từ JSON nhưng **không** thấy được đưa vào payload gửi lên trong đoạn code check-in này — chỉ có `ma` (= `maDiemDanh`) và `loaiQR` được POST.

---

## 3. Endpoint API điểm danh

```
POST https://gwdu.ptit.edu.vn/slink/sv-su-kien/qr
Headers:
  Authorization: Bearer <accessToken>
Body:
{
  "ma": "153591",
  "loaiQR": "Tham gia"
}
```

- Domain gốc `gwdu.ptit.edu.vn` được xác nhận qua việc user truy cập trực tiếp endpoint và nhận về `401 Unauthorized` — đúng như dự đoán, vì thiếu header `Authorization`.
- Domain này (`API_URL`) không hard-code trong bundle mà được đọc từ config môi trường lúc build app.
- Sau khi request thành công, app xử lý theo 2 nhánh:
  - Nếu response có `surveyId` → điều hướng sang `KhaoSatFormScreen` (khảo sát sau sự kiện).
  - Nếu không → hiện toast "Check-in sự kiện thành công" rồi quay lại/điều hướng `SuKienThamGiaScreen`.
  - Lỗi → hiện toast lỗi tương ứng.

---

## 4. Kiến trúc Authentication (SSO / OAuth2 – Keycloak)

### 4.1 Cấu hình SSO
Thư viện dùng: **`react-native-app-auth`** (native module `RNAppAuth`), chuẩn **OAuth2 / OpenID Connect** với **PKCE**, kết nối tới **Keycloak** (dựa trên path đặc trưng `/protocol/openid-connect/...`).

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
(`SSO_URL`, `SUB_NAME`, `REDIREC_URL` là biến môi trường, không có giá trị literal trong bundle.)

### 4.2 Đăng nhập
`RNAppAuth.authorize(CONFIG_SSO)` → mở màn hình đăng nhập Keycloak (WebView/Custom Tabs) → Authorization Code Flow + PKCE → trả về:
```
{ accessToken, refreshToken, idToken, tokenType, accessTokenExpirationDate }
```

### 4.3 Lưu trữ token
Một store (singleton, dạng class) giữ các field trong bộ nhớ: `accessToken`, `refreshToken`, `responseSSO`, `userMe`, `userInfo`, `permission`, `allPermissions`, `codePhanVung`...

Mỗi lần `setAccessToken()` / `setRefreshToken()`:
- Cập nhật biến in-memory.
- Đồng thời ghi xuống **AsyncStorage** với các key: `KEY_ACCESS_TOKEN`, `KEY_REFRESH_TOKEN`, `KEY_RESPONSE_SSO`, `KEY_USER_ME`, `KEY_USER_INFO`, `KEY_PERMISSION`, `KEY_ALL_PERMISSIONS`, `PHANVUNG`.

Khi mở app, `initData()` đọc lại các key này để khôi phục phiên đăng nhập (auto-login nếu còn token hợp lệ).

### 4.4 Gắn token vào request
Mọi request qua `apiService.api()` tự động thêm header:
```
Authorization: Bearer <accessToken>
```

### 4.5 Refresh token khi hết hạn (interceptor)
Khi API trả lỗi dạng `TokenInvalid`:
1. Có cờ `isRefreshing` chống gọi refresh trùng lặp song song; các request khác được đưa vào `processQueue` chờ.
2. Gọi `RNAppAuth.refresh(CONFIG_SSO, { refreshToken })` → POST tới `tokenEndpoint` (`grant_type=refresh_token`).
3. Thành công → `setAccessToken()`, `setRefreshToken()`, `setResponseSSO()` lưu token mới → retry lại request gốc với header mới (`buildRetryConfig`).
4. Thất bại (refresh token cũng hết hạn) → `logout()` (xoá token khỏi store + AsyncStorage) → điều hướng về `LoginScreen`.

### 4.6 Tính chất của token
- **Access token**: JWT ngắn hạn (thường vài phút–vài chục phút tùy cấu hình Keycloak realm), **sinh mới hoàn toàn mỗi lần login/refresh** — không cố định theo tài khoản.
- **Refresh token**: sống lâu hơn nhưng cũng có thể bị **rotate** (thay mới) sau mỗi lần dùng để refresh — thấy rõ qua logic:
  ```js
  refreshToken = response.refreshToken ?? oldRefreshToken
  ```
  Đây là cơ chế **refresh token rotation** chuẩn để tăng bảo mật: nếu bị lộ, token cũ nhanh chóng vô hiệu.
- Hết hạn refresh token → bắt buộc đăng nhập lại.

---

## 5. Sơ đồ tổng thể

```
┌─────────────┐     authorize (PKCE)     ┌──────────────┐
│ LoginScreen │ ───────────────────────► │   Keycloak   │
└─────────────┘ ◄─────────────────────── │  (SSO_URL)   │
      │          accessToken/refreshToken └──────────────┘
      ▼
┌───────────────────────────────┐
│ AuthStore (in-memory)         │
│  + persist → AsyncStorage     │
│  (KEY_ACCESS_TOKEN, ...)      │
└───────────────────────────────┘
      │  Authorization: Bearer <accessToken>
      ▼
┌───────────────────────────────┐     401 TokenInvalid     ┌──────────────┐
│  apiService.api()             │ ────────────────────────►│  refresh()   │
│  (mọi API call của app)       │ ◄──────────────────────── │  → Keycloak  │
└───────────────────────────────┘   token mới → retry       └──────────────┘
      │
      ▼  (ví dụ: check-in QR)
POST /slink/sv-su-kien/qr  { ma, loaiQR }
```

---

## 6. Ghi chú phạm vi & giới hạn của tài liệu

- Tài liệu này chỉ mô tả **kiến trúc/luồng xử lý phía client**, phục vụ mục đích tìm hiểu cách hệ thống hoạt động.
- **Không bao gồm**: cách server sinh `maDiemDanh`, chu kỳ/thuật toán sinh mã QR (logic này nằm hoàn toàn ở server, không có trong client bundle), cũng như bất kỳ hướng dẫn nào để tự gửi request điểm danh giả mạo (không thực sự tham dự sự kiện) hoặc dò/đoán mã điểm danh của người khác — các nội dung này không được cung cấp vì có thể dùng để gian lận hệ thống điểm danh của trường.
