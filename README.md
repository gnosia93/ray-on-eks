* 1교시 - AWS EC2 포트폴리오 소개
* 2교시 - ray data
* 3교시 / 4교시 - 실습

# ray-on-aws (EC2 bimonthly workshop)

* [C1. VPC 생성](https://github.com/gnosia93/ray-on-aws/blob/main/lesson/1-create-vpc.md)

* [C2. ray 클러스터 생성](https://github.com/gnosia93/ray-on-aws/blob/main/lesson/2-create-cluster.md)



```
import ray

# 1. Intel 노드에서만 실행 (AMX 가속 등을 가정)
@ray.remote(resources={"intel": 1})
def run_on_intel():
    import platform
    return f"Intel 노드 실행 중: {platform.machine()}"

# 2. Graviton 노드에서 실행 (가성비 전처리)
@ray.remote(resources={"arm": 1})
def run_on_arm():
    import platform
    return f"Graviton 노드 실행 중: {platform.machine()}"

# 3. 결과 확인
print(ray.get([run_on_intel.remote(), run_on_arm.remote()]))

```



### 3단계: Upscaling 폭격 (EC2 30대 소환)
이제 160코어의 한계를 시험하며 EC2 자원을 최대치로 땡기는 핵심 실습입니다.
* 스트레스 테스트 스크립트 실행
```
# 배스천에서 실행 (stress.py)
import ray, time
ray.init(address="auto")

@ray.remote(num_cpus=1)
def heavy_task():
    time.sleep(300) # 5분간 코어 점유
    return True

# 480개(30대 분량)의 작업을 한꺼번에 투척!
ray.get([heavy_task.remote() for _ in range(480)])
```
현상 관찰:
* Ray Status: Pending: 320 Tasks 발생 -> Autoscaler가 Launching 20 Nodes 시작.
* AWS Console: EC2 인스턴스 페이지에 c7i.4xlarge 20대가 한꺼번에 생성되며 Pending 상태가 되는 장관을 확인합니다

### 4단계: 오토 스케일링 설정 ###
* 세부 조절 파라미터 (YAML 설정)
  * 단순히 쌓인다고 바로 뜨는 게 아니라, YAML에 설정한 값에 따라 속도가 조절됩니다.
  * target_utilization_fraction (기본값 0.8):
  * 클러스터 전체 자원 사용률이 이 수치를 넘으면 미리 여유분을 준비하려 합니다. 160코어 중 128코어(80%)만 써도 "슬슬 더 뽑아야겠는데?"라고 준비하는 기준입니다.
  * upscaling_speed:
    한 번에 얼마나 공격적으로 늘릴지 결정합니다. 이 값이 높을수록(예: 10) 한 번에 많은 EC2를 동시에 요청합니다.

축소(Downscaling)의 기준
* 늘리는 것만큼 줄이는 기준도 중요합니다.
* Idle Timeout: 특정 노드에 할당된 작업이 하나도 없고, YAML에 설정한 idle_timeout_minutes(예: 5분) 동안 아무 일도 하지 않으면 해당 노드를 삭제 대상으로 간주합니다.
* Min Workers: 아무리 일이 없어도 min_workers 설정값 이하로는 줄이지 않습니다.


---
* static
* 오토 스케일링 설정
* spot 사용하기
* on-demand / spot 믹스하기
* ray 는 spot 회쉬되더라도 교체해서 동작한다. 어떻게?
  

```
Ray 인프라 & 오토스케일링 딥다이브 (총 8시간)
세션	실습 항목 (Hands-on)	내용 및 딥다이브 포인트	소요 시간
Session 1	인프라 프로비저닝	VPC, CloudFormation Bastion, Terraform IAM Role 생성 및 EICE 보안 연동 실습	90분
Session 2	클러스터 런칭 & 보안	배스천에서 Ray Cluster YAML 설정 후 PEM 키 없이 IAM Role 기반으로 10대(160코어) 기동	60분
Session 3	Upscaling 폭격	480개 태스크 투하로 EC2 30대 소환. monitor.out 로그 분석을 통한 Pending Task 기반 확장 원리 파헤치기	90분
Session 4	커스텀 자원 정밀 제어	Custom_Resource를 활용해 특정 인스턴스 타입을 골라 띄우는 Resource-specific Scaling 구현	60분
Session 5	Fault Tolerance 체험	가동 중인 워커 노드를 강제 종료(Terminate)하고, Ray가 작업 손실 없이 새 EC2로 복구하는 과정 실측	60분
Session 6	비용 최적화 챌린지	온디맨드 vs 스팟 인스턴스 믹스 전략 적용. idle_timeout 설정을 통한 자동 Downscaling 및 비용 절감액 계산	60분
Session 7	대규모 전처리 벤치마크	100GB급 텍스트 데이터를 30대 클러스터에서 전처리하며 Parallelism & Batch Size 최적화 값 찾기	60분
Wrap-up	인프라 Clean-up	ray down 및 Terraform/CfN 리소스 일괄 삭제로 클라우드 자원 관리 마무리	30분
```

* 오토스케일링 로그: /tmp/ray/session_latest/logs/monitor.out
* 노드 초기화 로그: /tmp/ray/session_latest/logs/raylet.out


## 아키텍처별 가이드 ##
네, 엄청난 차이가 있습니다. Ray는 분산 컴퓨팅 프레임워크이기 때문에 하드웨어 아키텍처의 특성을 아주 정직하게 타는 편입니다. 특히 님께서 선택하신 c7i(Intel)와 비교했을 때 아키텍처별로 다음과 같은 실습 포인트들이 생깁니다.
* 1. Intel (c7i) vs AMD (c7a)
- Intel (c7i): 가장 표준적입니다. 특히 AMX(Advanced Matrix Extensions) 가속기 덕분에 텍스트 임베딩이나 행렬 연산에서 압도적입니다. Intel Ray 가속 가이드를 참고하면 최적화 라이브러리 설정법이 나옵니다.
- AMD (c7a): 순수 연산 속도(Raw Clock)와 가성비가 좋습니다. 단순 텍스트 정규화나 정규식 처리 위주라면 Intel보다 저렴하면서 성능은 비슷하게 나옵니다.

* 2. Graviton (AWS 자체 칩 - arm64)
- 성능/비용: 가성비 끝판왕입니다. 동일 성능 대비 비용이 약 20% 저렴합니다.
- 주의사항: pip install 시 arm64 전용 바이너리를 내려받아야 하므로 AMI(Amazon Machine Image) 설정이 달라집니다.
- 교육 포인트: "비용 절감을 위해 아키텍처를 arm64로 전환하는 실습"을 넣으면 아주 고급 컨텐츠가 됩니다. AWS Graviton 가이드를 참고하세요.


#### [Section 8] 아키텍처 벤치마킹 실습 (추가 제안) ###
* 교육 시간을 조금 더 쓰고 싶다면, 10대는 c7i(Intel), 10대는 c7a(AMD)로 띄워서 동일한 텍스트 전처리 속도를 비교하게 해보세요.
* 관전 포인트: "왜 특정 작업에선 Intel이 빠르고, 다른 작업에선 AMD가 가성비가 좋을까?"
* EC2 활용 극대화: 인스턴스 타입을 섞어 쓰기 때문에 Mixed Instances Policy를 완벽하게 실습하게 됩니다

```
키텍처	추천 인스턴스	핵심 특징 (Deep Dive)	전처리 적합도
Intel	c7i.4xlarge	AMX 가속기 탑재. 복잡한 수치 연산 및 LLM 텍스트 임베딩 전처리에 최강. 가장 호환성이 높음.	⭐⭐⭐⭐⭐
AMD	c7a.4xlarge	최신 젠4 기반. 단순 정규식(Regex) 처리나 대량의 문자열 파싱에서 순수 연산 속도가 뛰어남.	⭐⭐⭐⭐
Graviton	c7g.4xlarge	AWS 전용 ARM 칩. 가격 대비 성능비(Price/Perf) 최고. 인스턴스 30대를 돌려도 비용 부담이 가장 적음.	⭐⭐⭐⭐⭐ (가성비)
```

```
실습 시나리오 적용 팁 (강사 가이드)
1. "Intel vs AMD" 속도 경쟁 (30분)
실습: 5대는 c7i, 5대는 c7a로 띄워 동일한 Common Voice 텍스트 전처리를 돌립니다.
관전 포인트: 특정 정규식 처리 속도는 AMD가 빠를 수 있으나, 라이브러리 최적화(MKL 등)가 들어간 연산은 Intel이 앞서는 모습을 Ray Dashboard로 비교합니다.
2. "Graviton 전환" 비용 절감 세션
내용: "비용이 20% 줄어든다면 성능을 조금 포기하시겠습니까?"라는 질문을 던집니다.
실습: AWS Graviton 가이드를 참고해 arm64 전용 AMI로 클러스터를 다시 띄우게 합니다. (이 과정에서 아키텍처 호환성 트러블슈팅 경험치 급상승)
3. "Mixed Instance Policy"의 정당화
논리: "재고가 부족한 스팟 인스턴스 환경에서는 Intel, AMD를 가리지 않고 닥치는 대로(?) 가져와서 클러스터를 유지하는 능력이 실력입니다."
방법: YAML 설정에 InstanceType: "c7i.4xlarge, c7a.4xlarge, c6i.4xlarge"를 모두 넣어 가용성을 극대화합니다.
```





## 데이터 ##
```
aws s3 ls s3://ray-example-data/ --no-sign-request
```
```
aws s3 ls s3://ray-example-data/common_voice_17/parquet/
2025-09-27 04:00:11  494117001 0.parquet
2025-09-27 04:00:11  494284009 1.parquet
2025-09-27 04:00:11  494615004 2.parquet
2025-09-27 04:00:11  492952215 3.parquet
2025-09-27 04:00:11  493082766 4.parquet
2025-09-27 04:00:22  497584052 5.parquet
2025-09-27 04:00:23  493463495 6.parquet
2025-09-27 04:00:23  494387266 7.parquet
2025-09-27 04:00:23  494530333 8.parquet
2025-09-27 04:00:23  476320458 9.parquet
```

## 전처리 ##
* 텍스트 전처리
* 이미지 전처리
* 보이스 전처리

## 레퍼런스 ##
* https://docs.ray.io/en/latest/ray-overview/installation.html
* https://github.com/dmatrix/ray-core-tutorial/blob/ad5f1fa700d87a9af1e21027f06f02cfdcc937f3//ex_07_ray_data.ipynb
* https://github.com/aws-samples/aws-samples-for-ray/tree/main/ec2
