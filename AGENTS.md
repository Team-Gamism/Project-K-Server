# CODEX

서브컬처 러닝 액션 게임 웹 서버 프로젝트의 전체 규칙 및 컨벤션 문서입니다.

---

## 프로젝트 개요

| 항목 | 내용 |
| --- | --- |
| 장르 | 서브컬처 러닝 액션 (쿠키런 스타일) |
| 클라이언트 | Unity |
| 서버 언어 | C# 15 |
| 프레임워크 | ASP.NET Core 10 |
| ORM | EF Core 10 (Code First) |
| DB | PostgreSQL |
| API 스타일 | RESTful |
| API 문서화 | Swagger (Swashbuckle) |
| 테스트 | xUnit (단위 + 통합) |
| 형상관리 | GitHub (Git Flow) |
| 로깅 | Serilog |
| 에러 핸들링 | 전역 ExceptionMiddleware + Problem Details (RFC 7807) |

---

## 인증/보안

| 항목 | 내용 |
| --- | --- |
| 인증 방식 | JWT Bearer (Access Token + Refresh Token) |
| Refresh Token 저장 | PostgreSQL (`refresh_tokens` 테이블) |
| 비밀번호 암호화 | BCrypt (`BCrypt.Net-Next`) |

---

## 도메인 목록

| 도메인 | 설명 |
| --- | --- |
| `auth` | 회원가입, 로그인, JWT 발급/갱신/폐기 |
| `player` | 플레이어 프로필, 계정 관리 |
| `game` | 러닝 게임 기록 저장, 최고 기록 조회 |
| `gacha` | 가챠 뽑기, 확률 처리, 뽑기 이력 |
| `currency` | 뽑기 재화 조회 및 소모 |

---

## 네이밍 컨벤션

### C# 일반

| 대상 | 규칙 | 예시 |
| --- | --- | --- |
| 클래스 / 인터페이스 | PascalCase | `PlayerService`, `IGachaRepository` |
| 메서드 | PascalCase | `GetPlayerById`, `DrawGachaAsync` |
| 프로퍼티 | PascalCase | `PlayerName`, `CurrencyAmount` |
| 지역 변수 / 파라미터 | camelCase | `playerId`, `gachaResult` |
| 상수 | PascalCase | `MaxDrawCount`, `AccessTokenExpiry` |
| private 필드 | `_camelCase` | `_playerRepository`, `_jwtService` |
| async 메서드 | `~Async` 접미사 | `GetPlayerAsync`, `DrawGachaAsync` |

### DTO 네이밍

| 대상 | 규칙 | 예시 |
| --- | --- | --- |
| 요청 DTO | `{Action}{Domain}Request` | `LoginRequest`, `DrawGachaRequest` |
| 응답 DTO | `{Action}{Domain}Response` | `LoginResponse`, `DrawGachaResponse` |

### API URL

- 기본 경로: `/api/v1/{domain}`
- 복수형: `/api/v1/players`, `/api/v1/gacha/draws`
- 케밥케이스: `/api/v1/game-records`

### EF Core / DB

| 대상 | 규칙 | 예시 |
| --- | --- | --- |
| 테이블명 | snake_case 복수형 | `players`, `gacha_draws`, `refresh_tokens` |
| 컬럼명 | snake_case | `created_at`, `player_id`, `password_hash` |
| PK (C#) | `Id` | `public long Id { get; private set; }` |
| PK (DB) | `id` | `id BIGSERIAL PRIMARY KEY` |
| FK (C#) | `{Entity}Id` | `PlayerId` |
| FK (DB) | `{entity}_id` | `player_id` |

---

## 프로젝트 레이어 구조

```
src/
├── Controllers/           — HTTP 요청 수신, DTO 바인딩, 응답 반환
├── Services/              — 비즈니스 로직 (Interface + Implementation)
├── Repositories/          — EF Core DB 접근
├── Domain/
│   ├── Entities/          — EF Core 엔티티 (setter private)
│   ├── Enums/             — 열거형 (Rarity, GameResult 등)
│   └── DTOs/              — 요청/응답 DTO
├── Infrastructure/
│   ├── Persistence/       — AppDbContext, EntityTypeConfiguration
│   ├── Security/          — JWT 발급/검증, BCrypt 래퍼
│   └── Extensions/        — DI 등록 확장 메서드
└── Middlewares/           — ExceptionHandlingMiddleware
```

---

## 응답 형식

### 성공

DTO를 Controller에서 직접 반환 — 래퍼 금지

```json
// GET /api/v1/players/{id}
{
  "id": 1,
  "username": "user01",
  "currencyAmount": 300
}
```

### 실패 (Problem Details — RFC 7807)

```json
{
  "type": "https://tools.ietf.org/html/rfc7807",
  "title": "Not Found",
  "status": 404,
  "detail": "플레이어를 찾을 수 없습니다."
}
```

---

## 예외 처리 규칙

- `GameException(string message, HttpStatusCode status)` 를 직접 사용
- 서브클래스 생성 **금지**
- 전역 `ExceptionHandlingMiddleware` 에서 일괄 처리 → Problem Details 반환
- 예외 메시지: 한국어, 마침표 없음
- Serilog 로그 메시지: 영어 전용, `{}` 플레이스홀더 사용
- 로그에 비밀번호 / 토큰 / 개인정보 포함 **금지**

---

## Git 브랜치 전략 (Git Flow)

| 브랜치 | 역할 |
| --- | --- |
| `main` | 프로덕션 배포본 |
| `develop` | 개발 통합 브랜치 |
| `{type}/{kebab-description}` | 기능 브랜치 (develop에서 분기) |

---

## 커밋 메시지 컨벤션

형식: `type(scope): 설명`

| type | 용도 |
| --- | --- |
| `add` | 새 기능 추가 |
| `update` | 기존 기능 수정/개선 |
| `fix` | 버그 수정 |
| `refactor` | 리팩토링 |
| `test` | 테스트 추가/수정 |
| `docs` | 문서 변경 |
| `ci/cd` | 빌드/배포 파이프라인 |
| `merge` | 브랜치 머지 |

scope: 도메인명 우선 (`auth` / `player` / `game` / `gacha` / `currency`) — 전역 변경은 `global`

설명: 한국어, 마침표 없음, `~한다/~됩니다/~했습니다` 어미 금지

```
add(gacha): 가챠 확률 테이블 엔티티 추가
fix(auth): Refresh Token 만료 검증 오류 수정
refactor(global): 전역 예외 핸들러 개선
```

---

## 테스트 규칙

| 항목 | 내용 |
| --- | --- |
| 프레임워크 | xUnit |
| Mock 라이브러리 | Moq 또는 NSubstitute |
| 통합 테스트 | `WebApplicationFactory` + Testcontainers (PostgreSQL) |
| 패턴 | Given-When-Then |
| 메서드명 | `{MethodName}_{Scenario}_{ExpectedResult}` |

---

## Skills 목록

| 스킬 | 설명 |
| --- | --- |
| `aspnet-game-arch` | ASP.NET Core 레이어 아키텍처 상세 가이드 |
| `efcore-guide` | EF Core 엔티티 설계, 쿼리, 설정 가이드 |
| `jwt-auth` | JWT 발급/검증/갱신 구현 가이드 |
| `unity-api-design` | Unity 클라이언트를 위한 API 설계 가이드 |
| `game-security-checklist` | 게임 서버 보안 점검 체크리스트 |
| `game-transaction` | 재화/가챠/보상 트랜잭션 처리 가이드 |
| `test-strategy` | 단위/통합 테스트 전략 및 작성 가이드 |
| `commit` | Git 커밋 컨벤션 및 브랜치 자동화 |
| `code-review` | 코드 리뷰 체크리스트 (✓/⚠/✗ 리포트) |
| `migration-guide` | EF Core 마이그레이션 절차 가이드 |
| `write-pr` | PR 제목/본문/라벨 자동 생성 |
| `review-pr` | PR 리뷰 코멘트 자동 처리 |
| `plan-deep-dive` | 구현 계획 수립 가이드 |
