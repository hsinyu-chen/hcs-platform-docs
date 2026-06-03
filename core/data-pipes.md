# Data Pipes & ITable(資料切面與多租戶過濾)

平台在**每個 entity 的讀寫路徑上**埋了兩條跨切面 hook,不管哪支 API、甚至手動存取都會自動套:

- **寫入切面 `IDataPipe<T>`**:在 Create / Update / Delete 的**前(Pre)後(Post)**跑——蓋 audit 欄位、蓋 OrgId、寫稽核軌跡。
- **查詢切面 `IDataFilterPipe<T>`**:每次查詢都先過一遍,往 `IQueryable` 加條件——**多租戶過濾就靠這個**。

兩者由**執行載體 [`ITable<T>`](#執行載體-itablet)** 串起來。最該記住的非直覺點:**查詢一律過 filter pipe(過濾是自動且隱形的)、寫入一律跑 Pre/Post**——這在 code 裡看不到呼叫點,卻每次都發生。

> ⚠️ 名詞釐清:`DataPipeType` 只列 **6 個寫入 phase**(`PreCreate`/`PostCreate`/`PreUpdate`/`PostUpdate`/`PreDelete`/`PostDelete`)。**「Query」不是 `DataPipeType` 的成員**——查詢側是另一條 `IDataFilterPipe<T>` 介面。寫入用 `IDataPipe`,查詢用 `IDataFilterPipe`,兩套不同型別。

---

## 兩種 pipe

### 寫入:`IDataPipe<T>`

```csharp
public interface IDataPipe<TEntity> where TEntity : class
{
    Task<EntityEntry<TEntity>> Pipe(DataPipeType type, EntityEntry<TEntity> entityEntry);
    DataPipeType Type { get; }   // 這支 pipe 關心哪些 phase(可多個)
}
```

- 拿到的是 EF Core 的 **`EntityEntry<T>`**(不只 entity 本身,還能改 `Property(x => x.Foo).IsModified`、看 `State`)。
- `Type` 是 **`[Flags]`**:一支 pipe 可宣告 `PreCreate | PreUpdate` 表示兩個 phase 都要跑。框架對每個 phase 用 `Type.HasFlag(phase)` 決定要不要呼叫這支 pipe。

### 查詢:`IDataFilterPipe<T>`

```csharp
public interface IDataFilterPipe<TEntity>
{
    Task<IQueryable<TEntity>> Query(IQueryable<TEntity> query);   // 往 query 加條件後回傳
}
```

每條 filter 收到上一條的結果、回傳加了條件的 `IQueryable`,鏈式疊加(不是 EF Core 的 `HasQueryFilter`,是平台自己的管道)。

---

## 執行載體 `ITable<T>`

`ITable<T>` 包住 `DbContext` + 該 entity 的 pipe provider,是「**會自動套 pipe 的 DbSet**」:

| 方法 | 做什麼 | 何時跑 pipe |
|---|---|---|
| `QueryAsync()` | `DbContext.Set<T>()` **過所有 filter pipe** 後回傳 | 查詢時,**每次** |
| `GetAsync(keys)` | `QueryAsync()` + 主鍵篩選取單筆 | 同上(單筆也過濾) |
| `AddAsync(model)` | 設 `State=Added`、**立刻跑 `PreCreate`**、記入待存清單 | **呼叫當下**(非 SaveChanges 時) |
| `UpdateAsync(model)` | 設 `State=Modified`、**立刻跑 `PreUpdate`**、記入待存清單 | 呼叫當下 |
| `DeleteAsync(model)` | 設 `State=Deleted`、**立刻跑 `PreDelete`**、記入待存清單 | 呼叫當下 |
| `SaveChangesAsync()` | `DbContext.SaveChanges()` → **跑各筆 `Post{Create/Update/Delete}`** | SaveChanges **之後** |

> 另有雙泛型 `ITable<T, TDbContext>`,暴露 typed 的 `TDbContext`(`ITable<T>` 用預設 `DbContext`)——模組自帶多個 DbContext、要指明操作哪一個時用它。

### gotcha

1. **Pre 在 `AddAsync` 當下就跑,不是等到 `SaveChanges`。** 所以 OrgId / audit 在你呼叫 `AddAsync` 那一刻就蓋好了;之後再改 entity,Pre 已經跑過。順序:`AddAsync`(蓋好)→ 改 → `SaveChanges`(寫 DB + 跑 Post)。

2. **繞過 `ITable` 直接用 `DbContext` = 繞過所有 filter / pipe。** `dbContext.Set<Customer>().Where(...)` **不會**套多租戶過濾,也不會跑 audit——這是多租戶最大的 footgun。要套切面就走 `ITable`(或走 entity-api,它內部就是 `ITable`)。

3. **同步 `SaveChanges()` 會 block 在 async 的 Post pipe 上**(內部 `GetAwaiter().GetResult()`)。能 async 就用 `SaveChangesAsync()`。

4. **同一個 `SaveChanges` 內混多種變更時,Post 順序固定為 Delete → Update → Create**,不是按你呼叫 `AddAsync`/`UpdateAsync`/`DeleteAsync` 的時間序(批次操作才會踩到;entity-api 單請求通常只一種變更不受影響)。

---

## 與 entity-api 的關係

[entity-api](entity-api.md) 五支端點的**內建 CRUD 步驟就是操作 `ITable`**:`CreateData` = `table.AddAsync` + `table.SaveChangesAsync`、`QueryData` = `table.QueryAsync`、`GetData` = `table.GetAsync`…。所以 `AddEntityApi` 一掛上,該 entity 的所有 data pipe 自動生效,**不必每支 API 重接**。

把兩種擴充點疊在一起看(以 Post 為例):

```
[OnCreating] → CreateData{ AddAsync→跑 PreCreate ; SaveChanges→跑 PostCreate } → [OnCreated]
   (entity-api hook)         (data pipe,entity-type-wide)              (entity-api hook)
```

- **Query**:`table.QueryAsync()` **先**套 filter pipe,**才**輪到 `[OnQueryed]`。所以你的 `OnQueryed` 拿到的 `IQueryable` **已經過完多租戶過濾**,你只是在已過濾的基礎上再加條件。
- data pipe 全程在 entity-api 的**交易內**(Pre 在寫入前、Post 在寫入後 commit 前),交易語意見 [entity-api](entity-api.md#交易範圍重要)。

**Data pipe 與 entity-api hook 怎麼選:**

| | data pipe(`IDataPipe`/`IDataFilterPipe`) | entity-api hook(`OnCreated`/`OnQueryed`…) |
|---|---|---|
| 範圍 | **entity 型別層級**:該型別所有 API + 手動 `ITable` 都套 | **單一 API 的單一端點** |
| 註冊 | `ConfigDataPipe<T>(...)`(模組設定期) | `ConfigPostApi(x => x.OnCreated(...))` |
| 用途 | 跨 entity / 跨 API **一律要套**的橫切:稽核、多租戶、自動蓋欄位 | 只有這一支 API 才需要的商業邏輯 |

口訣:**「忘了接就會出事」的橫切 → data pipe**(因為它不靠人記得在每支 API 接);**只屬於某支 API 的 → hook**。

---

## 平台內建的三支 pipe

平台對兩個契約介面用 **open generic** 註冊,自動套到**每個實作該介面的 entity**:

| pipe | 套在 | phase | 做什麼 |
|---|---|---|---|
| `PlatformEntityPipe<T>` | `IPlatformEntity` | PreCreate / PreUpdate | 蓋 `CreatedBy`/`CreatedTime`(建立)、`LastUpdatedBy`/`LastUpdatedTime`(更新);更新時鎖住 `CreatedBy`/`CreatedTime` 與標了 `[IgnoreOnUpdate]` 的欄位不被改 |
| `OrganizedPipe<T>` | `IOrganized` | PreCreate / PreUpdate | 蓋 `OrgId` 為當前使用者組織;更新時**鎖 `OrgId` 不被改**(防把資料搬到別的組織),除非開了子組織授權且目標在可及範圍 |
| `OrganizedDataFilter<T>` | `IOrganized` | (查詢) | 查詢時自動 `Where(x => x.OrgId == 使用者組織)`;依 `OrganizationDataFilterSetting` 的 `AllowChildOrganizationData` / `AllowParentsData` **各自獨立** OR 加掛子組織 / 上層組織範圍 |

所以你的 entity 只要 `: IOrganized`,**完全不必寫任何過濾 code**,查詢自動只回自己組織的資料、寫入自動蓋 OrgId、且不能被搬走。這正是多租戶「隱形」的來源。

> `OrganizationDataFilterSetting` 是 **scoped 且可變**的:hook 裡設 `setting.AllowChildOrganizationData = true`(看子組織)或 `setting.AllowParentsData = true`(看上層組織)可在這個請求內放寬查詢範圍(`AllowChildOrganizationData(role)` 的底層機制)。

---

## 自訂:`ConfigDataPipe`

在模組 `Build` 裡用 `moduleBuilder.ConfigDataPipe<TEntity>(opt => ...)` 掛。三種掛法:

```csharp
// 1) inline delegate —— lambda 參數自動 DI(最常用),適合具體 entity
moduleBuilder.ConfigDataPipe<Customer>(opt =>
    opt.AddFilter(b => b.Pipe((IUserRoleAccessor role, IQueryable<Customer> q) =>   // filter:TIn/TOut 是 IQueryable<T>
        role.Roles.Contains("Customer.ViewAll") ? q : q.Where(c => c.IsPublic)))
       .AddPipe(DataPipeType.PreUpdate, b => b.Pipe((EntityEntry<Customer> e) =>    // 寫入:TIn/TOut 是 EntityEntry<T>,改 e.Entity
        { e.Entity.Code = e.Entity.Code.Trim(); })));

// 2) typed —— 把 pipe 抽成 class(可注入、可重用);filter 版是 AddFilter<T>()
moduleBuilder.ConfigDataPipe<Customer>(opt => opt.AddPipe<MyAuditPipe>());

// 3) open generic —— 套到「所有實作某介面的 entity」,給介面 / 抽象型別用
moduleBuilder.ConfigDataPipe<IMySoftDelete>(opt => opt.AddFilter(typeof(SoftDeleteFilter<>)));
```

`.Pipe(...)` 的參數自動從 DI 注入,寫法(inline / static / extension)與 [pipe](pipe.md) 完全一致。

> ⚠️ **介面 / 抽象 `TEntity` 只能用 open-generic 那條**(`AddPipe(typeof(X<>))` / `AddFilter(typeof(X<>))`)。原因:open generic 會在啟動時對「每個符合約束的具體 entity」各關閉註冊一次(內建三支就是這樣套到每個 `IOrganized` / `IPlatformEntity` entity);其他掛法綁死在 `TEntity` 本身,對介面型別註冊永遠不會命中(provider 是按**具體** entity 型別解析的)。對介面 / 抽象型別用 **inline delegate**(`AddPipe(type, b)` / `AddFilter(b)`)或**閉合型別** `AddPipe(typeof(X))` 會直接丟 `InvalidOperationException`。
>
> 但 **typed 寫法 `AddPipe<T>()` / `AddFilter<T>()` 不會丟例外**——它老實註冊成 `IDataPipe<介面型別>`,然後**靜默失效**(provider 按具體 entity 解析,永遠取不到這條)。比丟例外更難 debug:沒有任何錯誤訊息,pipe 就是不跑。要套到一整類 entity 一律用 open-generic。

### 多租戶 filter 範例

平台內建已照 `OrgId` 過濾;要再加一層(例如「非管理員只看自己建立的」)時:

```csharp
moduleBuilder.ConfigDataPipe<Invoice>(opt => opt.AddFilter(b =>
    b.Pipe((IPlatformUser user, IUserRoleAccessor role, IQueryable<Invoice> q) =>
        role.Roles.Contains("Invoice.ViewAll")
            ? q                                        // 管理員:看全組織(OrgId 過濾已由內建套好)
            : q.Where(x => x.CreatedBy == user.Id))));  // 一般人:疊加「只看自己的」
```

這條 filter 會疊在內建 `OrgId` 過濾**之後**,兩層都生效。

---

## 兩層 provider(進階)

`IDataPipeProvider<T>` 把 pipe 分兩層跑,**scoped 先、DI 後**:

- **DI 層**:`ConfigDataPipe` 在模組設定期註冊的,進程全域、每個請求都在。
- **scoped 層**:`IScopedDataPipeProvider<T>` 的 `ScopedDataPipes` / `ScopedFilterPipes` 是**可變 list**,讓你在請求**執行期動態**往當前 entity 加一條臨時 pipe(只活在這個請求 scope)。

多數情況只用 DI 層(`ConfigDataPipe`)。需要「依執行期狀態臨時加一條 filter / pipe」時才碰 scoped 層。
