### 테스트 Job 실행 ###
vs-code 에서 새로운 터미널을 하나 더 열고 아래 명령어를 실행한다.   
```
cd ~
mkdir -p my-ray
cd my-ray

cat <<EOF > test-job.py
import ray
import platform

ray.init(runtime_env={"excludes": [".cache", ".local"]})

@ray.remote(resources={"Intel": 1})
def run_on_intel():
    return f"Intel 노드 실행 중: {platform.machine()}"

@ray.remote(resources={"Graviton": 1})
def run_on_arm():
    return f"Graviton 노드 실행 중: {platform.machine()}"

print("결과 확인:", ray.get([run_on_intel.remote(), run_on_arm.remote()]))
EOF

pwd
```

Job 을 제출한다. 동시에 ray 는 해당 디렉토리에 있는 파일을 전부 압축하여 클러스터로 업로드 한다. (업로드 제한 용량 100MB)
```
ray job submit --address http://localhost:8265 --working-dir . -- python test-job.py
````

[결과]
```
Job submission server address: http://localhost:8265
2026-01-20 08:26:36,534 INFO dashboard_sdk.py:338 -- Uploading package gcs://_ray_pkg_9b8be22046df1e43.zip.
2026-01-20 08:26:36,534 INFO packaging.py:588 -- Creating a file package for local module '.'.

-------------------------------------------------------
Job 'raysubmit_JN23e5DV1hcthHta' submitted successfully
-------------------------------------------------------

Next steps
  Query the logs of the job:
    ray job logs raysubmit_JN23e5DV1hcthHta
  Query the status of the job:
    ray job status raysubmit_JN23e5DV1hcthHta
  Request the job to be stopped:
    ray job stop raysubmit_JN23e5DV1hcthHta

Tailing logs until the job exits (disable with --no-wait):
2026-01-20 08:26:36,625 INFO job_manager.py:568 -- Runtime env is setting up.
2026-01-20 08:26:37,437 INFO worker.py:1691 -- Using address 10.0.2.183:6379 set in the environment variable RAY_ADDRESS
2026-01-20 08:26:37,440 INFO worker.py:1832 -- Connecting to existing Ray cluster at address: 10.0.2.183:6379...
2026-01-20 08:26:37,449 INFO worker.py:2003 -- Connected to Ray cluster. View the dashboard at http://10.0.2.183:8265 
결과 확인: ['Intel 노드 실행 중: x86_64', 'Graviton 노드 실행 중: aarch64']

------------------------------------------
Job 'raysubmit_JN23e5DV1hcthHta' succeeded
```

### 대시보드 확인 ###
실행된 Job 잡과 해당 Job 의 상세 정보를 확인한다.
![](https://github.com/gnosia93/ray-on-aws/blob/main/lesson/images/dashboard-job-list.png)

![](https://github.com/gnosia93/ray-on-aws/blob/main/lesson/images/dashboard-job-detail.png)
