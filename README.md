# HCS Platform

ERP 開發平台:ASP.NET Core(.NET)+ Angular 17 + OData + EF Core。後端以 fluent / 宣告式設定組裝——描述業務模型與規則,平台在執行期產生 Controller、查詢端點、權限與多租戶過濾,不需手寫 Controller / Repository / 權限中介層。

> 本文件庫皆為**參考資料**,可能與現狀有出入。**與 code 衝突時,一律以 code 為準。**

---

## 目錄

- [這是什麼](#這是什麼)
- [這樣設計帶來什麼](#這樣設計帶來什麼)
- [沿革](#沿革)
- [平台能力](#平台能力)
- [模組開發](#模組開發)
- [前端](#前端)
- [專案地圖](#專案地圖)
- [開發者快速上手](#開發者快速上手)

---

## 這是什麼

HCS Platform 是「平台框架 + 可組裝模組系統」,不是業務系統本身。它把下列能力綁在一個 host,讓模組只需描述業務模型:

- 自動產生 RESTful CRUD + OData 查詢端點
- JWT 驗證 + 即時 role claim 注入 + 細粒度權限
- 多租戶(Organization)資料隔離 + 子組織授權
- Pipe 擴充模型(DI 自動注入)
- Angular 17 SPA host 整合 + 自有 component library

具體說,同一個業務功能在傳統 ASP.NET + Angular 要手寫的一整套——Controller、Repository、DTO、OData 查詢、權限檢查中介層、多租戶過濾、前端 API client——在平台上**收斂成一段 `IPlatformModule.Build` 的 fluent 宣告**,Controller / 端點 / 權限 / 過濾在執行期生成。你寫的是「這模組有哪些 entity、套什麼規則」,而不是「怎麼接 HTTP、怎麼查 DB、怎麼擋權限」。

**核心精神 —— 一切組裝皆 pipe。** 平台的擴充不靠繼承、也不改框架,而是 **AOP 式「插管」**:把行為拆成一個個 pipe,掛到 CRUD / 驗證 / 查詢 / 服務注入的生命週期切點上。原則是**把邏輯切到最小可重用區塊、每個 pipe 維持短小且單一職責**——越小越短,就越能跨 entity / API / 模組自由組合與重用(參數還會自動 DI)。這是讀懂整個平台的鑰匙,見 [pipe](core/pipe.md)。

簽核流程、系統稽核、字典/代碼表、App 版本、2FA、第三方登入、多通道發訊等為**可選模組**(見[專案地圖](#專案地圖))。

---

## 這樣設計帶來什麼

- **樣板少一個量級**:一段 `IPlatformModule.Build` 宣告就是一個模組的端點 + 權限 + 多租戶過濾,沒有 Controller / Repository / DTO / mapping 的重複骨架要維護。
- **擴充全走同一套 pipe 模式**:客製(驗證、寫入副作用、查詢加工、服務注入)雖各有對應的 pipe 介面(`IDataPipe`、`ITable` 等),但都是同一種「短小單一職責、參數自動 DI、可跨 entity / API / 模組重用」的切點組裝——學會一種就能套用其餘,不必記一堆異質 hook 與攔截點(見 [pipe](core/pipe.md))。
- **多租戶貼合組織階層**:組織樹 + 「讀能上下、寫只能往下」直接對應 ERP 的總公司 / 分店 / 櫃位,不必每支查詢自己拼 `OrgId` 條件(見 [multi-tenant](core/multi-tenant.md))。
- **權限即時生效**:改角色不必重發 token、改密碼 / 改資料即時撤銷舊 token——權限變動不卡在登入週期(見 [login](core/login.md))。
- **前後端用同一條功能字串對齊**:後端宣告的功能碼 = 前端權限判斷 = 選單顯示依據;SDK 元件(grid / form / datasource)直接對接自動生成的 OData 端點,不手寫 API client。

---

## 沿革

平台始於 **2018-05**(1.0;核心以 `netstandard2.0` class library 起家),隨 .NET 大版本逐次升級。下表時間取自核心專案 `TargetFramework` 的變更時點:

| 時間 | .NET 目標 | 套件版本前綴 |
|---|---|---|
| 2018-05 | 平台啟動(`netstandard2.0`) | — |
| 2020-04 | .NET Core 3.1 | `31` |
| 2020-12 | .NET 5.0 | `50x`(首版 `501` = 5.0.1) |
| 2021-11 | .NET 6.0 | `60x` |
| 2023-05 | .NET 7.0 | `70x` |
| 2024-03 | .NET 8.0 | `80x`(如 `802` = 8.0.2) |
| 2026-04 | .NET 10.0(跳過 9.0) | `100x`(如 `1005` = 10.0.5) |

> **套件版本前綴 = 當時的 .NET 版本去掉點號**:3.1 記成 `31`;5.0 起補成三碼並帶 patch(`501` = 5.0.1、`802` = 8.0.2、`1005` = 10.0.5),其後才接平台自身的版號(如 `1005.4.3`)。看到套件號開頭就能反推它跑在哪個 .NET 上。
>
> 此前綴規則**自 2020-04(`31.x`)起**才採用;在那之前(2018–2019)套件是一般 semver(`1.0.0-beta1` → `1.1` → `2.x` → `3.x` → `22.x`),開頭數字**與 .NET 版本無關**,別拿去反推執行環境。

---

## 平台能力

以下是平台**核心**(`Hcs.Platform.Core`)一律提供的能力;功能性的東西(簽核、稽核、2FA…)是可選模組,不在此列。

### 1. 模組:一個 `IPlatformModule` 宣告整組功能

每個業務模組實作 `IPlatformModule.Build(IPlatformModuleBuilder)`,一段 fluent 設定同時宣告:

- Entity → DB 對應(`AddModel<>`)
- 五支 RESTful API:`AddEntityApi<TKey, TEntity>` 自動產生 Get / Query / Post / Put / Delete(**不寫 Controller**,控制器在執行期動態產生;CRUD 管線、生命週期 hook、交易範圍見 [entity-api](core/entity-api.md))
- 自訂端點:`AddQueryFlowApi<T>` / `AddGetFlowApi` / `AddPostFlowApi` / `AddPutFlowApi`(不綁 entity 也能產出端點)
- 子集合處理:`SaveChildsFor(s => s.Items)` / `QueryChildFor(c => c.Items)`
- 細粒度權限樹:`AddModuleFuncion` + `AddStandardApiRoles` + `AddPermission`
- 公開端點:`moduleBuilder.Everyone.AddRole(api)`(不需登入)
- 檔案欄位:`SetupFilePipes(x => x.Photo)`
- 模組私有服務:`moduleBuilder.Services.AddScoped<MyService>()`

### 2. Pipe 擴充點(參數自動 DI)

所有切入點(`OnCreated` / `OnUpdated` / `OnDeleted` / `OnValidate` / `OnQueryed` / `OnGeted` / `OnKeyGeted`)都接受 `Pipe(...)`,**Pipe lambda 的參數會自動從 DI 注入**:

```csharp
options.ConfigPostApi(x =>
    x.OnCreated(y => y.Pipe(
        ((ScopeDbContextState state, IHttpContextAccessor http) services, Customer x) =>
        {
            // services.state / services.http 由框架注入
        })));
```

支援純 model / async / 加 `DbContext` / 加任意 tuple 服務 / 加 i18n `ITranslate` 等多載,也支援條件分支 `SwitchCase`。核心 `DIPipe<TIn,TOut>` 與用法見 [core/pipe.md](core/pipe.md)。

### 3. 多租戶與上下級組織授權

資料隔離透過 `IDataFilterPipe` 管道機制(非 EF Core `HasQueryFilter`):

- 內建按 `OrgId` 自動過濾。
- `AllowChildOrganizationData(roleName)` — 一行宣告「擁有此 role 的使用者可看子組織資料」。

過濾 / 蓋 `OrgId` 的**機制**(查詢一律過 filter pipe、寫入一律跑 Pre/Post、`ITable<T>` 執行載體)見 [core/data-pipes.md](core/data-pipes.md)。**組織樹模型**(單一父組織,根為預設組織)、三層歸屬(`Organization` / `PlatformGroup` / `PlatformUser`,由 `Hcs.PlatformModule.Basic` 提供)、以及**跨組織存取「讀能上下、寫只能往下」的不對稱語意**見 [core/multi-tenant.md](core/multi-tenant.md)。

### 4. OData 查詢

啟動時掃描 query entity 自動建 `EdmModel`。每個 entity 支援 `$filter / $select / $orderby / $expand`,但 expand 必須透過 `AddOdataPermission<T>(b => b.AllowExpand(...))` 白名單(預設拒絕,防關聯資料外洩)。

### 5. 驗證與授權

- **登入與 Token**:多種登入模式(帳密 + 組織、`OrgKey` 組織連結、代理登入、IP 白名單)、JWT 發放、改密碼 / 改資料即時撤銷既發 token(見 [core/login.md](core/login.md))。
- **角色即時注入**:token 驗證時把使用者 role code 注入 claim——改權限不必重發 token。
- **權限樹**:`AddModuleFuncion` 宣告功能權限,五支 entity API 自動掛 View / Create / Modify / Delete,前後端靠權限字串對齊(見 [core/permissions.md](core/permissions.md))。
- **強制失效**:比對 token `notBefore` 與 `UserDataChangeTime`,可單方面標記舊 token 過期(見 [core/login.md](core/login.md))。
- **後端驗證錯誤**:`OnValidate` 收集錯誤 → 400 → 前端 `hcs-error-summary` 顯示(見 [core/validation-errors.md](core/validation-errors.md))。

### 6. ASP.NET + Angular SPA 一體

`UseHcsPlatform` 自動把所有非 `/api/*` 的請求 fallback 到 Angular SPA,靜態檔從 `wwwroot/<spaPath>/`,並支援同站多 SPA 部署。

### 7. 其他基礎建設

- 加密 payload middleware(前 4 byte 為 key,前端可選擇加密)。
- 檔案上傳 / 儲存:抽象 `IFileStorage`,內建本機磁碟與 **Azure Blob** 兩種後端(`UseLocalFileStroage` / `UseAzureBlobStorage`),含確認 / 孤兒清理生命週期(見 [core/file-upload.md](core/file-upload.md))。
- `X-HCS-Server-Ts` response header — 統一伺服器時鐘給前端校時。
- 分散式鎖(`Hcs.AtomLock.Generic`,SQL Server / MySQL / Redis)、快取過期時只讓單一請求重建以免一窩蜂打後端(cache stampede,`GetOrCreateAtomicAsync`)、一次性任務 idempotent 標記(`PlatformFlag`)。

---

## 模組開發

```
┌────────────────────────────────────────────────────────────────┐
│  你的模組（IPlatformModule.Build）                              │
│                                                                 │
│  AddModel<Customer> ─────────────► EF Core entity 註冊          │
│                                                                 │
│  AddEntityApi<long, Customer>                                  │
│    ├── ConfigPostApi(.OnValidate.OnCreated)  ─┐                │
│    ├── ConfigPutApi (.OnValidate.OnUpdated)   │                │
│    ├── ConfigDeleteApi(.OnDeleted)            ├─► GenericController
│    ├── ConfigGetApi  (.OnGeted)               │   執行期產生
│    └── ConfigQueryApi(.OnQueryed)           ─┘                │
│            ▲                                                    │
│            └── Pipe(lambda) ◄── DI 自動注入任意服務            │
│                                                                 │
│  ConfigDataPipe<Customer>(.AddFilter)  ───► 全域 query filter   │
│  SetupFilePipes(x => x.Photo)           ──► 檔案欄位            │
│  AllowChildOrganizationData(roleName)   ──► 多租戶授權          │
│                                                                 │
│  AddModuleFuncion("Module.Customer")                           │
│    ├── AddStandardApiRoles(api, ...)    ──► View/Modify/Delete │
│    │       └── .AddOdataPermission<T>(.AllowExpand(...))       │
│    └── AddPermission("CustomPerm")                             │
└────────────────────────────────────────────────────────────────┘
```

設計原則:

1. 不寫 Controller——用 `AddEntityApi` / `AddXxxFlowApi`。
2. 不繼承擴充行為——用 `Pipe(...)` 插管子,參數會自動 DI。
3. 不寫權限檢查——用 `AddModuleFuncion` 宣告權限,OData expand 走白名單。

---

## 前端

**前端不會由宣告自動生成**——這點與後端不同。平台提供框架、可重用元件(`hcs-data-grid`、`hcs-list-page`、`hcs-form-page`、各式 `hcs-*-input`、`hcs-pager`…)、基底類別(`BaseListComponent` / `BaseFormComponent`)與資料來源(`DataSourceFactory` / `OdataDataSource`);使用端按功能組裝。

每個 CRUD 功能,使用端寫一組精簡的 **list + form** 元件:

- **list**:`extends BaseListComponent<T>`,provide 功能名稱 / route,`super(service, T)`,用 reactive form 定義查詢欄位;template 以 `<hcs-data-grid>` + `*hcsDataGridColum` 指令排版欄位、`hcs-*-input` 排查詢條件。
- **form**:`extends BaseFormComponent<T>`,`super(service, T)`,用 reactive form(含 `Validators`)定義編輯欄位;template 排表單。
- **route**:在 Angular `Route[]` 註冊 list / `new` / `:id` / `:id/edit` 對應元件。

**資料抓取、分頁、查詢條件轉 OData、存檔、驗證錯誤顯示**由基底類別與 `DataSourceFactory.getDataSource(Entity)` 處理——**使用端不需自己寫 API 呼叫**,只要把 datasource 指向 entity model(對應後端 `/api/odata/<Entity>`)。

完整前端 SDK 說明(library 清單、7 個 SDK 慣例、`@hcs/core` 內容、新增功能步驟)見 [frontend.md](frontend.md)。各能力的深入文件:[表單](frontend/form.md)、[列表 / 資料表格](frontend/list.md)、[輸入元件](frontend/controls.md)、[外殼 / 版面](frontend/shell.md)、[登入(前端)](frontend/login.md)。

---

## 專案地圖

`Doc` 欄連到有 doc 涵蓋的 project(來源是各 doc 的維護端清單);`—` 代表尚無對應 doc。`Hcs.Platform.Core` 與前端 `core` 是 kitchen-sink 專案,只被 Core 基礎層各篇切片涵蓋一部分。

> 目前已寫的深入文件僅 entity-api、Pipe、Data Pipes、驗證錯誤、i18n、2FA 等少數幾篇,**陸續補齊中、不代表平台功能範圍**——平台完整能力見[平台能力](#平台能力)。

### 核心

| 專案 | 角色 | Doc |
|---|---|---|
| `Hcs.Platform.Abstractions` | Platform 對外公開介面(權限/角色契約、`IPlatformUser` 等) | [validation](core/validation-errors.md) |
| `Hcs.Platform.BaseModels` | 核心 entity(`PlatformUser` / `PlatformGroup` / `Organization` / `PlatformFlag`…) | [multi-tenant](core/multi-tenant.md) |
| `Hcs.Platform.Core` | 平台主體:模組組裝、Generic Controller、Pipe builders、JWT、OData EdmModel、多租戶過濾 | [entity-api](core/entity-api.md)、[pipe](core/pipe.md)、[data-pipes](core/data-pipes.md)、[permissions](core/permissions.md)、[login](core/login.md)、[multi-tenant](core/multi-tenant.md)、[file-upload](core/file-upload.md)、[validation](core/validation-errors.md)、[i18n](core/i18n-system.md) |
| `Hcs.Platform.Data` | 資料層共用契約(`ITable<T>`、`IScopedDbContext`、查詢 context 等) | [data-pipes](core/data-pipes.md) |
| `Hcs.Platform.Flow` | 通用 flow 引擎——被 ApprovalFlow 用,也可自行套用 | [approval-flow](modules/approval-flow.md) |

### Platform Modules(可選,各對應一個 `AddXxxModule()`)

| 模組 | 用途 | Doc |
|---|---|---|
| `Hcs.PlatformModule.Basic` | 核心三 entity(User / Group / Organization)CRUD + 子組織授權 + Proxy Login | [basic](modules/basic.md) |
| `Hcs.PlatformModule.ApprovalFlow` | 簽核流程引擎(流程定義、階段、動作、狀態) | [approval-flow](modules/approval-flow.md) |
| `Hcs.PlatformModule.AppUpdate` | App 版本管理,支援多平台多產品的版本檢查 | — |
| `Hcs.PlatformModule.CodeTable` | 字典/代碼表機制(含 i18n) | [code-table](modules/code-table.md) |
| `Hcs.PlatformModule.SystemLogging` | 稽核軌跡記錄(宣告式 diff / 欄位審查 / reference 解析) | [system-logging](modules/system-logging.md) |
| `Hcs.PlatformModule.TwoFactorAuthentication` | 2FA 框架(可擴充多 provider) | [2fa](modules/2fa.md) |

### Models 專案(獨立出來讓前端 / 第三方可 reference entity 定義而不依賴 server logic)

| 專案 | 用途 | Doc |
|---|---|---|
| `Hcs.Platform.ApprovalFlow.Models` | 簽核流程 entity | [approval-flow](modules/approval-flow.md) |
| `Hcs.Platform.AppUpdate.Models` | App 版本 entity | — |
| `Hcs.Platform.CodeTable.Models` | 代碼表 entity(含 i18n) | [code-table](modules/code-table.md) |
| `Hcs.Platform.SystemLogging.Models` | 稽核日誌 entity | [system-logging](modules/system-logging.md) |
| `Hcs.Platform.TwoFactorAuthentication.Models` | 2FA entity | [2fa](modules/2fa.md) |

### 基礎建設

| 專案 | 用途 | Doc |
|---|---|---|
| `Hcs.AtomLock.Generic` | 分散式原子鎖(SQL Server / MySQL / Redis) | — |
| `Hcs.Encryption` | AES / MD5 / SHA 等加密/雜湊 | — |
| `Hcs.Expressions` | Expression Tree 與屬性訪問器工具 | — |
| `Hcs.Pipe` | Pipe 責任鏈核心——`DIPipe<TIn,TOut>`,整個平台擴充模型的基石 | [pipe](core/pipe.md) |
| `Hcs.Pipes.Ckeditor` | CKEditor 上傳檔案的確認與孤兒檔案清理 pipe | — |
| `Hcs.Console` | .NET Console host 框架(DI + NLog + 命令列) | — |
| `Hcs.Serialize.Xlsx` | 以 ClosedXML 為底的 Excel 動態序列化 | — |
| `Hcs.ThirdPartyLogin` / `.Abstraction` | 第三方登入框架 + 綁定管理 | — |
| `Hcs.ThirdPartyLogin.Google` / `.Facebook` | OAuth 實作 | — |
| `Hcs.2FA.GoogleAuthenticator` | Google Authenticator (TOTP) | [2fa](modules/2fa.md) |
| `Hcs.Platform.IpWhiteListOnlyLogin` | 限制特定 IP 才能登入 | [login](core/login.md) |

### Extensions 套件

| 專案 | 用途 | Doc |
|---|---|---|
| `Hcs.Extensions.ApprovalFlow` | 簽核流程領域實作(搭配 `Hcs.Platform.Flow`) | [approval-flow](modules/approval-flow.md) |
| `Hcs.Extensions.Collections` | 字典擴充(`AddIfNotExists` / `AddOrUpdate`) | — |
| `Hcs.Extensions.DependencyInjection` | 命名服務工廠(按名稱解析多實作) | — |
| `Hcs.Extensions.EntityFrameworkCore` | DbSet / Queryable 擴充(主鍵篩選等) | — |
| `Hcs.Extensions.MemoryCache` | `GetOrCreateAtomicAsync` 防快取驚群 | — |
| `Hcs.Extensions.Message` / `.Email` / `.Android` / `.Ios` / `.Mitake` | 多通道發訊抽象 + SMTP / FCM / APNs / 三竹簡訊實作 | — |
| `Hcs.Extensions.OdataClient` | OData 查詢 client(LINQ → OData URL) | — |
| `Hcs.Extensions.RequestData` | HTTP request 資料字典 | — |
| `Hcs.Extensions.SystemLogging` | 系統日誌服務(追蹤資料變更 / 商業行為) | [system-logging](modules/system-logging.md) |
| `Hcs.Extensions.Type` | 反射工具(友善泛型類型名稱) | — |
| `Hcs.Extensions.Validation` | 實體驗證結果擴充(`AddErrorFor`) | [validation](core/validation-errors.md) |
| `Hcs.Extensions.Xlsx` | Excel 序列化格式化擴充(日期/數值),搭配 `Hcs.Serialize.Xlsx` | — |

### 前端(npm `@hcs/*` 套件)

Angular 17 SPA + ng library 套件。

| 套件 | 角色 | Doc |
|---|---|---|
| `@hcs/core` | 平台核心 lib(`HcsComponentsModule`、`DataGridComponent`、`OdataDataSource`、`ErrorHelper`、i18n loader…) | [validation](core/validation-errors.md)、[i18n](core/i18n-system.md) |
| `@hcs/basic` | PlatformUser / PlatformGroup / Organization 的 list 與 form(對應後端 `AddBasicModule`) | [basic](modules/basic.md) |
| `@hcs/app-update` | App 版本管理 UI | — |
| `@hcs/approval-flow` | 簽核流程階段式編輯器 + 流程圖視覺化 | [approval-flow](modules/approval-flow.md) |
| `@hcs/code-table` | 字典/代碼表 UI | [code-table](modules/code-table.md) |
| `@hcs/system-logging` | 稽核軌跡 UI | [system-logging](modules/system-logging.md) |
| `@hcs/third-party-login` | 第三方登入 UI | — |
| `@hcs/two-factor-authentication` | 2FA UI | [2fa](modules/2fa.md) |
| `@hcs/two-factor-authentication-google` | Google Authenticator UI | [2fa](modules/2fa.md) |
| `create-hcs-app` | 專案產生器 schematics(`npm create hcs-app`) | — |

> 另有 submodule `external/OdataQueryLite`(新版 OData 查詢引擎,獨立 repo),不在本表。

---

## 開發者快速上手

### 加一個業務功能(後端)

1. 建模組專案 `Hcs.PlatformModule.<YourFeature>`,加一個 class 實作 `IPlatformModule`。
2. `Build` 裡描述業務:

   ```csharp
   public void Build(IPlatformModuleBuilder b)
   {
       b.AddModel<MyFeature.ModelConfig>();
       var api = b.AddEntityApi<long, Invoice>(opt =>
       {
           opt.AllowChildOrganizationData("MyFeature.Invoice.ChildOrgData");
           opt.ConfigPostApi(x => x.OnValidate(v => v.Unique(u =>
               u.AddProperty(p => p.No).AddProperty(p => p.OrgId))));
       });
       b.AddModuleFuncion("MyFeature", "Invoice",
           o => o.AddStandardApiRoles(api));
   }
   ```

3. 註冊到 host:在 `AddHcsPlatform(...)` 裡加 `builder.AddModule<MyFeature.ModuleConfig>();`。
