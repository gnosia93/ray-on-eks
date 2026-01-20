## 오토 스케일링 ##
vs-code 콘솔에서 실행한다. 
```
cat <<EOF > stress.py
import ray, time
ray.init(address="auto")

@ray.remote(num_cpus=1)
def heavy_task():
    time.sleep(120)             # 2분간 코어 점유
    return True

# 480개(30대 분량)의 작업을 한꺼번에 투척!
ray.get([heavy_task.remote() for _ in range(256)])
EOF
```

```
ray run 
```
* Ray Status: Pending Tasks 발생 -> Autoscaler가 Launching 20 Nodes 시작.
* AWS Console: EC2 인스턴스 페이지에 인스턴스 생성을 관찰.
