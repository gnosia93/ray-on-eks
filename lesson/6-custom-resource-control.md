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
