| name | plan-deep-dive |
| --- | --- |
| description | 새 기능 구현 전 체계적 계획 수립 — 도메인 분석, DB 스키마 설계, API 엔드포인트 정의, 레이어별 구현 계획, 위험 요소 식별. 구현 시작 전 플랜 공유 및 승인 흐름 포함. |

# Plan Deep Dive

새 기능 또는 복잡한 변경 전에 이 스킬을 실행합니다.

## Step 1 — 요구사항 분석

기능 요구사항을 아래 항목으로 분해:

- **목표**: 무엇을 달성하는가?
- **도메인**: 어느 도메인에 속하는가? (`auth` / `player` / `game` / `gacha` / `currency`)
- **트리거**: 어떤 이벤트가 이 기능을 시작하는가?
- **성공 조건**: 어떤 상태가 "완료"인가?
- **실패 조건**: 어떤 경우에 오류를 반환하는가?

## Step 2 — 영향 범위 분석

기존 코드베이스 분석:

```bash
# 관련 도메인 파일 탐색
find . -type f -name "*.cs" | xargs grep -l "{keyword}" | grep -v "/obj/"

# 기존 엔티티 확인
find . -name "*.cs" -path "*/Entities/*" | head -20
```

- 수정이 필요한 기존 파일 목록
- 새로 생성할 파일 목록
- 영향받는 테스트 목록

## Step 3 — DB 스키마 설계

필요한 경우 새 테이블 또는 컬럼 설계:

```
테이블: {table_name}
컬럼:
  - id BIGSERIAL PRIMARY KEY
  - player_id BIGINT NOT NULL REFERENCES players(id)
  - {column} {type} {constraints}
  - created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()

인덱스:
  - idx_{table}_{column} ON {table}({column})
```

## Step 4 — API 엔드포인트 설계

```
Method: POST
URL: /api/v1/{domain}/{action}
인증: Bearer Token 필요 여부
요청 DTO: {ActionRequest}
  - {field}: {type} — {설명}
응답 DTO: {ActionResponse}
  - {field}: {type} — {설명}
에러:
  - 400: {사유}
  - 401: 인증 실패
  - 404: {사유}
```

## Step 5 — 레이어별 구현 계획

### Entity

```
추가할 프로퍼티:
수정할 메서드:
새로 추가할 도메인 메서드:
```

### EntityTypeConfiguration

```
새 컬럼 매핑:
새 인덱스:
관계 설정:
```

### Repository

```
새 메서드:
  - {MethodName}Async({params}): {return type}
수정할 메서드:
```

### Service

```
새 메서드:
  - {MethodName}Async({params}): {return type}
비즈니스 로직 흐름:
  1. ...
  2. ...
트랜잭션 필요 여부: Y/N
```

### Controller

```
새 엔드포인트:
  - [{Method}] {url}
DTO 목록:
  - {RequestDto}: {필드 목록}
  - {ResponseDto}: {필드 목록}
```

## Step 6 — 위험 요소 식별

| 위험 | 가능성 | 완화 방법 |
| --- | --- | --- |
| 재화 중복 소모 | 중 | 트랜잭션으로 원자성 보장 |
| N+1 쿼리 | 저 | Include() 적용 |
| 토큰 탈취 | 저 | HTTPS + 짧은 만료시간 |

## Step 7 — 구현 순서

```
1. EF Core 마이그레이션 생성
2. Entity 수정
3. Repository 구현
4. Service 구현 (단위 테스트 병행)
5. Controller 구현
6. 통합 테스트 작성
7. Swagger 문서 확인
```

## 플랜 출력 형식

```markdown
## 구현 계획: {기능명}

### 요약
{한 줄 설명}

### 변경 파일
- 신규: {파일 목록}
- 수정: {파일 목록}

### API
{엔드포인트 요약}

### DB 변경
{스키마 변경 요약}

### 구현 순서
{단계별 목록}

### 위험 요소
{위험 목록}
```

플랜 출력 후 사용자 승인을 기다린 뒤 구현 시작.
