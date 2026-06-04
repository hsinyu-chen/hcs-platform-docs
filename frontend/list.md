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
      <hcs-textbox-input formControlName="No" hcsDataGridQuery operator="contains"><label>單號</label></hcs-textbox-input>
      <hcs-select-input formControlName="Status" hcsDataGridQuery operator="=">
        <label>狀態</label>
        <hcs-option><span>全部</span></hcs-option>
        <hcs-option *ngFor="let o of (StatusEnum | enumOptions)" [value]="o.value">{{ o.text }}</hcs-option>
      </hcs-select-input>
      <div class="buttons">
        <hcs-default-button-search-bar [grid]="grid" [filterForm]="filterForm"></hcs-default-button-search-bar>
      </div>
    </ng-container>

    <!-- 欄位 -->
    <ng-container *hcsDataGridColum="let value;let entity=entity;field:'No';export:true,name:'單號'">
      <a [routerLink]="[entity.Id]" queryParamsHandling="merge">{{ value }}</a>
    </ng-container>
    <ng-container *hcsDataGridColum="let value;field:'Amount';align:'right';export:true,name:'金額'">{{ value }}</ng-container>
    <!-- 行操作(檢視/編輯/刪除/複製) -->
    <ng-container *hcsDataGridColum="let value;width:125;field:'Id';name:'';sortable:false">
      <hcs-default-button-list [data]="data" [key]="value"></hcs-default-button-list>
    </ng-container>

    <hcs-pager grid-foot></hcs-pager>
  </hcs-data-grid>
</hcs-list-page>
```

- **`hcs-default-button-search-bar`**:查詢 / 新增 / 匯出按鈕列(放 `grid-head`,吃 `[grid]` + `[filterForm]`)。
- **`hcs-default-button-list`**:每列的檢視 / 編輯 / 刪除 / 複製按鈕(吃 `[data]` + `[key]`)。

---

## 欄位與查詢(查詢條件 → OData)

**欄位**用 `*hcsDataGridColum` 結構指令的微語法宣告;`let value` 是該欄值、`let entity=entity` 是整列:

| 參數 | 作用 |
|---|---|
| `field:'No'` | 對應 entity 欄位 |
| `name:'單號'` | 欄頭文字(可接 `\| translate`) |
| `width` / `align` / `phoneAlign` | 寬度 / 對齊 |
| `sortable:false` | 關閉該欄排序 |
| `visible:expr` | 條件顯示 |
| `export:true`(或 `'欄名'`) | 納入匯出(字串=匯出時改用別的欄) |

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
  <hcs-textbox-input formControlName="No" hcsDataGridQuery operator="contains"><label>單號</label></hcs-textbox-input>
  ```

- **客製**:`[hcsDataGridQuery]="自訂fn"` 傳 `DataGridQuery<T>` 函式 `(ds, next) => next(改過的 ds)`,自接多欄 OR / 範圍 / 運算式 filter 等非標準查詢。

  ```html
  <hcs-textbox-input [hcsDataGridQuery]="searchNameOrCode"><label>名稱或代碼</label></hcs-textbox-input>
  ```
  ```typescript
  searchNameOrCode = (ds, next) => {
    const v = this.filterForm.value.keyword;
    next(v ? ds.where('Name', 'contains', v).or('Code', 'contains', v) : ds);
  };
  ```

排序欄位用 `hcs-sort-input`;`*hcsDataGridColum` 負責欄位顯示,`hcsDataGridQuery` 負責查詢,兩者分工。

---

## 分頁 / 排序 / 狀態持久化

查詢條件、排序、捲動位置、分頁頁碼**會被記住、回訪自動還原**——由 `PageStatusHolder` 存(查詢表單走 `bindFormGroup`、grid 狀態走 `getAccessor`),底層是 `page-state-container`(sessionStorage)/ `site-state-container`(localStorage),**不是裸 storage**(前端不直接碰 `localStorage`/`sessionStorage` 的約定見 [frontend](../frontend.md))。

- **`HCS_ENABLE_STATE`**(boolean):`false` → 排序 / 分頁改用臨時值、**不持久**(離開就忘)。
- **`hcs-pager`**:`pageSize`(預設 `20`)、`pageSizeOptions`(`[20,50,100,200]`);pageSize 記在 `UserStateStorage`。
- **`HCS_USER_STATE_SERVICE`**(`IUserStateService`):換掉狀態儲存後端(自訂存哪)。

> ⚠️ **state 用「路由」當 key**,所以**每個路由只存得下一組**。`hcs-data-grid` 的 `@Input statePrifix`(預設 `'0'`,原碼拼字 Prifix)是區隔手段——**同頁多個 grid、或對話框裡的 grid 疊在 host 路由上時,務必各給不同 `statePrifix`**,否則狀態互相覆蓋(對話框情境見 [form](form.md#對話框模式同一個表單元件也能當-dialog-開))。網址帶 `?clear=1`(或選單導航)會清掉該頁記住的狀態。

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
  platformUserList: () => import('./custom-user-list.component').then(x => x.CustomUserListComponent),
})
```

---

## Gotchas

- **`BaseComponentService` 必 provide**:同 form,基底非 root,漏了 `NullInjectorError`。
- **每路由只記一組狀態**:`statePrifix` 預設 `'0'`;同頁兩個 grid 或對話框內 grid 不設不同 prefix 會互相蓋掉查詢 / 排序 / 分頁記憶。
- **`HCS_EXPORT_LIMIT` 沒 fallback**:沒 provide 會在匯出時注入失敗;一定要在 host 設一個值。
- **關掉匯出連帶關掉選取**:`HCS_ENABLE_EXPORT=false` 時列選取 checkbox 也消失(選取本來就是為了挑要匯出的列)。
- **沒標 `export` 的欄不會匯出**:欄位要進匯出檔得在 `*hcsDataGridColum` 加 `export:true`。
- **刪除攔截在確認框之後**:`HCS_LIST_SERVICE` 跑在使用者按下「確認刪除」之後;它回 `false` 才是最終否決。

---

## 關聯

- 查詢 / 表單共用的輸入元件 — [controls](controls.md)
- 表單頁、`$`/`$$` 帶入的 canonical 說明、對話框內 grid 的 `statePrifix` 雷 — [form](form.md)
- 後端 Query 端點 / OData `$filter`/`$expand` / `AllowExpand` — [core/entity-api](../core/entity-api.md)
- 自動過 `OrgId` 多租戶過濾 — [core/data-pipes](../core/data-pipes.md)
- 前端不直接碰 storage 的約定 — [frontend](../frontend.md)
