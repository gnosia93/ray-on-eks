## Lab 5. 모델 서빙 (Ray Serve) ##

- Ray Serve로 학습된 모델 배포 (deployment/replica 개념)
- 오토스케일링: 요청량 기반 replica 스케일 + Karpenter 노드 스케일 연동
- 리소스 관리: num_cpus/num_gpus, fractional GPU, 모델 다중 배치
- 프로덕션 준비도: health check, graceful shutdown, 무중단 롤아웃/카나리
- Ingress/게이트웨이로 고객별 엔드포인트 노출
