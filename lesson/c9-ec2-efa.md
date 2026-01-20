
### 1. node_config 수정 (EFA 활성화 및 배치 그룹) ###
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

### 2. setup_commands 수정 (EFA 드라이버 설치) ###
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

### 3. 보안 그룹(Security Group) 필수 설정 ###
EFA가 작동하려면 동일한 보안 그룹 내의 모든 인바운드/아웃바운드 트래픽(All Traffic)이 자기 자신(Self-reference)을 대상으로 허용되어야 합니다. 기존 WORKER_SG 에 아래 내용을 추가한다. 
```
Type: All Traffic
Protocol: All
Source: ${WORKER_SG_ID} (자기 자신의 보안 그룹 ID)
```

### 4. 배치 그룹 ###
```
aws ec2 create-placement-group --group-name <이름> --strategy cluster 명령으로 미리 생성해두는 것이 좋습니다.
```

### 5. 검증 ###

검노드 접속 후 fi_info -p efa 명령어로 EFA 장치가 인식되는지 확인한다.





