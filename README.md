# HCS Platform

### 從 ERP 系統反推出的「全端可組裝平台 SDK」

> **後端不寫 Controller，前端不寫 CRUD 頁。**
> 你描述業務模型，平台長出整條 stack — Controller、Repository、權限檢查、稽核軌跡、簽核流程、OData 查詢端點、Angular 列表與表單頁，全部自動到位。

---

## 為什麼存在

寫過十個內部系統的人都知道：
**每個系統的 80% 都長一樣。**

登入、組織、權限、CRUD、清單頁、表單頁、字典表、稽核、簽核、檔案上傳、Excel 匯出、報表、推播、多語系、JWT、2FA…

業務需求只在最後那 20%。

可是工程師每次都從 `dotnet new` 開始重蓋一次那 80%。

**HCS Platform 把那 80% 變成宣告式設定。**

你只負責那 20%。

---

## 三段寫完

```csharp
// 後端：描述業務 model + 想要的 API + 權限
public class Module : IPlatformModule
{
    public void Build(IPlatformModuleBuilder b)
    {
        b.AddModel<MyModelConfig>();
        var api = b.AddEntityApi<long, Invoice>(opt =>
        {
            opt.AllowChildOrganizationData("Invoice.ChildOrg");
            opt.SetupFilePipes(x => x.Attachment);
            opt.EnableDefaultSystemLogging("MyApp.Invoice", m =>
                m.SetPrefix("models.MyApp.").CensorProperty(x => x.SecretField));
            opt.ConfigApprovalFlow(...);
        });
        b.AddModuleFuncion("MyApp.Invoice", o => o.AddStandardApiRoles(api));
    }
}
```

```csharp
// 註冊一行
services.AddHcsPlatform(b => b.AddModule<MyFeature.Module>());
```

```typescript
// 前端
constructor(f: DataSourceFactory) { this.data = f.getDataSource(Invoice); }
// <hcs-data-grid [dataSource]="data"></hcs-data-grid>
```

**得到什麼**：5 支 RESTful API (Get/Query/Post/Put/Delete) + OData 查詢 (`$filter / $select / $expand`) + 多租戶資料隔離 + 子組織授權 + 檔案上傳 + 完整稽核軌跡 + 三層權限樹（View / Modify / Delete）+ 簽核流程整合 + Angular 列表頁 + Angular 表單頁 + 權限選單。

**寫了多少 code**：一個 class，約 15 行。

---

## 核心特點

### 1. Declarative Composition — 描述，而非實作

平台所有功能都用 fluent builder 描述：

```csharp
options.ConfigPostApi(x => x
    .OnValidate(v => v.Unique(u => u.AddProperty(p => p.Code).AddProperty(p => p.OrgId)))
    .OnCreated(c => c.SaveChildsFor(s => s.Items))
);
```

**沒有 Controller、沒有 ViewModel、沒有 mapper、沒有 repository、沒有 unit of work。**

### 2. Pipe-based DI Extension Model — 不繼承、不改框架

所有切入點接受 `Pipe(...)`，**lambda 參數會自動 DI**：

```csharp
opt.ConfigPostApi(x => x.OnCreated(y => y.Pipe(
    ((DbContext db, IHttpContextAccessor http, ITranslate t) svc, Invoice inv) =>
    {
        // svc.db, svc.http, svc.t 全部自動注入
        // 想加新的服務？多寫一個就好，不必改 interface
    })));
```

支援同步 / 非同步 / DbContext / ScopedDbContext / 任意 tuple 服務 / i18n。
還有 **條件分支** (`SwitchCase` / `AddCase`)、**子集合處理** (`SaveChildsFor` / `QueryChildFor`)、**角色條件** (`PipeIfRole`)。

> **這是整個平台最不一樣的地方。** 一般 ORM / framework 擴充靠繼承或實作 interface — 改要動原始碼。Pipe 直接內聯，永遠回不到「為了加一個 hook 要建一個 class」的世界。

### 3. 多租戶 (Multi-tenant) 是內建，不是 add-on

`Organization` / `PlatformUser` / `PlatformGroup` 是平台一級公民：

```csharp
opt.AllowChildOrganizationData("Invoice.ChildOrg");   // 一行授權「能看子組織資料」
```

底層是 `IDataFilterPipe` 管道機制（不是 EF Core `HasQueryFilter`），可以針對 entity / role / 自訂條件動態組合過濾。

### 4. 權限既然要做就做到最細

每個 API 端點自動掛上角色 — `AddStandardApiRoles` 一行同時宣告 View / Modify / Delete 三層權限：

```csharp
b.AddModuleFuncion("MyApp.Invoice", opt => opt
    .AddStandardApiRoles(api, c => c.View.AddOdataPermission<Invoice>(b => b
        .AllowExpand(x => x.Customer.Name)      // OData $expand 走白名單
        .AllowExpand(x => x.Items, x => x.AllowExpand(z => z.Product))
    ))
    .AddPermission("Approve", b => b.AddRole(approveApi))
    .AddPermission("Export"));
```

**預設拒絕**：OData `$expand` 不在白名單一律打回，避免關聯資料外洩。
**即時失效**：JWT `OnTokenValidated` 動態注入 role claim — 改權限不必重發 token；`UserDataChangeTime` 機制可強制舊 token 過期。

### 5. 簽核流程是動態註冊，不是寫死

```csharp
opt.AddFlowCode<Invoice>("InvoiceFlow", flow =>
{
    flow.AddValueProvider("總金額", x => x.Pipe(inv => inv.Total));
    flow.AddValueProvider("是否高風險", x => x.Pipe(inv => inv.Total > 1000000));
    flow.AddUserProvider("主管", x => x.Pipe((DbContext db, Invoice inv) => GetMgr(db, inv)));
    flow.AddEventHandler("發送通知信", x => x.Pipe(SendNotificationMail));
});
```

流程由管理員透過 **前端的階段式編輯器** 設計 — 每個階段（stage）有展開面板可配置「入場事件 / 更新事件 / 離場事件」、可執行的「行動」清單、以及「下一階段的條件路由」；事件與條件透過條件編輯對話框組合判斷式；另有圖形視覺化頁（用 `@swimlane/ngx-graph` 渲染為有向圖）讓管理員檢查流程結構。配置存進 DB 動態執行。程式碼只負責註冊「平台不會自己知道」的部分 — 業務條件值怎麼算、簽核人怎麼解析、流程事件要做什麼。

### 6. 稽核軌跡是 chainable

```csharp
opt.EnableDefaultSystemLogging("MyApp.Invoice", log => log
    .SetPrefix("models.MyApp.")
    .CensorProperty(x => x.SecretField)                          // 不記敏感欄
    .IgnoreProperty(x => x.SystemFlag)                           // 不記系統欄
    .LoadRefData(x => new { x.CreatedByUser.Name })              // 帶入關聯資料
    .Map(x => x.CustomerId).ToValue(y => y.Customer.Name)        // ID → 可讀名稱
    .SetDataIdentityGetter(x => $"{x.Code}({x.Id})"));
```

操作者、時間、新舊值 diff、可讀格式 — 全自動，**而且能對使用者用語言檔翻譯**。

### 7. 前後端 SDK 同步演進

平台前端是 **9 個 Angular library projects**，每個對應一個後端模組：

```
core              ← 骨幹：OData / Form / Grid / 權限 / i18n / 登入
basic             ← 對應後端 Basic 模組 (User/Group/Org)
code-table        ← 對應 CodeTable 模組
approval-flow     ← 對應 ApprovalFlow 模組
system-logging    ← 對應 SystemLogging 模組
app-update        ← 對應 AppUpdate 模組
third-party-login
two-factor-authentication
two-factor-authentication-google
```

**權限字串、Function code、entity TypeScript 模型** — 前後端用同一條 `"Module.Entity.Action"` 字串對得起來。

> 後端 `AddModuleFuncion("Basic.PlatformUser")` ⟷ 前端 `HCS_FUNCTION_NAME: "Basic.PlatformUser"` ⟷ 角色字串 `"Basic.PlatformUser.View"`

### 8. 不 fork SDK 也能客製

前端每個 library 開放 `createRoute(overrides)`：

```typescript
HcsBasicProviderModule.createRoute({
    platformUserForm: () =>
        import('./my-custom-user-form.component').then(x => x.MyCustomUserFormComponent)
})
```

換掉預設組件，路由結構、權限、i18n key 完全保留。**SDK 升級時你的客製不會被覆蓋。**

### 9. 一份 codebase，Web + iOS + Android

Capacitor 5 + Ionic 7 整合：

```typescript
constructor(p: HcsPlatform) {
    if (p.isWeb) { /* desktop UI */ }
    if (p.isAndroid || p.isIOS) { /* native UI */ }
    if (p.isTablet) { /* tablet UI */ }
}
```

掃條碼、列印、分享、推播 — Capacitor plugin 全套位。

---

## 還有什麼

| 領域 | 內建能力 |
|---|---|
| **登入** | JWT、Google OAuth、Facebook OAuth、IP 白名單、Proxy login（管理員代登入） |
| **2FA** | Google Authenticator（TOTP），可擴充其他 provider |
| **儲存** | 本機檔案、Azure Blob、SMB／網路芳鄰 |
| **發訊** | Email（SMTP / MailKit）、Firebase Cloud Messaging（Android）、APNs（iOS）、三竹簡訊（台灣 SMS） |
| **匯出** | Excel（ClosedXML 樣板引擎，cell-level value transform） |
| **快取** | Distributed memory cache、HMAC 簽章保護的快取端點、防驚群的 `GetOrCreateAtomicAsync` |
| **鎖** | 分散式原子鎖（SQL Server / MySQL / Redis 三種實作） |
| **資料表** | 字典/代碼表機制（含多語系），刪除前自動檢查引用關係 |
| **背景任務** | `IHostedService` 不擋，標準 ASP.NET Core 模式 |
| **加密** | AES / MD5 / SHA、API payload 加密 middleware |
| **時鐘** | `X-HCS-Server-Ts` response header 統一伺服器時鐘給前端校時 |

---

## Track record

- **生產系統**：內部與客戶系統長期運行於此平台
- **API 穩定性**：經歷 Angular 11 → 17 大跳，**SDK 對外 API 幾乎不變**。舊系統升級主要是框架版本升級，業務程式碼不重寫
- **可組裝性**：每個內建模組（Basic / CodeTable / ApprovalFlow…）可獨立啟用，**不用就不付出代價**
- **權限的成熟度**：上線系統實際運行三層權限樹 + OData expand 白名單 + 子組織授權 + group-role 二維過濾，沒踩過權限漏洞

---

## 你不會得到什麼

誠實一點：

- **不是一個跨組織開源生態**。文件量、社群規模、第三方擴充庫沒辦法跟 ABP Framework / OpenIddict 比。
- **強制平台契約**。entity 要有 `Id` 與 `OrgId`、權限要走 Function code、前端 component 要宣告 `HCS_FUNCTION_NAME`。不接受這套契約，所有自動化都失效。
- **不是 microservice 框架**。單體 + 模組化的設計，多服務拆分需要自己處理 transaction boundary。

---

## 設計理念

> **業務邏輯該短，框架程式碼該長。**
> 大多數系統剛好相反 — 業務藏在三層抽象底下，每加一個欄位要動五個檔案。
>
> 這份 SDK 把選擇反過來：複雜度集中在平台核心一次解決，每個業務模組是一張可讀的清單 — entity、API、權限、簽核、稽核，**像在寫需求文件一樣寫程式**。

---