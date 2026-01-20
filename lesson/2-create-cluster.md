vs_code 터미널에서 아래 명령어를 순차적으로 실행한다. 

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

echo ${AWS_REGION}
echo ${AWS_ACCOUNT_ID}
echo ${CLUSTER_NAME}
echo ${VPC_ID}
echo ${X86_AMI_ID}
echo ${ARM_AMI_ID}
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

# s3 접근 권한 설정
aws iam attach-role-policy --role-name ray-autoscaling-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
```

### 3. ray 환경 설정 ###
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
# 기존 Head SG ID 조회 시도
HEAD_SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=RayHeadSG" "Name=vpc-id,Values=$VPC_ID" \
  --query "SecurityGroups[0].GroupId" --output text)

# 존재하지 않을 경우 생성
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

# 기존 Worker SG ID 조회 시도
WORKER_SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=RayWorkerSG" "Name=vpc-id,Values=$VPC_ID" \
  --query "SecurityGroups[0].GroupId" --output text)

# 존재하지 않을 경우 생성
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
```

시큐리티 그룹의 세부 정책(ingress/egress)을 생성한다. 
```
# A. 내부 통신 허용: Head <-> Worker (모든 TCP 포트)
aws ec2 authorize-security-group-ingress --group-id $HEAD_SG_ID \
  --protocol tcp --port 0-65535 --source-group $WORKER_SG_ID

aws ec2 authorize-security-group-ingress --group-id $WORKER_SG_ID \
  --protocol tcp --port 0-65535 --source-group $HEAD_SG_ID

# B. 자기 참조: Worker끼리 통신 허용 (데이터 셔플 등)
aws ec2 authorize-security-group-ingress --group-id $WORKER_SG_ID \
  --protocol tcp --port 0-65535 --source-group $WORKER_SG_ID

# C. Bastion 호스트의 접근 허용 (관리용)
BASTION_SG_ID=$(aws ec2 describe-security-groups --filters "Name=tag:aws:cloudformation:logical-id,Values=BastionSG" --query "SecurityGroups[0].GroupId" --output text)

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

### 4. ray 컨피그 파일 생성 ###
```
cat <<EOF > cluster.yaml
cluster_name: ${CLUSTER_NAME}

provider:
    type: aws
    region: ${AWS_REGION}                              
    availability_zone: ${AWS_REGION}a
    use_internal_ips: true
    cache_stopped_nodes: False              # ray down 시 인스턴스 모두 삭제

# ssh_private_key를 명시하지 않으면 SSH Agent의 키를 자동으로 사용합니다.
auth:
    ssh_user: ec2-user
    
# 각 노드에서 실행될 설정 (Python 설치 등)
setup_commands:
    - sudo dnf install -y python-unversioned-command
    - sudo dnf install -y python3-pip
    - pip install -U "ray[default,data]" pandas pyarrow boto3

max_workers: 128

# 노드별 상세 사양
available_node_types:
    # 헤드 노드
    head_node:
        resources: {"CPU": 0, "Intel": 1}              # "물리적으론 4코어지만, CPU 갯수를 0 으로 설정하여 Ray 가 워커로드로 쓰는 것을 방지함.
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
        resources: {"CPU": 8, "Intel": 1}              # 스케줄링 힌트 제공
        node_config:
            InstanceType: r7i.2xlarge
            BlockDeviceMappings:
                - DeviceName: /dev/xvda  # Amazon Linux 2023의 기본 루트 장치명
                  Ebs:
                      VolumeSize: 300    # GB 단위 (예: 300GB)
                      VolumeType: gp3    # 가성비 좋은 gp3 권장
            ImageId: ${X86_AMI_ID}
            SubnetId: ${PRIV_SUBNET_ID}                # 프라이빗 서브넷 ID 입력
            SecurityGroupIds:                          # 필요한 경우 보안 그룹 ID도 명시
                - ${WORKER_SG_ID}
            IamInstanceProfile:
                Name: ray-instance-profile            
        min_workers: 1                                 # 기본 1대 실행
        max_workers: 8                                 # 필요시 8대까지 자동 확장

    # Graviton 워커 노드 (데이터 처리용)
    arm_worker_node:
        resources: {"CPU": 8, "Graviton": 1}           # 스케줄링 힌트 제공
        node_config:
            InstanceType: r8g.2xlarge
            BlockDeviceMappings:
                - DeviceName: /dev/xvda  # Amazon Linux 2023의 기본 루트 장치명
                  Ebs:
                      VolumeSize: 300    # GB 단위 (예: 300GB)
                      VolumeType: gp3    # 가성비 좋은 gp3 권장
            ImageId: ${ARM_AMI_ID}                     # 헤드 노드와 동일한 이미지 사용
            SubnetId: ${PRIV_SUBNET_ID}                # 프라이빗 서브넷 ID 입력
            SecurityGroupIds:                          
                - ${WORKER_SG_ID}
            IamInstanceProfile:
                Name: ray-instance-profile            
        min_workers: 1                                 # 기본 1대 실행
        max_workers: 8                                 # 필요시 8대까지 자동 확장

head_node_type: head_node                              # 정의한 여러 노드 타입 중 어떤 것이 클러스터의 전체 제어를 담당할 '헤드'인지 확정
EOF
```
* CPU: 16 (물리적 자원) 실제 인스턴스의 하드웨어 스펙으로 태스크가 실행될 때마다 차감되며, 0이 되면 더 이상 작업이 할당되지 않는다. 
* Graviton: 1 (논리적 태그) 사용자가 임의로 붙인 "이 노드는 그라비톤 칩셋임" 이라는 마크이다. 


### 5. ray 클러스터 생성 ###
#### (주의) ray 클러스터는 로컬 PC 에서 생성해야 한다. #### 
```
# 로컬 PC에서 실행
ssh-add ~/your-aws-key.pem
ssh-add -l
# ssh-add -D  모든 키 삭제, 로그아웃시 권장
# eval $(ssh-agent) ssh-agent 확인

VS_CODE=$(cat VS_CODE) && echo ${VS_CODE}
ssh -A ec2-user@${VS_CODE}
 
ray up cluster.yaml -y
```
* ssh-add는 SSH Key 전용 메모리 저장소인 ssh-agent에 사용자의 비밀키(.pem 등)를 등록하는 명령어로, 로컬 PC의 키를 배스천에 물리적으로 복사하지 않고도 배스천을 거쳐 내부 노드(Head/Worker)에 접속할 수 있게 인증 정보를 중계해 준다.

![](https://github.com/gnosia93/ray-on-ec2/blob/main/lesson/images/ray-headnode.png)
콘솔에서 Head 노드가 생성된 것을 확인한다.

### 6. 클러스터 확인 ###
#### ray 클러스터 상태확인 ####
```
ray exec ~/cluster.yaml "ray status"
```
[결과]
```
Loaded cached provider configuration
If you experience issues with the cloud provider, try re-running the command with --no-config-cache.
/home/ec2-user/.local/lib/python3.9/site-packages/boto3/compat.py:89: PythonDeprecationWarning: Boto3 will no longer support Python 3.9 starting April 29, 2026. To continue receiving service updates, bug fixes, and security updates please upgrade to Python 3.10 or later. More information can be found here: https://aws.amazon.com/blogs/developer/python-support-policy-updates-for-aws-sdks-and-tools/
  warnings.warn(warning, PythonDeprecationWarning)
Fetched IP: 10.0.2.183
Warning: Permanently added '10.0.2.183' (ED25519) to the list of known hosts.
======== Autoscaler status: 2026-01-20 08:20:02.391213 ========
Node status
---------------------------------------------------------------
Active:
 (no active nodes)
Idle:
 1 head_node
 1 x86_worker_node
 1 arm_worker_node
Pending:
 (no pending nodes)
Recent failures:
 (no failures)

Resources
---------------------------------------------------------------
Total Usage:
 0.0/16.0 CPU
 0.0/1.0 Graviton
 0.0/2.0 Intel
 0B/99.44GiB memory
 0B/40.69GiB object_store_memory

From request_resources:
 (none)
Pending Demands:
 (no resource demands)
Shared connection to 10.0.2.183 closed.
```

#### 노드 리스트 확인 ####
```
ray exec ~/cluster.yaml "ray list nodes --detail"
```
[결과]
```
Loaded cached provider configuration
If you experience issues with the cloud provider, try re-running the command with --no-config-cache.
/home/ec2-user/.local/lib/python3.9/site-packages/boto3/compat.py:89: PythonDeprecationWarning: Boto3 will no longer support Python 3.9 starting April 29, 2026. To continue receiving service updates, bug fixes, and security updates please upgrade to Python 3.10 or later. More information can be found here: https://aws.amazon.com/blogs/developer/python-support-policy-updates-for-aws-sdks-and-tools/
  warnings.warn(warning, PythonDeprecationWarning)
Fetched IP: 10.0.2.183
Warning: Permanently added '10.0.2.183' (ED25519) to the list of known hosts.
---
-   node_id: a5489a002772f019149a22ec65cbf917864e8ffb10ce02631f9da83e
    node_ip: 10.0.2.183
    is_head_node: true
    state: ALIVE
    state_message: null
    node_name: 10.0.2.183
    resources_total:
        node:__internal_head__: 1.0
        memory: 9.837 GiB
        node:10.0.2.183: 1.0
        object_store_memory: 4.216 GiB
        Intel: 1.0
    labels:
        ray.io/node-group: head_node
        ray.io/node-id: a5489a002772f019149a22ec65cbf917864e8ffb10ce02631f9da83e
    start_time_ms: '2026-01-20 08:16:24.327000'
    end_time_ms: '1970-01-01 00:00:00'
-   node_id: dc74d1c02e1e35be5bca4c1d83d44a8f7a9c51ad9ad7d6657eec6da1
    node_ip: 10.0.2.133
    is_head_node: false
    state: ALIVE
    state_message: null
    node_name: 10.0.2.133
    resources_total:
        node:10.0.2.133: 1.0
        CPU: 8.0
        memory: 44.800 GiB
        Graviton: 1.0
        object_store_memory: 18.224 GiB
    labels:
        ray.io/node-group: arm_worker_node
        ray.io/node-id: dc74d1c02e1e35be5bca4c1d83d44a8f7a9c51ad9ad7d6657eec6da1
    start_time_ms: '2026-01-20 08:17:55.672000'
    end_time_ms: '1970-01-01 00:00:00'
-   node_id: dc8aed121d806e30805d72f2755f2427f7ca5632579cced62d0e6b10
    node_ip: 10.0.2.101
    is_head_node: false
    state: ALIVE
    state_message: null
    node_name: 10.0.2.101
    resources_total:
        CPU: 8.0
        Intel: 1.0
        memory: 44.800 GiB
        object_store_memory: 18.252 GiB
        node:10.0.2.101: 1.0
    labels:
        ray.io/node-group: x86_worker_node
        ray.io/node-id: dc8aed121d806e30805d72f2755f2427f7ca5632579cced62d0e6b10
    start_time_ms: '2026-01-20 08:17:44.758000'
    end_time_ms: '1970-01-01 00:00:00'
...

Shared connection to 10.0.2.183 closed.
```


### 7. 대시보드 접근 ###
vs-code 터미널로 이동하여 아래 명령어를 실행한다. 
```
ray dashboard /home/ec2-user/cluster.yaml
```

아래 터널링 명령어는 로컬 PC 에서 실행한다. 
```
cd ~/ray-on-ec2
ssh -i ~/키페어.pem -L 8265:localhost:8265 ec2-user@$(cat VS_CODE)
```
로컬 PC 의 웹브라우저로 http://localhost:8265 로 접속한다. 
![](https://github.com/gnosia93/ray-on-aws/blob/main/lesson/images/ray-dashboard.png)


## 클러스터 삭제하기 ##
클러스터 삭제는 vs_code 터미널에서 수행한다. 
```
ray down cluster.yaml -y
```


