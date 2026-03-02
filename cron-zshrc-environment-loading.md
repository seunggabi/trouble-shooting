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

## 해결: `/bin/zsh -c`로 실행

```bash
# crontab -e
* * * * * source ~/.zshrc && /path/to/script.sh
```

**`/bin/zsh -c`는 자동으로 `~/.zshrc`를 로드합니다.**
- `source ~/.zshrc` 명시적 호출 불필요
- `SHELL=/bin/zsh` 설정 불필요
- 래퍼 스크립트 불필요

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
- Crontab에서 환경 변수를 설정할 수 없으므로, `source ~/.zshrc && ...`로 직접 호출

## 실제 예제

```bash
# crontab -e

# 매일 자정에 실행
0 0 * * * source ~/.zshrc && ~/projects/script.sh

# 1시간마다 실행
0 * * * * source ~/.zshrc && ~/projects/task.sh >> ~/logs/task.log 2>&1

# 매시 33분에 실행 (cd 필요시)
33 * * * * source ~/.zshrc && cd ~/projects && npm run task
```

**이것이 전부입니다.** `source ~/.zshrc`, `SHELL=/bin/zsh`, 래퍼 스크립트, 추가 env 변수 모두 불필요.

## 트러블슈팅

환경 변수가 제대로 로드되는지 확인:

```bash
# crontab -e
# 환경 로그 저장
0 0 * * * source ~/.zshrc && env > /tmp/cron-env.log 2>&1

# PATH 확인
0 0 * * * source ~/.zshrc && echo "PATH=$PATH" >> /tmp/cron-test.log
```

스크립트 실행 권한 확인:

```bash
chmod +x ~/projects/script.sh
```

## 요약

| 항목 | 방법 |
|------|--------|
| **Cron에서 zshrc 자동 로드** | `source ~/.zshrc && script` |
| **필요 없는 것** | `source ~/.zshrc`, `SHELL=/bin/zsh`, 래퍼, env 설정 |

## 핵심 교훈

**유일한 방법:**
```bash
* * * * * source ~/.zshrc && /path/to/script.sh
```

**특징:**
- `/bin/zsh -c`는 자동으로 `~/.zshrc` 로드
- `source ~/.zshrc` 불필요
- `SHELL=/bin/zsh` 설정 불필요
- 래퍼 스크립트 불필요
- crontab env 변수 설정 불필요
