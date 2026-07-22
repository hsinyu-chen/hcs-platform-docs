# 自訂端點(Flow API)

`AddEntityApi` 一行長出 CRUD 五件套,但很多功能不是「對一顆 entity 做標準增刪改查」——結帳、簽核、排序、批次匯入、回傳一個算出來的視圖…這些用 **Flow API**:`AddPostFlowApi` / `AddGetFlowApi` / `AddPutFlowApi` / `AddDeleteFlowApi` / `AddQueryFlowApi`,**自己用 pipe 組件把一條 pipeline 組起來**,產出單一端點。

> 關鍵心智模型:**內建 `AddEntityApi` 本身就是用同一組 pipe 組件預先組好的 Flow API**。`AddPostApi` 的本體就是 `Pipe(Request).GetRequestModel<T>().RunValidation(...).CreateData().Ok()` 這條鏈。所以「自訂端點」不是另一套機制,而是把那條鏈拆開、換成你要的步驟。

---

## 本質

```csharp
// 在模組的 Build 裡,moduleBuilder 上呼叫
var sortApi = moduleBuilder.AddPostFlowApi<long, Invoice>("Invoice.Sort", b =>
    b.GetRequestModel<Invoice>()        // HttpRequest → Invoice(取回 model-bind 的 input)
     .Pipe(MyPipes.ReorderInvoices)     // 你的業務步驟(見 pipe.md 的三種寫法)
     .Ok());                            // → 200 + 回傳值
```

- **入口**:`moduleBuilder.Add{Post,Get,Put,Delete}FlowApi<TKey, TEntity>(name, flowBuilder, ...)`、`AddQueryFlowApi<TEntity>(name, flowBuilder, ...)`(Query 無 `TKey`)。泛型多載 `<…, TDbContext>` 預設 `PlatformDbContext`。
- **`name`** 是這支端點的 key,直接決定**前端 URL = `api/entity/<name>`**(見下「前端呼叫」)。慣例用 `功能.動作`,如 `"Invoice.Sort"`、`"Order.Confirm"`。
- **`flowBuilder`** 型別是 `DIPipeBuilder<HttpRequest, HttpRequest, IActionResult>`:給你一條從 `HttpRequest` 起步的 pipe,你疊步驟、**最後收尾成 `IActionResult`**(用 `.Ok()` / `.OkOrNotFound()` 等)。
- **回傳 `IRoleToken`**:這個 token **必須**綁進某個權限(見下「權限綁定」),否則沒人叫得動。
- 兩個選用參數 `preTransBuilder` / `afterTransBuilder` = 交易前 / 交易後 pipe(型別 `DIPipeBuilder<HttpRequest, HttpRequest, object>`,對應 entity API 的 `OnBeforeTransaction` / `OnAfterTransaction`,語意見 [entity-api](entity-api.md) 交易章)。**它們的 pipe 必須以 `object` 收尾**:這兩段不負責回 HTTP response(那是 `flowBuilder` 收尾成 `IActionResult` 的事),而 `DIPipe<TIn,TOut>` 沒有 void 輸出型別,所以「輸出丟掉」的 hook 用 `object` 當載體——最後一步得產出個 object(值被忽略,如結尾 `.Pipe(x => (object)x)`)。entity API 的 `OnAfterTransaction` 框架會幫你補這個 cast,**raw FlowApi 不補,得自己收**(實測:結尾停在 `DIPipe<HttpRequest, Order>` 編譯不過)。

`Post / Put / Delete` 整條跑在單一 DB 交易內(任一步 throw → rollback);`Get` / `Query` 唯讀、不開交易。要在 pipe 中改交易行為(關掉自動交易自管、或調隔離級別)用 `.SetAutoTransaction(enable, isolationLevel)` / `.SetAutoTransactionIsolationLevel(...)`。交易語意與 entity API 完全一致,見 [entity-api](entity-api.md)。

> **副作用放哪。** 串在主 `flowBuilder` 內的步驟跑在交易內、**拿得到剛處理的 entity**,但一拋例外就 rollback、且只在驗證通過時跑。**通知 / 推送這類副作用非掛 `afterTransBuilder` 不可**:pipe 還在交易內時你**不知道後續會不會 rollback**,要「確定 commit 成功才通知」就只能等交易外(否則可能通知了之後被 rollback 掉、不存在的資料)。次要好處:長 async I/O(SignalR、寄信、call 外部 API)不卡在交易內 hold 著 DB 鎖。`afterTransBuilder` 在**交易外、commit 後**跑,但它的 pipe **從 `HttpRequest` 起步、不直接給你那筆 entity**。afterTrans 要拿 entity 兩條路:
>
> - **`GetRequestModel<T>()`** 讀回 controller 早先 model-bind 好的那筆 input(`InputRequestContext<T>.Input`,**不是**重新反序列化 body)——拿到**送進來的形狀**(新增時還沒有 DB 配的 `Id` / audit)。
> - 主管線(交易內)`.RequestData().Set("k")` → afterTrans `.RequestData().GetIfExists<T>("k")`——把**存好的 entity**(含 `Id` / audit)帶過去;需先 `Services.AddRequestData()`(平台預設不註冊)+ 套件 `Hcs.Extensions.RequestData`。
>
> **何時跑 / null-guard:** afterTrans 只在 **commit 成功後**跑——主管線拋例外 → rollback **且 afterTrans 不跑**(不會拿到被 rollback 的 entity);但**驗證 400 不是例外**、交易照 commit、afterTrans **照跑**,此時 `pass` 分支沒執行,`GetIfExists` 回 **`null`**(`GetIfExists` 本就回 `default`,要 null-guard)。對應 entity API 的 `OnBeforeTransaction` / `OnAfterTransaction`,語意見 [entity-api](entity-api.md)。

---

## pipe 組件清單

組 pipeline 用的就是這些 extension。**module 檔通常兩個 `using` 都要**:`Hcs.Platform.Data` = `AddXxxFlowApi` 入口、`SaveChildsFor` / `QueryChildFor` / `PipeIfRole`、`ValidOdata`;`Hcs.Platform.Flow` = 取輸入 / CRUD / `ApplyOdataFilter` / `RunValidation` / 收尾組件。(`SwitchCase` 宣告在 `namespace System`,靠隱含的 `using System` 即可,不需額外 `using`。)`.Pipe(...)` / `.SwitchCase(...)` 這兩個 compose 基元見 [pipe](pipe.md)。

### 取輸入

| 組件 | 型別轉換 | 作用 |
|---|---|---|
| `GetRequestModel<TEntity>()` | `HttpRequest → TEntity` | 取回 model-bind 的 input(`InputRequestContext<T>.Input`) |
| `GetRequestKey<TKey>()` | `HttpRequest → TKey` | 取路由主鍵 |
| `GetKeyAndModel<TKey,TEntity>()` | `HttpRequest → GetKeyAndModelOutput` | 主鍵 + model 都要時;回傳物件帶 `.Key` / `.Entity`(**不是** ValueTuple,別解構) |

### 查 / 存(走 `ITable<T>`,自動套多租戶過濾 + audit/OrgId 自動蓋)

| 組件 | 型別轉換 | 作用 |
|---|---|---|
| `GetData<TKey,TEntity>(query?)` | `TKey → TEntity` | 依主鍵載一筆;`query` 選參可塑形 `IQueryable`(`Include`、加條件) |
| `Query<TEntity>()` | `HttpRequest → IQueryable<TEntity>` | 起一條已過租戶 filter 的查詢 |
| `CreateData<TEntity>()` | `TEntity → TEntity` | 新增寫入 |
| `UpdateData<TEntity>()` | `TEntity → TEntity` | 更新寫入 |
| `DeleteData<TEntity>()` | `TEntity → TEntity` | 刪除 |
| `SetModelKey<TKey,TEntity>()` | `TEntity → TEntity` | 把路由主鍵蓋到 model 上（Put 用） |
| `SaveChildsFor<…>(x => x.Childs)` | `T → T` | diff 同步子集合:比對 DB 既有與傳入，自動 新增/更新/刪除(**傳入子集合為 `null` 會 throw**;PUT 客戶端若把空集合序列化成 null 要注意) |
| `QueryChildFor<…>(x => x.Childs)` | `T → T` | 載入導覽子集合 |

> 這些 `*Data` 步驟操作的是 entity 型別層級的 `ITable<T>`,所以**寫入自動跑 audit / OrgId 蓋值、查詢自動過多租戶 filter**——跟內建 CRUD 同一層橫切,見 [data-pipes](data-pipes.md)。要拔掉預設寫入自己存,在 entity API 用 `UseDefaultSave(false)`;在 Flow API 則是「不串 `CreateData/UpdateData`、自己在 `.Pipe` 裡存」。

### 查詢輸出(Query 專用)

| 組件 | 型別轉換 | 作用 |
|---|---|---|
| `ValidOdata<TEntity>()` | OData 查詢字串驗證(放白名單 expand 等) |
| `ApplyOdataFilter<TEntity>()` | `IQueryable → object[]` | 套 `$filter/$select/$orderby/$expand` |
| `QueryOutout<T>(settings)` | 匯出處理(Excel 等)→ `IActionResult` |

### 驗證閘門

```csharp
b.GetRequestModel<Invoice>()
 .RunValidation(
     v => v.Pipe((EntityValidationResult<Invoice> r) => { /* r.AddError(...) */ }),  // 驗證 pipe
     pass => pass.Pipe(MyPipes.DoSave).Ok());                                         // 通過才跑
```

`RunValidation(validationBuilder, ifPass)`:跑驗證 pipe,有任何 error → **400**(後段不跑);無 error → 走 `ifPass`。驗證怎麼寫、`Unique` 等內建 validator 見 [validation-errors](validation-errors.md)。

### 收尾成 IActionResult(`Hcs.Platform.Flow`)

| 組件 | 結果 |
|---|---|
| `Ok(outputCurrentValue = true)` | 200,預設把當前值當 body |
| `OkOrNotFound()` | 當前值非 null → 200(預設帶 body);null → 404 |
| `NotFound()` / `BadRequest()` | 404 / 400,預設**不**帶 body(`outputCurrentValue = false`) |
| `StatusCode(code, outputCurrentValue = false)` | 任意狀態碼 |

### 條件分支

- `.SwitchCase(defaultBuilder, cases => cases.AddCase(when, caseBuilder))` — 見 [pipe](pipe.md)。
- `.PipeIfRole(role, doPipe)` — 只有持某權限的呼叫者才跑那段。

---

## 完整範例

**算出來的視圖(Get,唯讀)**——回傳一筆 entity 的衍生資料:

```csharp
var summaryApi = moduleBuilder.AddGetFlowApi<long, Order>("Order.Summary", b =>
    b.GetRequestKey<long>()
     .GetData<long, Order>(q => q.Pipe(x => x.Include(o => o.Lines).AsQueryable()))
     .Pipe(MyPipes.Order.BuildSummary)     // Order → OrderSummary
     .OkOrNotFound());
```

**自訂存檔(Post,帶子集合 + 條件編號)**:

```csharp
var saveApi = moduleBuilder.AddPostFlowApi<long, Order>("Order.Save", b =>
    b.GetRequestModel<Order>()
     .Pipe(MyPipes.Order.CalculateTotal)
     .SwitchCase(
         def => def.UpdateData(),                                        // 有單號 → 更新
         cases => cases.AddCase(o => o.No == null, n => n.Pipe(MyPipes.Order.AssignNo).CreateData()))  // 無 → 配號後新增
     .SaveChildsFor(x => x.Lines)
     .Ok());
```

**自訂查詢(Query,回 OData 列表)**:

```csharp
var pendingApi = moduleBuilder.AddQueryFlowApi<Order>("Order.Pending", b =>
    b.Query<Order>()
     .Pipe((IQueryable<Order> q) => q.Where(o => o.Status == OrderStatus.Pending))
     .ApplyOdataFilter()
     .Ok());
```

---

## 權限綁定(不綁就 403)

Flow API 回傳的 `IRoleToken` **必須**在 `AddModuleFuncion` 裡 `AddRole` 進某個權限,端點才放行(每支都 `[Authorize]` + 對 `name` 做 role 檢查):

```csharp
var saveApi = moduleBuilder.AddPostFlowApi<long, Order>("Order.Save", /* ... */);
var summaryApi = moduleBuilder.AddGetFlowApi<long, Order>("Order.Summary", /* ... */);

moduleBuilder.AddModuleFuncion("Sales", "Order", f =>
{
    f.AddStandardApiRoles(orderEntityApi);              // 標準 CRUD 權限
    f.AddPermission("Edit", p => p.AddRole(saveApi));   // 自訂存檔綁進 Edit 權限
    f.AddPermission("View", p => p.AddRole(summaryApi));// 摘要綁進 View
});
```

- 一個權限可 `AddRole` 多支 token(`p.AddRole(a).AddRole(b)`)。
- 也可把 token 追加到標準權限:`f.AddStandardApiRoles(api, ctx => ctx.View.AddRole(summaryApi))`。
- **token 沒綁進任何權限 → 沒有任何群組能取得它 → 所有人 403**(連 admin `updaterole` 後也拿不到)。
- 加了新權限記得從 localhost 打 `GET /api/console/updaterole` 把它灌進 admin 群組。

權限樹、`AddPermission` / `AddRole` / `$expand` 白名單細節見 [permissions](permissions.md)。

---

## 前端呼叫

Flow API **不是** entity 的 datasource 端點,所以**不走 `@ApiEntry` / `BaseListComponent` 那套**,直接用 Angular `HttpClient` 打 `api/entity/<name>`(`name` = 後端 `AddXxxFlowApi` 的第一參,逐字相同)。平台的 HTTP interceptor 會自動掛 JWT、處理錯誤摘要,消費端不需手動加 token:

```typescript
// 後端 AddPostFlowApi<long, Order>("Order.Save", …)
this.http.post('api/entity/Order.Save', order).subscribe(saved => { /* ... */ });

// 後端 AddGetFlowApi<long, Order>("Order.Summary", …)
this.http.get<OrderSummary>(`api/entity/Order.Summary/${id}`).subscribe(s => { /* ... */ });
```

驗證 400 的錯誤顯示:把 `HttpErrorResponse` 餵給 `ErrorHelper.addHttpError` → `<hcs-error-summary>`,見 [validation-errors](validation-errors.md)。

---

## 何時用 Flow API、何時用 entity hook

| 需求 | 用 |
|---|---|
| 標準 CRUD 加一段副作用 / 驗證(新增後通知、存檔前查重) | entity API 的 `ConfigPostApi(x => x.OnCreated(...))` 等 hook，見 [entity-api](entity-api.md) |
| 不是「標準增刪改查」的動作(結帳、簽核、排序、自訂視圖、批次) | **Flow API** |
| 同一顆 entity 既要標準 CRUD、又要幾支自訂動作 | 兩者並存:`AddEntityApi` + 數支 `AddPostFlowApi`,各自綁權限 |

---

## Canonical reference

- compose 基元(`.Pipe` / `.SwitchCase` / DI 注入)→ [pipe](pipe.md)
- 內建 CRUD 管線與交易語意 → [entity-api](entity-api.md)
- 驗證錯誤產生與顯示 → [validation-errors](validation-errors.md)
- 寫入 / 查詢的橫切(audit / 多租戶)→ [data-pipes](data-pipes.md)
- 權限綁定與 `$expand` 白名單 → [permissions](permissions.md)
