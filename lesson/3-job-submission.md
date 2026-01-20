vs-code 서버의 터미널에서 아래 명령어를 실행한다.   

```
cd ~
mkdir -p my-ray
cd my-ray

cat <<EOF > test-job.py
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

pwd
```

Job 을 제출한다. 동시에 ray 는 해당 디렉토리에 있는 파일을 전부 압축하여 클러스터로 업로드 한다. (업로드 제한 용량 100MB)
```
ray job submit --address http://localhost:8265 --working-dir . -- python test_job.py
````
