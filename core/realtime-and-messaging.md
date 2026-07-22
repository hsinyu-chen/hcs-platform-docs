# 即時推播與多通道發訊

兩件常一起出現、但機制各自獨立的事:**往前端即時推播**(SignalR)、以及**往外部通道發訊**(Email / 簡訊 / 推播通知)。平台不替你生 Hub,但把 SignalR 需要的登入橋接(query string token)先接好了;發訊則提供一組**通道無關的抽象介面**(`IEmailMessage` 等),你注入介面、選一個實作,呼叫端不綁死某家服務。

---

## SignalR 即時推播

典型場景:某人改了資料,同組織的其他人畫面要立刻反映(重載列表、跳通知)。平台本身沒有內建 Hub——你自己寫一個 `Hub`,平台幫你把最麻煩的一塊(SignalR 的 JWT 怎麼過)處理掉。

### 1. Hub:依組織分組

推播要「只推給該看的人」,慣例是連線時把 client 加進**以組織為界的群組**,推播時只對群組送:

```csharp
using Hcs.Platform.User;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.SignalR;

[Authorize]                                    // ← 只需這個;token 怎麼進來平台已接好(見下)
public class SampleHub : Hub
{
    public const string GroupSuffix = "sample";

    private readonly IUserService userService;
    public SampleHub(IUserService userService) => this.userService = userService;

    public override async Task OnConnectedAsync()
    {
        var orgId = await GetOrgIdAsync();
        if (orgId.HasValue)
            await Groups.AddToGroupAsync(Context.ConnectionId, GroupName(orgId.Value));
        await base.OnConnectedAsync();
    }

    // token 只帶 user id(unique_name)、沒有 orgId claim → 用 user id 查回組織。
    private async Task<long?> GetOrgIdAsync()
    {
        if (long.TryParse(Context.User?.Identity?.Name, out var userId))
            return (await userService.Get(userId))?.OrgId;
        return null;
    }

    public static string GroupName(long orgId) => $"{orgId}|{GroupSuffix}";
}
```

### 2. token 怎麼進來:平台已接好 query string

WebSocket 握手**不能帶 `Authorization` header**,SignalR 一律把 JWT 放 URL 的 `?access_token=…`。平台的 `JwtBearer` 設定裡已掛好 `OnMessageReceived`,從 query string 讀 `access_token` 填進 `context.Token`(見 `Hcs.Platform.Core/ConfigurationExtensions.cs`)——所以 **Hub 上只要 `[Authorize]`,你什麼都不用做**,前端 `accessTokenFactory` 給 token 就通過。

### 3. Hub 內服務有限制:別用依賴 HttpContext 的服務

Hub 的連線生命週期**不在一次 HTTP request 裡**——WebSocket 連著、沒有「當前 request」。所以**任何依賴 `HttpContext` 的平台服務在 Hub 內會 throw**,最常踩到的是 `IUserOrganization`(它從 HTTP 脈絡取當前組織)。Hub 裡要拿組織,改走**不碰 HttpContext 的服務**:

- token 帶的 user id 在 `Context.User.Identity.Name`。
- `IUserService.Get(id)` 只依賴 `DbContext` + cache,在 Hub 的 DI scope 可安全解析,由它回 `OrgId`。

> 對比:**推播的觸發點**(下面 §5)跑在一般 HTTP request(entity API 的 `OnAfterTransaction`)裡,那裡 `HttpContext` 存在,**可以**直接用 `IUserOrganization` 取當前組織。差別只在「有沒有活的 HTTP request」,不是 SignalR 本身的限制。

### 4. 掛上 Hub 端點

在 `UseHcsPlatform` 的 `ConfigureEndpoints` 裡 `MapHub`(**非** obsolete 的直接 `app.MapHub` 寫法);`AddSignalR()` 在建構期註冊:

```csharp
builder.Services.AddSignalR();
// ...
app.UseHcsPlatform(opt =>
{
    opt.ConfigureEndpoints = e => e.MapHub<SampleHub>("/hubs/sample");
});
```

### 5. 推播點放 `OnAfterTransaction` 的理由

推播是**副作用**,落點跟 [entity-api](entity-api.md) / [flow-api](flow-api.md) 講的「副作用擺交易外、commit 後」同一條原則:交易還沒 commit 時你**不知道會不會 rollback**,要「確定真的存進去了才通知」就只能等 commit 之後(在交易內推,萬一 rollback 就通知了不存在的資料,接收端 re-fetch 也讀不到)。次要好處:SignalR 推送是 async I/O,擺交易外不會 hold 著 DB 鎖。所以推播掛 `OnAfterTransaction`——那裡 commit 已成功、且 `HttpContext` 還在(可用 `IUserOrganization` 取 orgId 對到群組):

```csharp
opt.ConfigPutApi(x => x.OnAfterTransaction(a => a
    .GetRequestModel<Department>()
    .Pipe(async (
        (IHubContext<SampleHub> hub, IUserOrganization org) svc,
        Department dept) =>
    {
        var group = SampleHub.GroupName(svc.org.Organization.Id);
        await svc.hub.Clients.Group(group).SendAsync("DepartmentChanged", dept.Name);
        return (object)dept;                     // raw afterTrans 須以 object 收尾
    })));
```

### 6. 前端連線

用 `@microsoft/signalr` 的 `HubConnectionBuilder`;**token 走 `accessTokenFactory`**(SignalR 不吃平台的 HTTP interceptor,得自己餵)。token 從 **`UserState`**(`@hcs/core/hcs-lib`)拿——它是登入 token 的單一來源,**不要自己碰 `localStorage`**:

```typescript
import { HubConnection, HubConnectionBuilder } from '@microsoft/signalr';
import { UserState } from '@hcs/core/hcs-lib';

// component 裡注入 private userState: UserState
ngOnInit(): void {
  this.hub = new HubConnectionBuilder()
    .withUrl('/hubs/sample', { accessTokenFactory: () => this.userState.token ?? '' })
    .withAutomaticReconnect()
    .build();

  this.hub.on('DepartmentChanged', (name: string) => {
    // 收到推播:跳提示 + 重載列表
    this.grid?.queueLoad();
  });

  this.hub.start().catch(() => { /* 連線失敗不影響頁面基本操作 */ });
}

override ngOnDestroy(): void {
  this.hub?.stop();          // ← 一定要關,否則換頁後連線洩漏
  super.ngOnDestroy();
}
```

**dev proxy**:相對路徑 `/hubs/…` 要能 upgrade 成 WebSocket,`proxy-config.json` 對 `/hubs` 開 `"ws": true`:

```json
"/hubs": { "target": "http://localhost:5080", "secure": false, "ws": true }
```

---

## 多通道發訊(Message 抽象)

寄 Email、發簡訊、推 App 通知——平台把每個通道抽成一個**介面**(`Hcs.Extensions.Message` 家族),呼叫端注入介面、不綁實作。要換 SMTP 供應商 / 簡訊商,換註冊的實作即可,呼叫端一行不改。

### 四個通道介面

| 介面 | 方法 | 通道 |
|---|---|---|
| `IEmailMessage` | `SendAsync(string subject, string body, string[] recipients, MimeEntity[] Attachments = null)` | Email |
| `ISmsMessage` | `SendSmsAsync(string phone, string content)` | 簡訊 |
| `IAndroidMessage` | `SendAsync(string deviceToken, string content, object data = null)` | Android 推播(FCM) |
| `IIosMessage` | `SendAsync(string deviceToken, string content)` | iOS 推播(APNs) |

> 注意各介面**方法名不同**:Email/Android/iOS 是 `SendAsync`、簡訊是 `SendSmsAsync`;參數也各異(附件是 `MimeKit.MimeEntity[]`)。以介面簽名為準。

### 實作:平台內建 + 自訂

平台在 `Hcs.Extensions.Message.*` 各套件提供**正式實作**,以 SMTP 寄信為例——`Hcs.Extensions.Message.Email` 的 `Message` 用 **MailKit** 送信,建構吃一份 `EmailConfig`(帳號 / 寄件者名 / SMTP 主機…):

```csharp
// 正式 SMTP：註冊 EmailConfig + Hcs.Extensions.Message.Email 的實作
builder.Services.AddSingleton<IEmailMessage, Hcs.Extensions.Message.Email.Message>();
```

其他家族同理:`Hcs.Extensions.Message.Android`(FCM)、`.Ios`(APNs)、`.Mitake`(三竹簡訊)。

**要自訂**(接自家服務、或測試時不真的寄)就自己實作介面。sample 的 `ConsoleEmailMessage` 只把信印到 log,示範「注入 `IEmailMessage` 就能寄」而不需真 mail server:

```csharp
public class ConsoleEmailMessage : IEmailMessage
{
    private readonly ILogger<ConsoleEmailMessage> logger;
    public ConsoleEmailMessage(ILogger<ConsoleEmailMessage> logger) => this.logger = logger;

    public Task SendAsync(string subject, string body, string[] recipients, MimeEntity[]? Attachments = null)
    {
        logger.LogInformation("[Email] to={Recipients} subject={Subject}",
            string.Join(", ", recipients ?? Array.Empty<string>()), subject);
        return Task.CompletedTask;
    }
}
```

註冊後,平台任何地方都能注入:

```csharp
builder.Services.AddSingleton<IEmailMessage, ConsoleEmailMessage>();
```

### 呼叫:注入介面即可

發訊也是副作用,同樣擺 `OnAfterTransaction`(commit 成功才寄、且不 hold 交易鎖)。例:員工新增後發歡迎信:

```csharp
opt.ConfigPostApi(x => x.OnAfterTransaction(a => a
    .GetRequestModel<Employee>()
    .Pipe(async (IEmailMessage email, Employee emp) =>
    {
        await email.SendAsync(
            subject: $"Welcome, {emp.Name}!",
            body: $"Employee {emp.Name} ({emp.EmployeeNo}) has been created.",
            recipients: [$"{emp.EmployeeNo}@sample.local"]);
        return (object)emp;
    })));
```

### 範本替換慣例

平台**不內建範本引擎**。實務上把信件 / 簡訊範本存在 DB 一張範本表,內容用 `{{Key}}` 佔位,發送前逐一 `Regex.Replace` 換掉:

```csharp
var body = template.Body;
foreach (var (key, value) in values)
    body = Regex.Replace(body, $"{{{{{key}}}}}", value);   // {{Name}} → 實際值
await email.SendAsync(subject, body, recipients);
```

範本表由各專案自建(欄位、版本、語系怎麼設計是專案的事),平台只提供發送的介面。

---

## Gotchas 一覽

- **Hub 內用 `IUserOrganization` 會 throw**——它依賴 `HttpContext`,Hub 連線期沒有活的 request。改用 `IUserService.Get(userId)`(userId 在 `Context.User.Identity.Name`)。
- **推播 / 發訊擺 `OnAfterTransaction`**——交易內推,rollback 了就通知了不存在的資料;且 async I/O 不該 hold DB 鎖。
- **前端 token 走 `accessTokenFactory`**,不是 HTTP header——SignalR 握手不吃平台 interceptor;token 從 `UserState.token` 拿,別自己碰 `localStorage`。
- **`OnDestroy` 一定要 `hub.stop()`**——換頁不關連線會洩漏。
- **`/hubs` proxy 要開 `ws: true`**——否則 WebSocket 升級失敗,退回長輪詢或直接連不上。
- **`ConfigureEndpoints` 掛 Hub**——用 `opt.ConfigureEndpoints = e => e.MapHub<T>(...)`,不是 obsolete 的直接寫法。
- **發訊介面方法名不一致**——Email/Android/iOS 是 `SendAsync`、簡訊是 `SendSmsAsync`,以各介面簽名為準。

---

## 關聯

- 副作用為何擺交易外、`OnAfterTransaction` 掛點 → [entity-api](entity-api.md)、[flow-api](flow-api.md)
- `.Pipe(...)` 與 DI 注入 → [pipe](pipe.md)
- 多租戶組織與 `IUserOrganization` → [multi-tenant](multi-tenant.md)
- 前端 `UserState`、不碰 `localStorage` 的緣由 → [frontend](../frontend.md)
- 排程發送(孤兒清理、定時任務)的骨架 → [background-services](background-services.md)
