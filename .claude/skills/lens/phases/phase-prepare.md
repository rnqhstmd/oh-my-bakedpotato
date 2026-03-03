# Prepare Phase: 준비 & 후보 확정

## 0. 인자 파싱

ARGS에서 파싱:
- `--detail` → `DETAIL_MODE = true`. 기본 `false`.
- `--skip-update` → `SKIP_UPDATE = true`. 기본 `false`.
- `--projects <glob>` → fallback의 find 결과를 이 패턴으로 필터링한다.
- `--idea "<텍스트>"` → `IDEA_RAW`. Report Phase 9절에서 사용.
- 나머지 → `RAW_QUERY`.

## 1. 초기화 + 후보 수집 (병렬)

다음을 **모두 동시에** 발행한다:

1. `Read(<프로젝트 루트>/.claude/skills/lens/references/lens-map.base.md)` → base 맵 원문
2. `Bash(which gh)` → GHE CLI 존재 여부

**동시에 LLM이 수행** (도구 호출 불필요):

**(1) 쿼리 분석**:
- "~에서" 패턴으로 `EXPLICIT_REPOS` 추출. 레포명 제거 후 → `QUERY`.
- 불용어/조사 제거 → 비즈니스 키워드. shell 특수문자 제거.
- **의도어 제거**: 사용자가 알고 싶은 "의도"를 나타내는 메타 용어는 코드 검색에 유효하지 않으므로 키워드에서 제외한다. 의도어 목록: `정책, 규칙, 로직, 구현, 코드, 설명, 정리, 점검, 분석, 확인, 조회`.
- 키워드 0개 → AskUserQuestion. 최대 2회 재시도.

**(2) 설정 확정**: `MAKER_DIR` = `~/.shared-maker` (`~` → `$HOME`). `LENS_DIR` = `~/.shared-maker/repository` (`~` → `$HOME`).

도구 결과 도착 후:

**(3) base 맵 매칭**: base 맵 원문에서 키워드 정확 일치하는 프로젝트 추출.

## 2. 사용자 맵 + 로컬 확인

LENS_DIR 확정 후 **2단계**로 진행한다.

### 2-A. 사용자 맵 존재 확인 + 필수 작업 (병렬)

다음을 **모두 동시에** 발행한다:

1. `Bash(mkdir -p ~/.shared-maker/lens/sessions/<SESSION_ID>)` → `SESSION_DIR`. `SESSION_ID`는 LLM이 직접 4자리 hex를 생성한다 (e.g. `a3f1`). 레포 외부(`~/.shared-maker/`)에 생성한다.
2. `Bash(test -f <MAKER_DIR>/lens-map.md && echo exists || echo missing)` → 출력이 `exists`이면 파일 있음, `missing`이면 없음
3. gh 있으면: `Bash(gh auth status --hostname github.com 2>&1 || true)` → GHE_AUTH 판정:
   - 출력에 `Logged in` 포함 → `GHE_AUTH = "authenticated"`
   - 그 외 → `GHE_AUTH = "no_auth"`
   - (gh 없으면 1절 `which gh` 결과로 `GHE_AUTH = "no_gh"` 확정, 이 호출 자체를 스킵)
4. `EXPLICIT_REPOS` + base 매칭 결과의 각 프로젝트에 대해 로컬 존재 확인:
   ```bash
   ls -d <LENS_DIR>/<name>/.git <LENS_DIR>/<name>/main/.git 2>/dev/null || true
   ```
   단일 `ls -d` 호출로 2개 경로를 동시에 확인한다. `2>/dev/null || true`로 항상 exit 0을 보장하여 sibling error를 방지한다. 존재하는 경로만 출력된다.
   - `<name>/.git` 출력됨 → `direct` (일반 레포) → `path = <LENS_DIR>/<name>`
   - `<name>/main/.git` 출력됨 → `worktree` (워크트리 구조) → `path = <LENS_DIR>/<name>/main`
   - 아무것도 출력 안 됨 → `missing` → MISSING에 추가
   - 여러 경로가 존재하면 첫 번째 매칭(direct > worktree)을 사용한다.

> **주의**: 병렬 배치에서 하나의 도구가 non-zero exit code를 반환하면 동일 배치의 모든 도구가 sibling error로 실패한다. 따라서 (1) 사용자 맵 확인은 `test -f ... && echo exists || echo missing`으로 항상 exit 0을 보장하고, (2) `ls -d`에는 `2>/dev/null || true`를 붙이고, (3) `Read`는 이 배치에 넣지 않는다.

### 2-B. 사용자 맵 Read (조건부)

2-A의 `ls` 결과가 `exists`이면:
- `Read(<MAKER_DIR>/lens-map.md)` → 사용자 맵 원문

도구 결과 도착 후:

**(4) 사용자 맵 매칭**: 사용자 맵 원문에서 키워드 정확 일치하는 프로젝트 추출.

### 사용자 맵 추가분 로컬 확인 (조건부)

사용자 맵에서 매칭되었으나 2절 #4에서 확인하지 않은 프로젝트(base에 없었던 것)가 있으면:
```bash
ls -d <LENS_DIR>/<name>/.git <LENS_DIR>/<name>/main/.git 2>/dev/null || true
```
2-A #4와 동일한 방식으로 판정한다.
사용자 맵이 없거나, 추가 매칭 프로젝트가 없으면 이 단계는 스킵된다.

### 후보 통합 (unique)

`EXPLICIT_REPOS` ∪ base 매칭 ∪ user 매칭 → **중복 제거** → 후보 목록.
각 프로젝트의 path는 ls -d 결과에서 결정.
`missing`인 프로젝트 → `MISSING`.

### GHE 보완

`MISSING`이 있고 `GHE_AUTH = "authenticated"`이면:
- `Read(<프로젝트 루트>/.claude/skills/lens/references/ghe-search.md)` 절차로 검색 + 클론.
- 클론 성공 → CANDIDATES에 추가.
- 인증 없음 → skip + 안내.

### 후보 0개 → Fallback

CANDIDATES가 0개이면:
- `Bash(find ...)` 로 LENS_DIR 전체 스캔하여 `.git` 디렉토리가 있는 프로젝트를 수집.
- EXCLUDE_PROJECTS 제외 → CANDIDATES 후보로 변환. 상한 10개, 초과 시 AskUserQuestion.

### 상한

CANDIDATES **10개 초과** → 상위 10개. 안내 출력.

## 3. 최신화

`CANDIDATES`가 0개이면 건너뛴다.
`SKIP_UPDATE = true`이면 건너뛴다.

### 브랜치 감지 + checkout + pull (순차 단계, 프로젝트별 병렬)

각 후보에 대해 **동시에** (5개씩 배치) 아래 3단계를 순차 실행한다:

**Step 1 — 기본 브랜치 감지**:
```bash
git -C <path> symbolic-ref refs/remotes/origin/HEAD
```
- 성공 시 `refs/remotes/origin/<branch>` 출력 → 오케스트레이터가 `refs/remotes/origin/` 접두사를 제거하여 branch명 파싱.
- 실패 시 Step 2로 진행.

**Step 2 — fallback 브랜치 감지** (Step 1 실패 시):
```bash
git -C <path> branch --list main master develop
```
- 출력된 첫 줄에서 `*`와 공백을 제거하여 branch명으로 사용 (오케스트레이터가 파싱).
- 출력 없음 → 해당 프로젝트 스킵.

**Step 3 — checkout + pull**:
```bash
git -C <path> checkout <branch> --quiet
```
```bash
git -C <path> pull origin <branch> --ff-only --quiet
```
- checkout과 pull을 개별 호출로 분리한다. 둘 다 `Bash(git -C *)` 패턴 매칭.
- checkout 실패 시 pull 스킵.
- `timeout: 120000`.

**실패 처리**: best-effort. 실패해도 탐색 계속.

## 4. 사용자 보고

```
정책 탐지를 시작합니다.

- 탐색 후보: <K>개
  <후보 목록 — 명시/맵매칭 표시>
<찾지 못한 레포가 있으면>
- 찾지 못한 레포: <레포명>
</찾지 못한 레포>
- 쿼리: "<QUERY>"
- 검색 키워드: <한국어 키워드 나열>

코드를 탐색합니다.
```

Explore Phase로 진행한다.
