# CodeTable 模組（代碼表 / 字典）

`Hcs.PlatformModule.CodeTable`（前端 `@hcs/code-table` + 兩個核心 pipe）把「狀態、優先級、類別…」這類**由使用者維護的選項集**集中管理:後台有 CRUD 頁可增刪改,前端任何表單 / 清單用兩個 pipe 就能取用,且支援**每語系標籤**與**組織別 / 全域共用**兩種範圍。

它的價值不在管理頁,而在**取用機制**——下游模組不必自己做選項 API,一個 `codeTableOptions` pipe 就把某類代碼變成下拉選項、`codeTableViewer` 把存的數字 id 變回顯示文字。

---

## 啟用（後端）

```csharp
builder.AddCodeTableModule();
```

註冊 `CodeTable` entity API（GET 自動展開語系子表、POST/PUT 連同子表一起存）、`defaultOrgCodeTable` 查詢(只回預設組織的代碼表),並把**列表查詢(Query)開放給所有登入者**(代碼表到處被當選項用;單筆 GET 仍需 `.View`,見 Gotchas)。

> `AddCodeTableModule()` 只裝「代碼表這個 entity 與其查詢」。它**不會**幫你定義任何一種代碼表類型、也不掛任何選單——那是下游模組的事(見下)。

---

## 註冊一種代碼表 + 它的權限

代碼表以 `Type` 字串分類(`"Status"`、`"Priority"`…)。每種類型由**消費端模組**註冊一個功能 + 權限:

```csharp
moduleBuilder.AddCodeTableFuncion("Status");
// 產生功能 CodeTable.Status，含標準 CRUD 角色 + Admin 權限
```

這會生出 `CodeTable.Status.Create` / `.Modify` / `.Delete` / `.View` / `.Admin` **五個**權限(`.View` 是標準 CRUD 的讀取權,對應**單筆 GET + 列表 Query**)。⚠️ 但**列表 Query 另外被開放給所有登入者**(見下),所以實務上 `.View` 主要 gate 的是**單筆 GET**(`/api/entity/.../{id}`)——只用兩個取用 pipe(走 Query)不會踩到,做後台單筆檢視才需要派 `.View`。

### 寫入 / 刪除規則（驗證）

新增 / 修改 / 刪除一筆代碼表時:

- 必須有對應的 `CodeTable.{Type}.{Create|Modify|Delete}` 角色,否則回 `CodeTable.NoPermission`。
- `Type`、`Text` 為必填。
- **刪除帶 `Code` 值的列需 `Admin`**:沒有 `Code` 的純顯示列,有 `Delete` 權即可刪;一旦該列被指定了 `Code`(代表它被程式邏輯依賴),刪除需 `CodeTable.{Type}.Admin`,否則回 `CodeTable.HasCode`。
- 刪除還會跑**跨表參照檢查**(見下),被別的資料引用就擋下。

---

## 跨表參照保護

別的模組若用 `long` 欄位參照代碼表 id,可登記一條刪除前檢查,讓「正在被使用的代碼表」刪不掉:

```csharp
// 在依賴代碼表的模組裡登記：Order.StatusCodeId 參照了代碼表
moduleBuilder.AddCodeTableDeleteValidateRef<Order>(x => x.StatusCodeId);
// 可一次傳多個參照欄位
```

登記後,刪除任一代碼表時會數有多少筆 `Order` 的 `StatusCodeId` 指向它;`> 0` 就回 `allreference` 錯誤(含被引用筆數),刪除中止。

---

## 多語系（i18n）

每筆代碼表可掛多筆 `CodeTablei18n` 子記錄(`Lang` + 該語系的 `Text`)。取用時:

- 依**當前語系**挑對應的 `CodeTablei18n.Text`;
- 找不到對應語系 → 退回代碼表本身的預設 `Text`。

後端不做自動翻譯、不依使用者語系過濾——它只負責原樣存 `(Lang, Text)`,語系挑選與後援在前端 `CodeTableService` 做。

---

## 前端取用：兩個 pipe（核心）

⚠️ 這兩個 pipe 住在 **`@hcs/core/hcs-components`**(平台核心,任何元件都能用),**不在** `@hcs/code-table`——`@hcs/code-table` 只是管理頁。取用代碼表**不需要**引入 `@hcs/code-table`。

### `codeTableOptions`——把一類代碼變成選項陣列

```html
<hcs-select-input [formControl]="ctrl">
  <hcs-option *ngFor="let o of 'Status' | codeTableOptions" [value]="o.value">
    {{ o.text }}
  </hcs-option>
</hcs-select-input>
```

回傳 `CodeTableView[]`,每項有 `text`(已套語系)/ `value`(id)/ `sort` / `code`(字串陣列)/ `isEnabled`,已依 `sort` 排序。

### `codeTableViewer`——把存的 id 變回顯示文字

```html
<!-- 清單 / 唯讀畫面把數字 id 顯示成人看得懂的標籤 -->
{{ row.statusId | codeTableViewer:'Status' }}
```

第一參數是值、第二是代碼表 `Type`,回該值對應的 `text`(套當前語系)。

### 取用設定 `ICodeTablePipeData`

兩個 pipe 都收一個可選設定物件當第二 / 第三參數:

| 屬性 | 用途 |
|---|---|
| `isDefaultOrg` | `true` = 取**預設組織**的共用代碼表,而非當前使用者組織的(全域共用 vs 組織專屬) |
| `isEnadled` | `true` = 只回 `IsEnabled` 的列(⚠️ 屬性名是 `isEnadled`,code 裡的拼字,別寫成 `isEnabled`) |
| `codeSplit` | 分隔字元——把 `Code` 欄依此切成 `code` 字串陣列 |
| `checkMinute` | 快取保鮮分鐘數,**預設 10 分鐘**;設 `0` = 每次即時重抓 |

背後是 `CodeTableService`(`providedIn: 'root'`,用 IndexedDB 做離線優先快取)。兩個 pipe 都是 `pure: false`,但內部以 `type|語系|組織` 為 key 快取,不會每次變更偵測都打 API。

---

## 前端：管理 UI

```typescript
imports: [ HcsCodeTableProviderModule.forRoot() ]   // 只提供 I18N_INDEX 'code-table'
```

路由用 `HcsCodeTableProviderModule.defaultRoute`(落在 `code-table/` 下)掛進 app。頁面與路由都**以 `:type` 為鍵**:

- `code-table/:type`——某類型的代碼表清單(`hcs-code-table` / `CodeTableComponent`)
- `code-table/:type/new`、`/new/:copy`、`/:id`(唯讀)、`/:id/edit`——表單(`CodeTableFormComponent`,處理多語系子列)

權限由 `CodeTablePermission` 依路由的 `:type` 自動對應到 `CodeTable.{type}.*`。

> library **不提供任何選單項目**——要讓管理頁進得去,consumer app 得自己加一條選單(指向 `code-table/{你的type}`)並授對應權限。

---

## Gotchas

- **列表查詢對所有登入者開放,但單筆 GET 不是**:`AddCodeTableModule` 只把 `Query`(列表)掛 `Everyone`,**沒掛單筆 `Get`**。所以 `codeTableOptions`/`codeTableViewer`(走 Query)誰都能用,但打單筆 `/api/entity/.../{id}` 仍需 `CodeTable.{Type}.View`。寫入 / 刪除則一律 per-type 權限。
- **模組本身不含任何代碼表類型**:只裝 `AddCodeTableModule()` 不會有任何可管理的 Type;每種類型要 `AddCodeTableFuncion(type)` 註冊權限、再自己掛選單。library 兩者都不代勞。
- **取用不必引入 `@hcs/code-table`**:`codeTableOptions` / `codeTableViewer` 在核心,引 `@hcs/code-table` 只為了那組管理頁。
- **`isEnadled` 是 code 裡的拼字**:設定屬性名就是少一個字母,寫成 `isEnabled` 不會生效。
- **刪除帶 `Code` 的列要 Admin**:純顯示用(無 `Code`)的列一般 `Delete` 權即可刪;有 `Code`(被邏輯依賴)的列改需 `Admin`。
- **快取可能不即時**:pipe 預設 10 分鐘才重抓代碼表;後台剛改完選項要馬上反映,取用端得帶 `{ checkMinute: 0 }`。
- **預設組織 = 全域共用代碼**:不帶 `isDefaultOrg` 拿的是當前組織自己的代碼表;跨組織共用的字典要放在預設組織並以 `isDefaultOrg: true` 取用。

---

## 關聯

- entity 模型 / CRUD 管線 — [core/entity-api](../core/entity-api.md)
- 組織別 vs 預設組織(全域)資料範圍 — [core/multi-tenant](../core/multi-tenant.md)
- 權限樹 / 標準 CRUD 角色 — [core/permissions](../core/permissions.md)
- i18n 系統(語系切換、字串索引)— [core/i18n-system](../core/i18n-system.md)
- 選項類輸入元件(select / radiolist / checkboxlist 吃 `codeTableOptions`)— [frontend/controls](../frontend/controls.md)
- 清單欄位用 `codeTableViewer` 顯示 — [frontend/list](../frontend/list.md)
