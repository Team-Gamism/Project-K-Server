| name | unity-api-design |
| --- | --- |
| description | Unity 클라이언트를 위한 RESTful API 설계 가이드 — URL 구조, 인증 헤더, 응답 형식, 에러 처리, 버전 관리. |

# Unity API Design Guide

## URL 구조

```
/api/v1/{domain}/{resource}
```

| 도메인 | 예시 URL |
| --- | --- |
| auth | `/api/v1/auth/register`, `/api/v1/auth/login`, `/api/v1/auth/refresh` |
| players | `/api/v1/players/me`, `/api/v1/players/{id}` |
| game | `/api/v1/game/records`, `/api/v1/game/records/best` |
| gacha | `/api/v1/gacha/draw`, `/api/v1/gacha/history` |
| currency | `/api/v1/currency` |

---

## HTTP Method 규칙

| 동작 | Method | 예시 |
| --- | --- | --- |
| 조회 | GET | `GET /api/v1/players/me` |
| 생성/실행 | POST | `POST /api/v1/gacha/draw` |
| 전체 수정 | PUT | `PUT /api/v1/players/me` |
| 부분 수정 | PATCH | `PATCH /api/v1/players/me` |
| 삭제 | DELETE | `DELETE /api/v1/auth/logout` |

---

## 인증 헤더

인증이 필요한 모든 요청에 Bearer 토큰 포함:

```
Authorization: Bearer {accessToken}
```

---

## 응답 형식

### 성공

DTO를 직접 반환. 래퍼 없음.

```json
// GET /api/v1/currency
{
  "playerId": 1,
  "amount": 300
}

// POST /api/v1/gacha/draw
{
  "characterId": 42,
  "characterName": "엘리시아",
  "rarity": "SSR",
  "isNew": true
}
```

### 실패 (Problem Details — RFC 7807)

```json
{
  "title": "Bad Request",
  "status": 400,
  "detail": "뽑기 재화가 부족합니다."
}
```

Unity에서 `status` 필드로 에러 분기:

```csharp
var error = JsonConvert.DeserializeObject<ProblemDetailsDto>(response.downloadHandler.text);
switch (error.Status)
{
    case 400: ShowMessage(error.Detail); break;
    case 401: await RefreshTokenAndRetry(); break;
    case 404: ShowMessage("데이터를 찾을 수 없습니다."); break;
    default: ShowMessage("서버 오류가 발생했습니다."); break;
}
```

---

## HTTP 상태 코드 사용 기준

| 상황 | 코드 |
| --- | --- |
| 성공 (조회) | 200 OK |
| 성공 (생성) | 201 Created |
| 성공 (내용 없음) | 204 No Content |
| 잘못된 요청 (재화 부족 등) | 400 Bad Request |
| 인증 실패 / 토큰 만료 | 401 Unauthorized |
| 권한 없음 | 403 Forbidden |
| 리소스 없음 | 404 Not Found |
| 서버 오류 | 500 Internal Server Error |

---

## Unity에서 401 처리 (자동 토큰 갱신)

```csharp
public async Task<T> SendRequestAsync<T>(string url, string method, object body = null)
{
    var response = await MakeRequest(url, method, body);

    if (response.responseCode == 401)
    {
        // 토큰 갱신 시도
        var refreshed = await RefreshTokenAsync();
        if (!refreshed) { GoToLoginScene(); return default; }

        // 원래 요청 재시도
        response = await MakeRequest(url, method, body);
    }

    return JsonConvert.DeserializeObject<T>(response.downloadHandler.text);
}
```

---

## Swagger 문서화 규칙

모든 엔드포인트에 아래 어트리뷰트 필수:

```csharp
[HttpPost("draw")]
[Authorize]
[ProducesResponseType(typeof(DrawGachaResponse), StatusCodes.Status200OK)]
[ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status400BadRequest)]
[ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status401Unauthorized)]
[SwaggerOperation(Summary = "가챠 뽑기", Description = "뽑기 재화를 소모하여 캐릭터를 뽑습니다.")]
public async Task<DrawGachaResponse> Draw([FromBody] DrawGachaRequest request)
    => await _gachaService.DrawAsync(GetPlayerId(), request);
```

---

## 요청 DTO 유효성 검증

DataAnnotations로 필수 필드와 범위 제한:

```csharp
public class DrawGachaRequest
{
    [Required]
    [Range(1, 10, ErrorMessage = "뽑기 횟수는 1~10 사이여야 합니다.")]
    public int DrawCount { get; set; }
}

public class RegisterRequest
{
    [Required]
    [StringLength(50, MinimumLength = 4)]
    public string Username { get; set; } = null!;

    [Required]
    [StringLength(100, MinimumLength = 8)]
    public string Password { get; set; } = null!;
}
```
