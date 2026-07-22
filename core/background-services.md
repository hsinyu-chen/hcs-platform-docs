# 背景服務(BackgroundService / IHostedService)

有些工作不由 HTTP request 觸發、要在 host 內**自己跑**:定時清孤兒檔、輪詢外部狀態、開機補一次資料。這是**通用 .NET 的 `IHostedService` / `BackgroundService`**,平台沒有另一套機制。這篇只講骨架的兩三個必踩點,以及平台面上「哪些方法需要你自己排程」。

---

## 骨架

繼承 `BackgroundService`、覆寫 `ExecuteAsync`,用 `AddHostedService` 註冊。三個非踩不可的點,漏一個就出事:

```csharp
public class EmployeeCountReporter : BackgroundService
{
    private static readonly TimeSpan Interval = TimeSpan.FromSeconds(60);

    private readonly IServiceScopeFactory scopeFactory;
    private readonly ILogger<EmployeeCountReporter> logger;

    public EmployeeCountReporter(IServiceScopeFactory scopeFactory, ILogger<EmployeeCountReporter> logger)
    {
        this.scopeFactory = scopeFactory;
        this.logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                // ① scoped 服務(DbContext 等)必須自己開 scope 取——BackgroundService 是 singleton
                using var scope = scopeFactory.CreateScope();
                var db = scope.ServiceProvider.GetRequiredService<PlatformDbContext>();
                var count = await db.Set<Employee>().CountAsync(stoppingToken);
                logger.LogInformation("employee count = {Count}", count);
            }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
            {
                break;                                  // ② 關機:乾淨結束,不當成錯誤
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "report failed");   // ③ 其他例外吞住,別讓整個 host 掛掉
            }

            try { await Task.Delay(Interval, stoppingToken); }
            catch (OperationCanceledException) { break; }
        }
    }
}
```

註冊(在 `AddHcsPlatform` 之外的一般 DI):

```csharp
builder.Services.AddHostedService<EmployeeCountReporter>();
```

### 三個必踩點

1. **scoped 服務要自己開 scope。** `BackgroundService` 是 **singleton**,不能直接注入 `DbContext`(scoped)——會拿到被固定住的實例、或啟動就炸。注入 `IServiceScopeFactory`,每輪 `CreateScope()` 取當輪要用的 scoped 服務,`using` 讓它跟著釋放。
2. **尊重 `stoppingToken`。** 迴圈條件看 `IsCancellationRequested`、所有 async 呼叫(查詢、`Task.Delay`)都把 token 傳下去。關機時 token 一觸發,`Task.Delay` 立刻拋 `OperationCanceledException`,`catch` 到就 `break`——這樣 host 能**及時**停,不用等整個 interval 走完。
3. **例外別讓 host 掛。** `ExecuteAsync` 拋出去未被接住的例外會讓整個 host 停掉。單輪的業務例外要 `catch` 起來記 log、繼續下一輪;只有「關機取消」才是預期的正常結束路徑(用 `when (stoppingToken.IsCancellationRequested)` 把它跟真正的錯誤分開)。

---

## 典型用途

- **孤兒 / 過期資料清理** — 週期性刪掉沒被認領的暫存資料。平台的檔案上傳就有一個:`ClearUnConfirmed()` **不會自動跑**,要你自己排程呼叫(背景服務 / 排程工作 / `Hcs.Console` 定時任務),不排暫存孤兒檔會永久累積,見 [file-upload](file-upload.md)。
- **輪詢** — 定期查外部系統狀態、拉待處理佇列。
- **開機補資料** — host 起來時跑一次(補快取、對帳、seed 非種子的執行期資料)。這種一次性、要跨多 instance 只跑一次的工作,可搭配平台的 `PlatformFlag`(idempotent 標記)避免重複執行。

> 需要**跨機器只跑一份**(多 instance 部署)時,`BackgroundService` 每台都會各跑一份——要用分散式鎖(`Hcs.AtomLock.Generic`)或 `PlatformFlag` 收斂成一次,見 [platform-overview](../skills/platform-overview.md) 的基礎建設清單。

---

## 關聯

- 需自己排程的平台方法(檔案孤兒清理) → [file-upload](file-upload.md)
- 定時發送 Email / 推播 → [realtime-and-messaging](realtime-and-messaging.md)
