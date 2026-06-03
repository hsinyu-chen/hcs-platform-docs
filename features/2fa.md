# 2FA / OTP(Two-Factor Authentication)

二階段驗證。兩種觸發點:**登入**時要求 OTP,以及**寫入操作**(表單存檔 / 列表刪除)時要求 OTP。內建 Google Authenticator(TOTP)provider,可自訂其他機制。

## 何時用

- 某個 entity 的新增 / 修改 / 刪除是敏感操作,要在存檔前要求使用者輸入動態驗證碼。
- 登入流程要在密碼之外加第二因子。
- 要接入 Google Authenticator 以外的 2FA 機制(SMS、Email OTP…)→ 需自訂 provider(進階)。

---

## 啟用(最短路徑)

2FA 是 **opt-in**,而且前後端**各自**要掛。一個完整啟用 = 下面 5 步:

### 後端

1. **Host 註冊功能**(一次性):
   ```csharp
   builder.AddTwoFactorAuthentication(options =>
   {
       options.AddGoogleAuthenticator();
   });
   services.UseGoogleAuthenticator2FaLogin();   // 只有「登入也要 2FA」時才加
   ```

2. **在要保護的 entity 掛驗證** — 該 entity 的 `ModuleConfig`,對寫入 API 掛 `OnValidate`:
   ```csharp
   options.ConfigPostApi(x => x.OnValidate(v => v.ValidGoogle2Fa()));   // 新增要 OTP
   options.ConfigPutApi(x  => x.OnValidate(v => v.ValidGoogle2Fa()));   // 修改要 OTP
   ```
   多方法並存用泛用版 `Valid2Fa(allowTypes)`;刪除流程用 `builder.Valid2Fa(...)`。

### 前端

3. **Host 註冊 provider module**(一次性):
   ```typescript
   HcsTwoFactorAuthenticationProviderModule.forRoot(),
   HcsTwoFactorAuthenticationGoogleProviderModule.forRoot(),
   ```

4. **在要保護的表單 component 掛攔截** — providers 加一行,不必改 submit 邏輯:
   ```typescript
   import { FormSubmit2faService } from '@hcs/two-factor-authentication';
   import { GoogleAuthenticatorMethod } from '@hcs/two-factor-authentication-google';

   @Component({
     providers: [ FormSubmit2faService.use(GoogleAuthenticatorMethod) ]
   })
   export class CustomerFormComponent extends BaseFormComponent<Customer> { }
   ```

5. **前提**:使用者本人要先綁定過 Google Authenticator(掃 QR code)。綁定入口在 user menu(由 Google provider module 自動掛上),沒綁定的人 dialog 開不出可驗證的密鑰。

## 使用範例

### 表單存檔要求 OTP

```typescript
@Component({
  providers: [
    // 預設在 'new' / 'copy' / 'edit' 三種狀態存檔時要 OTP
    FormSubmit2faService.use(GoogleAuthenticatorMethod)
    // 只想限制特定狀態:傳第二個參數
    // FormSubmit2faService.use(GoogleAuthenticatorMethod, ['edit'])
  ]
})
```

### 列表刪除要求 OTP

```typescript
import { ListDeletingfaService } from '@hcs/two-factor-authentication';

@Component({
  providers: [ ListDeletingfaService.use(GoogleAuthenticatorMethod) ]
})
export class CustomerComponent extends BaseListComponent<Customer> { }
```

兩者運作相同:攔截 submit / delete → 開 dialog 等使用者輸入 → 把 OTP 塞進 HTTP header → 放行送出;使用者取消則中止操作。

## 前後端對接 & 失配 caveat

OTP 透過兩個 HTTP header 傳到後端:

| Header | 內容 |
|---|---|
| `X-HCS-2fa-method` | 方法名稱,如 `"GoogleAuthenticator"` |
| `X-HCS-2fa-code` | 使用者輸入的驗證碼 |

完整流程:前端攔截 → 開 dialog → 拿到碼塞 header → 送出 → 後端 `Valid2Fa` 從 header 取碼 → 查該使用者該 method 的密鑰 → 用對應 provider 驗證 TOTP。

> [!IMPORTANT]
> **前後端是兩個獨立的 opt-in,要手動保持對齊,而且失配方向不對稱:**
> - **後端掛了、前端沒掛** → 使用者存檔被後端打回(沒帶 header),但畫面上沒有輸入 OTP 的入口,只看到莫名失敗。
> - **前端掛了、後端沒掛** → 使用者輸了 OTP,但後端根本沒驗就放行(security theater,假的安全感)。
>
> **真正的 enforcement 只在後端那一行 `ValidGoogle2Fa()`**。前端的 `use()` 是 UX(提供輸入入口),不是安全邊界。漏配不會編譯失敗,只會 runtime 才現形 —— 加 2FA 時務必兩端一起改。

## Invariants

- 啟用前提:使用者要先在 user menu 綁定該 2FA 方法,否則 dialog 無密鑰可驗。
- 後端對同一使用者驗證 OTP 時序列化(per-user lock)且故意延遲 500~1000ms,防暴力嘗試 —— 高頻連續驗證會被排隊拖慢,屬正常。

## Anti-patterns

- ❌ **把前端 `use()` 當安全控制** —— 它只是 UX 入口,enforcement 在後端 `ValidXxx2Fa()`。少了後端那行,前端攔截形同虛設。
- ❌ **在 access log / APM 記錄 `X-HCS-2fa-code` header** —— OTP 會進日誌。確認 log 設定排除它。
- ❌ **自刻綁定流程直接打 `POST /api/entity/my2fa`** —— 用內建的 user menu 綁定元件,它已處理 QR / secret 產生與驗證。
