---
name: platform-create-module
description: 在 HCS Platform app 裡建一個新業務模組(entity 的容器)——後端 IPlatformModule + ModelConfig 空骨架並註冊到 host，前端 feature module + provider module + 選單並接進 app。建好後用 platform-add-entity 往裡加 entity。
---

# 建立一個模組（entity 的容器）

## When to use

要在既有 app 裡新增一個業務模組——一組相關 entity 的容器，連同它的選單、路由、註冊。**前置**：app 見 [platform-create-project](platform-create-project.md)。**建好後**往裡加 entity 見 [platform-add-entity](platform-add-entity.md)。

一個「模組」= 後端一個 `IPlatformModule`（宣告 entity 對應 + API + 權限）+ 前端一個 feature module（頁面）+ 一個 provider module（選單）。本篇先把這些**空容器**立好、接上線；entity 之後往裡加。以模組 `Sample` 為例。

---

## 後端：模組骨架

### ModelConfig（這個模組的 EF 對應；entity 之後往 `BuildModel` 加）

```csharp
using Hcs.Platform.Data;        // IModelConfig
using Microsoft.EntityFrameworkCore;

namespace Sample.Models;

public class SampleModelConfig : IModelConfig
{
    public void BuildModel(ModelBuilder modelBuilder)
    {
        // hcs:model-config —— 每顆 entity 的 e.Entity<X>(e => e.SetupBaseModel()) 加在這行上方（create-hcs-entity 的插入 anchor，保留原樣）
    }
    public void BuildSeedData(DbContext context) { }
}
```

### IPlatformModule（模組宣告本體；命名依**模組**、可容多顆 entity）

```csharp
using Hcs.Platform;
using Hcs.Platform.PlatformModule;
using Sample.Models;

namespace Sample;

public class SampleModuleConfig : IPlatformModule
{
    public void Build(IPlatformModuleBuilder moduleBuilder)
    {
        moduleBuilder.AddModel<SampleModelConfig>();   // 整個模組一個 ModelConfig
        // hcs:entity-api —— 每顆 entity 的 AddEntityApi<...> + AddModuleFuncion(...) 加在這行上方（create-hcs-entity 的插入 anchor，保留原樣）
    }
}
```

### 註冊到 host

`Program.cs` 的 `AddHcsPlatform(b => { ... })` 裡加：

```csharp
b.AddModule<SampleModuleConfig>();
```

> 加了新模組（之後也包含新功能碼）後，記得從 **localhost** 打一次 `GET /api/console/updaterole`，把新權限灌給既有 admin 群組。

---

## 前端：feature module + provider module

### routes（空；entity 之後往裡加 5 條）

`sample.routes.ts`：

```typescript
import { Route } from '@angular/router';

export const routes: Route[] = [
  // hcs:routes —— 每顆 entity 的 5 條 route（list / new / new:copy / :id / :id/edit）加在這行上方（create-hcs-entity 的插入 anchor，保留原樣）
];
```

### feature module（宣告頁面元件；`grid/form/inputs` 由 `HcsComponentsModule` 提供）

`sample.module.ts`：

```typescript
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { ReactiveFormsModule } from '@angular/forms';
import { RouterModule } from '@angular/router';
import { HcsComponentsModule } from '@hcs/core/hcs-components';
import { routes } from './sample.routes';

@NgModule({
  declarations: [
    // hcs:declarations —— 每顆 entity 的 list + form 元件加在這行上方（create-hcs-entity 的插入 anchor，保留原樣）
  ],
  imports: [CommonModule, ReactiveFormsModule, HcsComponentsModule, RouterModule.forChild(routes)],
})
export class SampleModule {}
```

### 選單（`extends MenuItemProvider`；entity 之後往 children 加項）

`Menu.ts`：

```typescript
import { Injectable } from '@angular/core';
import { MenuItem, MenuItemProvider } from '@hcs/core/hcs-lib';

@Injectable()
export class SampleMenu extends MenuItemProvider {
  get(): MenuItem[] {
    return [
      // MenuItem(title, icon, route, click, children, visible)；title 就是 i18n key menu.*（走模組語系檔，不硬寫）
      new MenuItem('menu.Sample', 'receipt_long', null, null, [
        // hcs:menu-items —— 每顆 entity 的 MenuItem（含 permission gate）加在這行上方（create-hcs-entity 的插入 anchor，保留原樣）
      ]),
    ];
  }
}
```

### provider module（`forRoot` 註冊 MENU_ITEMS + I18N_INDEX，**不註冊路由**）

`sample-provider.module.ts`：

```typescript
import { ModuleWithProviders, NgModule } from '@angular/core';
import { I18N_INDEX, MENU_ITEMS } from '@hcs/core/hcs-lib';
import { SampleMenu } from './Menu';

@NgModule()
export class SampleProviderModule {
  static forRoot(): ModuleWithProviders<SampleProviderModule> {
    return {
      ngModule: SampleProviderModule,
      providers: [
        { provide: MENU_ITEMS, useClass: SampleMenu, multi: true },
        { provide: I18N_INDEX, useValue: 'sample', multi: true },   // 模組語系檔目錄 assets/i18n/sample/
      ],
    };
  }
}
```

### 模組語系檔（`I18N_INDEX` 指向的字典目錄）

選單 title 與之後每顆 entity 的欄名 / 功能名全走 i18n key（key 缺會顯示 raw key，連內建刪除擋關聯訊息也靠它），所以模組自帶一份字典目錄。建 `assets/i18n/sample/zh-tw.json` 與 `en-us.json` **兩本**（key 樹同構、語言各一），先放模組層 key、entity 之後往裡加（見 [platform-add-entity](platform-add-entity.md) 第 5 步）：

```jsonc
{
  "menu":      { "Sample": "範例" },                 // 選單 title 用的 menu.* key
  "functions": { "Sample": { "Category": "範例" } }  // 模組分類名（權限頁 / 匯出用）
}
```

機制（主檔 vs 模組字典合併、`I18N_INDEX`、key 命名）見 [core/i18n-system](../core/i18n-system.md)。

### 接進 app

`app.route.ts` 加一條 lazy route（放在 `**` 萬用之前）：

```typescript
{ path: 'sample', loadChildren: () => import('./sample/sample.module').then(x => x.SampleModule), canActivate: [RequireLoginGuard] },
```

`app.module.ts` 的 `imports` 加 `SampleProviderModule.forRoot()`。

---

## 接下來

往這個模組加第一顆 entity（後端 CRUD + 前端頁）→ [platform-add-entity](platform-add-entity.md)

## 禁忌

- ❌ **`forRoot` 不要註冊路由**——只放 `MENU_ITEMS` / `I18N_INDEX`；路由由 host 的 `appRoutes` 持有（host 才有權替換組件）。
- ❌ **後端不要寫 Controller**——端點由 entity API 執行期生成。
- ❌ **模組類別依模組命名**（`SampleModuleConfig`）不是依某顆 entity——一個模組裝多顆 entity。

## Canonical reference

- 模組與 `IPlatformModule` → [core/entity-api](../core/entity-api.md)
- 前端 shell / 選單 / 三插槽 → [frontend/shell](../frontend/shell.md)
- 內建模組（簽核 / 稽核 / 字典 / 2FA …）→ [modules/](../modules/)
