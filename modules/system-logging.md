# SystemLogging 模組（系統紀錄 / 稽核軌跡）

平台的稽核機制分**兩半**,且可分開裝:

1. **記錄引擎**(`Hcs.Extensions.SystemLogging`):在**任何** entity API 上加一行 `EnableDefaultSystemLogging(...)`,新增 / 修改 / 刪除時自動擷取**欄位層級的前後值 diff** 寫進 `SystemLog`。你不必自己寫稽核欄位、不必比對舊值。
2. **檢視模組**(`Hcs.PlatformModule.SystemLogging` + 前端 `@hcs/system-logging`):提供 `SystemLog` 的查詢 API、一個檢視頁(清單 + 單筆 diff 檢視)、以及對應權限。

兩半獨立:只裝引擎 → 有記錄、但沒地方看;只裝檢視模組 → 有頁面、但沒人在記錄。實務上兩個都要。

---

## 啟用（後端）

三個動作,缺一不可:

```csharp
// ① 服務註冊(通常在 Startup,只需一次;內部有 idempotent 保護)
services.AddSystemLogging();

// ② 掛檢視模組(SystemLog 查詢 API + 權限)
builder.AddSystemLoggingModule();

// ③ 在「要被稽核」的 entity API 上開啟記錄
moduleBuilder.AddEntityApi<long, Invoice>(options =>
{
    options.EnableDefaultSystemLogging("MyApp.Invoice");   // 第一參數=功能代碼(分類用)
});
```

- ① 註冊記錄引擎用的服務(逐請求的 diff 擷取 pipe、寫入服務)。**漏掉**→ `EnableDefaultSystemLogging` 掛的 pipe 找不到服務,記不進去。
- ② 註冊 `SystemLog` 這個 entity 的查詢 API 與 `SystemLogging.SystemLog.*` 權限。**漏掉**→ 有記錄但前端 / API 讀不到。
- ③ **逐 entity、逐 API** 開啟。`EnableDefaultSystemLogging` 一次涵蓋 Post / Put / Delete 三支;也有只掛單一動作的多載。

> 第一參數的「功能代碼」(如 `"MyApp.Invoice"`)不必等於該功能的權限功能碼,但**慣例上對齊**——因為紀錄的可見範圍是用它比對使用者權限的(見[權限與可見範圍](#權限與可見範圍))。

---

## 預設記錄什麼

`EnableDefaultSystemLogging` 的預設行為:

- **Create**:記錄新增後的各欄位值(`To`)。
- **Update**:只記錄**真的有變**的欄位(舊值 `From` → 新值 `To`);沒變的欄位不入紀錄。
- **Delete**:記錄被刪資料的各欄位值(`From`)。
- **自動排除**主鍵、平台稽核欄位(`CreatedBy`/`CreatedByUser`/`CreatedTime`/`LastUpdatedBy`/`LastUpdatedByUser`/`LastUpdatedTime` 等 `IPlatformEntity` 欄位)與組織欄位(`OrgId` 等 `IOrganized` 欄位)——這些不會出現在 diff 裡。⚠️「誰改的 / 何時改」這組 `LastUpdated*` 也在排除之列,別誤以為它會自動記。
- 每筆紀錄帶一個 **`DataIdentity`**(人看得懂的識別,預設 = 主鍵值)與 `DataKey`(主鍵),方便清單上辨認「這是哪一筆」。

---

## 微調記錄內容（`ModelLoggingConfiguration`）

`EnableDefaultSystemLogging` 的第二參數可細調**這個 model 要怎麼記**:

```csharp
options.EnableDefaultSystemLogging("MyApp.Invoice", m => m
    .SetPrefix("models.MyApp.")                       // model / 欄位名的多語系前置詞
    .SetDataIdentityGetter(x => $"{x.No}({x.Id})")    // 清單上顯示的識別字串(預設只有主鍵)
    .CensorProperty(x => x.Password)                  // 遮罩:值記成 *****,但仍標示「有改」
    .IgnoreProperty(x => x.InternalFlag)              // 完全不記這個欄位
    .IncludeProperty(x => x.Status)                   // 強制記(即使值沒變)
    .SetClientType(x => x.TypeId, ClientTypeOption.CodeTable, _ => "MyApp.InvoiceType")  // 前端用代碼表元件顯示
);
```

| 方法 | 作用 |
|---|---|
| `SetPrefix` | model / 欄位名顯示用的多語系 key 前置詞 |
| `SetDataIdentityGetter` | 設定 `DataIdentity`(清單辨識用),預設只有主鍵 |
| `SetEnumPrefix` | enum 值顯示的多語系前置詞 |
| `CensorProperty` | **遮罩**:值記成 `*****`(密碼等敏感欄位);若前後值不同仍標示「有變更」,但不洩漏內容 |
| `IgnoreProperty` | **完全不記**這個欄位 |
| `IncludeProperty` | **強制記**這個欄位,即使值沒變(預設 Update 只記有變的) |
| `SetClientType` | 指定欄位的**前端顯示型別**(Enum / CodeTable / 自訂),決定 diff 畫面用哪個元件渲染值 |
| `LoadRefData` | 記錄時順帶查關聯資料、把某欄位記成衍生值(見下) |

### 記成關聯的衍生值（`LoadRefData`）

想把「存的是 FK id、但紀錄上要顯示關聯實體的名稱」這種需求一次查出:

```csharp
options.EnableDefaultSystemLogging("MyApp.Invoice", m => m
    .LoadRefData(x => new { x.CreatedByUser.Name })   // 一次把要用的關聯欄位查出
        .Map(x => x.CreatedBy).ToValue(r => r.Name)   // 把 CreatedBy 欄位記成關聯使用者的 Name
);
```

`LoadRefData` 用一次查詢把關聯資料撈出,`Map(欄位).ToValue(轉換)` 把該欄位在紀錄上的值換成你算出來的字串。

---

## 權限與可見範圍

檢視模組產生一個功能 `SystemLogging.SystemLog`,兩個權限:

| 權限 | 作用 |
|---|---|
| `SystemLogging.SystemLog.View` | 進得了檢視頁、查得到紀錄;同時涵蓋頁面用到的兩支輔助 API(功能清單下拉、建立者 reference picker) |
| `SystemLogging.SystemLog.AllowChildOrgData` | 看得到**子組織**的紀錄,且能在清單展開組織名稱欄 |

⚠️ **可見範圍是雙層 gate**,這是最容易誤解的地方:

1. 有 `SystemLogging.SystemLog.View` 才進得了這個頁面 / API。
2. **但即使有 View,每一筆紀錄還會再依「你有沒有那筆紀錄對應動作的權限」過濾**——一筆 `MyApp.Invoice.Create` 的紀錄,只有**持有 `MyApp.Invoice.Create` 權限**的人看得到。

也就是:**你只看得到「你本來就有權做的那些動作」的稽核軌跡**。給某人 `SystemLogging.SystemLog.View` 不等於讓他看到全平台的所有操作紀錄——他只會看到他自己權限範圍內那些功能的紀錄。要讓某角色看到某模組的全部操作,得連那個模組對應的權限一起給。

---

## 前端：檢視頁 + 值渲染

```typescript
imports: [ HcsSystemLoggingProviderModule.forRoot() ]   // 提供 i18n、選單、內建值元件
```

路由用 `HcsSystemLoggingProviderModule.defaultRoute`(落在 `system-logging/` 下)掛進 app:

- `system-logging/system-log`——紀錄清單(可依功能、`DataIdentity`、`DataKey`、建立時間〔預設最近 7 天〕、建立者、組織篩選)
- `system-logging/system-log/:id`——單筆紀錄的 diff 檢視(唯讀)

選單項目由 library 提供,以 `SystemLogging.SystemLog.View` gate。

### diff 值怎麼渲染：`SYSTEM_LOG_VALUE_COMPONENT`

紀錄裡每個欄位的前 / 後值,前端依該欄位的 **client type 或 CLR 型別**挑一個元件來顯示。內建支援:

| 內建鍵 | 比對的是 | 顯示為 |
|---|---|---|
| `Enum` | client type(enum 欄位**自動**標成 Enum) | enum 多語系標籤 |
| `CodeTable` | client type(需 `SetClientType`) | 代碼表標籤(接 CodeTable 模組) |
| `FunctionCode` / `FunctionRoleCode` | client type(需 `SetClientType`) | 權限功能碼 / 角色碼的友善名稱 |
| `DateTime` / `DateTimeOffset` / `Boolean` | **CLR 型別**(依欄位型別自動套,免設定) | 格式化日期時間 / 是非 |

差別在綁定方式:日期 / 布林是**依欄位的 CLR 型別自動配對**(不必 `SetClientType`);enum 自動標成 `Enum`;代碼表 / 權限碼則要用 `SetClientType` 顯式宣告 client type。

要為自訂型別加渲染元件,multi-provide 一個 `SYSTEM_LOG_VALUE_COMPONENT`,鍵可用 `clientType` / `type` / `field`(或組合,比對由精到寬取第一個命中):

```typescript
{
  provide: SYSTEM_LOG_VALUE_COMPONENT,
  useValue: { clientType: 'MyType', componentType: MyValueComponent } as ISystemLogValueComponentProvideData,
  multi: true
}
```

要讓某欄位的 client type 等於自訂值(如上面的 `'MyType'`),後端用 `SetClientType` 的**字串多載**——`ClientTypeOption` enum 只有 `None`/`Enum`/`CodeTable`,自訂字串走字串多載:

```csharp
m.SetClientType(x => x.SomeField, "MyType");   // 對應前端 clientType: 'MyType'
```

元件內以 `@Inject(SYSTEM_LOG_VALUE)` 取得當前值(`ISystemLogValue`:含 `value` / `clientType` / `type` / `field` / `clientTypeData`)。

---

## Gotchas

- **三件套缺一不可**:`services.AddSystemLogging()`(引擎) + `builder.AddSystemLoggingModule()`(檢視) + 各 entity 的 `EnableDefaultSystemLogging`(記錄)。漏引擎 → 記不進;漏檢視模組 → 看不到;漏 `EnableDefaultSystemLogging` → 那個 entity 不被稽核。
- **View ≠ 看全部**:`SystemLogging.SystemLog.View` 只開「進得了頁面」;每筆紀錄再依使用者**對該動作的權限**過濾。要看某模組全部操作,得連那模組的權限一起給。
- **Update 只記有變的欄位**:沒變的欄位不入 diff。要讓某欄位無論有無變更都出現,用 `IncludeProperty`。
- **遮罩 vs 忽略不同**:`CensorProperty` 仍會留一筆「有變更」的痕跡(值是 `*****`),`IgnoreProperty` 是整個欄位都不記。記密碼類欄位要用 `CensorProperty` 而非讓它原值寫進稽核。
- **`DataIdentity` 一個請求只留最後一個**:同一請求若觸發多個 model 的 identity 寫入,只會記到最後一個。複合操作要稽核多個實體的識別時注意。
- **主鍵 / 稽核 / 組織欄位自動排除**:diff 不含主鍵、`IPlatformEntity` 稽核欄位與 `IOrganized` 組織欄位——這些是平台層級欄位,不算業務變更。

---

## 關聯

- 在 entity API 上掛 pipe / 設定各動作 — [core/entity-api](../core/entity-api.md)
- 資料 pipe 機制(`EnableDefaultSystemLogging` 掛的是逐請求 data pipe)— [core/data-pipes](../core/data-pipes.md)
- 權限樹 / 功能碼 / 標準 CRUD 角色(可見範圍比對的來源)— [core/permissions](../core/permissions.md)
- 子組織資料可見性 — [core/multi-tenant](../core/multi-tenant.md)
- 代碼表值渲染(`SetClientType(..., CodeTable, ...)` 接的就是它)— [modules/code-table](code-table.md)
- i18n 前置詞(`SetPrefix` / `SetEnumPrefix` 對應的字串索引)— [core/i18n-system](../core/i18n-system.md)
