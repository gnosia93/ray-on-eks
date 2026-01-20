이 코드는 S3에 저장된 대용량 텍스트(JSON 또는 Parquet)를 불러와 분산 환경에서 전처리하는 샘플이다.

```
import ray
import pandas as pd
import re

# 1. 클러스터 접속 (배스천에서 실행 시 주소 명시)
ray.init(address="auto")

def preprocess_text(batch: pd.DataFrame) -> pd.DataFrame:
    """
    각 워커 노드에서 실행될 배치 단위 전처리 함수
    """
    def clean(text):
        # 특수문자 제거 및 소문자화 (LLM 학습 전처리 예시)
        text = re.sub(r"[^a-zA-Z0-9\s]", "", text)
        return text.lower()

    # 'text' 컬럼 전처리
    batch["cleaned_text"] = batch["text"].apply(clean)
    return batch

# 2. 대규모 데이터 로드 (S3에서 직접 스트리밍)
# 100GB 데이터를 로드해도 메모리에 바로 올리지 않고 메타데이터만 가져옵니다.
ds = ray.data.read_parquet("s3://your-bucket/large-dataset/")

# 3. 분산 처리 실행
# - compute=ray.data.ActorPoolStrategy(min_size=10, max_size=30): 
#   배치 처리를 위해 액터 풀을 사용하여 초기화 비용 절감
processed_ds = ds.map_batches(
    preprocess_text,
    batch_size=1024,           # 한 번에 처리할 행(row) 수
    concurrency=30,            # 병렬성 (워커 노드 수에 맞춰 조절)
)

# 4. 결과 저장 (Partitioning)
# 결과를 다시 S3에 저장하며, 자동으로 여러 파일로 분할 저장됩니다.
processed_ds.write_parquet("s3://your-bucket/preprocessed-output/")

print("전처리 완료 및 S3 저장 성공!")
```
