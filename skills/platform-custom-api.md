---
name: platform-custom-api
description: 在 HCS Platform 模組裡加一支「非標準 CRUD」的自訂端點——結帳、簽核、排序、批次、回傳算出來的視圖等。用 AddPostFlowApi / AddGetFlowApi / AddQueryFlowApi 等自組一條 pipe 管線、綁權限、前端用 HttpClient 打 api/entity/<name>。當需求不是「對一顆 entity 做標準增刪改查」時使用。
---

# 加一支自訂端點（Flow API）

## When to use

需求**不是**「對一顆 entity 做標準增刪改查」時：結帳、送簽、排序、批次匯入、回傳一個算出來的視圖、跨多表的動作…。

**先判斷別走錯路：**

| 你的需求 | 用 | 看哪 |
|---|---|---|
| 標準 CRUD，只是某一步要插邏輯（存檔前查重、新增後通知） | **不是這篇**——`AddEntityApi` 的 `ConfigXxxApi` 掛 hook | [platform-add-entity](platform-add-entity.md) 的「常見客製」 |
| 一個**不是**標準增刪改查的動作 | **Flow API**（本篇） | 繼續往下 |

> 心智模型：內建 `AddEntityApi` 本身就是用同一組 pipe 組件預組好的 Flow API。自訂端點不是另一套機制，是把那條鏈拆開、換成你要的步驟。整套組件與語意見 [core/flow-api](../core/flow-api.md)。

以下假設模組 `Sample` 已存在（見 [platform-create-module](platform-create-module.md)），要加一支訂單結帳 `Order.Confirm`。

---

## 後端

### 1. 選動詞、組管線

在模組的 `Build` 裡，`moduleBuilder` 上呼叫 `Add{Post,Get,Put,Delete}FlowApi<TKey, TEntity>` 或 `AddQueryFlowApi<TEntity>`。`flowBuilder` 給你一條從 `HttpRequest` 起步的 pipe，**疊步驟、最後收尾成 `IActionResult`**：

```csharp
var confirmApi = moduleBuilder.AddPostFlowApi<long, Order>("Order.Confirm", b =>
    b.GetRequestModel<Order>()                 // HttpRequest → Order（取回 model-bind 的 input）
     .Pipe(OrderPipes.CalculateTotal)          // 你的業務步驟
     .RunValidation(                            // 驗證閘門：有錯 → 400，後段不跑
         v => v.Pipe(OrderPipes.EnsureConfirmable),
         pass => pass.UpdateData().Ok()),       // 交易內：走 ITable<T>（自動租戶過濾 + audit）存檔
     afterTransBuilder: a => a.GetRequestModel<Order>()   // 交易外、commit 後：取回 model-bind 的 Order（confirm 既有單，input 帶 Id）
         .Pipe(OrderPipes.SendNotice));         // 外部推送擺這：不 hold 鎖、也不會推到 rollback 掉的資料
```

> **副作用落點是個 footgun。** 主管線(`pass` 分支)內的步驟跑在**交易內**、拿得到剛處理的 entity——但任一步 throw 會 rollback、且只在驗證通過時跑。第四參 `afterTransBuilder` 在**交易外、commit 後**跑(**含驗證 400 也跑**)。**通知 / 推送這類副作用非放這裡不可**:pipe 還在交易內時你**不知道後續會不會 rollback**——要「**確定真的存進去了才通知**」就只能等 commit 之後(在交易內通知,萬一 rollback 就通知了不存在的資料,接收端 re-fetch 也讀不到未 commit 的列)。次要好處:長 async I/O(SignalR 推送、寄信、call 外部 API)不會卡在交易內 hold 著 DB 鎖、拖垮並行。一定要跑的 log / 稽核也放這。它的 pipe 從 `HttpRequest` 起步、**不直接給你那筆 entity**(實測:在 afterTrans 寫 `.Pipe((Order o) => …)` 編譯就不過,框架把 `Order` 當要注入的服務)。afterTrans 要拿 entity 有兩條路:
>
> - **`GetRequestModel<T>()`** 讀回 controller 早先 model-bind 好的那筆 input(`InputRequestContext<T>.Input`,**不是**重新反序列化 body)——最簡單,但拿到的是**送進來的形狀**(新增時還沒有 DB 配的 `Id` / audit)。
> - 主管線(交易內)`.RequestData().Set("k")` → afterTrans `.RequestData().GetIfExists<T>("k")`——把**存好的 entity**(含 `Id` / audit)跨交易邊界帶過去;需先 `moduleBuilder.Services.AddRequestData()`(平台預設不註冊)+ 套件 `Hcs.Extensions.RequestData`。
>
> **(a) afterTrans 讀回 model-bind 的 input**(送進來的形狀,新增時沒有 `Id`/audit):
> ```csharp
> afterTransBuilder: a => a.GetRequestModel<Order>().Pipe(OrderPipes.AuditConfirm)
> ```
> **(b) 端到端接力**——交易內 `.Set` 存好的 entity、afterTrans `.GetIfExists` 取回(拿得到 `Id`/audit),再推播 / 發信。接力骨架與 `IEmailMessage` 副作用皆經實測,完整形狀:
> ```csharp
> // 前置:moduleBuilder.Services.AddRequestData();   // 平台預設不註冊
> moduleBuilder.AddPostFlowApi<long, Order>("Order.Confirm", b =>
>     b.GetRequestModel<Order>()
>      .Pipe((Order o) => { o.Total = Math.Ceiling(o.Total); })
>      .RunValidation(
>          v => v.Pipe((EntityValidationResult<Order> r) =>
>          {
>              if (r.Data.Total < 0) r.AddError(new() { Title = "Total", Message = "negative" });
>          }),
>          pass => pass.UpdateData().RequestData().Set("saved").Ok()),   // 主管線:存好的 entity 入 context
>     afterTransBuilder: a => a.RequestData().GetIfExists<Order>("saved")  // 交易外:取回交易內 Set 的 entity
>         .Pipe(async (IEmailMessage email, Order o) =>
>         {
>             if (o != null)                                              // ← 驗證 400 時 o 為 null,必須 guard(見下)
>                 await email.SendAsync($"訂單 {o.Id} 已確認", "…", new[] { "manager@example.com" });
>             return (object)o;                                           // raw afterTrans 須以 object 收尾
>         }));                                                            // SignalR 推播 / 多通道發訊見 core/realtime-and-messaging
> ```
>
> **(b) 的 null-guard:** afterTrans 只在 **commit 成功後**跑(主管線拋例外 → rollback **且 afterTrans 不跑**,所以不會拿到被 rollback 的 entity)。但**驗證 400 不是例外**(交易照 commit),此時 `pass` 分支沒跑 → `Set` 沒寫 → `GetIfExists` 回 **`null`**,afterTrans 的 pipe **必須 null-guard**(實測 400 時 afterTrans 收到 null)。
>
> **`pre` / `afterTransBuilder` 的 pipe 要以 `object` 收尾**——它們不回 HTTP response、`DIPipe` 又沒有 void 輸出型別,結尾補個 `.Pipe(x => (object)x)` 即可(raw FlowApi 不像 entity hook 會自動補,結尾停在 entity 型別會編譯不過)。`AddXxxFlowApi(name, flowBuilder, preTransBuilder?, afterTransBuilder?)`;改交易隔離級別用 `.SetAutoTransaction(enable, isolationLevel)`。

**常用組件**（`AddXxxFlowApi` 入口與 `SaveChildsFor` / `QueryChildFor` / `PipeIfRole` 在 `Hcs.Platform.Data`；取輸入 / CRUD / OData / 驗證 / 收尾組件在 `Hcs.Platform.Flow`——兩個 `using` 通常都要；`SwitchCase` 在 `namespace System`，免額外 `using`）：

- 取輸入：`GetRequestModel<T>()`（body）、`GetRequestKey<TKey>()`（路由主鍵）。
- 查 / 存（走 `ITable<T>`，自動租戶過濾 + audit/OrgId 蓋值）：`GetData<TKey,T>(q => …)`、`Query<T>()`、`CreateData<T>()`、`UpdateData<T>()`、`DeleteData<T>()`。
- 子集合：`SaveChildsFor(x => x.Lines)`（diff 同步：自動增/改/刪）、`QueryChildFor(x => x.Lines)`。
- 查詢輸出：`ApplyOdataFilter<T>()`、`QueryOutout(settings)`。
- 驗證：`RunValidation(validationBuilder, ifPass)`。
- 收尾：`Ok()` / `OkOrNotFound()` / `NotFound()` / `BadRequest()` / `StatusCode(code)`。
- 分支：`.SwitchCase(...)`、`.PipeIfRole(role, …)`。

`.Pipe(...)` 怎麼寫（inline / static / extension）、參數怎麼自動 DI → [core/pipe](../core/pipe.md)。完整組件清單與型別轉換表 → [core/flow-api](../core/flow-api.md)。

> **要回一個列表(OData 視圖)用 `AddQueryFlowApi<TEntity>`**，形狀跟上面的 Post 不一樣:沒有 `TKey`，收尾走 `b.Query<T>().Pipe(q => q.Where(...)).ApplyOdataFilter().Ok()`，前端是 `GET api/entity/<name>?$filter=…`(query string、無 `{id}` 區段)。別把 Post 範例硬套上去。範例見 [core/flow-api](../core/flow-api.md)。

> `name`（第一參）直接決定前端 URL = `api/entity/<name>`，且是權限檢查的 key。慣例用 `功能.動作`，如 `"Order.Confirm"`。

### 1.5 唯讀查詢端點（SQL View / 對照表當資料源）

要回的列表**不是**某顆 entity 的 CRUD 視圖,而是一個 **join 多表算出來的唯讀投影**(員工＋部門名、訂單＋客戶＋幣別…),典型做法是一支 **SQL View + keyless entity + `AddQueryFlowApi`**。前端拿它當下拉 / picker 的資料源,直接秀 join 出來的欄位,不必自己再查一次關聯表。

**1) keyless entity**——只有欄位、**不繼承 `BaseModel`**(沒有 `Id`/audit/`OrgId`,它不是一顆可寫的 entity):

```csharp
namespace Sample.Models;

public class VwEmployeeDirectory
{
    public long EmployeeId { get; set; }
    public string EmployeeNo { get; set; } = "";
    public string Name { get; set; } = "";
    public string? DepartmentName { get; set; }   // LEFT JOIN 可能為 null → string?（型別陷阱見 platform-add-entity）
}
```

**2) ModelConfig 對映到 view**——`HasNoKey().ToView(...)`,EF 只讀不建表:

```csharp
modelBuilder.Entity<VwEmployeeDirectory>().HasNoKey().ToView("VwEmployeeDirectory");
```

**3) 手寫 migration 建 view**——EF 不會替 `ToView` 產 DDL,自己在 migration 裡 `Sql("CREATE VIEW ...")`(`Down` 對應 `DROP VIEW`):

```csharp
migrationBuilder.Sql(
    "CREATE VIEW VwEmployeeDirectory AS " +
    "SELECT e.Id AS EmployeeId, e.EmployeeNo, e.Name, d.Name AS DepartmentName " +
    "FROM Employee e LEFT JOIN Department d ON e.DepartmentId = d.Id");
```

**4) `AddQueryFlowApi` 起頭 pipe**——資料源用 `db.Set<T>().AsNoTracking()`。**pipe 第一步的 lambda 必須帶 `HttpRequest` 參數**(即使沒用到):單一服務注入時省掉 `input` 編譯不過(框架把唯一參數當「輸入」而非「注入服務」),所以簽名寫成 `(PlatformDbContext db, HttpRequest _)`:

```csharp
var dirApi = moduleBuilder.AddQueryFlowApi<VwEmployeeDirectory>("Sample.EmployeeDirectory", b =>
    b.Pipe((PlatformDbContext db, HttpRequest _) => db.Set<VwEmployeeDirectory>().AsNoTracking())
     .ApplyOdataFilter()
     .Ok());
```

**權限判準**跟一般 Flow API 同一套(見下節):是**某頁專屬的輔助查詢**(如發票頁的經辦員工 picker)就追加進該頁的 View——`f.AddStandardApiRoles(invoiceApi, ctx => ctx.View.AddRole(dirApi))`,self-contained、免另開權限也免 `updaterole`;若是**任何人都能查的通用下拉**才 `moduleBuilder.Everyone.AddRole(dirApi)`。

**前端**照一般 entity model 宣告即可——`@ApiEntry` 綁**同名**端點(逐字等於後端第一參)、`@Key` 挑 view 的識別欄(view 沒有 `Id`,自己指一個唯一欄):

```typescript
@ApiEntry('api/entity/Sample.EmployeeDirectory')
export class VwEmployeeDirectory {
  @Key() EmployeeId: number;      // view 無 Id，挑一個唯一欄當 key
  @Property() EmployeeNo: string;
  @Property() Name: string;
  @Property() DepartmentName: string;   // join 出來的欄，picker columns 直接秀
}
```

picker `columns` 直接列 `DepartmentName` 這種 join 欄,不用前端再打一次部門 API(reference picker 組法見 [frontend/form](../frontend/form.md))。

> **別為此自寫 controller。** 「join 多表回唯讀投影」很容易讓人想自幹一支 controller 直查 `DbContext`——那會繞過 OData `$filter`/`$orderby`、租戶過濾與權限。view + `AddQueryFlowApi` 讓這條查詢**留在框架內**:前端照樣 `?$filter=…`、picker 照樣分頁排序。要租戶隔離就在 view 的 SQL 裡帶 `OrgId` 欄、entity 加 `OrgId` 屬性,讓平台 filter pipe 接手。

### 2. 綁權限（不綁就 403）

`AddXxxFlowApi` 回傳的 `IRoleToken` **必須** `AddRole` 進某權限，端點才放行：

```csharp
moduleBuilder.AddModuleFuncion("Sample", "Order", f =>
{
    f.AddStandardApiRoles(orderEntityApi);              // 標準 CRUD（若有）
    f.AddPermission("Confirm", p => p.AddRole(confirmApi));   // 自訂端點綁進新權限
});
```

- token 沒綁進任何權限 → 所有人 403（連 admin `updaterole` 後也拿不到）。
- **不一定要開新權限**：頁面專屬的輔助查詢傾向追加進該功能的標準權限（`AddStandardApiRoles(api, ctx => ctx.View.AddRole(token))`，self-contained）；只有「要能單獨給/不給某群組」的動作才開新 `AddPermission`。判準四情境 → [core/permissions](../core/permissions.md) 的「多支 API 怎麼分權限」。
- 加了新權限，從 **localhost** 打一次 `GET /api/console/updaterole` 灌進 admin 群組。
- 權限樹 / `AddPermission` / `AddRole` 細節 → [core/permissions](../core/permissions.md)。

---

## 前端

Flow API **不走** `@ApiEntry` / `BaseListComponent` 那套（那是 entity datasource 專用）。直接用 Angular `HttpClient` 打 `api/entity/<name>`（`name` 逐字等於後端第一參）。平台 HTTP interceptor 自動掛 JWT、處理錯誤摘要：

```typescript
// 後端 AddPostFlowApi<long, Order>("Order.Confirm", …)
this.http.post('api/entity/Order.Confirm', order).subscribe(confirmed => { /* ... */ });

// 後端 AddGetFlowApi<long, Order>("Order.Summary", …)
this.http.get<OrderSummary>(`api/entity/Order.Summary/${id}`).subscribe(s => { /* ... */ });
```

驗證 400 的錯誤顯示：把 `HttpErrorResponse` 餵給 `ErrorHelper.addHttpError` → `<hcs-error-summary>`，見 [core/validation-errors](../core/validation-errors.md)。

---

## 禁忌

- ❌ **為了「不是標準 CRUD」就自己生一個 Controller / 直寫 `DbContext`**——繞過平台 = 連帶繞過租戶過濾、權限、稽核。用 Flow API，留在框架內。**限定於平台內（後台）功能**；對外 C 端另有做法，見下節。
- ❌ **存檔不用 `CreateData/UpdateData` 而在 `.Pipe` 裡自己 `ctx.SaveChanges`**——會跳過 entity 的 audit / OrgId 橫切。要自管存檔是刻意決定，不是預設。
- ❌ **token 忘了 `AddRole`**——端點長出來了卻所有人 403。
- ❌ **把「標準 CRUD + 一段副作用」做成 Flow API**——那只要 `ConfigPostApi(x => x.OnCreated(...))`，別整條管線重寫。

---

## 對外（C 端）介面怎麼做

上面「禁自寫 Controller」的鐵則,對象是**平台內(後台)功能**——那些走平台權限樹、多租戶組織、後台使用者 JWT 的端點。**對外的 C 端**(會員 App、公開網站、合作夥伴 API)本質不同:它的使用者不是後台 `PlatformUser`、不該套後台的組織階層權限、通常要自己的登入與 token 生命週期。**這種介面反而建議跳出平台的 entity/flow 框架**:

- **另開一個專用 project(或至少自訂 controller)** 承載 C 端端點,與後台模組分開。
- **掛自己的 authentication scheme**(專屬的 JWT bearer scheme),與平台後台的驗證/權限系統**完全分離**——C 端 token 不帶後台 role claim,也不走 `AddModuleFuncion` 那套權限樹。

這樣後台與 C 端各管各的 auth、各自演進,不會為了「讓會員 App 能打」而把公開端點硬塞進後台權限、或反過來讓 C 端 token 汙染後台的角色注入。**邊界一句話**:繞過平台框架的禁令只適用於平台內功能;C 端本來就該另立門戶,別把兩套 auth 混在一起。

---

## Canonical reference

- 組件清單 / 型別轉換 / 交易語意 / 完整範例 → [core/flow-api](../core/flow-api.md)
- compose 基元（`.Pipe` / `.SwitchCase` / DI 注入）→ [core/pipe](../core/pipe.md)
- 驗證錯誤 → [core/validation-errors](../core/validation-errors.md)
- 權限綁定 → [core/permissions](../core/permissions.md)
- afterTrans 推播 / 發訊落點 → [core/realtime-and-messaging](../core/realtime-and-messaging.md)
- 標準 CRUD 與 hook → [platform-add-entity](platform-add-entity.md) · [core/entity-api](../core/entity-api.md)
