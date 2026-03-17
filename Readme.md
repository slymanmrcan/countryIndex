# CountryIndex API — Mimari & Tasarım Kılavuzu

## Proje Amacı
SQL Server üzerinde çalışan, ülke/şehir/ilçe verisi sunan REST API.
Kullanıcılar kayıt olur, API key alır, bu key ile istekleri yapar.

---

## Klasör Yapısı

```
CountryIndex.Api/
│
├── Controllers/                # HTTP katmanı — sadece istek al, cevap ver
│   ├── ApiControllerBase.cs    # abstract base — HandleResult, HandlePagedResult
│   ├── AuthController.cs
│   ├── CountriesController.cs
│   └── LocationsController.cs
│
├── Middleware/
│   ├── ExceptionHandlingMiddleware.cs   # Global hata yakalayıcı
│   ├── ApiKeyMiddleware.cs              # Her istekte API key doğrula
│   └── RateLimitMiddleware.cs          # İstek sınırlama
│
├── Common/
│   ├── Result.cs             # Result pattern
│   ├── ApiResponse.cs        # Response wrapper
│   ├── PagedList.cs          # Pagination
│   └── Error.cs              # Hata sabitleri
│
├── Domain/
│   └── Entities/
│       ├── AppUser.cs
│       ├── ApiKey.cs
│       ├── Country.cs
│       ├── City.cs
│       └── District.cs
│
├── Repositories/
│   ├── IRepository.cs              # Generic interface
│   ├── GenericRepository.cs        # Generic implementation
│   ├── ICountryRepository.cs
│   ├── CountryRepository.cs
│   ├── ICityRepository.cs
│   └── CityRepository.cs
│
├── Services/
│   ├── ICountryService.cs
│   ├── CountryService.cs
│   ├── ILocationService.cs
│   ├── LocationService.cs
│   ├── IAuthService.cs
│   └── AuthService.cs
│
├── Persistence/
│   ├── AppDbContext.cs
│   └── Configurations/             # EF Fluent API config
│       ├── CountryConfiguration.cs
│       ├── CityConfiguration.cs
│       └── ApiKeyConfiguration.cs
│
├── Validators/                     # FluentValidation
│   └── RegisterRequestValidator.cs
│
├── Dtos/                           # Request/Response modelleri
│   ├── Auth/
│   │   ├── RegisterRequest.cs
│   │   ├── LoginRequest.cs
│   │   └── LoginResponse.cs
│   ├── Country/
│   │   ├── CountryResponse.cs
│   │   └── CountryDetailResponse.cs
│   └── Location/
│       ├── CityResponse.cs
│       └── DistrictResponse.cs
│
├── appsettings.json
└── Program.cs
```

---

## Katmanlar ve Sorumluluklar

### 1. Controller
**Ne yapar:** HTTP isteğini alır, service'e gönderir, cevabı döner.
**Ne YAPMAZ:** Hiçbir iş mantığı yok, DB'ye dokunmaz.

```
GET /api/countries?page=1&pageSize=20
    ↓
CountriesController.GetAll()
    ↓
_countryService.GetAllAsync(query)
    ↓
Result<PagedList<CountryResponse>> döner
    ↓
200 OK veya 404 NotFound
```

### 2. Service
**Ne yapar:** İş mantığı burada. Cache kontrol eder, repository'yi çağırır, mapping yapar.
**Ne YAPMAZ:** HTTP'den haberi yok, SqlCommand yazmaz.

```csharp
public async Task<Result<PagedList<CountryResponse>>> GetAllAsync(GetCountriesQuery query)
{
    // 1. Cache'e bak
    var cached = await _cache.GetAsync<PagedList<CountryResponse>>(cacheKey);
    if (cached is not null) return Result<T>.Success(cached);

    // 2. Repository'den al
    var countries = await _repo.GetPagedAsync(query.Page, query.PageSize);

    // 3. Map et (Entity → DTO)
    var response = countries.Map(c => c.ToResponse());

    // 4. Cache'e yaz
    await _cache.SetAsync(cacheKey, response, TimeSpan.FromMinutes(30));

    return Result<T>.Success(response);
}
```

### 3. Repository
**Ne yapar:** Sadece veri okuma/yazma. EF Core sorguları burada.
**Ne YAPMAZ:** İş mantığı yok, cache yok, mapping yok.

```csharp
public async Task<PagedList<Country>> GetPagedAsync(int page, int size, string? search)
{
    var query = _context.Countries
        .Where(c => c.IsEnabled && !c.IsHidden);

    if (!string.IsNullOrEmpty(search))
        query = query.Where(c => c.NameTr.Contains(search) || c.NameEn.Contains(search));

    var total = await query.CountAsync();
    var items = await query.Skip((page - 1) * size).Take(size).ToListAsync();

    return new PagedList<Country>(items, total, page, size);
}
```

---

## CQRS Nedir, Monolit'te Mantıklı mı?

**CQRS** = Command Query Responsibility Segregation
Yani okuma (Query) ve yazma (Command) operasyonlarını ayır.

**MediatR ile tam CQRS:**
```
Controller → IMediator.Send(query) → Handler
```

**Service layer ile basit CQRS:**
```
Controller → ICountryService.GetAsync() → Repository
Controller → ICountryService.CreateAsync() → Repository
```

### Monolit'te ne kullanalım?

**Service layer yeterli** — MediatR over-engineering olur.
Ama CV için MediatR bildiğini göstermek istiyorsan:
→ Sadece Countries için MediatR kullan
→ Locations için düz service yaz
→ README'de "bilinçli tercih" olarak açıkla

Bu daha akıllıca görünür — "her şeye MediatR attı" değil,
"nerede gerekli nerede değil biliyor" izlenimi verir.

---

## Result Pattern

Neden? Exception fırlatmak yerine başarı/hata durumunu tip güvenli taşımak için.

```csharp
// Common/Result.cs
public class Result<T>
{
    public bool IsSuccess { get; }
    public T? Value { get; }
    public Error Error { get; }

    private Result(T value)        { IsSuccess = true;  Value = value; Error = Error.None; }
    private Result(Error error)    { IsSuccess = false; Error = error; }

    public static Result<T> Success(T value)   => new(value);
    public static Result<T> Failure(Error error) => new(error);
}

// Common/Error.cs
public record Error(string Code, string Message)
{
    public static readonly Error None        = new(string.Empty, string.Empty);
    public static readonly Error NotFound    = new("NOT_FOUND", "Kayıt bulunamadı.");
    public static readonly Error Unauthorized = new("UNAUTHORIZED", "Yetkisiz erişim.");

    public static Error Validation(string msg) => new("VALIDATION", msg);
    public static Error Custom(string code, string msg) => new(code, msg);
}
```

**Controller'da kullanım:**
```csharp
var result = await _service.GetByIso2Async("TR");

return result.IsSuccess
    ? Ok(ApiResponse<CountryResponse>.Ok(result.Value!))
    : NotFound(ApiResponse<CountryResponse>.Fail(result.Error.Message));
```

---

## Response Wrapper

Tüm response'lar aynı formatta döner:

```json
{
  "success": true,
  "data": { ... },
  "message": null,
  "errors": null,
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "totalCount": 250,
    "totalPages": 13
  }
}
```

```csharp
// Common/ApiResponse.cs
public class ApiResponse<T>
{
    public bool Success { get; init; }
    public T? Data { get; init; }
    public string? Message { get; init; }
    public List<string>? Errors { get; init; }
    public PaginationMeta? Pagination { get; init; }

    public static ApiResponse<T> Ok(T data, string? msg = null) =>
        new() { Success = true, Data = data, Message = msg };

    public static ApiResponse<T> Fail(string error) =>
        new() { Success = false, Errors = [error] };

    public static ApiResponse<T> Fail(List<string> errors) =>
        new() { Success = false, Errors = errors };

    public static ApiResponse<T> Paged(T data, PaginationMeta meta) =>
        new() { Success = true, Data = data, Pagination = meta };
}

public record PaginationMeta(int Page, int PageSize, int TotalCount, int TotalPages);
```

---

## Auth Sistemi

### Akış
```
1. POST /api/auth/register  → kullanıcı oluştur
2. POST /api/auth/login     → JWT token döner
3. GET  /api/auth/api-key   → API key görüntüle  [JWT gerekir]
4. POST /api/auth/api-key/rotate → yeni key üret [JWT gerekir]
5. POST /api/auth/api-key/revoke → key iptal      [JWT gerekir]

Sonra:
GET /api/countries  →  X-Api-Key: cki_xxxx  header ile
```

### Neden hem JWT hem API key?

- **JWT** → kullanıcı yönetimi için (kim olduğunu bilmek)
- **API key** → dışarıya açık istekler için (uygulamalar kullanır)

### Entity'ler

```csharp
// Domain/Entities/AppUser.cs
public class AppUser
{
    public int Id { get; set; }
    public string Email { get; set; }
    public string PasswordHash { get; set; }
    public string Role { get; set; } = "User";  // "User" | "Admin"
    public DateTime CreatedAt { get; set; }
    public ApiKey? ApiKey { get; set; }
}

// Domain/Entities/ApiKey.cs
public class ApiKey
{
    public int Id { get; set; }
    public int UserId { get; set; }
    public string KeyHash { get; set; }      // SHA256 hash — DB'de düz saklanmaz
    public string KeyPrefix { get; set; }    // "cki_xxxx" — göstermek için
    public bool IsActive { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? LastUsedAt { get; set; }
    public AppUser User { get; set; }
}
```

### API Key Format
```
cki_7b9d7c46a5c5b067413266a33405d0c57852f6a304ce8d37a4d5d62ac059e73a
 ↑       ↑
prefix  random 64 hex char (32 byte)
```

DB'de sadece SHA256(key) saklanır. Kullanıcı kaybedirse rotate etmeli.

---

## Middleware'ler

### 1. ApiKeyMiddleware
```
Her istek gelir
    ↓
X-Api-Key header var mı?
    ↓ Yok → 401 Unauthorized
    ↓ Var
SHA256 hash al → DB'de ara
    ↓ Bulunamadı → 401
    ↓ Bulundu ama IsActive=false → 401
    ↓ Bulundu
LastUsedAt güncelle → isteği geçir
```

**Public endpoint'ler** (/api/auth/register, /api/auth/login) bu middleware'i atlar.

### 2. RateLimitMiddleware
```
İstek gelir
    ↓
API key'e göre Redis'te say
Key: "ratelimit:{apiKeyPrefix}:{dakika}"
    ↓
Sayı > limit (örn: 60/dk) → 429 Too Many Requests
    ↓
Değilse isteği geçir, sayacı artır
```

Redis key TTL = 60 saniye → otomatik sıfırlanır.

### 3. ExceptionHandlingMiddleware
```csharp
try { await _next(context); }
catch (ValidationException ex)  → 422 Unprocessable Entity
catch (UnauthorizedException ex) → 401
catch (NotFoundException ex)     → 404
catch (Exception ex)             → 500 + log
```

---

## Pagination

```csharp
// Common/PagedList.cs
public class PagedList<T>
{
    public List<T> Items { get; }
    public int TotalCount { get; }
    public int Page { get; }
    public int PageSize { get; }
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasPrevious => Page > 1;
    public bool HasNext => Page < TotalPages;

    public PagedList(List<T> items, int totalCount, int page, int pageSize)
    {
        Items = items;
        TotalCount = totalCount;
        Page = page;
        PageSize = pageSize;
    }

    // Map helper — entity → dto dönüşümü için
    public PagedList<TResult> Map<TResult>(Func<T, TResult> mapper) =>
        new(Items.Select(mapper).ToList(), TotalCount, Page, PageSize);
}
```

---

## Caching (Redis)

```csharp
// ICacheService.cs
public interface ICacheService
{
    Task<T?> GetAsync<T>(string key, CancellationToken ct = default);
    Task SetAsync<T>(string key, T value, TimeSpan? expiry = null, CancellationToken ct = default);
    Task RemoveAsync(string key, CancellationToken ct = default);
    Task RemoveByPrefixAsync(string prefix, CancellationToken ct = default);
}
```

**Cache key stratejisi:**
```
countries:all:page:1:size:20          → 30 dk
countries:iso2:TR                     → 1 saat
locations:country:1:cities            → 1 saat
ratelimit:cki_xxxx:2024010112         → 60 sn (TTL)
```

---

## FluentValidation

```csharp
// Validators/RegisterRequestValidator.cs
public class RegisterRequestValidator : AbstractValidator<RegisterRequest>
{
    public RegisterRequestValidator()
    {
        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("Email boş olamaz.")
            .EmailAddress().WithMessage("Geçersiz email formatı.");

        RuleFor(x => x.Password)
            .NotEmpty()
            .MinimumLength(8).WithMessage("Şifre en az 8 karakter olmalı.")
            .Matches("[A-Z]").WithMessage("En az bir büyük harf gerekli.")
            .Matches("[0-9]").WithMessage("En az bir rakam gerekli.");
    }
}
```

Controller'a dokunmana gerek yok — middleware otomatik çalışır.

---

## Program.cs (Özet)

```csharp
var builder = WebApplication.CreateBuilder(args);

// Controllers
builder.Services.AddControllers();

// EF Core
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

// Redis
builder.Services.AddStackExchangeRedisCache(opt =>
    opt.Configuration = builder.Configuration.GetConnectionString("Redis"));
builder.Services.AddScoped<ICacheService, RedisCacheService>();

// Repositories
builder.Services.AddScoped(typeof(IRepository<>), typeof(GenericRepository<>));
builder.Services.AddScoped<ICountryRepository, CountryRepository>();
builder.Services.AddScoped<ICityRepository, CityRepository>();

// Services
builder.Services.AddScoped<ICountryService, CountryService>();
builder.Services.AddScoped<ILocationService, LocationService>();
builder.Services.AddScoped<IAuthService, AuthService>();

// FluentValidation
builder.Services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());

// JWT
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(opt => { /* config */ });

// Swagger / Scalar
builder.Services.AddOpenApi();

var app = builder.Build();

// Middleware sırası ÖNEMLİ
app.UseMiddleware<ExceptionHandlingMiddleware>();  // 1. hataları yakala
app.UseMiddleware<ApiKeyMiddleware>();             // 2. key doğrula
app.UseMiddleware<RateLimitMiddleware>();          // 3. limit kontrol

app.MapOpenApi();
app.MapScalarApiReference();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.Run();
```

---

## NuGet Paketleri

```xml
<PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="9.0.0" />
<PackageReference Include="Microsoft.EntityFrameworkCore.Tools" Version="9.0.0" />
<PackageReference Include="Microsoft.AspNetCore.Authentication.JwtBearer" Version="9.0.0" />
<PackageReference Include="FluentValidation.AspNetCore" Version="11.3.0" />
<PackageReference Include="StackExchange.Redis" Version="2.8.16" />
<PackageReference Include="Microsoft.Extensions.Caching.StackExchangeRedis" Version="9.0.0" />
<PackageReference Include="Mapster" Version="7.4.0" />
<PackageReference Include="Scalar.AspNetCore" Version="2.0.0" />
<PackageReference Include="MediatR" Version="12.4.1" />  <!-- opsiyonel -->
```

---

## Yazma Sırası (Önerilen)

```
1. Domain/Entities        → Country, City, District, AppUser, ApiKey
2. Common/                → Result, Error, ApiResponse, PagedList
3. Persistence/           → AppDbContext, Configurations, migration
4. Repositories/          → Generic, Country, City
5. Services/Auth          → AuthService (register, login, api-key)
6. Middleware/            → ExceptionHandling, ApiKey, RateLimit
7. Services/Country       → CountryService
8. Controllers/Auth       → AuthController
9. Controllers/Countries  → CountriesController
10. Services/Location     → LocationService
11. Controllers/Locations → LocationsController
12. Validators/           → RegisterRequest, LoginRequest
```

---

## Sık Sorulan: "Neden bu pattern?"

| Pattern | Neden |
|---|---|
| Repository | DB değişirse sadece repository değişir. Test edilebilir. |
| Result<T> | Exception yerine tip güvenli hata yönetimi. |
| Service layer | Controller'ı temiz tutar. İş mantığı bir yerde. |
| FluentValidation | Validation logic ayrı dosyada, reusable. |
| Response wrapper | Frontend her zaman aynı format bekler. |
| Redis cache | Aynı veriyi defalarca DB'den çekmemek için. |
| ApiKeyMiddleware | Her controller'a [Authorize] yazmak yerine merkezi kontrol. |


---

## Base Controller

Controller'larda `result.IsSuccess ? Ok(...) : NotFound(...)` tekrarını önlemek için.
Tüm HTTP status kodu kararları burada, `Error.Code`'a göre verilir.

```csharp
// Controllers/ApiControllerBase.cs
[ApiController]
public abstract class ApiControllerBase : ControllerBase
{
    protected IActionResult HandleResult<T>(Result<T> result) =>
        result.IsSuccess
            ? Ok(ApiResponse<T>.Ok(result.Value!))
            : result.Error.Code switch
            {
                "NOT_FOUND"    => NotFound(ApiResponse<T>.Fail(result.Error.Message)),
                "UNAUTHORIZED" => Unauthorized(ApiResponse<T>.Fail(result.Error.Message)),
                "VALIDATION"   => UnprocessableEntity(ApiResponse<T>.Fail(result.Error.Message)),
                _              => BadRequest(ApiResponse<T>.Fail(result.Error.Message))
            };

    protected IActionResult HandlePagedResult<T>(Result<PagedList<T>> result) =>
        result.IsSuccess
            ? Ok(ApiResponse<T>.Paged(
                result.Value!.Items,
                new PaginationMeta(
                    result.Value.Page,
                    result.Value.PageSize,
                    result.Value.TotalCount,
                    result.Value.TotalPages)))
            : HandleResult(result);
}
```

Kullanım — her controller bu kadar temiz olur:

```csharp
public class CountriesController : ApiControllerBase
{
    [HttpGet]
    public async Task<IActionResult> GetAll([FromQuery] GetCountriesQuery query, CancellationToken ct)
        => HandlePagedResult(await _service.GetAllAsync(query, ct));

    [HttpGet("{iso2}")]
    public async Task<IActionResult> GetByIso2(string iso2, CancellationToken ct)
        => HandleResult(await _service.GetByIso2Async(iso2, ct));
}
```

Klasör yapısına ekle: `Controllers/ApiControllerBase.cs`
Yazma sırasına ekle: `Common/` bittikten hemen sonra yaz, `Result` ve `Error`'a bağımlı.