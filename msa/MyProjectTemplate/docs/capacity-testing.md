# 처리량과 가용성 검증

## 왜 고정 TPS 표를 제공하지 않는가

동일한 코드도 쿼리 형태, 데이터 크기, 캐시 적중률, 네트워크, JVM heap, 외부 API에 따라 처리량이 크게 달라진다. 따라서 아키텍처 설명의 숫자는 계획 입력이며 보장값이 아니다.

## 테스트 단계

1. smoke: 1~5 TPS로 계약과 오류율 확인
2. baseline: 단일 인스턴스의 knee point 탐색
3. target: 목표 TPS를 30분 유지
4. spike: 30초 내 3배 상승 후 복구 확인
5. soak: 4~12시간 동안 leak과 lag 확인
6. failover: 앱 인스턴스, reader, broker 중 하나를 제거

## 결과 기록 필수값

- Git SHA와 이미지 digest
- CPU, 메모리, JVM, 인스턴스 수
- DB 사양, pool 크기, 데이터 행 수
- Redis cache hit ratio
- Kafka partition 수와 consumer 수
- 요청 조합과 payload 크기
- TPS, p50/p95/p99/max, 오류율
- 장애 주입 시 복구 시간과 손실 여부

가용성 목표 `99.9%`는 월 약 43분의 error budget을 의미하지만, 단일 부하 테스트 성공만으로 달성되는 것이 아니다. 배포, 백업 복구, 장애 대응까지 포함해 판단한다.
