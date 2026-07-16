## Lab 3. 모델 학습 하기 ##

- RayJob으로 학습 스크립트 제출 (namespace/toleration 격리)
- Ray Train vs native(torchrun/DDP) 언제 무엇을 (5번 문서 내용 활용)
- Spot 인스턴스 + 체크포인팅으로 학습 비용 절감
- ✅ 체크포인트: 학습이 고객 전용 GPU 노드에만 스케줄되는지 검증

### 훈련하기 ###

```
apiVersion: ray.io/v1
kind: RayJob
metadata:
  name: customer-a-mnist-train
  namespace: customer-a          # ⚠️ 고객 A의 격리된 네임스페이스 지정
spec:
  entrypoint: python train.py    # 실행할 학습 명령
  runtimeEnvYAML: |
    pip:
      - torch==2.1.0
      - torchvision
    working_dir: "http://gitea-http.gitea.svc.cluster.local:3000/admin/saas-infra/raw/branch/main/code" # Gitea 안의 소스코드 경로

  # 이 학습 작업을 실행할 임시 Ray 클러스터 사양
  rayClusterSpec:
    rayVersion: '2.9.0'
    headGroupSpec:
      rayStartParams:
        dashboard-host: '0.0.0.0'
    workerGroupSpecs:
      - groupName: gpu-workers
        replicas: 1              # GPU 일꾼 1대 필요 선언!
        minReplicas: 1
        maxReplicas: 2
        template:
          spec:
            # ⚠️ 핵심: 고객 A 전용 노드로만 들어가게 격리 딱지(Toleration) 부착
            tolerations:
              - key: "customer"
                operator: "Equal"
                value: "customer-a"
                effect: "NoSchedule"
            nodeSelector:
              customer: "customer-a"
            containers:
              - name: ray-worker
                image: rayproject/ray:2.9.0-py310
                resources:
                  limits:
                    nvidia.com/gpu: "1" # GPU 1장 요구
```
 
```
# 로컬에 학습 코드 폴더(mnist_src)가 있는 상태에서 실행
ray job submit \
  --address "http://ray-cluster-customer-a-head-svc.customer-a.svc.cluster.local:8265" \
  --working-dir ./mnist_src \
  --runtime-env-json '{"pip": ["torch==2.1.0", "torchvision"]}' \
  -- python train.py
```
