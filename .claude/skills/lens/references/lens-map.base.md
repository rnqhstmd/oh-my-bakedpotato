# lens 통합맵 (base)
#
# 스킬에 동봉되는 기본 통합맵.
# Prepare Phase에서 사용자 통합맵(<LENS_DIR>/lens-map.md)과 독립적으로 읽힌다.
# 두 맵의 매칭 결과를 합집합(중복 제거)으로 합친다. 파일 Write 없음.
#
# 형식:
# ## <프로젝트명>
# 키워드1, 키워드2, 키워드3
#
# glob 패턴 지원: ## shopping-* → 로컬 디렉토리명 매칭 프로젝트에 키워드 적용
# 프로젝트당 키워드 최대 30개
#
# 예시:
# ## loop-pack-be-l2-vol2-java
# 커머스, 상품, 주문, 배치, JPA, Redis, Kafka, API, 엔티티
