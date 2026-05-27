# Commit Scope Selection Guide

## 우선순위 규칙

**도메인명 > global**

변경된 파일이 속한 도메인명을 scope로 사용.
여러 도메인에 걸친 변경 또는 인프라 전반 변경만 `global` 또는 `ci/cd` 사용.

## 도메인 목록

| 도메인 | 해당 디렉터리/파일 |
| --- | --- |
| `auth` | `AuthController`, `AuthService`, `RefreshToken`, `JwtService` |
| `player` | `PlayerController`, `PlayerService`, `Player` 엔티티 |
| `game` | `GameController`, `GameService`, `GameRecord` 엔티티 |
| `gacha` | `GachaController`, `GachaService`, `GachaDraw` 엔티티, 확률 테이블 |
| `currency` | `CurrencyController`, `CurrencyService`, `Currency` 엔티티 |

## 전역/인프라 scope

| Scope | 사용 시점 |
| --- | --- |
| `global` | 여러 도메인에 걸친 변경 (예: 전역 예외 처리, 공통 인터페이스) |
| `ci/cd` | GitHub Actions, Dockerfile, 빌드 스크립트 |

## 잘못된 예 vs 올바른 예

| 잘못된 예 | 올바른 예 | 이유 |
| --- | --- | --- |
| `fix(global): Refresh Token 만료 검증 수정` | `fix(auth): Refresh Token 만료 검증 수정` | Refresh Token은 auth 도메인 |
| `add(global): 가챠 확률 테이블 추가` | `add(gacha): 가챠 확률 테이블 추가` | 가챠 도메인 변경 |
| `update(global): Currency 소모 로직 개선` | `update(currency): Currency 소모 로직 개선` | currency 도메인 변경 |

## 올바른 global 사용 예

```
refactor(global): 전역 예외 핸들러 미들웨어 개선
update(global): GameException 생성자 오버로드 추가
ci/cd: GitHub Actions 테스트 워크플로우 추가
```
