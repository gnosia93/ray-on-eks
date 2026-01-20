# 배스천에서 실행 (stress.py)
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
