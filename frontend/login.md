# 登入頁(前端)

> _待寫:一句話定位——前端登入視角(替換登入頁、閒置登出、第三方登入掛載);**後端登入流程 / Token / 撤銷見 [core/login.md](../core/login.md)**,本篇只談前端可換可掛的點。_

---

## 預設登入頁

> _待寫:SDK 預設登入頁提供什麼、`HcsPlatformProviderModule.forRoot()` 怎麼掛上。_

## 替換登入頁:`HCS_LOGIN_PAGE_COMPONENT`

> _待寫:整頁替換的去敏 demo;牽涉路由的注意事項。_

## 登入狀態挑戰處理:`HCS_LOGIN_STATUS_CODE_HANDLER`

> _待寫:登入回特定狀態碼時的攔截(2FA 挑戰掛這);去敏 demo。連 [features/2fa](../features/2fa.md)。_

## 閒置自動登出:`HCS_IDLE_TIMEOUT_MINUTES`

> _待寫:number,0=關閉;消費端不知道就永遠用不到的典型 opt-in。_

## 第三方登入:`HCS_GOOGLELOGIN_CLIENTID`

> _待寫:Google 登入才需給;與 third-party-login library 的關係。_

## Gotchas

> _待寫:閒置登出與喚醒/背景分頁的行為;login handler 與後端狀態碼對齊。_

## 關聯

> _待寫:連 core/login.md(後端)、shell.md、features/2fa.md、frontend.md。_
