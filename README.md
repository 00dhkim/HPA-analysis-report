# Kubernetes HPA 기반 검색 워크로드 부하 테스트
> Rerank 병목 분석과 Scale-out 효과 검증

## Summary
- Google Cloud 환경에서 HAP(Horizontal Pod Autoscaler) 구성에 따른 검색 서비스의  부하 처리 능력을 검증함.
- 실험 결과, `maxReplicas=5`일 때 처리량이 약 2,880회(2.0 RPS)로, `maxReplicas=1, 3`일 때(처리량 약 1,500회, 0.8 RPS) 대비 약 1.9배의 성능 향상을 확인함.
- 일반 검색(Vector/Keyword)은 수 초(3~9초) 내로 처리되나, CPU 집약적인 Rerank 작업은 95% Latency가 140초를 초과하는 심각한 병목 현상이 발생함. 따라서 고부하 작업의 아키텍처 분리가 필수적임.


<img width="80%" alt="bar_plots" src="https://github.com/user-attachments/assets/dc3ffc96-b003-49ff-94b7-83c8b57dc1bb" />
<img width="60%" alt="time_series_plots" src="https://github.com/user-attachments/assets/979cecbe-f9c2-47da-85af-d9897e40def1" />



## Experimental Setup
- 부하를 계단식으로 증가시키는 시나리오 상정
- 테스트 시나리오 (Step Load):
  - 0~5분: 5명 (워밍업)
  - 5~20분: 20명 (일상 부하)
  - 20~35분: 40명 (피크 부하)
- 워크로드 구성 (Locust Script):
  - Vector Search (가중치 3): 일반 검색 (빠름)
  - Keyword Search (가중치 1): 키워드 검색 (빠름)
  - Rerank (가중치 1): 문서 재순위화 (CPU 과부하 유발)

## System Performance
- 처리량(Throughput): `MaxRep=5`만이 사용자 증가에 비례하여 처리량(RPS)이 선형적으로 증가 (약 2.0 RPS 도달). 1과 3은 0.8 RPS에서 포화.
- 응답 속도(Latency): 40명 부하 시 `MaxRep=1, 3`은 95% 지연 시간이 140초 이상으로 사실상 서비스 불능 상태(Timeout)가 됨. 반면 `MaxRep=5`는 3초대(Median) 유지.

## Bottleneck Analysis
- Blocking 현상: 처리 비용이 높은 rerank 작업이 CPU를 독점하여, 상대적으로 가벼운 keyword search, vector search 작업까지 대기열에 밀리는 현상 확인.
- 느린 스케일링: pod의 초기화 시간이 약 8분으로 과도하게 소모되어, 급격한 트래픽 증가에 제때 대응하지 못함.

## Conclusion & Proposal
- 아키텍처 개선: rerank 작업만이 다른 검색 작업에 영향을 주지 않도록 전용 Pod 그룹으로 분리.
- 서버 로직 개선: 긴 작업에 대해 비동기적 처리 도입하여 사용자 대기 시간 단축.
- 대체 엔진 검토: 초기화 시간을 단축하기 위해, Cold Start가 빠른 고성능 Vector DB 제품(Qdrant, Milvus 등)으로의 마이그레이션 고려.

