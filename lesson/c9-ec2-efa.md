
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








EC2 EFA 설정은 EFA를 지원하는 인스턴스 타입 선택, 클러스터 배치 그룹 생성, 서브넷에서 EFA 활성화, 보안 그룹 설정, AMI 생성 및 인스턴스 시작 시 EFA 활성화, 그리고 OS 레벨의 EFA 소프트웨어 설치가 주요 단계이며, HPC/ML 워크로드 가속을 위해 MPI(Message Passing Interface) 등을 사용하는 애플리케이션에서 Low-latency 통신을 가능하게 합니다. 특히 네트워크 설정에서 EFA를 활성화하고, 모든 EFA가 통신할 수 있도록 보안 그룹을 열어주는 것이 중요합니다. 

### EFA 설정 단계 ###
* EFA 지원 인스턴스 선택: C5n, P4d 등과 같이 'n'이 포함되거나 대형 인스턴스 유형 중 EFA를 지원하는 인스턴스를 선택합니다.
* 클러스터 배치 그룹 생성: 인스턴스들이 물리적으로 가깝게 배치되도록 클러스터 배치 그룹을 생성합니다.
* 서브넷 설정: EFA를 활성화할 서브넷을 선택하고, 해당 서브넷에서 EFA를 활성화해야 합니다.
* 보안 그룹 설정: 모든 EFA 인스턴스가 서로 통신할 수 있도록 모든 포트와 프로토콜에 대한 인바운드/아웃바운드 트래픽을 허용하는 EFA 전용 보안 그룹을 생성합니다.

### AMI 생성 및 인스턴스 시작: ###
* EFA 지원 인스턴스용 AMI를 생성합니다. (Amazon Linux 2 AMI에는 EFA 소프트웨어가 기본 포함되어 있을 수 있습니다).
* 인스턴스 시작 시, 네트워크 설정에서 해당 서브넷을 선택하고, EFA 활성화 옵션을 켜줍니다.
* 생성한 클러스터 배치 그룹과 EFA 보안 그룹을 연결합니다.
* OS 내 EFA 활성화 (필요시): 인스턴스 부팅 후, fi_info -p efa 명령어로 EFA가 활성화되었는지 확인하고, 필요하다면 AWS 공식 가이드에 따라 EFA 소프트웨어를 설치하고 활성화합니다. 
