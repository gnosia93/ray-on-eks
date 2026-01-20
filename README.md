# _ray`[data]`-on-ec2_ 

![](https://github.com/gnosia93/ray-on-ec2/blob/main/lesson/images/ray-on-ec2-workshop-2.png)
_ray sample data - s3://ray-example-data_

Ray는 파이썬 코드를 수백 대의 서버로 즉시 확장해 주는 분산 컴퓨팅 프레임워크이며, Ray Data는 그 위에서 대규모 데이터를 병렬로 처리하는 최적의 엔진입니다. 본 워크숍은 이 두 기술을 활용해 S3의 대용량 데이터를 가장 효율적으로 전처리하는 실무 노하우에 집중한다. 특히 성능 극대화를 위한 최신 EC2 인스턴스 활용법과 비용을 90% 까지 절감하는 스팟 인스턴스 기반의 오토스케일링 전략을 소개한다. 또한 기업형 데이터 처리에 필수적인 보안을 위해 VPC 프라이빗 서브넷 환경에서의 클러스터 구성과 오브젝트 스필링을 통한 OOM 방지 대책을 실습하며, 모니터링 스택까지 더해진 완성도 높은 전처리 파이프라인을 경험한다.

### _Topics_ ###

* [C1. VPC 생성](https://github.com/gnosia93/ray-on-aws/blob/main/lesson/1-create-vpc.md)

* [C2. ray 클러스터 생성](https://github.com/gnosia93/ray-on-aws/blob/main/lesson/2-create-cluster.md)

* [C3. 작업 제출 (Job Submission)](https://github.com/gnosia93/ray-on-aws/blob/main/lesson/3-job-submission.md)

* [C4. 오토 스케일링](https://github.com/gnosia93/ray-on-aws/blob/main/lesson/4-auto-scaling.md)

* [C5. 스팟 인스턴스 사용하기](https://github.com/gnosia93/ray-on-aws/blob/main/lesson/5-spot-instance.md)

* [C6. 커스텀 자원 정밀 제어](https://github.com/gnosia93/ray-on-ec2/blob/main/lesson/6-custom-resource-control.md)

* [C7. 모니터링](https://github.com/gnosia93/ray-on-ec2/blob/main/lesson/7-ray-observability.md) - 작성중 ...

* [C8. 대규모 텍스트 처리](https://github.com/gnosia93/ray-on-ec2/blob/main/lesson/8-text-preprocessing.md)

* [C9. EC2 EFA 설정](https://github.com/gnosia93/ray-on-ec2/blob/main/lesson/c9-ec2-efa.md) - 작성중 ...
  

### _Refs_ ###

* [Installing Ray](https://docs.ray.io/en/latest/ray-overview/installation.html)
* [introduction to Ray datasets APIs](https://github.com/dmatrix/ray-core-tutorial/blob/ad5f1fa700d87a9af1e21027f06f02cfdcc937f3//ex_07_ray_data.ipynb)
