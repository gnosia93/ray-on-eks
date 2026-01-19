### 1. 환경 설정 ###
```
export AWS_REGION=$(aws ec2 describe-availability-zones --query 'AvailabilityZones[0].RegionName' --output text)
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export CLUSTER_NAME="ray-on-aws"
export VPC_ID=$(aws ec2 describe-vpcs --filters Name=tag:Name,Values="${CLUSTER_NAME}" --query "Vpcs[].VpcId" --output text)

pip install -U "ray[default]"
```
pip 로 ray default 패키지를 설치하면 아래의 구성요소가 인스톨 된다.  
* Ray Dashboard: 클러스터의 상태, 리소스 사용량(CPU, GPU, 메모리), 로그 등을 웹 브라우저에서 실시간으로 확인할 수 있는 UI 도구입니다.
* Cluster Launcher: AWS, GCP, Azure 같은 클라우드 환경이나 Kubernetes에서 Ray 클러스터를 자동으로 생성하고 관리하는 CLI 도구입니다.
* 기타 필수 의존성:
  * grpcio: 노드 간 통신을 위한 라이브러리.
  * aiohttp / requests: 대시보드 서버 및 HTTP 통신용.
  * opencensus: 메트릭 및 분산 트레이싱 지원.
  * prometheus_client: 메트릭 수집용

### 2. ray Role 생성 ###
Ray Head 노드가 Worker 노드들을 생성/삭제할 수 있도록 ray-instance-profile을 생성한다.
```
cat <<EOF > ray-trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name ray-autoscaling-role \
  --assume-role-policy-document file://ray-trust-policy.json
cat <<EOF > ray-autoscaling-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:RunInstances",
        "ec2:TerminateInstances",
        "ec2:DescribeInstances",
        "ec2:DescribeSubnets",
        "ec2:CreateTags"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name ray-autoscaling-role \
  --policy-name ray-autoscaling-policy \
  --policy-document file://ray-autoscaling-policy.json

# 인스턴스 프로파일 생성
aws iam create-instance-profile --instance-profile-name ray-instance-profile

# 프로파일에 역할 추가
aws iam add-role-to-instance-profile \
  --instance-profile-name ray-instance-profile \
  --role-name ray-autoscaling-role
```

### 3. ray 클러스터 설정하기 ###
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

# IAM 권한 확인: 배스천 호스트의 Role에 ec2-instance-connect:OpenTunnel 권한이 있는지 확인.
# AWS CLI 버전 확인: aws --version 명령어로 v2인지 확인하세요. (EICE 터널링 기능은 v2에서 지원).
auth:
    ssh_user: ec2-user
    ssh_proxy_command: "aws ec2-instance-connect open-tunnel --instance-id %h"

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
        max_workers: 4                                 # 필요시 8대까지 자동 확장

head_node_type: head_node                              # 정의한 여러 노드 타입 중 어떤 것이 클러스터의 전체 제어를 담당할 '헤드'인지 확정
```
* CPU: 16 (물리적 자원): 실제 인스턴스의 하드웨어 스펙입니다. 태스크가 실행될 때마다 이 숫자가 차감되며, 0이 되면 더 이상 작업을 받지 않습니다.
* intel: 1 (논리적 태그): 사용자가 임의로 붙인 "이 노드는 인텔 칩셋임"이라는 인증 마크입니다. 물리적인 개수와 상관없이, 이 노드의 정체성을 나타내는 '입장권'이 1장 있다고 선언하는 것입니다.
ray 에서 하나의 task 는 하나의 코어를 점유한다.

클러스터를 생성한다.
```
ray up cluster.yaml -y
```

### 4.작업 제출 (Python 스크립트 실행): ###
```
ray job submit --address http://<헤드노드_사설IP>:8265 -- python data_job.py
```

### 5. 대시보드 접근 ###
```

```

### 참고 - 클러스터 삭제하기 ###
```
ray down cluster.yaml -y
```


