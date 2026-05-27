| name | review-pr |
| --- | --- |
| description | PR 리뷰 코멘트 수집 → CODEX.md 기준으로 각 코멘트 평가 → VALID는 자동 코드 수정, INVALID는 반박 댓글 작성, PARTIAL은 사용자 확인 요청. |
| compatibility | git, gh (GitHub CLI), jq 필요 |
| allowed-tools | Bash(gh api:*), Bash(gh pr view:*), Bash(git add:*), Bash(git commit:*), Bash(git push:*), Bash(rm:*), Edit, Read |

## Step 1 — PR 데이터 수집

```bash
gh repo view --json nameWithOwner -q .nameWithOwner
gh pr view --json number,baseRefName -q '{number: .number, base: .baseRefName}'
gh api "repos/{owner}/{repo}/pulls/{pr_number}/comments" \
  --jq '[.[] | {id: .id, path: .path, line: .line, body: .body, user: .user.login}]'
```

## Step 2 — 각 코멘트 평가

각 코멘트에 대해 **우선순위 순서**로 판단:

1. **CODEX.md / 프로젝트 컨벤션** (1순위)
   - 네이밍 규칙, DTO 규칙, `GameException` 사용, 로깅 스타일, 트랜잭션 패턴
2. **C# / ASP.NET Core 모범 사례** (2순위)
   - 프로젝트 규칙에 해당 없는 경우만 적용

### 판정

| 판정 | 조건 | 처리 |
| --- | --- | --- |
| **VALID** | 리뷰어가 올바름 | 코드 자동 수정 후 커밋 |
| **INVALID** | 리뷰어가 틀렸고 명확한 반박 가능 | 반박 댓글 작성, 코드 수정 없음 |
| **PARTIAL** | 의도는 맞으나 적용 방법 또는 범위 불명확 | 사용자에게 확인 요청 |

판정 근거에는 구체적 출처 명시: `CODEX.md §예외처리`, `aspnet-game-arch SKILL §Controller`

## Step 3 — 판정별 처리

### VALID → 코드 자동 수정

1. 대상 파일 Read
2. Edit으로 수정
3. 커밋:
```bash
git add {file}
git commit -m "fix({scope}): {리뷰 내용 요약}"
git rev-parse --short=7 HEAD  # 커밋 해시 기록
```

### INVALID → 반박 댓글

코드 수정 없음. 반박 근거 기록 후 Step 6에서 댓글 작성.

### PARTIAL → 사용자 확인

```
⚠️ PARTIAL: [{file}:{line}] (@{reviewer})
리뷰: "{comment}"
판단 근거: ...
수락할까요? (y = 수정 / n = 반박 / s = 나중에)
```

## Step 4 — 결과 리포트 출력

```
## review-pr 결과

| # | 리뷰어 | 파일 | 판정 | 근거 | 처리 |
|---|--------|------|------|------|------|
| 1 | alice | GachaService.cs:45 | ✅ VALID | CODEX.md §트랜잭션 | 자동 수정 (abc1234) |
| 2 | bob | PlayerController.cs:12 | ❌ INVALID | CODEX.md §보안: JWT 클레임 사용 올바름 | 반박 댓글 작성 |
| 3 | alice | CurrencyService.cs:28 | ⚠️ PARTIAL | - | PENDING |
```

## Step 5 — 커밋 Push

VALID 수정이 있는 경우:
```bash
git push
```

## Step 6 — GitHub 댓글 작성

```bash
gh api "repos/{owner}/{repo}/pulls/{pr_number}/comments/{comment_id}/replies" \
  -f body="{reply_body}"
```

댓글 형식은 `references/reply-formats.md` 참고.

## Step 7 — 정리

```bash
rm -rf .pr-tmp
```
