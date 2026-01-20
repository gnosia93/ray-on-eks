## 오토 스케일링 ##
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
* Ray Status: Pending Tasks 발생 -> Autoscaler가 Launching 20 Nodes 시작.
* AWS Console: EC2 인스턴스 페이지에 인스턴스 생성을 관찰.
