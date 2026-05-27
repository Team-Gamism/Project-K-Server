| name | game-security-checklist |
| --- | --- |
| description | 게임 서버 보안 점검 체크리스트. 클라이언트 데이터 신뢰 금지, 소유권 검증, JWT 보안, 재화/가챠 조작 방지, 민감 정보 보호. ✓/⚠/✗ 리포트 형식으로 출력. |

# Game Security Checklist

변경된 파일 또는 전체 코드베이스를 대상으로 아래 항목을 검증합니다.

## 체크리스트

### 클라이언트 입력 신뢰 금지

- [ ] 재화 수량, 점수, 결과값을 클라이언트에서 전달받아 그대로 저장하지 않음
- [ ] 가챠 결과를 클라이언트가 결정할 수 없음 (서버에서만 계산)
- [ ] 게임 기록(거리, 점수)을 서버에서 2차 검증하거나 최소한 범위 검증
- [ ] 모든 수치 입력에 `[Range]` 또는 FluentValidation 적용

```csharp
// ❌ 위험: 클라이언트가 점수를 결정
[HttpPost("records")]
public async Task SaveRecord([FromBody] SaveRecordRequest req)
{
    await _gameService.SaveAsync(req.Score, req.Distance); // 클라이언트 데이터 그대로 저장
}

// ✅ 안전: 서버에서 범위 검증
public class SaveRecordRequest
{
    [Range(0, 999999)]
    public int Distance { get; set; } // 비정상적 값 필터링
}
```

### 소유권 검증

- [ ] 플레이어는 자신의 리소스에만 접근 가능
- [ ] 타인의 보상 수령, 가챠 이력 조회 불가
- [ ] JWT `sub` 클레임에서 `playerId` 추출, 쿼리 파라미터 `playerId` 신뢰 금지

```csharp
// ❌ 위험: 요청의 playerId를 그대로 사용
[HttpGet("currency")]
public async Task<CurrencyResponse> GetCurrency([FromQuery] long playerId)
    => await _currencyService.GetByPlayerIdAsync(playerId); // 타인 정보 조회 가능

// ✅ 안전: 토큰에서 playerId 추출
[HttpGet("currency")]
[Authorize]
public async Task<CurrencyResponse> GetCurrency()
{
    var playerId = long.Parse(User.FindFirstValue(ClaimTypes.NameIdentifier)!);
    return await _currencyService.GetByPlayerIdAsync(playerId);
}
```

### JWT 보안

- [ ] `[Authorize]` 어트리뷰트 누락 없음
- [ ] `SecretKey` 환경 변수 또는 Secret Manager 사용 (코드/레포 직접 포함 금지)
- [ ] `ClockSkew = TimeSpan.Zero` 설정 (만료 엄격 적용)
- [ ] Refresh Token 재사용 방지: 사용 시 즉시 폐기 후 신규 발급
- [ ] Refresh Token `IsRevoked` 검증 필수

```csharp
// ✅ Refresh Token 검증
if (!stored.IsActive) // IsRevoked || IsExpired
    throw new GameException("유효하지 않은 Refresh Token입니다.", HttpStatusCode.Unauthorized);
```

### 재화 / 가챠 조작 방지

- [ ] 재화 소모와 가챠 결과 저장이 단일 트랜잭션 내에서 처리
- [ ] 재화 소모 전 잔액 검증 (도메인 레이어)
- [ ] 가챠 확률 계산은 서버에서만 수행
- [ ] 보상 중복 수령 방지 (`IsClaimed` 플래그 트랜잭션 내 확인)

### 민감 정보 보호

- [ ] 비밀번호 BCrypt 해시 저장 (평문 금지)
- [ ] 로그에 비밀번호, 토큰, 개인정보 포함 금지
- [ ] 에러 응답에 스택 트레이스 포함 금지 (Problem Details 만 반환)
- [ ] `appsettings.json` 에 실제 시크릿 저장 금지 — 환경 변수로 분리

```csharp
// ❌ 위험: 로그에 토큰 포함
_logger.LogInformation("Login success, token: {Token}", accessToken);

// ✅ 안전: 토큰 제외
_logger.LogInformation("Player {PlayerId} logged in successfully", player.Id);
```

### HTTPS / 네트워크

- [ ] 프로덕션 환경 HTTPS 강제 (`app.UseHttpsRedirection()`)
- [ ] CORS 설정에서 Unity 클라이언트 도메인만 허용 (와일드카드 `*` 금지)

### SQL Injection 방지

- [ ] Raw SQL 사용 시 파라미터화 쿼리만 사용
- [ ] EF Core LINQ 쿼리 사용 시 자동 파라미터화 — 문자열 보간 금지

```csharp
// ❌ 위험: SQL Injection 가능
var players = _context.Players.FromSqlRaw($"SELECT * FROM players WHERE username = '{username}'");

// ✅ 안전: 파라미터화
var players = _context.Players.FromSqlRaw("SELECT * FROM players WHERE username = {0}", username);
// 또는 LINQ (권장)
var player = await _context.Players.FirstOrDefaultAsync(p => p.Username == username);
```

---

## 리포트 형식

각 항목:
- ✓ 통과
- ⚠ 경고 (권장 사항)
- ✗ 오류 (반드시 수정)

최종 요약:
```
총 {n}개 항목 확인
{p}개 통과, {w}개 경고, {e}개 오류
```
