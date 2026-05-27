# PR Reply Formats

## VALID (자동 수정 완료)

```
수정하였습니다. ({commit_hash})

`{파일명}` 의 {수정 내용}을 {CODEX.md §항목} 규칙에 따라 개선하였습니다.
```

예시:
```
수정하였습니다. (abc1234)

`CurrencyRepository.cs` 의 읽기 전용 쿼리에 `.AsNoTracking()` 을 추가하였습니다.
```

---

## INVALID (반박)

```
확인하였습니다. 현재 구현은 {근거} 를 따르고 있습니다.

{구체적 설명}
```

예시:
```
확인하였습니다. 현재 구현은 CODEX.md §보안 규칙을 따르고 있습니다.

`playerId` 는 `User.FindFirstValue(ClaimTypes.NameIdentifier)` 를 통해 JWT 클레임에서 추출하고 있어, 클라이언트 요청 파라미터를 신뢰하지 않습니다.
```

---

## PARTIAL (수락 후 수정 완료)

```
확인 후 수정하였습니다. ({commit_hash})

{수정 내용 설명}
```
