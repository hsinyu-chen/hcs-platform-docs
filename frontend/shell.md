# 外殼 / 版面(Index / Toolbar / Menu)

> _待寫:一句話定位——平台預設外殼長怎樣(index 框架包工具列 + 選單 + 內容區),以及怎麼整塊或逐塊替換成自家品牌的外殼。_

---

## 版面結構

> _待寫:index component → toolbar + 選單 + `router-outlet` 的組成關係;預設外殼提供什麼。_

## 替換外殼元件

> _待寫:本節總覽三個外殼插槽,逐一給去敏 demo component + provider。_

### 首頁框架:`HCS_INDEX_COMPONENT`

> _待寫:整個外殼框架替換點;去敏 demo。_

### 工具列:`HCS_TOOLBAR_COMPONENT`

> _待寫:頂部工具列替換;去敏 demo。_

### 使用者選單:`HCS_USER_MENU_COMPONENT`

> _待寫:右上角使用者選單替換;去敏 demo。_

## 選單項:`MENU_ITEMS`

> _待寫:各 library `Menu extends MenuItemProvider` 掛 multi、`hasPermission` 自動隱藏;與 frontend.md 慣例 4 互連(避免重複,深度放這、概念放 frontend.md)。_

## Header 按鈕開關:`HCS_DEFAULT_APP_CONFIG`

> _待寫:header 四個按鈕的開關設定。_

## Gotchas

> _待寫:插槽都是單一 provide(非 multi)、替換後權限/路由仍保留之類的雷。_

## 關聯

> _待寫:連 frontend.md、login.md(前端)、core/i18n-system.md(選單 i18n)、core/permissions.md。_
