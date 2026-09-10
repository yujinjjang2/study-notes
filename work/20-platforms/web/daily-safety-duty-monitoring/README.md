# 일일안전당직자모니터링

웹 화면 `MontInqCompanyPersonInChargeMonitoring` 개발 중 확인한 개인조치율 쿼리와 실행계획 분석을 관리한다.

- 대상 Mapper: `MontInqCompanyPersonInChargeMonitoringSql.xml`
- 대상 Query: `selectIndivActRateList`
- 화면: 일일안전당직자모니터링

## 문서

- [개인조치율 쿼리 개선 이력](select-indiv-act-rate-list-optimization-history.md)
- [실행계획 분석 템플릿](query-execution-plan-analysis-template.md)
- [쿼리 비교 및 운영 조회](daily-safety-duty-query-comparison.md)

## 현재 상태

현재 2차 논리 개선은 적용됐으며, 표준 정책에 따라 `/*+ Materialize */` 힌트는 제거했다. 힌트 제거 후 동일 파라미터로 결과 건수와 실행계획을 재측정해야 한다.
