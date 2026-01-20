* 1교시 - AWS EC2 포트폴리오 소개 / CPU 별 특징 소개
* 2교시 - ray data / 데이터 전처리 이론 교육
    * 머신 러닝 데이터 전처리
    * LLM text 데이터 전처리
    * spark ETL 대체   
* 3교시 / 4교시 - ray 실습

# ray-on-aws

* [C1. VPC 생성](https://github.com/gnosia93/ray-on-aws/blob/main/lesson/1-create-vpc.md)

* [C2. ray 클러스터 생성](https://github.com/gnosia93/ray-on-aws/blob/main/lesson/2-create-cluster.md)

* [C3. 작업 제출 (Job Submission)](https://github.com/gnosia93/ray-on-aws/blob/main/lesson/3-job-submission.md)

* [C4. 오토 스케일링](https://github.com/gnosia93/ray-on-aws/blob/main/lesson/4-auto-scaling.md)

* [C5. 스팟 인스턴스 사용하기]() - ray 는 spot 회쉬되더라도 교체해서 잘 동작한다. 어떻게?

* [C6. On-demand / Spot 함께 사용하기]

* C7. 커스텀 자원 정밀 제어	Custom_Resource를 활용해 특정 인스턴스 타입을 골라 띄우는 Resource-specific Scaling 구현	

* C8. Fault Tolerance 체험	가동 중인 워커 노드를 강제 종료(Terminate)하고, Ray가 작업 손실 없이 새 EC2로 복구하는 과정 실측	

* C9. 대규모 전처리 벤치마크	100GB급 텍스트 데이터를 30대 클러스터에서 전처리하며 Parallelism & Batch Size 최적화 값 찾기	


## 아키텍처별 가이드 ##

Ray는 분산 컴퓨팅 프레임워크이기 때문에 하드웨어 아키텍처의 특성을 아주 정직하게 타는 편입니다. 
- Intel (c7i): 가장 표준적입니다. 특히 AMX(Advanced Matrix Extensions) 가속기 덕분에 텍스트 임베딩이나 행렬 연산에서 압도적입니다. 
- AMD (c7a): 순수 연산 속도(Raw Clock)와 가성비가 좋습니다. 단순 텍스트 정규화나 정규식 처리 위주라면 Intel보다 저렴하면서 성능은 비슷하게 나옵니다.
- Graviton (AWS 자체 칩 - arm64)
  - 성능/비용: 가성비 끝판왕입니다. 동일 성능 대비 비용이 약 20% 저렴합니다.
  - 주의사항: pip install 시 arm64 전용 바이너리를 내려받아야 하므로 AMI(Amazon Machine Image) 설정이 달라집니다.

* EC2 활용 극대화: 인스턴스 타입을 섞어 쓰기 때문에 Mixed Instances Policy를 완벽하게 실습하게 됩니다

* Mixed Instance Policy 


## 데이터 ##
```
aws s3 ls s3://ray-example-data/ --no-sign-request
```
```
aws s3 ls s3://ray-example-data/common_voice_17/parquet/
2025-09-27 04:00:11  494117001 0.parquet
2025-09-27 04:00:11  494284009 1.parquet
2025-09-27 04:00:11  494615004 2.parquet
2025-09-27 04:00:11  492952215 3.parquet
2025-09-27 04:00:11  493082766 4.parquet
2025-09-27 04:00:22  497584052 5.parquet
2025-09-27 04:00:23  493463495 6.parquet
2025-09-27 04:00:23  494387266 7.parquet
2025-09-27 04:00:23  494530333 8.parquet
2025-09-27 04:00:23  476320458 9.parquet
```

## 전처리 ##
* 텍스트 전처리
* 이미지 전처리
* 보이스 전처리


## 레퍼런스 ##

* https://docs.ray.io/en/latest/ray-overview/installation.html
* https://github.com/dmatrix/ray-core-tutorial/blob/ad5f1fa700d87a9af1e21027f06f02cfdcc937f3//ex_07_ray_data.ipynb
* https://github.com/aws-samples/aws-samples-for-ray/tree/main/ec2
