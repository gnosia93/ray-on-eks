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
