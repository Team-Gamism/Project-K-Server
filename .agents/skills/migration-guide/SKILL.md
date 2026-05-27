| name | migration-guide |
| --- | --- |
| description | EF Core Code First 마이그레이션 가이드 — 엔티티 변경 시 영향 분석, 올바른 변경 순서 (Entity → Configuration → Repository → Service → Test), 컬럼 삭제 2단계 배포 전략. |

# DB Migration Guide

## 변경 순서

엔티티 또는 DB 스키마 변경 시 반드시 아래 순서를 지킵니다:

1. **Entity 수정** — `Domain/Entities/` 내 엔티티 클래스
2. **EntityTypeConfiguration 수정** — `Infrastructure/Persistence/Configurations/`
3. **Migration 생성** — `dotnet ef migrations add {MigrationName}`
4. **Repository 수정** — 쿼리 조정 (Include, 필터 등)
5. **Service 수정** — 비즈니스 로직 반영
6. **DTO 수정** — 요청/응답 DTO 반영
7. **테스트 수정** — Entity, Service, Controller 테스트 갱신

---

## Migration 명령어

```bash
# 마이그레이션 생성
dotnet ef migrations add {MigrationName} --project src/{ProjectName}

# DB 적용
dotnet ef database update --project src/{ProjectName}

# 마이그레이션 목록 확인
dotnet ef migrations list --project src/{ProjectName}

# 특정 마이그레이션으로 롤백
dotnet ef database update {PreviousMigrationName} --project src/{ProjectName}
```

---

## 엔티티 변경 체크리스트

- [ ] 기존 데이터에 미치는 영향 분석
- [ ] 마이그레이션 스크립트 필요 여부 결정
  - 컬럼 추가: `DEFAULT` 값 지정 필요?
  - 컬럼 타입 변경: 데이터 손실 위험?
  - 컬럼 삭제: 2단계 배포 전략 사용
- [ ] 롤백 전략 수립
- [ ] 테스트 데이터 준비

---

## 컬럼 추가 패턴

```csharp
// Entity에 새 프로퍼티 추가
public class Player
{
    // 기존 프로퍼티
    public long Id { get; private set; }
    public string Username { get; private set; } = null!;

    // 새 컬럼 추가 (nullable 또는 기본값 필수)
    public string? Nickname { get; private set; }  // nullable
}
```

```csharp
// Configuration에 추가
builder.Property(p => p.Nickname)
    .HasColumnName("nickname")
    .HasMaxLength(30)
    .IsRequired(false); // nullable
```

```
dotnet ef migrations add AddPlayerNickname
```

---

## 컬럼 삭제 전략 (2단계 배포)

컬럼 삭제는 한 번에 하지 않음. 2단계로 나눔:

**Phase 1 — Deprecate (코드에서 제거)**
```csharp
// 1. Configuration에서 컬럼 매핑 제거
// 2. Entity 프로퍼티 제거
// 3. 모든 참조 코드 제거
// 4. 마이그레이션 생성 및 배포
dotnet ef migrations add DeprecatePlayerLegacyField
```

**Phase 2 — Delete (DB에서 제거)**
```
// Phase 1 배포 후 충분한 시간(예: 다음 스프린트) 경과 후
dotnet ef migrations add DropPlayerLegacyField
```

---

## DDL 전략

| 환경 | 전략 |
| --- | --- |
| 개발 | `dotnet ef database update` 수동 적용 |
| 프로덕션 | 수동 마이그레이션 (`dotnet ef database update`) |

`EnsureCreated()` 는 개발 초기에만 허용 — 마이그레이션과 함께 사용 **금지**

---

## 마이그레이션 네이밍 규칙

| 작업 | 이름 형식 | 예시 |
| --- | --- | --- |
| 테이블 생성 | `Create{TableName}Table` | `CreatePlayersTable` |
| 컬럼 추가 | `Add{Column}To{Table}` | `AddNicknameToPlayers` |
| 컬럼 삭제 | `Drop{Column}From{Table}` | `DropLegacyFieldFromPlayers` |
| 인덱스 추가 | `AddIndexOn{Column}` | `AddIndexOnPlayerUsername` |
| 관계 추가 | `Add{Relation}Relation` | `AddPlayerGachaRelation` |
