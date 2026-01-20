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
