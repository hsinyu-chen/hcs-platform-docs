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
(`TEntity` 若實作 `IPlatformEntity`,查詢會自動 `Include` 建立者 / 最後更新者。)

**Query(列表 / OData)**
```
OData 驗證 → 查詢 → [OnQueryed] → (投影) → 套用 $filter/$select/$orderby/$expand → [OnOdataFiltered] → 匯出處理 → 回應
```

### 關鍵語意

- **`OnValidate` 是閘門**:收集到任何錯誤就回 **400**(`hcs-error-summary` 顯示,見 [validation-errors](validation-errors.md)),**後面的寫入 / 回應步驟完全不跑**。靠的是 [pipe](pipe.md) 的 `SwitchCase`。
- **生命週期成對**:`On{X}ing` 在內建動作**之前**、`On{X}ed` 在**之後**(Creating/Created、Updating/Updated、Deleting/Deleted)。`OnModelGeted` 在 model 反序列化後、`OnKeyGeted` 在主鍵取出後、`OnQueryed` 用來改 `IQueryable`(加條件)。
- **`UseDefaultSave(false)`**:拔掉內建的寫入 / 更新 / 刪除步驟——這時你得在 `On{X}ing` 自己處理持久化(用於非標準存檔)。

---

## 交易範圍(重要)

每個請求開一個獨立 DI scope(request-scoped `DbContext`、驗證 context、pipeline state)。**寫入(Post / Put / Delete)整條管線跑在單一 DB 交易內;Get 不開交易(唯讀)。**

- 交易在 scope 內首次用到 `DbContext` 時惰性開啟(預設隔離層級 `ReadCommitted`)。
- **整條 pipe**(`OnValidate` → 內建寫入 → `On{X}ed`)都在交易內:**任何一步拋例外就整筆 rollback**,不留半套資料;`OnValidate` 有錯(回 400)也視為失敗 → rollback。全部跑完才 **commit**。多個 `DbContext` 時 scope 結束一起 commit / rollback。

每個 `ConfigXxxApi` 都可掛四個共用 hook:`OnRequest` / `OnResponse`(pipe 頭尾),以及 **`OnBeforeTransaction`(交易前)/ `OnAfterTransaction`(commit 成功後)——這兩個在交易外**。

> 實務鐵則:**有外部副作用、不該被 rollback 的事(寄信、推播、丟 queue、呼叫外部 API)放 `OnAfterTransaction`**——它只在 commit 成功後跑。別放 `OnCreated` / `OnUpdated`:那在交易內,後續若 rollback,信已經寄出去收不回。要跟資料一起 commit / rollback 的 DB 寫入才放 `On{X}ed`。

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

不想要整套 CRUD、只要一支自訂端點時,用 `AddGetFlowApi` / `AddPostFlowApi` / `AddPutFlowApi` / `AddQueryFlowApi`——直接給一條 pipe(內建 `ApplyOdataFilter().Ok()` 等組合子),產出單一端點而不建立 entity 的五件套。
