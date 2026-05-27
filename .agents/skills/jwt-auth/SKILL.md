| name | jwt-auth |
| --- | --- |
| description | JWT Access Token + Refresh Token 발급/검증/갱신/폐기 구현 가이드. BCrypt 비밀번호 처리, RefreshToken PostgreSQL 저장, Unity 클라이언트 통신 패턴 포함. |

# JWT Authentication Guide

## 토큰 구조

| 토큰 | 유효기간 | 저장 위치 |
| --- | --- | --- |
| Access Token | 15분 | Unity: PlayerPrefs 또는 메모리 |
| Refresh Token | 7일 | Unity: PlayerPrefs (SecureStorage 권장) / 서버: PostgreSQL |

---

## 엔티티 및 DB 설정

```csharp
// Domain/Entities/RefreshToken.cs
public class RefreshToken
{
    public long Id { get; private set; }
    public long PlayerId { get; private set; }
    public string Token { get; private set; } = null!;
    public DateTime ExpiresAt { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public bool IsRevoked { get; private set; }

    private RefreshToken() { }

    public static RefreshToken Create(long playerId, string token, DateTime expiresAt) => new()
    {
        PlayerId = playerId,
        Token = token,
        ExpiresAt = expiresAt,
        CreatedAt = DateTime.UtcNow,
        IsRevoked = false
    };

    public void Revoke() => IsRevoked = true;
    public bool IsExpired => DateTime.UtcNow > ExpiresAt;
    public bool IsActive => !IsRevoked && !IsExpired;
}
```

```csharp
// Infrastructure/Persistence/Configurations/RefreshTokenConfiguration.cs
public class RefreshTokenConfiguration : IEntityTypeConfiguration<RefreshToken>
{
    public void Configure(EntityTypeBuilder<RefreshToken> builder)
    {
        builder.ToTable("refresh_tokens");
        builder.HasKey(r => r.Id);
        builder.Property(r => r.Id).HasColumnName("id").UseIdentityByDefaultColumn();
        builder.Property(r => r.PlayerId).HasColumnName("player_id").IsRequired();
        builder.Property(r => r.Token).HasColumnName("token").HasMaxLength(512).IsRequired();
        builder.Property(r => r.ExpiresAt).HasColumnName("expires_at").IsRequired();
        builder.Property(r => r.CreatedAt).HasColumnName("created_at").HasDefaultValueSql("NOW()");
        builder.Property(r => r.IsRevoked).HasColumnName("is_revoked").HasDefaultValue(false);

        builder.HasIndex(r => r.Token).IsUnique();
        builder.HasIndex(r => r.PlayerId);

        builder.HasOne<Player>()
            .WithMany()
            .HasForeignKey(r => r.PlayerId)
            .OnDelete(DeleteBehavior.Cascade);
    }
}
```

---

## JWT 서비스 구현

```csharp
// Infrastructure/Security/JwtService.cs
public class JwtService
{
    private readonly IConfiguration _config;

    public JwtService(IConfiguration config) => _config = config;

    public string GenerateAccessToken(long playerId, string username)
    {
        var key = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(_config["Jwt:SecretKey"]!));
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, playerId.ToString()),
            new Claim(ClaimTypes.Name, username),
            new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())
        };

        var token = new JwtSecurityToken(
            issuer: _config["Jwt:Issuer"],
            audience: _config["Jwt:Audience"],
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(15),
            signingCredentials: creds);

        return new JwtSecurityTokenHandler().WriteToken(token);
    }

    public string GenerateRefreshToken() => Convert.ToBase64String(RandomNumberGenerator.GetBytes(64));
}
```

---

## Auth Service 핵심 로직

```csharp
public class AuthService : IAuthService
{
    public async Task<LoginResponse> LoginAsync(LoginRequest request)
    {
        var player = await _playerRepository.GetByUsernameAsync(request.Username)
            ?? throw new GameException("아이디 또는 비밀번호가 올바르지 않습니다.", HttpStatusCode.Unauthorized);

        if (!BCrypt.Net.BCrypt.Verify(request.Password, player.PasswordHash))
            throw new GameException("아이디 또는 비밀번호가 올바르지 않습니다.", HttpStatusCode.Unauthorized);

        var accessToken = _jwtService.GenerateAccessToken(player.Id, player.Username);
        var refreshTokenValue = _jwtService.GenerateRefreshToken();
        var refreshToken = RefreshToken.Create(player.Id, refreshTokenValue, DateTime.UtcNow.AddDays(7));

        await _refreshTokenRepository.AddAsync(refreshToken);

        return new LoginResponse
        {
            AccessToken = accessToken,
            RefreshToken = refreshTokenValue,
            ExpiresIn = 900 // 15분 (초)
        };
    }

    public async Task<RefreshTokenResponse> RefreshAsync(RefreshTokenRequest request)
    {
        var stored = await _refreshTokenRepository.GetByTokenAsync(request.RefreshToken)
            ?? throw new GameException("유효하지 않은 Refresh Token입니다.", HttpStatusCode.Unauthorized);

        if (!stored.IsActive)
            throw new GameException("만료되었거나 폐기된 Refresh Token입니다.", HttpStatusCode.Unauthorized);

        stored.Revoke();
        await _refreshTokenRepository.UpdateAsync(stored);

        var player = await _playerRepository.GetByIdAsync(stored.PlayerId)!;
        var newAccessToken = _jwtService.GenerateAccessToken(player.Id, player.Username);
        var newRefreshToken = RefreshToken.Create(player.Id, _jwtService.GenerateRefreshToken(), DateTime.UtcNow.AddDays(7));

        await _refreshTokenRepository.AddAsync(newRefreshToken);

        return new RefreshTokenResponse
        {
            AccessToken = newAccessToken,
            RefreshToken = newRefreshToken.Token,
            ExpiresIn = 900
        };
    }
}
```

---

## Program.cs JWT 설정

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:SecretKey"]!)),
            ClockSkew = TimeSpan.Zero // 만료 시각 엄격 적용
        };
    });
```

---

## appsettings.json 구조

```json
{
  "Jwt": {
    "SecretKey": "your-256-bit-secret-key-here-minimum-32-chars",
    "Issuer": "game-server",
    "Audience": "game-client"
  }
}
```

**주의**: `SecretKey` 를 코드/레포에 직접 포함 **금지** — 환경 변수 또는 Secret Manager 사용

---

## Unity 클라이언트 패턴

```csharp
// Unity: 토큰 저장
PlayerPrefs.SetString("access_token", loginResponse.accessToken);
PlayerPrefs.SetString("refresh_token", loginResponse.refreshToken);

// Unity: API 요청마다 헤더 첨부
request.SetRequestHeader("Authorization", $"Bearer {accessToken}");

// Unity: 401 수신 시 토큰 갱신 후 재요청
if (response.responseCode == 401)
{
    await RefreshTokenAsync();
    // 원래 요청 재시도
}
```

---

## API 엔드포인트

| Method | URL | 인증 | 설명 |
| --- | --- | --- | --- |
| POST | `/api/v1/auth/register` | 없음 | 회원가입 |
| POST | `/api/v1/auth/login` | 없음 | 로그인, 토큰 발급 |
| POST | `/api/v1/auth/refresh` | 없음 | Access Token 갱신 |
| POST | `/api/v1/auth/logout` | Bearer | Refresh Token 폐기 |
