
1. cluster.yaml 설정 파일 (자동 프로비저닝용)
이 파일은 Ray Cluster Launcher가 어떤 사양의 EC2를 몇 대 띄울지 정의하는 설계도입니다
```
# 클러스터 식별 이름
cluster_name: ray-data-workshop

# 인스턴스를 띄울 리전
provider:
    type: aws
    region: ap-northeast-2 # 서울 리전
    availability_zone: ap-northeast-2a

# 각 노드에서 실행될 설정 (Python 설치 등)
setup_commands:
    - pip install -U "ray[default,data]" pandas pyarrow boto3

# 노드별 상세 사양
available_node_types:
    # 1. 헤드 노드 (관리용)
    head_node:
        node_config:
            InstanceType: m6i.xlarge
            ImageId: ami-0c9c942bd7bf113a2 # Ubuntu 22.04 (서울 리전 기준 확인 필요)
        min_workers: 0
        max_workers: 0
    # 2. 워커 노드 (데이터 처리용)
    worker_node:
        node_config:
            InstanceType: m6i.2xlarge
        min_workers: 2 # 기본 2대 실행
        max_workers: 5 # 필요시 5대까지 자동 확장

# 헤드 노드를 우선 시작하도록 설정
head_node_type: head_node

```
