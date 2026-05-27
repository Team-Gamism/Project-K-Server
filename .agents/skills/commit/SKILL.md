| name | commit |
| --- | --- |
| description | Git 커밋 컨벤션에 따라 변경사항을 논리 단위로 분리하여 커밋. Git Flow 자동 처리 — develop 브랜치 감지 시 feature 브랜치 생성 후 커밋. |
| allowed-tools | Bash |

## Step 0 — 브랜치 확인 (필수)

```
git branch --show-current
```

**현재 브랜치가 `develop` 인 경우:**

Git Flow 규칙에 따라 feature 브랜치를 생성해야 합니다.

1. `git status`, `git diff` 로 변경사항 전체 분석
2. 변경사항에서 브랜치명 추론:
   - 형식: `{type}/{kebab-case-description}`
   - 커밋 type과 동일 사용 (예외: `ci/cd` → `cicd/`)
   - 예시: `add/gacha-probability-table`, `fix/auth-refresh-token-expiry`, `refactor/currency-consume-logic`
3. 브랜치 생성 및 체크아웃:

```
git checkout -b {type}/{inferred-name}
```

4. 아래 커밋 흐름으로 진행

**현재 브랜치가 `develop` 이 아닌 경우:** 바로 커밋 흐름으로 진행.

---

## 커밋 메시지 규칙

형식: `type(scope): 설명`

### Types (영어)

| type | 용도 |
| --- | --- |
| `add` | 새 기능, 파일, 엔티티 추가 |
| `update` | 기존 기능 수정/개선 |
| `fix` | 버그 수정 |
| `refactor` | 리팩토링 (기능 변경 없음) |
| `test` | 테스트 추가/수정 |
| `docs` | 문서 변경 |
| `ci/cd` | 빌드/배포 파이프라인 |
| `merge` | 브랜치 머지 |

### Scopes (영어)

- **도메인명 우선**: `auth` / `player` / `game` / `gacha` / `currency`
- **전역 변경**: `global` (여러 도메인에 걸친 변경)
- **인프라**: `ci/cd`
- 스코프 선택 상세 규칙: `references/scope-guide.md` 참고

### 설명 규칙

- 한국어
- 마침표 없음
- 금지 어미: `~한다/~된다`, `~하기`, `~합니다/~됩니다`, `~했습니다`
- 좋은 예: `가챠 확률 테이블 엔티티 추가`, `Refresh Token 만료 검증 오류 수정`
- AI 도구 co-author 추가 금지

### 예시

```
add(gacha): 가챠 확률 테이블 엔티티 추가
fix(auth): Refresh Token 만료 검증 오류 수정
refactor(currency): Currency 도메인 메서드 개선
test(player): 플레이어 서비스 단위 테스트 추가
docs(global): CODEX 네이밍 컨벤션 보완
```

---

## 커밋 흐름

1. 변경 확인

```
git status
git diff
```

2. 논리 단위로 분류 (기능 / 버그 / 리팩토링 / 테스트 등)
3. 단위별 파일 그룹화
4. 각 그룹 커밋:

```
git add {관련 파일만}
git commit -m "type(scope): 설명"
```

5. 결과 확인:

```
git log --oneline -n 5
```
