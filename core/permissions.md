# 權限 / 角色樹

平台的授權核心:**你不寫權限檢查**——後端用 `AddModuleFuncion` 宣告權限樹,五支 entity API 自動掛上對應權限;前端用 `permission.hasPermission(...)` 控制 UI。後端宣告與前端檢查靠**同一條權限字串**對齊,這條字串對不上就是「按鈕消失 / 403」的頭號原因。

---

## 三層模型

```
Function(功能,有 code)
  └── Permission(權限,如 View / Create / 自訂)
        └── 綁定一到多支 API 端點(誰持有此權限就能呼叫這些端點)
```

- **權限字串** = `{Function.Code}.{PermissionCode}`,例 `MyApp.Invoice.View`、`MyApp.Invoice.Approve`。這條字串就是對齊契約:後端宣告它、指派給使用者、前端查它。
- 使用者被指派的是**權限字串的集合**(角色 = 一組權限);端點要求的是**某支 API 的存取權**,框架負責把「使用者持有的權限」對應到「端點要求的 API」。

---

## 後端:宣告權限樹

在模組的 `Build` 裡,對每個功能 `AddModuleFuncion`,再用 `AddStandardApiRoles` / `AddPermission` 描述權限:

```csharp
var api = b.AddEntityApi<long, Invoice>(/* ... */);   // 回傳 5 支 API 的 role token

b.AddModuleFuncion("MyApp", "Invoice", f =>
{
    f.AddStandardApiRoles(api);                        // View / Create / Modify / Delete 四個標準權限
});
```

> `AddModuleFuncion` 的拼字就是這樣(少一個 `t`)——是公開 API 的既定名稱,照用即可。兩個多載:`AddModuleFuncion(code, build)` 直接給完整 code;`AddModuleFuncion(module, function, build)` 會幫你串成 `"{module}.{function}"`。

### 五支 API 怎麼自動掛權限

`AddStandardApiRoles(api)` 一次建立四個標準權限,並把五支 entity API 綁上去:

| 標準權限 | 綁定的 API | 產生的權限字串 |
|---|---|---|
| `View` | `Get`(單筆)+ `Query`(列表) | `MyApp.Invoice.View` |
| `Create` | `Post` | `MyApp.Invoice.Create` |
| `Modify` | `Put` | `MyApp.Invoice.Modify` |
| `Delete` | `Delete` | `MyApp.Invoice.Delete` |

持有 `MyApp.Invoice.View` 的使用者即可呼叫 Get 與 Query;持有 `Create` 即可 Post,依此類推。

### 自訂權限

標準四個不夠時,`AddPermission` 加一個,再用 `AddRole` 把它綁到額外的 API token,或當成純前端旗標:

```csharp
b.AddModuleFuncion("MyApp", "Invoice", f =>
{
    f.AddStandardApiRoles(api);
    f.AddPermission("Approve", p => p.AddRole(approveApi.Post));   // 綁一支自訂 API
    f.AddPermission("ExportAll");                                  // 無綁 API:純前端 UI 旗標
});
```

- `AddPermission("Approve", …)` → 權限字串 `MyApp.Invoice.Approve`。
- 沒 `AddRole` 任何 API 的權限(如上面的 `ExportAll`)不會 gate 任何端點,只供前端 `hasPermission('ExportAll')` 控制 UI。
- 也可往標準權限**追加** API:`f.AddStandardApiRoles(api, ctx => ctx.View.AddRole(extraQuery.Query))` 讓持有 View 的人也能用某支額外查詢。

### 多支 API 怎麼分權限(判準)

一個功能頁往往不只 entity 的五支 CRUD——還有 picker 查詢、報表、動作型 flow API。判準是「**這支 API 的授權該跟誰走**」:

| 情境 | 做法 |
|---|---|
| 跨頁共用的 lookup(多個頁面都要的基礎資料查詢) | 掛 `moduleBuilder.Everyone`,不佔功能權限 |
| **頁面專屬的輔助 API**(該頁的 picker 查詢、報表、檢查) | **追加進標準權限**——查詢類進 `View`、動作類進語意最近的 `Modify`/`Create`,讓「授了 View 整頁就能動」(self-contained) |
| 要能單獨給/不給某群組的動作(下架、簽核、展延) | 開新 `AddPermission("X", p => p.AddRole(token))` |
| 純 UI 旗標 / 資料範圍開關 | `AddPermission("X")` 不綁 role |

```csharp
// self-contained:Employee 頁的部門 picker 查詢跟著 Employee.View 走
moduleBuilder.AddModuleFuncion("Hr", "Employee", f => f.AddStandardApiRoles(employeeApi, ctx =>
{
    ctx.View.AddRole(departmentApi.Get).AddRole(departmentApi.Query);
    ctx.Modify.AddRole(sortToken);                 // 動作型 flow API 掛語意最近的標準權限
}));

// 跨頁共用 lookup:對所有登入者開放(可與 $expand 白名單混掛)
moduleBuilder.Everyone.AddRole(codeLookup).AddOdataPermission<Department>(o => o.AllowExpand(x => x.Parent.Name));
```

語意邊界:追加進 `View` 的是「**資料讀取能力**」,不是頁面——持有 Employee.View 的人查得到部門資料(picker 正是靠這個),但部門自己的頁面入口仍由 `….Department.View` 的選單 gate 控制、寫入 API 也不會跟著開。

> 舊寫法「`AddStandardApiRoles(api)` 之後另外 `AddPermission(StandardRoles.View, b => b.AddRole(...))` 同名二次宣告」也能動(授權端與群組同步端都按 Code 合併),但會產生重複的 permission builder——**追加一律用 ctx 第二參數**;鏈式 `AddPermission` 留給新 code 的自訂權限。

---

## 授權怎麼跑(執行模型)

關鍵且非直覺——**role 是每次請求即時注入的,不寫進 token**:

1. **宣告期(啟動)**:整個權限樹攤平成「權限字串 → 它綁定的 API 集合」的目錄。
2. **登入**:回傳的 `LoginResult.Roles` 是該使用者持有的權限字串陣列(給前端用)。
3. **每個請求**:JWT 通過驗證時,平台**當場**用使用者 id 向 role service 重撈一次權限字串,逐個加成 `ClaimTypes.Role` claim。
4. **端點授權**:每支 entity API 要求「對應這支 API 的存取權」;框架檢查使用者持有的權限裡,有沒有任一個其綁定集合涵蓋這支 API——有就放行,否則 `403 Forbidden`(未登入則 `401`)。

> **改權限不必重發 token**:因為 role claim 是每次請求即時撈的,後台改了使用者權限,**下一個請求**就生效,不需重新登入或換 token。代價:每請求一次 role 查詢(平台對使用者權限做了 10 分鐘記憶體快取緩衝)。
>
> **但前端 `user.roles` 是登入當下存的、不會自己更新**——後台改權限後,前端 UI(按鈕顯隱)要等使用者重新登入或呼叫 refresh token 才會反映。後端已即時生效、前端尚未,是正常的不對稱,不是 bug。

---

## 權限怎麼指派給人(群組)

宣告好的權限字串,**一律透過「群組」指派給使用者**——平台沒有「把權限直接掛在使用者身上」這回事:

- **群組(`PlatformGroup`)持有一組角色設定(`PlatformGroupRole`)**:每筆是一個權限字串(`{Function.Code}.{PermissionCode}`)配一個**三態**——
  - `Granted`(允許)
  - `Denied`(拒絕)
  - `NotSet`(未設,對這條沒意見)
- **使用者透過 `PlatformUserGroup` 加入群組**,可同時屬多個群組。
- **有效權限 = 所屬「啟用中」群組的 `Granted` 角色聯集**,再扣掉任一群組標為 `Denied` 的:
  - 只有 `IsEnabled` 的群組計入,停用群組的角色一律不算。
  - `NotSet` 完全忽略(等於沒設這條)。
  - **`Denied` 最優先、蓋過 `Granted`**:同一條權限字串只要在**任一**所屬群組被 `Denied`,即使在別的群組是 `Granted`,使用者最終**沒有**這條權限。這讓「一個大範圍授權的群組 + 一個專門拔權的群組」能精準收權。

> 這就是「[授權怎麼跑](#授權怎麼跑執行模型)」裡 role service 每次請求重撈的來源:它把使用者所屬啟用群組的 `PlatformGroupRole` 聚合成權限字串集合(先扣掉 Denied),再逐個加成 role claim。
>
> 指派這些的管理 UI 是 Basic 模組的群組頁(權限格設三態、成員設定),見 [modules/basic](../modules/basic.md)。

---

## 前後端對齊(最容易做錯的地方)

前端每個 CRUD 元件 provide 一個 `HCS_FUNCTION_NAME`(= 後端的 Function code),再用 `permission.hasPermission(verb)` 查權限:

```typescript
// 元件 provider
{ provide: HCS_FUNCTION_NAME, useValue: 'MyApp.Invoice' }

// template / 程式
*ngIf="permission.hasPermission('View')"      // 查 'MyApp.Invoice.View'
*ngIf="permission.hasPermission('Approve')"   // 查 'MyApp.Invoice.Approve'
```

`hasPermission(x)` 的查法是**兩段式**:

1. 先原樣查 `x` 在不在 `user.roles`(讓**全域權限**不必前綴就能查,例如每個登入者預設持有的 `Everyone.All`)。
2. 沒中、且元件有 `HCS_FUNCTION_NAME`,再查 `'{functionName}.{x}'`。

**對齊契約**:`HCS_FUNCTION_NAME` + verb 組出的字串,必須和後端宣告的權限字串**逐字相同**(大小寫、每一段都要對)。

> **常見對不上的情形:**
> - 後端 `AddModuleFuncion("MyApp", "Invoice")` 產生 `MyApp.Invoice.View`,前端 `HCS_FUNCTION_NAME` 卻填成 `'MyApp.Invoices'` / `'Invoice'` → 永遠查不到 → 按鈕永遠藏起來。
> - **選單項目通常寫死完整字串**(`hasPermission('MyApp.Invoice.View')`),因為選單沒有 `HCS_FUNCTION_NAME` 的 component context,沒有第 2 段 fallback;這裡更要逐字對齊。
> - 標準 CRUD 按鈕(複製 / 編輯 / 刪除)內部就是查 `'Create'` / `'Modify'` / `'Delete'`,靠元件的 `HCS_FUNCTION_NAME` 補前綴——別在這類元件忘了 provide。

權限名稱在 UI 上的顯示(i18n)走 `permissionTranslate` pipe(`permissions.{code}.{perm}` → `permissions.{perm}` → 原字串 fallback),屬 i18n 主題,見 [i18n 系統](i18n-system.md)。欄位名 i18n key 用到的 `functionName` 前綴對齊,見 [驗證錯誤](validation-errors.md) 的欄位名前綴一節。

---

## OData `$expand` 白名單

查詢 API(`Query`)的 `$expand` 預設**全部擋掉**;要放行特定關聯,在權限上用 `AddOdataPermission` 宣告白名單:

```csharp
f.AddStandardApiRoles(api, ctx => ctx.View
    .AddOdataPermission<Invoice>(o => o
        .AllowExpand(x => x.Customer)                       // 允許 $expand=Customer
        .AllowExpand(x => x.Lines, l => l.AllowExpand(z => z.Product))));  // 巢狀:Lines.Product（只放行 Lines.Product，不含 Lines 本身）
```

- 白名單是**綁在權限上**的:不同權限可放行不同的 expand 路徑(例:只有 `Admin` 能展開敏感子關聯,一般 `View` 不行)。
- 請求帶了不在白名單內的 `$expand`,查詢驗證直接擋下回 `expand {path} not allow`。
- 使用者實際可用的白名單 = 其持有的各權限白名單的**聯集**。
- ⚠️ **巢狀 `AllowExpand` 只放行該子路徑、不含中介層**:上例 `AllowExpand(x => x.Lines, l => …)` 只把 `Lines.Product` 加進白名單,`$expand=Lines`(不帶子項)仍會被擋(比對是 strict prefix)。要連 `Lines` 本身也放行,得再單獨加一條 `.AllowExpand(x => x.Lines)`。

---

## Gotchas 一覽

- **role claim 即時注入、不入 token** → 改權限即時生效(後端);前端 `user.roles` 需 re-login / refresh 才更新。
- **`Denied` 覆蓋 `Granted`**:同一條權限若使用者跨群組同時被授予(Granted)與拒絕(Denied),**Denied 優先**、該權限視為沒有。「明明指派了卻沒生效」常是這個。
- **`AddModuleFuncion` 拼字固定**(少一個 t),照用。
- **前後端權限字串逐字對齊**;選單寫死完整字串、無前綴 fallback。
- **`$expand` 預設全擋**,要 `AllowExpand` 逐條開,且綁在權限上。
- 子組織資料授權(`AllowChildOrganizationData`)是**資料列過濾**、不是這裡的端點授權,屬多租戶機制,見 [data-pipes](data-pipes.md)。

---

## 關聯

- 五支 API 怎麼來、管線怎麼組:[entity-api](entity-api.md)。
- 權限/驗證步驟掛在 pipe 上:[pipe](pipe.md)。
- 欄位名 / 權限名 i18n、`functionName` 前綴對齊:[validation-errors](validation-errors.md)、[i18n-system](i18n-system.md)。
- 資料層級的多租戶過濾(與端點授權不同層):[data-pipes](data-pipes.md)。
