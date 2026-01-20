### 1. 환경 설정 ###
```
export AWS_REGION=$(aws ec2 describe-availability-zones --query 'AvailabilityZones[0].RegionName' --output text)
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export CLUSTER_NAME="ray-on-aws"
export VPC_ID=$(aws ec2 describe-vpcs --filters Name=tag:Name,Values="RayVPC" --query "Vpcs[].VpcId" --output text)
export X86_AMI_ID=$(aws ssm get-parameters --names /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --region ${AWS_REGION} --query "Parameters[0].Value" --output text)
export ARM_AMI_ID=$(aws ssm get-parameters --names /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-arm64 \
  --region ${AWS_REGION} --query "Parameters[0].Value" --output text)
```


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

aws iam create-role --role-name ray-autoscaling-role --assume-role-policy-document file://ray-trust-policy.json

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
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeImages",
        "ec2:DescribeVpcs",
        "ec2:CreateTags"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": "arn:aws:iam::${AWS_ACCOUNT_ID}:role/ray-autoscaling-role"
    }
  ]
}
EOF

aws iam put-role-policy --role-name ray-autoscaling-role --policy-name ray-autoscaling-policy \
  --policy-document file://ray-autoscaling-policy.json

# 인스턴스 프로파일 생성
aws iam create-instance-profile --instance-profile-name ray-instance-profile

# 프로파일에 역할 추가
aws iam add-role-to-instance-profile --instance-profile-name ray-instance-profile --role-name ray-autoscaling-role
```

### 3. ray 클러스터 설정하기 ###
ray 패키지를 설치한다. 
```
sudo dnf install -y python-unversioned-command
sudo dnf install -y python3-pip

pip install -U "ray[default]"
pip install boto3
```
프라이빗 서브넷의 ID 값을 가져온다.
```
export PRIV_SUBNET_ID=$(aws ec2 describe-subnets \
    --filters "Name=tag:Name,Values=Ray-Private-Subnet" "Name=vpc-id,Values=${VPC_ID}" \
    --query "Subnets[*].{ID:SubnetId}" --output text)

echo "private subnet: ${PRIV_SUBNET_ID}"
```

시큐리티 그룹을 생성한다. 
* 헤드 노드: 8265(대시보드), 6379(GCS), 10001(Ray Client) 포트가 열려 있어야 한다.
* 워커 노드: 헤드와 워커 노드 사이에는 모든 TCP 포트가 서로 통신 가능하도록 해당 보안 그룹이 자기 자신을 소스(Self-reference)로 허용해야 한다.
```
# 1. Head SG 생성
#HEAD_SG_ID=$(aws ec2 create-security-group --group-name RayHeadSG \
#  --description "Security group for Ray Head Node" --vpc-id $VPC_ID --query 'GroupId' --output text)

HEAD_SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=RayHeadSG" "Name=vpc-id,Values=$VPC_ID" \
  --query "SecurityGroups[0].GroupId" --output text)

if [ "$HEAD_SG_ID" == "None" ] || [ -z "$HEAD_SG_ID" ]; then
  echo "RayHeadSG가 존재하지 않아 새로 생성합니다..."
  HEAD_SG_ID=$(aws ec2 create-security-group \
    --group-name RayHeadSG \
    --description "Security group for Ray Head Node" \
    --vpc-id $VPC_ID \
    --query 'GroupId' --output text)
else
  echo "기존 RayHeadSG를 발견했습니다: $HEAD_SG_ID"
fi

echo "Head SG ID: $HEAD_SG_ID"


# 1. 기존 Worker SG ID 조회 시도
WORKER_SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=RayWorkerSG" "Name=vpc-id,Values=$VPC_ID" \
  --query "SecurityGroups[0].GroupId" --output text)

# 2. 존재하지 않을 경우 생성
if [ "$WORKER_SG_ID" == "None" ] || [ -z "$WORKER_SG_ID" ]; then
  echo "RayWorkerSG가 존재하지 않아 새로 생성합니다..."
  WORKER_SG_ID=$(aws ec2 create-security-group \
    --group-name RayWorkerSG \
    --description "Security group for Ray Worker Nodes" \
    --vpc-id $VPC_ID \
    --query 'GroupId' --output text)
else
  echo "기존 RayWorkerSG를 발견했습니다: $WORKER_SG_ID"
fi

echo "Worker SG ID: $WORKER_SG_ID"





# 2. Worker SG 생성
#WORKER_SG_ID=$(aws ec2 create-security-group --group-name RayWorkerSG \
#  --description "Security group for Ray Worker Nodes" --vpc-id $VPC_ID --query 'GroupId' --output text)

#echo "Head SG: $HEAD_SG_ID / Worker SG: $WORKER_SG_ID"


# A. 내부 통신 허용: Head <-> Worker (모든 TCP 포트)
aws ec2 authorize-security-group-ingress --group-id $HEAD_SG_ID \
  --protocol tcp --port 0-65535 --source-group $WORKER_SG_ID

aws ec2 authorize-security-group-ingress --group-id $WORKER_SG_ID \
  --protocol tcp --port 0-65535 --source-group $HEAD_SG_ID

# B. 자기 참조: Worker끼리 통신 허용 (데이터 셔플 등)
aws ec2 authorize-security-group-ingress --group-id $WORKER_SG_ID \
  --protocol tcp --port 0-65535 --source-group $WORKER_SG_ID

# C. Bastion 호스트의 접근 허용 (관리용)
BASTION_SG_ID=$(aws ec2 describe-security-groups --filters "Name=tag:Name,Values=BastionSG" --query "SecurityGroups[0].GroupId" --output text)

# Head 노드: SSH(22), Dashboard(8265), Client(10001) 허용
aws ec2 authorize-security-group-ingress --group-id $HEAD_SG_ID \
  --protocol tcp --port 22 --source-group $BASTION_SG_ID

aws ec2 authorize-security-group-ingress --group-id $HEAD_SG_ID \
  --protocol tcp --port 8265 --source-group $BASTION_SG_ID

aws ec2 authorize-security-group-ingress --group-id $HEAD_SG_ID \
  --protocol tcp --port 10001 --source-group $BASTION_SG_ID

# Worker 노드: SSH(22) 허용
aws ec2 authorize-security-group-ingress --group-id $WORKER_SG_ID \
  --protocol tcp --port 22 --source-group $BASTION_SG_ID
```


```
cat <<EOF > cluster.yaml
cluster_name: ${CLUSTER_NAME}

provider:
    type: aws
    region: ${AWS_REGION}                              
    availability_zone: ${AWS_REGION}a
    use_internal_ips: true

# ssh_private_key를 명시하지 않으면 SSH Agent의 키를 자동으로 사용합니다.
auth:
    ssh_user: ec2-user
    
# 각 노드에서 실행될 설정 (Python 설치 등)
setup_commands:
    - sudo dnf install -y python-unversioned-command
    - sudo dnf install -y python3-pip
    - pip install -U "ray[default,data]" pandas pyarrow boto3

max_workers: 16

# 노드별 상세 사양
available_node_types:
    # 헤드 노드
    head_node:
        resources: {"CPU": 4, "Intel": 1}              # 스케줄링 힌트 제공
        node_config:
            InstanceType: m7i.xlarge
            ImageId: ${X86_AMI_ID}                     # amazon linux 2023
            SubnetId: ${PRIV_SUBNET_ID}                # 프라이빗 서브넷 ID 입력
            SecurityGroupIds:                          # 필요한 경우 보안 그룹 ID도 명시
                - ${HEAD_SG_ID}
            IamInstanceProfile:
                Name: ray-instance-profile            
        min_workers: 0                                 # 헤드 노드는 관리용이므로 워커 역할을 담당하지 않는다.
        max_workers: 0                             
    # Intel 워커 노드 (데이터 처리용)
    x86_worker_node:
        resources: {"CPU": 4, "Intel": 1}              # 스케줄링 힌트 제공
        node_config:
            InstanceType: m7i.2xlarge
            ImageId: ${X86_AMI_ID}
            SubnetId: ${PRIV_SUBNET_ID}                # 프라이빗 서브넷 ID 입력
            SecurityGroupIds:                          # 필요한 경우 보안 그룹 ID도 명시
                - ${WORKER_SG_ID}
            IamInstanceProfile:
                Name: ray-instance-profile            
        min_workers: 4                                 # 기본 4대 실행
        max_workers: 8                                 # 필요시 8대까지 자동 확장

    # Graviton 워커 노드 (데이터 처리용)
    arm_worker_node:
        resources: {"CPU": 4, "Graviton": 1}           # 스케줄링 힌트 제공
        node_config:
            InstanceType: m8g.2xlarge
            ImageId: ${ARM_AMI_ID}                     # 헤드 노드와 동일한 이미지 사용
            SubnetId: ${PRIV_SUBNET_ID}                # 프라이빗 서브넷 ID 입력
            SecurityGroupIds:                          
                - ${WORKER_SG_ID}
            IamInstanceProfile:
                Name: ray-instance-profile            
        min_workers: 4                                 # 기본 4대 실행
        max_workers: 8                                 # 필요시 8대까지 자동 확장

head_node_type: head_node                              # 정의한 여러 노드 타입 중 어떤 것이 클러스터의 전체 제어를 담당할 '헤드'인지 확정
EOF
```
* CPU: 16 (물리적 자원) 실제 인스턴스의 하드웨어 스펙으로 태스크가 실행될 때마다 차감되며, 0이 되면 더 이상 작업이 할당되지 않는다. 
* Graviton: 1 (논리적 태그) 사용자가 임의로 붙인 "이 노드는 그라비톤 칩셋임" 이라는 마크이다. 

ray 클러스터를 생성한다. ssh-add는 SSH Key 전용 메모리 저장소인 ssh-agent에 사용자의 비밀키(.pem 등)를 등록하는 명령어로, 로컬 PC의 키를 배스천에 물리적으로 복사하지 않고도 배스천을 거쳐 내부 노드(Head/Worker)에 접속할 수 있게 인증 정보를 중계해 준다.
```
# 로컬 PC에서 실행
ssh-add ~/your-aws-key.pem
ssh-add -l
# ssh-add -D  모든 키 삭제, 로그아웃시 권장
# eval $(ssh-agent) ssh-agent 확인

VS_CODE=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=Ray-Bastion-VSCode" "Name=instance-state-name,Values=running" \
  --query "Reservations[].Instances[0].PublicDnsName" --output text)

ssh -A ec2-user@${VS_CODE}
 
ray up cluster.yaml -y
```

### 4. 클러스터 확인 ###
#### ray status ####
```
ray exec ~/cluster.yaml "ray status"
```
[결과]
```
Loaded cached provider configuration
If you experience issues with the cloud provider, try re-running the command with --no-config-cache.
/home/ec2-user/.local/lib/python3.9/site-packages/boto3/compat.py:89: PythonDeprecationWarning: Boto3 will no longer support Python 3.9 starting April 29, 2026. To continue receiving service updates, bug fixes, and security updates please upgrade to Python 3.10 or later. More information can be found here: https://aws.amazon.com/blogs/developer/python-support-policy-updates-for-aws-sdks-and-tools/
  warnings.warn(warning, PythonDeprecationWarning)
Fetched IP: 10.0.2.177
======== Autoscaler status: 2026-01-20 02:34:48.560347 ========
Node status
---------------------------------------------------------------
Active:
 (no active nodes)
Idle:
 4 arm_worker_node
 4 x86_worker_node
 1 head_node
Pending:
 (no pending nodes)
Recent failures:
 (no failures)

Resources
---------------------------------------------------------------
Total Usage:
 0.0/36.0 CPU
 0.0/4.0 Graviton
 0.0/5.0 Intel
 0B/189.07GiB memory
 0B/76.43GiB object_store_memory

From request_resources:
 (none)
Pending Demands:
 (no resource demands)
Shared connection to 10.0.2.177 closed.
```

#### ray list nodes --detail ####
```
ray exec ~/cluster.yaml "ray list nodes --detail"
```
[결과]
```
Loaded cached provider configuration
If you experience issues with the cloud provider, try re-running the command with --no-config-cache.
/home/ec2-user/.local/lib/python3.9/site-packages/boto3/compat.py:89: PythonDeprecationWarning: Boto3 will no longer support Python 3.9 starting April 29, 2026. To continue receiving service updates, bug fixes, and security updates please upgrade to Python 3.10 or later. More information can be found here: https://aws.amazon.com/blogs/developer/python-support-policy-updates-for-aws-sdks-and-tools/
  warnings.warn(warning, PythonDeprecationWarning)
Fetched IP: 10.0.2.177
Warning: Permanently added '10.0.2.177' (ED25519) to the list of known hosts.
---
-   node_id: 29a5b1bae8b5ef92dde03ffd5d53373e67550c7af77d4917985cbb30
    node_ip: 10.0.2.138
    is_head_node: false
    state: ALIVE
    state_message: null
    node_name: 10.0.2.138
    resources_total:
        object_store_memory: 9.022 GiB
        memory: 22.400 GiB
        node:10.0.2.138: 1.0
        Graviton: 1.0
        CPU: 4.0
    labels:
        ray.io/node-id: 29a5b1bae8b5ef92dde03ffd5d53373e67550c7af77d4917985cbb30
        ray.io/node-group: worker_node
    start_time_ms: '2026-01-20 02:18:19.716000'
    end_time_ms: '1970-01-01 00:00:00'
-   node_id: 6f82cca035c93590123b5198184aa7a4cf67d1e119cebb22a5df0c09
    node_ip: 10.0.2.233
    is_head_node: false
    state: ALIVE
    state_message: null
    node_name: 10.0.2.233
    resources_total:
        node:10.0.2.233: 1.0
        object_store_memory: 9.020 GiB
        memory: 22.400 GiB
        Graviton: 1.0
        CPU: 4.0
    labels:
        ray.io/node-id: 6f82cca035c93590123b5198184aa7a4cf67d1e119cebb22a5df0c09
        ray.io/node-group: worker_node
    start_time_ms: '2026-01-20 02:18:37.727000'
    end_time_ms: '1970-01-01 00:00:00'
-   node_id: cf44b45fa911750cf543d3a0c79f03eb5bf8a01902358240dd0e134d
    node_ip: 10.0.2.191
    is_head_node: false
    state: ALIVE
    state_message: null
    node_name: 10.0.2.191
    resources_total:
        node:10.0.2.191: 1.0
        object_store_memory: 9.020 GiB
        memory: 22.400 GiB
        Graviton: 1.0
        CPU: 4.0
    labels:
        ray.io/node-id: cf44b45fa911750cf543d3a0c79f03eb5bf8a01902358240dd0e134d
        ray.io/node-group: worker_node
    start_time_ms: '2026-01-20 02:18:32.943000'
    end_time_ms: '1970-01-01 00:00:00'
-   node_id: d13964359101be8ca107b6bc5cb096241c4c85efca8ef49a761b36f1
    node_ip: 10.0.2.212
    is_head_node: false
    state: ALIVE
    state_message: null
    node_name: 10.0.2.212
    resources_total:
        object_store_memory: 9.022 GiB
        memory: 22.400 GiB
        node:10.0.2.212: 1.0
        Graviton: 1.0
        CPU: 4.0
    labels:
        ray.io/node-id: d13964359101be8ca107b6bc5cb096241c4c85efca8ef49a761b36f1
        ray.io/node-group: worker_node
    start_time_ms: '2026-01-20 02:18:06.619000'
    end_time_ms: '1970-01-01 00:00:00'
-   node_id: de92eb1e441abf36ed1970f9084458d412154ad83063fad06d3e0711
    node_ip: 10.0.2.177
    is_head_node: true
    state: ALIVE
    state_message: null
    node_name: 10.0.2.177
    resources_total:
        Intel: 1.0
        object_store_memory: 4.232 GiB
        memory: 9.876 GiB
        node:__internal_head__: 1.0
        node:10.0.2.177: 1.0
        CPU: 4.0
    labels:
        ray.io/node-id: de92eb1e441abf36ed1970f9084458d412154ad83063fad06d3e0711
        ray.io/node-group: head_node
    start_time_ms: '2026-01-20 02:17:01.604000'
    end_time_ms: '1970-01-01 00:00:00'
```


### 5. 대시보드 접근 ###
```
# 로컬 PC가 아닌 '배스천 호스트' 터미널에서 실행
ray dashboard /home/ec2-user/cluster.yaml

# 로컬 PC 에서 실행
ssh -i "키파일.pem" -L 8265:localhost:8265 ec2-user@${VS_CODE}
```
로컬 PC 의 웹브라우저로 http://localhost:8265 로 접속한다. 
![](https://github.com/gnosia93/ray-on-aws/blob/main/lesson/images/ray-dashboard.png)


### 참고 - 클러스터 삭제하기 ###
```
ray down cluster.yaml -y
```


