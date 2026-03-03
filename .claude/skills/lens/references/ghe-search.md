# github.com 관련 레포 탐색

> **GH 도메인**: 이 파일에서 사용하는 GHE 도메인은 `github.com`이다. 도메인 변경 시 이 파일 단독 수정으로 반영된다.

Prepare Phase에서 참조한다. 로컬에 없는 관련 레포를 GitHub Enterprise에서 찾아 클론할 수 있다.

## 사전 조건 확인

```bash
gh auth status --hostname github.com 2>&1
```

출력 결과에 따라 3가지로 분기한다:

### Case A: 인증 완료
출력에 `Logged in to github.com`가 포함되면: GH 탐색을 진행한다.

### Case B: gh 미설치
출력에 `command not found`가 포함되면 (zsh: `command not found: gh`, bash: `gh: command not found`): 이 단계를 **전체 스킵**한다. 아래 안내를 출력한다:
```
> **gh CLI가 설치되어 있지 않습니다.**
>
> gh를 설치하면 github.com에서 관련 레포를 자동으로 검색·클론하여
> 더 넓은 범위의 정책을 탐지할 수 있습니다.
>
> 설치: `brew install gh`
> 인증: `gh auth login --hostname github.com`
```

### Case C: gh 설치됨, 미인증
위 두 경우가 아닌 경우 (인증 에러, 호스트 미등록 등): 이 단계를 **전체 스킵**한다. 아래 안내를 출력한다:
```
> **github.com 인증이 설정되지 않았습니다.**
>
> GHE에 로그인하면 로컬에 없는 관련 레포도 자동으로 찾아 분석합니다.
>
> 인증: `gh auth login --hostname github.com`
```

## GHE 레포 검색

Prepare Phase GHE 보완 단계에서 호출되며, MISSING의 프로젝트명으로 검색한다.

**주의**: `gh search repos`는 `--hostname` 플래그를 지원하지 않는다. GHE 검색은 반드시 `gh api`를 사용한다.

### 조직 목록 조회
```bash
gh api /user/orgs --hostname github.com --jq '.[].login'
```
- API 실패 시 (rate limit, 네트워크 오류 등): GHE 탐색 전체를 스킵한다.
- 조직이 0개이면: GHE 탐색 전체를 스킵한다.
- org 이름 검증: 영문, 숫자, 하이픈만 허용 (`^[a-zA-Z0-9-]+$`). 매치하지 않으면 해당 org 스킵.

### 레포 검색
```bash
# <QUERY 핵심 단어>는 불용어를 제거한 명사만 추출하여 사용한다.
# shell 특수문자(;, |, &, $, `, \, ', ", (, ))를 제거한 후 삽입한다.
# <QUERY 핵심 단어>와 <org>는 URL 인코딩하여 삽입한다.
gh api "search/repositories?q=<QUERY 핵심 단어>+org:<org>&per_page=20" --hostname github.com
```
- 응답의 `.items[]`에서 `name`, `description`, `html_url`, `archived`, `fork` 필드를 추출한다.
- `<org>`와 `<repo>` 값은 반드시 gh API 응답에서 추출한 것만 사용한다. 사용자 입력을 직접 URL에 삽입하지 않는다.
- CANDIDATES에 이미 있는 레포명은 결과에서 제외한다.

## 검색 결과 필터링

검색 결과에서 명백히 무관한 레포를 제거한다:
- archived 레포 제외
- fork 레포 제외
- 레포명이 B 버킷 패턴(docs, infra, deploy 등)에 매치되면 제외

## 사용자 선택

검색 결과가 있으면:
```
github.com에서 관련될 수 있는 레포를 찾았습니다.

  1. payment-gateway — 결제 게이트웨이 서비스
  2. billing-core — 빌링 코어 로직
  3. settlement-batch — 정산 배치

함께 탐색할 레포를 선택해주세요. (복수 선택 가능, 스킵하려면 "없음" 선택)
```

AskUserQuestion으로 multiSelect 선택지를 제시한다.

## 클론

사용자가 선택한 레포를 shallow clone한다:

**클론 위치**: `LENS_DIR`에 클론한다.

```bash
# <org>와 <repo>는 gh api의 JSON 결과에서 추출한 값만 사용한다.
# 레포명 검증: 영문, 숫자, 하이픈, 언더스코어, 점만 허용 (^[a-zA-Z0-9._-]+$). 검증 실패 시 해당 레포 스킵.
git clone --depth 1 https://github.com/<org>/<repo>.git <LENS_DIR>/<repo>
```

- 클론 성공한 레포를 CANDIDATES에 추가한다. `cloned = true`로 표시한다.
- 클론 실패 시: 해당 레포 스킵, 사유를 안내한다.
- 클론된 레포는 유지된다 (다음 실행 시 로컬 프로젝트로 자동 감지).
