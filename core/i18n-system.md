# i18n 系統

平台多語系機制:字典從哪來、**前端**如何載入/合併/解析/使用、**後端 `ITranslate`** 如何運作,以及前後端共用同一份字典的關係與**陷阱**。

---

## 字典來源(前後端共用)

- **主檔**:`assets/i18n/{lang}.json`
- **模組字典**:各模組自帶 `assets/i18n/{index}/{lang}.json`(`{index}` 見下方 `I18N_INDEX`)
- **前端**透過 HTTP 載入;**後端**從 `WebRootPath/assets/i18n` 讀**同一批檔案** → 前後端同源。
- 語言:`zh-tw` / `en-us`(`HCS_LANG_OPTIONS` 預設)。
- **key 命名規則(前後端一致)**:

  | 類別 | key 格式 | 例 |
  |---|---|---|
  | 共用 | `common.{item}` | `common.true` |
  | 列舉 | `enums.{EnumType}.{value}` | `enums.UserStatus.Active` |
  | 權限 | `permissions.{code}.{perm}` / `permissions.{perm}` | |
  | Model 欄位 | `models.{ns}.{Model}.{Field}` | `models.BaseModels.PlatformUser.Name` |
  | 元件/訊息 | `components.{feature}.{item}` | |
  | 驗證錯誤 | `errors.{key}` | `errors.required`(見 [validation-errors](validation-errors.md)) |

---

## 前端

### 1. 語言初始化

- `HCS_DEFAULT_LANG`(預設 `'zh-tw'`)、`HCS_LANG_OPTIONS`(預設 `['zh-tw','en-us']`)由 `HcsPlatformProviderModule` 提供。
- `appInitializerFactory` 決定啟動語言,**順序**:`localStorage['hcs-lang']` → `navigator.languages` 比對支援清單 → `HCS_DEFAULT_LANG`;接著 `setDefaultLang` + 設定 `DateAdapter` / `DatetimeAdapter` locale。
- 切語言時 `onLangChange` 同步日期 locale;`LanguageOptionService.getLangAsync()` 供查語言清單。

### 2. 字典載入與合併(`PlatformTranslateLoader`)

- 主檔 + 各 `I18N_INDEX` 模組字典**並行載入**,再以 `pathObject()` 遞迴合併進主字典。
- **`I18N_INDEX`**(`InjectionToken`,multi):各模組在 `forRoot()` 用 `{ provide: I18N_INDEX, useValue: '...', multi: true }` 註冊自己的字典目錄。現有:`basic`、`system-logging`、`app-update`、`code-table`、`approval-flow`、`third-party-login`、`two-factor-authentication`、`two-factor-authentication-google`。
- **`fixObject()`**:含 `.` 的 key(如 `"PlatformModule.Test.CustomerType"`)自動展開成巢狀。

### 3. link 機制

字典值可用特殊字串指向其他路徑,載入時 resolve(前端 lazy getter + cache;**後端同樣 resolve**,見 [前後端一致性](#前後端一致性)):

| 語法 | 用途 | 例 |
|---|---|---|
| `#{path}` | 整個值 ref 到另一路徑(**可指向子樹**) | `"#{models.Test}"`、`"#{../common.done}"` |
| `@{path}` | 內插 ref,可與文字混用 | `"生@{../../../functions.X}日"` |

- **path**:絕對(`a.b.c`)或相對(`./`、`../`,以 `/` 分段,相對當前 key 所在節點)。
- self-reference 會 `console.error`;目標路徑不存在會 `console.warn`。

### 4. 使用方式

- **ngx-translate `translate` pipe**:`{{ 'key' | translate }}`;帶參 `{{ 'key' | translate:params }}`。
- **平台自訂 pipe**(`projects/core/hcs-components/*.pipe.ts`):

  | pipe | 用法 | 對應 key |
  |---|---|---|
  | `enumTranslate` | `value \| enumTranslate:'EnumType'` | `enums.{EnumType}.{value}` |
  | `errorMessage` | `errors \| errorMessage` | `errors.{key}`(驗證錯誤) |
  | `permissionTranslate` | `perm \| permissionTranslate:'CodeType'` | `permissions.{code}.{perm}` |
  | `translateIfExists` | `key \| translateIfExists:'default'` | key 不存在時回預設值 |
  | `i18nNames` | `obj \| i18nNames` | 從物件的 Lang/Text 清單取當前語言(非字典) |
  | `enumOptions` | `Enum \| enumOptions` | 轉 `{value,text}[]`(純前端,不翻譯) |

- **`TranslateService`**(component/service 注入):`.get()`(回 Observable)、`.instant()`(同步)、`.onLangChange`(切語言事件)。

### 5. `{{}}` 內插

- ngx-translate 標準:`"歡迎 {{name}}"` + 呼叫時給 params `{name}`。
- 平台延伸:後端 `ValidationError` 的 `Data` 會攤平到頂層當 params(見 [validation-errors](validation-errors.md))。

---

## 後端(`ITranslate`)

- **介面**:`string Get(string key, Dictionary<string,string> parameters = null)`。
- **實作 `JsonFileTranslate`**:
  - 讀 `WebRootPath/assets/i18n` 下**所有** `{lang}.json`(含模組子目錄,`SearchOption.AllDirectories`),與前端**同一份字典**。
  - `WalkNode` 遞迴**扁平化**成 `Map<點路徑, 值>`;`static` 快取(依 lang + 檔案 mtime,改檔自動失效)。
  - **載入時 eager 解析 link**(`ResolveLinks`):扁平化後一次解完 `#{}` / `@{}`,快取存的是**成品字串**(已無 link 語法)。子樹 ref(`#{models.Test}`)會把目標子樹的葉節點**展開**到來源 prefix 底下,所以 `models.PlatformModule.Test.Customer.Name` 等都查得到。`{{key}}` 佔位**刻意保留**,留到 `Get` 時用呼叫端參數替換。
  - 語言來自 `IClientInfo.GetClientLang()`(前端請求帶 header `X-HCS-Lang`)。
  - `Get`:查 map,有 `parameters` 則逐個把 `{{key}}` 替換成值;**查無 key 回傳 key 原樣**。
- **DI**:`services.AddScoped<ITranslate, JsonFileTranslate>()`。
- **典型用途**:資料匯出(Excel)時翻譯欄位值,例 `translate.Get($"enums.UserStatus.{v}")`、`translate.Get("common.true")`。

---

## 前後端一致性

`#{}` / `@{}` link **前後端都會 resolve**:前端 `PlatformTranslateLoader` 在巢狀樹上 lazy getter 解析;後端 `JsonFileTranslate` 在扁平 map 上載入時 eager 解析(`ResolveLinks`)。兩邊語意對齊:

- **path**:絕對(`a.b.c`)或相對(`./`、`../`,以 `/` 分段,相對當前 key 所在節點);整值 `#{}` 可指向子樹,`@{}` 內插與文字混用。
- **fallback**:整值 `#{}` 解不到 → 該 key 無 entry(`Get` 回 key);內插 `@{}` 段解不到 → 該段空字串;self-ref / 循環 → 偵測後當解不到(後端靜默,前端 `console.error/warn`)。內插 `@{}` 段若指到**子樹**(非單一值):後端視為解不到 → 空字串,前端 `nestedGet` 拿到物件 → stringify 成 `[object Object]`(實務上不會在內插指子樹,後端行為較合理)。
- **子樹 ref 在後端的呈現**:扁平 map 沒有「指向子樹」的單一 entry,改以**展開**呈現——`#{models.Test}` 會把 `models.Test.*` 的葉節點複製到來源 prefix 底下。因此 ref 節點本身(`models.PlatformModule.Test`)在後端**不是 scalar、查無 entry**,但其下的葉(`...Test.Customer.Name`)都查得到,結果與前端一致。
- `{{}}` 內插兩邊都支援,但 params 來源不同:前端由 ngx-translate 呼叫端給、後端由 `ITranslate.Get` 的 `parameters` 字典給。link resolve(載入期)發生在 `{{}}` 替換(執行期)**之前**,兩者不衝突——`#{}` 指到含 `{{field}}` 的值時,佔位會被保留到 `Get` 才填。