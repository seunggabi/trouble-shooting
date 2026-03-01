# Supabase Free Tier 자동 일시정지 방지 (ping-db)

## 증상

```
Your project revu(ID: ...) was one of those and it is scheduled to be paused in a couple of days.
Your project is not currently paused, but if it continues not to receive sufficient activity,
it will be paused automatically.
```

Supabase로부터 위와 같은 메일을 수신하고, 이후 프로젝트가 자동으로 pause됨.

## 원인

Supabase free tier는 **7일 이상 DB 활동이 없으면 프로젝트를 자동 일시정지**함.

- 일시정지 후 90일 이내에 대시보드에서 unpause 가능
- 90일 초과 시 unpause 불가 (데이터 다운로드만 가능)
- 방지하려면 Pro 플랜으로 업그레이드하거나, 주기적으로 DB에 쿼리를 보내야 함

## 해결: ping-db GitHub Actions 자동화

### 1. GitHub Actions workflow 생성

`.github/workflows/ping-db.yml`

```yaml
name: Ping Supabase DB

on:
  schedule:
    - cron: '0 0 */5 * *'  # 5일마다 자정 실행 (7일 기한 전)
  workflow_dispatch:         # 수동 실행 허용

jobs:
  ping:
    runs-on: ubuntu-latest
    steps:
      - name: Ping Supabase
        run: |
          curl -s -o /dev/null -w "%{http_code}" \
            "${{ secrets.SUPABASE_URL }}/rest/v1/?select=1" \
            -H "apikey: ${{ secrets.SUPABASE_ANON_KEY }}" \
            -H "Authorization: Bearer ${{ secrets.SUPABASE_ANON_KEY }}"
```

### 2. GitHub Secrets 설정

저장소 → Settings → Secrets and variables → Actions에 추가:

| Secret 이름 | 값 |
|------------|-----|
| `SUPABASE_URL` | `https://<project-id>.supabase.co` |
| `SUPABASE_ANON_KEY` | Supabase 대시보드 → Settings → API → anon key |

### 3. DB 쿼리 방식 (REST API 대신 직접 쿼리)

```yaml
      - name: Ping via DB query
        env:
          SUPABASE_DB_URL: ${{ secrets.SUPABASE_DB_URL }}
        run: |
          npx supabase db execute --db-url "$SUPABASE_DB_URL" "SELECT 1;"
```

## 대안: Supabase Edge Function + pg_cron

### Edge Function 생성

```typescript
// supabase/functions/ping/index.ts
import { createClient } from 'jsr:@supabase/supabase-js@2'

Deno.serve(async () => {
  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!
  )
  const { error } = await supabase.from('_ping').select('1').limit(1)
  return new Response(JSON.stringify({ ok: !error }), {
    headers: { 'Content-Type': 'application/json' }
  })
})
```

### pg_cron으로 주기 실행 (SQL Editor)

```sql
-- pg_cron 활성화 (Extensions에서 먼저 활성화 필요)
select cron.schedule(
  'ping-daily',
  '0 12 * * *',   -- 매일 낮 12시
  $$ select 1 $$
);
```

## 대안: Uptime Robot 무료 플랜

1. [uptimerobot.com](https://uptimerobot.com) 가입
2. New Monitor → HTTP(s)
3. URL: `https://<project-id>.supabase.co/rest/v1/?select=1`
4. Monitoring Interval: 5분 (무료 플랜 최소)
5. Header 추가: `apikey: <anon-key>`

## 교훈

- GitHub Actions cron은 **5일 주기**가 적절 (7일 기한보다 여유 확보)
- `workflow_dispatch` 추가로 수동 ping 가능하게 설정
- free tier 프로젝트가 여러 개라면 하나의 workflow에서 모두 ping
- 장기적으로는 Pro 플랜 전환이 관리 부담 없음
