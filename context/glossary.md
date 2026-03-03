# 공통 용어 사전

> 도메인을 가리지 않고 프로젝트 전체에서 쓰이는 용어입니다.
> 도메인별 용어는 `context/{도메인}/glossary.md`를 참조하세요.

| 용어 | 설명 |
|------|------|
| oh-my-bakedpotato | Java Spring Boot 프로젝트용 AI 개발 플러그인 |
| GH | GitHub (github.com) |
| PRD | Product Requirements Document. 제품 요구사항 문서 |
| context/ | 도메인별 아키텍처·용어·구현 상태를 정리하는 디렉토리 |
| 4계층 아키텍처 | interfaces → application → domain → infrastructure 패키지 구조 |
| Facade 패턴 | Controller에서 호출하는 유스케이스 오케스트레이터. Service를 조합 |
| BaseEntity | 모든 JPA 엔티티의 부모. id, createdAt, updatedAt, deletedAt 자동 관리 |
| ApiResponse | 통합 API 응답 래퍼. meta(result, errorCode, message) + data |
| Spotless | Gradle 코드 포맷터 플러그인. 네이버 코딩 컨벤션 적용 |
