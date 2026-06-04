# 外殼 / 版面(Index / Toolbar / Menu)

平台的外殼是 `hcs-default-app`:頂部工具列 + 左側選單抽屜 + 中間內容區(router-outlet)。本篇講怎麼**替換或擴充**這層外殼——換首頁、加工具列按鈕、改使用者選單、控制 header 內建鈕——而不必 fork SDK。

---

## 版面結構

```
┌─────────────────────────────────────────────┐
│ Toolbar  [home][scan][fullscreen][notify] 👤 │  ← header 內建鈕 + HCS_TOOLBAR_COMPONENT + 使用者選單
├──────────┬──────────────────────────────────┤
│  選單     │                                  │
│ (MENU_   │        內容區(router-outlet)     │  ← 首頁是 HCS_INDEX_COMPONENT
│  ITEMS)  │                                  │
└──────────┴──────────────────────────────────┘
```

選單抽屜的開合、深色主題、視窗模式等是**跨頁偏好**,存在 `PageStatusHolder` 的 `site` scope(localStorage),不綁路由。

---

## 替換外殼元件

三個外殼插槽都是 `ComponentType<any>` 的 injection token,**而且三個都是 `multi`——你的元件是「加進去」、不是「換掉」**,經 `DynamicComponentHostComponent` 和內建項並列渲染。

### 首頁區塊:`HCS_INDEX_COMPONENT`(multi)

登入後落地的首頁由一或多個 index 元件**並列**組成(`hcs-index` 渲染)。provide 你的元件**加一塊**首頁區塊(如儀表板卡片);多個模組各自加自己的。

```typescript
{ provide: HCS_INDEX_COMPONENT, useValue: DashboardComponent, multi: true }
```

### 工具列:`HCS_TOOLBAR_COMPONENT`(multi)

往頂部工具列**加**自訂按鈕 / 狀態。multi——你的元件出現在內建項旁邊。

```typescript
{ provide: HCS_TOOLBAR_COMPONENT, useValue: MyToolbarWidgetComponent, multi: true }
```

(平台自己就是這樣掛東西——Basic 模組把一顆 localhost 限定的開發捷徑鈕掛在這個 token 上。)

### 使用者選單:`HCS_USER_MENU_COMPONENT`(multi)

往右上角使用者選單**加**項目。multi——預設已有一顆語系切換鈕(`HcsPlatformProviderModule.forRoot()` 提供),你 provide 的會並列。

```typescript
{ provide: HCS_USER_MENU_COMPONENT, useValue: MyAccountMenuComponent, multi: true }
```

> 那顆內建語系鈕(`LangButtonComponent`)**同時也掛在登入頁**(`HCS_LOGIN_PAGE_COMPONENT`,見 [login](login.md));要拿掉內建語系鈕,兩處都得處理。

---

## 選單項:`MENU_ITEMS`

左側選單由各 library 註冊的 `MenuItemProvider`(`multi`)合併而成,概念見 [frontend](../frontend.md) 慣例 4。每個 menu item 有一個 `visible: () => boolean` 回呼(預設 `() => true`),template 用它過濾;**慣例是把 `permission.hasPermission(...)` 當這個 callback 傳進去——權限不通過自動不顯示**。(父項在有子項時,`visible` 會被覆寫成「任一子項 visible 才顯示」。)

兩個外殼相關行為:

- 點選單項會呼叫 `PageStatusHolder.clearNext()`——**從選單導航到某頁時,清掉該頁上次記住的查詢 / 排序 / 分頁狀態**(等同帶 `?clear=1`)。這就是「從選單點進列表是乾淨的、用瀏覽器上一頁回來才保留狀態」的原因(列表狀態記憶見 [list](list.md))。
- 選單展開/收合狀態會被記住(`Menu.saveState()`)。

### 選單徽章(badge)

每個 `MenuItem` 可帶一個 `getCount: () => number | undefined` 回呼產生紅點數字(顏色由 `badgeColor` 控制,預設 `warn`)。兩個自動行為:

- **父項自動加總子項**的 badge;**工具列顯示所有選單的總數**(例如待辦 / 未讀)。
- badge 是**動態**的:`badgeCount` 是 getter、每次變更偵測重算 `getCount()`,不是建立時的快照。`MenuItemProvider.get(onDestroy)` 收到一個 `onDestroy` 註冊器,可在裡面訂閱即時數量來源、並登記清理(離開時取消訂閱)。

```typescript
export class TaskMenu extends MenuItemProvider {
  get(onDestroy: (fn: () => void) => void): MenuItem[] {
    const sub = this.taskService.pendingCount$.subscribe(/* 更新 count */);
    onDestroy(() => sub.unsubscribe());
    return [ new MenuItem('待辦', 'task', ['/', 'tasks'], null, null,
                          undefined, undefined, () => this.count) ];  // ← getCount
  }
}
```

---

## Header 內建鈕開關:`HCS_DEFAULT_APP_CONFIG`

控制工具列上四顆內建鈕的顯示:

```typescript
{
  provide: HCS_DEFAULT_APP_CONFIG,
  useValue: { header: { home: true, scan: false, fullScreen: false, notification: true } },
}
```

| 鍵 | 預設 | 鈕 |
|---|---|---|
| `home` | `true` | 回首頁 |
| `scan` | `false` | 條碼掃描 |
| `fullScreen` | `false` | 全螢幕 / kiosk |
| `notification` | `true` | 通知 |

預設值由 `HcsPlatformProviderModule.forRoot()` 提供;要改就在 host 自己 provide 一份覆寫。

---

## 外殼層全域快捷鍵

`hcs-default-app` 掛了三顆跨頁快捷鍵(只在非 iframe 時觸發,所有 host 都繼承):

- **`Alt+F7`** → PWA 強制更新(`updateApp`)
- **`Alt+F8`** → 推一筆測試通知
- **`Alt+F9`** → 清掉 service worker caches 後 reload(清前端快取救援)

---

## Gotchas

- **三個插槽都是「加」不是「換」**:`HCS_INDEX_COMPONENT` / `HCS_TOOLBAR_COMPONENT` / `HCS_USER_MENU_COMPONENT` **全是 `multi`**,你的元件**並列**在內建項(如語系鈕)旁邊;範例**漏 `multi: true` 會壞掉**(`injector.get` 取單值、`*ngFor` 不展開)。想完全取代內建項得連內建 provider 一起處理,不能光靠 provide 自己的。
- **選單導航會清狀態**:從 `MENU_ITEMS` 點進一頁會 `clearNext()` 洗掉該頁的列表狀態記憶——這是刻意的(選單=重新開始);用上一頁 / 連結回去才保留。
- **外殼偏好存 `site` scope**:深色主題 / 視窗模式 / **抽屜開合(`menu-open`)** 存 `site-state-container`(localStorage)、跨路由共用,不隨頁面狀態被 `?clear=1` 清掉。**注意選單「各項展開/收合」是另一個 entry(`menu-state`,由 `Menu.saveState()` 寫)**,跟抽屜開合不同 key。

---

## 關聯

- 列表狀態記憶(被選單 `clearNext` 清的那個)— [list](list.md)
- 選單 i18n 合併(`I18N_INDEX`)— [core/i18n-system](../core/i18n-system.md)
- menu item 的 `hasPermission` 權限對齊 — [core/permissions](../core/permissions.md)
- 前端登入頁替換(`HCS_LOGIN_PAGE_COMPONENT`)— [login](login.md)
- `forRoot()` 預設 provide 哪些 token、SDK 慣例 — [frontend](../frontend.md)
