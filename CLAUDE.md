# CLAUDE.md

## Command Center — Java Spring Boot 개발 플러그인

Java Spring Boot 멀티모듈 프로젝트를 위한 AI 기반 개발 플러그인입니다.
PRD 작성, 설계, 구현, 리뷰, 커밋/PR까지 전체 개발 사이클을 에이전트 팀이 수행합니다.

한국어로 응답하세요. 코드와 커밋 메시지도 한국어를 기본으로 합니다.

---

## 프로젝트 적응

프로젝트 작업 시:
1. **프로젝트의 CLAUDE.md를 먼저 읽습니다.** 프로젝트 루트의 `CLAUDE.md`에 아키텍처, 패턴, 컨벤션이 정의되어 있습니다.
2. **프로젝트 CLAUDE.md의 컨벤션이 이 플러그인의 일반 규칙보다 우선합니다.**
3. **빌드/테스트 명령**: `./gradlew build`, `./gradlew test`, `./gradlew spotlessApply`

---

## 작업 범위

- **PR 생성까지만.** PR 머지(`gh pr merge` 등)는 절대 실행하지 마세요.
- 사용자가 직접 머지를 요청하더라도 거절하고, PR 링크를 제공하여 직접 머지하도록 안내하세요.
