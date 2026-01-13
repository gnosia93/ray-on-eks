# ray-on-eks


LLM 훈련/파인튜닝 시 Ray가 기존 Kubeflow 방식보다 유리한 포인트 3가지는 이렇습니다:

#### 1. 7B를 넘어 70B 모델로 갈 때 (분산 전략의 유연성) ####
일반적인 PyTorchJob은 단순 데이터 병렬 처리(DDP)에는 강하지만, 모델 병렬화나 복잡한 샤딩 전략을 짤 때는 코드가 지저분해집니다.
Ray는 DeepSpeed나 PyTorch FSDP 설정을 파이썬 코드 몇 줄로 추상화해줍니다. 특히 모델이 커질수록 발생하는 Checkpoints 저장/로드 병목을 Ray Data를 통해 분산 처리하여 시간을 크게 단축합니다.

#### 2. 가변적인 GPU 자원 관리 (Spot Instance 활용) ####
LLM 훈련은 비쌉니다. Ray는 Preemptible/Spot Instance 대응 능력이 뛰어납니다.
EKS에서 Spot 인스턴스가 회수되어 노드가 죽어도, Ray의 Object Store와 상태 관리 기능을 통해 훈련을 자동으로 재개(Fault Tolerance)하는 메커니즘이 Kubeflow보다 훨씬 유연합니다.

#### 3. 데이터 로딩 병목 해결 ####
LLM은 읽어와야 할 텍스트 데이터셋 자체가 거대합니다. GPU는 노는데 CPU에서 전처리하느라 시간이 다 가는 경우가 많죠.
Ray Data를 쓰면 스트리밍 방식으로 데이터를 GPU에 밀어 넣어주기 때문에, 전체 학습 데이터를 다 로드할 필요 없이 훈련과 동시에 전처리를 병렬화할 수 있습니다.


기존 Kubeflow 기반 방식에서 Pre-training 시 겪으셨을 페인 포인트를 Ray가 해결하는 방식입니다.
#### 1. 전처리와 학습의 "심리스한" 병렬화 (Ray Data) ####
* 기존: PyTorch DataLoader는 단일 노드의 CPU/메모리에 의존합니다. 멀티 노드 학습 시 데이터 셔플링이나 로딩 병목이 생겨 GPU 가동률(Util)이 떨어지기 쉽습니다.
* Ray: Ray Data를 사용하면 수 TB의 데이터를 클러스터 전체 CPU 노드에 분산시켜 실시간으로 전처리(Tokenizing, Global Shuffling)하고, 학습 노드의 GPU로 스트리밍합니다. GPU가 데이터 로딩을 기다리는 시간을 0에 수렴하게 만듭니다.

#### 2. 하드웨어 장애 대응 (Fault Tolerance) ####
* 기존: 수백 개의 GPU로 몇 주간 학습하는 Pre-training 중 노드 하나만 죽어도 전체 PyTorchJob이 터지거나 체크포인트부터 수동 재시작해야 합니다.
* Ray: Ray Train은 노드 장애 시 자동으로 가용 노드를 찾아 학습 프로세스를 재구성하고, 가장 최근의 분산 체크포인트에서 복구하는 기능이 더 세밀하게 설계되어 있습니다.

#### 3. 고성능 분산 전략의 통합 ####
* DeepSpeed/FSDP: Pre-training 필수인 Zero-3 전략이나 텐서 병렬화(TP), 파이프라인 병렬화(PP)를 적용할 때, Ray의 DeepSpeed 통합 가이드를 따르면 복잡한 rank 설정이나 네트워킹 구성을 Ray가 내부적으로 알아서 핸들링합니다.

#### 🛠️ 추천 드리는 이행 전략 (EKS 환경) ####
* KubeRay Operator 설치: 기존 EKS 클러스터에 KubeRay를 설치하여 Ray 클러스터를 하나의 커스텀 리소스로 관리하세요.
* 데이터 소스 연결: S3 등에 저장된 대규모 코퍼스를 Ray Data로 읽어 들이는 로직을 먼저 테스트해보세요.
* Hugging Face Accelerate 연동: 이미 HF 기반이시라면 accelerate 설정을 Ray Train이 감싸는 형태로 시작하면 코드 수정을 최소화할 수 있습니다.
