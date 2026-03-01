# Supabase Migration Version Conflict

## 증상

```
Remote migration versions not found in local migrations directory.
supabase migration repair --status reverted <VERSION>
```

```
ERROR: duplicate key value violates unique constraint "schema_migrations_pkey"
Key (version)=(20260218) already exists.
```

```
ERROR: policy "..." already exists (SQLSTATE 42710)
```

## 원인

1. **같은 날짜의 파일 두 개** — `20260218_github_sponsors.sql`, `20260218_usage_cleanup_7days.sql` 처럼 날짜 prefix가 동일하면 version key 충돌
2. **고아(orphaned) remote 버전** — 대시보드에서 직접 실행한 SQL이 schema_migrations에 다른 version으로 기록됨
3. **push 실패 시 롤백** — 하나의 migration 실패 시 해당 push 배치 전체가 롤백되어 schema_migrations 기록이 사라짐 → 다음 push에서 또 충돌
4. **비멱등 DDL** — `CREATE POLICY` 등 `IF NOT EXISTS` 미지원 구문

## 해결

### 1. 중복 버전 파일 → 타임스탬프 분리

```bash
mv 20260218_usage_cleanup_7days.sql 20260218100000_usage_cleanup_7days.sql
```

### 2. 고아 remote 버전 제거

```bash
npx supabase migration repair --status reverted <VERSION>
# 복수 지정 가능
npx supabase migration repair --status reverted 20260205 20260218
```

### 3. 비멱등 DDL 수정

```sql
-- before
CREATE POLICY "..." ON public.foo FOR ALL USING (true);

-- after
DROP POLICY IF EXISTS "..." ON public.foo;
CREATE POLICY "..." ON public.foo FOR ALL USING (true);
```

### 4. 순서 어긋난 migration 포함하여 push

```bash
npx supabase db push --yes --include-all
```

### 5. 실패 반복 시 패턴

push 실패 → 롤백 → 고아 버전 재등장 → repair → push 반복.
매 push 실패 후 `migration list`로 상태 확인 후 repair 진행.

```bash
npx supabase migration list
npx supabase migration repair --status reverted <VERSION>
npx supabase db push --yes --include-all
```

## 교훈

- Migration 파일은 **날짜+시간(yyyyMMddHHmmss)** 형식으로 생성해 중복 방지
- `CREATE POLICY`, `CREATE TRIGGER` 등 비멱등 DDL은 `DROP ... IF EXISTS` 선행
- `db push`는 `--project-ref` 플래그 미지원 → 먼저 `supabase link` 필요
