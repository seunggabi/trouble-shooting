# Cron에서 ~/.zshrc 환경 변수 로드하기

## 증상

Cron job에서 스크립트를 실행할 때 `~/.zshrc`에 정의한 환경 변수, alias, 함수를 인식하지 못함.

```bash
$ cat > /tmp/test.sh << 'EOF'
#!/bin/bash
echo "PATH: $PATH"
echo "Custom var: $MY_VAR"
EOF

$ crontab -e
# 다음 추가
* * * * * /tmp/test.sh

# 결과: ~/.zshrc의 설정이 로드되지 않음
```

## 원인

- Cron job은 **non-login, non-interactive shell**로 실행됨
- Bash를 기본 쉘로 설정했을 때: `~/.bashrc` 또는 `~/.profile`만 로드 가능
- Zsh를 기본 쉘로 변경했을 때: Cron은 여전히 **Bash 또는 /bin/sh**로 스크립트를 실행할 수 있음
- `.zshenv` vs `.zshrc`:
  - `.zshenv`: 모든 Zsh 인스턴스에서 로드 (login/non-login 상관없음)
  - `.zshrc`: interactive shell에서만 로드

## 해결: Cron에서 Zsh 환경 로드

### 권장: Cron에서 명시적으로 Zsh 지정 (가장 간단)

```bash
# crontab -e
* * * * * /bin/zsh -c 'source ~/.zshrc && /path/to/script.sh'
```

이것이 유일하게 필요한 방식입니다. `SHELL=/bin/zsh` 설정이나 래퍼 스크립트는 불필요합니다.

### 환경 변수 설정

환경 변수들은 모두 `~/.zshrc`에 정의합니다:

```bash
# ~/.zshrc 끝에 추가
export MY_VAR="value"
export API_KEY="secret"
export PATH="/usr/local/bin:$PATH"
```

**중요 사항:**
- `.zshenv` 설정은 불필요
- Crontab 상단의 `SHELL=/bin/zsh`도 불필요
- Crontab에서 환경 변수를 설정할 수 없으므로, `/bin/zsh -c 'source ~/.zshrc && ...'`로 직접 호출

## 실제 예제

```bash
# crontab -e

# 매일 자정에 실행
0 0 * * * /bin/zsh -c 'source ~/.zshrc && ~/projects/script.sh'

# 1시간마다 실행
0 * * * * /bin/zsh -c 'source ~/.zshrc && ~/projects/task.sh >> ~/logs/task.log 2>&1'

# 매시 33분에 실행
33 * * * * /bin/zsh -c 'source ~/.zshrc && cd ~/projects && npm run task'
```

**이것이 모든 것입니다.** `SHELL=/bin/zsh`, 래퍼 스크립트, 추가 env 변수 설정 불필요.

## 트러블슈팅

환경 변수가 제대로 로드되는지 확인:

```bash
# crontab -e
# 환경 로그 저장
0 0 * * * /bin/zsh -c 'source ~/.zshrc && env > /tmp/cron-env.log 2>&1'

# 또는 직접 확인
0 0 * * * /bin/zsh -c 'source ~/.zshrc && echo "PATH=$PATH" >> /tmp/cron-test.log'
```

스크립트 실행 권한 확인:

```bash
chmod +x ~/sg/blog/sh/key.sh
```

## 요약

| 항목 | 답변 |
|------|--------|
| **Cron에서 zshrc 로드** | `/bin/zsh -c 'source ~/.zshrc && script'` |
| **.zshenv 설정** | **불필요** — `.zshrc`로 충분 |
| **`SHELL=/bin/zsh` crontab 설정** | **불필요** |
| **래퍼 스크립트** | **불필요** |
| **Cron env 변수 설정** | **불필요** |

## 핵심 교훈

**가장 간단한 방법:**
```bash
* * * * * /bin/zsh -c 'source ~/.zshrc && /path/to/script.sh'
```

**제거할 것들:**
- `SHELL=/bin/zsh` — crontab에서 불필요
- 래퍼 스크립트 — `/bin/zsh -c` 직접 사용하면 됨
- `.zshenv` 설정 — `.zshrc`로 충분
- cron env 변수 설정 — 없어도 작동

**확인하기:**
```bash
# 환경 변수 로드 확인
* * * * * /bin/zsh -c 'source ~/.zshrc && env > /tmp/cron-env.log 2>&1'
```
