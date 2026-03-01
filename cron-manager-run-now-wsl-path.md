# cron-manager 즉시실행 버튼이 설치된 앱에서만 안 되는 트러블슈팅

> 날짜: 2026-03-02
> 프로젝트: [cron-manager](https://github.com/seunggabi/cron-manager)
> 관련 버전: v0.22.1

---

## 증상

즉시실행(Run Now) 버튼이 개발 빌드(WSL2에서 `npm run dev`)에서는 정상 동작하지만, Windows에 설치된 앱에서는 아무 반응이 없음.

```
[개발 빌드] ✅ 즉시실행 정상 동작
[설치된 앱] ❌ 버튼 클릭해도 아무 반응 없음 (스피너도 안 뜸)
```

DevTools Console에서는 에러 없음. 메인 프로세스 로그 확인 시 아래 에러 발생:

```
stderr: "'wsl'은(는) 내부 또는 외부 명령, 실행할 수 있는 프로그램, 또는 배치 파일이 아닙니다."
exitCode: 1
```

---

## 원인 1: `execAsync`에 전달하는 `env`에서 PATH 누락

### 문제

`runJob`에서 `mergedEnv`를 구성할 때 `process.env` 전체 대신 최소한의 변수만 복사.
`job.env`나 `globalEnv`에 `PATH`가 설정된 경우 기존 PATH가 완전히 대체됨.

```ts
// BEFORE: C:\Windows\System32 누락 가능
const mergedEnv = {
  PATH: process.env.PATH || 'C:\\Windows\\System32',
  HOME: process.env.HOME || process.env.USERPROFILE || '',
  ...globalEnv,   // PATH를 덮어쓸 수 있음
  ...job.env,     // PATH를 덮어쓸 수 있음
};
```

### 수정

```ts
// AFTER: process.env 전체를 기반으로 상속
const mergedEnv = {
  ...process.env,
  HOME: process.env.HOME || process.env.USERPROFILE || '',
  ...globalEnv,
  ...job.env,
};
```

---

## 원인 2: `wsl` 상대 경로 사용 → 패키징된 앱에서 찾지 못함

### 문제

`wsl bash -c "command"` 실행 시 `wsl`을 PATH에서 찾는데, 패키징된 Electron 앱의 실행 환경에서는 `C:\Windows\System32`가 PATH에 없어 `wsl.exe`를 찾지 못함.

```ts
// BEFORE
cmd = `wsl bash -c ${JSON.stringify(bashCmd)}`;
```

### 수정

`SystemRoot` 환경변수를 이용해 `wsl.exe` 절대 경로 사용.

```ts
// AFTER: wslExe getter 추가
private get wslExe(): string {
  const sysRoot = process.env.SystemRoot || process.env.SYSTEMROOT || 'C:\\Windows';
  return `"${sysRoot}\\System32\\wsl.exe"`;
}

// runJob에서 절대 경로 사용
cmd = `${this.wslExe} bash -c ${JSON.stringify(bashCmd)}`;
```

---

## 원인 3: `workingDir`가 Linux 경로일 때 `cwd` 옵션에 직접 전달

### 문제

`job.workingDir`가 WSL 경로(`~/...`, `/home/...`)일 때 Windows의 `execAsync` `cwd` 옵션에 그대로 전달하면 경로를 찾지 못함.

```ts
// BEFORE
const { stdout } = await execAsync(cmd, {
  cwd: job.workingDir || undefined,  // Linux 경로를 Windows cwd로 전달 → 에러
});
```

### 수정

Windows에서는 `workingDir`를 bash 명령 안의 `cd`로 처리하고, `cwd`는 Windows 네이티브 경로일 때만 전달.

```ts
// AFTER
const isWslPath = (p: string) => p.startsWith('~') || p.startsWith('/');

if (this.isWindows) {
  const bashCmd = job.workingDir
    ? `cd ${this.shellEscape(job.workingDir)} && ${job.command}`
    : job.command;
  cmd = `${this.wslExe} bash -c ${JSON.stringify(bashCmd)}`;
  execCwd = (job.workingDir && !isWslPath(job.workingDir))
    ? job.workingDir
    : undefined;
} else {
  cmd = job.command;
  execCwd = job.workingDir || undefined;
}
```

---

## 왜 개발 빌드에서는 됐나?

개발 빌드는 WSL2(Linux) 환경에서 실행됨.

| 환경 | `process.platform` | `isWindows` | 실행 방식 |
|------|-------------------|-------------|---------|
| 개발 빌드 (WSL2) | `linux` | `false` | `job.command` 직접 실행, Linux PATH 정상 |
| 설치된 앱 (Windows) | `win32` | `true` | `wsl bash -c ...` 실행 → PATH 문제 발생 |

---

## 관련 커밋

| 내용 | 파일 |
|------|------|
| `wslExe` getter로 절대 경로 사용 | `src/main/services/crontab.service.ts` |
| `mergedEnv`를 `process.env` 전체 기반으로 변경 | `src/main/services/crontab.service.ts` |
| WSL 경로 `workingDir`를 bash 명령 내 `cd`로 처리 | `src/main/services/crontab.service.ts` |
