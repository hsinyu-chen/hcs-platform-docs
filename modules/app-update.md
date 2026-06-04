# AppUpdate(App 版本管理)

行動 App 的版本發佈表。後端只做一件事:**讓裝置查「我這個產品、這個平台,目前該裝哪一版」**,並把安裝檔的下載位址一起給出去。要不要更新、怎麼擋人、怎麼跳商店——都是**裝置端 App 自己決定**,平台不介入。

換句話說,這是個「版本資訊看板」,不是更新引擎。

## 何時用

- 自家 Android / iOS App 需要一支「檢查最新版」的端點,讓 App 開機時比對版本、提示使用者更新。
- 想用後台 UI(而非改設定檔 / 重佈署)維護各版本的上線時間與安裝檔。
- 想排程「未來某時刻才生效」的版本——先建好、設好 `GoLiveTime`,時間到才會被查到。

不適用的情境:這模組**不處理 Web 前端的版本更新**(只有 Android / iOS 兩個平台),也**不下發強制更新指令**——它不知道裝置現在是哪一版。

---

## 啟用(後端)

Host 一行註冊:

```csharp
builder.AddAppUpdateModule();
```

這會掛上 `AppVersion` 的五支 CRUD API(給後台維護用)、版本查詢端點,以及 `AppUpdate.AppVersion` 權限碼。

---

## 資料模型:AppVersion

一筆 `AppVersion` 描述「某產品、某平台、某版本」的一個安裝檔:

| 欄位 | 說明 |
|---|---|
| `Platform` | `Android` 或 `iOS`(列舉)。 |
| `Product` | 產品代號(字串)。同一個 host 可同時服務多個 App,用 `Product` 區分。 |
| `Version` | 版本號。**純數字字串**(前端表單只接受 `0-9`)——比的是數字大小,不是 semantic versioning。 |
| `File` | 安裝檔(走平台的檔案上傳機制,見 [file-upload](../core/file-upload.md))。 |
| `GoLiveTime` | 上線時間。只有 `GoLiveTime <= 現在` 的版本才會被查詢端點回傳。 |
| `IsEnabled` | 是否啟用。關掉就不會被查到。 |

**唯一鍵 = (`Platform`, `Product`, `Version`)**——同一產品、同一平台不能有兩筆相同版本號。

---

## 版本查詢端點

裝置端打這支(**匿名,不需登入**):

```
GET /api/app-version/{product}/{platform}.xml
```

`{platform}` 是列舉值(`Android` / `iOS`)。回傳「該產品該平台、已啟用、且已到上線時間」的**最新一版**(以 `GoLiveTime` 由新到舊取第一筆),格式是 XML:

```xml
<update>
  <version>120</version>
  <name>app_myproduct_Android_120</name>
  <url>https://your-host/api/file/{File}/app_myproduct_Android_120.apk</url>
</update>
```

- `<version>` — 最新版本號,裝置拿去跟自己現在的版本比。
- `<url>` — 安裝檔下載位址,**用當下這個請求的 host 即時組出來**(同一份資料換個網域服務,URL 自動跟著變)。
- 副檔名由平台決定:**Android → `.apk`,iOS → `.plist`**(iOS 走 OTA manifest)。

查不到符合條件的版本就回 **404**。

> [!IMPORTANT]
> **「要不要更新」的判斷在裝置端,不在平台。** 端點只告訴你「最新是第幾版、去哪下載」;裝置自己比對版本號、決定提示或強制、自己跳轉安裝。平台沒有「強制更新」旗標,也不知道裝置目前裝的是哪一版。

---

## 排程上線

`GoLiveTime` 是「生效門檻」而非單純的建立時間:把它設在未來,該筆版本會先存著、但查詢端點查不到,直到時間到。要排定一次改版,先把新版建好、`GoLiveTime` 設成預定時刻、`IsEnabled` 開著即可,不必到時候手動切換。

要臨時下架某一版,把它的 `IsEnabled` 關掉,查詢端點就會退回到次新的一版。

> [!WARNING]
> **`GoLiveTime` 是跟 `DateTime.UtcNow` 比的。** 排程時間請以 UTC 思考——若維護端以本地時間(如 GMT+8)填入、DB 卻當 UTC 存,「生效」會比你看的時鐘晚 8 小時。排定上線時刻時務必換算時區。

---

## 前端(後台維護 UI)

`@hcs/app-update` 提供的是**後台管理介面**——版本清單 + 新增 / 編輯表單,給維運人員維護版本資料用。它**不是**裝置端的更新檢查邏輯(那在各 App 專案自己實作)。

- 掛載:`HcsAppUpdateProviderModule.forRoot()`(掛選單與 i18n),路由用 `HcsAppUpdateProviderModule.defaultRoute`。
- 路由本身只擋「需登入」(`RequireLoginGuard`);`AppUpdate.AppVersion.View` 權限管的是選單顯示與後端 API 授權——沒權限的人選單看不到,硬打網址也只會看到空列表(API 擋)。
- 表單欄位對應上面的 `AppVersion`;`Version` 限純數字、`File` 限單檔(`.apk` / `.ipa` / `.plist`)。

---

## Gotchas

- **版本號是純數字,不是 semver。** 前端表單擋掉非數字輸入,所以版本比對是數字大小比較(`120 > 119`)。不要塞 `1.2.0` 這種格式。**後端 entity 沒有同一道 validator**——若用 entity API 自動化建版,要自己擋格式,否則塞了 `1.2.0` 進去查詢端點照樣回,裝置端數字比對就會壞掉。
- **查詢端點是匿名的。** 版本資訊與下載位址公開可查,不要把不該外流的東西塞進 `File`。
- **下載 URL 依請求 host 即時組出。** 走反向代理 / 多網域時,裝置拿到的 URL 會是它打進來的那個 host——通常是想要的,但要留意代理層的 host header 設定。
- **只有 Android / iOS。** 沒有 Web 平台;Web 前端的版本更新不歸這模組管。
- **同產品多平台共存。** `Product` 區分不同 App,`Platform` 區分 OS,所以一個 host 可同時服務多個 App 的多平台版本檢查。

---

## 關聯

- 安裝檔走平台檔案機制 → [file-upload](../core/file-upload.md)
- CRUD API / 權限碼 / 模組註冊的通則 → [entity-api](../core/entity-api.md)、[permissions](../core/permissions.md)
- 前端 list / form 寫法 → [frontend](../frontend.md)
