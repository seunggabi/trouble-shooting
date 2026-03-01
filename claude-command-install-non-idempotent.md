# claude-command 설치 스크립트 재실행 시 hang 트러블슈팅

> 날짜: 2026-03-01
> 프로젝트: [claude-command](https://github.com/seunggabi/claude-command)
> 관련 버전: PR #39 수정

---

## 증상

`./install.sh`를 처음 실행하면 정상 설치되지만, **두 번째 실행부터 중간에 멈추고(hang) 응답이 없음**.

```bash
$ ./install.sh
Installing skills...
# → 여기서 멈춤 (Ctrl+C로만 종료 가능)
```

---

## 원인 1: `set -e` + non-zero exit code

### 문제

스크립트 상단에 `set -e`가 설정되어 있어 어떤 명령이든 non-zero exit code를 반환하면 스크립트가 즉시 종료됨.

이미 설치된 항목을 건너뛸 때 일부 명령이 non-zero를 반환하면 후속 단계가 모두 중단됨.

```bash
# BEFORE
set -e  # ← 문제: 어떤 실패도 스크립트 종료

((counter++))  # ← bash에서 값이 0이면 exit code 1 반환 → set -e에 의해 종료
```

### 수정

```bash
# AFTER: set -e 제거, 각 명령별 에러 처리
# set -e 제거

# ((var++)) 대신 산술 표현식 사용
var=$((var + 1))  # ← 항상 exit code 0 반환
```

---

## 원인 2: `npx skills add` interactive 프롬프트

### 문제

`npx skills add` 명령이 이미 설치된 skill이 있을 때 "덮어쓸까요?" 같은 **interactive 프롬프트**를 표시하고 사용자 입력을 기다림.

CI 환경이나 재설치 시 응답할 수 없어 hang이 발생.

```bash
# BEFORE: interactive 프롬프트 발생
npx skills add some-skill
# → "Skill already exists. Overwrite? [y/N]" ← 응답 없으면 무한 대기
```

### 수정

```bash
# AFTER: -y (자동 승인) 플래그 추가
npx skills add -y -g some-skill
```

---

## 원인 3: 이미 설치된 항목 미감지로 재설치 시도

### 문제

이미 동일한 설정이 있어도 매번 덮어쓰기를 시도해 불필요한 작업이 반복됨.

### 수정

```bash
# AFTER: 설치 전후 디렉토리 수 비교로 설치 여부 감지
before=$(ls ~/.agents/skills/ | wc -l)
npx skills add -y -g some-skill
after=$(ls ~/.agents/skills/ | wc -l)

if [ "$after" -gt "$before" ]; then
  echo "✓ installed"
else
  echo "⟳ skipped"
fi
```

```bash
# settings.json: diff로 변경 여부 확인 후 스킵
if diff -q current.json new.json > /dev/null 2>&1; then
  echo "⟳ skipped (already up to date)"
else
  cp new.json current.json
  echo "↑ updated"
fi
```

---

## 결과

| 상황 | 수정 전 | 수정 후 |
|------|--------|--------|
| 첫 실행 | ✅ 설치됨 | ✅ 설치됨 (`✓ installed`) |
| 재실행 | ❌ hang | ✅ 즉시 완료 (`⟳ skipped`) |
| 업데이트 후 재실행 | ❌ hang | ✅ 자동 업데이트 (`↑ updated`) |

---

## 검증 방법

```bash
# 1회차
./install.sh
# → ✓ installed (모든 항목)

# 2회차 (idempotent 확인)
./install.sh
# → ⟳ skipped (모든 항목, hang 없이 즉시 완료)

# --global 옵션도 동일
./install.sh --global
# → ⟳ skipped
```

---

## 관련 PR

| PR | 내용 |
|----|------|
| #39 | install.sh 멱등성(idempotent) 보장, interactive hang 수정 |
