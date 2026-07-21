---
name: platform-add-entity
description: 往一個既有的 HCS Platform 模組加一顆 entity——後端 entity class + ModelConfig 一行 + AddEntityApi/AddModuleFuncion，前端 TS model(@ApiEntry) + 列表/表單頁 + 註冊到模組的路由與選單。可重複，每顆 entity 跑一輪。
---

# 往模組加一顆 Entity（後端 CRUD + 前端頁）

## When to use

往一個**既有模組**加一顆 entity，從零到「列表能查、表單能增刪改」。**前置**：app 見 [platform-create-project](platform-create-project.md)；module（容器）見 [platform-create-module](platform-create-module.md)。本篇假設模組 `Sample` 已存在（後端 `SampleModuleConfig` / `SampleModelConfig`，前端 `SampleModule` / `SampleMenu` / `sample.routes.ts`）。

> **快速路徑**：標準形狀（本篇後端 1–3 + 前端 1–4 + migration）可以用 `npx @hcs/create-hcs-entity <定義檔.json>` 一次生成——欄位定義寫一份 JSON，工具產檔、插註冊、跑 migration，收尾只剩重啟 + `--updaterole` + 重新登入。客製（hook、`$expand` 白名單、自訂端點）仍照本篇手動加。

以一顆 `Invoice`（entity 全名 `Sample.Models.Invoice`、功能碼 `Sample.Invoice`）為例。每加一顆 entity 重跑本流程，把名字換掉。

---

## 後端

### 1. entity class

繼承 base 型別拿到 `Id` + 6 個 audit 欄（不用手寫）：

```csharp
using Hcs.Platform.BaseModels;
using System.ComponentModel.DataAnnotations;

namespace Sample.Models;

public class Invoice : BaseModel              // = BaseModel<long>，已實作 IPlatformEntity（audit 自動）
{
    [StringLength(50)]  public string No { get; set; } = "";
    [StringLength(200)] public string Title { get; set; } = "";
    public decimal Amount { get; set; }
}
```

**要多租戶隔離**改繼承 `BaseOrganizedModel` **並自己加 `, IOrganized`**——`BaseOrganizedModel` 只給 `OrgId` 屬性、本身沒實作 `IOrganized`，少了那個介面就**有 OrgId 欄位卻不會被租戶過濾**。語意見 [core/multi-tenant](../core/multi-tenant.md)。

### 2. 往模組的 ModelConfig 加一行

在既有 `SampleModelConfig.BuildModel` 裡加這顆 entity 的對應：

```csharp
modelBuilder.Entity<Invoice>(e =>
{
    e.SetupBaseModel();                       // 組織版用 SetupBaseOrganizedModel
    e.Property(x => x.Amount).HasPrecision(18, 2);   // decimal 指定精度，否則 EF 警告
});
```

### 3. 往模組的 Build 加 API + 權限

在既有 `SampleModuleConfig.Build` 裡加這顆 entity 的端點與權限：

```csharp
var api = moduleBuilder.AddEntityApi<long, Invoice>(opt =>
{
    // 生命週期 hook 掛這裡，例：唯一性驗證
    // opt.ConfigPostApi(x => x.OnValidate(v => v.Unique(u => u.AddProperty(p => p.No))));
});

moduleBuilder.AddModuleFuncion("Sample", "Invoice",       // 拼字就是 Funcion（少一個 t，既定 API 名）
    f => f.AddStandardApiRoles(api));                     // 自動產 View/Create/Modify/Delete 四權限並綁五端點
```

`AddEntityApi<long, Invoice>` 一行就長出 Get/Query/Post/Put/Delete 五個端點（OData 查詢、CRUD 管線、交易範圍全內建）。

### 4. DB：補 migration

**既有 DB 不會自動長出新表**（平台啟動只在資料庫不存在時建全套），所以每加一顆 entity 都要（先停掉運行中的 host）：

```bash
dotnet ef migrations add Add<Entity>
dotnet ef database update
```

migration 前置儀式與 baseline 陷阱見 [platform-create-project](platform-create-project.md) 的「DB schema 演進」節。

> 收尾三步：重啟 host → 從 **localhost** 打一次 `GET /api/console/updaterole` 把新權限灌給既有 admin 群組（否則登入後看不到這顆 entity 的頁/按鈕）→ 前端**重新登入**（權限是登入時載的，不重登選單不會出現）。

**深度**：管線/hook/交易語意 → [core/entity-api](../core/entity-api.md)；Pipe lambda 寫法 → [core/pipe](../core/pipe.md)；權限樹 → [core/permissions](../core/permissions.md)。

---

## 前端

前端是 TypeScript、沒有 C# 的 entity class，所以**手寫一個 TS model 宣告它對應哪個後端 entity**——datasource 靠它拼 URL。

### 1. TS model（`@ApiEntry` 綁後端全名）

```typescript
import { ApiEntry, Key, Property } from '@hcs/core/models';

@ApiEntry('api/entity/Sample.Models.Invoice')   // = 後端 entity 的 C# 全名 namespace.Type
export class Invoice {
  @Key() Id: number;                             // 識別欄
  @Property() No: string;
  @Property() Title: string;
  @Property() Amount: number;
}
```

### 2. 列表頁（`extends BaseListComponent`）

`invoice-list.component.ts`：

```typescript
import { Component } from '@angular/core';
import { FormGroup, FormControl } from '@angular/forms';
import { BaseComponentService, BaseListComponent } from '@hcs/core/hcs-components';
import { HCS_FUNCTION_NAME, HCS_FUNCTION_ROUTE } from '@hcs/core/hcs-lib';
import { Invoice } from '../invoice.model';

@Component({
  selector: 'sample-invoice-list',
  templateUrl: './invoice-list.component.html',
  providers: [
    { provide: HCS_FUNCTION_NAME, useValue: 'Sample.Invoice' },   // = 後端 AddModuleFuncion 碼
    { provide: HCS_FUNCTION_ROUTE, useValue: 'sample/invoice' },  // = 本頁 route path
    BaseComponentService,                                          // 一定要 provide（漏了 NullInjectorError）
  ],
})
export class InvoiceListComponent extends BaseListComponent<Invoice> {
  constructor(service: BaseComponentService) {
    super(service, Invoice);                                       // 傳 model 型別，基底自動建 datasource
    this.filterForm = new FormGroup({ No: new FormControl(), Title: new FormControl() });
  }
}
```

`invoice-list.component.html`：

```html
<hcs-list-page>
  <hcs-data-grid [data]="data" #grid [autoLoad]="true">
    <ng-container grid-head [formGroup]="filterForm">
      <hcs-textbox-input formControlName="No" hcsDataGridQuery operator="contains"><label>單號</label></hcs-textbox-input>
      <hcs-textbox-input formControlName="Title" hcsDataGridQuery operator="contains"><label>標題</label></hcs-textbox-input>
      <div class="buttons">
        <hcs-default-button-search-bar [grid]="grid" [filterForm]="filterForm"></hcs-default-button-search-bar>
      </div>
    </ng-container>

    <ng-container *hcsDataGridColum="let value;let entity=entity;field:'No';export:true,name:'單號'">
      <a [routerLink]="[entity.Id]" queryParamsHandling="merge">{{ value }}</a>
    </ng-container>
    <ng-container *hcsDataGridColum="let value;field:'Title';export:true,name:'標題'">{{ value }}</ng-container>
    <ng-container *hcsDataGridColum="let value;align:'right';field:'Amount';export:true,name:'金額'">{{ value }}</ng-container>
    <ng-container *hcsDataGridColum="let value;width:125;field:'Id';name:'';sortable:false">
      <hcs-default-button-list [data]="data" [key]="value"></hcs-default-button-list>
    </ng-container>
    <hcs-pager grid-foot></hcs-pager>
  </hcs-data-grid>
</hcs-list-page>
```

### 3. 表單頁（`extends BaseFormComponent`）

`invoice-form.component.ts`：

```typescript
import { Component } from '@angular/core';
import { FormGroup, FormControl, Validators } from '@angular/forms';
import { BaseFormComponent, BaseComponentService } from '@hcs/core/hcs-components';
import { HCS_FUNCTION_NAME, HCS_FUNCTION_ROUTE } from '@hcs/core/hcs-lib';
import { Invoice } from '../invoice.model';

@Component({
  selector: 'sample-invoice-form',
  templateUrl: './invoice-form.component.html',
  providers: [
    { provide: HCS_FUNCTION_NAME, useValue: 'Sample.Invoice' },
    { provide: HCS_FUNCTION_ROUTE, useValue: 'sample/invoice' },
    BaseComponentService,
  ],
})
export class InvoiceFormComponent extends BaseFormComponent<Invoice> {
  constructor(baseService: BaseComponentService) {
    super(baseService, Invoice);
    this.formGroup = new FormGroup({
      No: new FormControl(null, [Validators.required]),
      Title: new FormControl(null, [Validators.required]),
      Amount: new FormControl(null, [Validators.required]),
    });
  }
}
```

`invoice-form.component.html`：

```html
<hcs-form-page [formGroup]="formGroup" #fp [dataSource]="datasource">
  <hcs-form-row>
    <hcs-textbox-input formControlName="No" required><label>單號</label></hcs-textbox-input>
    <hcs-textbox-input formControlName="Title" required><label>標題</label></hcs-textbox-input>
    <hcs-textbox-input type="number" formControlName="Amount" required><label>金額</label></hcs-textbox-input>
  </hcs-form-row>
</hcs-form-page>
```

### 4. 註冊進既有模組

往模組那三個既有檔各加一條這顆 entity 的登錄（檔本身在 [platform-create-module](platform-create-module.md) 建）：

- **`sample.module.ts`** 的 `declarations` 加 `InvoiceListComponent, InvoiceFormComponent`。
- **`sample.routes.ts`** 加這顆的 5 條 route：

  ```typescript
  { path: 'invoice', component: InvoiceListComponent },
  { path: 'invoice/new', component: InvoiceFormComponent },
  { path: 'invoice/new/:copy', component: InvoiceFormComponent },
  { path: 'invoice/:id', component: InvoiceFormComponent, data: { readonly: true } },
  { path: 'invoice/:id/edit', component: InvoiceFormComponent },
  ```

- **`Menu.ts`** 的 `Sample` children 加一個 MenuItem（含權限 gate）：

  ```typescript
  new MenuItem('Invoice', null, ['/', 'sample', 'invoice'], null, null,
    () => this.permission.hasPermission('Sample.Invoice.View')),
  ```

**深度**：data-grid 全功能 → [frontend/list](../frontend/list.md)；表單生命週期/進階欄位 → [frontend/form](../frontend/form.md)。

---

## 常見客製（純 CRUD 之外）

到這裡列表 / 表單已能增刪改查。實務上幾乎每顆 entity 還會用到下面幾項——都不是「另一套機制」，而是往 `AddEntityApi(opt => …)` 那個 `opt` 上掛 hook，或在權限上補白名單。**先在這裡認得它存在、抓對掛點**，深寫照各自的 route。

### 後端：寫入前驗證（查重等）

DB 才驗得了的規則（唯一性、跨表關聯、業務規則）掛 `OnValidate`；純格式（必填 / 長度）留前端 reactive form。唯一性有內建一行掛法（自動掛 Post+Put）：

```csharp
api = moduleBuilder.AddEntityApi<long, Invoice>(opt =>
{
    opt.UniqueValidation(u => u.AddProperty(x => x.No));     // No 不可重複；複合鍵就連續 AddProperty
});
```

刪除前擋關聯用 `opt.ConfigDeleteApi(d => d.OnValidate(v => v.CheckAllRefForDelete()))`。錯誤怎麼產生、i18n、自訂 validator → [core/validation-errors](../core/validation-errors.md)。

### 後端：生命週期副作用（通知 / 稽核）

新增 / 更新後做後續處理掛 `On{X}ed`；**一定要跑（含驗證失敗也跑）的橫切工作（log / 稽核）掛 `OnAfterTransaction`**：

```csharp
opt.ConfigPostApi(x => x.OnCreated(c => c.Pipe((INotifier n, Invoice e) => n.Notify(e))));
```

掛點全表（`OnCreating/OnCreated/OnUpdated/OnQueryed…`）、交易邊界、`UseDefaultSave(false)` → [core/entity-api](../core/entity-api.md)；pipe 怎麼寫 + DI 注入 → [core/pipe](../core/pipe.md)。

### 後端：列表要帶關聯欄位（$expand 白名單）

`$expand` 預設**全擋**。列表要顯示建立者、或某導覽屬性的欄位，前端 OData 要 `$expand=…`，後端得在權限上開白名單：

```csharp
moduleBuilder.AddModuleFuncion("Sample", "Invoice", f =>
    f.AddStandardApiRoles(api, ctx => ctx.View
        .AddOdataPermission<Invoice>(o => o.AllowExpand(x => x.CreatedByUser))));
```

巢狀 / 嚴格前綴語意 → [core/permissions](../core/permissions.md)。

### 不是標準增刪改查的動作 → 自訂端點

結帳、簽核、排序、批次、回傳算出來的視圖——這些不該硬塞進 CRUD，用 **Flow API**（`AddPostFlowApi` / `AddGetFlowApi` / …）自組一條管線、單獨綁權限、前端用 `HttpClient` 打 `api/entity/<name>`。整套見 skill [platform-custom-api](platform-custom-api.md)、深 doc [core/flow-api](../core/flow-api.md)。

---

## 對齊契約（最常出錯的地方）

三條字串必須對上，錯一個就「按鈕消失 / 403 / 查不到」：

| 前端 | 必須等於 | 後端 |
|---|---|---|
| `@ApiEntry('api/entity/Sample.Models.Invoice')` | entity 的 C# 全名 | `namespace Sample.Models; class Invoice` |
| `HCS_FUNCTION_NAME: 'Sample.Invoice'` | 功能碼 | `AddModuleFuncion("Sample", "Invoice")` |
| `HCS_FUNCTION_ROUTE: 'sample/invoice'` | 本頁 route path | 模組 lazy 路徑 `sample` + `sample.routes.ts` 的 `invoice` |

權限字串對齊細節（大小寫逐字、選單寫死全字串）見 [core/permissions](../core/permissions.md)。

---

## 禁忌

- ❌ **後端不要寫 Controller / 繼承 ControllerBase / 寫 DTO + AutoMapper**——控制器執行期動態生成。
- ❌ **繼承 `BaseOrganizedModel` 就以為有租戶隔離**——少了 `, IOrganized` 介面，有 `OrgId` 欄位也不會被過濾。
- ❌ **前端不要手刻 / codegen entity 的 raw API URL**——`@ApiEntry` + SDK datasource 會自己拼（含 `$select`/`$filter`/auth/密碼 hash/加密），consumer 一律走 SDK 元件、不直接打 raw API。
- ❌ **加了 entity 忘了 migration**——既有 DB 不會自動長新表，該 entity 一查就 500 `Invalid object name '<Entity>'`。
- ❌ **加了功能碼忘了 `updaterole` 或忘了重新登入**——admin 群組拿不到新權限（或舊 session 還載著舊權限清單），頁面/按鈕會被權限 gate 藏起來。

---

## Canonical reference

- 後端：[core/entity-api](../core/entity-api.md) · [core/pipe](../core/pipe.md) · [core/permissions](../core/permissions.md) · [core/multi-tenant](../core/multi-tenant.md) · [core/data-pipes](../core/data-pipes.md) · [core/validation-errors](../core/validation-errors.md)
- 前端：[frontend/list](../frontend/list.md) · [frontend/form](../frontend/form.md) · [frontend/controls](../frontend/controls.md)
- 模組容器與 app → [platform-create-module](platform-create-module.md) · [platform-create-project](platform-create-project.md)
