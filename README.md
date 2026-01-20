# ray-on-ec2

![](https://github.com/gnosia93/ray-on-ec2/blob/main/lesson/images/ray-on-ec2-workshop.png)
_ray sample data - s3://ray-example-data_

* [C1. VPC 생성](https://github.com/gnosia93/ray-on-aws/blob/main/lesson/1-create-vpc.md)

* [C2. ray 클러스터 생성](https://github.com/gnosia93/ray-on-aws/blob/main/lesson/2-create-cluster.md)

* [C3. 작업 제출 (Job Submission)](https://github.com/gnosia93/ray-on-aws/blob/main/lesson/3-job-submission.md)

* [C4. 오토 스케일링](https://github.com/gnosia93/ray-on-aws/blob/main/lesson/4-auto-scaling.md)

* [C5. 스팟 인스턴스 사용하기](https://github.com/gnosia93/ray-on-aws/blob/main/lesson/5-spot-instance.md)

* [C6. 커스텀 자원 정밀 제어](https://github.com/gnosia93/ray-on-ec2/blob/main/lesson/6-custom-resource-control.md)

* [C7. 모니터링]()

* [C8. 대규모 텍스트 전처리](https://github.com/gnosia93/ray-on-ec2/blob/main/lesson/8-text-preprocessing.md)


## CPU 아키텍처 특징 ##

- Intel (c7i): 가장 표준적입니다. 특히 AMX(Advanced Matrix Extensions) 가속기 덕분에 텍스트 임베딩이나 행렬 연산에서 압도적이다. 
- AMD (c7a): 순수 연산 속도(Raw Clock)와 가성비가 좋습니다. 단순 텍스트 정규화나 정규식 처리 위주라면 Intel보다 저렴하면서 성능은 비슷하게 나온다.
- Graviton (AWS 자체 칩 - arm64)
  - 동일 성능 대비 비용이 최대 20% 저렴하다.
  - arm64 전용 바이너리를 내려받아야 한다. 일부 라이브러리의 경우 환환성 문제를 일으킬 수 있다.


## 교육 내용 ##
* 1교시 - AWS EC2 포트폴리오 소개 / CPU 별 특징 소개
* 2교시 - ray data / 데이터 전처리 이론 교육
    * 머신 러닝 데이터 전처리
    * LLM text 데이터 전처리
    * spark ETL 대체   
* 3교시 / 4교시 - ray 실습
* engage strategy
   * 신규 니즈 발굴
   * 단독 워크샵 형태
   * ML 워크샵시 컨텐츠 소개 


## 레퍼런스 ##

* https://docs.ray.io/en/latest/ray-overview/installation.html
* https://github.com/dmatrix/ray-core-tutorial/blob/ad5f1fa700d87a9af1e21027f06f02cfdcc937f3//ex_07_ray_data.ipynb
* ray sample data - s3://ray-example-data
