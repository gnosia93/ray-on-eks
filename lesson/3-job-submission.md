
vs-code 서버의 콘솔에서 아래 명령어를 실행한다.   
```
# 1. 테스트 잡 생성
cat <<EOF > test_job.py
import ray
import platform

ray.init()

@ray.remote(resources={"Intel": 1})
def run_on_intel():
    return f"Intel 노드 실행 중: {platform.machine()}"

@ray.remote(resources={"Graviton": 1})
def run_on_arm():
    return f"Graviton 노드 실행 중: {platform.machine()}"

print("결과 확인:", ray.get([run_on_intel.remote(), run_on_arm.remote()]))
EOF

# 2. 작업 제출 (배스천의 8265 포트로 연결)
ray job submit --address http://localhost:8265 --working-dir . -- python test_job.py
````
