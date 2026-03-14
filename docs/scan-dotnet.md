# .NET Repo Tarama — AŞAMA 2 ve AŞAMA 3

Bu dosya **yalnızca .NET / ASP.NET Core repoları** için geçerlidir.
Dil tespiti: `*.csproj`, `*.sln`, `global.json` dosyaları tespit edildiğinde bu dosyayı uygula.

---

## ⚡ OKUMA STRATEJİSİ — LLM Stabilite Protokolü

> **Bu bölümü önce oku. Token bütçeni ve okuma sıranı belirler.**

### Token Bütçesi (toplam ~45k token)

| Bölüm | Token Hedefi | Öncelik |
|---|---|---|
| Controller'lar | ~8k | KRİTİK |
| Service'ler | ~12k | KRİTİK |
| Entity'ler + DbContext | ~5k | YÜKSEK |
| Repository'ler | ~4k | YÜKSEK |
| DTO'lar + Validator'lar | ~5k | ORTA |
| Security + Middleware | ~4k | YÜKSEK |
| Migration + Config | ~3k | ORTA |
| Test + Teknik Borç | ~4k | DÜŞÜK |

### Dosya Boyutuna Göre Okuma Kuralı

```
< 150 satır  → TAM OKU
150–400 satır → class declaration + constructor + public metod imzaları + attribute'lar
> 400 satır  → SADECE imzalar + attribute'lar + class-level XML doc summary
```

> **DbContext (`OnModelCreating`) genellikle 400+ satırdır.** Tüm `HasIndex()`, `HasOne().WithMany()`, `HasQueryFilter()` çağrılarını al; `HasData()` seed bloklarını atla.

### Okuma Öncelik Sırası

```
1. Controller → Service/Handler → Repository    (execution path)
2. Entity → DbContext → DTO                     (data model)
3. Program.cs (DI + middleware pipeline)        (wiring)
4. SecurityConfig + ExceptionHandler            (cross-cutting)
5. ActionFilter + Middleware                    (side-effects)
6. Migration → Config → Test                   (infrastructure)
```

### LLM Stabilite Kuralları

- Her dosyayı okurken **önce bağlamı yaz**: "Şimdi OrderService okuyorum — OrderController'dan çağrılan iş katmanı"
- 5+ dosya okuduktan sonra kısa bir ara özet yaz: "Şimdiye kadar öğrendiklerim: ..."
- Büyük metodlarda sadece **yan etkileri** çıkar: DB yazımı, event yayını, cache temizleme, harici HTTP çağrısı
- Emin olmadığında `null` yaz, tahmin etme — yanlış bilgi doğru bilgisizlikten tehlikelidir
- `record` gördüğünde immutability işaretle; `class` ile karıştırma

---

## REPO BOYUTU TESPİTİ — Çok Projeli Solution Protokolü

> `*.sln` veya `Directory.Build.props` içinde **5+ proje** varsa bu bölümü uygula.

### Solution Proje Haritası

```
[Solution] ─┬─ MyApp.Api/             ← ASP.NET Core Web API (giriş noktası)
            ├─ MyApp.Application/     ← Use case'ler, servisler, MediatR handler'lar
            ├─ MyApp.Domain/          ← Entity'ler, value object'ler, domain event'ler
            ├─ MyApp.Infrastructure/  ← EF Core, repository impl, HTTP client'lar
            ├─ MyApp.Contracts/       ← DTO'lar, enum'lar, interface'ler (paylaşılan)
            └─ MyApp.Tests/           ← Unit + integration testler
```

**Proje tiplerini sınıfla:**
- **Core (çekirdek):** Domain + Application — diğer tüm projeler referans alır → değişiklikte domino etkisi
- **Infrastructure:** Core'u referans alır, dışarıya açık değil
- **API:** Sadece giriş noktası — asla Domain'e doğrudan erişmemeli
- **Contracts/Shared:** Harici servislerle paylaşılan — versiyonlama kritik

### Hangi Projeleri Derin Tara?

- **Domain:** Her zaman tam tara — entity, value object, domain event
- **Application:** Her zaman tam tara — use case, service, MediatR handler
- **API (Controllers):** Her zaman tam tara
- **Infrastructure:** Repository implementasyonları + DbContext tam tara; migration'lar son 5'i
- **Contracts/Shared:** Tüm DTO + enum tam tara
- **Test:** Tam değil — hangi sınıfların test kapsaması var, tespiti yeterli

### Proje Bağımlılık Matrisi (KB'ye yaz)

```json
"proje_bagimlilik_matrisi": {
  "MyApp.Api": { "referans_ettigi": ["MyApp.Application", "MyApp.Infrastructure"] },
  "MyApp.Application": { "referans_ettigi": ["MyApp.Domain", "MyApp.Contracts"] },
  "MyApp.Infrastructure": { "referans_ettigi": ["MyApp.Application", "MyApp.Domain"] },
  "MyApp.Domain": { "referans_ettigi": [] },
  "core_proje_uyarisi": "Domain değişirse Application + Infrastructure + Api etkilenir"
}
```

---

## AŞAMA 2 — Çekirdek Tarama (~20k token)

**Hedef:** Bir geliştirici "nereye ne yazacak?" sorusunu yanıtlayacak temel detayı topla.

### 2.1 Controller dosyaları (EN KRİTİK)

Her `[ApiController]` / `ControllerBase` türeyen sınıfı oku. Her endpoint için:
- HTTP metodu: `[HttpGet]`, `[HttpPost]`, `[HttpPut]`, `[HttpDelete]`, `[HttpPatch]`
- Route: class-level `[Route("api/[controller]")]` + method-level `[HttpGet("{id}")]` (tam path'i birleştir)
- Parametre kaynakları:
  - `[FromRoute] Guid id` → path parametresi
  - `[FromQuery] string filter` → query string
  - `[FromBody] CreateOrderRequest request` → body (hangi DTO/request sınıfı?)
  - `[FromHeader(Name = "X-Tenant-Id")] string tenantId` → header
  - `[FromServices] IXxxService service` → DI from action method
- Döndürdüğü tip: `Task<ActionResult<UserDto>>`, `IActionResult`, `Ok(result)`, `NotFound()`, `CreatedAtAction(...)`
- Auth: `[Authorize]`, `[Authorize(Roles = "Admin")]`, `[Authorize(Policy = "RequireAdminRole")]`, `[AllowAnonymous]`
- Özel attribute'lar: `[RateLimit]`, `[AuditLog]`, `[RequiresTwoFactor]`, `[ValidateModel]` gibi action filter'lar

#### Kritik Endpoint İşaretleme

Her endpoint için aşağıdaki kritiklik puanını hesapla ve KB'ye yaz:

| Sinyal | +Puan |
|---|---|
| Finansal işlem: ödeme, tahsilat, iade, fatura | +3 |
| State-mutating: DB write + event yayını | +2 |
| Harici servis çağrısı içeriyor | +2 |
| 3+ servis/handler çağırıyor | +2 |
| Transaction scope içinde harici HTTP var | +3 |
| Auth bypass riski: `[AllowAnonymous]` ama data değiştiriyor | +3 |
| `IHostedService` veya `BackgroundService` besliyor | +1 |

```json
"kritik_endpoint_listesi": [
  {
    "endpoint": "POST /api/payments/process",
    "kritiklik_puani": 8,
    "neden": "Finansal işlem + harici banka client + DB yazımı + event yayını",
    "degisiklik_riski": "YÜKSEK — regression test şart"
  }
]
```

### 2.2 Service sınıfları

Her servis sınıfından (genellikle `IXxxService` implemente eden):
- **Constructor injection** → bağımlılık listesi:
  - `IXxxRepository` / `IXxxStore` / `DbContext` → repository/data layer
  - `IXxxService` / `IXxxManager` → servis→servis bağımlılığı
  - `IXxxClient` / `IXxxApiClient` → harici HTTP client
  - `ILogger<XxxService>` / `IMapper` / `IMediator` → altyapı
- **Her public metod için:**
  - İmza: `public async Task<OrderDto> CreateOrderAsync(CreateOrderRequest request, CancellationToken ct)`
  - `async/await` kullanımı
  - Önemli attribute'lar: `[Cacheable]` (özel), transaction scope
  - Kısa iş mantığı özeti (1 cümle) — yan etkiler: async email, event yayını, cache temizleme

> 400 satırdan uzun dosyalarda: class + constructor + public method imzaları + attribute'lar yeterli.

**MediatR kullanan projeler:**
- Her `IRequestHandler<TRequest, TResponse>` → hangi command/query'yi handle ediyor?
- Pipeline behavior'lar: `IPipelineBehavior<,>` → validation, logging, transaction pipeline sırasını belirle
- `INotificationHandler<TEvent>` → domain event handler'lar

### 2.3 Entity sınıfları

EF Core entity'leri:
- **Data Annotations kullanan:**
  - `[Table("tablo_adi")]` → DB tablo adı
  - `[Key]` → primary key
  - `[Column("kolon_adi")]`, `[Column(TypeName = "decimal(18,2)")]`
  - `[Required]`, `[MaxLength(255)]`, `[MinLength(1)]`
  - `[Index(nameof(Email), IsUnique = true)]` (sınıf seviyesi attribute)
  - `[ForeignKey("UserId")]`
- **Fluent API kullanan (OnModelCreating):**
  - `modelBuilder.Entity<User>().ToTable("users")` → tablo adı
  - `.HasKey(u => u.Id)`, `.Property(u => u.Email).IsRequired().HasMaxLength(255)`
  - `.HasIndex(u => u.Email).IsUnique()` → index tanımları
  - `.HasOne(u => u.Role).WithMany(r => r.Users).HasForeignKey(u => u.RoleId)`
  - `.HasQueryFilter(u => !u.IsDeleted)` → global soft delete filter
- Navigasyon property'ler: `public ICollection<Order> Orders { get; set; }`
- Soft delete: `IsDeleted`, `DeletedAt` property var mı?
- Audit fields: `CreatedAt`, `UpdatedAt`, `CreatedBy` — `IEntity`, `IAuditable` gibi base class'tan mı geliyor?
- Value object var mı? (`IEquatable<T>` implemente eden record/class)
- `[Owned]` attribute'lu owned entity'ler

### 2.4 Repository / Data Access

`IXxxRepository` interface'leri ve implementasyonlarını oku:
- **Generic Repository pattern:** `IRepository<T>` → `GetByIdAsync`, `AddAsync`, `UpdateAsync`, `DeleteAsync`, `FindAsync`
- **Specific Repository metodları:** `FindByEmailAsync`, `GetOrdersByCustomerIdAsync`
- **EF Core LINQ sorguları:**
  - `_context.Users.AsNoTracking().Where(...).Select(...).ToListAsync()`
  - `.Include(u => u.Orders).ThenInclude(o => o.Items)` → eager loading
  - `.AsQueryable()` + Specification pattern
- **Raw SQL:** `FromSqlRaw(...)`, `ExecuteSqlRawAsync(...)`
- **Dapper kullanan projeler:** `connection.QueryAsync<T>(sql, params)` → tam SQL sorgusu
- **Stored procedure çağrıları:** `_context.Database.ExecuteSqlInterpolatedAsync($"EXEC sp_name {param}")`
- **Transaction yönetimi:** `IDbContextTransaction`, `TransactionScope`, `BeginTransactionAsync`
- **Unit of Work:** `IUnitOfWork.SaveChangesAsync()` → hangi servisler kullanıyor?

### 2.5 Bağımlılık Grafını Çıkar (KRİTİK ADIM)

Her servis sınıfının constructor'ını oku:
- `IXxxRepository` / `IXxxStore` / `DbContext` → `bagimliliklar.repository`
- `IXxxService` / `IXxxManager` / `IXxxFacade` → `bagimliliklar.servis`
- `IXxxClient` / `IHttpClientFactory` → `bagimliliklar.harici_client`
- `ILogger<T>` / `IMapper` / `IMediator` / `IValidator<T>` → `bagimliliklar.diger`

`bu_servisi_kullananlar`: Tüm constructor'ları tara, her servisi hangi sınıfların inject ettiğini bul.

`kritik_merkez_servisler`: 3+ farklı noktadan inject edilen servis = merkez servis. İşaretle.

#### Bağımlılık Grafı Görselleştirme

KB'ye JSON adjacency list'e ek olarak şu metin görselini de yaz — geliştiriciler JSON'u parse etmez ama bunu okur:

```
BAĞIMLILIK AĞACI:
OrderService ──> [IOrderRepository, IUserService, IPaymentService, IPublishEndpoint]
PaymentService ──> [IPaymentRepository, IBankApiClient, IAuditService]
UserService ──> [IUserRepository, IMemoryCache]                   ← HUB (4 servis kullanıyor)

ETKİ ANALİZİ (değişirse etkilenenler):
IUserService   → OrderService, PaymentService, AuthService, ReportService  [4 etki]
IPaymentService → OrderService, SubscriptionService                        [2 etki]
IBankApiClient  → PaymentService                                           [1 etki]

DÖNGÜSEL BAĞIMLILIK KONTROLÜ: ✅ Yok | ⚠️ [ServiceA ↔ ServiceB] döngü var!
CAPTIVE DEPENDENCY KONTROLÜ: ✅ Yok | ⚠️ Singleton içinde Scoped inject: [XxxService → DbContext]
```

### 2.6 DI Registration ve Startup

`Program.cs` (minimal API / .NET 6+) veya `Startup.cs` (eski):

**DI Registration (ServiceCollection extensions):**
```csharp
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.AddSingleton<ICacheService, RedisCacheService>();
builder.Services.AddTransient<IEmailSender, SmtpEmailSender>();
```
- Hangi interface → hangi implementation?
- Lifetime: `AddScoped` (request), `AddSingleton` (app), `AddTransient` (her resolve)
- `AddDbContext<AppDbContext>` → connection string key'i
- `AddStackExchangeRedisCache` / `AddMemoryCache`
- `AddHttpClient<IXxxClient, XxxClient>` → Typed HttpClient

**Middleware Pipeline (sıralama kritik!):**
```csharp
app.UseExceptionHandler("/error");
app.UseHttpsRedirection();
app.UseAuthentication();   // Auth middleware sırası önemli!
app.UseAuthorization();
app.UseRateLimiter();
app.MapControllers();
```
- Custom middleware'ler: `app.UseMiddleware<TenantMiddleware>()` → ne yapıyor?
- Middleware sırasını tam olarak belirt

**MediatR kullanan projeler:**
- `builder.Services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(...))` → hangi assembly?
- Pipeline behavior'lar: `IPipelineBehavior<,>` → validation, logging, transaction pipeline

---

## AŞAMA 3 — Derinlemesine Ek Tarama (~25k token)

**Hedef:** Geliştirici bağlamını tam olarak kurmak.

### 3.1 DTO / Request / Response Şema Kataloğu

Önceliklendirme:
1. Controller'larda `[FromBody]` olarak kullanılan request sınıfları
2. Controller'lardan dönen response/DTO sınıfları
3. Queue consumer/producer'lardaki mesaj sınıfları
4. HttpClient metodlarındaki parameter/response tipleri

Her DTO/Request/Response için:
- Tüm property isimleri ve .NET tipleri: `string`, `Guid`, `long`, `decimal`, `DateTime`, `DateOnly`
- Validation:
  - Data Annotations: `[Required]`, `[MaxLength(255)]`, `[EmailAddress]`, `[RegularExpression("...")]`, `[Range(1, 100)]`
  - FluentValidation: `RuleFor(x => x.Email).NotEmpty().EmailAddress()` → validator sınıfını oku
- JSON:
  - `[JsonPropertyName("snake_case_name")]`
  - `[JsonIgnore]`
  - `[JsonConverter(typeof(DateOnlyConverter))]`
- `record` mu yoksa `class` mı? (immutability)
- `IValidator<T>` — MediatR pipeline'da otomatik validate ediliyor mu?

### 3.2 Enum Kataloğu

Tüm enum dosyalarını oku:
- Tüm değerler: `Active = 1, Passive = 2, Suspended = 3`
- `[JsonConverter(typeof(JsonStringEnumConverter))]` → string mi int mi serialize ediyor?
- EF Core'da: `HasConversion<string>()` mi yoksa sayısal mı?
- `[EnumMember(Value = "active")]` → özel serialization değeri
- Kullanım yerleri: hangi entity property, hangi DTO, hangi switch-case
- Domain-critical enum'lar: `PaymentStatus`, `OrderStatus`, `TransactionType`, `UserRole`

### 3.3 Repository Custom Query Kataloğu

Her repository'nin tüm metodlarını tara:
- **EF Core LINQ:** Kompleks LINQ sorgularını tam olarak çıkar:
  ```csharp
  return await _context.Orders
      .AsNoTracking()
      .Where(o => o.CustomerId == customerId && !o.IsDeleted)
      .Include(o => o.Items)
      .OrderByDescending(o => o.CreatedAt)
      .ToListAsync(ct);
  ```
- **Raw SQL:** `FromSqlRaw` / `FromSqlInterpolated` → tam SQL metnini al
- **Dapper:** `QueryAsync<T>(sql, params)` → tam SQL ve parametre tiplerini al
- **EF Core ExecuteUpdate / ExecuteDelete** (.NET 7+): toplu güncelleme/silme
- **Pagination:** `Skip(page * pageSize).Take(pageSize)` pattern
- **Specification pattern:** `ISpecification<T>` kullanılıyorsa örneklerini belirt

### 3.4 Güvenlik Haritası

`Program.cs` / `Startup.cs` güvenlik konfigürasyonunu tam oku:

**Authentication:**
```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options => { ... });
```
- JWT: `ValidateIssuer`, `ValidateAudience`, `ValidateLifetime`, `IssuerSigningKey`
- Cookie auth: `.AddCookie(...)` → session yapısı
- API Key: custom `IAuthenticationHandler`
- OAuth2 / OpenID Connect: `.AddOpenIdConnect(...)`

**Authorization:**
```csharp
builder.Services.AddAuthorization(options => {
    options.AddPolicy("RequireAdminRole", policy => policy.RequireRole("Admin"));
    options.AddPolicy("MinAge18", policy => policy.Requirements.Add(new MinAgeRequirement(18)));
});
```
- Policy'ler ve gereksinimleri
- `IAuthorizationHandler` implementasyonları (custom requirement handler)
- Resource-based authorization var mı?

**Endpoint güvenliği:**
- `[AllowAnonymous]` → authentication gerekmeyen endpoint'ler tam listesi
- `[Authorize(Roles = "Admin")]` → rol gereksinimleri
- `[Authorize(Policy = "...")]` → policy gereksinimleri
- MinimalAPI: `.RequireAuthorization()`, `.AllowAnonymous()`

**Custom middleware güvenliği:**
- API Key middleware, Tenant middleware, Rate Limit middleware
- `app.UseAuthentication()` öncesinde çalışan middleware var mı? (sıra kritik!)

### 3.5 Exception Handling

Global exception handler'ı oku:
- `app.UseExceptionHandler(...)` veya `app.UseMiddleware<GlobalExceptionMiddleware>()`
- `IExceptionHandler` (.NET 8+) implementasyonu
- `ProblemDetails` formatı kullanılıyor mu? RFC 7807 uyumu
- Custom exception hiyerarşisi:
  - `DomainException` / `BusinessException` → 422 Unprocessable Entity
  - `NotFoundException` → 404 Not Found
  - `ValidationException` / `BadRequestException` → 400 Bad Request
  - `UnauthorizedException` → 401
  - `ForbiddenException` → 403

**Polly ile resilience:**
```csharp
builder.Services.AddHttpClient<IPartnerApiClient>()
    .AddPolicyHandler(GetRetryPolicy())
    .AddPolicyHandler(GetCircuitBreakerPolicy());
```
- Retry policy: kaç deneme, hangi exception'larda, backoff stratejisi
- Circuit breaker: kaç başarısızlıktan sonra açılıyor, recovery timeout
- Timeout policy: `Policy.TimeoutAsync<HttpResponseMessage>(TimeSpan.FromSeconds(30))`
- Fallback policy var mı?

**Message broker DLQ:**
- Hangi kuyruklar dead letter queue'ya düşüyor?
- DLQ'daki mesajlar için reprocessing mekanizması var mı?

### 3.6 Action Filter / Middleware Detayları

Custom attribute ve filter'ları oku:
- `IActionFilter` / `IAsyncActionFilter` → `OnActionExecuting`, `OnActionExecuted`
- `IResourceFilter` → cache'leme amacıyla
- `IExceptionFilter` → spesifik exception override
- Custom middleware (`IMiddleware` veya convention-based): sıralama + ne iş yapıyor
- Hangi attribute tetikler? Model validation, audit log, rate limit, idempotency key
- Yan etkiler: Redis'e yazıyor mu? Response header set ediyor mu?

### 3.7 Mesaj / Event Şemaları

**Azure Service Bus:**
- `ServiceBusProcessor` → hangi queue/topic+subscription
- `ServiceBusSender` → hangi queue/topic'e gönderilen
- Message DTO: serialize format (JSON), property isimleri
- Dead-lettering: `DeadLetterReasonHeaderName`, `EnableDeadLetteringOnMessageExpiration`

**RabbitMQ (MassTransit / EasyNetQ / RawRabbit):**
- MassTransit: `cfg.ReceiveEndpoint("queue-name", e => e.Consumer<XxxConsumer>(context))`
- Consumer: `IConsumer<TMessage>` → `Consume(ConsumeContext<TMessage> context)`
- Producer: `IPublishEndpoint.Publish<T>()` veya `ISendEndpoint.Send<T>()`

**Azure Service Bus / RabbitMQ ortak:**
- Mesaj DTO sınıfı → tam property listesi
- Event zinciri: "X mesajı gelince Y olur, sonra Z kuyruğuna gider"
- `IHostedService` / `BackgroundService` ile çalışan consumer var mı?

### 3.8 Migration / Schema Durumu

**EF Core Migrations:**
- `dotnet ef migrations list` çıktısına eşdeğer: `Migrations/` klasöründeki dosyaları listele
- Son 5 migration: dosya adı + ne değiştirildi (`Up()` metoduna bak)
- `dotnet ef database update` → migration uygulanıyor mu CI/CD'de mi?
- `HasData(...)` seed data var mı?
- `migrationBuilder.Sql("CREATE INDEX ...")` → custom index'ler

**SQL Script kullanan projeler:**
- `Scripts/` veya `Database/` klasöründeki `.sql` dosyaları
- Versiyonlama convention'ı nedir?
- CI/CD'de otomatik mi çalışıyor?

**Ortak:**
- Connection string key: `ConnectionStrings:Default` / `ConnectionStrings:ReadReplica`
- Son migration'larda eklenen kritik index'ler ve constraint'ler
- `AppDbContext.OnModelCreating` → Fluent API ile tanımlanan index'ler

### 3.9 Test Durumu

`*.Tests` / `*.UnitTests` / `*.IntegrationTests` projelerini tara:
- **Unit test framework:** xUnit, NUnit, MSTest
- **Mocking:** Moq (`Mock<IXxxService>()`), NSubstitute, FakeItEasy
- **Integration test:** `WebApplicationFactory<Program>` → in-process test server
- **TestContainers:** gerçek PostgreSQL/Redis/RabbitMQ container'ları
- **In-memory database:** `UseInMemoryDatabase("test")` → dikkat: EF Core in-memory gerçek SQL desteklemez
- Test naming convention: `MethodName_StateUnderTest_ExpectedBehavior` veya `Should_DoX_When_Y`
- `[Theory] [InlineData(...)]` → data-driven testler
- `Bogus` / `AutoFixture` → fake data generation
- Hangi prodüksiyon sınıflarının test'i var? Hangileri eksik?

### 3.10 HttpClient Kataloğu

Typed HttpClient'ları oku:
```csharp
builder.Services.AddHttpClient<IPartnerApiClient, PartnerApiClient>(client => {
    client.BaseAddress = new Uri(config["Clients:PartnerApi:BaseUrl"]);
    client.Timeout = TimeSpan.FromSeconds(30);
});
```
- Base URL config key'i
- Timeout ayarları
- DelegatingHandler'lar: authentication header ekleme, logging, retry
- Her metod: HTTP metodu + path + parametre tipi + dönüş tipi
- `GetAsync`, `PostAsync`, `PutAsync` + JSON serialization/deserialization
- `HttpClientFactory` kullanımı (IHttpClientFactory) vs Typed client

### 3.11 Kritik Konfigürasyonlar

`appsettings.json` + `appsettings.Production.json` + `appsettings.{Environment}.json` tam oku:

```json
{
  "ConnectionStrings": { "Default": "..." },
  "Redis": { "Configuration": "..." },
  "Jwt": { "SecretKey": "...", "Issuer": "...", "ExpiryMinutes": 60 },
  "Clients": { "PartnerApi": { "BaseUrl": "..." } },
  "FeatureFlags": { "NewPaymentFlow": false }
}
```

- `IOptions<T>` pattern ile bağlanan config section'ları: hangi sınıf, hangi section?
- Environment variable override: `ASPNETCORE_ENVIRONMENT`, `DOTNET_ENVIRONMENT`
- User Secrets: `dotnet user-secrets` kullanımı (development ortamı)
- Azure Key Vault / AWS Secrets Manager entegrasyonu var mı?
- Feature flag'ler: `Microsoft.FeatureManagement` library var mı?
- Scheduled job'lar: `IHostedService` / `BackgroundService` + Quartz.NET cron expression'ları

### 3.12 Teknik Borç Haritası

Kaynak kodda şunları ara:
- `// TODO:`, `// FIXME:`, `// HACK:`, `// WORKAROUND:` comment'leri → konum + içerik
- `[Obsolete("Kullan: XxxAsync")]` attribute → alternatif belirtilmiş mi?
- Magic number'lar: hardcoded sayısal değerler (timeout, limit, retry count)
- Hardcoded URL: `"http://internal-service/api/..."` string'leri
- `#pragma warning disable` → kapatılan uyarılar (neden?)
- `Thread.Sleep(...)` → async olmayan bekleme (performance riski)
- `.Result` / `.GetAwaiter().GetResult()` → async deadlock riski

---

### 3.13 Endpoint → Execution Flow (EN BÜYÜK KAZANÇ)

> **Bu bölüm bir geliştirici için en değerli çıktıdır.** Kritiklik puanı 5+ olan her endpoint için tam execution path'i çıkar.

#### Nasıl Çıkarılır

Controller metodunu oku → doğrudan servis mi çağırıyor yoksa MediatR `Send()` mi? → ilgili handler/servis metodunu bul → transaction scope'unu belirle → repository çağrılarını listele → harici HTTP çağrılarını bul → event/mesaj yayınlarını işaretle.

#### Format (Direkt Servis)

```
POST /api/orders/create                                    [kritiklik: 7]
│
├─ OrderController.CreateAsync([FromBody] CreateOrderRequest)
│   └─ [Authorize(Policy = "OrderCreate")]
│
├─ IOrderService.CreateOrderAsync(request, ct)
│   ├─ [1] IUserService.ValidateUserAsync(request.UserId)   [servis çağrısı]
│   │       └─ _context.Users.FindAsync(id)                 [DB okuma — EF Core]
│   ├─ [2] IInventoryClient.CheckStockAsync(request.Items)  [HTTP POST /inventory/check, timeout:5s]
│   │       └─ ⚠️  IDbContextTransaction açıkken — bağlantı havuzu meşgul kalır
│   ├─ [3] _context.Orders.AddAsync(order)                  [DB yazımı — EF Core]
│   ├─ [4] _context.SaveChangesAsync(ct)                    [commit]
│   ├─ [5] _publishEndpoint.Publish<OrderCreatedEvent>(evt) [MassTransit — fire & forget]
│   └─ [6] return OrderResponseDto
│
└─ YAN ETKİLER (async):
    ├─ OrderCreatedConsumer → IEmailService.SendConfirmationAsync()
    └─ OrderCreatedConsumer → IAnalyticsService.TrackOrderAsync()

UYARILAR:
⚠️  IDbContextTransaction açıkken harici HTTP çağrısı (IInventoryClient) — timeout riski
⚠️  Event yayını SaveChanges öncesinde yapılırsa DB rollback durumunda tutarsızlık
```

#### Format (MediatR CQRS)

```
POST /api/orders/create                                    [kritiklik: 7]
│
├─ OrderController.CreateAsync([FromBody] CreateOrderRequest)
│   └─ _mediator.Send(new CreateOrderCommand(request))
│
├─ MediatR Pipeline:
│   ├─ [P1] ValidationBehavior → FluentValidation: CreateOrderCommandValidator
│   ├─ [P2] LoggingBehavior → Serilog structured log
│   ├─ [P3] TransactionBehavior → IDbContextTransaction.BeginAsync()
│   └─ [P4] CreateOrderCommandHandler.Handle(command, ct)
│               ├─ [1] IUserRepository.GetByIdAsync(command.UserId)   [DB okuma]
│               ├─ [2] IBankApiClient.AuthorizeAsync(command.Amount)   [HTTP POST, timeout:10s]
│               │       └─ ⚠️  Transaction açıkken harici HTTP
│               ├─ [3] IOrderRepository.AddAsync(order)               [DB yazımı]
│               └─ [4] _publisher.Publish(new OrderCreatedEvent())    [domain event]
│
└─ Domain Event Handler: OrderCreatedEventHandler
    └─ IEmailService.SendConfirmationAsync()

UYARILAR:
⚠️  Polly retry + idempotency: POST /bank/authorize banka tarafında idempotent mi?
⚠️  TransactionBehavior + ValidationBehavior sırası: validation transaction açılmadan önce çalışmalı
```

**KB'de `execution_flows` alanına yaz:**
```json
"execution_flows": [
  {
    "endpoint": "POST /api/orders/create",
    "kritiklik_puani": 7,
    "pattern": "MediatR CQRS",
    "pipeline": ["ValidationBehavior", "LoggingBehavior", "TransactionBehavior", "CreateOrderCommandHandler"],
    "adimlar": [
      { "sira": 1, "tip": "DB_READ", "detay": "IUserRepository.GetByIdAsync()" },
      { "sira": 2, "tip": "HTTP_CALL", "detay": "IBankApiClient.AuthorizeAsync() — ⚠️ transaction açıkken" },
      { "sira": 3, "tip": "DB_WRITE", "detay": "IOrderRepository.AddAsync()" },
      { "sira": 4, "tip": "EVENT_PUBLISH", "detay": "OrderCreatedEvent → async email" }
    ],
    "uyarilar": ["Transaction içinde harici HTTP", "Polly retry idempotency riski"]
  }
]
```

---

### 3.14 Performance Risk Detection

> Her servis ve repository metodunu okurken şu anti-pattern'leri işaretle.

#### N+1 Sorgu Riski (EF Core)

```csharp
// ⚠️ RİSK: Her order için ayrı SQL çalışır
var orders = await _context.Orders.ToListAsync();
foreach (var order in orders)
    Console.WriteLine(order.Items.Count); // LAZY navigasyon — ayrı SELECT!

// ✅ ÇÖZÜM: Eager loading ile tek sorgu
var orders = await _context.Orders
    .Include(o => o.Items)
    .ThenInclude(i => i.Product)
    .ToListAsync();
```

Tespit: `.ToListAsync()` sonrasında navigasyon property'ye döngü içinde erişim.

#### `AsNoTracking` Eksikliği

```csharp
// ⚠️ RİSK: Read-only sorgu ama change tracking açık → gereksiz memory ve CPU
var users = await _context.Users.Where(u => u.IsActive).ToListAsync();

// ✅ ÇÖZÜM: Read-only sorgularda her zaman
var users = await _context.Users.AsNoTracking().Where(u => u.IsActive).ToListAsync();
```

Tespit: `GetAsync`, `ListAsync`, `FindAsync` gibi sorgu metodlarında `.AsNoTracking()` eksikliği.

#### `Select()` Projeksiyon Eksikliği

```csharp
// ⚠️ RİSK: Tüm kolonlar çekiliyor, sadece 2 alan kullanılıyor
var orders = await _context.Orders.ToListAsync();
var result = orders.Select(o => new { o.Id, o.Status });

// ✅ ÇÖZÜM: DB'den sadece gerekeni al
var result = await _context.Orders
    .Select(o => new { o.Id, o.Status })
    .ToListAsync();
```

#### Sayfalama Eksikliği

```csharp
// ⚠️ RİSK: Tüm tabloyu belleğe yükler
public async Task<List<OrderDto>> GetAllOrdersAsync()
    => await _context.Orders.ToListAsync(); // Pageable yok!

// ✅ ÇÖZÜM:
public async Task<PagedResult<OrderDto>> GetOrdersAsync(int page, int pageSize, CancellationToken ct)
    => await _context.Orders.Skip(page * pageSize).Take(pageSize).ToListAsync(ct);
```

Tespit: `List<T>` döndüren repository metodları + `ToListAsync()` çağrısında `Skip/Take` yoksa.

#### IDbContextTransaction İçinde Harici HTTP

```csharp
// ⚠️ RİSK: DB bağlantısı açık kalırken HTTP timeout bekleniyor
await using var tx = await _context.Database.BeginTransactionAsync();
var bankResult = await _bankClient.ChargeAsync(amount); // timeout: 30s — bağlantı tutuluyor!
await _context.SaveChangesAsync();
await tx.CommitAsync();
```

Tespit: `BeginTransactionAsync()` scope'u içinde `HttpClient` veya Typed Client çağrısı.

#### Captive Dependency (Singleton → Scoped)

```csharp
// ⚠️ KRİTİK: Singleton servis içinde Scoped DbContext inject
public class CacheWarmupService // AddSingleton ile kayıtlı
{
    private readonly AppDbContext _context; // AddScoped — her request yeni instance!
    // Sonuç: İlk request'in DbContext'i tüm uygulama ömrü boyunca tutulur → stale data
}
```

Tespit: `AddSingleton` ile kayıtlı sınıfların constructor'ında `IXxxRepository`, `DbContext` veya başka Scoped servis.

#### `.Result` / `.GetAwaiter().GetResult()` Deadlock Riski

```csharp
// ⚠️ RİSK: ASP.NET Core'da SynchronizationContext + blocking wait → deadlock
var user = _userService.GetByIdAsync(id).Result; // Deadlock potansiyeli!

// ✅ ÇÖZÜM: Her zaman await kullan
var user = await _userService.GetByIdAsync(id);
```

**KB'de `performance_riskleri` alanına yaz:**
```json
"performance_riskleri": [
  {
    "tip": "N+1_SORGU",
    "dosya": "OrderService.cs:87",
    "aciklama": "Order.Items navigasyon property döngü içinde lazy erişim",
    "risk": "yuksek",
    "cozum": ".Include(o => o.Items) veya .Select() projeksiyon ekle"
  },
  {
    "tip": "TRANSACTION_ICINDE_HTTP",
    "dosya": "PaymentService.cs:134",
    "aciklama": "IBankApiClient.ChargeAsync() BeginTransactionAsync scope içinde, timeout: 30s",
    "risk": "kritik",
    "cozum": "HTTP çağrısını transaction dışına al veya IHostedService ile ayır"
  },
  {
    "tip": "CAPTIVE_DEPENDENCY",
    "dosya": "Program.cs:45 → CacheWarmupService",
    "aciklama": "Singleton servis içinde Scoped DbContext inject edilmiş",
    "risk": "kritik",
    "cozum": "IServiceScopeFactory kullan veya lifetime'ı eşleştir"
  }
]
```

---

### 3.15 Security Risk Scan

> 3.4'teki güvenlik haritasına ek olarak kod seviyesinde güvenlik açıklarını tara.

#### SQL Injection (`FromSqlRaw` String Interpolation)

```csharp
// ⚠️ KRİTİK: String concatenation — SQL injection açığı
var users = _context.Users.FromSqlRaw("SELECT * FROM Users WHERE Name = '" + name + "'");

// ✅ GÜVENLİ: Parameterized
var users = _context.Users.FromSqlRaw("SELECT * FROM Users WHERE Name = {0}", name);
// VEYA interpolated (EF Core otomatik parametrize eder):
var users = _context.Users.FromSqlInterpolated($"SELECT * FROM Users WHERE Name = {name}");
```

Tespit: `FromSqlRaw(...)` içinde `+` operatörü veya string `$"..."` format olmadan interpolation.

#### Dapper'da Parameterized Query Eksikliği

```csharp
// ⚠️ KRİTİK: SQL injection
var result = await conn.QueryAsync<User>($"SELECT * FROM Users WHERE email = '{email}'");

// ✅ GÜVENLİ:
var result = await conn.QueryAsync<User>("SELECT * FROM Users WHERE email = @Email", new { Email = email });
```

#### Eksik Model Validation

```csharp
// ⚠️ RİSK: [ApiController] attribute yoksa ModelState otomatik validate edilmez
[HttpPost]
public async Task<IActionResult> Create([FromBody] CreateUserRequest request)
// [ApiController] yoksa:  if (!ModelState.IsValid) return BadRequest(ModelState); satırı eksik!

// FluentValidation pipeline'da varsa sorun yok:
// builder.Services.AddFluentValidationAutoValidation()
```

Tespit: `[ApiController]` attribute'u yoksa + FluentValidation auto-validation kapalıysa + ManualState kontrolü yoksa.

#### CORS Wildcard

```csharp
// ⚠️ RİSK: Tüm origin'lere izin
builder.Services.AddCors(options => {
    options.AddPolicy("AllowAll", policy => policy.AllowAnyOrigin()); // Production'da tehlikeli!
});

// ⚠️ RİSK: Controller seviyesinde:
[EnableCors] // Policy belirtmeden — varsayılan policy ne?
```

#### Mass Assignment (Entity Doğrudan Bind)

```csharp
// ⚠️ RİSK: Entity doğrudan request'ten bind ediliyor
[HttpPost]
public async Task<IActionResult> Create([FromBody] User user) // Entity, DTO değil!
{
    await _context.Users.AddAsync(user); // Id, IsAdmin gibi alanlar dışarıdan set edilebilir!
}
```

Tespit: Controller parametresinde entity tipi (`DbSet`'te kayıtlı sınıf) doğrudan `[FromBody]` ile kullanım.

#### Hardcoded Secret

```csharp
// ⚠️ KRİTİK: JWT secret kod içinde
var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes("my-super-secret-key-123")); // HARDCODED!

// appsettings.json içinde plaintext:
"Jwt": { "SecretKey": "my-super-secret-key-123" } // Git history'de kalır!
```

#### `AllowAnonymous` ile State-Mutating Endpoint

```csharp
// ⚠️ RİSK: Auth gerektirmeyen POST/PUT/DELETE endpoint
[AllowAnonymous]
[HttpPost("reset-password")] // Kimlik doğrulama olmadan şifre sıfırlama
public async Task<IActionResult> ResetPassword([FromBody] ResetPasswordRequest request) { ... }
```

#### Actuator / Health Endpoint Açığı

```csharp
// ⚠️ Kontrol et: /health, /metrics, /swagger Production'da açık mı?
app.MapHealthChecks("/health"); // [Authorize] yok!
app.UseSwagger(); // if (app.Environment.IsDevelopment()) bloğu dışında mı?
```

**KB'de `guvenlik_riskleri` alanına yaz:**
```json
"guvenlik_riskleri": [
  {
    "tip": "SQL_INJECTION",
    "dosya": "UserRepository.cs:45",
    "aciklama": "FromSqlRaw içinde string concatenation",
    "risk": "kritik",
    "cve_kategori": "OWASP A03:2021"
  },
  {
    "tip": "HARDCODED_SECRET",
    "dosya": "appsettings.json:12",
    "aciklama": "JWT SecretKey plaintext olarak config dosyasında",
    "risk": "kritik",
    "cozum": "Azure Key Vault veya environment variable kullan"
  },
  {
    "tip": "CORS_WILDCARD",
    "dosya": "Program.cs:67",
    "aciklama": "AllowAnyOrigin() — production'da tüm origin'lere izin",
    "risk": "orta"
  }
]
```

---

### 3.16 Config Drift Detection

> Farklı ortam konfigürasyonları arasındaki farkları bul. "Prod'da neden çalışmıyor?" sorularının %60'ının cevabı burada.

#### Ortam Dosyalarını Karşılaştır

Şu dosyaları oku ve karşılaştır:
- `appsettings.json` (default)
- `appsettings.Development.json`
- `appsettings.Staging.json`
- `appsettings.Production.json`

#### Drift Kategorileri

**1. Production'da var, default'ta yok (gizli prod davranışı):**
```json
// appsettings.Production.json'da var, appsettings.json'da yok:
"Logging": { "LogLevel": { "Default": "Warning" } }
// → Dev'de Debug log, prod'da Warning — log kaybı yaratır, dikkat!

"FeatureFlags": { "NewCheckout": true }
// → Bu flag sadece prod'da aktif mi? Dev'de test edilmemiş feature production'da açık!
```

**2. Default'ta var, Production'da override edilmemiş:**
```json
// appsettings.json'da:
"Jwt": { "ExpiryMinutes": 1440 } // 24 saat — production için güvenli mi?
// appsettings.Production.json'da bu key yok → production da 1440 dakika kullanıyor
```

**3. Kritik değer farkları:**
```json
// appsettings.Development.json: "MaxPoolSize": 5
// appsettings.Production.json:  "MaxPoolSize": 100  // 20x fark — kapasite planlaması kritik
```

**4. Prod'da eksik zorunlu key:**
```json
// appsettings.json'da placeholder:
"Clients": { "PartnerApi": { "BaseUrl": "http://localhost:8081" } }
// appsettings.Production.json'da bu section yok → prod'da localhost'a bağlanıyor olabilir!
```

**KB'de `config_drift` alanına yaz:**
```json
"config_drift": {
  "ortamlar_karsilastirildi": ["default", "Development", "Production"],
  "sadece_production_da_var": [
    { "key": "FeatureFlags:NewCheckout", "deger": "true", "risk": "Dev'de test edilmemiş feature prod'da aktif" }
  ],
  "deger_farklari": [
    { "key": "ConnectionStrings:Default.MaxPoolSize", "development": "5", "production": "100", "risk": "dusuk" },
    { "key": "Jwt:ExpiryMinutes", "development": "1440", "production": "1440 (aynı—güvenli mi?)", "risk": "orta" }
  ],
  "production_da_eksik_olabilecekler": [
    "Clients:PartnerApi:BaseUrl — prod override'ı yok, localhost kullanılıyor olabilir"
  ],
  "environment_variable_bagimli": [
    { "key": "ConnectionStrings:Default", "env_var": "ASPNETCORE_ConnectionStrings__Default" }
  ]
}
```

---

## REPO KOMPLEKSİTE SKORU

> KB'ye meta bilgi olarak ekle. Mimari risk ve bakım maliyetini tek sayıda özetler.

### Hesaplama

| Faktör | Metrik | Puan |
|---|---|---|
| Solution proje sayısı | 1-3: 0p / 4-8: 2p / 9+: 4p | |
| Servis/handler sayısı | 1-10: 0p / 11-25: 2p / 26+: 4p | |
| Servis-servis bağımlılık kenar sayısı | per 5 kenar: +1p | |
| Kritik merkez servis sayısı | per merkez: +2p | |
| Captive dependency var mı? | Evet: +3p | |
| Harici HTTP entegrasyonu | per client: +1p | |
| Message consumer + producer | per 5 adet: +1p | |
| Döngüsel bağımlılık var mı? | Evet: +5p | |
| Test coverage boşluğu | %0-30 kapsama: +3p / %30-60: +1p | |
| Teknik borç (yüksek risk) | per FIXME: +0.5p | |

**Skor yorumu:**
- 0-4: Düşük — yerleşik geliştirici olmadan değişiklik yapılabilir
- 5-8: Orta — etki analizi gerekli, dikkatli davran
- 9-12: Yüksek — her değişiklik için geniş test kapsamı şart
- 13+: Kritik — mimari gözden geçirme önerilir

**KB'de `kompleksite_skoru` alanına yaz:**
```json
"kompleksite_skoru": {
  "toplam": 7.5,
  "seviye": "ORTA",
  "kirilim": {
    "proje_sayisi": 2,
    "servis_sayisi": 2,
    "bagimlilik_kenarlari": 2,
    "kritik_merkez_servisler": 2,
    "harici_entegrasyon": 1
  },
  "oneri": "IUserService ve IPaymentService değişikliklerinde geniş regresyon testi yap"
}
```

---

## KB BOYUT KORUMASI

> JSON çok büyük olursa agent'ın ileride okuma maliyeti artar. Şu kuralları uygula.

### Boyut Hedefleri

| Durum | Hedef |
|---|---|
| Küçük repo (1-3 servis/handler) | < 200KB JSON |
| Orta repo (4-10 servis) | < 400KB JSON |
| Büyük repo (11+ servis) | < 600KB JSON; solution başına ayrı JSON |

### Sıkıştırma Stratejisi (büyük repolar için)

**DTO'lar:** 20+ property olan DTO'larda sadece ilk 10'u yaz, geri kalanı için `"devami_var": true, "dosya": "path/to/Dto.cs"` ekle.

**Servis metodları:** 15+ metod olan servislerde sadece attribute'lı metodları tam yaz; attribute'suz olanları yalnızca imzayla listele.

**EF Core LINQ sorguları:** 5 satırdan uzun sorguları kes, `"tam_sorgu_icin": "dosya:satir"` ekle.

**Migration listesi:** Son 5 migration tam; öncekiler için sadece toplam sayı: `"toplam_migration": 47, "gosterilen": "son 5"`.

**Teknik borç:** Düşük risk öğeleri sadece sayısını yaz: `"dusuk_risk_toplam": 9, "ornekler": [ilk 3 tanesi]`

### Çok Projeli Solution'da Ayrı JSON Stratejisi

```
kb/
├── _index.json              ← proje listesi + kısa açıklamalar + bağımlılık matrisi
├── myapp-api.json           ← Controller + middleware + DI registration
├── myapp-application.json   ← Service'ler + MediatR handler'lar
├── myapp-domain.json        ← Entity'ler + value object'ler + domain event'ler
└── myapp-infrastructure.json ← Repository impl + DbContext + HttpClient'lar
```

`_index.json` içeriği:
```json
{
  "projeler": [
    { "isim": "MyApp.Api", "dosya": "kb/myapp-api.json", "ozet": "HTTP giriş noktası", "kritiklik": "YUKSEK" },
    { "isim": "MyApp.Domain", "dosya": "kb/myapp-domain.json", "ozet": "Core iş mantığı", "kritiklik": "KRITIK" }
  ],
  "proje_bagimlilik_matrisi": { ... },
  "global_kompleksite_skoru": 9.0
}
```

---

## .NET'e Özel: Yeni Özellik Eklerken

### Tipik Dosya Sırası

```
1. src/Domain/Entities/              → Entity ekle/güncelle
2. src/Infrastructure/Data/          → DbContext'e DbSet ekle, OnModelCreating güncelle
3. Migrations/                       → dotnet ef migrations add XxxMigration
4. src/Application/DTOs/             → Request ve Response DTO'ları
5. src/Application/Validators/       → FluentValidation validator (varsa)
6. src/Application/Queries|Commands/ → MediatR handler (CQRS varsa)
7. src/Domain/Interfaces/            → Repository interface metod ekle
8. src/Infrastructure/Repos/         → Repository implementasyonu
9. src/Application/Services/         → Service metodu
10. src/Api/Controllers/             → Controller action
11. Program.cs                       → DI registration (yeni service)
12. *.Tests/                         → Unit ve integration test yaz
```

### .NET'e Özel Dikkat Edilecekler

- **EF Core migration zorunlu mu?** Entity değişikliği → `dotnet ef migrations add` → `dotnet ef database update`
- **HasQueryFilter (soft delete):** Global query filter varsa `IgnoreQueryFilters()` kullanmadan silinen kayıtlar görünmez
- **Async/await all the way down:** `.Result` / `.GetAwaiter().GetResult()` → deadlock riski, her zaman `await` kullan
- **CancellationToken:** Her async metodda `CancellationToken ct` parametresi olmalı, DB çağrılarına geçirilmeli
- **IOptions scope:** `IOptionsSnapshot<T>` (scoped, her request) vs `IOptions<T>` (singleton — config değişimi yansımaz)
- **DbContext lifetime:** `AddDbContext` default Scoped → singleton servis inject edemez (captive dependency!)
- **AsNoTracking:** Read-only sorgularda performans için zorunlu — eksi görünür ama büyük tablolarda kritik
- **Polly retry + idempotency:** HttpClient'a retry eklenince POST'ların banka/dış servis tarafında idempotent olduğundan emin ol
- **AutoMapper circular reference:** `PreserveReferences()` olmadan circular → stack overflow
- **MediatR pipeline sırası:** Validation → Authorization → Transaction → Actual handler — sıra yanlışsa transaction olmadan validation hatası fırlar
- **IDbContextTransaction içinde harici HTTP:** DB bağlantısı meşgul kalır, connection pool tükenir → harici HTTP'yi transaction dışına al
- **Config drift:** Production deployment öncesi `appsettings.Production.json` ile default'u karşılaştır
- **Feature flag prod'da açık mı?** `appsettings.Production.json`'da `FeatureFlags` section'ını dev ile karşılaştır
- **Swagger Production'da kapalı mı?** `if (env.IsDevelopment())` bloğu dışında `app.UseSwagger()` var mı kontrol et
- **`[AllowAnonymous]` audit:** Tüm `[AllowAnonymous]` endpoint'lerinin listesini güvenlik haritasına ekle — state değiştiriyorsa özellikle işaretle
