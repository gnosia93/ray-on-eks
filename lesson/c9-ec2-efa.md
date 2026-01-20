### 1. efa 인스턴스 조회 ###
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

### 2. 배치그룹(Placement Group) 생성 ###
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


### 2. node_config 수정 (EFA 활성화 및 배치 그룹) ###
EFA를 사용하려면 NetworkInterfaces 설정에서 InterfaceType: efa를 명시해야 하며, 성능을 위해 인스턴스들이 동일한 Placement Group에 있어야 한다.
```
...
available_node_types:
    x86_worker_node:
        node_config:
            InstanceType: ${INSTANCE_TYPE}
            NetworkInterfaces:
                - DeviceIndex: 0
                  InterfaceType: efa
                  Groups: ["${WORKER_SG_ID}"] # SecurityGroupIds 대신 여기서 설정
                  SubnetId: ${PRIV_SUBNET_ID}
                  DeleteOnTermination: true
            # Placement Group 추가 (미리 생성된 그룹 이름 입력)
            Placement:
                GroupName: ${PLACEMENT_GROUP_NAME} 
...
```

### 3. setup_commands 수정 (EFA 드라이버 설치) ###
Amazon Linux 2023 환경에서 EFA 드라이버와 통신 라이브러리(libfabric)를 설치해야 한다.
```
setup_commands:
    # ... 기존 명령어 유지 ...
    - |
      sudo dnf install -y gcc kernel-devel-$(uname -r)
      curl -O https://efa-installer.amazonaws.com
      tar -xf aws-efa-installer-latest.tar.gz
      cd aws-efa-installer && sudo ./efa_installer.sh -y
      # 환경 변수 로드
      source /etc/profile.d/efa.sh
    - pip install -U "ray[default,data]" pandas pyarrow boto3
```

### 4. 보안 그룹(Security Group) 필수 설정 ###
EFA가 작동하려면 동일한 보안 그룹 내의 모든 인바운드/아웃바운드 트래픽(All Traffic)이 자기 자신(Self-reference)을 대상으로 허용되어야 합니다. 기존 WORKER_SG 에 아래 내용을 추가한다. 
```
Type: All Traffic
Protocol: All
Source: ${WORKER_SG_ID} (자기 자신의 보안 그룹 ID)
```

### 5. 검증 ###

검노드 접속 후 fi_info -p efa 명령어로 EFA 장치가 인식되는지 확인한다.





