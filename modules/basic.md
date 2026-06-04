# Basic 模組（使用者 / 群組 / 組織）

`Hcs.PlatformModule.Basic`（前端 `@hcs/basic`）提供平台三本柱——**使用者（PlatformUser）／群組（PlatformGroup）／組織（Organization）**——的管理 API 與後台頁面，外加**子組織授權**與**代理登入（Proxy Login）**。這三個 entity 的型別本身定義在 `Hcs.Platform.BaseModels`（見 [entity-api](../core/entity-api.md) 與 [multi-tenant](../core/multi-tenant.md)），Basic 模組做的是「把它們變成可管理的功能」:CRUD 端點、權限碼、後台清單 / 表單頁。

幾乎每個平台 app 都會掛 Basic——它是「誰能登入、屬於哪些群組、落在哪個組織」的管理面。

---

## 啟用（後端）

```csharp
builder.AddBasicModule();
```

掛上後，模組以一段 `IPlatformModule.Build` 宣告註冊:

- **三個 entity API**:`PlatformUser` / `PlatformGroup` / `Organization` 的標準 CRUD（含 OData 查詢、匯出、系統稽核）。
- **子組織授權 entity**:`AuthorizedOrganization`（父→子組織的授權關聯），歸在 `Basic.Organization.Admin` 權限下。
- **一組非 CRUD flow API**:批次更新群組角色 / 使用者群組、取得 OrgKey、子組織查詢 / 批次加入、代理登入。
- **三條資料過濾**:`PlatformUser` / `PlatformGroup` / `Organization` 各掛一條組織過濾，讓清單只回當前組織可見的資料(機制見 [data-pipes](../core/data-pipes.md))。

---

## 權限碼

Basic 註冊三個功能（function），各自帶標準 / 自訂權限:

| 功能碼 | 權限 | 用途 |
|---|---|---|
| `Basic.Organization` | `.View` | 讀組織 + 取 OrgKey |
| | `.Admin` | 子組織授權管理（`AuthorizedOrganization` 增刪查、批次加子組織） |
| `Basic.PlatformUser` | `.View` / `.Modify` | 讀 / 增刪改使用者(`.View` 含查使用者群組) |
| | `.AllowChildOrgData` | 准許跨組織**往下**存取子組織的使用者資料 |
| | `.ProxyLogin` | 代理登入(見下) |
| `Basic.PlatformGroup` | `.View` / `.Modify` | 讀 / 增刪改群組(`.View` 含查使用者群組 / 群組角色,`.Modify` 含批次設角色、設群組成員) |
| | `.AllowChildOrgData` | 准許往下存取子組織的群組資料 |
| | `.Services` | 管理群組可用的「服務權限」(進階,預設不開) |

- `.View` / `.Modify` 是平台標準 CRUD 權限對(讀 vs 增刪改),由標準 API 角色自動產生(見 [permissions](../core/permissions.md))。
- `.AllowChildOrgData` 對應多租戶的「寫只能往下、讀可上下」不對稱(見 [multi-tenant](../core/multi-tenant.md));沒這個權限只能管自己組織的資料。
- ⚠️ **群組的 Get / Query 掛在 `Everyone`**:任何登入者都讀得到群組清單(因為群組到處被當下拉選項用),不受 `Basic.PlatformGroup.View` 限制。`.View` 控的是「進得了群組管理頁」,不是「讀得到群組」。

---

## entity 底層與密碼處理

三個 entity 來自 `BaseModels`(`PlatformUser` / `PlatformGroup` / `Organization`),模型結構、`OrgId` 多租戶契約、audit 欄位等見 [entity-api](../core/entity-api.md) 與 [multi-tenant](../core/multi-tenant.md),此處不重述。Basic 在 API 層額外加的規則:

- **唯一性**:使用者的 `(Account, OrgId)` 與 `(No, OrgId)` 同組織內不可重複(新增 / 修改都驗)。
- **密碼絕不外流**:
  - GET 單筆使用者時 `Password` 一律清成 `null`——前端永遠拿不到密碼欄。
  - 修改使用者時,**未帶 `Password`(值為 `null`)則不覆寫**原密碼——平台前端表單預設就送 `null`。⚠️ 這只認 `null`:若送空字串 `""` 會真的把密碼覆寫成空,別在自訂表單把空欄送成 `""`。
  - 系統稽核紀錄會把 `Password` 遮蔽(censor),不寫進 log。
- **刪除前檢查參照**:
  - 使用者 / 群組:掃所有外鍵參照,被引用則擋下。
  - 組織:**兩段**——先自動移除指向該組織的所有子組織授權(`AuthorizedOrganization` 父 / 子列),再掃模型中**每一個** `IOrganized` entity,只要有任一筆資料的 `OrgId` 等於待刪組織就擋下(回 `HasReferencedData`)。即「先清授權關聯、再檢查 OrgId 歸屬」,不是泛用 FK 檢查。

---

## 子組織授權

`AuthorizedOrganization`(父→子組織授權)歸在 `Basic.Organization.Admin` 權限下,讓組織建立「我可以往下管哪些子組織」的關係(配合多租戶的「寫只能往下」)。除了直接 CRUD,還有幾條相關行為:

- **建立組織時可一鍵授權**:POST 新組織時若帶 `JoinSubOrg = true`,系統自動建一筆「當前使用者組織 → 新組織」的授權——建完新組織它就已經是你的子組織,不必再手動加。
- **批次加子組織**(`Sub.Orgs.BatchAdd`):一次掛多筆子組織授權,已存在的關聯會跳過(不重複建)。`Admin` 權限**只開放查 / 刪 + 這支批次加**,沒開放 `AuthorizedOrganization` 的直接 Post / Put——所有新增都統一走 `Sub.Orgs.BatchAdd`,授權頁也不提供編輯 / 複製。
- **可選子組織清單**(`Sub.Orgs`)有排除規則,所以授權頁的下拉**不會列出所有組織**:
  - 永遠排除自己。
  - 若當前是預設(admin)組織:排除已是其直接子組織者。
  - 若當前不是 admin 組織:排除 admin 組織本身、且只列「無父組織」或「唯一父組織就是 admin」的組織——即一個非 admin 組織不能同時被兩個非 admin 組織當子組織。

> 組織階層設計上是**單父樹**(root 為預設組織);上面的排除規則就是在維持這個單父不變式。多對多的 `AuthorizedOrganization` 只是它的 DB 表示法,見 [multi-tenant](../core/multi-tenant.md)。

---

## 代理登入（Proxy Login）

讓有 `Basic.PlatformUser.ProxyLogin` 權限的管理者「以某使用者的身分」登入,用來重現對方看到的畫面 / 權限。

**後端機制**:`GET /api/entity/proxyLogin/{使用者id}` →

1. 用目標使用者的組織 + 群組,建立(或重用)一個**影子 system 帳號**,帳號名格式 `Proxy(我的org|我的id:目標org|目標id)`、`IsSystemData = true`、顯示名 `{目標姓名}(Proxy:{我的姓名})`。
2. 把目標使用者的群組**整批複製**到影子帳號(先清空再 bulk insert)。
3. 以影子帳號發 `LoginResult` 回前端。交易隔離級別為 `Serializable`(避免並發建立重複影子帳號)。

> 代理登入是「目標**當下的快照**」,不是即時鏡射:群組、顯示名、Email、Phone 都在每次 proxyLogin 時依目標當下值重設。目標之後再被改,只在**下次** proxyLogin 反映,既有的代理 session 仍是舊值。

**前端接線**:

```typescript
// app 路由要自己掛上代理登入落地頁（library 的 createRoute 不含它）
{ path: 'proxy-login', component: ProxyLoginComponent }
```

- 使用者管理清單每列有一顆代理登入鈕(`permission.hasPermission('ProxyLogin')` 且非自己時才顯示),按下 `window.open('./proxy-login/?proxy={id}')`——**開新分頁**。
- 落地頁 `ProxyLoginComponent` 讀 `?proxy=` query 參數 → `userState.proxy(id)` → 打 proxyLogin API → 用 **`proxy-` 儲存前綴**存登入資料 → 導回首頁 reload。前綴隔離讓代理 session 與本尊 session 互不覆蓋。
- 代理中(`user.isProxyLogin`)平台會蓋一層遮罩標示、且**停用登出鈕**。
- ⚠️ **退出代理 = 關掉該分頁**:本尊 session 仍活在原本的分頁,代理 session 因為用 `window.open` 開在新分頁、且儲存前綴不同,關掉新分頁即結束代理,本尊不受影響。

---

## 啟用（前端）

`@hcs/basic` 同時提供「provider」與「路由」兩部分:

```typescript
// providers：選單項目 + i18n 索引 + 開發工具列
imports: [ HcsBasicProviderModule.forRoot() ]
```

`forRoot()` 提供:

- `MENU_ITEMS` → Basic 的選單(「使用者與群組」群組,下含組織 / 使用者 / 群組三項,各自掛對應的 `.View` 權限才顯示;見 [shell](../frontend/shell.md))。
- `I18N_INDEX` → `'basic'`(模組自帶的語系字串索引)。
- `HCS_TOOLBAR_COMPONENT` → 開發用工具列(`multi`)。

路由則用 `createRoute()`(或免覆寫的 `defaultRoute`)掛進 app 的路由表,落在 `basic/` 路徑下,含三組頁面的 `new` / `new/:copy`(複製)/ `:id`(唯讀檢視)/ `:id/edit` 路由,整段套 `RequireLoginGuard`。

頁面元件(selector):

| 頁面 | 清單 | 表單 |
|---|---|---|
| 組織 | `hcs-organization` | `hcs-organization-form` |
| 使用者 | `hcs-platform-user` | `hcs-platform-user-form` |
| 群組 | `hcs-platform-group` | `hcs-platform-group-form` |

子組織授權另有 `hcs-authorized-organization`。

---

## 管理頁的運作與權限關係

三個 entity 的管理頁各有一套**依當前登入者 / 目標身分 / 持有權限**收放功能的規則,有些(尤其群組)是核心運作方式、有些是防呆。⚠️ 共通前提:這些幾乎都是**前端管理頁的呈現與防呆**(藏按鈕、鎖欄位、前端跳過存檔),**不是後端硬性安全閘**——直接打 API 不一定都擋得住,真正的權限邊界仍是後端角色檢查。想改掉任何一條,就得整頁替換(見下面的 `createRoute` override),因為它們寫在內建頁元件裡。

### 使用者

- **改別人的所屬組織要 `Basic.PlatformUser.AllowChildOrgData`**:沒這權限表單看不到「組織」選擇器,新增的使用者一律落在當前組織;**編輯既有使用者時「組織」欄一律唯讀**——這頁不能把人搬到別的組織。
- **不能改自己的群組**:編輯自己時群組指派區塊整段隱藏、按存檔也直接略過——避免自我提權 / 降權把自己鎖在外面。要改自己的群組得由別的管理者操作。
- **不能刪 / 停用組織管理員**:清單對 `IsOrgAdmin` 的使用者隱藏刪除鈕、表單對 `IsOrgAdmin` 隱藏「狀態(啟用 / 停用)」欄。
- **不能代理登入自己**:清單代理鈕對自己那列隱藏(且需 `.ProxyLogin`,見上)。

### 群組（兩個畫面）

群組管理本質是「這個群組**能做什麼**(權限)」加「**誰**在這個群組(成員)」,平台把兩件事拆成兩個畫面:

- **編輯群組 + 指派權限**(`platform-group/:id/edit`):除了名稱 / 啟用,主體是一張**權限格**——列出所有功能(`PlatformFunction`)及其角色,每一格設成三態之一(拒絕 / 未設 / 允許),按「儲存權限」送出。**群組的權限就是在這裡指派的**(三態語意見 [permissions](../core/permissions.md))。`Services` 區塊只有持有 `Basic.PlatformGroup.Services` 才出現。
- **設定成員**(`platform-group/:id/setUsers`):此模式下群組本身的欄位(名稱 / 啟用 / 服務)全部鎖定,只管「哪些使用者屬於這個群組」;成員清單**排除自己**——不能在這裡把自己加進 / 移出群組。
- **組織欄同使用者**:要 `Basic.PlatformGroup.AllowChildOrgData` 才看得到組織選擇器,編輯既有群組時鎖定。

### 組織

- **`JoinSubOrg`(建立時自動成為我的子組織)只在新增時出現**:非 `Basic.Organization.Admin` 時此開關被鎖定為開(一定自動加),只有 `Admin` 能取消(後端對應的 `AutoJoinSubOrg` 副作用見上「子組織授權」)。
- **清單「子組織」鈕需 `Basic.Organization.Admin`**:點進子組織授權頁;「組織連結」鈕則對所有人開放。
- **子組織授權頁**:新增需 `Admin`,父組織固定為當前組織(鎖定),不提供複製 / 編輯。

---

## 客製：兩層替換機制

要改 Basic 的頁面有**兩個不同層級**的 opt-in,別搞混:

### 1. 換「整頁」——`createRoute(overrides)`

`createRoute` 收一個 `BasicComponentOverrides`,把任一路由頁換成你自己的元件(lazy `loadComponent`):

```typescript
HcsBasicProviderModule.createRoute({
  platformUserForm: () => import('./my-user-form.component').then(m => m.MyUserFormComponent),
  // organization / organizationForm / platformUser / platformGroup /
  // platformGroupForm / authorizedOrganization 同理
})
```

換掉的是**整個路由頁元件**——你接管整頁的 layout 與行為。

### 2. 換「頁內片段」——`HCS_PLATFORM_*_COMPONENT` token

每個內建頁把它的**搜尋列 / 清單 / 表單**三塊做成可注入的內嵌元件,各有一個替換 token(共 9 個):

| entity | 搜尋列 | 清單 | 表單 |
|---|---|---|---|
| 使用者 | `HCS_PLATFORM_USER_SEARCH_COMPONENT` | `HCS_PLATFORM_USER_LIST_COMPONENT` | `HCS_PLATFORM_USER_FORM_COMPONENT` |
| 組織 | `HCS_PLATFORM_ORG_SEARCH_COMPONENT` | `HCS_PLATFORM_ORG_LIST_COMPONENT` | `HCS_PLATFORM_ORG_FORM_COMPONENT` |
| 群組 | `HCS_PLATFORM_GROUP_SEARCH_COMPONENT` | `HCS_PLATFORM_GROUP_LIST_COMPONENT` | `HCS_PLATFORM_GROUP_FORM_COMPONENT` |

```typescript
{ provide: HCS_PLATFORM_USER_SEARCH_COMPONENT, useValue: MyUserSearchBarComponent }
```

保留內建頁的外殼(工具列、分頁、權限判斷),只替換其中一塊——**想微調搜尋條件或某欄位呈現**時用這層,比整頁替換省力。

> 兩層的取捨:整頁行為要大改 → `createRoute` override;只想換搜尋 / 清單 / 表單其中一塊 → token。token 不是 `multi`,provide 一個就是換掉那一塊。

---

## Gotchas

- **群組讀取對所有登入者開放**:`PlatformGroup` 的 Get / Query 掛 `Everyone`,`Basic.PlatformGroup.View` 只擋管理頁入口,不擋資料讀取。
- **修改使用者時密碼留空 = 不改**:空白不會清掉密碼;GET 也永遠拿不到密碼欄(被清成 `null`)。
- **代理登入要 app 自己掛落地路由**:`createRoute` 不含 `proxy-login`,consumer 得自行 `{ path: 'proxy-login', component: ProxyLoginComponent }`,否則清單的代理鈕開出來是 404。
- **退出代理靠關分頁**:代理開在新分頁、用 `proxy-` 儲存前綴隔離,代理中登出鈕被停用;關掉分頁即回到本尊。
- **跨子組織存取要 `.AllowChildOrgData`**:沒這權限只看得到 / 改得了自己組織的人與群組;往下管子組織需另授此權限。特別地,「設群組成員(`updateGroupUsers`)」這支硬閘的是 `Basic.PlatformUser.AllowChildOrgData`——就算你有 `Basic.PlatformGroup.Modify`,目標群組屬子組織而你沒這個 user 端權限,仍會被擋。
- **兩層替換別混用同一塊**:同一頁同時用 `createRoute` override 換整頁、又 provide 該頁的片段 token,片段 token 不會作用在你自己的整頁元件上(它只被內建頁讀)。

---

## 關聯

- entity 模型 / CRUD 管線 / audit — [core/entity-api](../core/entity-api.md)
- 組織樹 / 多租戶讀寫不對稱 / `.AllowChildOrgData` — [core/multi-tenant](../core/multi-tenant.md)
- 權限樹 / 標準 CRUD 角色（`.View` / `.Modify`）— [core/permissions](../core/permissions.md)
- 登入 / Token / OrgKey — [core/login](../core/login.md)
- 清單頁 / 表單頁 / 選單插槽用法 — [frontend/list](../frontend/list.md)、[frontend/form](../frontend/form.md)、[frontend/shell](../frontend/shell.md)
