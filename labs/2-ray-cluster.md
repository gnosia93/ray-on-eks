## Lab 2. Ray 클러스터 프로비저닝 ##

- Gitea repo 생성 (API/tea CLI)
- 템플릿에서 customer-a 클러스터 복사 → commit → push
- ArgoCD 자동 동기화 관찰
- 🔑 베스트 프랙티스: 고객 온보딩을 "폴더 복사 + PR"로 표준화



Phase 1. API를 이용해 원격으로 Gitea에 Repo 생성하기 (curl)
```
# Gitea 내부 ClusterIP 주소를 활용해 API로 'saas-infra' 저장소 생성
curl -X POST "http://gitea-http.gitea.svc.cluster.local:3000/api/v1/user/repos" \
  -H "Content-Type: application/json" \
  -u "admin:admin1234" \
  -d '{"name": "saas-infra", "private": false, "auto_init": true}'
```

Phase 2. 터미널에서 코드 수정하고 Push 하기 (git)
```
# 1. 생성된 저장소 Clone 받기
git clone http://admin:admin1234@gitea-http.gitea.svc.cluster.local:3000/admin/saas-infra.git
cd saas-infra

# 2. 템플릿 폴더에서 새 고객(customer-a) 폴더 구조 복사해 오기
mkdir -p apps/customer-a
cp -r ../templates/kuberay-cluster.yaml apps/customer-a/

# 3. Git Commit & Push 실행
git add .
git commit -m "feat: provision customer-a ray cluster"
git push origin main
```

Phase 3. ArgoCD 동기화 관찰하기 (argocd-cli 또는 kubectl)
```
# ArgoCD CLI로 동기화 상태 확인
argocd app get saas-platform-app --watch

# 또는 간편하게 kubectl로 실시간 Pod 생성 관찰
kubectl get pods -n customer-a -w
```

#### Gitea CLI 공식 도구 'tea' 활용하기 (고급 팁) ####
만약 curl 명령어가 너무 길고 지저분하다고 느껴지신다면, Gitea의 공식 터미널 도구인 **tea**를 수강생 EC2에 미리 설치해 제공하는 것도 아주 좋은 방법입니다.
수강생들은 아래와 같이 심플한 데브옵스 명령어 단 한 줄로 모든 Git 인프라를 통제할 수 있습니다.
```
# Gitea CLI 로그인
tea login add --url http://gitea-http.gitea.svc.cluster.local:3000 --name local-git --token <토큰>

# 터미널에서 Gitea 저장소 바로 생성
tea repo create --name saas-infra
```
