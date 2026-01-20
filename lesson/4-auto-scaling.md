## 오토 스케일링 ##
이제 160코어의 한계를 시험하며 EC2 자원을 최대치로 땡기는 핵심 실습입니다.

vs-code 콘솔에서 실행한다. 
```
cat <<EOF > stress.py
import ray, time
ray.init(address="auto")

@ray.remote(num_cpus=1)
def heavy_task():
    time.sleep(300) # 5분간 코어 점유
    return True

# 480개(30대 분량)의 작업을 한꺼번에 투척!
ray.get([heavy_task.remote() for _ in range(480)])
EOF
```

```
ray run 
```
