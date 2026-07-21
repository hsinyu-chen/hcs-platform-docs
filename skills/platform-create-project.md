---
name: platform-create-project
description: 從零建一個 HCS Platform app——後端 host（吃 Hcs.* 套件、minimal hosting 接 AddHcsPlatform）＋前端 app（create-hcs-app）＋建第一個 admin ＋ dev loop。當要 scaffold / bootstrap 一個新的 HCS Platform 專案時使用。
---

# 建立一個新 HCS Platform 專案

## When to use

要從零起一個新的 HCS Platform 應用：一個後端 host（提供 API）＋一個前端 SPA。建好之後，加業務功能見 [platform-add-entity](platform-add-entity.md)。

## 總覽

```
後端 host  手寫一個 minimal-hosting 專案，吃 Hcs.* nuget，AddHcsPlatform 接 DB/JWT/Basic
前端 app   npx @hcs/create-hcs-app（已含登入、shell、Basic 管理頁）
第一個 admin   後端跑起來後，從 localhost 打內建 console 端點建 admin
dev loop   後端 run + 前端 ng serve（proxy /api → 後端）→ 登入
```

平台把 Controller / Repository / DTO / 權限中介 / 多租戶過濾收進框架，**host 只負責組裝**（接 DB、接 JWT、裝模組）。

---

## 1. 後端 host

### feed 設定

`nuget.config`（Hcs.* 在私有 feed、其餘走 nuget.org；用 packageSourceMapping 分流）：

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" protocolVersion="3" />
    <add key="hcs" value="<你的 Hcs.* nuget feed v3 index>" />
  </packageSources>
  <packageSourceMapping>
    <packageSource key="nuget.org"><package pattern="*" /></packageSource>
    <packageSource key="hcs"><package pattern="Hcs.*" /></packageSource>
  </packageSourceMapping>
</configuration>
```

私有 feed 的認證走 NuGet 既有機制（如 Azure Artifacts credential provider）；本機通常已快取，`dotnet restore` 不需互動。

### 專案

`<Host>.csproj`（SDK `Microsoft.NET.Sdk.Web`、目標 net10）：

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Hcs.Platform.Core" Version="1005.4.*" />
    <PackageReference Include="Hcs.Platform.BaseModels" Version="1005.4.*" />
    <PackageReference Include="Hcs.PlatformModule.Basic" Version="1005.4.*" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="10.0.*" />
  </ItemGroup>
</Project>
```

`Program.cs`（minimal hosting）：

```csharp
using Hcs.Platform;
using Hcs.PlatformModule.Basic;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddHcsPlatform(b =>
{
    b.UseLocalFileStroage(@"C:\temp\app-files");            // 注意拼字 Stroage（既定 API 名）；正式環境可改 Azure Blob / SMB
    b.ConfigDbOptions(opt => opt.UseSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection"),
        sql => sql.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery)));
    b.ConfigJwtOptions(jwt => { jwt.Issuer = "your.app"; jwt.IssuerSigningKey = "<change-me-long-secret>"; });
    b.AddBasicModule();                                     // User / Group / Organization 三層 + 登入所需
    // b.AddModule<YourFirstModuleConfig>();                // 之後加業務模組（見 platform-add-entity）
});

var app = builder.Build();
app.UseHcsPlatform(opt => { });
app.Run();
```

`appsettings.json`：

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\MSSQLLocalDB;Database=YourApp;Trusted_Connection=True;MultipleActiveResultSets=true;TrustServerCertificate=true"
  }
}
```

### ⚠️ 建一個 `wwwroot/` 資料夾

`UseHcsPlatform` 啟動時在 `wwwroot` 掛靜態檔 provider；**沒有這資料夾 app 一啟動就 `DirectoryNotFoundException` 崩**。`dotnet new web` 不帶它——手動建一個（裡面可空，放個 placeholder `index.html` 即可）。production 時前端 build 的產物會放進這裡。

> ⚠️ **DB schema 只在「資料庫不存在」時自動建**：平台啟動時**動態組裝** model（base models + 你 `AddModel` 的對應），若資料庫不存在就 `CREATE DATABASE` + 建所有表 + seed（`EnsureCreated` 語意）。所以**第一次跑起來不用寫 migration**——ConnectionString 指對一個 SQL 實例即可，看到 log「Platform initialized / System Ready」即成功。但**資料庫已存在時，啟動不做任何 schema 動作——之後新增的 entity 不會自動長出資料表**（症狀：該 entity 的查詢 500 `Invalid object name '<Entity>'`）。schema 要演進（含 dev 期加 entity）見下節。

### DB schema 演進與上 prod

DB 建出來之後，每一次 schema 變更（新 entity、加欄位）都要走下面其中一條路：

**正常路線：EF migration（推薦）。** 平台的 `DbContext` 在套件裡、model 啟動時動態組裝，但 `dotnet ef` 能透過 host 的 DI 找到它、design-time 建出完整 model——**不需另立 DbContext**。儀式兩樣：

1. host 加 `Microsoft.EntityFrameworkCore.Design` 套件。
2. `ConfigDbOptions` 的 `UseSqlServer(conn, b => b.MigrationsAssembly("<YourHost>"))` 加 `.MigrationsAssembly(...)`（`DbContext` 在套件、migration 檔要落進你的專案）。

然後每次 schema 變更：

- `dotnet ef migrations add <Name>` — 產 migration（捕捉 base + 你的 entity 完整 schema）。
- `dotnet ef database update` — 套用到 dev DB。
- `dotnet ef migrations script <from> <to>` — 產版本化 SQL diff 上 prod。

**⚠️ EF migrate 注意事項：**

- **先停掉運行中的 host 再跑 `dotnet ef`**——`dotnet ef` 會先 build，運行中的 host 鎖住組建輸出會直接 build 失敗。
- **baseline 陷阱**：若 dev DB 是平台自動建的（先有 DB、後補 migration），`__EFMigrationsHistory` 裡**沒有** Init migration 的紀錄，`database update` 會想從 Init 整套重跑而撞表（`There is already an object named 'Organization'`）。解法是把 Init 手動補記進 history 當 baseline，之後 update 就只套新的 migration：

  ```sql
  INSERT INTO __EFMigrationsHistory (MigrationId, ProductVersion)
  VALUES ('<InitMigrationId>', '<EF 版本>');   -- MigrationId = Migrations/ 底下 Init 檔名（不含 .cs）
  ```

- 加了 entity 之後記得整套收尾：套完 migration → 重啟 host → localhost 打 `GET /api/console/updaterole` 灌權限 → 前端重新登入。

**legacy 路線：砍掉重建 + schema diff 工具。** 舊流程：改 schema 時砍掉 dev DB 讓平台重建，再用 SQL schema-compare 工具（如 SSDT）對 prod 產 diff。能用，但砍掉重建不方便、diff 非版本化、dev 資料全失。新專案建議走 migration。

---

## 2. 前端 app

```
npx @hcs/create-hcs-app <app-name>
```

先在 `~/.npmrc` 設 **scope-only** 映射（**只**把 `@hcs` 指到私有 feed，default 留 npmjs）：

```
@hcs:registry=<你的 npm feed>
```

> 不要把 default registry 整個指到私有 feed——那會讓公開套件走私有 feed 的 upstream proxy，踩到某些 npm 版本的 manifest normalize bug（install 噴 `paths[0] ... undefined`）。scope-only 讓公開套件直連 npmjs、繞開。

scaffold 出來的 app **開箱即含登入頁、shell、Basic 的 User/Group/Org 管理頁**。把 `proxy-config.json` 的 `/api` target 指到你後端 host 的 port。

---

## 3. 建第一個 admin

沒有任何使用者之前無法從 UI 建帳號，所以用**內建 console 端點**（`IsLocalRequest` 鎖 localhost）：

1. 後端跑起來。
2. 從 **localhost** 打：`GET http://localhost:<port>/api/console/newadmin/<account>/<password>`
   → 建一個 admin user（在預設組織）＋一個握有全部權限的 "Administrator" 群組，回 `create admin success`。
3. **之後每次加新模組/功能**（新的 `AddModuleFuncion`），再打一次 `GET /api/console/updaterole`，把新權限灌進既有 admin 群組。

> 登入時前端登入元件會自己處理：組織選擇（OrgKey）、密碼 client 端 hash。你不需手動弄這些——直接用上面的 account / password 在登入頁登入即可。登入/Token 機制見 [core/login](../core/login.md)。

---

## 4. Dev loop

- **後端**：`dotnet run`（聽某 port，如 5080）。
- **前端**：`ng serve`（自己的 dev server，預設 4200），`proxy-config.json` 把 `/api` 轉到後端 port。瀏覽器開前端、登入。
- create-hcs-app 的 `npm start` 預設帶 `--ssl`（https dev server + 自簽憑證）。若工具鏈不吃自簽，可改用不帶 `--ssl` 的 `ng serve` 走 http。
- **production**：前端 `ng build` 把產物輸出到後端 `wwwroot`，由後端 host 一起伺服。**改了 `wwwroot` 內容要重啟後端**才生效（靜態 provider 啟動時綁定）。

---

## 接下來

- 加第一顆業務 entity（後端 CRUD + 前端頁）→ [platform-add-entity](platform-add-entity.md)
- 裝內建模組（簽核 / 稽核 / 字典 / 2FA / 第三方登入 …）→ [modules/](../modules/)
- 平台能力總覽與專案地圖 → [README](../README.md)

## 容易漏的點

- **`wwwroot/` 必須存在**，否則啟動崩。
- **DB schema 啟動時自動同步**（model 動態組裝），不寫 migration 就能跑；schema 演進歷史上靠砍 dev DB 重建 + 對 prod 產 SQL diff。
- **第一個 admin 走 `/api/console/newadmin`（localhost）**，加模組後 `/api/console/updaterole` 補權限。
- **前端 `@hcs:registry` 用 scope-only**，別覆寫 default registry。
- **改 `wwwroot` 要重啟後端**。
