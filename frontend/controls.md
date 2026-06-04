# 輸入元件(`hcs-*-input`)

一組共用的輸入控制元件。**同一批元件在兩個地方用**:表單裡當欄位(綁 `formControlName`,見 [form](form.md)),列表查詢列裡當查詢輸入(綁 `[hcsDataGridQuery]`,見 [list](list.md))。本篇是逐項參考。

---

## 共通綁定模式

每個 `hcs-*-input` 都實作了 Angular 的 `ControlValueAccessor` + `Validator`,所以**綁定方式跟原生 input 一樣**——用 `formControlName`(或 `[formControl]` / `[(ngModel)]`),驗證錯誤會自動冒出來,不必自己接 `mat-error`。

```html
<hcs-form-page #fp [formGroup]="formGroup">
  <hcs-form-row>
    <hcs-textbox-input formControlName="Name" required>
      <label>名稱</label>
      <span hint>顯示在欄位下方的提示</span>
    </hcs-textbox-input>
  </hcs-form-row>
</hcs-form-page>
```

四個共通約定:

- **標籤**用內容投影的 `<label>`(不是 `label=` 屬性)。要 i18n 就在 `<label>` 上掛 `translate`。
- **提示**用帶 `hint` 屬性的元素(`<span hint>…</span>`),對應 Material 的 `mat-hint`。
- **`required` 接「布林或字串」**——光寫裸屬性 `required` 就成立(等同 `[required]="true"`),每個元件都有。**停用**則一律用 reactive form 的 `disable()`(所有元件都透過 `setDisabledState` 響應);只有 textbox / textarea / slide-toggle 另接裸 `disabled` 屬性,radio / checkbox / checkboxlist **沒有 `disabled` 輸入**,要停用只能靠 form control。
- **錯誤訊息自動顯示**:控制元件把外層 form control 的 errors 同步進來,透過 `errorMessage` pipe 渲染成 `mat-error`,**驗證錯誤不必手接**(驗證怎麼產生見 [validation-errors](../core/validation-errors.md))。
- **排版**:多個欄位用 `<hcs-form-row>` 包成一列;`hcs-form-page` 負責整體版面(見 [form](form.md))。

> 在列表查詢列時不綁 `formControlName`,改綁 `[hcsDataGridQuery]`(同一個元件、不同宿主),見 [list 的查詢指令客製](list.md#查詢指令客製hcsdatagridquery)。

---

## 文字類

### `hcs-textbox-input`

單行輸入。

| 屬性 | 值 | 說明 |
|---|---|---|
| `type` | `text`(預設)/ `number` / `password` / `email` / `tel` | 決定 input 型別 |
| `maxlength` / `min` / `max` | number | 長度 / 數值範圍 |
| `autocomplete` | `off`(預設)/ `on` | |
| `required` / `disabled` | bool | |

`type="number"` 時值自動轉成數字(轉不出來→`null`,**不是空字串**)。

```html
<hcs-textbox-input formControlName="Price" type="number" [max]="9999">
  <label>單價</label>
</hcs-textbox-input>
```

### `hcs-textarea-input`

多行輸入,預設 `textareaAutosize=true`(隨內容長高)。屬性 `maxlength` / `required` / `disabled`。

```html
<hcs-textarea-input formControlName="Remark">
  <label>備註</label>
</hcs-textarea-input>
```

---

## 選擇類

選項一律用內容投影的 `<hcs-option [value]="…">` 宣告,可 `*ngFor` 動態產生;**數字列舉**可搭 `enumOptions` pipe 直接展開(字串列舉不支援——`enumOptions` 只挑得出數字 key,字串列舉會得到空清單)。

### `hcs-select-input`

下拉選單。屬性 `multiple`(多選→值變陣列)、`required`、`disabled`。放一個空 `<hcs-option></hcs-option>` 當「不選」。

```html
<hcs-select-input formControlName="CategoryId">
  <label>分類</label>
  <hcs-option></hcs-option>
  <hcs-option *ngFor="let c of categories" [value]="c.id">{{ c.name }}</hcs-option>
</hcs-select-input>

<!-- 列舉直接展開 -->
<hcs-select-input formControlName="Status">
  <label>狀態</label>
  <hcs-option *ngFor="let o of (StatusEnum | enumOptions)" [value]="o.value">{{ o.text }}</hcs-option>
</hcs-select-input>
```

### `hcs-radiolist-input`

單選按鈕群。屬性 `layout`(`row` 預設 / `column`)、`required`、`desabledShowAll`(預設 `true`)。

```html
<hcs-radiolist-input formControlName="Gender" layout="column">
  <label>性別</label>
  <hcs-option [value]="1">男</hcs-option>
  <hcs-option [value]="2">女</hcs-option>
</hcs-radiolist-input>
```

### `hcs-checkbox-input`

**單一**勾選框。**最多一個 `<hcs-option>`**(超過一個會 console error):

- **不給 option** → 值是 `boolean`(勾=`true`)。
- **給一個 `<hcs-option value="X">`** → 勾選時值是 `X`、未勾是 `undefined`(用於「勾了就送某個固定值」)。

```html
<!-- 布林 -->
<hcs-checkbox-input formControlName="IsEnabled">
  <label>啟用</label>
</hcs-checkbox-input>
```

### `hcs-checkboxlist-input`

多選清單。屬性 `layout`(`row`/`column`)、`required`,以及決定**值的形狀**的 `mode`——這是最容易接錯後端型別的一點:

| `mode` | 送出的值 | 對應後端 |
|---|---|---|
| `value`(預設) | 被選中的 value **陣列**,如 `[1, 3]` | 集合 / 多對多 |
| `map` | `{ value: bool }` **物件**,如 `{1:true, 2:false}` | 旗標物件 |
| `flag` | 各選中 value 的**整數加總**,如選 1+4 → `5` | `[Flags]` 列舉 |

每個 `<hcs-option>` **都必須有 `value`**(漏了會 console error 且不渲染)。`flag` 是直接把選中的 value 相加——**只有 option value 用 2 的次方(1/2/4/8…)時,加總才剛好等於位元 OR**;這正是 `[Flags]` 列舉的慣例,但若你自訂了會位元重疊的 value,加總結果就不是你想的 OR。

```html
<!-- [Flags] 列舉:option 的 value 是 2 的次方 -->
<hcs-checkboxlist-input formControlName="Permissions" mode="flag" layout="column">
  <label>權限</label>
  <hcs-option [value]="1">讀</hcs-option>
  <hcs-option [value]="2">寫</hcs-option>
  <hcs-option [value]="4">刪除</hcs-option>
</hcs-checkboxlist-input>
```

### `hcs-slide-toggle-input`

開關(值 `boolean`)。屬性 `required`、`disabled`、`color`(Material `ThemePalette`)。功能等同布林 checkbox,差在 UI 是滑動開關。

---

## 日期時間類

值都是 `Date`。**日期顯示語系自動跟著平台目前語系切換**(切語言時 `DateAdapter` 同步換 locale,不必自己處理)。

### `hcs-date-input`

純日期。屬性 `required`。

### `hcs-datetime-input`

日期 + 時間。屬性 `required`、`openOnFocus`(預設 `false`,聚焦即開面板)、`timeInterval`(分鐘間隔,預設 `1`)。

### `hcs-time-input`

純時間。屬性 `required`。

```html
<hcs-form-row>
  <hcs-date-input formControlName="StartDate" required><label>起始日</label></hcs-date-input>
  <hcs-datetime-input formControlName="Deadline" [timeInterval]="15"><label>截止</label></hcs-datetime-input>
</hcs-form-row>
```

---

## 其他

### `hcs-color-input`

色彩選擇。`type` 決定**存進 form 的序列化形狀**:

| `type` | 值 |
|---|---|
| `rgba`(預設) | `rgba(…)` 字串 |
| `hex` | `#RRGGBB` 字串 |
| `json` | 整個色彩物件的 JSON 字串 |
| `obj` | 色彩物件 |

另有 `touchUi`(預設 `false`,觸控版面板)。後端欄位是字串就用 `hex` / `rgba`,要完整資訊就 `json` / `obj`。

```html
<hcs-color-input formControlName="ThemeColor" type="hex"><label>主題色</label></hcs-color-input>
```

> 另有 `hcs-select-pad`(快捷鍵選擇盤,`hotKey` + `*hcsSelectPadOption`)供高頻鍵盤操作場景,用法見 [form](form.md)。

---

## Gotchas

- **裸 `required` 就生效**:`required` 接 `boolean | string`,寫 `required`(無值)等於 `true`——這也是為什麼不能用 `required="false"` 字串關閉(非空字串會被當真;要關就別寫,或 `[required]="false"`)。
- **`disabled` 不是每個元件都有**:radio / checkbox / checkboxlist 沒有 `disabled` 輸入,停用一律靠 reactive form 的 `disable()`(所有元件都響應);別假設裸 `disabled` 屬性到處能用。
- **textbox / textarea 的「停用」其實是 `readonly`**:停用時綁的是 `[readonly]` 而非原生 `disabled`——欄位**仍可聚焦、可選取文字,只是不能改值**。若你依賴 disabled 的樣式或 tab 跳過行為,這裡會不如預期。
- **`type="number"` 的 textbox 清空是 `null` 不是 `''`**:驗證 / 送出時注意這個型別差。
- **checkbox 的 option 只能一個**:要多選用 `checkboxlist`;checkbox 給 option 是「勾→送固定值」的特例,不是多選。
- **checkboxlist 的 `mode` 要對齊後端型別**:後端是 `[Flags]` 列舉就用 `flag`、是集合就用 `value`;接錯送出的 JSON 形狀就不對。每個 option 必須有 `value`。
- **日期語系免處理**:`hcs-date-input` 系列會跟著 i18n 語系自動換顯示格式;locale 格式設定見 [core/i18n-system](../core/i18n-system.md)。
- **標籤用 `<label>` 投影、不是屬性**:寫成 `label="名稱"` 不會顯示。

---

## 關聯

- 這些元件在表單怎麼組、進階欄位(參照選擇 / 子表 / 檔案 / RichText)— [form](form.md)
- 同一批元件當查詢輸入、`hcs-sort-input` 排序、查詢指令客製 — [list](list.md)
- 驗證錯誤怎麼產生與顯示 — [core/validation-errors](../core/validation-errors.md)
- 日期 / 數字 locale 格式 — [core/i18n-system](../core/i18n-system.md)
