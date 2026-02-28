# ai-heatmap 멀티머신 머지 버그 트러블슈팅

> 날짜: 2026-03-01
> 프로젝트: [ai-heatmap](https://github.com/seunggabi/ai-heatmap)
> 관련 버전: v1.17.9 ~ v1.17.12

---

## 증상

`npx ai-heatmap update` 실행 후 두 머신(`T4MV6DCPVD`, `bee321a874b9`)의 데이터가 합산되지 않고, 현재 머신 데이터만 `data.json`에 반영됨.

```
No existing machine data files found (first run).
Merging 1 file(s): data-T4MV6DCPVD.json
```

---

## 원인 1: ccusage 중복 날짜 덮어쓰기 (v1.17.9)

### 문제

`ccusage daily --json` 결과에 같은 날짜가 두 개 이상 있을 경우, `new Map(daily.map(...))` 방식으로 Map을 생성하면 **나중 항목이 이전 항목을 덮어씀**.

```js
// BEFORE: 중복 날짜 시 마지막 값만 남음
const dataMap = new Map(daily.map((d) => [d.date, d]));
```

### 수정

```js
// AFTER: 중복 날짜 누적 합산
const dataMap = new Map();
for (const d of daily) {
  if (!dataMap.has(d.date)) {
    dataMap.set(d.date, { ...d });
  } else {
    const existing = dataMap.get(d.date);
    existing.totalCost = (existing.totalCost ?? 0) + (d.totalCost ?? 0);
    // inputTokens, outputTokens, totalTokens, cacheTokens 등도 동일하게 누적
  }
}
```

---

## 원인 2: 파일 fetch 루프의 단일 try-catch (v1.17.11)

### 문제

다른 머신의 `data-*.json` 파일을 GitHub에서 다운로드하는 루프 전체를 하나의 `try-catch`가 감싸고 있었음. 파일 하나라도 fetch 실패하면 나머지 파일도 건너뜀.

```js
// BEFORE: 루프 전체가 하나의 try-catch
try {
  const raw = execSync(`gh api repos/${repo}/contents/public ...`);
  const files = JSON.parse(raw);
  for (const file of files) {
    const content = execSync(`gh api repos/${repo}/contents/${file.path} ...`);
    // ...
  }
} catch {
  console.log("  No existing machine data files found (first run).");
}
```

### 수정

```js
// AFTER: 디렉토리 목록 조회와 파일별 fetch를 분리
let remoteFiles = [];
try {
  const raw = execSync(`gh api repos/${repo}/contents/public ...`);
  remoteFiles = JSON.parse(raw);
} catch {
  console.log("  No existing machine data files found (first run).");
}
for (const file of remoteFiles) {
  try {
    const content = execSync(`gh api repos/${repo}/contents/${file.path} ...`);
    // ...
  } catch {
    console.log(`  Failed to fetch ${file.name}, skipping.`);
  }
}
```

---

## 원인 3: jq invalid escape sequence (v1.17.12) ← 핵심 버그

### 문제

`bin/cli.mjs`에서 `execSync`로 `gh api` 명령을 실행할 때 jq 표현식에 `\\.json`을 사용했음.

```js
// JS template literal
`gh api ... --jq '[.[] | select(.name | test("^data-.+\\.json$"))]'`
```

**JS → 쉘 변환 과정:**

| 단계 | 값 |
|------|-----|
| JS template literal `\\.` | `\.` (JS에서 `\\` = 역슬래시 1개) |
| 쉘에 전달되는 문자열 | `test("^data-.+\.json$")` |
| jq 파싱 결과 | ❌ `invalid escape sequence "\."` |

jq는 문자열 리터럴 안에서 `\.`를 허용하지 않음 (유효한 jq 이스케이프: `\\`, `\"`, `\n`, `\t` 등).

### 오류 메시지

```
failed to parse jq expression (line 1, column 37)
    [.[] | select(.name | test("^data-.+\.json$"))]
                                        ^  invalid escape sequence "\." in string literal
```

### 수정

```js
// AFTER: [.] 사용 (character class로 리터럴 점 매칭)
`gh api ... --jq '[.[] | select(.name | test("^data-.+[.]json$"))]'`
```

**왜 `[.]`인가?**
- jq 정규표현식에서 `[.]`는 `.` 문자 하나만 매칭하는 character class
- 역슬래시 이스케이프 없이 리터럴 점을 표현하는 가장 안전한 방법

### 교훈: JS → 쉘 → jq 이스케이프 체인 주의

```
JS source  →  JS string  →  shell arg  →  jq regex
"\\\\.json"  →  "\\.json"  →  "\.json" ?  →  jq error
"\\\\.json"  →  "\\.json"  as jq string  →  \.  in regex ✓ (복잡)
"[.]json"   →  "[.]json"  →  "[.]json"   →  [.] in regex ✓ (권장)
```

---

## 검증 방법

```bash
# 직접 실행 시 성공 여부 확인
gh api repos/seunggabi/seunggabi-ai-heatmap/contents/public \
  --jq '[.[] | select(.name | test("^data-.+[.]json$"))]'

# Node.js execSync 컨텍스트에서도 확인
node -e "
const { execSync } = require('child_process');
const raw = execSync(
  \`gh api repos/seunggabi/seunggabi-ai-heatmap/contents/public --jq '[.[] | select(.name | test(\"^data-.+[.]json\$\"))]'\`,
  { encoding: 'utf-8' }
);
console.log(JSON.parse(raw).map(f => f.name));
"
```

---

## 날짜별 합산 검증

```bash
node -e "
const { readFileSync } = require('fs');
const date = '2026-02-21';
const files = ['data-T4MV6DCPVD.json', 'data-bee321a874b9.json']
  .map(f => '/Users/seunggabi/sg/seunggabi-ai-heatmap/public/' + f);
let total = 0;
for (const f of files) {
  const entry = JSON.parse(readFileSync(f, 'utf-8')).find(e => e.date === date);
  console.log(f.split('/').pop(), ':', entry?.count ?? 0);
  total += entry?.count ?? 0;
}
console.log('합산:', Math.round(total * 100) / 100);
"
# 2026-02-21: T4MV6DCPVD=133.12, bee321a874b9=5.72, 합산=138.84
```

---

## 관련 PR

| 버전 | PR | 내용 |
|------|----|------|
| v1.17.9 | #94 | ccusage 중복 날짜 누적 합산 |
| v1.17.11 | #100 | 파일별 try-catch 분리 |
| v1.17.12 | #102 | jq invalid escape `\\.json` → `[.]json` |
