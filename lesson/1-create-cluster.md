### 1. Ray 클러스터 런처 설치하기 ###
```
pip 사용 시: pip install -U "ray[default]"
Conda 사용 시: conda install -c conda-forge "ray-default" 
```

### 2. 환경설정하기 ###
```
export AWS_REGION=$(aws ec2 describe-availability-zones --query 'AvailabilityZones[0].RegionName' --output text)
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export CLUSTER_NAME="ray-on-aws"
export K8S_VERSION="1.34"
export KARPENTER_VERSION="1.8.1"
export VPC_ID=$(aws ec2 describe-vpcs --filters Name=tag:Name,Values="${CLUSTER_NAME}" --query "Vpcs[].VpcId" --output text)
```

### 3. 클러스터 설정하기 (cluster.yaml) ###
```
cluster_name: ${CLUSTER_NAME}

provider:
    type: aws
    region: ${AWS_REGION}                              # 서울 리전
    availability_zone: ap-northeast-2a
    use_internal_ips: true                             # 이 줄을 추가하세요!

# 각 노드에서 실행될 설정 (Python 설치 등)
setup_commands:
    - pip install -U "ray[default,data]" pandas pyarrow boto3

# 노드별 상세 사양
available_node_types:
    # 헤드 노드
    head_node:
        node_config:
            InstanceType: m6i.xlarge
            ImageId: ami-0c9c942bd7bf113a2             # Ubuntu 22.04 (서울 리전 기준 확인 필요)
            SubnetId: subnet-xxxxxxxxxxxxxxxxx         # 프라이빗 서브넷 ID 입력
            SecurityGroupIds:                          # 필요한 경우 보안 그룹 ID도 명시
                - sg-xxxxxxxxxxxxxxxxx
        min_workers: 0                                 # min_workers/max_workers: 0 - 헤드 노드는 관리용이므로 스스로 워커 역할을 겸하지 않도록 설정
        max_workers: 0                             
    # 워커 노드 (데이터 처리용)
    worker_node:
        node_config:
            InstanceType: m6i.2xlarge
            ImageId: ami-0c9c942bd7bf113a2 # 헤드 노드와 동일한 이미지 사용
            SubnetId: subnet-xxxxxxxxxxxxxxxxx         # 프라이빗 서브넷 ID 입력
            SecurityGroupIds:                          # 필요한 경우 보안 그룹 ID도 명시
        min_workers: 4                                 # 기본 4대 실행
        max_workers: 8                                 # 필요시 8대까지 자동 확장

head_node_type: head_node                              # 정의한 여러 노드 타입 중 어떤 것이 클러스터의 전체 제어를 담당할 '헤드'인지 확정
```
보안 그룹 (Security Group): 헤드 노드와 워커 노드 간에 모든 TCP 포트가 서로 열려 있어야 합니다. 보통 동일한 보안 그룹을 부여하고 Security Group 자기 참조 규칙 (Self-reference)을 추가하여 해결합니다.
AWS 보안 그룹 설정 시, 내부 통신(Self-reference) 외에 배스천 호스트의 보안 그룹으로부터 오는 22번(SSH)과 8265번(대시보드) 포트 허용도 꼭 확인하세요!

### 4. ray 클러스터 생성하기 ###
로컬 환경에 AWS 자격 증명(aws configure)이 설정되어 있어야 합니다.
```
# YAML 설정을 바탕으로 EC2 생성 및 Ray 설치 진행
ray up cluster.yaml -y
```

### 5.작업 제출 (Python 스크립트 실행): ###
```
ray job submit --address http://<헤드노드_사설IP>:8265 -- python data_job.py
```

### 6. 작업 확인 (PC에서 확인): ###
베스천 호스트에서 다음 명령어를 실행한다. 
```
ray dashboard cluster.yaml
```
PC 에서 터널링을 뚫어주고, 웹브라우저로 http://localhost:8265 으로 접속한다.  
```
ssh -i <로컬PC의_프라이빗_키_파일_경로> -L 8265:<헤드노드의_사설_IP>:8265 <사용자>@<배스천_호스트의_공인_IP>
```

### 7. 클러스터 삭제하기 ###
```
ray down cluster.yaml -y
```

*  S3 대용량 데이터 로드 및 분산 처리 예제 코드입니다. 이 코드는 EC2 클러스터의 모든 CPU 코어를 활용하여 병렬로 동작합니다

