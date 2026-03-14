```
██████╗ ███████╗██████╗  ██████╗
██╔══██╗██╔════╝██╔══██╗██╔═══██╗
██████╔╝█████╗  ██████╔╝██║   ██║
██╔══██╗██╔══╝  ██╔═══╝ ██║   ██║
██║  ██║███████╗██║      ╚██████╔╝
╚═╝  ╚═╝╚══════╝╚═╝       ╚═════╝

██╗  ██╗███╗   ██╗ ██████╗ ██╗    ██╗██╗     ███████╗██████╗  ██████╗ ███████╗
██║ ██╔╝████╗  ██║██╔═══██╗██║    ██║██║     ██╔════╝██╔══██╗██╔════╝ ██╔════╝
█████╔╝ ██╔██╗ ██║██║   ██║██║ █╗ ██║██║     █████╗  ██║  ██║██║  ███╗█████╗
██╔═██╗ ██║╚██╗██║██║   ██║██║███╗██║██║     ██╔══╝  ██║  ██║██║   ██║██╔══╝
██║  ██╗██║ ╚████║╚██████╔╝╚███╔███╔╝███████╗███████╗██████╔╝╚██████╔╝███████╗
╚═╝  ╚═╝╚═╝  ╚═══╝ ╚═════╝  ╚══╝╚══╝ ╚══════╝╚══════╝╚═════╝  ╚═════╝ ╚══════╝
```

<div align="center">

**Repo'ları bir kez tara. Sonsuza kadar hızlı geliştir.**

![Claude](https://img.shields.io/badge/Powered%20by-Claude%20Sonnet-7C3AED?style=for-the-badge&logo=anthropic&logoColor=white)
![Java](https://img.shields.io/badge/Java-Spring%20Boot-ED8B00?style=for-the-badge&logo=spring&logoColor=white)
![.NET](https://img.shields.io/badge/.NET-ASP.NET%20Core-512BD4?style=for-the-badge&logo=dotnet&logoColor=white)
![GitLab](https://img.shields.io/badge/GitLab-MCP-FC6D26?style=for-the-badge&logo=gitlab&logoColor=white)
![Azure](https://img.shields.io/badge/Azure%20DevOps-MCP-0078D4?style=for-the-badge&logo=azuredevops&logoColor=white)

</div>

---

## Sorun

Yeni bir projeye katıldın. İlk sprint'te şu soruları sordun:

```
"Yeni endpoint nereye yazılıyor?"        → 30 dk araştırma
"Bu DTO'nun alanları neler?"             → 15 dk araştırma
"UserService değişirse ne etkilenir?"    → 45 dk araştırma
"Bu endpoint auth gerektiriyor mu?"      → 20 dk araştırma
```

**Toplam: Kod yazmadan önce 110 dakika kayıp.**

---

## Çözüm

```
Repo URL'si ver → Agent 4 fazda tarar → kb/repo.json oluşturur → Sonsuza kadar anında cevap
```

```
"UserService değişirse ne etkilenir?"

▶  kb/order-api.json okunuyor...

  ETKİ ANALİZİ:
  UserService → OrderService       (sipariş oluştururken kullanıcı doğrulama)
               → PaymentService    (ödeme için kullanıcı limit kontrolü)
               → AuthService       (token yenileme)
               → ReportService     (kullanıcı bazlı rapor)

  ⚠️  UserService.GetByIdAsync() metodu @Cacheable — değişiklik sonrası
      "users" cache'ini invalidate etmeyi unutma!

  Cevap süresi: <1 saniye  |  Araştırma süresi: 0 dakika
```

---

## Nasıl Çalışır

```
┌─────────────────────────────────────────────────────────────────┐
│                        4 FAZLI TARAMA                           │
└─────────────────────────────────────────────────────────────────┘

   REPO URL
      │
      ▼
┌─────────────┐    ~5k token
│   FAZ 1     │ ──────────────  Klasör ağacı + dil tespiti
│   KEŞİF     │                 Mimari pattern tanıma
└──────┬──────┘                 Dosya öncelik listesi
       │
       ├──────────────────────────────────────────────┐
       │                                              │
       ▼                                              ▼
┌─────────────┐    ~20k token              ┌─────────────┐
│   FAZ 2     │ ──────────────             │   FAZ 2     │
│ Java/Spring │  Controller'lar            │  .NET/Core  │
│   ÇEKÜDEK   │  Service'ler               │   ÇEKÜDEK   │
│   TARAMA    │  Entity'ler                │   TARAMA    │
└──────┬──────┘  Repository'ler            └──────┬──────┘
       │                                          │
       └─────────────────┬────────────────────────┘
                         │
                         ▼
              ┌─────────────────────┐    ~25k token
              │       FAZ 3         │ ──────────────
              │   DERİNLEMESİNE     │  Execution Flow
              │      EK TARAMA      │  Performance Risk
              │                     │  Security Scan
              └──────────┬──────────┘  Config Drift
                         │
                         ▼
              ┌─────────────────────┐
              │       FAZ 4         │
              │   KB'YE YAZ         │──▶  kb/repo-adi.json
              └─────────────────────┘

  Toplam: ~50-100k token  |  1 kez yapılır  |  Sonsuza kadar kullanılır
```

---

## Ne Öğrenir?

```
┌──────────────────────────────────────────────────────────┐
│  kb/order-api.json                          ~400KB       │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  📍 API Endpoint Haritası                                │
│     POST /api/orders/create  →  [Auth: USER rolü]        │
│     GET  /api/orders/{id}    →  [Auth: JWT gerekli]      │
│     DELETE /api/orders/{id}  →  [Auth: ADMIN rolü]       │
│                                                          │
│  🔀 Execution Flow (EN DEĞERLİ)                          │
│     Controller → Service → Repository → DB               │
│     + Harici HTTP çağrıları                              │
│     + Kuyruk mesajları                                   │
│     + Async yan etkiler                                  │
│                                                          │
│  🕸️  Bağımlılık Grafı                                    │
│     OrderService ──▶ [UserService, PaymentService]       │
│     UserService  ──▶ [UserRepo, CacheManager]  ← HUB    │
│     ETKİ: UserService değişirse → 4 servis etkilenir     │
│                                                          │
│  ⚡ Performance Riskleri                                 │
│     N+1 sorgu: OrderService.java:87                      │
│     Transaction içinde HTTP: PaymentService.java:134     │
│                                                          │
│  🔒 Güvenlik Riskleri                                    │
│     SQL Injection: UserRepository.java:45                │
│     Eksik @Valid: OrderController.java:23                │
│                                                          │
│  🌍 Config Drift                                         │
│     Dev: MaxPoolSize=5  |  Prod: MaxPoolSize=50          │
│     FeatureFlags:NewCheckout → sadece prod'da aktif!     │
│                                                          │
│  🏗️  Mimari Karmaşıklık Skoru: 7.2/10  [ORTA]           │
│     Değişiklikte geniş test kapsamı öneriliyor           │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## Desteklenen Diller

<table>
<tr>
<td width="50%">

### ☕ Java / Spring Boot

```
✅ @RestController endpoint'leri
✅ @Service iş mantığı
✅ @Entity + JPA ilişkileri
✅ @Query JPQL + Native SQL
✅ @Aspect AOP detayları
✅ @RabbitListener / @KafkaListener
✅ @Transactional sınırları
✅ Flyway / Liquibase migration
✅ N+1 sorgu tespiti
✅ Security filter chain
✅ application.yml drift
```

</td>
<td width="50%">

### 🔷 .NET / ASP.NET Core

```
✅ [ApiController] endpoint'leri
✅ IXxxService iş mantığı
✅ EF Core Entity + Fluent API
✅ LINQ + Dapper + Raw SQL
✅ IActionFilter / Middleware
✅ MassTransit / Azure Service Bus
✅ IDbContextTransaction sınırları
✅ EF Core Migration
✅ Captive Dependency tespiti
✅ IAuthorizationHandler policy
✅ appsettings.json drift
```

</td>
</tr>
<tr>
<td>

**MediatR CQRS desteği:**
```
Command → Pipeline → Handler
  ↓           ↓          ↓
Validation  Logging   DB + Event
```

</td>
<td>

**MediatR CQRS desteği:**
```
Command → IPipelineBehavior → IRequestHandler
  ↓              ↓                  ↓
Validate     Transaction        DB + Publish
```

</td>
</tr>
</table>

---

## Çıktı Formatı

```jsonc
// kb/order-api.json  (örnek)
{
  "meta": {
    "tarama_tarihi": "2026-03-14",
    "dil": "java",
    "kompleksite_skoru": { "toplam": 7.2, "seviye": "ORTA" }
  },

  // 🗺️  Her endpoint'in tam execution path'i
  "execution_flows": [
    {
      "endpoint": "POST /api/orders/create",
      "kritiklik_puani": 8,
      "adimlar": [
        { "sira": 1, "tip": "DB_READ",    "detay": "UserRepository.findById()" },
        { "sira": 2, "tip": "HTTP_CALL",  "detay": "InventoryClient.checkStock() ⚠️ @Transactional içinde" },
        { "sira": 3, "tip": "DB_WRITE",   "detay": "OrderRepository.save()" },
        { "sira": 4, "tip": "QUEUE_SEND", "detay": "order.created → async email + analytics" }
      ]
    }
  ],

  // 🕸️  Değişiklik etki analizi
  "bagimlilik_grafigi": {
    "etki_analizi": {
      "UserService_degisirse": ["OrderService", "PaymentService", "AuthService", "ReportService"]
    },
    "kritik_merkez_servisler": ["UserService", "PaymentService"]
  },

  // ⚡ Performance + güvenlik riskleri
  "performance_riskleri": [...],
  "guvenlik_riskleri": [...],
  "config_drift": {...}
}
```

---

## Kurulum

Bu repo'yu klonla ve AI asistanınla aynı workspace'te aç:

```bash
git clone https://github.com/your-org/repo-knowledge
# Claude Code veya VS Code ile aç
```

Başka bir şey kurman gerekmez — agent ve skill dosyaları `.rk/` klasöründe hazır.

---

## Ön Gereklilikler — MCP Sunucuları

Repo kaynak koduna erişmek için AI asistanının aşağıdaki MCP sunucularından en az birine bağlı olması gerekir:

```
┌────────────────────────────────────────────────────────────┐
│                    ZORUNLU MCP ARAÇLARI                    │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  GitLab MCP          →  GitLab repo erişimi                │
│  (gitlab-org/gitlab  │  Branch, dosya, klasör ağacı        │
│   MCP server)        │                                     │
│                                                            │
│  Azure DevOps MCP    →  Azure DevOps repo erişimi          │
│  (microsoft/azure-   │  Repo, pipeline, artifact           │
│   devops-mcp)        │                                     │
│                                                            │
│  Filesystem MCP      →  kb/ klasörüne yaz/oku              │
│  (built-in veya      │  Knowledge base JSON'ları           │
│   @modelcontextprot  │                                     │
│   ocol/server-fs)    │                                     │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

**Sadece public GitHub repo'ları için** ek MCP gerekmez — Claude doğrudan web fetch ile erişebilir.

### Claude Code için MCP kurulumu (örnek)

```bash
# GitLab MCP
claude mcp add gitlab -- npx -y @modelcontextprotocol/server-gitlab

# Azure DevOps MCP
claude mcp add azure-devops -- npx -y @microsoft/azure-devops-mcp

# Filesystem MCP (kb/ klasörü için)
claude mcp add filesystem -- npx -y @modelcontextprotocol/server-filesystem /path/to/repo-knowledge/kb
```

---

## Nasıl Kullanılır

### Claude Code (Slash Commands)

```bash
# Repo tara
/rk-scan https://gitlab.com/org/order-api

# Ecosystem haritası oluştur
/rk-map

# Geliştirici sorusu sor
/rk-advise "UserService değişirse ne etkilenir?"
```

### GitHub Copilot (VS Code)

`.github/copilot-instructions.md` otomatik yüklenir. Direkt yaz:

```
@workspace şu repo'yu tara: https://gitlab.com/org/order-api
```

```
@workspace UserService değiştirirsem ne etkilenir?
```

### Doğal Dil (her AI asistan)

```
"şu repo'yu tara: https://gitlab.com/org/order-api"
"Hangi repo hangisini kullanıyor?"
"POST /api/orders endpoint'i nasıl çalışıyor?"
```

CLAUDE.md ve .github/copilot-instructions.md routing kurallarını barındırır — AI hangi agent'ı çağıracağını otomatik bilir.

---

## Klasör Yapısı

```
repo-knowledge/
│
├── 📄 CLAUDE.md                    ← Claude Code giriş noktası + routing
├── 📄 .github/copilot-instructions.md  ← GitHub Copilot giriş noktası
│
├── 📁 .rk/                         ← Tüm agent ve skill'ler (single source)
│   ├── 📁 agents/
│   │   ├── 📄 repo-scanner.agent.md    ← 4 fazlı tarama protokolü
│   │   ├── 📄 ecosystem-mapper.agent.md ← Multi-repo bağımlılık haritası
│   │   └── 📄 dev-advisor.agent.md     ← Geliştirici soru/etki analizi
│   └── 📁 skills/
│       ├── 📁 scan-java/SKILL.md       ← Java/Spring Boot derin tarama
│       └── 📁 scan-dotnet/SKILL.md     ← .NET/ASP.NET Core derin tarama
│
├── 📁 .claude/commands/            ← Claude Code slash komutları
│   ├── 📄 rk-scan.md               ← /rk-scan
│   ├── 📄 rk-map.md                ← /rk-map
│   └── 📄 rk-advise.md             ← /rk-advise
│
└── 📁 kb/                          ← Oluşturulan knowledge base'ler
    ├── 📄 _progress.json           ← Tarama ilerleme takibi
    ├── 📄 _index.json              ← Multi-repo genel index
    ├── 📄 order-api.json           ← Taranmış repo #1
    └── 📄 payment-service.json     ← Taranmış repo #2
```

---


## Analiz Çıktısı — Örnek

```
Kullanıcı: "Order API'ye fatura oluşturma endpoint'i eklemek istiyorum"

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  📍 Hangi repo? → order-api (kb/order-api.json)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Yazılacak Dosyalar (sırayla):

  1.  src/main/java/com/example/domain/entity/Invoice.java
      └─ @Entity  @Table(name="invoices")  Soft delete: isDeleted

  2.  src/main/resources/db/migration/V12__create_invoices_table.sql
      └─ ⚠️  ddl-auto: none — migration ZORUNLU!

  3.  src/main/java/com/example/dto/CreateInvoiceDto.java
      └─ Mevcut pattern: @Validated + @NotNull + @Builder (Lombok)

  4.  src/main/java/com/example/repository/InvoiceRepository.java
      └─ JpaRepository<Invoice, Long> extend et

  5.  src/main/java/com/example/service/InvoiceService.java
      └─ @Service  @Transactional(readOnly = true)  —  sınıf seviyesi pattern bu

  6.  src/main/java/com/example/controller/InvoiceController.java
      └─ @RestController  @RequestMapping("/api/invoices")

  7.  SecurityConfig.java
      └─ POST /api/invoices → USER rolü gerekiyor mu? Şu an "/**" authenticated

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ⚠️  Dikkat Edilecekler:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  → InvoiceService, OrderService'e bağımlı olacaksa:
    OrderService 4 noktadan inject ediliyor (kritik merkez servis)
    Mutlaka @Transactional(readOnly=true) uyumluluğunu kontrol et

  → Fatura PDF üretimi harici servis gerektiriyorsa:
    @Transactional scope dışında yap (HTTP timeout riski!)

  → Test: OrderService için test var ama InvoiceService için yok olacak
    kapsanmayan_kritik_siniflar listesine InvoiceService eklenecek

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---


## Tarama Tamamlandığında

```
✅ Kaydedildi: kb/order-api.json
──────────────────────────────────────────────────────────
Dil        : Java 21
Mimari     : Modüler Monolit — N-Tier (4 katman)
Stack      : Spring Boot 3.2 + PostgreSQL + Redis + RabbitMQ
Endpoint   : 6 controller, 24 endpoint
Servis     : 12 service, 80+ metod
Model      : 8 entity, index detayları dahil
DTO        : 14 istek/yanıt şeması
Enum       : 7 enum, tüm değerler
Repo       : 9 repository, 18 custom query
Güvenlik   : permitAll listesi, 1 chain, 4 özel annotation
Mesaj      : 6 consumer, 10 producer
Flow       : 8 kritik endpoint execution path'i
Perf Risk  : 3 N+1, 1 HTTP-in-transaction, 2 sayfalama eksik
Sec Risk   : 1 SQL injection (KRITIK), 2 eksik @Valid
Config     : Dev↔Prod 4 değer farkı tespit edildi
Kompleksite: 7.2/10 [ORTA] — geniş test kapsamı öneriliyor
Test       : 4/12 kritik sınıf kapsanmış, 8 boşluk
Borç       : 3 FIXME (1 yüksek risk), 2 DEPRECATED
Token      : Faz1 ~5k + Faz2 ~20k + Faz3 ~25k = ~50k
──────────────────────────────────────────────────────────
```

---

<div align="center">

**Bir kez tara. Sonsuz kez kazan.**

*Her repo taraması, tüm ekibin önündeki belirsizliği sıfırlar.*

</div>
