---
name: platform-overview
description: HCS Platform（ASP.NET Core + Angular + OData + EF Core 的宣告式 ERP 開發平台）的能力總覽與文件索引。當在 HCS Platform 專案裡開發、或要寫後端模組 / entity / API / 前端列表表單 / 權限 / 多租戶相關 code、或被問「平台怎麼做 X」時，先讀這篇搞清楚有哪些內建能力、該往哪篇文件深入——不要憑空重造平台已經提供的東西。
---

# HCS Platform 能力總覽與文件索引

## 最重要的一件事

**這個平台有完整的參考文件，動手寫 code 前先去讀對應那篇。** 平台把一整套 ERP 樣板（Controller、Repository、DTO、OData、權限中介、多租戶過濾、前端 API client）收進框架，**九成「我需要寫一個 X」的需求，平台已經有宣告式的做法**。憑印象硬寫，結果通常是繞過平台、重造輪子、或踩到框架的既定慣例。流程永遠是：**先認得能力 → 讀那篇 doc → 再寫**。

> 文件是參考資料，可能與現狀有出入；**與 code 衝突時一律以 code 為準**。

## 核心心智模型：洋蔥，不是開關

平台的每個高階 API 都是用低一階的**公開** API 組出來的，內建沒有特權（`AddEntityApi` 本身就是用公開 pipe 組件串的一條 pipeline）。所以「最高階的不合用」**永遠不等於**「只能整個自幹」——往下剝一層、換掉需要的那一小塊、其餘照用。**永遠只客製最小的那一塊**；繞過平台直打 DB / HTTP 等於連帶關掉租戶過濾、權限與稽核。

一切擴充都靠 **pipe（`DIPipe`）**：把行為拆成短小單一職責的步驟，掛到 CRUD / 驗證 / 查詢 / 服務注入的生命週期切點，參數自動 DI。讀懂 pipe 就讀懂平台 → [core/pipe](../core/pipe.md)。

---

## 我要做 X → 走哪

### 建專案 / 模組 / 功能（操作型 how-to skill）

| 要做的事 | skill |
|---|---|
| 從零起一個新 app（後端 host + 前端 + 第一個 admin + DB schema） | [platform-create-project](platform-create-project.md) |
| 加一個新模組（entity 的容器：後端 module + 前端 feature/menu） | [platform-create-module](platform-create-module.md) |
| 往模組加一顆 entity（標準 CRUD：後端 API + 前端列表/表單，含常見客製） | [platform-add-entity](platform-add-entity.md) |
| 加一支非標準 CRUD 的自訂端點（結帳/簽核/排序/批次/視圖） | [platform-custom-api](platform-custom-api.md) |

### 平台機制（概念 + API 深入 doc）

| 主題 | 一句話 | doc |
|---|---|---|
| Pipe | 平台擴充的核心基元，`.Pipe` / `.SwitchCase` / 自動 DI | [core/pipe](../core/pipe.md) |
| Entity API | `AddEntityApi` 一行長 CRUD 五件套 + 生命週期 hook + 交易 | [core/entity-api](../core/entity-api.md) |
| Flow API | 自訂端點：用 pipe 組件自組管線，仍在權限/交易框架內 | [core/flow-api](../core/flow-api.md) |
| 驗證錯誤 | `OnValidate` → 400 → 前端 `hcs-error-summary`；`Unique` 等 | [core/validation-errors](../core/validation-errors.md) |
| Data Pipes | 寫入/查詢的橫切（多租戶過濾、audit、`ITable<T>`） | [core/data-pipes](../core/data-pipes.md) |
| 權限 | `AddModuleFuncion` / `AddPermission` / `AddRole` / `$expand` 白名單 | [core/permissions](../core/permissions.md) |
| 多租戶 | 組織樹 + 「讀能上下、寫只能往下」 | [core/multi-tenant](../core/multi-tenant.md) |
| 登入 / Token | 多登入模式、JWT、即時撤銷、OrgKey | [core/login](../core/login.md) |
| 檔案上傳 | `IFileStorage`（本機 / Azure Blob）+ 確認/孤兒清理 | [core/file-upload](../core/file-upload.md) |
| i18n | 字典合併、link 語法、欄位名 key 前綴 | [core/i18n-system](../core/i18n-system.md) |

### 前端 SDK

| 主題 | doc |
|---|---|
| 前端 SDK 總覽（library 清單、慣例、`@hcs/core`、新增功能步驟） | [frontend](../frontend.md) |
| 列表 / 資料表格（`hcs-data-grid`、`BaseListComponent`） | [frontend/list](../frontend/list.md) |
| 表單（`hcs-form-page`、`BaseFormComponent`、生命週期） | [frontend/form](../frontend/form.md) |
| 輸入元件（各式 `hcs-*-input`） | [frontend/controls](../frontend/controls.md) |
| 外殼 / 版面 | [frontend/shell](../frontend/shell.md) |
| 登入（前端） | [frontend/login](../frontend/login.md) |

### 可選模組（各對應一個 `AddXxxModule()`）

| 模組 | doc |
|---|---|
| Basic（User / Group / Organization） | [modules/basic](../modules/basic.md) |
| 簽核流程 | [modules/approval-flow](../modules/approval-flow.md) |
| App 版本管理 | [modules/app-update](../modules/app-update.md) |
| 字典 / 代碼表 | [modules/code-table](../modules/code-table.md) |
| 系統稽核 | [modules/system-logging](../modules/system-logging.md) |
| 2FA | [modules/2fa](../modules/2fa.md) |

---

## 常見「不要自己造」清單

寫之前先確認平台是不是已經有：

- 不寫 Controller / Repository / DTO / AutoMapper → `AddEntityApi`（執行期動態生成）。
- 不繼承覆寫框架行為 → `Pipe(...)` 掛 hook。
- 不自己拼 `OrgId` 查詢條件 → 多租戶過濾自動套（entity 實作 `IOrganized`）。
- 不手刻權限檢查 → `AddModuleFuncion` 宣告。
- 不前端手刻 raw API URL → `@ApiEntry` + datasource（自訂端點才用 `HttpClient` 打 `api/entity/<name>`）。
- 不直接碰 `localStorage` / `sessionStorage` → 走平台的使用者狀態儲存。
- 不寫 EF migration 也能跑（啟動自動同步 schema）；要帶上 prod 才需 migration，見 [platform-create-project](platform-create-project.md)。

完整能力清單與專案地圖見文件庫首頁 [README](../README.md)。
