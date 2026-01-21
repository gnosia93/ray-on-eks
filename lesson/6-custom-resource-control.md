### 1. 코드에서 Custom Resource 요청 ###
ray.remote나 map_batches 호출 시, resources 인자에 YAML에서 정의한 키 값을 명시한다.

```
# 1. 특정 Task를 Graviton 노드에만 할당
@ray.remote(resources={"Graviton": 1})
def heavy_processing(batch):
    return [re.sub(r"[^가-힣]", "", str(t)) for t in batch]

# 2. Ray Data 사용 시 (MapBatches)
processed_ds = ds.map_batches(
    TextPreprocessor,
    # 이 설정이 있으면 Ray Autoscaler는 'Graviton' 리소스가 있는 노드만 추가로 띄웁니다.
    num_cpus=1,
    resources={"Graviton": 1} 
)
```

### 2. Autoscaler 작동 원리 ###
* 리소스 감시: Ray가 작업을 스케줄링하려고 할 때, 클러스터에 {"Graviton": 1} 여유분이 있는지 확인
* 노드 요청: 여유가 없다면 Autoscaler는 available_node_types 중 {"Graviton": 1}을 리소스로 가진 노드(여기서는 arm_worker_node)를 생성하도록 AWS에 요청.
* 격리 실행: Intel 리소스만 필요한 작업은 x86_worker_node로, Graviton 리소스가 명시된 작업은 arm_worker_node로 분리되어 실행.

## 오브젝트 스필링(Object Spilling) ##
오브젝트 스필링(Object Spilling)은 Ray의 공유 메모리가 꽉 찼을 때, 데이터를 삭제하는 대신 로컬 디스크(SSD 등)로 잠시 옮겨두는 기능이다. 수십만 장의 이미지를 처리하다 보면 메모리 부족(OOM)으로 작업이 터지기 쉬운데, 이를 방지하는 '안전장치' 이다.
설정 방법은 크게 두 가지인데,

### 1. ray.init() 코드 내 설정 ###
```
import ray

ray.init(
    _system_config={
        # 1. 스필링을 저장할 로컬 디스크 경로
        "object_spilling_config": {
          "type": "filesystem",
          "params": {"directory_path": "/tmp/ray/spill"},
        },
        # 2. 공유 메모리가 몇 % 찼을 때 디스크로 넘길지 설정 (예: 70%)
        "object_store_full_delay_ms": 100,
    }
)
```

### 2. 클러스터 구동 시 설정 (YAML 파일) ###
```
# cluster.yaml 예시
...

worker_nodes:
    # ... 인스턴스 설정
    ray_start_params:
        system-config: '{"object_spilling_config": {"type": "filesystem", "params": {"directory_path": "/tmp/ray/spill"}}}'
```
