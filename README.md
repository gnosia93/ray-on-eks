# ray-on-ec2

![](https://github.com/gnosia93/ray-on-ec2/blob/main/lesson/images/ray-on-ec2-workshop.png)
_ray sample data - s3://ray-example-data_

Ray Data는 S3나 데이터 웨어하우스에 흩어진 수 테라바이트(TB) 급 데이터를 여러 노드에서 병렬로 읽고 전처리할 수 있는 스트리밍 데이터 처리 라이브러리이다. 메모리 크기를 초과하는 데이터셋도 Object Spilling 기술을 통해 끊김 없이 처리하며, GPU 가속기와 CPU 워커 간의 파이프라이닝을 최적화하여 훈련 대기 시간을 최소화한다. 특히 단순한 전처리를 넘어 분산 LLM 추론 및 학습 전처리에 특화되어 있어 현대적인 AI 인프라의 필수 요소로 자리 잡고 있다.


* [C1. VPC 생성](https://github.com/gnosia93/ray-on-aws/blob/main/lesson/1-create-vpc.md)

* [C2. ray 클러스터 생성](https://github.com/gnosia93/ray-on-aws/blob/main/lesson/2-create-cluster.md)

* [C3. 작업 제출 (Job Submission)](https://github.com/gnosia93/ray-on-aws/blob/main/lesson/3-job-submission.md)

* [C4. 오토 스케일링](https://github.com/gnosia93/ray-on-aws/blob/main/lesson/4-auto-scaling.md)

* [C5. 스팟 인스턴스 사용하기](https://github.com/gnosia93/ray-on-aws/blob/main/lesson/5-spot-instance.md)

* [C6. 커스텀 자원 정밀 제어](https://github.com/gnosia93/ray-on-ec2/blob/main/lesson/6-custom-resource-control.md)

* [C7. 모니터링](https://github.com/gnosia93/ray-on-ec2/blob/main/lesson/7-ray-observability.md) - 작성중 ...

* [C8. 대규모 텍스트 전처리](https://github.com/gnosia93/ray-on-ec2/blob/main/lesson/8-text-preprocessing.md)


## 레퍼런스 ##

* https://docs.ray.io/en/latest/ray-overview/installation.html
* https://github.com/dmatrix/ray-core-tutorial/blob/ad5f1fa700d87a9af1e21027f06f02cfdcc937f3//ex_07_ray_data.ipynb
