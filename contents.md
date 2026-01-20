## CPU 아키텍처 특징 ##

- Intel (c7i): 가장 표준적입니다. 특히 AMX(Advanced Matrix Extensions) 가속기 덕분에 텍스트 임베딩이나 행렬 연산에서 압도적이다. 
- AMD (c7a): 순수 연산 속도(Raw Clock)와 가성비가 좋습니다. 단순 텍스트 정규화나 정규식 처리 위주라면 Intel보다 저렴하면서 성능은 비슷하게 나온다.
- Graviton (AWS 자체 칩 - arm64)
  - 동일 성능 대비 비용이 최대 20% 저렴하다.
  - arm64 전용 바이너리를 내려받아야 한다. 일부 라이브러리의 경우 환환성 문제를 일으킬 수 있다.

## 인스턴스별 특징 ##
성능 극대화를 위한 최신 EC2 활용법은 단순히 '비싼 서버'를 쓰는 게 아니라, Ray 워크로드의 특성(Compute vs I/O vs Memory)에 맞춰 AWS의 최신 아키텍처를 전략적으로 매칭하는 것을 의미합니다. 워크숍에서는 다음 3가지 포인트가 핵심입니다.
#### 1. 최신 인텔(m7i) vs 가성비 그라비톤(r8g) 선택 ####
* m7i (Intel Sapphire Rapids): 최신 AMX(Advanced Matrix Extensions) 가속기를 내장하여, 전처리 중 벡터 연산이나 딥러닝 추론이 포함될 때 압도적인 성능을 냅니다.
* r8g (AWS Graviton4): 전력 대비 성능이 우수하며, 동일 비용 대비 더 많은 메모리 대역폭을 제공합니다. Ray Data의 대규모 셔플(Shuffle) 작업처럼 데이터 이동이 많을 때 유리합니다.
 
#### 2. 고속 네트워크(EFA)와 데이터 대역폭 최적화 ####
* Ray는 노드 간 데이터 이동이 잦습니다. 인스턴스 타입 뒤에 'n'이 붙은 모델(예: r7i-n)을 사용하면 최대 200Gbps의 네트워크 대역폭을 제공하여 S3에서 데이터를 긁어오거나 노드 간 객체를 전송하는 병목 현상을 해결합니다.
* AWS EFA(Elastic Fabric Adapter)를 활성화하면 분산 학습 시 지연 시간을 획기적으로 줄일 수 있습니다.

#### 3. 고성능 로컬 NVMe 스토리지 활용 (d 타입) ####
* 오브젝트 스필링 대책으로 EBS 대신 'd'가 붙은 인스턴스(예: m7i.2xlarge -> m7d.2xlarge)를 활용합니다.
* 내장된 로컬 NVMe SSD는 EBS보다 수십 배 빠른 I/O 속도를 제공하므로, 메모리가 부족해 디스크로 데이터를 밀어낼 때도 성능 저하를 최소화합니다.

## 교육 목차 ##
* 1교시 - AWS EC2 포트폴리오 소개 / CPU 별 특징 소개
* 2교시 - ray data / 데이터 전처리 이론 교육
    * 머신 러닝 데이터 전처리
    * LLM text 데이터 전처리
    * spark ETL 대체   
* 3교시 / 4교시 - ray 실습
* engage strategy
   * 신규 니즈 발굴
   * 단독 워크샵 형태
   * ML 워크샵시 컨텐츠 소개 

## PPT Slide ##
```
Slide 1: 타이틀
제목: Ray를 이용한 대규모 데이터 전처리 및 인프라 오토스케일링 실무
부제: 160 vCPU를 넘어 480 vCPU까지, 클라우드 자원을 쥐어짜는 법
강사명: [강사님 성함]
Slide 2: 왜 다시 '분산 컴퓨팅'인가?
문제 제기: 왜 내 Pandas 코드는 코어 1개만 쓰고 나머지는 놀고 있는가? (GIL의 한계)
해결책: Python 코드를 수정 없이 수백 대의 서버로 확장하는 Ray의 등장 배경.
핵심 키워드: 병렬화(Parallelization), 분산 객체 저장소(Plasma), 확장성(Scalability).
Slide 3: 워크샵 인프라 아키텍처 (Deep Dive)
도식화: [로컬 PC] → [Bastion Host] → [Private Subnet: Ray Head/Worker]
보안의 정석:
EICE(EC2 Instance Connect Endpoint): PEM 키 없이 안전하게 접속.
IAM Instance Profile: 서버 자체가 AWS API를 호출하는 권한 체계.
참고: AWS EC2 Instance Connect 가이드
Slide 4: Ray의 심장, Autoscaler의 동작 원리
핵심 질문: Ray는 어떻게 서버를 더 늘려야 할지 알까?
동작 매커니즘:
사용자가 Task 요청 (예: num_cpus=1 작업 400개)
현재 클러스터 자원(160코어) 확인 → 320코어 부족!
Pending Task 발생 → Autoscaler가 "주문서" 확인.
AWS API 호출하여 부족한 만큼 EC2 즉시 소환.
참고: Ray Autoscaler 문서
Slide 5: 자원 제어의 마법 - Custom Resources
설정 코드 분석: resources: {"CPU": 16, "intel": 1}
의미:
CPU: 실제 물리적인 의자 개수.
intel: 이 노드가 Intel 칩셋임을 나타내는 '출입증'.
실습 예고: @ray.remote(resources={"intel": 1})을 통해 특정 아키텍처(c7i vs c7g)로 작업을 정밀 타겟팅하는 법.
Slide 6: 비용 최적화 - 스팟 인스턴스 전략
가성비 끝판왕: 온디맨드 대비 최대 90% 저렴한 스팟 인스턴스 활용.
Fault Tolerance: 서버가 갑자기 꺼져도 Ray가 작업을 재스케줄링하여 완수하는 내결함성 설명.
명분: "우리는 커피 한 잔 값으로 슈퍼컴퓨터를 빌려 씁니다."
Slide 7: [실습] 160코어 폭격 및 30대 소환
미션: 480개 헤비 태스크를 던져 c7i.4xlarge 20대를 추가로 소환하라!
체크리스트:
watch -n 1 ray status 로그 모니터링.
AWS 콘솔에서 EC2 생성 확인.
Dashboard에서 480개 코어가 100% 점유되는 장관 관람.
```
Slide 8: Q&A 및 정리
요약: 인프라를 코드로 제어(IaC)하고, 자원을 동적으로 확장하는 데이터 엔지니어링의 정수.
다음 단계: 실제 데이터(Common Voice) 전처리 실무 적용.
