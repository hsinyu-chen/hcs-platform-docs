# 前端 SDK(Angular 17)

平台前端是一套 **Angular 17 + Material + Capacitor 的 ng library SDK**:每個後端模組對應一個前端 library,搭配 `@hcs/core` 提供的 OData datasource / 權限 / 表單 / 列表元件。

> **前端不會由宣告自動生成**(與後端不同)。SDK 提供框架、可重用元件與基底類別;每個功能的頁面(list / form)、路由、選單仍由使用端撰寫——只是大量樣板(資料抓取、分頁、查詢轉 OData、存檔、權限)已被元件與基底類別包好。

---

## Library projects

| Library | npm 套件 | 對應後端模組 | 提供內容 |
|---|---|---|---|
| `core` | `@hcs/core/*` | — | **所有 SDK 的基石**:API client、OData datasource、表單/列表/對話框元件、權限、i18n loader、login 頁、user state |
| `basic` | `@hcs/basic` | `Hcs.PlatformModule.Basic` | User / Group / Organization 列表 + 表單頁 |
| `code-table` | `@hcs/code-table` | `Hcs.PlatformModule.CodeTable` | 字典表管理頁 |
| `approval-flow` | `@hcs/approval-flow` | `Hcs.PlatformModule.ApprovalFlow` | 簽核流程設計頁(階段式編輯器 + 流程圖視覺化) |
| `system-logging` | `@hcs/system-logging` | `Hcs.PlatformModule.SystemLogging` | 稽核軌跡檢視頁 |
| `app-update` | `@hcs/app-update` | `Hcs.PlatformModule.AppUpdate` | App 版本管理頁 |
| `third-party-login` | `@hcs/third-party-login` | `Hcs.ThirdPartyLogin.*` | OAuth 登入頁與綁定流程 |
| `two-factor-authentication` | `@hcs/two-factor-authentication` | `Hcs.PlatformModule.TwoFactorAuthentication` | 2FA 設定主框架(可掛 provider) |
| `two-factor-authentication-google` | `@hcs/two-factor-authentication-google` | `Hcs.2FA.GoogleAuthenticator` | Google Authenticator 設定 UI |
| `create-hcs-app` | `@hcs/create-hcs-app` | — | Angular Schematic:一個指令建立新專案骨架 |

> `npm run build-lib` 一次建好全部(core 必先,再各 module library)。

---

## SDK 慣例(重要)

看懂這七個慣例 → 看懂所有 SDK library 的結構。

### 1. `HcsXxxProviderModule.forRoot()`

每個 library 對外曝露一個 `HcsXxxProviderModule`,`static forRoot()` 只註冊三件事:**選單**(`MENU_ITEMS`)、**i18n 檔名**(`I18N_INDEX`)、**工具列**。**不負責路由**——路由另由 `defaultRoute` / `createRoute()` 提供,因為 host 可能要替換元件。

```typescript
static forRoot(): ModuleWithProviders<HcsBasicProviderModule> {
    return {
        ngModule: HcsBasicProviderModule,
        providers: [
            { provide: MENU_ITEMS, useClass: BasicMenu, multi: true },
            { provide: I18N_INDEX, useValue: 'basic', multi: true },
        ]
    };
}
```

### 2. `defaultRoute` / `createRoute(overrides)`

Host 在 `app.route.ts` 組路由:直接用 `Xxx.defaultRoute`,或用 `createRoute({...})` 傳入要替換的元件:

```typescript
children: [
    HcsCodeTableProviderModule.defaultRoute,            // 用預設
    HcsBasicProviderModule.createRoute({                // 替換某頁元件
        platformGroupForm: () =>
            import('./custom/custom-platform-group-form.component')
                .then(x => x.CustomPlatformGroupFormComponent)
    }),
    { path: 'platformmodule/test',                      // 自訂頁面
      loadChildren: () => import('./hcs-test/hcs-test.module').then(m => m.HcsTestModule),
      canActivate: [RequireLoginGuard] },
]
```

`HcsUrlMatcher` 是 SDK 的自訂 URL matcher,處理多組織 / 多語系前綴。

### 3. Component Override 機制

每個 library 開放 `XxxComponentOverrides` interface——**不必 fork SDK**,傳入 lazy-loaded component 就能替換預設頁面;路由結構、權限路徑、i18n key 都保留。

### 4. `MENU_ITEMS` — 模組註冊選單

每個 library 的 `Menu` class 繼承 `MenuItemProvider` 掛在 `MENU_ITEMS`(multi)。每個 menu item 自帶 `hasPermission` 檢查——**權限不通過自動不顯示**。SDK menu 元件收集所有 multi-provider 後合併渲染。

### 5. `I18N_INDEX` — 多套 i18n 檔合併

每個 library 註冊一個字串 key(如 `'basic'`),loader 啟動時去 `assets/i18n/<key>/<lang>.json` deep-merge。新增模組不必碰 host 的 i18n 設定。(機制細節見 [core/i18n-system.md](core/i18n-system.md)。)

### 6. `HCS_FUNCTION_NAME` + `Permission`

每個頁面 component 透過 `providers` 宣告自己對應的後端功能碼:

```typescript
@Component({
  providers: [
    { provide: HCS_FUNCTION_NAME, useValue: 'Basic.PlatformUser' },
    Permission
  ]
})
export class PlatformUserComponent {
  constructor(public permission: Permission) {}
  // 模板可呼叫 permission.hasPermission('Modify') → 自動拼成 'Basic.PlatformUser.Modify'
}
```

前後端用同一條 `Function.Permission` 字串對齊(對應後端 `AddModuleFuncion` + `AddStandardApiRoles` + `AddPermission`)。

### 7. `DataSourceFactory` + `OdataDataSource`

對接後端 `AddEntityApi<T>` 自動產生的 OData 端點:

```typescript
constructor(datasourceFactory: DataSourceFactory) {
    this.data = datasourceFactory.getDataSource(PlatformUser);
    //   ↑ 自動對接 GET /api/odata/PlatformUser,使用端不必手寫 API 呼叫
}
```

兩個 overload:傳 entity class → `OdataDataSource<T>`(HTTP + OData);傳陣列 → `MemoryDatasource<T>`(本地)。兩者實作同一 `IDataSource<T>`,所以 grid / form 元件不在乎資料從哪來。

---

## `@hcs/core` 提供什麼

`core` 是骨幹,含四個 entry point:

- **`@hcs/core/hcs-lib`** — 純 TS 工具與 service:資料層(`DataSourceFactory` / `OdataDataSource` / `MemoryDatasource` / `OdataFilter`)、權限狀態(`UserState` / `Permission` / `RequireLoginGuard`)、路由(`HcsUrlMatcher` / `OrgRouter`)、i18n loader、檔案/列印/通知、工具、DI tokens、裝置偵測(`HcsPlatform`,web / iOS / Android 判斷)。
- **`@hcs/core/hcs-components`** — 可重用 UI 元件:列表(`hcs-data-grid` + 欄位/查詢指令)、表單(`hcs-form-page` 系列)、對話框、參考選擇、檔案、條碼掃描、CKEditor、登入頁、錯誤摘要(`hcs-error-summary`)、各式 pipe / directive。
- **`@hcs/core/hcs-platform`** — 前端 bootstrap 入口:`HcsPlatformProviderModule.forRoot()`(locale / 預設語系 / login 頁 / user menu)、`appInitializerFactory`、ngx-translate loader 註冊。
- **`@hcs/core/models`** — 後端 entity 的 TypeScript 對應(手寫 + 工具生成)。

---

## host app 怎麼組

`app.module.ts` 的典型結構:`HcsPlatformProviderModule.forRoot()` 一定第一個,接著 ngx-translate、Router,再把各 library 的 `forRoot()`(順序不拘,都是 multi-provider)與自家模組逐一 import;`providers` 放全域設定(`HCS_LANG_OPTIONS`、`HCS_EXPORT_LIMIT`、`HCS_LOCALE_FORMATS` 等)。

---

## 新增一個業務功能(前端)

假設後端已用 `AddEntityApi<Invoice>` 宣告好模組,前端:

1. **建模組目錄** `src/app/my-feature/`:`MyFeatureProviderModule.ts`、`route.ts`、`menu.ts`,加 `invoice/`(列表)與 `invoice-form/`(表單)兩個元件。
2. **ProviderModule**:`forRoot()` 註冊 `MENU_ITEMS` + `I18N_INDEX`;`defaultRoute` 指向本模組路由。
3. **list 元件**:`extends BaseListComponent<Invoice>`,provide `BaseComponentService`(基底依賴,非 root,漏了會 `NullInjectorError`)+ `HCS_FUNCTION_NAME`(對應後端功能碼),`super(service, Invoice)`,建查詢用 reactive form;template 用 `<hcs-data-grid>` + `*hcsDataGridColum` 宣告欄位、`hcs-*-input` 宣告查詢條件。
4. **form 元件**:`extends BaseFormComponent<Invoice>`,provide `BaseComponentService` + `HCS_FUNCTION_NAME`,`super(service, Invoice)`,建編輯用 reactive form(含 `Validators`)。
5. **i18n**:在 `src/assets/i18n/my-feature/{zh-tw,en-us}.json` 各放一份,loader 自動載入。
6. **註冊**:`app.module.ts` import `MyFeatureProviderModule.forRoot()`;`app.route.ts` 加 `MyFeatureProviderModule.defaultRoute`。

資料抓取、分頁、查詢轉 OData、存檔、驗證錯誤顯示由基底類別 + `DataSourceFactory` 處理,**不需自己寫 API 呼叫**。

---

## 打包與行動裝置

- `npm start` dev server(HTTPS + proxy 至 `/api`)、`npm run build-lib` 建 SDK library、`npm run prod-build` 出 production(輸出至後端 `wwwroot/`)。
- 同一份 codebase 透過 **Capacitor 5 + Ionic 7** 同時跑 Web / iOS / Android;`HcsPlatform` service 判斷執行環境;build 時用 `--configuration=app` 切 native。

---

## 平台前端開關總覽(opt-in 索引)

平台前端的對外設定點幾乎都是 `HCS_*` 的 Angular `InjectionToken`。**很多是「不 provide 就用不到」的隱形 opt-in**(如閒置自動登出),所以這裡列一張全集當發現性樞紐;細節各看對應能力篇。

> 維護契約:**之後任何 PR 新增 / 改 `HCS_*` token,同 PR 補這張表 + 對應能力篇。**

### 全域開關

| Token | 用途 | 預設(沒 provide 時) | 文件 |
|---|---|---|---|
| `HCS_IDLE_TIMEOUT_MINUTES` | 閒置自動登出分鐘數 | 未給 / `≤0` = 關 | [login](frontend/login.md) |
| `HCS_EXPORT_LIMIT` | 匯出筆數上限 | **無 fallback,必須 provide** | [list](frontend/list.md) |
| `HCS_ENABLE_EXPORT` | 匯出 + 列選取總開關 | undefined → `true` | [list](frontend/list.md) |
| `HCS_ENABLE_STATE` | 排序 / 分頁狀態持久化 | 由 host 提供;`false` = 不持久 | [list](frontend/list.md) |
| `HCS_GOOGLELOGIN_CLIENTID` | Google OAuth client id | 用 Google 登入才需給 | [login](frontend/login.md) |
| `HCS_LANG_OPTIONS` | 可選語系 | `['zh-tw','en-us']`(forRoot) | [i18n](core/i18n-system.md) |
| `HCS_DEFAULT_LANG` | 預設語系 | `'zh-tw'`(forRoot) | [i18n](core/i18n-system.md) |
| `HCS_FUNCTION_NAME` / `_ROUTE` | 頁面對應的後端功能碼 / 路由 | 每頁自行 provide | [form](frontend/form.md) / [permissions](core/permissions.md) |

### 設定值

| Token | 用途 | 預設 | 文件 |
|---|---|---|---|
| `HCS_DEFAULT_APP_CONFIG` | header 四鈕(home/scan/fullScreen/notification) | `{home:true, scan:false, fullScreen:false, notification:true}` | [shell](frontend/shell.md) |
| `CKEDITOR_CONFIG` | RichText 編輯器設定 | fallback `DefaultCkEditorConfig` | [form](frontend/form.md) |
| `HCS_EXPANSION_PANEL_DEFAULT_OPTIONS` | 表單區塊預設展開 | `{ expand: true }` | [form](frontend/form.md) |
| `HCS_LOCALE_FORMATS` | 日期 / 數字 locale 格式 | host 提供 | [i18n](core/i18n-system.md) |

### 攔截 service(multi,回 `false` 中止)

| Token | 介面 | 時機 | 文件 |
|---|---|---|---|
| `HCS_FORM_SUBMIT_SERVICE` | `IFormService` | 表單送出前 | [form](frontend/form.md) |
| `HCS_FORM_MODEL_LOADED_SERVICE` | `IFormService` | 表單載入後 | [form](frontend/form.md) |
| `HCS_LIST_SERVICE` | `IListService` | 刪除前 | [list](frontend/list.md) |
| `HCS_USER_STATE_SERVICE` | `IUserStateService` | 登入 / 登出前後 | [login](frontend/login.md) |
| `HCS_LOGIN_STATUS_CODE_HANDLER` | `ILoginStatusCodeHandler` | 登入挑戰(2FA 等) | [login](frontend/login.md) |
| `HCS_PRINT_SERVICE` | `IPrint` | 列印 | [list](frontend/list.md) |

### 組件插槽(multi,「加」不是「換」)

| Token | 掛在哪 | 文件 |
|---|---|---|
| `HCS_INDEX_COMPONENT` | 首頁區塊 | [shell](frontend/shell.md) |
| `HCS_TOOLBAR_COMPONENT` | 頂部工具列 | [shell](frontend/shell.md) |
| `HCS_USER_MENU_COMPONENT` | 使用者選單 | [shell](frontend/shell.md) |
| `HCS_LOGIN_PAGE_COMPONENT` | 登入頁底部 | [login](frontend/login.md) |
| `MENU_ITEMS` | 左側選單 | [shell](frontend/shell.md) |
| `I18N_INDEX` | i18n 檔合併 | [i18n](core/i18n-system.md) |

> **功能專屬 token** 不在此表(歸各功能 doc):2FA 掛載點 `HCS_2FA_DIALOG` 見 [2fa](features/2fa.md);Basic 的 User/Org/Group 替換 token、SystemLogging 的值元件等,各見其功能文件。
