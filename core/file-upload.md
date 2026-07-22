# 檔案上傳與儲存

平台內建一套完整的檔案上傳機制:前端 `hcs-file` 元件上傳拿到 **key**,key 存進 entity 的字串欄位;後端把內容交給可插拔的 `IFileStorage`(本機磁碟或 Azure Blob),並自動處理**確認 / 孤兒清理 / 更新時清舊檔**的生命週期。你不用自己寫上傳 controller,也不用手動管檔案何時該刪。

---

## 架構

- **`IFileStorage`**:儲存後端抽象(`Create` / `Open` / `Update` / `Delete` / `Exists`)。`Create` 收一個 `Stream`、回傳一個 GUID **key**。內容存在後端,key 是唯一識別。
- **兩個內建後端**,啟動時擇一掛載:

  ```csharp
  builder.UseLocalFileStroage(@"D:\app-files");                       // 本機磁碟
  builder.UseAzureBlobStorage(connectionString, "my-container");      // Azure Blob
  ```

  > 方法名 `UseLocalFileStroage` 的拼字就是這樣(`Stroage`),照用。兩者都有「用 `IServiceProvider` 動態解析路徑 / 連線字串」的多載。

- **metadata 存 DB**:每個上傳檔對應一筆 `PlatformFile` 記錄(key、原始檔名、大小、MIME、上傳時間、目錄 `Dir`、`Confirmed` 旗標)。`IFileManager` 是「儲存後端 + DB 記錄 + 快取」的統合層,controller 與 pipe 都透過它操作(另有 `Rename` / `Move` 維運操作)。

---

## 檔案怎麼被 entity 引用

entity 上用**字串欄位**存 key(多檔以逗號分隔),再在 entity API 一行掛上生命週期管線:

```csharp
b.AddEntityApi<long, Article>(opt =>
{
    opt.SetupFilePipes(x => x.CoverImage);      // 一般檔案欄位
    opt.ConfirmCkeditorImages(x => x.Content);  // 內文 HTML 內嵌圖(CKEditor)
});
```

- `SetupFilePipes(x => x.CoverImage)`:接管該欄位的**確認 + 更新/刪除時清舊檔**(見下方生命週期)。
- `ConfirmCkeditorImages(x => x.Content)`:掃 HTML 內**所有** `api/file/{key}` 連結(不限圖片,夾帶的檔案連結也算),**只做確認**(不做更新時的差異刪除)。

---

## 上傳生命週期(核心非直覺)

檔案的去留靠 `PlatformFile.Confirmed` 旗標決定,流程分四段:

1. **上傳當下** — `POST /api/file` 把內容存進後端、建一筆 `Confirmed=false` 的暫存記錄。檔案**立刻可預覽 / 取用**,但還沒被任何 entity 認領。
2. **entity 存檔(認領)** — `SetupFilePipes` 掛的 pipe 在 `OnCreated` / `OnUpdated` 把欄位裡的 key 逐個 `ConfirmFile` → `Confirmed=true`。此後這個檔被視為「已被引用」。
3. **更新時清舊檔** — 舊 key 在 `OnKeySet` 先暫存(避免更新途中讀不到舊值),`OnUpdated` 才比對「舊 key 集合」與「新 key 集合」;被移除掉、**且沒有其他資料列還引用**的 key,**立即刪除**(內容 + 記錄)。刪除 entity 時 `OnDeleted` 同理清掉該列的檔。
4. **孤兒清理** — `ClearUnConfirmed()` 刪掉**未確認(`Confirmed=false`)且上傳超過 1 天**的檔。這些是「上傳了但 entity 從沒存成功 / 使用者放棄」的暫存檔。

> ⚠️ **`ClearUnConfirmed()` 不會自動跑**——平台只提供這個方法,**你要自己排程呼叫**(背景服務 / 排程工作 / `Hcs.Console` 定時任務)。不排,暫存孤兒檔會永久累積。背景服務骨架(scope 取 `DbContext`、`stoppingToken`、例外處理)見 [background-services](background-services.md)。
>
> CKEditor 那條(`ConfirmCkeditorImages`)只做「確認」、**沒有更新時的差異刪除**;內文裡被刪掉的圖只能靠上面的 1 天孤兒清理回收(前提是它一開始就沒被 confirm),已 confirm 的內嵌圖不會被自動清。

---

## HTTP API

| 端點 | 方法 | 授權 | 用途 |
|---|---|---|---|
| `/api/file/{*dir}` | POST | `[Authorize]` | 上傳(multipart `file`,`dir` 為選填目錄,catch-all 可多層如 `a/b/c`),回傳 key(純文字) |
| `/api/file/{key}` | GET | `[Authorize]` | 取 metadata(`PlatformFileInfo`) |
| `/api/file/{key}/{*name}` | GET | `[AllowAnonymous]` | 檢視檔案(ETag + 快取 30h),`name` 只是友善檔名 |
| `/api/file/download/{key}/{*name}` | GET | `[AllowAnonymous]` | 下載(以 attachment,ETag + 快取 30h) |

> 檢視 / 下載端點的快取帶 `Vary: User-Agent`(`ResponseCache`),中間 proxy / CDN 會以 User-Agent 區隔快取。

> **檢視 / 下載端點是匿名的**:知道 key 即可取檔(key 是 GUID,難猜但**本身不是授權**)。真要對檔案做存取控制,別只靠 key 的不可猜性。metadata 端點(`/api/file/{key}`)才需登入。

---

## 前端

- **`hcs-file` / `hcs-file-input`**:檔案上傳 / 預覽 / 管理元件,實作 `ControlValueAccessor`,直接綁 reactive form。值是**逗號分隔的 key 字串**(對應後端 entity 欄位)。
- 主要 `@Input`:`layout`(text / preview / both)、`previewSize`、`multiple`、`accept`、`dirPath`(上傳目錄)、`capture`(行動裝置相機 / 麥克風原生擷取)、`maxFileSize`(前端大小上限,預設 5MB)、`cropperSettings`(影像裁切比例)、`imageCompressQuality`、`maxResolution`。
- **client 端影像處理**:上傳前可壓縮(JPEG / BMP **一律重編碼為 JPEG,檔名副檔名也改成 `.jpg`**)、裁切、依 `maxResolution` 縮放。多檔可排序 / 刪除。
- **`FileManager` service**:`upload(file, name, dir)` 回 key、`getFileInfo(keys)` 取 metadata(per-key 快取,**換頁時自動清空**)、`getUrl/getFileUrl/getStyleUrl` 組出檢視 URL、`downloadFile(...)` 觸發瀏覽器下載。

---

## CKEditor 整合

- 編輯器內插入圖片時,自動上傳到 `/api/file/Ckeditor`(目錄 `Ckeditor`),內文存成 `<img src="/api/file/{key}/...">`。
- 表單存檔時,`ConfirmCkeditorImages(x => x.Content)` 用正則從內文 HTML 撈出所有 `api/file/{key}` 並 `ConfirmFile`,把這些圖轉正。
- 未被任何內文引用的上傳圖,靠 1 天孤兒清理回收。

---

## Gotchas 一覽

- **`ClearUnConfirmed()` 不自動** → 一定要自己排程,否則暫存孤兒檔永久累積。
- **後端上傳無大小限制**(`[DisableRequestSizeLimit]`)——只有前端 `maxFileSize` 擋。要硬限制得自己在後端加。
- **多檔 = 單一字串欄位逗號分隔**,不是集合 / 關聯表。
- **檢視 / 下載端點匿名**:key 不可猜 ≠ 授權;敏感檔需另設存取控制。
- **跨列共用同一 key**:更新 / 刪除前會檢查是否仍被其他列引用,被引用的不刪——共用檔案安全,但表示刪除有「reference 檢查」成本。
- **無 server 端 MIME 驗證**:`ContentType` 照 `IFormFile` 原樣存,前端只用來決定預覽方式。

---

## 關聯

- `OnCreated` / `OnUpdated` / `OnDeleted` 這些掛載點怎麼來:[entity-api](entity-api.md)。
- 底層的 `.Pipe(...)` 與 DI 注入:[pipe](pipe.md)。
- 排程 `ClearUnConfirmed()` 的背景服務骨架:[background-services](background-services.md)。
