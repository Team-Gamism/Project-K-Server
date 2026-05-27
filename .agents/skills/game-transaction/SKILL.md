| name | game-transaction |
| --- | --- |
| description | 재화 소모, 가챠 뽑기, 보상 지급 등 게임 내 중요 데이터 변경 시 트랜잭션 처리 가이드. 원자성 보장 패턴, 교착 방지, 멱등성 설계 포함. |

# Game Transaction Guide

## 트랜잭션 필수 적용 대상

| 작업 | 이유 |
| --- | --- |
| 뽑기 재화 소모 + 가챠 결과 저장 | 재화만 빠지고 결과 저장 실패 방지 |
| 보상 수령 + 재화 적립 | 보상은 사라졌는데 재화 미지급 방지 |
| 게임 기록 저장 + 보상 지급 | 기록 없이 보상만 지급 방지 |
| Refresh Token 폐기 + 신규 발급 | 이중 사용 방지 |

---

## 기본 트랜잭션 패턴

```csharp
public async Task<DrawGachaResponse> DrawAsync(long playerId, DrawGachaRequest request)
{
    await using var transaction = await _context.Database.BeginTransactionAsync();
    try
    {
        // 1. 재화 조회 (트래킹 필요 — AsNoTracking 사용 금지)
        var currency = await _context.Currencies
            .FirstOrDefaultAsync(c => c.PlayerId == playerId)
            ?? throw new GameException("재화 정보를 찾을 수 없습니다.", HttpStatusCode.NotFound);

        // 2. 재화 충분 여부 검증
        int totalCost = request.DrawCount * CostPerDraw;
        if (currency.Amount < totalCost)
            throw new GameException("뽑기 재화가 부족합니다.", HttpStatusCode.BadRequest);

        // 3. 재화 소모 (도메인 메서드)
        currency.Consume(totalCost);

        // 4. 가챠 결과 계산
        var result = CalculateGachaResult();

        // 5. 뽑기 이력 저장
        var draw = GachaDraw.Create(playerId, result.CharacterId);
        _context.GachaDraws.Add(draw);

        // 6. 일괄 저장 + 커밋
        await _context.SaveChangesAsync();
        await transaction.CommitAsync();

        return result.ToResponse();
    }
    catch
    {
        await transaction.RollbackAsync();
        throw;
    }
}
```

---

## 재화 엔티티 도메인 메서드

```csharp
public class Currency
{
    public long Id { get; private set; }
    public long PlayerId { get; private set; }
    public int Amount { get; private set; }

    private Currency() { }

    public static Currency Create(long playerId, int initialAmount = 0) => new()
    {
        PlayerId = playerId,
        Amount = initialAmount
    };

    public void Earn(int amount)
    {
        if (amount <= 0) throw new GameException("지급 재화는 0보다 커야 합니다.", HttpStatusCode.BadRequest);
        Amount += amount;
    }

    public void Consume(int amount)
    {
        if (amount <= 0) throw new GameException("소모 재화는 0보다 커야 합니다.", HttpStatusCode.BadRequest);
        if (Amount < amount) throw new GameException("뽑기 재화가 부족합니다.", HttpStatusCode.BadRequest);
        Amount -= amount;
    }
}
```

---

## 주의사항

### 트랜잭션 내 AsNoTracking 금지

```csharp
// ❌ AsNoTracking → EF Core가 변경 감지 불가 → SaveChanges 무시
var currency = await _context.Currencies
    .AsNoTracking()
    .FirstOrDefaultAsync(c => c.PlayerId == playerId);
currency.Consume(cost); // 이 변경이 저장되지 않음!

// ✅ 트랜잭션 내 수정 대상 엔티티는 트래킹 활성화
var currency = await _context.Currencies
    .FirstOrDefaultAsync(c => c.PlayerId == playerId);
currency.Consume(cost); // 변경 감지 정상 동작
await _context.SaveChangesAsync();
```

### 비즈니스 예외는 롤백 전 throw

```csharp
// ✅ 비즈니스 예외(GameException)도 catch → rollback → rethrow
catch (Exception) // GameException 포함
{
    await transaction.RollbackAsync();
    throw; // 전역 Middleware가 처리
}
```

### 읽기 전용 조회는 트랜잭션 외부에서

```csharp
// ✅ 변경 없는 단순 조회는 트랜잭션 불필요
public async Task<CurrencyResponse> GetCurrencyAsync(long playerId)
{
    var currency = await _context.Currencies
        .AsNoTracking()
        .FirstOrDefaultAsync(c => c.PlayerId == playerId)
        ?? throw new GameException("재화 정보를 찾을 수 없습니다.", HttpStatusCode.NotFound);

    return currency.ToResponse();
}
```

---

## 보상 지급 패턴

```csharp
public async Task ClaimRewardAsync(long playerId, long rewardId)
{
    await using var transaction = await _context.Database.BeginTransactionAsync();
    try
    {
        var reward = await _context.Rewards
            .FirstOrDefaultAsync(r => r.Id == rewardId && r.PlayerId == playerId)
            ?? throw new GameException("보상을 찾을 수 없습니다.", HttpStatusCode.NotFound);

        if (reward.IsClaimed)
            throw new GameException("이미 수령한 보상입니다.", HttpStatusCode.BadRequest);

        var currency = await _context.Currencies
            .FirstOrDefaultAsync(c => c.PlayerId == playerId)
            ?? throw new GameException("재화 정보를 찾을 수 없습니다.", HttpStatusCode.NotFound);

        reward.Claim();          // 보상 수령 처리
        currency.Earn(reward.Amount); // 재화 지급

        await _context.SaveChangesAsync();
        await transaction.CommitAsync();
    }
    catch
    {
        await transaction.RollbackAsync();
        throw;
    }
}
```
