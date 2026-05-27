| name | test-strategy |
| --- | --- |
| description | xUnit 기반 단위/통합 테스트 전략 가이드. Given-When-Then 패턴, Moq Mock 사용, WebApplicationFactory + Testcontainers 통합 테스트, 실패 분석 방법 포함. |
| allowed-tools | Bash |

# Test Strategy Guide

## 테스트 범위 결정

| 범위 | 명령 |
| --- | --- |
| 특정 테스트 클래스 | `dotnet test --filter "FullyQualifiedName~GachaServiceTest"` |
| 특정 도메인 | `dotnet test --filter "Category=gacha"` |
| 전체 | `dotnet test` |

실패 상세 출력:
```
dotnet test --logger "console;verbosity=detailed"
```

---

## 단위 테스트

대상: Service, 도메인 메서드, 유효성 검증 로직

### 기본 구조 (Given-When-Then)

```csharp
public class GachaServiceTest
{
    private readonly Mock<IGachaRepository> _gachaRepoMock = new();
    private readonly Mock<ICurrencyRepository> _currencyRepoMock = new();
    private readonly GachaService _sut; // System Under Test

    public GachaServiceTest()
    {
        _sut = new GachaService(_gachaRepoMock.Object, _currencyRepoMock.Object);
    }

    [Fact]
    public async Task DrawAsync_WhenCurrencyInsufficient_ThrowsGameException()
    {
        // Given
        var playerId = 1L;
        var currency = Currency.Create(playerId, 0); // 재화 없음
        _currencyRepoMock.Setup(r => r.GetByPlayerIdAsync(playerId))
            .ReturnsAsync(currency);

        var request = new DrawGachaRequest { DrawCount = 1 };

        // When
        var act = async () => await _sut.DrawAsync(playerId, request);

        // Then
        var exception = await Assert.ThrowsAsync<GameException>(act);
        Assert.Equal(HttpStatusCode.BadRequest, exception.StatusCode);
        Assert.Equal("뽑기 재화가 부족합니다.", exception.Message);
    }

    [Fact]
    public async Task DrawAsync_WhenCurrencySufficient_ReturnsDraw()
    {
        // Given
        var playerId = 1L;
        var currency = Currency.Create(playerId, 300);
        _currencyRepoMock.Setup(r => r.GetByPlayerIdAsync(playerId))
            .ReturnsAsync(currency);
        _gachaRepoMock.Setup(r => r.AddDrawAsync(It.IsAny<GachaDraw>()))
            .Returns(Task.CompletedTask);

        var request = new DrawGachaRequest { DrawCount = 1 };

        // When
        var result = await _sut.DrawAsync(playerId, request);

        // Then
        Assert.NotNull(result);
        _gachaRepoMock.Verify(r => r.AddDrawAsync(It.IsAny<GachaDraw>()), Times.Once);
    }
}
```

### 메서드명 규칙

`{MethodName}_{Scenario}_{ExpectedResult}`

```
DrawAsync_WhenCurrencyInsufficient_ThrowsGameException
DrawAsync_WhenCurrencySufficient_ReturnsDraw
LoginAsync_WhenPasswordWrong_ThrowsGameException
SaveRecordAsync_WhenNewBest_UpdatesBestRecord
```

### 도메인 메서드 단위 테스트

```csharp
public class CurrencyTest
{
    [Theory]
    [InlineData(300, 100, 200)] // 정상 소모
    [InlineData(100, 100, 0)]   // 정확히 맞음
    public void Consume_WhenAmountValid_DeductsCorrectly(int initial, int consume, int expected)
    {
        // Given
        var currency = Currency.Create(1L, initial);

        // When
        currency.Consume(consume);

        // Then
        Assert.Equal(expected, currency.Amount);
    }

    [Fact]
    public void Consume_WhenInsufficientFunds_ThrowsGameException()
    {
        // Given
        var currency = Currency.Create(1L, 50);

        // When / Then
        Assert.Throws<GameException>(() => currency.Consume(100));
    }
}
```

---

## 통합 테스트

대상: Controller, 인증 흐름, DB 연동

### 설정

```csharp
// Tests/Integration/GameWebApplicationFactory.cs
public class GameWebApplicationFactory : WebApplicationFactory<Program>
{
    private readonly PostgreSqlContainer _dbContainer = new PostgreSqlBuilder()
        .WithDatabase("game_test")
        .WithUsername("postgres")
        .WithPassword("postgres")
        .Build();

    public async Task InitializeAsync() => await _dbContainer.StartAsync();
    public new async Task DisposeAsync() => await _dbContainer.DisposeAsync();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // 실제 DB 연결 제거
            var descriptor = services.SingleOrDefault(d =>
                d.ServiceType == typeof(DbContextOptions<AppDbContext>));
            if (descriptor != null) services.Remove(descriptor);

            // Testcontainers PostgreSQL 주입
            services.AddDbContext<AppDbContext>(opt =>
                opt.UseNpgsql(_dbContainer.GetConnectionString()));
        });
    }
}
```

### 인증 통합 테스트 예시

```csharp
public class AuthControllerTest : IClassFixture<GameWebApplicationFactory>
{
    private readonly HttpClient _client;

    public AuthControllerTest(GameWebApplicationFactory factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task Login_WhenCredentialsValid_ReturnsTokens()
    {
        // Given
        var request = new LoginRequest { Username = "testuser", Password = "Test1234!" };

        // When
        var response = await _client.PostAsJsonAsync("/api/v1/auth/login", request);

        // Then
        response.EnsureSuccessStatusCode();
        var result = await response.Content.ReadFromJsonAsync<LoginResponse>();
        Assert.NotNull(result!.AccessToken);
        Assert.NotNull(result.RefreshToken);
    }

    [Fact]
    public async Task GetCurrency_WhenUnauthorized_Returns401()
    {
        // Given — 토큰 없음

        // When
        var response = await _client.GetAsync("/api/v1/currency");

        // Then
        Assert.Equal(HttpStatusCode.Unauthorized, response.StatusCode);
    }
}
```

---

## 결과 분석

실패 시 확인 항목:

- 테스트명 및 클래스
- 실패 메시지와 근본 원인
- 관련 스택 트레이스 핵심 라인

실패 발생 시 관련 소스 파일을 읽고 가장 가능성 높은 수정 방안 제시.
