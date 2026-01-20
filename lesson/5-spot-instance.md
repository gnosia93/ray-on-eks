## 스팟 인스턴스 사용하기 ##

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

```
HEAD_SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=RayHeadSG" "Name=vpc-id,Values=$VPC_ID" \
  --query "SecurityGroups[0].GroupId" --output text)
echo "Head SG ID: $HEAD_SG_ID"

WORKER_SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=RayWorkerSG" "Name=vpc-id,Values=$VPC_ID" \
  --query "SecurityGroups[0].GroupId" --output text)
echo "Worker SG ID: $WORKER_SG_ID"

export PRIV_SUBNET_ID=$(aws ec2 describe-subnets \
    --filters "Name=tag:Name,Values=Ray-Private-Subnet" "Name=vpc-id,Values=${VPC_ID}" \
    --query "Subnets[*].{ID:SubnetId}" --output text)
echo "private subnet: ${PRIV_SUBNET_ID}"
```


### 2. 클러스터 설정파일 ###
```
cat <<EOF > cluster-spot.yaml
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
    - sudo growpart /dev/xvda 1 || true
    - sudo xfs_growfs -d / || true
    # - sudo resize2fs /dev/xvda1 || true
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
            # --- 스팟 인스턴스 설정 추가 시작 ---
            InstanceMarketOptions:
                MarketType: spot
            # --- 스팟 인스턴스 설정 추가 끝 ---
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
            # --- 스팟 인스턴스 설정 추가 시작 ---
            InstanceMarketOptions:
                MarketType: spot
            # --- 스팟 인스턴스 설정 추가 끝 ---
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

### 3. 클러스터 Update ###
```
ray up cluster-spot.yaml -y
```
[결과]
```
Cluster: ray-on-aws

Checking AWS environment settings
/home/ec2-user/.local/lib/python3.9/site-packages/boto3/compat.py:89: PythonDeprecationWarning: Boto3 will no longer support Python 3.9 starting April 29, 2026. To continue receiving service updates, bug fixes, and security updates please upgrade to Python 3.10 or later. More information can be found here: https://aws.amazon.com/blogs/developer/python-support-policy-updates-for-aws-sdks-and-tools/
  warnings.warn(warning, PythonDeprecationWarning)
AWS config
  IAM Profile: ray-instance-profile
  EC2 Key pair (all available node types): ray-autoscaler_1_ap-northeast-1 [default]
  VPC Subnets (all available node types): subnet-0624daf24debfc5de, subnet-0483c355addb47285 [default]
  EC2 Security groups (head_node): sg-0dd9628c2d0251c64
  EC2 Security groups (x86_worker_node): sg-0a6f304e7fe78fd1a
  EC2 Security groups (arm_worker_node): sg-0a6f304e7fe78fd1a
  EC2 AMI (head_node): ami-03d1820163e6b9f5d
  EC2 AMI (x86_worker_node): ami-03d1820163e6b9f5d
  EC2 AMI (arm_worker_node): ami-0a68dd57c124abef3

Updating cluster configuration and running full setup.
Cluster Ray runtime will be restarted. Confirm [y/N]: y [automatic, due to --yes]

Usage stats collection is enabled. To disable this, add `--disable-usage-stats` to the command that starts the cluster, or run the following command: `ray disable-usage-stats` before starting the cluster. See https://docs.ray.io/en/master/cluster/usage-stats.html for more details.

<1/1> Setting up head node
  Prepared bootstrap config
  Autoscaler v2 is now enabled by default (since Ray 2.50.0). To switch back to v1, set RAY_UP_enable_autoscaler_v2=0. This message can be suppressed by setting RAY_UP_enable_autoscaler_v2 explicitly.
  New status: waiting-for-ssh
  [1/7] Waiting for SSH to become available
    Running `uptime` as a test.
    Fetched IP: 10.0.2.183
 09:45:09 up  1:30,  2 users,  load average: 0.07, 0.17, 0.08
Shared connection to 10.0.2.183 closed.
    Success.
/home/ec2-user/.local/lib/python3.9/site-packages/boto3/compat.py:89: PythonDeprecationWarning: Boto3 will no longer support Python 3.9 starting April 29, 2026. To continue receiving service updates, bug fixes, and security updates please upgrade to Python 3.10 or later. More information can be found here: https://aws.amazon.com/blogs/developer/python-support-policy-updates-for-aws-sdks-and-tools/
  warnings.warn(warning, PythonDeprecationWarning)
  Updating cluster configuration. [hash=47f54d0b3cfc2b2e581da2003ca06a9591a15f6b]
  New status: syncing-files
  [2/7] Processing file mounts
Shared connection to 10.0.2.183 closed.
Shared connection to 10.0.2.183 closed.
  [3/7] No worker file mounts to sync
  New status: setting-up
  [4/7] No initialization commands to run.
  [5/7] Initializing command runner
  [6/7] Running setup commands
    (0/6) sudo growpart /dev/xvda 1 || t...
NOCHANGE: partition 1 is size 16752607. it cannot be grown
Shared connection to 10.0.2.183 closed.
    (1/6) sudo xfs_growfs -d / || true
meta-data=/dev/nvme0n1p1         isize=512    agcount=2, agsize=1047040 blks
         =                       sectsz=4096  attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=0, rmapbt=0
         =                       reflink=0    bigtime=1 inobtcount=1 nrext64=0
         =                       exchange=0  
data     =                       bsize=4096   blocks=2094075, imaxpct=25
         =                       sunit=128    swidth=128 blks
naming   =version 2              bsize=16384  ascii-ci=0, ftype=1, parent=0
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=4096  sunit=4 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
...

```


### 주의사항 및 고려사항 ###
* 헤드 노드는 On-Demand 사용
헤드 노드는 클러스터의 관리 센터 역할을 하므로, 갑자기 중단되면 전체 클러스터 작업이 실패한다.
* 스팟 인스턴스 중단 대비
스팟 인스턴스는 AWS의 여유 용량에 따라 언제든지 중단(Interrupt)될 수 있다. Ray는 이러한 중단에 대응하기 위해 작업 재시도(Task Retries) 및 액터 재시작(Actor Restarting) 기능을 내장하고 있다.
ray job submit으로 작업을 제출하면 기본적으로 실패한 태스크를 자동으로 재시도한다.
* 비용 절감
데이터 처리 작업처럼 유연하게 중단 및 재시작이 가능한 워크로드의 경우, 스팟 인스턴스를 사용하면 On-Demand 대비 최대 70~90%까지 비용을 절감할 수 있다.

### State Reconstruction (상태 재구성) ### 
* Ray는 작업의 계보(Lineage)를 기록하는데 특정 노드가 죽으면 해당 노드에서 실행 중이던 객체(Object)가 사라졌음을 감지하고, 다른 노드에서 해당 작업을 재실행(Re-execution)한다. 이는 Ray의 기본 결함 허용(Fault Tolerance) 메커니즘이다.
* 체크포인팅(ray.data.Dataset.checkpoint())은 중간 계산 결과를 메모리가 아닌 영구 저장소(Persistent Storage, 예: S3, 디스크)에 물리적인 파일로 저장하여, 재실행에 필요한 계산량 자체를 줄이는 역할을 한다.
* AWS로부터 Spot Termination Notice를 받으면, Ray Autoscaler는 즉시 새로운 인스턴스(Spot 혹은 On-demand)를 보충하도록 설계되어 있다.
