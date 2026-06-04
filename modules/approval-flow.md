# ApprovalFlow 模組（簽核流程）

讓任一業務 entity 掛上**多關卡簽核**:由誰審、依什麼條件往下一關、每關要做什麼副作用(寄信、改狀態),全部可在後台用**流程設計器**畫出來,不必改程式重新部署。

和其他模組最大的不同:**流程的「形狀」是資料,不是程式碼。** 階段(stage)、按鈕(action)、往下一關的條件與分流,都存在 DB 裡、由管理者在前端設計器即時編輯;程式碼這端只負責兩件事——把一個**流程代碼**綁到某個 entity 型別,並提供設計器會用到的**具名鉤子**(取值、取人、事件副作用)。

分三塊,可分開理解:

1. **流程引擎**(`Hcs.Extensions.ApprovalFlow`):簽核狀態機本體。`AddApprovalFlow()` 註冊服務,`ConfigApprovalFlow(...)` 把一個流程代碼綁到 entity 型別,並登記它可用的具名 provider 與事件處理器。
2. **設定模組**(`Hcs.PlatformModule.ApprovalFlow`):`AddApprovalFlowModule()`。對外開出 `FlowDefine` / `FlowStage` / `FlowAction` 的 CRUD API(設計器存取流程定義用)、以及執行期的「取得可用動作 / 送出動作」兩支端點。
3. **前端**(`@hcs/approval-flow` + `@hcs/core` 的簽核動作元件):一個**流程設計器**頁(管理者畫流程),加一個**簽核動作元件**(嵌進業務表單,給簽核者按審核 / 退回)。

---

## 啟用（後端）

三個動作:

```csharp
// ① 服務註冊(通常在 Startup,一次)
services.AddApprovalFlow();

// ② 掛設定模組(FlowDefine/Stage/Action 的 CRUD API + 動作端點 + 權限)
builder.AddApprovalFlowModule();

// ③ 把一個「流程代碼」綁到要簽核的 entity 型別
moduleBuilder.ConfigApprovalFlow(options =>
{
    options.AddFlowCode<Invoice>("Invoice", opt =>
    {
        // 設計器會用到的具名鉤子(見下節)
        opt.AddUserProvider("主管", x => x.Pipe((DbContext db, Invoice m) => GetManagerId(db, m)));
        opt.AddValueProvider("金額", x => x.Pipe((Invoice m) => m.Amount));
        opt.AddEventHandler("發送通知信", x => x.Pipe(ctx => { /* 寄信 */ }));
    });
});
```

- ① 註冊引擎服務(狀態機、條件 / 分流 / 事件三個 evaluator、流程上下文)。漏掉 → 動作端點取不到服務。
- ② 註冊流程定義的管理 API 與 `ApprovalFlow.FlowDefine.*` 權限,以及給前端打的 `approvalFlow.action`(GET 取可用動作、PUT 送出動作)。漏掉 → 設計器與簽核 UI 都沒有端點可打。
- ③ **每種要簽核的 entity 綁一個流程代碼**。代碼(如 `"Invoice"`)是這條流程在系統內的識別,前端設計器、選單、動作元件都用它對應。一個 entity 型別可綁多個代碼(不同簽核情境)。

> ③ 只「綁定 + 登記鉤子」,**不畫流程**。實際的關卡、按鈕、分流條件由管理者在前端設計器建立並存進 DB。程式碼這端登記的具名 provider / event handler,是設計器裡的條件式和分流規則「**按名字**」引用的對象。

---

## 流程的組成

一條流程由三層**定義**(管理者畫的)+ 三層**狀態**(執行期產生的)構成。

### 定義(存 DB,設計器編輯)

- **FlowDefine**——一條具體流程,掛在某流程代碼下,可啟用 / 停用(`IsEnabled`)。同一代碼可有多條定義。
- **FlowStage**(階段)——流程的關卡,型別分 **Start / Default / End**。每個階段帶:
  - **往下一關的分流規則**:某個動作被按了「達到幾票 / 幾%」就進到哪一關、那一關指派給誰。
  - **事件鉤子**:進入此關(inbound)、離開此關(outbound)、有人回應時(update)各自要跑哪些具名 event handler。
- **FlowAction**(動作)——關卡上的按鈕(如「核准」「退回」)。帶顯示顏色、**出現條件**(達成才顯示這顆按鈕)、**是否強制填意見**、啟用旗標、排序。

### 狀態(執行期產生,一筆業務資料一組)

- **FlowApprovalState**——某筆業務資料(`ObjectId`)在某條流程定義下的一次簽核歷程;跑到 End 關時 `IsEnded`。
- **FlowApprovalSequence**——歷程中的一個**關卡實例**(同一關可能因退件重跑,故用遞增 `Sequence` 編號);當關結束時 `IsEnded`。
- **FlowApprovalSequenceDetail**——關卡裡**指派給某人的一筆待辦**:被指派者、他回應了哪個動作、回應時間、意見。一個關卡可指派多人(會簽 / 或簽)。明細分兩種:分流時**預設指派**進來的(`IsPreset`,會簽票數的分母只算這些)、與**主動送單 / 非預期回應**補出來的——Start 關沒有預設指派,第一個送單者會自動補一筆非 preset 明細。

`ObjectId` 就是業務 entity 的主鍵(字串化)——簽核狀態靠它跟業務資料關聯,簽核 entity 本身不持有業務外鍵。

---

## 設定每個流程代碼（`FlowCodeOption`）

`AddFlowCode<TModel>(code, opt => ...)` 裡能登記四類鉤子,**全部以「名稱」被設計器引用**:

| 方法 | 作用 | 被誰引用 |
|---|---|---|
| `AddValueProvider(name, pipe)` | 從 model 算出一個值(字串 / 數字 / 布林) | 動作出現條件、分流條件的運算元 |
| `AddUserProvider(name, pipe)` | 算出一個 / 一組**使用者** id | 分流時「指派給誰」、條件比對 |
| `AddGroupProvider(name, pipe)` | 算出一個 / 一組**群組** id | 同上(群組展開成成員) |
| `AddEventHandler(eventCode, pipe)` | 一段**副作用**(寄信、回寫業務狀態…) | 階段的 inbound / outbound / update 事件 |

```csharp
options.AddFlowCode<Invoice>("Invoice", opt =>
{
    // 取值:設計器的條件式可寫「金額 >= 100000」
    opt.AddValueProvider("金額", x => x.Pipe((Invoice m) => m.Amount));

    // 取人:分流規則可把下一關指派給「主管」
    opt.AddUserProvider("主管", x => x.Pipe((DbContext db, Invoice m) => GetManagerId(db, m)));
    opt.AddUserProvider("承辦人", x => x.Pipe((DbContext db, Invoice m) => m.OwnerId));

    // 事件:離開某關時寄通知
    opt.AddEventHandler("發送通知信", x => x.Pipe(ctx => { /* 用 ctx 取 model / 觸發動作 / 觸發人 */ }));

    // 覆寫「怎麼依 objectId(字串化主鍵)撈出 model」(預設即用主鍵比對,示意)
    opt.SetGetDataPipe(x => x.Pipe((DbContext db, string id) => db.Set<Invoice>().FindAsync(long.Parse(id)).AsTask()));
});
```

`AddUserProvider` / `AddGroupProvider` 各有「回單一 id」與「回一串 id」兩種多載;value provider 依回傳型別自動歸類成 String / Number / Boolean(供條件式判斷運算)。

### 內建 provider(免登記就能在設計器選用)

每個流程代碼會自動帶上這些:

| 名稱 | 是 | 典型用途 |
|---|---|---|
| `ViewerSender` | 目前操作者 | 條件比對「是不是本人」 |
| `Assigned` | 目前關卡被指派的人 | 分流沿用同一批人 |
| `Applicant` | 送單者(第一關的建立者) | 把某關退回給申請人 |
| `LastStageUsers` | 上一關的簽核者 | 加簽 / 回上一關 |
| `CurrentFlowStateCount` / `TotalFlowStateCount` | **其他**進行中 / 全部的簽核歷程數(`Current` 不含目前這條,剛新開時為 0) | 條件控管併發單數 |

若 entity 實作 `IPlatformEntity`,另自動帶 `CreatedBy` / `LastUpdatedBy` 兩個 user provider。

---

## 狀態機怎麼跑

以「送出一個動作」為主軸:

1. 前端對某筆 `objectId` 取得**可用動作**:引擎列出該筆資料各條進行中歷程的當關動作,並依**出現條件**過濾;同時**無條件**附上該代碼下所有啟用定義的 **Start 關**動作(= 開一條新歷程)。注意是「並列」不是「替代」——就算已經在跑,Start 動作仍會列出,所以同一筆資料可再開一條新歷程(見下方「多條定義並存」)。
2. 使用者按一個動作(可帶意見)。引擎以 `objectId` 取**分散式鎖**,序列化同一筆的並發送出。
3. 記錄這名使用者的回應(動作 / 時間 / 意見)到他的待辦明細。首次送單會建立 state + Start 關的 sequence。
4. **判斷是否進下一關**:對被按的動作,檢查該關的分流規則——同一動作的回應「**達到設定的票數或百分比**」、且分流條件成立,才觸發。
5. 觸發時:當關標記結束,建立下一關的 sequence 並把**指派對象**(由分流規則的 Provider / User / Group 決定)寫成待辦明細;沿途跑對應的事件鉤子。下一關若是 End → 整個 state 結束。

> 票數 / 百分比是**會簽**的核心:一關指派 3 人、分流設「核准達 100%」→ 三人都按核准才前進;設「達 1 票」→ 任一人核准即前進(或簽)。

幾個非直覺點:

- **多條定義並存**:取可用動作時,引擎會把該代碼下**所有啟用**定義的 Start 動作都列出來——所以「送單」階段可能同時看得到多條流程的起點。
- **狀態碼防竄改 / 防過期**:每筆歷程的識別(state + 目前 sequence)被 AES 加密成一段 `StateCode` 發給前端,送出時解密並比對「現在的當關是否仍是當時那一關」——關卡已被別人推進就拒收這次送出。
- **重複回應擋掉**:同一人在同一關已回應過就不再受理。

---

## 改一條已上線的流程（版本管理）

流程定義不是「隨時可重寫」的——**一旦有業務資料跑過這條流程,它用到的部分就被鎖住,不能砍掉重來**。這是**後端硬性保護**(不是只在 UI 上 disable):

- **FlowDefine**:只要底下有任何一筆簽核歷程(`FlowApprovalState`),就**不能刪**。
- **FlowStage**:只要某關被實例化過(產生過 sequence),那一關**不能刪**。
- **FlowAction**:只要某動作被指派 / 回應過(產生過 sequence detail),那顆按鈕**不能刪**。

設計器編輯既有流程時,增 / 刪關卡與動作是直接對 `FlowStage` / `FlowAction` 各自的 CRUD 動刀——所以想**移除一個已被跑過的關卡 / 動作會在後端被擋下**(刪除參照保護)。還能做的有:改流程名稱、切換 `IsEnabled`、調整既有關卡 / 動作的欄位、加**全新**的關卡;被鎖死的只有「移除已被使用的關卡 / 動作 / 整條定義」。

因此要實質改寫一條**已經在跑**的流程,正確姿勢不是在原定義上硬改,而是**版本替換**:同代碼**新建一份定義**(預設 `IsEnabled=false` 草稿)、設計好、再**啟用**它。送單時引擎只列出**啟用中**定義的起始關,新單就走新版。同代碼可同時存在多條定義,但通常只保持**一條啟用**,以免送單者同時看到多個起點。

> [!WARNING]
> **有「在途單」(進行中歷程)時,不要改動、也不要直接停用那條流程定義。** 執行期狀態**直接參照活的定義**——sequence 指向現存的 `FlowStage` / `FlowAction`,分流與動作的 JSON 在每次取動作 / 送出時即時讀取,**沒有逐單快照**。後果:
> - **改既有定義**:對在途單**立即生效**——它們可能改走和當初不同的分流、看到不一樣的按鈕 / 強制意見規則;若刪掉一個它們還沒走到的關卡(尚未實例化、刪得掉),下一跳就會斷。
> - **停用定義**:取動作端點**只列啟用中定義的動作**,所以停用後該定義的在途單會**拿不到任何按鈕而卡住**(歷程不會被取消,但也走不下去)。
>
> 安全的改版時機是**該定義已無在途單**時。若舊單還在跑、又想讓新單改走新版:**新建並啟用新定義**,而舊定義**保持啟用、只把它的「起始動作」(Start 關的 action)停用**(`IsEnabled=false`)——定義仍啟用故在途單按鈕照常,但起始動作沒了,新單不會再從它起跑;等舊單全部結束,再停用整條舊定義。

> 前端框架預留了「複製既有定義另存新檔」的路由(`:code/new/:copy`:載入來源 → 清主鍵 → 當新建存),但目前流程清單頁**沒有放出複製按鈕**,所以實務上是手動新建一份再設計。

---

## 權限與可見範圍

這裡有**兩種完全不同**的存取控制,別混為一談:

### 設計流程 → 看權限

`ApprovalFlow.FlowDefine` 功能的四個權限(`Create` / `View` / `Modify` / `Delete`)gate 的是**流程設計器**——誰能新增 / 檢視 / 修改 / 刪除流程定義、階段、動作。這是給管理者 / 系統設定者的。

### 參與簽核 → 看「有沒有被指派」,不看權限

「取得可用動作」「送出動作」這兩支端點對**任何登入者**開放(`Everyone`)——但你**實際看得到、按得動**什麼,由**簽核狀態裡的指派**決定,不是靠權限角色。關鍵在保護落在**取動作(GET)**這一端,而非送出端:

- **取可用動作**時過濾:只回傳你**送過單的**、或**這條歷程裡曾有任一關指派給你**的、或當下有可按動作的歷程——這是可見性的主要關卡。(指派比對是對整條歷程的所有 sequence 做的,所以退件重跑後,前面某關的指派人仍看得到這條歷程。)每筆回傳會附一段該歷程當關的 **StateCode**(加密的歷程識別)。
- **送出動作**時驗的是:StateCode 解得開、動作屬當關且啟用、狀態未被別人推進、你**尚未回應**、且該動作的**出現條件當下仍成立**(送出時會重跑一次 `ConditionJson`——GET 到 PUT 之間 model 若被改到不再符合條件,送出也會被拒)。**注意:送出端並不再次檢查你是否為當關指派人**——保護是間接的:沒先從 GET 端拿到合法 StateCode 就送不出來,而 StateCode 無法偽造。

也就是:能不能簽某一關,實質取決於系統有沒有把這關的合法 StateCode 發給你(GET 端只發給送單者 / 被指派者)。`FlowDefine.View` 只讓人看得到流程怎麼設計,**不會**讓他能簽別人的關。StateCode 本身只含**歷程識別、不綁特定使用者**——它等同「這一關的入場券」,別把它當成使用者身分憑證而任其外流。

---

## 前端

### 流程設計器（`@hcs/approval-flow`）

```typescript
imports: [ HcsApprovalFlowProviderModule.forRoot() ]   // 提供 i18n、選單
```

用 `HcsApprovalFlowProviderModule.defaultRoute`(落在 `approval-flow/` 下)掛進 app。選單會**為每個流程代碼動態長出一項**(打 `approvalFlow` 端點取代碼清單)。

設計器路由(都在代碼 `:code` 之下):

- `approval-flow/:code`——該代碼下的流程定義清單
- `approval-flow/:code/new`、`.../new/:copy`——新建 / 複製一條流程
- `approval-flow/:code/:id`(唯讀檢視)、`.../:id/edit`(編輯)——含階段 / 動作 / 分流條件的視覺化編輯與流程圖

### 簽核動作元件（`@hcs/core`)

把簽核按鈕嵌進**業務 entity 的表單頁**:

```html
<hcs-approval-flow-action [code]="'Invoice'" [objectId]="invoiceId" (updated)="reload()">
</hcs-approval-flow-action>
```

- 依 `code` + `objectId` 載入目前可用動作,渲染成按鈕。
- 按下時:若動作要求填意見而沒填則擋下;否則跳確認框 → 送出 → 重載動作;預設連帶重整整張表單(`autoReloadForm`),`(updated)` 事件也會發出供外部接手。

> 簽核流程的客製化集中在**後端的 provider / event handler** 與**設計器畫的流程**;前端動作元件本身沒有額外的注入式擴充點,照 `[code]`/`[objectId]` 用即可。

---

## Gotchas

- **流程形狀是資料、鉤子才是程式**:改關卡 / 條件 / 分流不必動程式(設計器存 DB);但條件 / 分流 / 事件引用的**具名 provider 與 event handler 必須先在 `AddFlowCode` 登記**——設計器選單裡能選的名字,就是程式登記過的那些。少登記一個,設計器就沒得選。
- **三件套各自獨立**:`AddApprovalFlow()`(引擎服務) + `AddApprovalFlowModule()`(管理 API + 動作端點) + `ConfigApprovalFlow(...)`(綁代碼)。漏服務 → 端點報錯;漏模組 → 沒 API;漏綁定 → 那個 entity 沒有任何流程可選。
- **設計權限 ≠ 簽核權限**:`FlowDefine.*` 控的是「能不能設計流程」;能不能簽某一關只看「**有沒有被指派到這關**」。要讓某人能簽,得讓流程的分流規則把人指派給他(透過 user/group provider 或設計器指定),不是給他權限碼。
- **會簽靠票數 / 百分比**:一關多人指派時,前進與否由分流規則的「票數達標」或「百分比達標」決定;設成 100% = 全員會簽,設成達 1 票 = 任一人即可(或簽)。
- **同一代碼可有多條啟用定義**:送單階段會同時列出所有啟用定義的起點動作。若不希望使用者看到多個起點,停用其餘定義或收斂成一條。
- **狀態碼會過期(含重啟)**:前端拿到的動作若卡著沒送、期間別人已推進了關卡,送出會被拒(防止對舊狀態誤操作);此外 StateCode 的加密金鑰是**程序啟動時隨機生成**,服務一重啟(部署 / 重啟)既發的 StateCode 全部失效。兩種情況都是重載動作即可。
- **一支端點可一次查多個代碼**:取可用動作的端點接受逗號分隔的多個流程代碼(同一 `objectId`),一次撈跨代碼的動作;前端動作元件預設只帶單一 `code`,要跨代碼彙整需自行組裝。
- **指派為空會擋(整筆回滾)**:分流到一個非 End 的關卡卻算不出任何指派對象,引擎丟例外(訊息要求 contact IT admin)。由於整次送出包在交易內,**這次送出整筆 rollback**——連當關剛寫進去的回應也一併退掉,操作者的體感是「按了沒反應」。通常是 user/group provider 回空或設計器漏指派所致。
- **跑過單的流程定義鎖定**:有簽核歷程後,定義 / 已用關卡 / 已用動作都**刪不掉**(後端刪除參照保護);改一條已上線流程靠「新建一份 + 啟停切換」做版本替換,不是原地重寫。見〈改一條已上線的流程〉。

---

## 關聯

- 在 entity API 上掛動作 / 生命週期 hook(簽核動作端點建在自訂 flow API 上)— [core/entity-api](../core/entity-api.md)
- Pipe 機制(provider / event handler 都是 `Pipe(...)`,參數自動 DI)— [core/pipe](../core/pipe.md)
- 權限樹 / 功能碼(`ApprovalFlow.FlowDefine.*` 的來源)— [core/permissions](../core/permissions.md)
- 多租戶(分流取人會限定在流程定義所屬組織)— [core/multi-tenant](../core/multi-tenant.md)
- 簽核過程的副作用常搭配發訊 — 多通道發訊抽象(`Hcs.Extensions.Message`)
