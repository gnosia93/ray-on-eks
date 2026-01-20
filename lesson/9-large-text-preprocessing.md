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
    # 각 배치당 10,000행 생성 (1행당 약 1KB -> 배치당 약 1MB)
    num_rows = 1000
    data = {
        "id": np.random.randint(0, 10**9, size=num_rows),
        "text": [''.join(np.random.choice(list(string.ascii_letters + " "), 1000)) 
                 for _ in range(num_rows)],
        "label": np.random.choice(["news", "blog", "forum", "wiki"], size=num_rows)
    }
    return pd.DataFrame(data)

# 2. 10GB 목표 설정
# 배치당 1MB이므로, 10,000개의 배치를 만들면 100GB가 됩니다.
num_batches = 10000 

# 3. Ray Data로 생성 및 저장
# range().map_batches를 사용하면 실제 데이터를 메모리에 다 올리지 않고 
# 생성 즉시 S3로 스트리밍 저장합니다.
ds = ray.data.range(num_batches) \
    .map_batches(generate_fake_data, batch_size=1)

# 4. S3 업로드 실행 (병렬 쓰기)
# parallelism은 10GB라는 전체 데이터를 최종적으로 몇 개의 조각(파일)으로 나누어 저장할 것인가를 결정하는 설정
# parallelism과 워커 노드의 총 코어 수가 같은 값이면 최대 속도로 업로드할 수 있다.
ds.repartition(100).write_parquet("s3://${BUCKET_NAME}/raw-100gb-data/")

print("100GB 샘플 데이터 생성 및 S3 업로드 완료! - ${BUCKET_NAME}")
EOF
```
```
ray job submit --address http://localhost:8265 --working-dir . -- python generate-data.py
```
[결과]
```
Job submission server address: http://localhost:8265
2026-01-20 08:56:30,411 INFO dashboard_sdk.py:338 -- Uploading package gcs://_ray_pkg_20e56bc7ef94aec4.zip.
2026-01-20 08:56:30,412 INFO packaging.py:588 -- Creating a file package for local module '.'.

-------------------------------------------------------
Job 'raysubmit_eMq6km2Wugix36gk' submitted successfully
-------------------------------------------------------

Next steps
  Query the logs of the job:
    ray job logs raysubmit_eMq6km2Wugix36gk
  Query the status of the job:
    ray job status raysubmit_eMq6km2Wugix36gk
  Request the job to be stopped:
    ray job stop raysubmit_eMq6km2Wugix36gk

Tailing logs until the job exits (disable with --no-wait):
2026-01-20 08:56:30,421 INFO job_manager.py:568 -- Runtime env is setting up.
2026-01-20 08:56:31,366 INFO worker.py:1691 -- Using address 10.0.2.183:6379 set in the environment variable RAY_ADDRESS
2026-01-20 08:56:31,369 INFO worker.py:1832 -- Connecting to existing Ray cluster at address: 10.0.2.183:6379...
2026-01-20 08:56:31,378 INFO worker.py:2003 -- Connected to Ray cluster. View the dashboard at http://10.0.2.183:8265 
2026-01-20 08:56:32,137 INFO streaming_executor.py:85 -- A new progress UI is available. To enable, set `ray.data.DataContext.get_current().enable_rich_progress_bars = True`.
2026-01-20 08:56:32,137 INFO logging.py:397 -- Registered dataset logger for dataset dataset_14_0
2026-01-20 08:56:32,147 INFO streaming_executor.py:170 -- Starting execution of Dataset dataset_14_0. Full logs are in /tmp/ray/session_2026-01-20_08-16-22_198865_26744/logs/ray-data
2026-01-20 08:56:32,147 INFO streaming_executor.py:171 -- Execution plan of Dataset dataset_14_0: InputDataBuffer[Input] -> TaskPoolMapOperator[ReadRange] -> TaskPoolMapOperator[MapBatches(generate_fake_data)] -> AllToAllOperator[Repartition] -> TaskPoolMapOperator[Write]
[dataset]: Run `pip install tqdm` to enable progress reporting.
2026-01-20 08:56:32,176 WARNING resource_manager.py:134 -- ⚠️  Ray's object store is configured to use only 40.9% of available memory (40.7GiB out of 99.4GiB total). For optimal Ray Data performance, we recommend setting the object store to at least 50% of available memory. You can do this by setting the 'object_store_memory' parameter when calling ray.init() or by setting the RAY_DEFAULT_OBJECT_STORE_MEMORY_PROPORTION environment variable.
(autoscaler +55s) Tip: use `ray status` to view detailed cluster status. To disable these messages, set RAY_SCHEDULER_EVENTS=0.
(autoscaler +55s) Adding 1 node(s) of type x86_worker_node.
(autoscaler +55s) Resized to 32 CPUs.
(autoscaler +2m27s) Adding 1 node(s) of type x86_worker_node.
(autoscaler +2m27s) Resized to 40 CPUs.
(autoscaler +3m44s) Adding 1 node(s) of type x86_worker_node.
(autoscaler +3m44s) Adding 1 node(s) of type arm_worker_node.
(autoscaler +3m44s) Resized to 56 CPUs.
2026-01-20 09:00:19,947 INFO streaming_executor.py:298 -- ✔️  Dataset dataset_14_0 execution finished in 227.80 seconds
2026-01-20 09:00:19,994 INFO dataset.py:5106 -- Data sink Parquet finished. 10000000 rows and 10.5GB data written.
100GB 샘플 데이터 생성 및 S3 업로드 완료! - ray-on-aws-1768897738

------------------------------------------
Job 'raysubmit_eMq6km2Wugix36gk' succeeded
------------------------------------------
```
파케이 형식으로 10기가 raw 데이터 파일들이 생성되었다.
![](https://github.com/gnosia93/ray-on-ec2/blob/main/lesson/images/raydata-s3.png)

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
# 10GB 데이터를 로드해도 메모리에 바로 올리지 않고 메타데이터만 가져옵니다.
ds = ray.data.read_parquet("s3://${BUCKET_NAME}/raw-100gb-data/")

# 3. 분산 처리 실행
# - compute=ray.data.ActorPoolStrategy(min_size=10, max_size=30): 
#   배치 처리를 위해 액터 풀을 사용하여 초기화 비용 절감
processed_ds = ds.map_batches(
    preprocess_text,
    batch_size=1024,            # 한 번에 처리할 행(row) 수
    batch_format="pandas",      # 포맷을 pandas 사용
    #concurrency=30,            # 병렬성 (워커 노드 수에 맞춰 조절)
    # 고정된 숫자(30) 대신 리소스를 명시하여 Ray가 가용 메모리에 맞춰 조절하게 함
    num_cpus=1, 
    # 혹은 액터 풀 전략 사용 시 메모리 부족을 방지하기 위해 크기 조절
    #compute=ray.data.ActorPoolStrategy(min_size=2, max_size=32) 
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
[결과]
```
Job submission server address: http://localhost:8265
2026-01-20 09:21:14,358 INFO dashboard_sdk.py:338 -- Uploading package gcs://_ray_pkg_d2d673d2a527ff86.zip.
2026-01-20 09:21:14,358 INFO packaging.py:588 -- Creating a file package for local module '.'.

-------------------------------------------------------
Job 'raysubmit_6VBJDxszQc7cfr7i' submitted successfully
-------------------------------------------------------

Next steps
  Query the logs of the job:
    ray job logs raysubmit_6VBJDxszQc7cfr7i
  Query the status of the job:
    ray job status raysubmit_6VBJDxszQc7cfr7i
  Request the job to be stopped:
    ray job stop raysubmit_6VBJDxszQc7cfr7i

Tailing logs until the job exits (disable with --no-wait):
2026-01-20 09:21:14,366 INFO job_manager.py:568 -- Runtime env is setting up.
2026-01-20 09:21:15,314 INFO worker.py:1691 -- Using address 10.0.2.183:6379 set in the environment variable RAY_ADDRESS
2026-01-20 09:21:15,316 INFO worker.py:1832 -- Connecting to existing Ray cluster at address: 10.0.2.183:6379...
2026-01-20 09:21:15,327 INFO worker.py:2003 -- Connected to Ray cluster. View the dashboard at http://10.0.2.183:8265 
[dataset]: Run `pip install tqdm` to enable progress reporting.
2026-01-20 09:21:17,587 INFO parquet_datasource.py:725 -- Estimated parquet encoding ratio is 1.007.
2026-01-20 09:21:17,587 INFO parquet_datasource.py:785 -- Estimated parquet reader batch size at 131458 rows
2026-01-20 09:21:17,687 INFO streaming_executor.py:85 -- A new progress UI is available. To enable, set `ray.data.DataContext.get_current().enable_rich_progress_bars = True`.
2026-01-20 09:21:17,687 INFO logging.py:397 -- Registered dataset logger for dataset dataset_24_0
2026-01-20 09:21:17,699 INFO streaming_executor.py:170 -- Starting execution of Dataset dataset_24_0. Full logs are in /tmp/ray/session_2026-01-20_08-16-22_198865_26744/logs/ray-data
2026-01-20 09:21:17,699 INFO streaming_executor.py:171 -- Execution plan of Dataset dataset_24_0: InputDataBuffer[Input] -> TaskPoolMapOperator[ReadParquet] -> TaskPoolMapOperator[MapBatches(preprocess_text)->Write]
2026-01-20 09:21:17,763 WARNING resource_manager.py:134 -- ⚠️  Ray's object store is configured to use only 40.8% of available memory (77.2GiB out of 189.0GiB total). For optimal Ray Data performance, we recommend setting the object store to at least 50% of available memory. You can do this by setting the 'object_store_memory' parameter when calling ray.init() or by setting the RAY_DEFAULT_OBJECT_STORE_MEMORY_PROPORTION environment variable.
(autoscaler +4s) Tip: use `ray status` to view detailed cluster status. To disable these messages, set RAY_SCHEDULER_EVENTS=0.
(autoscaler +4s) Adding 1 node(s) of type x86_worker_node.
(autoscaler +4s) Resized to 40 CPUs.
2026-01-20 09:21:37,254 INFO streaming_executor.py:298 -- ✔️  Dataset dataset_24_0 execution finished in 19.55 seconds
2026-01-20 09:21:37,340 INFO dataset.py:5106 -- Data sink Parquet finished. 10000000 rows and 20.3GB data written.
전처리 완료 및 S3 저장 성공! - s3://ray-on-aws-1768897738/preprocessed-output/

------------------------------------------
Job 'raysubmit_6VBJDxszQc7cfr7i' succeeded
------------------------------------------
```


