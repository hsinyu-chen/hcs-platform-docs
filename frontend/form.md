# 表單頁(BaseFormComponent)

一個編輯頁 = `extends BaseFormComponent<T>` + 一個 reactive form + 一個 `<hcs-form-page>`。基底與 `hcs-form-page` 把樣板都包了:依路由判斷新增 / 編輯 / 複製、載入既有資料、送出(create / update)、把後端 400 驗證錯誤回填到欄位。你只寫**這張表單長什麼樣**(reactive form + 欄位元件)。對應後端 [entity-api](../core/entity-api.md) 的 Post / Put。

---

## BaseFormComponent 本質

```typescript
@Component({
  selector: 'invoice-form',
  templateUrl: './invoice-form.component.html',
  providers: [BaseComponentService, { provide: HCS_FUNCTION_NAME, useValue: 'Sales.Invoice' }, Permission],
})
export class InvoiceFormComponent extends BaseFormComponent<Invoice> {
  formGroup = new FormGroup({
    Id: new FormControl(),
    No: new FormControl(null, Validators.required),
    Amount: new FormControl(0),
  });

  constructor(service: BaseComponentService) {
    super(service, Invoice);   // ← 傳 entity 型別,基底自動建 datasource
  }
}
```

`BaseFormComponent<T>` 替你準備兩樣:

- **`datasource`**:`DataSourceFactory.getDataSource(T)` 自動對接後端 `/api/odata/<T>`,**不必手寫 API 呼叫**(見 [frontend](../frontend.md) 慣例 7)。
- **`formPage`**:`@ViewChild` 抓到模板裡的 `<hcs-form-page>`,需要時可程式呼叫(如 `formPage.saveForm()`)。

你只負責 `formGroup`(這張表單的 reactive form)。基底**不會**幫你建 form——欄位、驗證器都由你宣告。

---

## 建一個表單頁

1. **component**:`extends BaseFormComponent<T>`、`super(service, T)`、component-level provide `BaseComponentService`(基底依賴,**非 root、漏了會 `NullInjectorError`**)+ `HCS_FUNCTION_NAME`(對應後端功能碼,權限與 i18n 前綴都靠它,見 [permissions](../core/permissions.md))。
2. **reactive form**:`formGroup = new FormGroup({...})`,欄位掛 `Validators`。
3. **template**:`<hcs-form-page [formGroup]="formGroup" [dataSource]="datasource">` 內用 `<hcs-form-row>` 排欄位元件。

```html
<hcs-form-page [formGroup]="formGroup" [dataSource]="datasource" fieldI18nPrefix="models.Sales.Invoice">
  <hcs-form-row>
    <hcs-textbox-input formControlName="No" required><label>單號</label></hcs-textbox-input>
    <hcs-textbox-input formControlName="Amount" type="number"><label>金額</label></hcs-textbox-input>
  </hcs-form-row>
</hcs-form-page>
```

基本輸入欄位用 `hcs-*-input`,逐項參考見 [controls](controls.md)。

---

## hcs-form-page 與生命週期

`<hcs-form-page>` 是表單容器,**它驅動整個流程**——你不必自己 call API:

```
路由帶 id?       → state='edit' → datasource.get(id)  → loadModel
路由帶 copy?     → state='copy' → datasource.get(copy)→ 清主鍵 → loadModel
都沒帶           → state='new'  → loadModel(空表單)
                                       │
                          [HCS_FORM_MODEL_LOADED_SERVICE / modelPreprocess]
                                       │
                                   填入 formGroup + 套 $/$$ 帶入值
按存檔 → saveForm:驗證閘門 → [HCS_FORM_SUBMIT_SERVICE / modelPostprocess]
                                       │
                          state==='edit' ? update(id) : create
```

`state` 有三種:**`new`(新增)/ `edit`(編輯)/ `copy`(複製——載入來源資料但清掉主鍵,存成新筆)**。

常用 `@Input`:

| 屬性 | 預設 | 作用 |
|---|---|---|
| `formGroup` / `dataSource` | — | 必給;reactive form 與資料來源 |
| `fieldI18nPrefix` | `models.<功能碼>` | 欄位標籤 / 錯誤訊息的 i18n 前綴 |
| `backWhenSaved` | `true` | 存檔成功後自動返回列表 |
| `backToView` | `false` | 返回時去檢視頁而非列表 |
| `showBackButton` / `showCopyButton` / `showModifyButton` / `showResetButton` | `true` | 工具列按鈕開關 |
| `modelPreprocess` / `modelPostprocess` | — | 載入後 / 送出前改 model 的 inline hook(見[攔截](#攔截與覆寫點opt-in)) |
| `queryParamsHandling` | `merge` | 存檔返回列表時 URL query 的處理(`merge`/`preserve`/`''`),用來保留列表的查詢參數 |

---

## 對話框模式(同一個表單元件也能當 dialog 開)

`Dialog.form(data, FormComponent)` 直接把表單元件開成 Material 對話框——**同一個 `XxxFormComponent` 不必改寫**,既能掛路由當整頁、也能彈窗開:

```typescript
constructor(private dialog: Dialog) {}

openEdit(id: number) {
  this.dialog.form({ id, isReadonly: false }, InvoiceFormComponent);
}
```

`IFormDialogData` = `{ id?, copyId?, fillProperty?, isReadonly }`(`isDialog` 由 `Dialog.form` 自動設 `true`)。form-page 偵測到 `isDialog` 後改從 `dialogData` 拿 `id`/`copyId`/`isReadonly`/`fillProperty`(取代路由的 params/queryParams),`loadModel`/`saveForm` 完全同一套:

- `id` → edit、`copyId` → copy、都沒給 → new(與路由版一致)。
- `isReadonly: true` → 整張 form `disable()`、純檢視。
- `fillProperty` → 等同路由的 `$`/`$$`(見[下節](#url-帶入欄位值-預填--鎖定)),程式帶入 / 鎖定欄位。

> ⚠️ **頁面狀態記憶用「路由」當 key**:列表的查詢條件 / 排序 / 捲動 / 分頁由 `PageStatusHolder` 依**目前路由**記住、回訪自動還原。對話框疊在某列表頁上時**共用那個 host 路由的 key**,所以**每個路由只存得下一組**——對話框裡若也放 `hcs-data-grid`,務必給它獨立的 `statePrifix`(預設 `'0'`),否則會和底下列表的狀態互相覆蓋。完整狀態記憶機制見 [list](list.md)。

---

## 進階欄位元件

比簡單輸入重、幾乎只在表單用的欄位,各自綁 `formControlName`:

### `hcs-reference-input` — 參照選擇(外鍵)

挑另一個 entity 當外鍵。值是被選 entity 的主鍵;`(entityChange)` 另外吐出整個被選 entity。要給 `[pickerSetting]`(彈窗挑選來源),可選 `[searchSetting]`(自動完成搜尋)。

```html
<hcs-reference-input formControlName="CustomerId" [pickerSetting]="customerPicker" required>
  <label>客戶</label>
</hcs-reference-input>
```

其他屬性:`[settings]="{picker, search}"`(一次給兩個)、`autocompleteSearch`(預設 `false`;搜尋只剩一筆時自動帶入)、`showClearButton`(預設 `true`)、`searchDelay`(預設 `500` ms)。

### `hcs-child-panel` — 主檔明細(一對多)

在同一張表單編輯子集合(對應後端 `SaveChildsFor`)。值是子物件陣列。給 `[type]` 子 entity 型別,它依該型別的屬性自動為每列建 `FormGroup`;子列版面用內容投影的 `<ng-template>`。

```html
<hcs-child-panel formControlName="Items" [type]="InvoiceItem">
  <label>明細</label>
  <ng-template let-row let-i="index">
    <!-- row($implicit)= 該列 FormGroup,外層已套好 [formGroup],直接 formControlName -->
    <hcs-textbox-input formControlName="ProductName"><label>品名</label></hcs-textbox-input>
    <hcs-textbox-input formControlName="Qty" type="number"><label>數量</label></hcs-textbox-input>
  </ng-template>
</hcs-child-panel>
```

屬性:`label`、`buttonAlign`(`left`/`right`)、`showDeleteButton`、`validatorMap` / `formStateMap`(逐欄驗證器 / 初值)。**複製(`copy`)時自動清掉每列的主鍵**,確保存成新明細。子表無效時整個欄位回 `childItemInvalid` 錯誤、擋下存檔。

### `hcs-file-input` — 檔案欄位

檔案上傳欄位,綁 `formControlName`;搭配後端 `SetupFilePipes` 的確認 / 孤兒清理生命週期。機制(`IFileStorage`、確認流程)見 [core/file-upload](../core/file-upload.md)。

### `hcs-ckeditor` — RichText(WYSIWYG)欄位

CKEditor 富格式欄位,綁 `formControlName`;編輯器設定見下方 [`CKEDITOR_CONFIG`](#richtext-編輯器設定ckeditor_config)。

---

## URL 帶入欄位值:`$` 預填 / `$$` 鎖定

導頁時用 query string 把欄位帶進來——`?$field=值` 只在**新增**表單預填(可改的預設值)、`?$$field=值` 預填**並鎖死**(`disable()`,任何狀態都鎖);兩者依 entity 屬性型別自動 Boolean / Number 轉型。對話框表單則改用 `dialogData.fillProperty` 程式帶入,同一套規則。

這就是 [multi-tenant](../core/multi-tenant.md)「UI 把 `OrgId` 拉出來」的前端機制——例如從列表「新增」按鈕導到 `?$$OrgId=5`,把固定組織脈絡帶進表單並鎖住。列表查詢列也用同一套(無 state),見 [list](list.md)。

```typescript
// 列表「新增子項」按鈕:把父 id 帶進新表單並鎖死
[routerLink]="['../new']" [queryParams]="{ '$$ParentId': parent.Id }"
```

---

## 驗證與錯誤顯示

存檔時 `hcs-form-page` 先把整張 form 標記 touched、跑 `formGroup.valid`:

- **前端驗證**(`Validators`):不過就**不送出**,錯誤直接顯示在各欄位。
- **後端驗證**:送出後若後端 `OnValidate` 回 **400**,錯誤被 `ErrorHelper` 收集、回填到對應欄位 + `hcs-error-summary` 總覽。

`(modelValidatorError)` 事件:**前端驗證失敗** emit `true`、**存檔成功** emit `false`(後端 400 走另一條 error 路徑,只收集錯誤、不 emit 此事件)。前後端驗證錯誤怎麼產生與顯示,見 [validation-errors](../core/validation-errors.md)。

---

## 攔截與覆寫點(opt-in)

### 存檔前 / 載入後攔截:`HCS_FORM_SUBMIT_SERVICE` / `HCS_FORM_MODEL_LOADED_SERVICE`

兩個 token **共用同一個 `IFormService<T>` 介面**,靠 `action` 參數區分,都是 multi(可掛多個,依序跑):

```typescript
export interface IFormService<T> {
  // action='loaded'(載入後)或 'submit'(送出前);回 false 中止該動作
  runAsync(model: T, state: 'edit'|'copy'|'new', action: 'loaded'|'submit',
           ds: IHttpDataSource<T>): Promise<T | false>;
}
```

- **`HCS_FORM_MODEL_LOADED_SERVICE`**:`loadModel` 時跑(`action='loaded'`),可改載入的 model。回 `false` → 中止載入。
- **`HCS_FORM_SUBMIT_SERVICE`**:`saveForm` **驗證通過後**跑(`action='submit'`),可改送出的 model。回 `false` → 中止存檔。2FA 等掛這裡攔截送出。

```typescript
@Injectable()
export class StampUserService implements IFormService<any> {
  async runAsync(model: any, state: string, action: string) {
    if (action === 'submit') model.EditedBy = currentUser();
    return model;
  }
}
// 消費端 provider(multi:true 不可省)
{ provide: HCS_FORM_SUBMIT_SERVICE, useClass: StampUserService, multi: true }
```

> 同一件事的 **inline 版**是 `hcs-form-page` 的 `[modelPreprocess]`(=loaded)/ `[modelPostprocess]`(=submit)`@Input`——只想改一頁、不想做成全域 service 時用它。三層執行序是 **token service → `formPage.modelPreprocessHandlers`/`modelPostprocessHandlers`(public 陣列,可程式 `push` 多支)→ inline `@Input`(一支)**;要單頁又動態多支就用中間那層。

### RichText 編輯器設定:`CKEDITOR_CONFIG`

provide 一份 CKEditor 設定覆寫預設;沒 provide 時 `hcs-ckeditor` 落回 `DefaultCkEditorConfig`(`@Optional` 注入)。

```typescript
{ provide: CKEDITOR_CONFIG, useValue: { toolbar: [...], height: 300 } }
```

### 區塊預設展開:`HCS_EXPANSION_PANEL_DEFAULT_OPTIONS`

表單摺疊區塊的預設展開狀態(預設 `{ expand: true }`)。

---

## 替換整個表單頁(Component Override)

不掛攔截、要整頁換掉預設表單時,用模組的 component override——`createRoute({ ... })` 傳入 lazy-loaded 元件,路由 / 權限路徑 / i18n key 都保留,不必 fork SDK(見 [frontend](../frontend.md) 慣例 3)。

```typescript
HcsBasicProviderModule.createRoute({
  platformUserForm: () => import('./custom-user-form.component').then(x => x.CustomUserFormComponent),
})
```

---

## Gotchas

- **基底不建 form**:`BaseFormComponent` 只給 `datasource` / `formPage`,`formGroup` 要自己宣告;忘了建會整頁空白。
- **送出用 `getRawValue()`**:含被 `disable()` 的欄位(例如 `$$` 鎖定的)也會一起送出——鎖欄位的值仍進 payload,這是刻意的(帶固定脈絡)。
- **攔截 service 回 `false` 會中止**:`submit` 中止存檔、`loaded` 中止載入(`loaded` 中止時表單不會被填);別在 service 裡不小心回到 `false`。
- **`copy` 會清主鍵**:`copy` 狀態載入來源後清掉 identity（含 `child-panel` 每列),存成新筆;若你的 entity 主鍵不是慣例名稱,確認 `dataSource.identityProperty` 正確。
- **service 是全域、preprocess 是單頁**:同一個 `HCS_FORM_SUBMIT_SERVICE` 會套到**所有**表單;只想改一頁用 `[modelPostprocess]`,別 provide 全域 service。

---

## 關聯

- 基本輸入欄位逐項 — [controls](controls.md)
- 列表頁(對應 Query 端點)、`$`/`$$` 在查詢列的用法 — [list](list.md)
- 前後端驗證錯誤 — [core/validation-errors](../core/validation-errors.md)
- 檔案欄位的後端機制 — [core/file-upload](../core/file-upload.md)
- 後端 CRUD 管線 / 生命週期 hook — [core/entity-api](../core/entity-api.md)
- 2FA 掛 `HCS_FORM_SUBMIT_SERVICE` 攔截 — [features/2fa](../features/2fa.md)
