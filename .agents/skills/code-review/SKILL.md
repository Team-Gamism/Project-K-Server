| name | code-review |
| --- | --- |
| description | 변경 파일 대상 구조화된 코드 리뷰 — DTO 규칙, C# 스타일, EF Core/트랜잭션 정확성, 보안, 테스트 커버리지, 커밋 컨벤션 검증. ✓/⚠/✗ 리포트 출력. |

# Code Review Guide

## 변경 확인

```
git diff develop...HEAD --stat
git diff develop...HEAD
```

## 체크리스트

### DTO

- [ ] 요청 DTO: `{Action}{Domain}Request` 네이밍?
- [ ] 응답 DTO: `{Action}{Domain}Response` 네이밍?
- [ ] 필수 필드에 `[Required]` 적용?
- [ ] 수치 범위에 `[Range]` 적용?
- [ ] 컨트롤러에서 DTO 직접 반환? (래퍼 객체 금지)

### C# 스타일

- [ ] 클래스/메서드/프로퍼티: PascalCase?
- [ ] 지역변수/파라미터: camelCase?
- [ ] private 필드: `_camelCase`?
- [ ] async 메서드: `~Async` 접미사?
- [ ] 생성자 주입만 사용? (프로퍼티 주입 금지)
- [ ] 엔티티 프로퍼티: `private set`?

### EF Core / DB

- [ ] 읽기 전용 쿼리: `.AsNoTracking()` 적용?
- [ ] N+1 문제 없음? (`Include()` 사용?)
- [ ] 트랜잭션 내 수정 대상 엔티티에 `AsNoTracking` 없음?
- [ ] 테이블/컬럼명: snake_case?
- [ ] `EntityTypeConfiguration` 클래스로 분리?

### 트랜잭션

- [ ] 재화 소모 + 결과 저장이 단일 트랜잭션?
- [ ] `try-catch-rollback` 패턴 적용?
- [ ] 읽기 전용 조회에 불필요한 트랜잭션 없음?

### 예외 처리

- [ ] `GameException` 직접 사용? (서브클래스 금지)
- [ ] 예외 메시지: 한국어, 마침표 없음?
- [ ] 동적 데이터(ID, 이름 등)가 메시지에 포함되지 않음?

### 보안

- [ ] `[Authorize]` 누락 없음?
- [ ] `playerId`: JWT 클레임에서 추출? (요청 파라미터 신뢰 금지)
- [ ] 로그에 비밀번호/토큰 없음?
- [ ] 코드에 하드코딩된 시크릿 없음?

### 로깅

- [ ] 영어 메시지?
- [ ] Serilog `{}` 플레이스홀더 사용? (문자열 보간 금지)
- [ ] 민감 정보 없음?

### 테스트

- [ ] 새 기능에 단위 테스트 작성?
- [ ] Given-When-Then 패턴 준수?
- [ ] 메서드명: `{Method}_{Scenario}_{ExpectedResult}`?
- [ ] Mock `Verify` 적절히 사용?

### 커밋

- [ ] `type(scope): 설명` 형식 준수?
- [ ] scope에 도메인명 사용? (`auth`/`player`/`game`/`gacha`/`currency`)
- [ ] 논리 단위로 커밋 분리?
- [ ] 금지 어미 없음? (`~한다/~됩니다`)

---

## 리포트 형식

```
## Code Review 결과

| # | 파일 | 항목 | 결과 | 내용 |
|---|------|------|------|------|
| 1 | GachaService.cs | 트랜잭션 | ✓ | 재화 소모와 뽑기 이력 저장이 단일 트랜잭션 |
| 2 | PlayerController.cs | 보안 | ✗ | playerId를 쿼리 파라미터에서 직접 사용 — JWT 클레임 추출 필요 |
| 3 | CurrencyRepository.cs | EF Core | ⚠ | AsNoTracking 누락 (읽기 전용 쿼리) |

총 {n}개 항목 확인
{p}개 통과, {w}개 경고, {e}개 오류
```
