| name | aspnet-game-arch |
| --- | --- |
| description | ASP.NET Core 10 게임 서버 레이어 아키텍처 가이드 — Controller/Service/Repository 책임 분리, 의존성 주입 패턴, GameException 사용 규칙, Entity↔DTO 변환 패턴. |

# ASP.NET Core Game Server Architecture

## 레이어 책임

### Controller

- HTTP 요청 수신, 입력 유효성 검증, 응답 반환만 담당
- 비즈니스 로직 절대 금지
- 라우팅: `[ApiController]` + `[Route("api/v1/[controller]")]`
- 응답: DTO를 직접 반환 — 래퍼 객체 금지

```csharp
[ApiController]
[Route("api/v1/[controller]")]
public class GachaController : ControllerBase
{
    private readonly IGachaService _gachaService;
    public GachaController(IGachaService gachaService) => _gachaService = gachaService;

    [HttpPost("draw")]
    [Authorize]
    [ProducesResponseType(typeof(DrawGachaResponse), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status400BadRequest)]
    public async Task<DrawGachaResponse> Draw([FromBody] DrawGachaRequest request)
    {
        var playerId = long.Parse(User.FindFirstValue(ClaimTypes.NameIdentifier)!);
        return await _gachaService.DrawAsync(playerId, request);
    }
}
```

### Service

- 비즈니스 로직 전담
- Interface + Implementation 패턴 필수: `IGachaService` / `GachaService`
- 의존성: 생성자 주입만 사용
- `DbContext` 직접 주입 금지 — Repository를 통해 접근

```csharp
public interface IGachaService
{
    Task<DrawGachaResponse> DrawAsync(long playerId, DrawGachaRequest request);
}

public class GachaService : IGachaService
{
    private readonly IGachaRepository _gachaRepository;
    private readonly ICurrencyRepository _currencyRepository;

    public GachaService(IGachaRepository gachaRepository, ICurrencyRepository currencyRepository)
    {
        _gachaRepository = gachaRepository;
        _currencyRepository = currencyRepository;
    }

    public async Task<DrawGachaResponse> DrawAsync(long playerId, DrawGachaRequest request)
    {
        var currency = await _currencyRepository.GetByPlayerIdAsync(playerId)
            ?? throw new GameException("재화 정보를 찾을 수 없습니다.", HttpStatusCode.NotFound);

        if (currency.Amount < request.DrawCount * CostPerDraw)
            throw new GameException("뽑기 재화가 부족합니다.", HttpStatusCode.BadRequest);

        // 이하 비즈니스 로직
    }
}
```

### Repository

- EF Core를 통한 DB 접근만 담당
- Interface + Implementation 패턴 필수
- 읽기 전용 쿼리: 반드시 `.AsNoTracking()` 적용
- 복잡한 조회: LINQ 사용, N+1 방지를 위해 `Include()` 적용
- 비즈니스 로직 금지

```csharp
public interface IGachaRepository
{
    Task<List<GachaDraw>> GetDrawsByPlayerIdAsync(long playerId);
    Task AddDrawAsync(GachaDraw draw);
}

public class GachaRepository : IGachaRepository
{
    private readonly AppDbContext _context;
    public GachaRepository(AppDbContext context) => _context = context;

    public async Task<List<GachaDraw>> GetDrawsByPlayerIdAsync(long playerId)
        => await _context.GachaDraws
            .AsNoTracking()
            .Where(d => d.PlayerId == playerId)
            .Include(d => d.Character)
            .ToListAsync();

    public async Task AddDrawAsync(GachaDraw draw)
    {
        _context.GachaDraws.Add(draw);
        await _context.SaveChangesAsync();
    }
}
```

---

## DI 등록 패턴

`Infrastructure/Extensions/ServiceCollectionExtensions.cs` 에서 일괄 등록:

```csharp
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddGameServices(this IServiceCollection services)
    {
        services.AddScoped<IPlayerService, PlayerService>();
        services.AddScoped<IGachaService, GachaService>();
        services.AddScoped<IAuthService, AuthService>();
        services.AddScoped<ICurrencyService, CurrencyService>();
        return services;
    }

    public static IServiceCollection AddGameRepositories(this IServiceCollection services)
    {
        services.AddScoped<IPlayerRepository, PlayerRepository>();
        services.AddScoped<IGachaRepository, GachaRepository>();
        services.AddScoped<ICurrencyRepository, CurrencyRepository>();
        services.AddScoped<IRefreshTokenRepository, RefreshTokenRepository>();
        return services;
    }
}
```

---

## Entity 설계 원칙

- 외부에서 setter 호출 불가: `private set`
- 상태 변경은 도메인 메서드로 캡슐화
- EF Core 리플렉션용 protected/private 생성자 필수

```csharp
public class Player
{
    public long Id { get; private set; }
    public string Username { get; private set; } = null!;
    public string PasswordHash { get; private set; } = null!;
    public DateTime CreatedAt { get; private set; }

    // EF Core 전용 생성자
    private Player() { }

    public static Player Create(string username, string passwordHash) => new()
    {
        Username = username,
        PasswordHash = passwordHash,
        CreatedAt = DateTime.UtcNow
    };
}
```

---

## DTO 변환 패턴

확장 메서드(Extension Method)로 변환 로직 분리:

```csharp
// Domain/DTOs/PlayerExtensions.cs
public static class PlayerExtensions
{
    public static PlayerResponse ToResponse(this Player player) => new()
    {
        Id = player.Id,
        Username = player.Username,
        CreatedAt = player.CreatedAt
    };

    public static Player ToEntity(this RegisterRequest request) => Player.Create(
        request.Username,
        BCrypt.Net.BCrypt.HashPassword(request.Password)
    );
}
```

---

## GameException 규칙

- `GameException` 직접 사용 — 서브클래스 생성 **금지**
- `ExceptionHandlingMiddleware` 가 전역 처리 → Problem Details 응답 반환

```csharp
// 올바른 사용
var player = await _playerRepository.GetByIdAsync(id)
    ?? throw new GameException("플레이어를 찾을 수 없습니다.", HttpStatusCode.NotFound);

// 금지: 서브클래스 생성
public class PlayerNotFoundException : GameException { } // ❌
```

```csharp
// ExceptionHandlingMiddleware.cs
public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;

    public ExceptionHandlingMiddleware(RequestDelegate next, ILogger<ExceptionHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (GameException ex)
        {
            _logger.LogWarning("Game exception: {Message}", ex.Message);
            context.Response.StatusCode = (int)ex.StatusCode;
            await context.Response.WriteAsJsonAsync(new ProblemDetails
            {
                Title = ex.StatusCode.ToString(),
                Status = (int)ex.StatusCode,
                Detail = ex.Message
            });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception occurred");
            context.Response.StatusCode = 500;
            await context.Response.WriteAsJsonAsync(new ProblemDetails
            {
                Title = "Internal Server Error",
                Status = 500,
                Detail = "서버 내부 오류가 발생했습니다."
            });
        }
    }
}
```
