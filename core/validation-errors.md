# 驗證錯誤(Validation Errors)

平台的資料驗證錯誤如何從**後端產生** → 經 API 回傳 → 到**前端顯示**,以及 i18n 訊息與參數的解析機制。也涵蓋如何把驗證邏輯打包成可重用的 validator(像內建 `Unique` 那樣)。

---

## 何時用

後端驗證錯誤適合**只有後端才驗得了**的規則:

| 情境 | 放哪 |
|---|---|
| 唯一性(查 DB)、跨表關聯、權限、跨欄位 / 業務規則 | ✅ 後端 `OnValidate` |
| 純格式:必填、長度、pattern、型別 | ❌ 前端 reactive form validator(即時、省一次來回) |
| DB 硬約束(FK / unique index) | 可加,但錯誤訊息不友善;要友善訊息仍用 validator |

原則:**前端能即時驗的放前端;需要 DB / 伺服器狀態 / 權限才判斷得了的,才走後端驗證**。兩者可並存(前端擋格式、後端擋業務規則),最終都顯示在同一個 `hcs-error-summary`。

後端 validator 該寫 inline 還是打包,見下方〔[進階:打包可重用 validator](#進階打包可重用-validator)〕與 [pipe](pipe.md) 的三種組織寫法。

---

## 使用端

### 1. 後端:產生驗證錯誤

驗證掛在 entity API 的 `OnValidate` pipe,操作傳入的 `EntityValidationResult<T>`:

```csharp
options.ConfigPostApi(x => x.OnValidate(v => v.Pipe(
    (EntityValidationResult<Customer> c) =>
    {
        if (string.IsNullOrWhiteSpace(c.Data.Name))
            c.AddError(new() { Title = "Name", Message = "required" });
    })));
```

或用 `AddErrorFor` helper,自動以欄位運算式當 `Title`:

```csharp
c.AddErrorFor(x => x.Name, "required");             // Title="Name", Message="required"
c.AddErrorFor(x => x.Name, "duplicate", dataJson);  // 第三參數 → Data
c.AddErrorFor(x => x.Address.City, "required");     // 巢狀運算式 → Title="Address.City"
```

> 巢狀運算式的 `Title` 是用 `.` 串起整條路徑(`Address.City`),前端欄位名 i18n key 因此變成 `models.{functionName}.Address.City`——要翻得出來,字典裡得照這條巢狀路徑建 key。

**`ValidationError` 三個欄位:**

| 欄位 | 用途 |
|---|---|
| `Title` | 分組鍵,通常 = 欄位名;前端據此對應欄位、組欄位名 i18n key |
| `Message` | 訊息的 i18n key(前端查 `errors.{Message}`) |
| `Data` | 選填,JSON 字串,作為 i18n 內插參數(見 [4. i18n](#4-i18n)) |

**同一欄位多筆錯誤** — 對**同一個 `Title`** 多次 `AddError`(各帶不同 `Message`)即可,後端以 `IList` 累積、不去重:

```csharp
c.AddError(new() { Title = "Name", Message = "duplicate", Data = "{\"who\":\"alice\"}" });
c.AddError(new() { Title = "Name", Message = "tooLong",   Data = "{\"who\":\"alice\"}" });
```

> ⚠️ **此情境需客製 validator。** 內建 validator(如 `Unique`)是「每個欄位各一筆」,不會對同一 `Title` 產生多筆;只有你自己在 `OnValidate` 對同欄多次 `AddError` 才會用到。

**內建 validators**(`BuiltInValidators`):

| 名稱 | 用途 | 掛法 |
|---|---|---|
| `Unique` / `UniqueValidation` | 欄位唯一性(查 DB) | `apiBuildContext.UniqueValidation(c => c.AddProperty(x => x.Name))`(一次掛 Post+Put,自動設 `IsCreate`) |
| `CheckRefForDelete` | 刪除前檢查**指定**關聯,有關聯則擋 | `apiBuildContext.CheckRefForDelete(x => x.Orders)` |
| `CheckAllRefForDelete` | 刪除前**自動掃 entity 所有導覽集合**,任一有關聯就擋,免逐個指定 | `ConfigDeleteApi(d => d.OnValidate(v => v.CheckAllRefForDelete()))`(只有 pipe 層級,無 `apiBuildContext.` 便利封裝) |

> **複合唯一鍵**:`AddProperty` 可連續鏈接,判定「多欄組合」唯一,如 `c => c.AddProperty(x => x.Platform).AddProperty(x => x.Product).AddProperty(x => x.Version)`。
> **`Unique` 的 `NullCheck`**:預設 `false` 時,若設定的 property **全為 null** 會直接跳過唯一性檢查;要讓「全 null 也算重複」,在 configurator 設 `NullCheck = true`。

### 2. API 回應形狀

驗證失敗回 **400 BadRequest**,body 是 `Title → ValidationError[]` 的字典:

```json
{
  "Name": [
    { "Title": "Name", "Message": "duplicate", "Data": "{\"who\":\"alice\"}" },
    { "Title": "Name", "Message": "tooLong",   "Data": "{\"who\":\"alice\"}" }
  ]
}
```

### 3. 前端:顯示

最常見——`<hcs-form-page>` 內建 `<hcs-error-summary>`,表單 submit 失敗會自動把錯誤灌進去,**消費端不需額外處理**:

```html
<hcs-form-page [formGroup]="formGroup" [dataSource]="datasource"> ... </hcs-form-page>
```

自訂頁面則建立 `ErrorHelper`,在錯誤回呼呼叫 `addHttpError`,再交給 summary:

```typescript
errors = new ErrorHelper();
// ...
this.http.post(url, model).subscribe({
  error: (err: HttpErrorResponse) => this.errors.addHttpError(err)
});
```
```html
<hcs-error-summary [errors]="errors"></hcs-error-summary>
```

**顯示行為:**
- 每筆 `ValidationError` 各成一行,垂直堆疊(同一欄位多筆 → 多行)。
- form 原生驗證錯誤(reactive form validator)經 `errorMessage` pipe 顯示時,**一個欄位只顯示第一條**(逐條修);`hcs-error-summary` 則逐筆全列。

### 4. i18n

- **訊息**:`Message` → i18n key `errors.{Message}`(例 `errors.duplicate`)。查不到時 fallback 顯示 key 原樣。
- **參數(後端 `Data`)**:`Data` 的 JSON 會**攤平到頂層**,直接用 `{{欄位}}` 取;整個物件同時掛在保留鍵 `data` 下,相容舊寫法 `{{data.欄位}}`:

  ```
  後端 Data: {"who":"alice"}
  i18n:      "errors": { "duplicate": "客戶 {{who}} 已重複" }   // {{who}} → alice
             (舊寫法 {{data.who}} 仍可用;欄位撞名 data 時用 {{data.data}})
  ```

- ⚠️ **欄位名(field)的 i18n key 前綴** — summary 顯示的欄位名 key = `models.{functionName}.{Title}`,`functionName` 來自後端 `AddModuleFuncion`。**若 model 翻譯的前綴跟 functionName 不一致**(例:model 掛在 `models.Test.Customer`,但 functionName 是 `PlatformModule.Test.Customer`),欄位名就翻不出來。解法是用 i18n 的 link 機制(`#{}`)把前綴 alias 過去:

  ```json
  "models": { "PlatformModule": { "Test": "#{models.Test}" } }
  ```

> i18n 的 link 語法(`#{}` / `@{}`)、模組字典合併、相對/絕對路徑等通用機制,屬獨立主題,見 [i18n 系統](i18n-system.md)。

---

## 進階:打包可重用 validator

當驗證邏輯要**跨 entity 重用、需配置、或依賴 DI 服務**時,把它包成 validator(像內建 `Unique`)。三級複雜度,由簡入繁:

### Level 1 — inline(一次性)
直接寫在 `OnValidate` 裡(見上方〔後端:產生驗證錯誤〕)。適合單一 entity、簡單條件。

### Level 2 — extension method(無獨立 class)
邏輯短、無狀態時,包成 pipe extension(範本:`ValidOdata`):

```csharp
public static DIPipe<EntityValidationResult<TEntity>, EntityValidationResult<TEntity>> MyCheck<TEntity>(
    this DIPipe<EntityValidationResult<TEntity>, EntityValidationResult<TEntity>> builder) where TEntity : class
    => builder.Pipe((ISomeService svc, EntityValidationResult<TEntity> errors) =>
    {
        if (!svc.IsValid(errors.Data))
            errors.AddError(new() { Title = "...", Message = "..." });
    });
```

> **DI 注入**:`Pipe` 的 lambda **首參(或 tuple)就是 DI 服務**,框架自動解析,validator 不必自己抓。`errors.Data` 是被驗證的實體、`errors.AddError` 是錯誤收集 callback。

### Level 3 — 完整三件套(範本:`Unique`)
邏輯複雜、需配置時,拆三部分:

1. **Validator class** — 封裝邏輯,對外 `Validate(entity, addError)`:
   ```csharp
   internal class MyValidate<TEntity, TDbContext> where TDbContext : DbContext where TEntity : class
   {
       internal MyValidate(TDbContext ctx) { /* 收 DI 服務 */ }
       public async Task Validate(TEntity entity, Action<ValidationError> addError)
       {
           if (/* ... */) addError(new() { Title = "...", Message = "..." });
       }
   }
   ```
2. **Configurator** — 流暢配置 API:
   ```csharp
   public class MyValidateConfigurator<TEntity, TDbContext> /* ... */
   {
       public MyValidateConfigurator<TEntity, TDbContext> AddProperty(/* ... */) { /* ...; */ return this; }
   }
   ```
3. **Extension** — 掛載 + DI + 套用配置:
   ```csharp
   public static DIPipe<EntityValidationResult<TEntity>, EntityValidationResult<TEntity>> MyValidator<TEntity, TDbContext>(
       this DIPipe<EntityValidationResult<TEntity>, EntityValidationResult<TEntity>> builder,
       Action<MyValidateConfigurator<TEntity, TDbContext>> options) where TDbContext : DbContext where TEntity : class
       => builder.Pipe(async (TDbContext ctx, EntityValidationResult<TEntity> errors) =>
       {
           var validator = new MyValidate<TEntity, TDbContext>(ctx);
           options(new MyValidateConfigurator<TEntity, TDbContext>(validator));
           await validator.Validate(errors.Data, errors.AddError);
       });
   ```

### entity 級便利封裝
再包一層 `EntityApiBuildContext` extension,讓消費端一行掛多個 API(範本:`UniqueValidation` 一次掛 Post+Put、`CheckRefForDelete` 掛 Delete):

```csharp
public static EntityApiBuildContext<TKey, TEntity> MyValidation<TKey, TEntity>(
    this EntityApiBuildContext<TKey, TEntity> ctx,
    Action<MyValidateConfigurator<TEntity, PlatformDbContext>> options) where TEntity : class
    => ctx.ConfigPostApi(x => x.OnValidate(v => v.MyValidator(options)))
          .ConfigPutApi(x => x.OnValidate(v => v.MyValidator(options)));
```

消費端只要:
```csharp
apiBuildContext.MyValidation(c => c.AddProperty(x => x.Name));
```

> i18n 的 link 機制(`#{}` / `@{}`)、模組字典合併等通用機制屬獨立主題,見 [i18n 系統](i18n-system.md)。
