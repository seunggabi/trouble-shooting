# mac-ops Homebrew 설치 시 잘못된 버전 표시 트러블슈팅

> 날짜: 2026-03-01
> 프로젝트: [mac-ops](https://github.com/seunggabi/mac-ops)
> 관련 버전: v1.2.0

---

## 증상

Homebrew로 설치한 `mac-ops`의 버전을 확인하면 실제 버전(`v1.2.0`)이 아닌 엉뚱한 버전이 출력됨.

```bash
$ mac-ops version
v5.0.14   # ← 잘못된 버전 (Homebrew 자체 버전)
```

개발 환경(git clone)에서는 정상적으로 표시됨.

```bash
$ ./bin/mac-ops version
v1.2.0   # ← 정상
```

---

## 원인: `git describe`의 부모 디렉토리 탐색

### 문제

`bin/mac-ops`에서 버전을 출력할 때 `git describe --tags`를 사용했음.

```bash
# BEFORE
VERSION=$(git describe --tags --abbrev=0 2>/dev/null)
echo "$VERSION"
```

Homebrew로 설치하면 `.git` 디렉토리가 없음. `git describe`는 `.git`을 찾을 때까지 **부모 디렉토리를 재귀적으로 탐색**하는데, Homebrew 자체가 git 저장소이므로 `/opt/homebrew/.git`을 발견하고 Homebrew의 최신 태그(`v5.0.14` 등)를 반환함.

```
/opt/homebrew/
├── .git/               ← Homebrew 자체 git 저장소
└── Cellar/
    └── mac-ops/
        └── 1.2.0/
            └── bin/
                └── mac-ops   ← 여기서 git describe 실행 시 /opt/homebrew/.git 탐색
```

### 오류 흐름

```
git describe 실행
  → .git 없음 (현재 디렉토리)
  → 부모 디렉토리 탐색
  → /opt/homebrew/.git 발견
  → Homebrew 태그 반환 (v5.0.14)
```

---

## 수정

### 접근 방식

`.git` 존재 여부를 먼저 확인하고, 없으면 `VERSION` 파일에서 읽도록 fallback 추가.

```bash
# AFTER: bin/mac-ops
if git rev-parse --git-dir > /dev/null 2>&1; then
  # 개발 환경: git describe 사용
  VERSION=$(git describe --tags --abbrev=0 2>/dev/null)
else
  # 배포 환경 (Homebrew 등): VERSION 파일 사용
  SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
  VERSION=$(cat "$SCRIPT_DIR/../VERSION" 2>/dev/null)
fi
echo "${VERSION:-unknown}"
```

### VERSION 파일 추가

```
# VERSION
1.2.0
```

### 릴리즈 워크플로우에서 tarball에 VERSION 파일 포함

```yaml
# .github/workflows/release.yml
- name: Patch tarball with VERSION file
  run: |
    echo "${{ steps.tag.outputs.version }}" > VERSION
    tar -czf mac-ops.tar.gz bin lib VERSION
```

---

## 검증 방법

```bash
# git 없는 환경 시뮬레이션
cd /tmp/test
cp -r /path/to/mac-ops/{bin,lib,VERSION} .
./bin/mac-ops version
# ✅ v1.2.0 (VERSION 파일에서 읽음)

# git 있는 환경
cd /path/to/mac-ops
./bin/mac-ops version
# ✅ v1.2.0 (git describe에서 읽음)
```

---

## 관련 PR

| 버전 | PR | 내용 |
|------|----|------|
| v1.2.1 | #29 | Homebrew 설치 시 잘못된 버전 표시 수정 |
