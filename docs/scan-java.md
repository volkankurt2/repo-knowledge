# Java Repo Tarama — AŞAMA 2 ve AŞAMA 3

Bu dosya **yalnızca Java/Spring Boot repoları** için geçerlidir.
Dil tespiti: `pom.xml`, `build.gradle`, `settings.gradle` dosyaları tespit edildiğinde bu dosyayı uygula.

---

## ⚡ OKUMA STRATEJİSİ — LLM Stabilite Protokolü

> **Bu bölümü önce oku. Token bütçeni ve okuma sıranı belirler.**

### Token Bütçesi (toplam ~45k token)

| Bölüm | Token Hedefi | Öncelik |
|---|---|---|
| Controller'lar | ~8k | KRİTİK |
| Service'ler | ~12k | KRİTİK |
| Entity'ler | ~5k | YÜKSEK |
| Repository'ler | ~4k | YÜKSEK |
| DTO'lar | ~5k | ORTA |
| Security + AOP | ~4k | YÜKSEK |
| Migration + Config | ~3k | ORTA |
| Test + Teknik Borç | ~4k | DÜŞÜK |

### Dosya Boyutuna Göre Okuma Kuralı

```
< 150 satır  → TAM OKU
150–400 satır → class declaration + constructor + public metod imzaları + annotation'lar
> 400 satır  → SADECE imzalar + annotation'lar + class-level Javadoc
```

> **Lombok projelerinde:** `@Data`, `@Builder`, `@Getter`/`@Setter` gören yerde getter/setter satırlarını atla — bunlar implementation değil, boilerplate.

### Okuma Öncelik Sırası

```
1. Controller → Service → Repository    (execution path)
2. Entity → DTO                         (data model)
3. SecurityConfig + ExceptionHandler    (cross-cutting)
4. AOP Aspects → Consumer/Producer      (side-effects)
5. Migration → Config → Test            (infrastructure)
```

### LLM Stabilite Kuralları

- Her dosyayı okurken **önce bağlamı yaz**: "Şimdi OrderService okuyorum — OrderController'dan çağrılan iş katmanı"
- 5+ dosya okuduktan sonra kısa bir ara özet yaz: "Şimdiye kadar öğrendiklerim: ..."
- Büyük metodlarda sadece **yan etkileri** çıkar: DB yazımı, kuyruk gönderimi, cache temizleme, harici HTTP çağrısı
- Emin olmadığında `null` yaz, tahmin etme — yanlış bilgi doğru bilgisizlikten tehlikelidir

---

## REPO BOYUTU TESPİTİ — Monorepo Protokolü

> `settings.gradle` veya root `pom.xml` içinde **5+ submodül** varsa bu bölümü uygula.

### Monorepo Modül Haritası

```
[root] ─┬─ api-gateway/          ← giriş noktası
        ├─ user-service/         ← kullanıcı yönetimi
        ├─ order-service/        ← sipariş yönetimi
        ├─ common/               ← paylaşılan DTO, util
        └─ infra/                ← config, security base
```

**Modül tiplerini sınıfla:**
- **Leaf (yaprak):** Başka iç modüle bağımlı değil, sadece `common` kullanır
- **Core (çekirdek):** 3+ başka modül tarafından import edilen → değişiklikte domino etkisi
- **Gateway (geçit):** Tüm trafiği yönlendiren — routing config kritik

### Hangi Modülleri Derin Tara?

- **Core modüller:** Her zaman tam tara
- **Görevle ilgili modüller:** Kullanıcının sorduğu domain → tam tara
- **Leaf modüller:** Sadece controller listesi + servis imzaları yeterli
- **Infra/Common:** Entity'ler + shared DTO'lar + enum'lar tam tara; util sınıfları atla

### Modül Bağımlılık Matrisi (KB'ye yaz)

```json
"modul_bagimlilik_matrisi": {
  "order-service": { "import_ettigi": ["common", "user-service-client"], "import_eden": [] },
  "common": { "import_ettigi": [], "import_eden": ["order-service", "user-service", "payment-service"] },
  "core_modul_uyarisi": "common değişirse 3 modül etkilenir"
}
```

---

## AŞAMA 2 — Çekirdek Tarama (~20k token)

**Hedef:** Bir geliştirici "nereye ne yazacak?" sorusunu yanıtlayacak temel detayı topla.

### 2.1 Controller / Resource dosyaları (EN KRİTİK)

Her `@RestController` / `@Controller` sınıfını oku. Her endpoint için:
- HTTP metodu: `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, `@PatchMapping`
- Route: class-level `@RequestMapping` + method-level path (tam path'i birleştir)
- Parametre tipleri:
  - `@PathVariable Long id` → path parametresi
  - `@RequestParam String filter` → query string
  - `@RequestBody CreateOrderDto dto` → body (hangi DTO?)
  - `@RequestHeader("X-Tenant-Id") String tenantId` → header
- Döndürdüğü tip: `ResponseEntity<UserDto>`, `List<OrderDto>`, `void`
- Security: `@PreAuthorize("hasRole('ADMIN')")`, `@RolesAllowed`, class-level `@Secured`
- Özel annotation'lar: `@RequiresOtp`, `@AuditLog`, `@TwoFactor`, `@RateLimited` gibi AOP trigger'lar

#### Kritik Endpoint İşaretleme

Her endpoint için aşağıdaki kritiklik puanını hesapla ve KB'ye yaz:

| Sinyal | +Puan |
|---|---|
| Finansal işlem: ödeme, tahsilat, iade, provizyon | +3 |
| State-mutating: DB write + kuyruk gönderimi | +2 |
| Harici servis çağrısı içeriyor | +2 |
| 3+ servis çağırıyor | +2 |
| `@Transactional` scope içinde harici HTTP var | +3 |
| Auth bypass riski: `permitAll` ama data değiştiriyor | +3 |

```json
"kritik_endpoint_listesi": [
  {
    "endpoint": "POST /api/payments/process",
    "kritiklik_puani": 8,
    "neden": "Finansal işlem + harici banka çağrısı + DB yazımı + kuyruk",
    "degisiklik_riski": "YÜKSEK — regression test şart"
  }
]
```

### 2.2 Service sınıfları

Her `@Service` / `@Component` sınıfından:
- **Sınıf seviyesi annotation'lar:** `@Transactional`, `@CacheConfig(cacheNames="...")`, `@Async`
- **Constructor injection** → bağımlılık listesi:
  - `XxxRepository` / `XxxJpaRepository` → repository
  - `XxxService` / `XxxManager` → servis→servis bağımlılığı
  - `XxxFeignClient` / `XxxRestTemplate` / `XxxWebClient` → harici client
  - Geri kalanlar → diger
- **Her public metod için:**
  - İmza: `public OrderDto createOrder(CreateOrderDto dto)`
  - Metod annotation'ları: `@Transactional`, `@Transactional(readOnly = true)`,
    `@Cacheable(value="orders", key="#id")`, `@CacheEvict`, `@CachePut`,
    `@Async`, `@Scheduled(cron="...")`, `@Retryable(maxAttempts=3)`
  - Kısa iş mantığı özeti (1 cümle) — özellikle yan etkiler: async email, kuyruk mesajı, cache temizleme

> 400 satırdan uzun dosyalarda: class + constructor + public method imzaları + annotation'lar yeterli.

### 2.3 Entity sınıfları

Her `@Entity` sınıfından:
- `@Table(name="tablo_adi")` → DB tablo adı
- **Her field:**
  - `@Id @GeneratedValue` → PK tipi (AUTO, SEQUENCE, IDENTITY)
  - `@Column(name="...", unique=true, nullable=false, length=255)`
  - `@NotNull`, `@NotBlank`, `@Size`, `@Email` validation
  - `@Enumerated(EnumType.STRING)` veya `EnumType.ORDINAL`
  - `@Lob` → büyük veri
  - `@JsonIgnore`, `@JsonProperty` serialization
- **İlişkiler:**
  - `@OneToMany(cascade=CascadeType.ALL, fetch=FetchType.LAZY, mappedBy="order")`
  - `@ManyToOne(fetch=FetchType.EAGER) @JoinColumn(name="user_id")`
  - `@ManyToMany @JoinTable(...)`
- `@Table(indexes={@Index(columnList="email", unique=true)})` → index tanımları
- Soft delete: `isDeleted`, `deletedAt` field var mı?
- Audit fields: `@CreatedDate`, `@LastModifiedDate`, `@CreatedBy` var mı?
- `@Inheritance(strategy=InheritanceType.SINGLE_TABLE)` → kalıtım pattern var mı?

**Hibernate XML mapping kullanan projeler:**
Her `*.hbm.xml` dosyasını oku — `<class name="..." table="...">`, `<property>`, `<many-to-one>` eşlemelerini çıkar.

### 2.4 Repository / DAO interface'leri

`JpaRepository<Entity, ID>` extend eden her interface'i oku:
- Spring Data method isimleri: `findByEmail`, `findByStatusAndDeletedFalse`, `countByCustomerId`
- `@Query("SELECT u FROM User u WHERE ...")` → JPQL sorgu metni
- `@Query(value="SELECT * FROM users WHERE ...", nativeQuery=true)` → native SQL
- `@Modifying @Transactional @Query(...)` → güncelleme/silme işlemleri (önemle belirt)
- `@Lock(LockModeType.PESSIMISTIC_WRITE)` → hangi sorgular lock alıyor
- `@Procedure(name="stored_proc_name")` → stored procedure çağrısı
- `Specification<T>` veya `Criteria API` kullanan custom metodlar

**MyBatis kullanan projeler:**
`*Mapper.java` interface metodları + `*Mapper.xml` dosyasındaki SQL sorgularını oku.

### 2.5 Bağımlılık Grafını Çıkar (KRİTİK ADIM)

Her servis sınıfının constructor'ını oku ve şu ayrımı yap:
- `XxxRepository` / `*JpaRepository` → `bagimliliklar.repository`
- `XxxService` / `XxxManager` / `XxxFacade` → `bagimliliklar.servis`
- `XxxFeignClient` / `XxxRestTemplate` / `XxxWebClient` → `bagimliliklar.harici_client`
- `ILogger` / `ObjectMapper` / `ModelMapper` / `ApplicationEventPublisher` → `bagimliliklar.diger`

`bu_servisi_kullananlar`: Tüm bağımlılık listelerini tara, her servisi hangi class'ların constructor'ında inject ettiğini bul.

`kritik_merkez_servisler`: 3+ farklı noktadan inject edilen servis = merkez servis. İşaretle.

#### Bağımlılık Grafı Görselleştirme

KB'ye JSON adjacency list'e ek olarak şu metin görselini de yaz — geliştiriciler JSON'u parse etmez ama bunu okur:

```
BAĞIMLILIK AĞACI:
OrderService ──> [OrderRepository, UserService, PaymentService, RabbitTemplate]
PaymentService ──> [PaymentRepository, BankApiClient, AuditService]
UserService ──> [UserRepository, CacheManager]               ← HUB (4 servis kullanıyor)

ETKİ ANALİZİ (değişirse etkilenenler):
UserService    → OrderService, PaymentService, AuthService, ReportService  [4 etki]
PaymentService → OrderService, SubscriptionService                         [2 etki]
BankApiClient  → PaymentService                                            [1 etki]

DÖNGÜSEL BAĞIMLILIK KONTROLÜ: ✅ Döngü yok | ⚠️ [ServiceA ↔ ServiceB] döngü!
```

### 2.6 Startup ve DI Konfigürasyonu

`@SpringBootApplication` class'ı ve `@Configuration` class'larını oku:
- `@Bean` tanımları: hangi interface → hangi implementation
- `@ComponentScan(basePackages="...")` → tarama kapsamı
- `@EnableJpaRepositories`, `@EnableCaching`, `@EnableAsync`, `@EnableScheduling`
- `@Profile("prod")` gibi ortam bazlı konfigürasyonlar
- `@ConditionalOnProperty` ile feature flag benzeri bean'ler

---

## AŞAMA 3 — Derinlemesine Ek Tarama (~25k token)

**Hedef:** Geliştirici bağlamını tam olarak kurmak.

### 3.1 DTO / Request / Response Şema Kataloğu

Önemli DTO'ları oku. Önceliklendirme:
1. Controller'larda `@RequestBody` olarak kullanılanlar
2. Controller'lardan dönülenler
3. Queue consumer/producer'larda kullanılan mesaj DTO'ları
4. Feign client parameterlerindeki DTO'lar

Her DTO için:
- Tüm field isimleri ve Java tipleri (`String`, `Long`, `BigDecimal`, `LocalDateTime`)
- Validation: `@NotNull`, `@NotBlank`, `@Size(min=1, max=100)`, `@Pattern(regexp="...")`,
  `@DecimalMin`, `@Email`, `@Valid` (nested DTO)
- JSON: `@JsonProperty("snake_case_name")`, `@JsonIgnore`, `@JsonInclude(NON_NULL)`,
  `@JsonFormat(pattern="yyyy-MM-dd")`
- İç içe DTO varsa onları da belirt (recursive)
- Lombok: `@Data`, `@Builder`, `@AllArgsConstructor`, `@NoArgsConstructor` kullanımı

### 3.2 Enum Kataloğu

Tüm enum dosyalarını oku:
- Tüm değerler: `ACTIVE, PASSIVE, SUSPENDED`
- Sayısal değer varsa: `PAYMENT(1), CANCEL(2)` — `@JsonValue`, ordinal mi string mi?
- `@Enumerated(EnumType.STRING)` mı `ORDINAL` mı? (kritik!)
- Kullanım yerleri: hangi entity field'ı, hangi DTO, hangi factory switch-case
- Domain-critical enum'lara özellikle dikkat: `PaymentStatus`, `TransactionType`,
  `ProvisionType`, `OrderStatus`, `UserRole`, `NotificationType`

### 3.3 Repository Custom Query Kataloğu

Her repository'nin tüm metodlarını tara:
- Derived method isimleri (Spring Data'nın anlamlandırdığı): `findByStatusIn`, `existsByEmail`
- `@Query` ile JPQL sorgular → tam sorgu metnini al
- `@Query(nativeQuery=true)` → tam native SQL al
- `@Modifying @Transactional @Query(...)` → özellikle işaretle (güncelleme/silme riski)
- `@Lock(PESSIMISTIC_WRITE)` → deadlock riski olan sorgular
- `@Procedure` → stored procedure
- `Pageable` kullanan metodlar: sayfalama nasıl yapılıyor?

### 3.4 Güvenlik Haritası

`WebSecurityConfigurerAdapter` / `SecurityFilterChain` @Bean metodlarını tam oku:

```
.antMatchers("/healthcheck", "/actuator/**").permitAll()
.antMatchers("/api/admin/**").hasRole("ADMIN")
.antMatchers("/**").authenticated()
```

- `permitAll()` → authentication gerektirmeyen URL'lerin tam listesi
- `hasRole()` / `hasAuthority()` → URL-rol eşleşmeleri
- Birden fazla security chain varsa (`@Order(1)`, `@Order(2)`) → ayrı ayrı belirt
- Session policy: `.sessionManagement().sessionCreationPolicy(STATELESS)`
- CORS: `.cors()` konfigürasyonu, hangi origin'ler izinli?
- Custom filter'lar: `addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)`
- Custom annotation güvenliği:
  - `@PreAuthorize("hasRole('ADMIN')")` → hangi metodlarda?
  - `@RequiresOtp` gibi custom → AOP implementasyonunu oku (ne yapıyor?)
- `@PermitAll` / `@DenyAll` method-level güvenlik

### 3.5 Exception Handling

`@ControllerAdvice` / `@RestControllerAdvice` sınıflarını oku:
- Her `@ExceptionHandler(XxxException.class)` → hangi HTTP status, hangi response body format?
- Custom exception class'ları: isim + tipik fırlatılma senaryosu + inherit ettiği sınıf
- `@ResponseStatus(HttpStatus.NOT_FOUND)` annotation'lı exception'lar
- `@Retryable(value={ConnectException.class}, maxAttempts=3, backoff=@Backoff(delay=1000, multiplier=2))`
- Resilience4j / Hystrix circuit breaker: `@CircuitBreaker(name="partnerApi", fallbackMethod="fallback")`
- Dead letter queue: hangi kuyruklar DLQ'ya düşüyor, DLQ processing var mı?

### 3.6 AOP / Aspect Detayları

Her `@Aspect` sınıfını oku:
- `@Around("@annotation(RequiresOtp)")` → hangi annotation'ı intercept ediyor?
- `@Before("execution(* com.example.service.*.*(..))")` → hangi paket/method pattern?
- `@After`, `@AfterReturning`, `@AfterThrowing`
- Yan etkiler: Redis'e yazıyor mu? Audit log'a yazıyor mu? Header set ediyor mu? OTP kontrol ediyor mu?
- `@Order(1)` → birden fazla aspect varsa sıralama önemli
- `@EnableAspectJAutoProxy` → AspectJ kullanılıyor mu yoksa Spring Proxy mi?

### 3.7 Mesaj / Event Şemaları

**RabbitMQ:**
- `@RabbitListener(queues="${queue.name}")` → hangi kuyruk
- `@RabbitListener(bindings=@QueueBinding(value=@Queue("..."), exchange=@Exchange("..."), key="routing.key"))`
- Dead letter: `x-dead-letter-exchange`, `x-dead-letter-routing-key`
- `prefetch` / `concurrency` ayarları
- `@SendTo("response.queue")` → reply-to pattern

**Kafka:**
- `@KafkaListener(topics="${kafka.topic}", groupId="${kafka.group}")` → topic + consumer group
- `@KafkaListener(topicPartitions=@TopicPartition(...))`
- `KafkaTemplate.send(topic, key, value)` → producer
- `ErrorHandler` / `SeekToCurrentErrorHandler` → hata yönetimi

**Her consumer/producer için:**
- Mesaj DTO sınıfı → alanları listele (3.1'de tanımlandıysa referans ver)
- Event zinciri: "X kuyruğundan gelince Y olur, sonra Z kuyruğuna gider"

### 3.8 Migration / Schema Durumu

**Flyway kullanan projeler:**
- `db/migration/` klasöründeki dosyaları listele
- Naming: `V{versiyon}__{aciklama}.sql` — son 5 migration'ı oku
- `spring.flyway.locations`, `spring.flyway.baseline-on-migrate`

**Liquibase kullanan projeler:**
- `changelog.xml` / `changelog.yaml` master dosyasını oku
- `<changeSet author="..." id="...">` ile son değişiklikleri bul
- `liquibase.changelog`, `liquibase.rollback-file`

**Ortak:**
- `spring.jpa.hibernate.ddl-auto` → `none` (migration zorunlu!), `validate`, `update`
- Migration komutu: `mvn flyway:migrate`, `mvn liquibase:update`
- Son migration'larda eklenen kritik index'ler ve constraint'ler

### 3.9 Test Durumu

`src/test/` klasörünü tara:
- `@SpringBootTest` → tam uygulama context'i mi? Hangi profile? (`@ActiveProfiles("test")`)
- `@WebMvcTest(UserController.class)` → sadece controller layer mı?
- `@DataJpaTest` → sadece JPA repository layer mı? H2 mi yoksa Testcontainers mı?
- Mock framework: `@MockBean`, `@Mock` (Mockito), `@Spy`
- WireMock kullanan test var mı? (harici servis mock'u)
- `@Testcontainers` + `@Container` → gerçek Docker container'ı
- Test naming: `void should_doX_when_Y()` convention var mı?
- Hangi prodüksiyon sınıflarının test'i mevcut? Hangileri eksik?

### 3.10 Feign / REST Client Kataloğu

`@FeignClient` interface'lerini oku:
- `@FeignClient(name="partner-api", url="${clients.partner.url}")` → URL config key'i
- Her metod: HTTP metodu + path + parametre tipleri + dönüş tipi
- `@RequestHeader("Authorization")` → header gereksinimleri
- Timeout: `feign.client.config.{name}.connectTimeout`, `readTimeout`
- Retry: `@Retryable` veya Feign `Retryer` konfigürasyonu
- Fallback: `fallback=PartnerApiFallback.class`

`RestTemplate` / `WebClient` kullanan custom client'lar:
- Bean tanımı + base URL + timeout konfigürasyonu
- Interceptor'lar: auth header ekleme, logging

### 3.11 Kritik Konfigürasyonlar

`application.yml` / `application.properties` / `application-prod.yml` tam oku:

```yaml
# Her placeholder'ı çıkar:
spring.datasource.url: ${DB_URL:jdbc:postgresql://localhost:5432/mydb}
spring.redis.host: ${REDIS_HOST:localhost}
rabbitmq.host: ${RABBITMQ_HOST:localhost}
clients.partner.url: ${PARTNER_API_URL:http://localhost:8081}
```

- Tüm `${ENV_VAR:defaultValue}` placeholder'larını listele
- DB connection pool: `maximum-pool-size`, `minimum-idle` (HikariCP)
- Cache TTL konfigürasyonları
- Scheduled job cron expression'ları
- Feature flag tarzı boolean property'ler: `features.new-payment-flow.enabled: false`
- Logging level overrides: `logging.level.com.example=DEBUG`

`bootstrap.yml` varsa:
- Spring Cloud Config bağlantısı: `spring.cloud.config.uri`
- Vault entegrasyonu: `spring.cloud.vault.uri`

### 3.12 Teknik Borç Haritası

Kaynak kodda şunları ara:
- `// TODO:`, `// FIXME:`, `// HACK:`, `// XXX:` comment'leri → konum + içerik
- `@Deprecated` annotation'lı metodlar → alternatif belirtilmiş mi?
- Magic number'lar: hardcoded sayısal sabitler (özellikle limit, timeout değerleri)
- Hardcoded URL: `http://internal-service/api/...`
- Hardcoded credential: şifre, token, API key
- `@SuppressWarnings("unchecked")` → tip güvensizliği

---

### 3.13 Endpoint → Execution Flow (EN BÜYÜK KAZANÇ)

> **Bu bölüm bir geliştirici için en değerli çıktıdır.** Kritiklik puanı 5+ olan her endpoint için tam execution path'i çıkar.

#### Nasıl Çıkarılır

Controller metodunu oku → çağırdığı servis metodunu bul → o servisin `@Transactional` scope'unu belirle → repository çağrılarını listele → harici HTTP çağrılarını bul → async işlemleri işaretle.

#### Format

```
POST /api/orders/create                                    [kritiklik: 7]
│
├─ OrderController.createOrder(@Valid @RequestBody CreateOrderDto)
│   └─ @PreAuthorize("hasRole('USER')")
│
├─ OrderService.createOrder(dto)                          [@Transactional]
│   ├─ [1] UserService.validateUser(dto.userId)           [servis çağrısı]
│   │       └─ UserRepository.findById()                  [DB okuma]
│   ├─ [2] InventoryClient.checkStock(dto.items)          [HTTP POST /inventory/check, timeout:5s]
│   │       └─ ⚠️  @Transactional içinde — bağlantı havuzu meşgul kalır
│   ├─ [3] OrderRepository.save(order)                    [DB yazımı]
│   ├─ [4] rabbitTemplate.send("order.created", event)   [kuyruk — fire & forget]
│   └─ [5] return OrderResponseDto
│
└─ YAN ETKİLER (async):
    ├─ OrderCreatedConsumer → EmailService.sendConfirmation()
    └─ OrderCreatedConsumer → AnalyticsService.trackOrder()

UYARILAR:
⚠️  @Transactional içinde harici HTTP çağrısı (InventoryClient) — timeout riski
⚠️  Kuyruk gönderimi transaction başarısız olursa yine de gerçekleşir (at-least-once)
```

**KB'de `execution_flows` alanına yaz:**
```json
"execution_flows": [
  {
    "endpoint": "POST /api/orders/create",
    "kritiklik_puani": 7,
    "adimlar": [
      { "sira": 1, "tip": "DB_READ", "detay": "UserRepository.findById()" },
      { "sira": 2, "tip": "HTTP_CALL", "detay": "InventoryClient.checkStock() — ⚠️ @Transactional içinde" },
      { "sira": 3, "tip": "DB_WRITE", "detay": "OrderRepository.save()" },
      { "sira": 4, "tip": "QUEUE_SEND", "detay": "order.created → async email + analytics" }
    ],
    "uyarilar": ["@Transactional içinde harici HTTP", "kuyruk at-least-once semantiği"]
  }
]
```

---

### 3.14 Performance Risk Detection

> Her servis ve repository metodunu okurken şu anti-pattern'leri işaretle.

#### N+1 Sorgu Riski

```java
// ⚠️ RİSK: Her order için ayrı SQL çalışır
orders.forEach(o -> o.getItems().size()); // LAZY collection loop içinde

// ✅ ÇÖZÜM: JOIN FETCH veya @EntityGraph
@EntityGraph(attributePaths = {"items", "items.product"})
List<Order> findAllWithItems();
```

Tespit: `@OneToMany(fetch=LAZY)` olan ilişkiyi → döngü içinde erişen service metodunu ara.

#### Sayfalama Eksikliği

```java
// ⚠️ RİSK: Tüm tabloyu belleğe yükler
List<Order> findByStatus(OrderStatus status); // Pageable yok!

// ✅ ÇÖZÜM:
Page<Order> findByStatus(OrderStatus status, Pageable pageable);
```

Tespit: `List<Entity>` döndüren repository metodları + bunları `findAll()` şeklinde çağıran servisler.

#### @Transactional İçinde Harici HTTP

```java
@Transactional
public void processPayment(dto) {
    // ⚠️ DB bağlantısı açık kalırken HTTP timeout bekleniyor
    BankResponse r = bankClient.charge(dto.amount); // timeout: 30s
    paymentRepo.save(payment);
}
```

Tespit: `@Transactional` metodun body'sinde `FeignClient` / `RestTemplate` / `WebClient` çağrısı.

#### Cache Anti-Pattern

```java
@Cacheable("users")
@Transactional  // ⚠️ Proxy bypass: cache çalışmaz, her seferinde DB'ye gider
public UserDto getUserById(Long id) { ... }

// Aynı sınıftaki başka metod çağrısı:
this.getUserById(id); // ⚠️ Self-invocation: cache bypass!
```

#### Scheduled Job + Full Table Scan

```java
@Scheduled(cron = "0 0 * * * *")  // Her saat
public void processExpiredOrders() {
    List<Order> all = orderRepo.findByStatus(PENDING); // ⚠️ Index yok mu?
}
```

**KB'de `performance_riskleri` alanına yaz:**
```json
"performance_riskleri": [
  {
    "tip": "N+1_SORGU",
    "dosya": "OrderService.java:87",
    "aciklama": "Order.items LAZY, döngü içinde erişiliyor",
    "risk": "yuksek",
    "cozum": "@EntityGraph veya JOIN FETCH ekle"
  },
  {
    "tip": "TRANSACTIONAL_ICINDE_HTTP",
    "dosya": "PaymentService.java:134",
    "aciklama": "BankClient.charge() @Transactional scope içinde, timeout: 30s",
    "risk": "kritik",
    "cozum": "HTTP çağrısını transaction dışına al veya @Async ile ayır"
  }
]
```

---

### 3.15 Security Risk Scan

> 3.4'teki güvenlik haritasına ek olarak kod seviyesinde güvenlik açıklarını tara.

#### SQL Injection (Native Query'lerde)

```java
// ⚠️ KRİTİK: String concatenation ile native query
@Query(value = "SELECT * FROM users WHERE name = '" + name + "'", nativeQuery = true)

// ✅ GÜVENLİ: Named parameter
@Query(value = "SELECT * FROM users WHERE name = :name", nativeQuery = true)
void findByName(@Param("name") String name);
```

Tespit: `nativeQuery=true` olan sorgularda string concatenation (`+` operatörü).

#### Eksik Input Validation

```java
// ⚠️ RİSK: @Valid yok — bean validation çalışmaz
@PostMapping("/users")
public ResponseEntity<UserDto> create(@RequestBody CreateUserDto dto) { ... }

// ✅ DOĞRU:
public ResponseEntity<UserDto> create(@Valid @RequestBody CreateUserDto dto) { ... }
```

Tespit: `@RequestBody` olan parametrelerde `@Valid` veya `@Validated` eksikliği.

#### Actuator Endpoint Açığı

```yaml
# ⚠️ RİSK: Tüm actuator endpoint'leri açık
management.endpoints.web.exposure.include: "*"
# SecurityConfig'de "/actuator/**" → permitAll() ise KRİTİK
```

#### CORS Wildcard

```java
// ⚠️ RİSK: Tüm origin'lere izin
@CrossOrigin("*")
// veya
.cors().configurationSource(r -> { config.setAllowedOrigins(List.of("*")); })
```

#### Mass Assignment Riski

```java
// ⚠️ RİSK: Entity doğrudan @RequestBody'den geliyor
@PostMapping
public void create(@RequestBody User user) { // Entity, DTO değil!
    userRepo.save(user);
}
```

Tespit: Controller'larda `@RequestBody @Entity` tipinde parametre.

#### Hardcoded Secret

```java
private static final String SECRET = "mysecretkey123"; // ⚠️ KRİTİK
String token = Jwts.builder().signWith(Keys.hmacShaKeyFor(SECRET.getBytes()))
```

**KB'de `guvenlik_riskleri` alanına yaz:**
```json
"guvenlik_riskleri": [
  {
    "tip": "SQL_INJECTION",
    "dosya": "UserRepository.java:45",
    "aciklama": "nativeQuery içinde string concatenation",
    "risk": "kritik",
    "cve_kategori": "OWASP A03:2021"
  },
  {
    "tip": "EKSIK_VALIDATION",
    "dosya": "OrderController.java:23",
    "aciklama": "@RequestBody'de @Valid eksik",
    "risk": "orta"
  }
]
```

---

### 3.16 Config Drift Detection

> Farklı ortam konfigürasyonları arasındaki farkları bul. "Prod'da neden çalışmıyor?" sorularının %60'ının cevabı burada.

#### Ortam Dosyalarını Karşılaştır

Şu dosyaları oku ve karşılaştır:
- `application.yml` (default)
- `application-dev.yml`
- `application-staging.yml`
- `application-prod.yml`
- `application-test.yml`

#### Drift Kategorileri

**1. Prod'da var, default'ta yok (gizli prod davranışı):**
```yaml
# application-prod.yml'de var, default'ta yok:
spring.jpa.hibernate.ddl-auto: none  # Dev'de validate, prod'da none → farklı davranış!
feign.client.config.default.readTimeout: 60000  # Dev'de 10s, prod'da 60s
```

**2. Default'ta var, prod'ta yok (prod'da NPE riski):**
```yaml
# application.yml'de var ama application-prod.yml'de override edilmemiş:
app.feature.new-checkout: true  # Prod'da da true mu? Bilinmiyor!
```

**3. Kritik değer farkları:**
```yaml
# Dev:  spring.datasource.hikari.maximum-pool-size: 5
# Prod: spring.datasource.hikari.maximum-pool-size: 50  # 10x fark!
```

**KB'de `config_drift` alanına yaz:**
```json
"config_drift": {
  "ortamlar_karsilastirildi": ["default", "dev", "prod"],
  "sadece_prod_da_var": [
    { "key": "spring.jpa.hibernate.ddl-auto", "deger": "none", "risk": "migration zorunlu olduğunu gösterir" }
  ],
  "deger_farklari": [
    { "key": "hikari.maximum-pool-size", "dev": "5", "prod": "50", "risk": "dusuk" },
    { "key": "feign.readTimeout", "dev": "10000", "prod": "60000", "risk": "orta — harici timeout analizini etkiler" }
  ],
  "bilinmeyen_prod_davranislari": [
    "app.feature.new-checkout prod'da tanımsız — Spring default mi kullanıyor?"
  ]
}
```

---

## REPO KOMPLEKSİTE SKORU

> KB'ye meta bilgi olarak ekle. Mimari risk ve bakım maliyetini tek sayıda özetler.

### Hesaplama

| Faktör | Metrik | Puan |
|---|---|---|
| Servis sayısı | 1-5: 0p / 6-15: 2p / 16+: 4p | |
| Servis-servis bağımlılık kenar sayısı | per 5 kenar: +1p | |
| Kritik merkez servis sayısı | per merkez: +2p | |
| Harici HTTP entegrasyonu | per client: +1p | |
| Kuyruk consumer + producer | per 5 adet: +1p | |
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
  "toplam": 8.5,
  "seviye": "ORTA",
  "kirilim": {
    "servis_sayisi": 2,
    "bagimlilik_kenarlari": 3,
    "kritik_merkez_servisler": 2,
    "harici_entegrasyon": 1,
    "test_boslugu": 1
  },
  "oneri": "UserService ve PaymentService değişikliklerinde geniş regresyon testi yap"
}
```

---

## KB BOYUT KORUMASI

> JSON çok büyük olursa agent'ın ileride okuma maliyeti artar. Şu kuralları uygula.

### Boyut Hedefleri

| Durum | Hedef |
|---|---|
| Küçük repo (1-3 servis) | < 200KB JSON |
| Orta repo (4-10 servis) | < 400KB JSON |
| Büyük repo (11+ servis) | < 600KB JSON; monorepo ise modül başına ayrı JSON |

### Sıkıştırma Stratejisi (büyük repolar için)

**DTO'lar:** 20+ alan olan DTO'larda sadece ilk 10 alanı yaz, geri kalanı için `"devami_var": true, "dosya": "path/to/Dto.java"` ekle.

**Servis metodları:** 15+ metod olan servislerde sadece annotation'lı metodları tam yaz; annotation'sız olanları yalnızca imzayla listele.

**Repository sorguları:** Native SQL'i 200 karakterden uzunsa kes, `"tam_sorgu_icin": "dosya:satir"` ekle.

**Teknik borç:** Düşük risk FIXME'leri sadece sayısını yaz: `"dusuk_risk_toplam": 14, "ornekler": [ilk 3 tanesi]`

### Monorepo'da Ayrı JSON Stratejisi

```
kb/
├── _index.json          ← modüller + kısa açıklamalar + bağımlılık matrisi
├── order-service.json   ← order modülünün tam KB'si
├── user-service.json    ← user modülünün tam KB'si
└── common.json          ← shared entity/DTO'lar (tüm modüller referans eder)
```

`_index.json` içeriği:
```json
{
  "modueller": [
    { "isim": "order-service", "dosya": "kb/order-service.json", "ozet": "Sipariş yönetimi", "kritiklik": "YUKSEK" }
  ],
  "modul_bagimlilik_matrisi": { ... },
  "global_kompleksite_skoru": 11.5
}
```

---

## Java'ya Özel: Yeni Özellik Eklerken

### Tipik Dosya Sırası

```
1. src/main/java/.../domain/entity/     → Entity ekle/güncelle
2. src/main/resources/db/migration/     → Flyway migration yaz (ddl-auto:none ise zorunlu!)
3. src/main/java/.../dto/               → Request ve Response DTO'ları
4. src/main/java/.../repository/        → Repository interface metodları
5. src/main/java/.../service/           → Service metodu + @Transactional
6. src/main/java/.../controller/        → Controller action + @RequestMapping
7. src/main/java/.../consumer|producer/ → Kuyruk gerekliyse ekle
8. SecurityConfig.java                  → permitAll gerekiyorsa ekle
9. src/test/java/.../                   → Unit ve integration test yaz
```

### Java'ya Özel Dikkat Edilecekler

- **ddl-auto: none ise** schema OTOMATİK güncellenmez — Flyway/Liquibase migration zorunlu
- **@Transactional sınırı:** Async metodlar (`@Async`) transaction'ın dışına çıkar — dikkat!
- **Lazy loading tuzağı:** `@OneToMany(fetch=LAZY)` → transaction dışında `LazyInitializationException`
- **N+1 sorgu riski:** `@OneToMany` ilişkilerde `JOIN FETCH` veya `@EntityGraph` kullan
- **Hibernate XML mapping:** Entity değiştirince `*.hbm.xml` eşleme dosyasını güncelle
- **ClassGraph ile classpath scanning:** Yeni handler base class'tan extend edilmeli
- **@Cacheable + @Transactional:** Aynı class içinde çağrı → proxy bypass eder, cache çalışmaz
- **RabbitMQ prefetch:** Yüksek prefetch = memory baskısı, düşük = throughput kaybı
- **Bean validation ile @Valid:** Controller'da `@Valid @RequestBody` yoksa validation çalışmaz
- **Multi-tenant:** TenantId header'dan alınıyorsa security filter'da set edilmeli
- **Spring Security CSRF:** Stateless JWT API'lerde `csrf().disable()` olmalı
- **@Async + thread pool:** `taskExecutor` bean'i configure edilmemişse default pool küçük kalır
- **@Transactional içinde harici HTTP:** DB bağlantısı meşgul kalır, connection pool tükenir
- **Config drift:** Prod'a deploy etmeden önce application-prod.yml ile default'u karşılaştır
- **Monorepo'da `common` değişikliği:** Tüm downstream modülleri build + test et
