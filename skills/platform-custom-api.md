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
    b.GetRequestModel<Order>()                 // HttpRequest → Order（反序列化 body）
     .Pipe(OrderPipes.CalculateTotal)          // 你的業務步驟
     .RunValidation(                            // 驗證閘門：有錯 → 400，後段不跑
         v => v.Pipe(OrderPipes.EnsureConfirmable),
         pass => pass
            .UpdateData()                       // 走 ITable<T>：自動套租戶過濾 + audit
            .Ok()),                             // → 200 + 回傳值
     afterTransBuilder: a => a.Pipe(OrderPipes.SendNotice));  // 通知掛交易外：不被 rollback 連帶撤掉
```

> **副作用落點是個 footgun。** 串在主管線(`pass` 分支)內的步驟跑在**交易內**——把 `SendNotice` 接在 `UpdateData()` 後面，通知一失敗就 rollback 整筆結帳、且只在驗證通過時跑。**一定要跑(log / 稽核)或不該被 rollback 撤掉(寄送通知)的副作用，掛第三參 `afterTransBuilder`**(交易外、commit 後、含驗證 400 也跑)。`AddXxxFlowApi(name, flowBuilder, preTransBuilder?, afterTransBuilder?)` 的後兩個選用參數就是交易前 / 後 hook。要改交易隔離級別用 `.SetAutoTransaction(enable, isolationLevel)`。

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

- ❌ **為了「不是標準 CRUD」就自己生一個 Controller / 直寫 `DbContext`**——繞過平台 = 連帶繞過租戶過濾、權限、稽核。用 Flow API，留在框架內。
- ❌ **存檔不用 `CreateData/UpdateData` 而在 `.Pipe` 裡自己 `ctx.SaveChanges`**——會跳過 entity 的 audit / OrgId 橫切。要自管存檔是刻意決定，不是預設。
- ❌ **token 忘了 `AddRole`**——端點長出來了卻所有人 403。
- ❌ **把「標準 CRUD + 一段副作用」做成 Flow API**——那只要 `ConfigPostApi(x => x.OnCreated(...))`，別整條管線重寫。

---

## Canonical reference

- 組件清單 / 型別轉換 / 交易語意 / 完整範例 → [core/flow-api](../core/flow-api.md)
- compose 基元（`.Pipe` / `.SwitchCase` / DI 注入）→ [core/pipe](../core/pipe.md)
- 驗證錯誤 → [core/validation-errors](../core/validation-errors.md)
- 權限綁定 → [core/permissions](../core/permissions.md)
- 標準 CRUD 與 hook → [platform-add-entity](platform-add-entity.md) · [core/entity-api](../core/entity-api.md)
