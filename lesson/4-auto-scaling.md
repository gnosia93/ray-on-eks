## 오토 스케일링 ##

### 1. Job 실행하기 ###
vs-code 콘솔에서 실행한다. 
```
cat <<EOF > stress-job.py
import ray, time
ray.init(address="auto")

@ray.remote(num_cpus=1)
def heavy_task():
    time.sleep(120)             # 2분간 코어 점유
    return True

# 512개의 태스크를 한꺼번에 실행한다. (하나당 2분)  
ray.get([heavy_task.remote() for _ in range(512)])
EOF
```

클러스터의 워커노드는 m8g.2xlarge, m7i.2xlarge 각각 1대씩 총 16장의 CPU 가 할당되어 있고, 최대 8 GPU * 16 대 = 256 개의 CPU 까지 확장 가능하다 
아래 명령어로 Job 을 제출하고 어떻게 동작하는지 관찰한다. 
```
ray job submit --address http://localhost:8265 --working-dir . -- python stress-job.py
```

### 2. 스케일링 관찰하기 ###
```
ray exec ~/cluster.yaml "ray status"
```
[결과]
```
Loaded cached provider configuration
If you experience issues with the cloud provider, try re-running the command with --no-config-cache.
/home/ec2-user/.local/lib/python3.9/site-packages/boto3/compat.py:89: PythonDeprecationWarning: Boto3 will no longer support Python 3.9 starting April 29, 2026. To continue receiving service updates, bug fixes, and security updates please upgrade to Python 3.10 or later. More information can be found here: https://aws.amazon.com/blogs/developer/python-support-policy-updates-for-aws-sdks-and-tools/
  warnings.warn(warning, PythonDeprecationWarning)
Fetched IP: 10.0.2.177
======== Autoscaler status: 2026-01-20 04:28:21.914768 ========
Node status
---------------------------------------------------------------
Active:
 8 x86_worker_node
 8 arm_worker_node
 1 head_node
Idle:
 (no idle nodes)
Pending:
 (no pending nodes)
Recent failures:
 (no failures)

Resources
---------------------------------------------------------------
Total Usage:
 68.0/68.0 CPU
 0.0/8.0 Graviton
 0.0/9.0 Intel
 0B/368.27GiB memory
 0B/148.62GiB object_store_memory

From request_resources:
 (none)
Pending Demands:
 {'CPU': 1.0}: 408+ pending tasks/actors
Shared connection to 10.0.2.177 closed.
```

실행한 Job 의 상세 정보를 대시보드에서 조회한다. 
![](https://github.com/gnosia93/ray-on-aws/blob/main/lesson/images/dashboard-job-log.png)

해당 Job에서 실행중인 태스크도 관찰할 수 있다.
![](https://github.com/gnosia93/ray-on-aws/blob/main/lesson/images/dashboard-job-task.png)

### 3. 세부 조정 파라미터 (YAML) ###

* target_utilization_fraction (기본값 0.8)    
클러스터 전체 자원 사용률이 이 수치를 넘으면 미리 여유분을 준비한다. 예를 들어 160코어 중 128코어(80%)만 써도 EC2 노드를 신규로 프러비저닝 한다. 

* upscaling_speed  
한 번에 얼마나 공격적으로 인스턴스를 늘릴지 결정하는데 이 값이 높을수록(예: 10) 한 번에 많은 EC2를 동시에 요청한다.

* Idle Timeout:
노드에 할당된 작업이 하나도 없고, idle_timeout_minutes(예: 5분) 동안 아무 일도 하지 않으면 해당 노드를 삭제 대상으로 간주하고 노드의 수량을 줄인다.
이때 Min Workers 이하로는 줄어들지 않는다.
