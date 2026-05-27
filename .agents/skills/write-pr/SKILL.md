| name | write-pr |
| --- | --- |
| description | develop 이후 커밋에서 PR 제목/본문/라벨을 자동 생성하고 GitHub에 PR을 생성. 베이스 브랜치 감지, 라벨 선택, PR 생성 전체 자동화. |
| allowed-tools | Bash(git *), Bash(gh *), Bash(cat *), Read, Write |

## Step 1 — 컨텍스트 수집

```bash
git branch --show-current
git log origin/develop..HEAD --oneline 2>/dev/null || git log --oneline -15
git diff origin/develop...HEAD --stat 2>/dev/null || git diff HEAD~5...HEAD --stat
git diff origin/develop...HEAD 2>/dev/null || git diff HEAD~5...HEAD
```

PR 템플릿 읽기:
```bash
cat .github/PULL_REQUEST_TEMPLATE.md 2>/dev/null || echo "템플릿 없음"
```

## Step 2 — 라벨 결정

`references/labels.md` 를 읽고 변경 성격에 맞는 라벨 1~2개 선택.

## Step 3 — PR 내용 생성

**제목** — 3개 후보 생성, 형식: `[scope] 설명`

- scope: 변경 파일 기반 도메인 (`[auth]`, `[gacha]`, `[player]` 등) / 전역 변경: `[global]`
- 설명: 한국어, 간결, 이모지 없음, 최대 50자
- 클래스명/어노테이션/기술 용어는 백틱으로 감싸기 (예: `` `GameException` ``, `` `[Authorize]` ``)

**본문** — PR 템플릿 구조를 따르되:
- 한국어 합쇼체: `~하였습니다`, `~추가하였습니다`, `~수정하였습니다`
- 이모지 없음
- 최대 2500자
- 모든 클래스명/메서드명/어노테이션/파일명 백틱으로 감싸기

## Step 4 — 미리보기 및 제목 선택

```
## PR 제목 후보
1. [gacha] 가챠 뽑기 확률 테이블 엔티티 추가
2. [gacha] `GachaProbabilityTable` 엔티티 및 EF Core 설정 추가
3. [gacha] 가챠 시스템 확률 테이블 도메인 모델 구현

## 선택된 라벨
- enhancement:개선작업

## PR 본문 미리보기
[본문 내용]
```

사용자에게 1/2/3 중 선택 요청. 선택 후 Step 5 진행.

## Step 5 — PR 생성

```bash
# 본문 파일 저장
echo "{body}" > PR_BODY.md

# PR 생성
gh pr create \
  --title "{confirmed-title}" \
  --body-file PR_BODY.md \
  --base develop \
  --label "{label1},{label2}"

# 정리
rm -f PR_BODY.md
```

PR URL 출력 후 종료.
