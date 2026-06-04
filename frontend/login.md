# 登入頁(前端)

前端登入視角:登入頁怎麼擴充、登入 / 登出能掛哪些 hook、閒置自動登出、第三方登入。**後端登入流程(登入模式、JWT、Token 撤銷、OrgKey、Proxy、IP 白名單)見 [core/login.md](../core/login.md)**;本篇只談前端可掛 / 可換的點。

---

## 登入頁附加區塊:`HCS_LOGIN_PAGE_COMPONENT`(multi)

登入頁底部(`login-foot`)是一個 `multi` 元件插槽——**往登入頁「加」東西**(第三方登入鈕、語系切換…),不是替換整頁。經 `DynamicComponentHostComponent` 並列渲染。

```typescript
{ provide: HCS_LOGIN_PAGE_COMPONENT, useValue: GoogleLoginButtonComponent, multi: true }
```

- 預設已有一顆語系切換鈕(`HcsPlatformProviderModule.forRoot()` 提供;同一顆也掛在 user-menu,見 [shell](shell.md))。
- third-party-login library 就是用這個 token 把 OAuth 登入鈕掛上去。
- 要**整頁替換**登入頁則走路由覆寫(component override),不是這個 token。

---

## 登入挑戰處理:`HCS_LOGIN_STATUS_CODE_HANDLER`

登入回應若帶特定 `MessageCode`,平台會交給對應的 handler 處理——**這是 2FA 等「登入中途要再問一步」的掛點**(`multi`,依 `code` 比對):

```typescript
export interface ILoginStatusCodeHandler {
  code: string;
  // 回新的 data → 平台用它「重登一次」;回 false → 中止登入
  handle(data: any, storage: UserStateStorage): Promise<any | false>;
}
```

流程:`baseLogin(data)` → 後端回應 `MessageCode` 命中某 handler → `handle(data, storage)`:

- 回**新 data**(如原 data + OTP)→ 平台拿它**遞迴重登**(`baseLogin(ndata)`)。
- 回 **`false`** → 中止,什麼都不做。

成功回應帶 code、或錯誤回應帶 code,**兩條路徑都會觸發 handler**。2FA 就是登入回「需要 2FA」碼 → handler 跳 2FA 對話框拿驗證碼 → 回帶碼的 data 重登(見 [modules/2fa](../modules/2fa.md))。

```typescript
{ provide: HCS_LOGIN_STATUS_CODE_HANDLER, useClass: TwoFactorChallengeHandler, multi: true }
```

---

## 登入 / 登出生命週期 hook:`HCS_USER_STATE_SERVICE`

`IUserStateService`(`multi`)在登入 / 登出前後各給一個 hook:

```typescript
export interface IUserStateService {
  beforeLoginAsync?(data): Promise<boolean>;     // 回 false → 中止登入
  afterLoginedAsync?(storage): Promise<void>;    // 登入成功後
  beforeLogoutAsync?(storage): Promise<boolean>; // 回 false → 中止登出
  afterLogoutedAsync?(): Promise<void>;          // 登出後
}
```

- `beforeLoginAsync` / `beforeLogoutAsync` 任一回 `false` → 該動作中止(全部回 truthy 才繼續)。
- 用途:登入前置檢查、登入後初始化、登出前確認 / 清理(third-party-login 就用它清第三方登入狀態)。

```typescript
{ provide: HCS_USER_STATE_SERVICE, useClass: MyLoginLifecycleService, multi: true }
```

---

## 閒置自動登出:`HCS_IDLE_TIMEOUT_MINUTES`

provide 一個分鐘數即開啟閒置自動登出;**沒 provide 或 `<= 0` = 關閉**(由 `SessionSyncService` 實作,`hcs-default-app` 啟動時 `init()`)。

```typescript
{ provide: HCS_IDLE_TIMEOUT_MINUTES, useValue: 30 }   // 閒置 30 分鐘自動登出
```

- **活動偵測**:`mousemove` / `mousedown` / `keydown` / `scroll` / `touchstart` 任一就重置計時。
- **跨分頁同步**:任一分頁有活動會重置**所有**分頁的計時;先到期的分頁廣播登出,其他分頁(含被節流的背景分頁)一起離開(`BroadcastChannel`)。
- **睡眠 / 喚醒**:`last-activity` 時間戳會持久化;分頁重新可見 / 取得焦點時比對——若已超過閒置上限就登出(涵蓋 `setTimeout` 在機器睡眠時被凍結的情況)。
- **iframe 模式不啟用**:`SessionSyncService.init()` 包在 `!isIframe()` 內,嵌在 iframe 裡的平台不跑閒置登出。
- ⚠️ **這不是安全控制**:純前端判斷(伺服器分不出真人活動與背景輪詢),只是替誠實使用者 / 共用終端登出本機;持有有效 token 的人仍能自行 refresh。**消費端不 provide 就永遠不會啟用**——這是典型「不知道它存在就用不到」的 opt-in。

---

## 第三方 / Google 登入:`HCS_GOOGLELOGIN_CLIENTID`

用 Google 登入時 provide Google OAuth client id(由 third-party-login library 消費);不用 Google 登入就不必給。

```typescript
{ provide: HCS_GOOGLELOGIN_CLIENTID, useValue: 'xxxx.apps.googleusercontent.com' }
```

OAuth 登入頁與綁定流程由 `third-party-login` library 提供(登入鈕透過 `HCS_LOGIN_PAGE_COMPONENT` 掛上)。

---

## Gotchas

- **`HCS_LOGIN_PAGE_COMPONENT` 是「加」不是「換」**:multi 附加插槽;要整頁換登入頁走路由覆寫。
- **status handler 成功 / 失敗都會觸發**:`MessageCode` 不分成功或錯誤回應,只要命中 `code` 就跑 `handle`;回 `false` 才是最終中止、回新 data 會遞迴重登(別不小心造成無限重登)。
- **before-hook 回 `false` 會擋掉登入 / 登出**:`HCS_USER_STATE_SERVICE` 的 `beforeLoginAsync` / `beforeLogoutAsync` 是閘門,別在裡面意外回 falsy。
- **閒置登出不 provide 就沒有**:`HCS_IDLE_TIMEOUT_MINUTES` 預設關;且它是體驗 / 合規措施,不是資安邊界。

---

## 關聯

- 後端登入流程 / JWT / Token 撤銷 / OrgKey / Proxy / IP 白名單 — [core/login.md](../core/login.md)
- 2FA 挑戰(掛 `HCS_LOGIN_STATUS_CODE_HANDLER`)— [modules/2fa](../modules/2fa.md)
- 登入頁語系鈕同源、user-menu 插槽 — [shell](shell.md)
- 多租戶與組織(登入後落在哪個組織)— [core/multi-tenant](../core/multi-tenant.md)
