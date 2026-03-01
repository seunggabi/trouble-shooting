# ai-heatmap npx 캐시로 인해 최신 버전이 적용되지 않는 트러블슈팅

> 날짜: 2026-03-01
> 프로젝트: [ai-heatmap](https://github.com/seunggabi/ai-heatmap)
> 관련 버전: v1.17.2 ~

---

## 증상

`npx ai-heatmap@latest update` 또는 `npx ai-heatmap@latest deploy`를 실행했는데 새 버전에서 수정된 버그가 여전히 발생함. npm에 최신 버전이 배포되어 있음에도 변경사항이 반영되지 않음.

```bash
$ npx ai-heatmap@latest --version
1.17.1   # ← 최신 버전(1.17.5)이 아닌 캐시된 구버전 실행
```

---

## 원인: npx 캐시

### 문제

`npx`는 한 번 다운로드한 패키지를 로컬 캐시(`~/.npm/_npx/`)에 저장하고, 이후 실행 시 npm 레지스트리를 새로 확인하지 않고 **캐시된 버전을 그대로 사용**함.

`@latest` 태그를 명시해도 캐시가 있으면 캐시 우선 사용.

```bash
# 캐시 확인
ls ~/.npm/_npx/
# → 이미 ai-heatmap이 캐시되어 있음

# 실제 최신 버전 확인
npm view ai-heatmap version
# → 1.17.5

# npx 실행 (캐시에서 가져옴)
npx ai-heatmap@latest --version
# → 1.17.1  ← 캐시된 구버전
```

---

## 수정

### 실행 전 npx 캐시 삭제

```bash
npx clear-npx-cache --yes && npx ai-heatmap@latest update
```

```bash
npx clear-npx-cache --yes && npx ai-heatmap@latest deploy
```

`--yes` 플래그: 삭제 확인 프롬프트를 건너뜀 (cron 자동화 환경에서 필수).

### cron에 적용

```bash
# BEFORE: 캐시 삭제 없이 실행
0 */6 * * * npx ai-heatmap@latest update

# AFTER: 캐시 삭제 후 실행
0 */6 * * * npx clear-npx-cache --yes && npx ai-heatmap@latest update
```

---

## 검증 방법

```bash
# 1. 캐시 삭제
npx clear-npx-cache --yes

# 2. 최신 버전으로 실행되는지 확인
npx ai-heatmap@latest --version
# ✅ 1.17.5 (최신 버전)
```

---

## 참고

- npx 캐시 위치: `~/.npm/_npx/`
- `clear-npx-cache` 패키지: https://www.npmjs.com/package/clear-npx-cache
- 업그레이드 후 버그가 재현된다면 항상 캐시 삭제 후 재시도

---

## 관련 PR

| 버전 | PR | 내용 |
|------|----|------|
| v1.17.2 | #82 | Upgrade 섹션에 npx 캐시 트러블슈팅 안내 추가 |
| v1.17.3 | #84 | cron 업데이트 예시에 캐시 삭제 추가 |
| v1.17.4 | #86 | init.mjs 생성 README의 cron 예시에 캐시 삭제 추가 |
| v1.17.5 | #88 | deploy 명령 실행 전 캐시 삭제 안내 추가 |
| v1.17.6 | #90 | 모든 `ai-heatmap@latest` 명령 앞에 캐시 삭제 추가 |
| v1.17.7 | #98 | `--yes` 플래그 추가 (프롬프트 자동 건너뜀) |
