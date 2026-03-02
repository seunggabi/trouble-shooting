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

### 1. 스크립트를 Zsh로 명시적으로 실행

**방법 1: Shebang 변경 (권장)**

```bash
#!/bin/zsh -c
# 이 줄이 있으면 자동으로 zsh으로 실행되고 ~/.zshrc 로드
source ~/.zshrc
echo "PATH: $PATH"
```

**방법 2: Cron에서 명시적으로 지정**

```bash
# crontab -e
* * * * * /bin/zsh -c 'source ~/.zshrc && /path/to/script.sh'
```

**방법 3: 래퍼 스크립트로 감싸기**

```bash
# ~/.local/bin/run-with-zsh
#!/bin/bash
/bin/zsh -c "source ~/.zshrc && $@"

# crontab -e
* * * * * ~/.local/bin/run-with-zsh /path/to/script.sh
```

### 2. ~/.zshrc에 Cron 환경 변수 추가

```bash
# ~/.zshrc 끝에 추가
# 환경 변수들
export MY_VAR="value"
export API_KEY="secret"
export PATH="/usr/local/bin:$PATH"
```

**`.zshenv` 설정은 불필요** — `.zshrc`에 모든 설정을 유지하면 됨.

### 3. Cron 환경 변수 설정은 불필요

Crontab 상단에서 환경 변수를 설정할 수 없음:

```bash
# ❌ 작동 안 함
MY_VAR=value
* * * * * /path/to/script.sh

# ✅ 대신 ~/.zshrc에 설정하거나, 스크립트 내에서 정의
* * * * * /bin/zsh -c 'source ~/.zshrc && /path/to/script.sh'
```

## 실제 예제

### ../blog/sh/key.sh 예제

```bash
#!/bin/zsh -c
# -*- mode: shell-script -*-

# ~/.zshrc에서 로드된 환경 변수 사용
# 예: $GITHUB_TOKEN, $API_KEY, $PROJECT_ROOT 등

# 스크립트 실행
source ~/.zshrc

echo "Project: $PROJECT_ROOT"
echo "Token: ${GITHUB_TOKEN:0:10}..." # 첫 10글자만 출력

# 실제 작업
if [[ -z "$GITHUB_TOKEN" ]]; then
  echo "Error: GITHUB_TOKEN not set in ~/.zshrc"
  exit 1
fi

# ... 추가 로직
```

### Crontab 설정

```bash
# crontab -e
SHELL=/bin/zsh

# 매일 자정에 실행
0 0 * * * /bin/zsh -c 'source ~/.zshrc && ~/.local/bin/key.sh'

# 또는 래퍼 사용
0 0 * * * ~/.local/bin/run-with-zsh ~/.local/bin/key.sh
```

## 트러블슈팅 팁

### 1. Cron 스크립트에서 PATH 확인

```bash
#!/bin/zsh -c
source ~/.zshrc
echo "Cron PATH: $PATH" > /tmp/cron-path.log
which python3 >> /tmp/cron-path.log 2>&1
```

### 2. 로그 파일로 환경 확인

```bash
# crontab -e
* * * * * /bin/zsh -c 'source ~/.zshrc && env > /tmp/cron-env.log 2>&1'
```

### 3. 스크립트 실행 권한 확인

```bash
chmod +x ~/.local/bin/key.sh
chmod +x ~/.local/bin/run-with-zsh
```

## 요약

| 상황 | 해결책 |
|------|--------|
| **간단한 스크립트** | `#!/bin/zsh -c` shebang 사용 |
| **복잡한 환경 로드** | `/bin/zsh -c 'source ~/.zshrc && script'` |
| **.zshenv 설정 필요?** | **불필요** — `.zshrc`로 충분 |
| **Cron env 변수 설정?** | **불필요** — 스크립트 내에서 source하거나 정의 |

## 핵심 교훈

- Cron은 shell을 명시적으로 지정해야 `~/.zshrc` 로드 가능
- `source ~/.zshrc`를 명시적으로 호출하는 것이 가장 안정적
- `.zshenv`와 cron env 설정은 추가로 필요하지 않음
- 로그 파일을 활용해 환경 변수 로드 상태 확인
