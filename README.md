### 1단계: 인프라 프로비저닝 (Terraform & CfN) ###
먼저 사전에 정의한 CloudFormation과 Terraform을 실행하여 판을 깝니다.
* VPC 인프라: AWS CloudFormation으로 Public(Bastion 위치) 및 Private(Ray 위치) 서브넷을 생성합니다.
* IAM 권한: 앞서 작성한 Terraform 코드를 실행하여 ray-instance-profile을 만듭니다.
* 강조: 이 프로파일이 있어야 Ray가 스스로 EC2를 사고팔며(?) 오토스케일링을 할 수 있습니다.

### 2단계: 배스천 접속 및 Ray 클러스터 가동 ###
교육생들이 각자의 로컬 PC에서 배스천에 로그인하여 사령관이 되는 단계입니다.
* 배스천 점프: ssh -A ec2-user@<Bastion-IP> (키 포워딩 필수)
* YAML 설정: ray-cluster.yaml에 max_workers: 30과 IamInstanceProfile을 기입합니다.
* 클러스터 런칭:
```
ray up ray-cluster.yaml -y
```

### 3단계: Upscaling 폭격 (EC2 30대 소환)
이제 160코어의 한계를 시험하며 EC2 자원을 최대치로 땡기는 핵심 실습입니다.
* 스트레스 테스트 스크립트 실행
```
# 배스천에서 실행 (stress.py)
import ray, time
ray.init(address="auto")

@ray.remote(num_cpus=1)
def heavy_task():
    time.sleep(300) # 5분간 코어 점유
    return True

# 480개(30대 분량)의 작업을 한꺼번에 투척!
ray.get([heavy_task.remote() for _ in range(480)])
```
현상 관찰:
* Ray Status: Pending: 320 Tasks 발생 -> Autoscaler가 Launching 20 Nodes 시작.
* AWS Console: EC2 인스턴스 페이지에 c7i.4xlarge 20대가 한꺼번에 생성되며 Pending 상태가 되는 장관을 확인합니다


---
* static
* 오토 스케일링 설정
* spot 사용하기
* on-demand / spot 믹스하기
* ray 는 spot 회쉬되더라도 교체해서 동작한다. 어떻게?
  





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

## 레퍼런스 ##
* https://docs.ray.io/en/latest/ray-overview/installation.html
* https://github.com/dmatrix/ray-core-tutorial/blob/ad5f1fa700d87a9af1e21027f06f02cfdcc937f3//ex_07_ray_data.ipynb
* https://github.com/aws-samples/aws-samples-for-ray/tree/main/ec2
