이 코드는 S3에 저장된 대용량 텍스트(JSON 또는 Parquet)를 불러와 분산 환경에서 전처리하는 샘플이다. 
vs-code 터미널에서 아래 내용을 실행한다.

### 1. S3 버킷 생성 ###
```
export AWS_REGION=$(aws ec2 describe-availability-zones --query 'AvailabilityZones[0].RegionName' --output text)
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export CLUSTER_NAME="ray-on-aws"

# 버킷 이름을 변수로 지정 (예: ray-data-benchmark-12345)
BUCKET_NAME="ray-on-aws-$(date +%s)"

# 버킷 생성
aws s3api create-bucket --bucket ${BUCKET_NAME} --region ${AWS_REGION} \
  --create-bucket-configuration LocationConstraint=${AWS_REGION}
```
```
BUCKET_NAME=$(aws s3 ls | grep ${BUCKET_NAME} | cut -d' ' -f3) 
echo ${BUCKET_NAME}
```
[결과]
```
ray-on-aws-1768888873
```

### 2. 샘플 데이터 생성 ###
```
cd ~
mkdir -p c9
cd c9

cat <<EOF > generate-data.py
import ray
import pandas as pd
import numpy as np
import string

ray.init(address="auto")

# 1. 샘플 텍스트 생성 함수 (약 1KB 크기의 문장 생성)
def generate_fake_data(batch_info):
    # 각 배치당 10,000행 생성 (1행당 약 1KB -> 배치당 약 10MB)
    num_rows = 10000
    data = {
        "id": np.random.randint(0, 10**9, size=num_rows),
        "text": [''.join(np.random.choice(list(string.ascii_letters + " "), 1000)) 
                 for _ in range(num_rows)],
        "label": np.random.choice(["news", "blog", "forum", "wiki"], size=num_rows)
    }
    return pd.DataFrame(data)

# 2. 100GB 목표 설정
# 배치당 10MB이므로, 10,000개의 배치를 만들면 100GB가 됩니다.
num_batches = 10000 

# 3. Ray Data로 생성 및 저장
# range().map_batches를 사용하면 실제 데이터를 메모리에 다 올리지 않고 
# 생성 즉시 S3로 스트리밍 저장합니다.
ds = ray.data.range(num_batches) \
    .map_batches(generate_fake_data, batch_size=1)

# 4. S3 업로드 실행 (병렬 쓰기)
# parallelism은 100GB라는 전체 데이터를 최종적으로 몇 개의 조각(파일)으로 나누어 저장할 것인가를 결정하는 설정
# parallelism과 워커 노드의 총 코어 수가 같은 값이면 최대 속도로 업로드할 수 있다.
ds.repartition(100).write_parquet("s3://${BUCKET_NAME}/raw-100gb-data/")

print("100GB 샘플 데이터 생성 및 S3 업로드 완료! - ${BUCKET_NAME}")
EOF
```
```
ray job submit --address http://localhost:8265 --working-dir . -- python generate-data.py
```


### 3. 데이터 전처리 ###
```
cat <<EOF > preprocessing.py
import ray
import pandas as pd
import re

# 1. 클러스터 접속 (배스천에서 실행 시 주소 명시)
ray.init(address="auto")

def preprocess_text(batch: pd.DataFrame) -> pd.DataFrame:
    """
    각 워커 노드에서 실행될 배치 단위 전처리 함수

    정규식 re.sub(r"[^가-힣a-zA-Z0-9\s]", "", text)
       [^ ... ] (부정 조건): 괄호 안에 있는 문자가 아닌(NOT) 것들을 찾으라는 의미.
       가-힣: 한글.
       a-zA-Z: 모든 영문자(대소문자).
       0-9: 모든 숫자. 
       \s: 공백(Space, Tab, Newline 등).

    예시 결과
       입력: "Hello, World!!! 2026 @Ray_Data#"
       출력: "Hello World 2026 RayData"
       쉼표(,), 느낌표(!), 골뱅이(@), 샵(#), 언더바(_) 등이 모두 삭제.
    """
    def clean(text):
        # 특수문자 제거 및 소문자화 (LLM 학습 전처리 예시)
        text = re.sub(r"[^가-힣a-zA-Z0-9\s]", "", text)
        return text.lower()

    # 'text' 컬럼 전처리
    batch["cleaned_text"] = batch["text"].apply(clean)
    return batch

# 2. 대규모 데이터 로드 (S3에서 직접 스트리밍)
# 100GB 데이터를 로드해도 메모리에 바로 올리지 않고 메타데이터만 가져옵니다.
ds = ray.data.read_parquet("s3://${BUCKET_NAME}/large-dataset/")

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
processed_ds.write_parquet("s3://${BUCKET_NAME}/preprocessed-output/")

print("전처리 완료 및 S3 저장 성공! - s3://${BUCKET_NAME}/preprocessed-output/")
EOF
```
* Resource Scheduling:  
Intel/Graviton 혼합 클러스터라면 map_batches(..., resources={"Intel": 1}) 처럼 특정 노드에서만 전처리를 수행하게 강제하여 성능 차이를 측정할 수 있다.
* Parallelism 최적화:   
예를들어 100GB 데이터라면 파일 개수가 수천 개일 수 있는데, read_parquet(..., parallelism=200) 설정을 통해 초기에 데이터를 몇 개의 조각으로 나눌지 결정하는 것이 성능의 좌우한다.
* Object Spilling:  
메모리가 부족하면 Ray는 자동으로 로컬 디스크나 S3로 데이터를 흘려보낸다(Spilling). 대규모 벤치마크 시 이 현상이 발생하는지 대시보드에서 관찰하는 것도 좋다.
* 클러스터의 병렬성 (Parallelism)  
Ray는 데이터를 읽을 때 클러스터의 전체 CPU 코어 수의 약 2~3배를 기본 파티션 수로 잡는 경향이 있고, 연산 후 이 파티션 갯수만큼 결과 파일이 생긴다.
* 연산 결과 파일의 개수는 repartiton 또는 min_row_per_file 함수로 조정할 수 있다.
    * 파일 개수 지정 (repartition):
        ```
        processed_ds.repartition(100).write_parquet("s3://bucket/path/")
        ```
    * 파일 크기 기반 조절 (min_rows_per_file):
        ```
        processed_ds.write_parquet("s3://bucket/path/", min_rows_per_file=10000
        ```

데이터 전처리 병렬 프로세싱을 실행한다.
```
ray job submit --address http://localhost:8265 --working-dir . -- python preprocessing.py
```



