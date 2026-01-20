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
