# cron-manager run-now 실행 오류 트러블슈팅

> 날짜: 2026-03-01
> 프로젝트: [cron-manager](https://github.com/seunggabi/cron-manager)
> 관련 버전: PR #92, #94

---

## 증상 1: 앱 강제 종료 후 run-now 버튼이 영구 비활성화됨

앱이 예기치 않게 종료(크래시, 강제 종료)된 뒤 재시작하면 특정 job의 **Run Now 버튼이 계속 비활성화**됨.

```
[Run Now 클릭]
→ 버튼 비활성화 (회색)
→ 아무 반응 없음
→ 재시작해도 동일
```

---

## 원인 1: stale lock 파일

### 문제

`run-now` 실행 시 중복 실행 방지를 위해 `~/.cron-manager/locks/{jobId}.lock` 파일을 생성함. 정상 종료 시에는 lock 파일을 삭제하지만, **앱이 크래시되면 lock 파일이 남아 있음**.

다음 실행 시 lock 파일이 존재하므로 "이미 실행 중"으로 판단해 버튼이 차단됨.

```
1회차 실행 → lock 파일 생성 (~/.cron-manager/locks/job-abc.lock)
앱 크래시  → lock 파일 삭제 못 함
재시작      → lock 파일 존재 → "실행 중" 판단 → Run Now 비활성화 ♻️
```

### 수정

앱 시작 시 모든 lock 파일을 초기화.

```ts
// AFTER: main process 시작 시 lock 디렉토리 전체 초기화
async function clearStaleLocks() {
  const lockDir = path.join(os.homedir(), '.cron-manager', 'locks');
  try {
    const files = await fs.readdir(lockDir);
    await Promise.all(files.map(f => fs.unlink(path.join(lockDir, f))));
  } catch {
    // 디렉토리 없으면 무시
  }
}

app.on('ready', async () => {
  await clearStaleLocks();  // 시작 시 항상 초기화
  // ...
});
```

앱이 방금 시작됐다면 어떤 job도 실행 중일 수 없으므로 전체 초기화가 안전함.

---

## 증상 2: `~/` 경로를 사용하는 명령이 run-now에서 실패

cron 스케줄로 실행하면 정상이지만, Run Now 버튼으로 즉시 실행하면 `~/` 경로를 사용하는 명령이 exit code 2로 실패함.

```bash
# cron 스케줄: ✅ 정상 동작
echo "$(date)" >> ~/logs/echo.log 2>&1

# Run Now: ❌ exit code 2
echo "$(date)" >> ~/logs/echo.log 2>&1
# → /bin/sh: /root/logs/echo.log: No such file or directory
```

---

## 원인 2: run-now 실행 환경에 HOME 누락

### 문제

`runJob()` 함수의 환경변수 병합 로직에서 `HOME`이 누락됨. `~/`는 쉘이 `HOME` 환경변수를 기반으로 확장하므로, `HOME`이 없으면 `/root` 등 잘못된 경로로 확장됨.

```ts
// BEFORE: HOME 없음
const mergedEnv = {
  PATH: process.env.PATH,
  // HOME 누락!
  ...job.env,
};
```

### 수정

```ts
// AFTER: HOME 추가 (Linux/Windows(WSL) 모두 지원)
const mergedEnv = {
  PATH: process.env.PATH,
  HOME: process.env.HOME || process.env.USERPROFILE || '',
  ...job.env,
};
```

| 환경 | 변수 |
|------|------|
| Linux / macOS | `process.env.HOME` |
| Windows / WSL | `process.env.USERPROFILE` |

---

## 검증 방법

```bash
# 증상 1 검증
1. Run Now 실행 중 앱 강제 종료 (Cmd+Q)
2. 앱 재시작
3. 같은 job Run Now 클릭 → ✅ 정상 실행

# 증상 2 검증
1. ~/logs/echo.log 를 쓰는 job 등록
2. Run Now 클릭 → ✅ exit code 0, 파일 생성 확인
```

---

## 관련 PR

| PR | 내용 |
|----|------|
| #92 | run-now 실행 환경에 HOME 환경변수 추가 |
| #94 | 앱 시작 시 stale lock 파일 초기화 |
