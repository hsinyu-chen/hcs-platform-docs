# 列表頁(BaseListComponent / hcs-data-grid)

一個列表頁 = `extends BaseListComponent<T>` + 一個查詢 reactive form + 一個 `<hcs-data-grid>`。基底與 grid 把資料抓取、分頁、查詢條件轉 OData、排序、匯出、狀態記憶全包了;你只宣告**要顯示哪些欄、可以用哪些條件查**。對應後端 [entity-api](../core/entity-api.md) 的 Query 端點。

---

## BaseListComponent 本質

```typescript
@Component({
  selector: 'invoice-list',
  templateUrl: './invoice-list.component.html',
  providers: [BaseComponentService, { provide: HCS_FUNCTION_NAME, useValue: 'Sales.Invoice' }, Permission],
})
export class InvoiceListComponent extends BaseListComponent<Invoice> {
  filterForm = new FormGroup({
    No: new FormControl(),
    Status: new FormControl(),
  });

  constructor(service: BaseComponentService) {
    super(service, Invoice);   // ← 傳 entity 型別,基底自動建 datasource
  }
}
```

基底準備好:

- **`data`**:`OdataDataSource<T>`,自動對接後端 `/api/odata/<T>`(見 [frontend](../frontend.md) 慣例 7)。
- **`listDataSource`**:投影用 `OdataDataSource<TView>`;`super(service, T, TView)` 給第三個型別才有差,否則同 `data`。
- **`grid`** / **`listPage`**:`@ViewChild` 抓到模板的 `<hcs-data-grid>` / `<hcs-list-page>`。
- **`uiOptions`**:平台 UI 偏好(如操作欄靠左/右)。

你只負責 `filterForm`(查詢條件的 reactive form)與模板。component-level **一定要 provide `BaseComponentService`**(基底依賴、非 root,漏了 `NullInjectorError`)。

---

## 建一個列表頁

1. **component**:`extends BaseListComponent<T>`、`super(service, T)`、provide `BaseComponentService` + `HCS_FUNCTION_NAME`、宣告 `filterForm`。
2. **template**:`<hcs-list-page>` 包 `<hcs-data-grid [data]="data">`;查詢輸入放 `grid-head` slot、欄位用 `*hcsDataGridColum`、`<hcs-pager grid-foot>`。

```html
<hcs-list-page>
  <hcs-data-grid [data]="data" #grid [autoLoad]="true">
    <!-- 查詢列:grid-head slot + filterForm -->
    <ng-container grid-head [formGroup]="filterForm">
      <hcs-textbox-input formControlName="No" hcsDataGridQuery operator="contains"><label translate>models.Sales.Invoice.No</label></hcs-textbox-input>
      <hcs-select-input formControlName="Status" hcsDataGridQuery operator="=">
        <label translate>models.Sales.Invoice.Status</label>
        <hcs-option><span>{{ 'common.all' | translate }}</span></hcs-option>
        <hcs-option *ngFor="let o of (StatusEnum | enumOptions)" [value]="o.value">{{ o.text }}</hcs-option>
      </hcs-select-input>
      <div class="buttons">
        <hcs-default-button-search-bar [grid]="grid" [filterForm]="filterForm"></hcs-default-button-search-bar>
      </div>
    </ng-container>

    <!-- 欄位;name 是欄頭 i18n key,過 |translate(不硬寫字串) -->
    <ng-container *hcsDataGridColum="let value;let entity=entity;field:'No';export:true,name:'models.Sales.Invoice.No'|translate">
      <a [routerLink]="[entity.Id]" queryParamsHandling="merge">{{ value }}</a>
    </ng-container>
    <ng-container *hcsDataGridColum="let value;field:'Amount';align:'right';export:true,name:'models.Sales.Invoice.Amount'|translate">{{ value }}</ng-container>
    <!-- 行操作(編輯 / 刪除 / 複製) -->
    <ng-container *hcsDataGridColum="let value;width:125;field:'Id';name:'';sortable:false">
      <hcs-default-button-list [data]="data" [key]="value"></hcs-default-button-list>
    </ng-container>

    <hcs-pager grid-foot></hcs-pager>
  </hcs-data-grid>
</hcs-list-page>
```

- **`hcs-default-button-search-bar`**:查詢 / 新增 / 匯出按鈕列(放 `grid-head`,吃 `[grid]` + `[filterForm]`)。
- **`hcs-default-button-list`**:每列的編輯 / 刪除 / 複製按鈕(`showEdit` / `showDelete` / `showCopy` 個別開關,吃 `[data]` + `[key]`)。

常用的 `<hcs-data-grid>` `@Input`:`autoLoad`(進頁即查,預設 `false`)、`defaultSortState`(預設排序)、`sortable`、`forceMode`(`'table'`/`'page'`)、`enableRowSelect`(布林或 `(entity) => boolean` 逐列控制可否勾選)、`statePrifix`。

---

## 欄位與查詢(查詢條件 → OData)

**欄位**用 `*hcsDataGridColum` 結構指令的微語法宣告;`let value` 是該欄值、`let entity=entity` 是整列:

| 參數 | 作用 |
|---|---|
| `field:'No'` | 對應 entity 欄位;**支援點路徑** `'Customer.Name'`(自動逐層取值) |
| `name:'models.Sales.Invoice.No'` | 欄頭 i18n key(接 `\| translate`;不硬寫字串) |
| `width` / `align` / `phoneAlign` | 寬度 / 桌面對齊 / 手機對齊 |
| `orderby:'X'` | 排序時改用別的欄位(預設用 `field`) |
| `sortable:false` | 關閉該欄排序 |
| `visible:expr` | **程式層**顯示;`false` 永不出現(連欄位設定都不列) |
| `userVisibleDefault:false` | 使用者欄位設定裡的**預設**顯示(使用者可在設定對話框再開關) |
| `cellClass` / `headerClass` | 儲存格 / 表頭 CSS class |
| `cellStyle` | 物件或 `(entity) => 物件`,逐儲存格樣式 |
| `export:true`(或 `'欄名'`) | 納入匯出(字串=匯出時改用別的欄) |

> `*hcsDataGridColum` 的 cell template context 還帶 `entity`(整列)、`array`(整個 data source)、`index`、`summary`/`summaryIndex`(摘要列旗標),可 `let entity=entity; let summary=summary` 取用。

**查詢條件**:把 `hcs-*-input` 放進 `grid-head` 的 `[formGroup]="filterForm"`,各掛 `hcsDataGridQuery` 自動轉 `$filter`(見[下節](#查詢指令客製hcsdatagridquery))。輸入元件逐項見 [controls](controls.md)。

要帶關聯欄位用 OData `$expand`(後端 `AllowExpand` 白名單內),與 entity-api 的 Get 自動 Include / Query 不自動 的差異見 [entity-api](../core/entity-api.md)。

---

## URL 帶入欄位值:`$` 預填 / `$$` 鎖定

URL `?$field=值` 預填查詢條件、`?$$field=值` 預填並鎖死查詢欄位——與表單同一套機制,**查詢列無 state 概念,故 `$` 一律生效**。canonical 說明在 [form](form.md#url-帶入欄位值-預填--鎖定)。

---

## 查詢指令客製(`hcsDataGridQuery`)

查詢控制元件(`hcs-textbox-input` / `hcs-select-input` / `hcs-date-input` …)掛 `[hcsDataGridQuery]` 轉 `$filter`,兩種模式:

- **預設**:給 `field`(省略則用 control name)+ `operator`(`contains` / `=` / `in` …)+ 選用 `valueTransform`;空值自動略過,組成 `ds.where(field, operator, value)`。

  ```html
  <hcs-textbox-input formControlName="No" hcsDataGridQuery operator="contains"><label translate>models.Sales.Invoice.No</label></hcs-textbox-input>
  ```

- **客製**:`[hcsDataGridQuery]="自訂fn"` 傳 `DataGridQuery<T>` 函式 `(ds, next) => next(改過的 ds)`,自接多欄 OR / 範圍 / 運算式 filter 等非標準查詢。

  ```html
  <!-- 複合語意 label(名稱或代碼)無單一欄位對應,用 components.{feature}.{item} 類別 key -->
  <hcs-textbox-input [hcsDataGridQuery]="searchNameOrCode"><label translate>components.invoiceSearch.nameOrCode</label></hcs-textbox-input>
  ```
  ```typescript
  searchNameOrCode = (ds, next) => {
    const v = this.filterForm.value.keyword;
    next(v ? ds.where('Name', 'contains', v).or('Code', 'contains', v) : ds);
  };
  ```

排序欄位用 `hcs-sort-input`;`*hcsDataGridColum` 負責欄位顯示,`hcsDataGridQuery` 負責查詢,兩者分工。

## 查詢管線:排序 / 分頁 / filter 都是 plugin(`addQuery`)

grid 的查詢不是寫死的,而是一條**可插拔的管線**。任何元件 / 指令都能 `grid.addQuery(q)` 註冊一個查詢貢獻者(`IDataGridQuery = { query: (ds, next) => void }`);載入時 grid 把所有貢獻者串成一條 middleware chain(`buildQuery`),依序轉換 datasource、最後才送出 OData。

**平台內建的東西其實都是這條管線上的 plugin**,地位完全相同:

| plugin | 貢獻 |
|---|---|
| 排序(grid 自帶) | `orderby` |
| **`hcs-pager`** | `skip` / `take`(`grid.addQuery(this)`) |
| 每個 `[hcsDataGridQuery]` | `where`(`$filter`) |

所以 **pager 不是內建特例,它只是一個註冊了 `skip`/`take` 的查詢 plugin**——你也能用同一招注入自訂查詢:

```typescript
this.grid.addQuery({ query: (ds, next) => next(ds.where('OrgId', '=', myOrgId)) });
```

> 這也解釋了為什麼[客製 `hcsDataGridQuery`](#查詢指令客製hcsdatagridquery)忘了呼叫 `next` 會讓整個 grid 卡住——一個 plugin 不呼叫 `next`,整條 chain 就斷在那裡、查詢永遠送不出去。

---

## 列選取(checkbox)

只要 entity 有主鍵,grid 左側預設就有一欄勾選框、表頭有「全選本頁」:

- **`enableRowSelect`**(**預設 `true`**):要關掉設 `[enableRowSelect]="false"`(或 `HCS_ENABLE_EXPORT=false` 連帶關)。也可給 `(entity) => boolean` **逐列**決定哪些列可勾(回 `false` 的列勾選框 disabled)。
- 點整列也會切換勾選;選取結果餵給[匯出](#匯出)(有勾→只匯出勾選的)。
- 事件:`(rowClick)`、`(rowSelectionChanged)`(單列)、`(selectionCleared)`。

```html
<hcs-data-grid [data]="data" [enableRowSelect]="canSelect">…</hcs-data-grid>
```
```typescript
canSelect = (row: Invoice) => row.Status !== 'Closed';   // 已結案的不可選
```

> `HCS_ENABLE_EXPORT=false` 會連帶把 `enableRowSelect` 關掉(選取本來就是為匯出服務,見 [匯出](#匯出))。

## 摘要列(summary)

`[summaryData]` 給一個物件陣列,會用**同一批 `*hcsDataGridColum` 欄位模板**渲染成釘在底部的摘要列(放合計 / 小計)。欄位模板的 context 帶 `summary` 旗標,可用 `let summary=summary` 在摘要列改顯示:

```html
<hcs-data-grid [data]="data" [summaryData]="[{ Amount: totalAmount }]">
  <ng-container *hcsDataGridColum="let value;let summary=summary;field:'Amount';align:'right',name:'models.Sales.Invoice.Amount'|translate">
    <strong *ngIf="summary">合計:{{ value }}</strong>
    <ng-container *ngIf="!summary">{{ value }}</ng-container>
  </ng-container>
</hcs-data-grid>
```

> 摘要物件用的是同一組欄位 `field`,所以欄位名要對得上(或在模板裡用 `summary` 分流)。

---

## 欄位設定:顯示 / 隱藏 / 凍結

grid 表頭的設定鈕開一個對話框,讓**使用者自己**逐欄切換「顯示」與「凍結」;設定存在 `grid-column-parameter:<路由>:<statePrifix>`(`UserStateStorage`,跟帳號走),回訪保留。

- **顯示**:`userVisibleDefault`(欄位的預設顯示)是使用者可改的;`visible`(程式層)若為 `false` 則該欄**永不出現、也不列進設定對話框**——兩者不同層。
- **凍結(freeze)**:把欄位釘在左側(sticky)。**只有第一欄、或設了固定寬度(`px`/`em`/`rem`/`%`/`vw`/`vh`)的欄可凍結**(`canFreeze`);沒固定寬度的欄不給凍。凍結欄的左偏移會自動把選取 checkbox 欄(40px)算進去。

## 手機 / 桌面雙模式

grid 會依裝置自動切版面:桌面用**表格**、手機用**卡片**(`usePhoneTable` 預設 `true`);使用者可自行切換,偏好存 `hcs-grid-phone-table:<路由>:<statePrifix>`。

- **`forceMode`**(`'table'` / `'page'`):強制鎖定某一種版面、不給切換。
- **`disableHeadToggle`**:關掉表格/卡片切換鈕。
- **`phoneAlign`**(欄位參數):卡片版面的對齊,獨立於桌面 `align`。
- **`miniHead`**:精簡表頭。

> 逐列外觀用 `[getRowStyle]="(entity, i) => …"` / `[getRowClasses]="(entity, i, phoneMode, summary) => …"` 回傳 Angular style / class 物件(回 `undefined` 表示不變);兩種版面都吃得到。

---

## 分頁 / 排序 / 狀態持久化

查詢條件、排序、捲動位置、分頁頁碼**會被記住、回訪自動還原**——由 `PageStatusHolder` 存(查詢表單走 `bindFormGroup`、grid 狀態走 `getAccessor`),底層是 `page-state-container`(sessionStorage)/ `site-state-container`(localStorage),**不是裸 storage**(前端不直接碰 `localStorage`/`sessionStorage` 的約定見 [frontend](../frontend.md))。

- **`HCS_ENABLE_STATE`**(boolean):`false` → 排序 / 分頁改用臨時值、**不持久**(離開就忘)。
- **`hcs-pager`**(放 `grid-foot`):顯示總筆數 / 目前範圍、提供每頁筆數選單。`pageSize`(預設 `20`)、`pageSizeOptions`(`[20,50,100,200]`)。注意兩者生命週期不同:**pageSize 記在 `UserStateStorage`(跟使用者帳號走)、page 頁碼走 `PageStatusHolder` 的 sessionStorage(關分頁就忘)**。換查詢(`grid.reset()`)會回到第 1 頁。

> ⚠️ **state 用「路由」當 key。** `hcs-data-grid` 的 `@Input statePrifix`(預設 `'0'`,原碼拼字 Prifix)**只切開這四項**:排序、捲動位置、欄位設定、手機/桌面切換——同頁多個 grid 給不同 `statePrifix` 能把這些區隔開。**但查詢條件(filter form)與分頁頁碼沒有 prefix 機制**(key 分別寫死成 `'form'` / `'pager-page'`、只認路由),所以**同一路由放兩組獨立查詢的列表,查詢與頁碼一定互蓋,`statePrifix` 救不了**——真要避免就別在同路由擺兩個獨立查詢的列表。網址帶 `?clear=1`(或選單導航)會清掉該頁記住的狀態。

---

## 匯出

`hcs-default-button-search-bar` 的匯出鈕把目前查詢結果輸出(後端帶 `export=true`)。

- **`HCS_EXPORT_LIMIT`**(number,**無 fallback、必須 provide**):匯出筆數上限;超過 → 跳提示不匯出。
- **`HCS_ENABLE_EXPORT`**(boolean,`@Optional`,**undefined → `true`**):匯出總開關;設 `false` 時連列選取(checkbox)也一併關閉。
- **匯出哪些欄**:只有 `*hcsDataGridColum` 標了 `export:true`(或 `export:'別的欄'`)的欄會輸出;沒有任何欄標 export → 不顯示匯出鈕。
- **匯出範圍**:有勾選列 → 只匯出勾選的;沒勾 → 匯出符合查詢的全部(再受 `HCS_EXPORT_LIMIT` 約束)。

---

## 列印

`HCS_PRINT_SERVICE`(`{ name, serviceType }[]`,multi)註冊一或多個列印實作(`IPrint`),由 `PrinterService` 統籌;消費端 provide 自己的列印 service 即可掛上。

---

## 刪除前攔截:`HCS_LIST_SERVICE`

`hcs-default-button-list` 的刪除鈕在**確認對話框之後、真正 `delete` 之前**跑所有 `HCS_LIST_SERVICE`(multi):

```typescript
export interface IListService<TKey, TModel> {
  // 回 false 中止刪除;全部回 true 才真的刪
  runAsync(id: TKey, action: 'delete', ds: IHttpDataSource<TModel>): Promise<boolean>;
}
```

```typescript
{ provide: HCS_LIST_SERVICE, useClass: ConfirmTwiceService, multi: true }
```

任一 service 回 `false` → 不刪(例如需要二次驗證 / 額外確認)。

---

## 替換整個列表頁(Component Override)

要整頁換掉預設列表時用模組的 component override——`createRoute({ ... })` 傳 lazy-loaded 元件,路由 / 權限 / i18n 都保留(見 [frontend](../frontend.md) 慣例 3)。

```typescript
HcsBasicProviderModule.createRoute({
  platformUser: () => import('./custom-user-list.component').then(x => x.CustomUserListComponent),
})
```

---

## Gotchas

- **`BaseComponentService` 必 provide**:同 form,基底非 root,漏了 `NullInjectorError`。
- **`statePrifix` 只切開部分狀態**:排序 / 捲動 / 欄位設定 / 手機切換可用不同 `statePrifix` 區隔;**查詢條件與分頁頁碼只認路由、無 prefix**,同路由兩個獨立列表這兩項一定互蓋,`statePrifix` 救不了。
- **pageSize 與手機/桌面偏好共用同一個 storage key**(`hcs-grid-phone-table:<路由>:<prefix>`)——兩者互相覆寫(存了 pageSize 後手機偏好讀不回布林、切過手機模式後 pageSize 跑回預設)。已知 code 雷,目前無 work around。
- **`HCS_EXPORT_LIMIT` 沒 fallback**:沒 provide 會在匯出時注入失敗;一定要在 host 設一個值。
- **關掉匯出連帶關掉選取**:`HCS_ENABLE_EXPORT=false` 時列選取 checkbox 也消失(選取本來就是為了挑要匯出的列)。
- **沒標 `export` 的欄不會匯出**:欄位要進匯出檔得在 `*hcsDataGridColum` 加 `export:true`。
- **刪除攔截在確認框之後**:`HCS_LIST_SERVICE` 跑在使用者按下「確認刪除」之後;它回 `false` 才是最終否決。
- **客製 `hcsDataGridQuery` 一定要呼叫第二個參數**:`DataGridQuery<T>` 簽名是 `(ds, next) => void`,`next` 是 callback——**忘了呼叫 `next(ds)`,這條 grid 就永遠不發查詢**(整個列表卡住不載入)。

---

## 關聯

- 查詢 / 表單共用的輸入元件 — [controls](controls.md)
- 表單頁、`$`/`$$` 帶入的 canonical 說明、對話框內 grid 的 `statePrifix` 雷 — [form](form.md)
- 後端 Query 端點 / OData `$filter`/`$expand` / `AllowExpand` — [core/entity-api](../core/entity-api.md)
- 自動過 `OrgId` 多租戶過濾 — [core/data-pipes](../core/data-pipes.md)
- 前端不直接碰 storage 的約定 — [frontend](../frontend.md)
