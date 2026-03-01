# win-ops Task Scheduler 실행 시 PowerShell 창이 닫히지 않는 트러블슈팅

> 날짜: 2026-03-01
> 프로젝트: [win-ops](https://github.com/seunggabi/win-ops)
> 관련 버전: v0.7.7 → v0.7.8

---

## 증상

win-ops를 Task Scheduler에 등록한 후 자동 실행될 때마다 PowerShell 창이 화면에 나타나 그대로 남아 있음.

```
[작업 스케줄러 실행 시]
- PowerShell 창이 열림
- 스크립트 실행 완료 후에도 창이 닫히지 않음
- 매 실행마다 반복 발생
```

수동으로 실행하면 정상이나, Task Scheduler 자동 실행 시에만 발생.

---

## 원인: pwsh 실행 인수에 `-WindowStyle Hidden` 미적용

### 문제

`WinOpsTask.xml` (작업 스케줄러 등록 XML)의 `Arguments` 항목과 `TaskScheduler.psm1`의 pwsh 호출 부분에 창 숨김 옵션이 없었음.

```xml
<!-- BEFORE: WinOpsTask.xml -->
<Arguments>-ExecutionPolicy Bypass -File "C:\win-ops\win-ops.ps1"</Arguments>
```

```powershell
# BEFORE: TaskScheduler.psm1
$action = New-ScheduledTaskAction -Execute "pwsh" `
    -Argument "-ExecutionPolicy Bypass -File `"$scriptPath`""
```

Task Scheduler는 기본적으로 새 콘솔 창을 생성하며, `-WindowStyle Hidden`이 없으면 창이 그대로 화면에 표시됨.

---

## 수정

### `-WindowStyle Hidden -NonInteractive` 추가

```xml
<!-- AFTER: WinOpsTask.xml -->
<Arguments>-WindowStyle Hidden -NonInteractive -ExecutionPolicy Bypass -File "C:\win-ops\win-ops.ps1"</Arguments>
```

```powershell
# AFTER: TaskScheduler.psm1
$action = New-ScheduledTaskAction -Execute "pwsh" `
    -Argument "-WindowStyle Hidden -NonInteractive -ExecutionPolicy Bypass -File `"$scriptPath`""
```

| 옵션 | 역할 |
|------|------|
| `-WindowStyle Hidden` | 콘솔 창을 생성하지 않음 |
| `-NonInteractive` | 사용자 입력 프롬프트 비활성화 (자동화 환경에서 hang 방지) |

### 재설치 시 자동 업데이트 처리

기존에는 이미 등록된 작업 스케줄러 태스크가 있을 때 `-InstallScheduledTask` 플래그를 명시해야 업데이트됐음. 수정 후 `install.ps1`이 기존 태스크를 자동 감지해 업데이트.

```powershell
# AFTER: install.ps1
$existingTask = Get-ScheduledTask -TaskName "WinOps" -ErrorAction SilentlyContinue
if ($existingTask) {
    # 기존 태스크 자동 업데이트 (플래그 없이도 동작)
    Unregister-ScheduledTask -TaskName "WinOps" -Confirm:$false
}
# 태스크 새로 등록
Register-ScheduledTask ...
```

---

## 검증 방법

```powershell
# 1. 태스크 등록
.\install.ps1

# 2. Arguments에 -WindowStyle Hidden 포함 여부 확인
(Get-ScheduledTask -TaskName "WinOps").Actions.Arguments
# ✅ -WindowStyle Hidden -NonInteractive -ExecutionPolicy Bypass -File "..."

# 3. 태스크 수동 실행 후 창 미표시 확인
Start-ScheduledTask -TaskName "WinOps"
# ✅ PowerShell 창 미표시
```

---

## 관련 PR

| 버전 | PR | 내용 |
|------|----|------|
| v0.7.8 | #39 | Task Scheduler 실행 시 PowerShell 창 숨김 처리 |
