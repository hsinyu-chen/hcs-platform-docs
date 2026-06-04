# 列表頁(BaseListComponent / hcs-data-grid)

> _待寫:一句話定位——列表頁包了資料抓取、分頁、查詢轉 OData、匯出、狀態持久化;使用端只宣告欄位與查詢條件。對應後端 `AddEntityApi` 的 Query。_

---

## BaseListComponent 本質

> _待寫:`extends BaseListComponent<T>` + `super(service, Entity)` 的最小骨架;`DataSourceFactory` 怎麼自動對接 OData 端點。_

## 建一個列表頁

> _待寫:逐步——建 component、provide `HCS_FUNCTION_NAME`、建查詢 form、template 用 `hcs-data-grid` + `*hcsDataGridColum`;接 route。_

## 欄位與查詢(查詢條件 → OData)

> _待寫:`*hcsDataGridColum` 宣告欄位、`hcs-*-input` 宣告查詢條件怎麼自動轉成 `$filter`;`$expand` 帶關聯欄位(連 entity-api 的 Get/Query Include 差異)。_

> _待寫:URL `?$field=值` 預填查詢條件、`?$$field=值` 預填並鎖死查詢欄位(查詢列無 state,故 `$` 一律生效)——與表單同一套機制,canonical 說明在 [form](form.md#url-帶入欄位值-預填--鎖定)。_

## 查詢指令客製(`hcsDataGridQuery`)

> _待寫:查詢控制元件群(同一批 `hcs-*-input`,逐項參考見 [controls](controls.md))掛 `[hcsDataGridQuery]` 怎麼轉 `$filter`。兩種模式:_
>
> - _**預設**:`field` + `operator`(`IFilterOperator`)+ 選用 `valueTransform`,空值自動略過,組 `ds.where(field, operator, value)`。_
> - _**客製**:`[hcsDataGridQuery]="自訂fn"` 傳 `DataGridQuery<T>` 函式 `(ds, next) => next(改過的 ds)`——自接多欄 OR / 範圍 / 運算式 filter 等非標準查詢。去敏 demo:一個輸入框同時比對兩個欄位。_
>
> _另:`hcs-sort-input` 排序、`hcsDataGridColum` 欄位宣告與查詢指令的分工。_

## 分頁 / 排序 / 狀態持久化

> _待寫:`HCS_ENABLE_STATE`(false→臨時儲存,排序/分頁不持久)、`HCS_USER_STATE_SERVICE`(自訂儲存後端);連前端絕不直接碰 storage 的規矩。_

## 匯出

> _待寫:`HCS_EXPORT_LIMIT`(無 fallback,必給)、`HCS_ENABLE_EXPORT`(undefined→true,匯出+選列總開關)。_

## 列印

> _待寫:`HCS_PRINT_SERVICE` 替換列印行為。_

## 攔截與覆寫點(opt-in)

### 刪除前攔截:`HCS_LIST_SERVICE`

> _待寫:`IListService` 介面、平台跑迴圈、回 `false` 中止刪除;去敏 demo(實作 + provider multi)。_

## 替換整個列表頁(Component Override)

> _待寫:`createRoute({ xxxList: () => import(...) })` 去敏 demo。_

## Gotchas

> _待寫:`HCS_EXPORT_LIMIT` 無預設漏給的後果;`HCS_ENABLE_STATE` 關掉時的行為;Query 不自動 Include 建立者(要 `$expand`)。_

## 關聯

> _待寫:連 frontend.md、form.md、core/entity-api.md(Query 端點)、core/data-pipes.md。_
