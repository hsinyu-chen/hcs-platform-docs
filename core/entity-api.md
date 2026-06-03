# Entity API(AddEntityApi)

一行 `AddEntityApi<TKey, TEntity>` 就為一個 entity 長出五支 RESTful 端點(Get / Query / Post / Put / Delete);每支都是一條由**內建步驟 + 你掛的生命週期 hook** 組成的 [DIPipe](pipe.md) 管線。

---

## 本質

```csharp
var api = b.AddEntityApi<long, Customer>(opt =>
{
    opt.ConfigPostApi(x => x.OnValidate(...).OnCreated(...));
    opt.ConfigQueryApi(x => x.OnQueryed(...));
    // ConfigPutApi / ConfigGetApi / ConfigDeleteApi ...
});
```

- 泛型:`<TKey, TEntity, TDbContext>`,**`TKey` 預設 `long`、`TDbContext` 預設 `PlatformDbContext`**——只給 `<TEntity>` 即用預設。
- 回傳 `EntityApiRoles { Get, Put, Delete, Post, Query }`,五個 token 交給 `AddModuleFuncion(... AddStandardApiRoles(api))` 綁權限。
- `ConfigXxxApi(...)` **可重複呼叫、會疊加**(後面的設定接在前面之後)。
- 也有投影多載 `AddEntityApi<TKey, TEntity, TView>(projection, ...)`:Query 端點回傳投影後的 `TView`。

---

## 每支端點的管線

每支端點 = 一串內建步驟,中間插入你的 hook(各 hook 收一個 [DIPipeBuilder](pipe.md),可疊多段)。`[ ]` 標的是 hook、其餘是內建步驟。

**Post(新增)**
```
取 model → [OnModelGeted] → 驗證閘門[OnValidate] → [OnCreating] → (內建寫入) → [OnCreated] → 回應
```

**Put(修改)**
```
取 model → [OnModelGeted] → 設定主鍵 → [OnKeySet] → 驗證閘門[OnValidate] → [OnUpdating] → (內建更新) → [OnUpdated] → 回應
```

**Delete(刪除)**
```
取主鍵 → [OnKeyGeted] → 查出 entity[OnQueryed] → 驗證閘門[OnValidate] → [OnDeleting] → (內建刪除) → [OnDeleted] → 回應
```

**Get(取單筆)**
```
取主鍵 → [OnKeyGeted] → 查出 entity[OnQueryed] → [OnGeted] → 驗證閘門[OnValidate] → OkOrNotFound
```
(`TEntity` 若實作 `IPlatformEntity`,**只有 Get** 會自動 `Include` 建立者 / 最後更新者。)

**Query(列表 / OData)**
```
OData 驗證 → 查詢 → [OnQueryed] → (投影) → 套用 $filter/$select/$orderby/$expand → [OnOdataFiltered] → 匯出處理 → 回應
```
> ⚠️ Query 列表**不會**自動 Include `IPlatformEntity` 的建立者 / 更新者(只有 Get 會)。列表要帶這些關聯欄位,得靠前端 OData `$expand=CreatedByUser,LastUpdatedByUser`(且該關聯需在 `AllowExpand` 白名單內)。

### 關鍵語意

- **`OnValidate` 是閘門**:收集到任何錯誤就回 **400**(`hcs-error-summary` 顯示,見 [validation-errors](validation-errors.md)),**後面的寫入 / 回應步驟完全不跑**。靠的是 [pipe](pipe.md) 的 `SwitchCase`。
- **生命週期成對**:`On{X}ing` 在內建動作**之前**、`On{X}ed` 在**之後**(Creating/Created、Updating/Updated、Deleting/Deleted)。`OnModelGeted` 在 model 反序列化後、`OnKeyGeted` 在主鍵取出後、`OnQueryed` 用來改 `IQueryable`(加條件)。
- **`UseDefaultSave(false)`**:拔掉內建的寫入 / 更新 / 刪除步驟——這時你得在 `On{X}ing` 自己處理持久化(用於非標準存檔)。

---

## 交易範圍(重要)

每個請求開一個獨立 DI scope(request-scoped `DbContext`、驗證 context、pipeline state)。**寫入(Post / Put / Delete)整條管線跑在單一 DB 交易內;Get / Query 不開交易(唯讀)。**

- 交易在 scope 內首次用到 `DbContext` 時惰性開啟(預設隔離層級 `ReadCommitted`);多個 `DbContext` 時 scope 結束一起處理。
- **rollback 只在「拋例外」時發生**:寫入步驟與 `On{X}ed` 都在交易內,任一步 throw → 整筆 rollback;沒拋例外就 **commit**。
- **驗證失敗(`OnValidate` 回 400)不是例外**:走 `SwitchCase` 回 `BadRequest`、不 throw → 交易照常 commit、`OnAfterTransaction` 一樣跑。**這是刻意的**——`OnAfterTransaction` 無論成敗都跑,正是**記錄 / 稽核能涵蓋每個請求**(含被驗證擋下的)的關鍵。

各 hook 與交易的關係:

| hook | 何時跑 | 拿得到 | 位置 |
|---|---|---|---|
| `On{X}ing` / `On{X}ed` | 驗證**通過**後(success 分支) | entity | 交易內(pre-commit) |
| `OnBeforeTransaction` | 最前 | `HttpRequest` | 交易外(前) |
| `OnAfterTransaction` | 結尾(含驗證 400) | `HttpRequest` | 交易外(commit 後) |

> 副作用放哪,看「要不要無論成敗都跑」:**一定要跑的橫切工作(log / 稽核)→ `OnAfterTransaction`**——它就是為此而設(400 也跑、commit 後跑、參數是 `HttpRequest`)。**只在成功、且需要該筆 entity 的事(寄該筆通知)→ `OnCreated` 等 `On{X}ed`**——只在驗證通過時跑、拿得到 entity,但在 commit **之前**。

> ⚠️ **不是每支端點都吃這些共用 hook**:`OnRequest` 五支都有;`OnResponse` 與兩個 transaction hook 只在 **Post / Put / Delete +(Query 非投影版)** 生效——**Get 沒串**(也無交易)、**Query 投影版未串** transaction hook。

需要自管交易 / 調隔離層級時用 `SetAutoTransaction(enable, isolationLevel)`。

---

## 常見掛法

```csharp
b.AddEntityApi<long, Customer>(opt =>
{
    // 寫入前驗證(唯一性等),見 validation-errors
    opt.ConfigPostApi(x => x.OnValidate(v => v.Unique(u => u.AddProperty(p => p.No))));
    opt.ConfigPutApi (x => x.OnValidate(v => v.Unique(u => u.AddProperty(p => p.No))));

    // 列表只回自己組織的資料(改 IQueryable)
    opt.ConfigQueryApi(x => x.OnQueryed(q => q.Pipe((IUserOrganization org, IQueryable<Customer> query)
        => query.Where(c => c.OrgId == org.OrgId))));

    // 新增後做後續處理
    opt.ConfigPostApi(x => x.OnCreated(c => c.Pipe((INotifier n, Customer e) => n.Notify(e))));
});
```

hook 內怎麼寫(inline / static / extension)、DI 怎麼注入,見 [pipe](pipe.md);驗證錯誤怎麼產生與顯示,見 [validation-errors](validation-errors.md)。

---

## 不綁 entity 的自訂端點

不想要整套 CRUD、只要一支自訂端點時,用 `AddGetFlowApi` / `AddPostFlowApi` / `AddPutFlowApi` / `AddDeleteFlowApi` / `AddQueryFlowApi`——直接給一條 pipe(內建 `ApplyOdataFilter().Ok()` 等組合子),產出單一端點而不建立 entity 的五件套。
