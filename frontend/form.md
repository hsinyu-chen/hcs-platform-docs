# 表單頁(BaseFormComponent)

> _待寫:一句話定位——表單頁是什麼、基底替你包了哪些樣板(資料載入、存檔、驗證錯誤回填),使用端只寫 reactive form 與欄位。對應後端 `AddEntityApi` 的 Post/Put。_

---

## BaseFormComponent 本質

> _待寫:`extends BaseFormComponent<T>` + `super(service, Entity)` 的最小骨架;它包了什麼(載入既有資料、送出、`hcs-error-summary` 接 400)、留什麼給你寫(reactive form + validators)。_

## 建一個表單頁

> _待寫:逐步——建 component、provide `HCS_FUNCTION_NAME`、建編輯 form、template 用 `hcs-form-page` 系列;接 route。_

## hcs-form-page 與版面

> _待寫:表單容器元件、區塊(expansion panel)、欄位元件群、與 Material 的關係。基本輸入欄位(textbox/select/date…)用 `hcs-*-input`,**逐項參考見 [controls](controls.md)**;這裡只示範 2~3 個常用的怎麼綁進表單。_

## 進階欄位元件(參照 / 子表 / 檔案 / RichText)

> _待寫:比簡單輸入重、幾乎只在表單用的欄位元件,各一節給去敏 demo:_
>
> - _**`hcs-reference-input`**:參照選擇(挑另一個 entity 當外鍵),牽 `reference-dialog` 彈窗挑選 + 自動 `$expand`。_
> - _**`child-panel`**:主檔-明細子表(一對多在同一個表單編輯)。_
> - _**`hcs-file-input` / file**:檔案上傳欄位(連 [core/file-upload](../core/file-upload.md))。_
> - _**`hcs-ckeditor`**:RichText(WYSIWYG)欄位(設定見下方 `CKEDITOR_CONFIG`)。_

## URL 帶入欄位值:`$` 預填 / `$$` 鎖定

> _待寫:導頁時用 query string 把欄位帶進來——`?$field=值` 只在**新增**表單預填(可改的預設值)、`?$$field=值` 預填**並鎖死**(`disable()`,任何狀態都鎖);兩者依 entity 屬性型別自動 Boolean/Number 轉型。對話框表單則改用 `dialogData.fillProperty` 程式帶入,同一套規則。這就是 [multi-tenant](../core/multi-tenant.md)「UI 把 `OrgId` 拉出來」的前端機制(`?$$OrgId=5` 帶固定組織脈絡並鎖)。去敏 demo:從列表「新增」帶父層 id。機制同樣用在列表查詢列(見 [list](list.md)),但那邊無 state。_

## 驗證與錯誤顯示

> _待寫:`Validators` 前端驗證 vs 後端 `OnValidate` 400 回填;`hcs-error-summary` 怎麼顯示。連 [validation-errors](../core/validation-errors.md)。_

## 攔截與覆寫點(opt-in)

> _待寫:本節總覽四個表單相關 opt-in,逐一給去敏 demo。_

### 存檔前攔截:`HCS_FORM_SUBMIT_SERVICE`

> _待寫:`IFormService` 介面、平台跑迴圈、回 `false` 中止存檔;去敏 demo(實作 + provider multi)。2FA 掛這個。_

### 載入後攔截:`HCS_FORM_MODEL_LOADED_SERVICE`

> _待寫:model 反序列化後、render 前改值;去敏 demo。_

### RichText 編輯器設定:`CKEDITOR_CONFIG`

> _待寫:fallback `DefaultCkEditorConfig`;覆寫寫法。_

### 區塊預設展開:`HCS_EXPANSION_PANEL_DEFAULT_OPTIONS`

> _待寫:預設 `{expand:true}`;何時關。_

## 替換整個表單頁(Component Override)

> _待寫:不掛攔截、直接換掉預設表單元件的情境;`createRoute({ xxxForm: () => import(...) })` 去敏 demo。連 [frontend](../frontend.md) 慣例 3。_

## Gotchas

> _待寫:攔截 service 是 multi(全部都跑)、回 false 才中止;載入後攔截改的是顯示用 model 不是送出值之類的雷。_

## 關聯

> _待寫:連 frontend.md、list.md、validation-errors.md、features/2fa.md。_
