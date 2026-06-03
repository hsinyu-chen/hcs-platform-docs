# Pipe(DIPipe)

SDK 的**核心 compose 基元**。整個資料處理鏈(CRUD、驗證、匯出、服務注入)都是把一個個 `DIPipe` 步驟串起來組成的。

---

## 本質

`DIPipe<TIn,TOut>` 就是一個 delegate:

```csharp
public delegate Task<TOut> DIPipe<in TIn, TOut>(IServiceProvider serviceProvider, TIn input);
```

一個「吃 `TIn`、可透過 `IServiceProvider` 取 DI 服務、非同步吐 `TOut`」的步驟。`EmptyDIPipe.Empty` 是 identity pipe(原樣回傳 input),當 pipeline 起點。

---

## 串接:`.Pipe(...)`

`.Pipe(...)` 把「下一步」接到 pipe 尾端,回傳一條新 pipe。下一步可以是(同步 / async × 純資料 / 帶 DI):

| 形式 | 簽名 | 用途 |
|---|---|---|
| 純副作用 | `Action<TOut>` | 改 input、log...,回傳同一個 `TOut` |
| 純轉換 | `Func<TOut,TNext>` | `TOut` → `TNext` |
| 帶 DI 副作用 | `Action<TServices,TOut>` | 同上 + 注入服務 |
| 帶 DI 轉換 | `Func<TServices,TOut,TNext>` | 同上 + 注入服務 |
| async | 以上各形式 + `Task` / `Task<TNext>` | 非同步版 |
| merge | `DIPipe<TOut,TNext>` | 接上另一條現成 pipe |
| builder | `DIPipeBuilder<...>` | 接一段子 pipeline(見下) |

範例:

```csharp
pipe.Pipe(i => { i.Value += 1; });                               // Action(純副作用)
pipe.Pipe(async i => { await ...; i.Value += 1; return i; });    // async Func(轉換)
pipe.Pipe(((SA a, SB b) services, TestContext i) => { ... });    // 帶 DI:首參 tuple = 服務
```

---

## DI 注入機制(關鍵)

帶 DI 的 `.Pipe` 多載,**lambda 的第一個參數就是要注入的服務**,第二個才是資料:

- **單一服務**:`(SA a, TIn x) => ...` → 框架 `sp.GetService<SA>()`
- **多服務用 `ValueTuple`**:`((SA a, SB b, SC c) services, TIn x) => ...` → 逐欄從 DI 取

框架內部用 `Expression` 編譯一個「從 `sp` 逐欄 `GetService` 填值」的 factory:

- `TServices` 是 `ValueTuple` 時,**遞迴**支援巢狀 tuple(C# 的 `ValueTuple` 超過 7 欄會嵌套 `TRest`,所以服務數很多也行)。
- 否則直接 `sp.GetService<TServices>()`。
- factory 依型別**快取**,只編譯一次。

> 這就是為什麼驗證能寫 `.Pipe((TDbContext ctx, EntityValidationResult<T> errors) => ...)`、匯出能拿到 `ITranslate` —— 框架自動把首參(或 tuple)的服務從 DI 解析好再呼叫。

---

## 分支:`SwitchCase`

```csharp
pipe.SwitchCase(
    defaultBuilder,                                 // 預設分支
    cases => cases.AddCase(when, caseBuilder));     // 條件分支(可多個)
```

跑完 `pipe` 得到 `result`,依序測 `when(result)`,命中就走該 case 的子 pipe,全沒中走 default。**驗證流程就靠它**:有錯 → `BadRequest`、無錯 → 繼續寫 DB(見 [validation-errors](validation-errors.md))。

---

## 子管線:`DIPipeBuilder`

`DIPipeBuilder<TIn,TOut[,TNext]>` = `(DIPipe) => DIPipe`:收一個起點 pipe、回傳組好的 pipe。上層 API 常以 builder 形式把某段子 pipeline 交給你擴充——你拿到起點 pipe(通常是 `EmptyDIPipe.Empty` 或上游接好的 pipe),在它上面 `.Pipe` 疊步驟即可。

---

## pipe step 的三種組織寫法

同樣是「往 pipeline 加一步」,依重用程度有三種寫法:

### 1. inline lambda(一次性)
當場寫,最直接。適合單一處、不重用;多步直接串:

```csharp
builder
    .Pipe((SomeService svc, TIn x) => { /* ... */ })   // 帶 DI
    .Pipe(x => { /* ... */ });                          // 純資料
```

### 2. static pre-build(抽成 static field 重用)
把 step 抽成 `static readonly Func<TServices, TIn, Task>`(副作用版)或 `...Task<TNext>>`(轉換版),多處 / 多 API 共用同一份邏輯。例如代碼表的 `ValidatePost`/`ValidatePut`/`ValidateDelete` 三個 step 共用一個私有 `Validate`:

```csharp
public static readonly Func<(IPrincipal user, DbContext db,
        IEnumerable<CodeTableConfigService> configs, IUserOrganization org),
    EntityValidationResult<CodeTable>, Task>
    ValidatePost = async (services, vr) => await Validate(Action.Create, services, vr);

// 掛載:step 的型別就是 .Pipe 接受的 lambda 形狀,直接傳
builder.Pipe(ValidatePost);
```

- 首參 tuple = DI 服務、次參 = 資料,跟 inline 一樣,只是抽成具名 field。
- 好處:邏輯集中一份、Post/Put/Delete 共用,且不必每次 new lambda。

### 3. extension helper(包成 `.Xxx()`,最高重用 + 可配置)
邏輯要跨 entity 重用、或要對外提供配置時,包成 `this DIPipe<...>` 的 extension method。例如內建的 `.Unique`:

```csharp
public static DIPipe<EntityValidationResult<T>, EntityValidationResult<T>> Unique<T, TDbContext>(
    this DIPipe<EntityValidationResult<T>, EntityValidationResult<T>> builder,
    Action<UniqueValidateConfigurator<T, TDbContext>> options) where ...
    => builder.Pipe(async (TDbContext ctx, EntityValidationResult<T> errors) =>
    {
        var validator = new UniqueValidate<T, TDbContext>(ctx);
        options(new UniqueValidateConfigurator<T, TDbContext>(validator));
        await validator.Validate(errors.Data, errors.AddError);
    });

// 掛載:對外是流暢的 .Unique(...),內部仍是 builder.Pipe(...)
builder.Unique(c => c.AddProperty(x => x.Name));
```

> 完整的 validator 打包(三件套、再包一層 entity 級便利封裝)見 [validation-errors](validation-errors.md) 的「打包可重用 validator」。

**選哪種**:一次性 → inline;同邏輯多處 / 多 API 共用 → static field;跨 entity 重用或要對外配置 → extension。

---

## 執行模型

- `DIPipe` 是 delegate,最終由框架以 `await pipe(serviceProvider, input)` 執行(controller / endpoint 餵入 **request-scoped** 的 `sp` 與輸入)。
- DI 服務都從**傳入的 `sp`** 解析 → 跟著 request scope 走。
- 資料流:每個 `.Pipe` 包一層 async,`await` 前一步的 output 餵給下一步。
- 中止:`SwitchCase` 切到別的分支,或 lambda 內拋例外中斷。

---

## 其他 pipe 形式

SDK 另有 `IDataPipe<T>` 體系,用於 `ConfigDataPipe` 的資料處理鏈,建在更上層,與這裡的 `DIPipe` compose 基元是不同層次。
