### 1. 환경설정 ###
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

export PRIV_SUBNET_ID=$(aws ec2 describe-subnets \
    --filters "Name=tag:Name,Values=Ray-Private-Subnet" "Name=vpc-id,Values=${VPC_ID}" \
    --query "Subnets[*].{ID:SubnetId}" --output text)

echo "private subnet: ${PRIV_SUBNET_ID}"
```

### 2. efa 인스턴스 조회 ###
```
 aws ec2 describe-instance-types \
    --filters "Name=instance-type,Values=*7i*,*8g*,*7a*" \
              "Name=network-info.efa-supported,Values=true" \
    --query "InstanceTypes[].[InstanceType, VCpuInfo.DefaultVCpus, VCpuInfo.DefaultCores, MemoryInfo.SizeInMiB]" \
    --output text | awk '{printf "Instance: %-18s | vCPU: %3d | Cores: %2d | Memory: %6d GiB\n", $1, $2, $3, $4/1024}'
```
[결과]
```
Instance: r7i.metal-48xl     | vCPU: 192 | Cores: 96 | Memory:   1536 GiB
Instance: u7i-6tb.112xlarge  | vCPU: 448 | Cores: 224 | Memory:   6144 GiB
Instance: m8g.24xlarge       | vCPU:  96 | Cores: 96 | Memory:    384 GiB
Instance: r8g.metal-24xl     | vCPU:  96 | Cores: 96 | Memory:    768 GiB
Instance: m8g.metal-48xl     | vCPU: 192 | Cores: 192 | Memory:    768 GiB
Instance: c8g.48xlarge       | vCPU: 192 | Cores: 192 | Memory:    384 GiB
Instance: c8g.metal-48xl     | vCPU: 192 | Cores: 192 | Memory:    384 GiB
Instance: c8g.24xlarge       | vCPU:  96 | Cores: 96 | Memory:    192 GiB
Instance: u7i-12tb.224xlarge | vCPU: 896 | Cores: 448 | Memory:  12288 GiB
Instance: m7i.48xlarge       | vCPU: 192 | Cores: 96 | Memory:    768 GiB
Instance: r8g.48xlarge       | vCPU: 192 | Cores: 192 | Memory:   1536 GiB
Instance: r8g.24xlarge       | vCPU:  96 | Cores: 96 | Memory:    768 GiB
Instance: u7in-16tb.224xlarge | vCPU: 896 | Cores: 448 | Memory:  16384 GiB
Instance: c7i.metal-48xl     | vCPU: 192 | Cores: 96 | Memory:    384 GiB
Instance: u7i-8tb.112xlarge  | vCPU: 448 | Cores: 224 | Memory:   8192 GiB
Instance: m8g.48xlarge       | vCPU: 192 | Cores: 192 | Memory:    768 GiB
Instance: i8g.48xlarge       | vCPU: 192 | Cores: 192 | Memory:   1536 GiB
Instance: c8g.metal-24xl     | vCPU:  96 | Cores: 96 | Memory:    192 GiB
Instance: r8g.metal-48xl     | vCPU: 192 | Cores: 192 | Memory:   1536 GiB
Instance: i7i.metal-48xl     | vCPU: 192 | Cores: 96 | Memory:   1536 GiB
Instance: i7i.24xlarge       | vCPU:  96 | Cores: 48 | Memory:    768 GiB
Instance: r7i.48xlarge       | vCPU: 192 | Cores: 96 | Memory:   1536 GiB
Instance: m8g.metal-24xl     | vCPU:  96 | Cores: 96 | Memory:    384 GiB
Instance: i7i.48xlarge       | vCPU: 192 | Cores: 96 | Memory:   1536 GiB
Instance: m7i.metal-48xl     | vCPU: 192 | Cores: 96 | Memory:    768 GiB
Instance: c7i.48xlarge       | vCPU: 192 | Cores: 96 | Memory:    384 GiB
```

### 3. 배치그룹(Placement Group) 생성 ###
```
aws ec2 create-placement-group --group-name ray-placement-group --strategy cluster 
```
[결과]
```
{
    "PlacementGroup": {
        "GroupName": "ray-placement-group",
        "State": "available",
        "Strategy": "cluster",
        "GroupId": "pg-099e0e184fa11f7ec",
        "GroupArn": "arn:aws:ec2:ap-northeast-1:499514681453:placement-group/ray-placement-group"
    }
}
```

### 4. 보안 그룹(Security Group) 수정 ###
EFA가 작동하려면 동일한 보안 그룹 내의 모든 인바운드/아웃바운드 트래픽(All Traffic)이 자기 자신(Self-reference)을 대상으로 허용되어야 한다.
기존 워커노드 시큐리티 그룹에 추가한다.
```
HEAD_SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=RayHeadSG" "Name=vpc-id,Values=$VPC_ID" \
  --query "SecurityGroups[0].GroupId" --output text)

# 기존 Worker SG ID 조회 시도
WORKER_SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=RayWorkerSG" "Name=vpc-id,Values=$VPC_ID" \
  --query "SecurityGroups[0].GroupId" --output text)

aws ec2 authorize-security-group-ingress --group-id ${WORKER_SG_ID} \
  --protocol -1 --source-group ${WORKER_SG_ID}

aws ec2 authorize-security-group-egress --group-id ${WORKER_SG_ID} \
  --protocol -1 --source-group ${WORKER_SG_ID}
```

### 5. EFA 드라이버 설치 ###
Amazon Linux 2023 환경에서 EFA 드라이버와 통신 라이브러리(libfabric)를 설치해야 한다.
```
curl -O https://efa-installer.amazonaws.com/aws-efa-installer-1.45.1.tar.gz
tar -xf aws-efa-installer-1.45.1.tar.gz
cd aws-efa-installer && sudo ./efa_installer.sh -y
```
cluster.yaml 의 setup_commands 블록에 위의 스크립트를 입력한다. 

### cluster-efa.yaml 작성 ###
EFA를 사용하려면 NetworkInterfaces 설정에서 InterfaceType: efa를 명시해야 하며, 성능을 위해 인스턴스들이 동일한 Placement Group에 있어야 한다.
```
cat <<EOF > cluster-efa.yaml
cluster_name: ${CLUSTER_NAME}-efa

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
    - curl -O https://efa-installer.amazonaws.com/aws-efa-installer-1.45.1.tar.gz
    - tar -xf aws-efa-installer-1.45.1.tar.gz
    - cd aws-efa-installer && sudo ./efa_installer.sh -y
    - sudo dnf install -y python-unversioned-command
    - sudo dnf install -y python3-pip
    - pip install -U "ray[default,data]" pandas pyarrow boto3

max_workers: 96

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
   
    efa_worker_node:
        node_config:
            InstanceType: i7i.24xlarge
            NetworkInterfaces:
                - DeviceIndex: 0
                  InterfaceType: efa
                  Groups: ["${WORKER_SG_ID}"]                # SecurityGroupIds 대신 여기서 설정
                  SubnetId: ${PRIV_SUBNET_ID}                # 프라이빗 서브넷 ID 입력
            Placement:
                GroupName: ray-placement-group
            BlockDeviceMappings:
                - DeviceName: /dev/xvda         # Amazon Linux 2023의 기본 루트 장치명
                  Ebs:
                      VolumeSize: 300           # GB 단위 (예: 300GB)
                      VolumeType: gp3           # 가성비 좋은 gp3 권장
            ImageId: ${X86_AMI_ID}                     # 헤드 노드와 동일한 이미지 사용         
            IamInstanceProfile:
                Name: ray-instance-profile            
        min_workers: 1                                 # 기본 1대 실행
        max_workers: 2                                

head_node_type: head_node                              # 정의한 여러 노드 타입 중 어떤 것이 클러스터의 전체 제어를 담당할 '헤드'인지 확정
EOF
```

(주의) ray 클러스터는 로컬 PC 에서 생성해야 한다.
```
# 로컬 PC에서 실행
ssh-add ~/your-aws-key.pem
ssh-add -l
# ssh-add -D  모든 키 삭제, 로그아웃시 권장
# eval $(ssh-agent) ssh-agent 확인

VS_CODE=$(cat VS_CODE) && echo ${VS_CODE}
ssh -A ec2-user@${VS_CODE}
 
ray up cluster-efa.yaml -y
```

### 검증 ###
검노드 접속 후 fi_info -p efa 명령어로 EFA 장치가 인식되는지 확인한다.



## 레퍼런스 ##
* https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/efa-start.html#efa-start-enable



