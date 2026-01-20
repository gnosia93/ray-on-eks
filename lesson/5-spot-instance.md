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
### 3. 실행하기 ###
```
ray up -f cluster-spot.yaml -y
```

### 주의사항 및 고려사항 ###
* 헤드 노드는 On-Demand 사용:
헤드 노드는 클러스터의 관리 센터 역할을 하므로, 갑자기 중단되면 전체 클러스터 작업이 실패한다.

* 스팟 인스턴스 중단 대비:
스팟 인스턴스는 AWS의 여유 용량에 따라 언제든지 중단(Interrupt)될 수 있다. Ray는 이러한 중단에 대응하기 위해 작업 재시도(Task Retries) 및 액터 재시작(Actor Restarting) 기능을 내장하고 있다.
ray job submit으로 작업을 제출하면 기본적으로 실패한 태스크를 자동으로 재시도한다.

* 비용 절감:
데이터 처리 작업처럼 유연하게 중단 및 재시작이 가능한 워크로드의 경우, 스팟 인스턴스를 사용하면 On-Demand 대비 최대 70~90%까지 비용을 절감할 수 있다.


[C5] Spot 인스턴스 회수 시 Ray의 동작 원리 질문하신 "어떻게 잘 동작하나?"에 대한 교육 포인트는 다음과 같습니다. State Reconstruction (상태 재구성): Ray는 작업의 계보(Lineage)를 기록합니다. 특정 노드가 죽으면 해당 노드에서 실행 중이던 객체(Object)가 사라졌음을 감지하고, 다른 노드에서 해당 작업을 재실행(Re-execution)합니다. Object Spilling (객체 유출): 메모리가 부족하거나 노드가 사라질 것에 대비해 데이터를 S3 등에 백업해두는 기능을 활용합니다. Autoscaler 연동: AWS로부터 Spot Termination Notice를 받으면, Ray Autoscaler는 즉시 새로운 인스턴스(Spot 혹은 On-demand)를 보충하도록 설계되어 있습니다.
