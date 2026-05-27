| name | efcore-guide |
| --- | --- |
| description | EF Core 10 + PostgreSQL 엔티티 설계, EntityTypeConfiguration, 쿼리 최적화, N+1 방지, 마이그레이션 규칙. Code First 방식 전용. |

# EF Core Guide

## EntityTypeConfiguration 필수 사용

엔티티 설정은 반드시 `IEntityTypeConfiguration<T>` 로 분리. `OnModelCreating` 에 직접 작성 금지.

```csharp
// Infrastructure/Persistence/Configurations/PlayerConfiguration.cs
public class PlayerConfiguration : IEntityTypeConfiguration<Player>
{
    public void Configure(EntityTypeBuilder<Player> builder)
    {
        builder.ToTable("players");

        builder.HasKey(p => p.Id);
        builder.Property(p => p.Id)
            .HasColumnName("id")
            .UseIdentityByDefaultColumn(); // PostgreSQL BIGSERIAL

        builder.Property(p => p.Username)
            .HasColumnName("username")
            .HasMaxLength(50)
            .IsRequired();

        builder.Property(p => p.PasswordHash)
            .HasColumnName("password_hash")
            .HasMaxLength(72) // BCrypt max
            .IsRequired();

        builder.Property(p => p.CreatedAt)
            .HasColumnName("created_at")
            .HasDefaultValueSql("NOW()");

        builder.HasIndex(p => p.Username).IsUnique();
    }
}
```

```csharp
// Infrastructure/Persistence/AppDbContext.cs
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Player> Players => Set<Player>();
    public DbSet<GachaDraw> GachaDraws => Set<GachaDraw>();
    public DbSet<GameRecord> GameRecords => Set<GameRecord>();
    public DbSet<Currency> Currencies => Set<Currency>();
    public DbSet<RefreshToken> RefreshTokens => Set<RefreshToken>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}
```

---

## 쿼리 규칙

### 읽기 전용 — 반드시 AsNoTracking

```csharp
// ✅ 읽기 전용 쿼리
var player = await _context.Players
    .AsNoTracking()
    .FirstOrDefaultAsync(p => p.Id == id);

// ❌ 트래킹 불필요한데 AsNoTracking 누락
var player = await _context.Players
    .FirstOrDefaultAsync(p => p.Id == id);
```

### N+1 방지 — Include 사용

```csharp
// ❌ N+1 발생
var draws = await _context.GachaDraws.ToListAsync();
foreach (var d in draws) Console.WriteLine(d.Character.Name); // N개의 추가 쿼리

// ✅ Eager Loading
var draws = await _context.GachaDraws
    .AsNoTracking()
    .Include(d => d.Character)
    .Where(d => d.PlayerId == playerId)
    .ToListAsync();
```

### 단일 결과 조회

```csharp
// 있을 수도 없을 수도: FirstOrDefaultAsync (null 반환)
var player = await _context.Players
    .AsNoTracking()
    .FirstOrDefaultAsync(p => p.Username == username);

// 반드시 존재해야 함: FirstOrDefaultAsync + null 체크
var player = await _context.Players
    .AsNoTracking()
    .FirstOrDefaultAsync(p => p.Id == id)
    ?? throw new GameException("플레이어를 찾을 수 없습니다.", HttpStatusCode.NotFound);
```

---

## 쓰기 작업

### 단건 저장

```csharp
_context.Players.Add(player);
await _context.SaveChangesAsync();
```

### 수정

```csharp
// 트래킹된 엔티티 수정
var currency = await _context.Currencies // AsNoTracking 없이!
    .FirstOrDefaultAsync(c => c.PlayerId == playerId)
    ?? throw new GameException("재화 정보를 찾을 수 없습니다.", HttpStatusCode.NotFound);

currency.Consume(amount); // 도메인 메서드로 수정
await _context.SaveChangesAsync();
```

### 삭제

```csharp
_context.RefreshTokens.Remove(token);
await _context.SaveChangesAsync();
```

---

## 컬럼 네이밍 규칙

| C# 프로퍼티 | DB 컬럼 | 비고 |
| --- | --- | --- |
| `Id` | `id` | BIGSERIAL PRIMARY KEY |
| `PlayerId` | `player_id` | FK |
| `CreatedAt` | `created_at` | UTC, `NOW()` 기본값 |
| `UpdatedAt` | `updated_at` | UTC |
| `PasswordHash` | `password_hash` | BCrypt 해시 |
| `ExpiresAt` | `expires_at` | 만료 시각 |

---

## PostgreSQL 전용 타입 처리

```csharp
// appsettings.json 연결 문자열
"ConnectionStrings": {
  "Default": "Host=localhost;Database=game_db;Username=postgres;Password=postgres"
}

// Program.cs
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseNpgsql(builder.Configuration.GetConnectionString("Default")));
```

---

## DDL 전략

| 환경 | DDL 전략 |
| --- | --- |
| 개발 | `EnsureCreated()` 또는 Migration 자동 적용 |
| 프로덕션 | 수동 Migration (`dotnet ef database update`) |

`migrate:auto` 설정 **금지** (프로덕션에서 컬럼 자동 삭제 위험)
