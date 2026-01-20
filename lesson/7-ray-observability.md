<< 작성중 >> 

## 익스포터 ##

### 1. Ray 내장 익스포터 (Ray Metrics) ###
* 설치 위치: 모든 노드에 자동으로 포함되어 있습니다.
* 특징: 별도로 설치할 필요가 없습니다. Ray 프로세스가 뜰 때 각 노드의 44217번 포트(기본값)를 통해 Ray 전용 지표(태스크 상태, 오브젝트 스토어 사용량 등)를 내보냅니다.
* 설정: Ray Prometheus 가이드에 따라 프로메테우스 서버가 각 노드의 이 포트를 긁어(Scrape)가도록 설정만 하면 됩니다.

### 2. Node Exporter (하드웨어 지표) ###
* 설치 위치: 모든 EC2 노드에 직접 설치해야 합니다.
* 특징: CPU 온도, 디스크 I/O 상세, 네트워크 인터페이스 상태 등 OS/하드웨어 수준의 데이터를 수집합니다.
* 자동화 방법: cluster.yaml의 setup_commands에 설치 스크립트를 넣어두면 노드가 추가될 때마다 자동으로 깔립니다.
```
# cluster.yaml 예시
setup_commands:
    - wget https://github.com
    - tar xvfz node_exporter-*.tar.gz
    - ./node_exporter-1.7.0.linux-amd64/node_exporter &  # 백그라운드 실행
```

## Docker Compose 로 Prometheus 스택 설치 ###

vs-code 터미널에서 ray attach 를 이용하여 헤드 노드에 접속한다.
```
ray attach cluster_config.yaml
```
[결과]
```
ray attach cluster.yaml
Loaded cached provider configuration
If you experience issues with the cloud provider, try re-running the command with --no-config-cache.
/home/ec2-user/.local/lib/python3.9/site-packages/boto3/compat.py:89: PythonDeprecationWarning: Boto3 will no longer support Python 3.9 starting April 29, 2026. To continue receiving service updates, bug fixes, and security updates please upgrade to Python 3.10 or later. More information can be found here: https://aws.amazon.com/blogs/developer/python-support-policy-updates-for-aws-sdks-and-tools/
  warnings.warn(warning, PythonDeprecationWarning)
Fetched IP: 10.0.2.183
[ec2-user@ip-10-0-2-183 ~]$ 
```

아래 명령어로 헤드 노드에 docker 를 설치한다.
```
sudo dnf install -y docker
sudo dnf install -y docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user
# 재로그인 또는 현재 세션 적용
newgrp docker
docker compose version
```

### docker-compose.yml ###
docker-compose.yml에서 /tmp/ray를 볼륨 마운트하는 이유는 Ray가 실행되면서 생성하는 동적 설정 파일과 메트릭 정보를 Grafana나 Prometheus 같은 모니터링 도구가 읽을 수 있게 하기 위해서이다. 여기서 핵심이 되는 'Ray 지표 관련 파일'은 크게 세 가지이다.
#### 1. Prometheus 서비스 디스커버리 파일 #### 
Ray는 클러스터 내의 여러 노드에서 발생하는 지표를 수집하기 위해, 현재 활성화된 노드들의 주소를 담은 JSON 파일을 자동으로 생성한다.
* 경로: /tmp/ray/prom_metrics_service_discovery.json
* 역할: Prometheus가 이 파일을 읽어 어떤 노드에서 지표(metrics)를 긁어올지(scrape) 스스로 찾아낼 수 있다. 

#### 2. 자동 생성된 모니터링 설정 파일 ####
Ray가 시작될 때, 해당 세션에 최적화된 Prometheus와 Grafana용 설정 파일을 /tmp/ray/session_latest/metrics/ 폴더 안에 생성한다. 
* Prometheus 설정: /tmp/ray/session_latest/metrics/prometheus/prometheus.yml
* Grafana 설정: /tmp/ray/session_latest/metrics/grafana/grafana.ini
별도의 복잡한 설정 없이도 Ray 대시보드와 지표가 연동되도록 미리 정의된 환경 값을 제공한다. 

#### 3. 세션 로그 및 임시 데이터 ####
지표 외에도 문제 해결을 위한 시스템 로그들이 저장된다.
* 경로: /tmp/ray/session_latest/logs/
* 역할: 워커 노드의 상태나 에러 로그를 확인하는 데 사용된다. 

요약하자면, /tmp/ray를 마운트하는 것은 "Ray가 실시간으로 만들어내는 모니터링 지도(JSON)와 설정(YAML/INI)을 컨테이너 간에 공유"하여 대시보드에 데이터가 즉시 나오게 하려는 것이다.
```
cat <<EOF > docker-compose.yml 
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:v2.45.0
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - /tmp/ray:/tmp/ray:ro  # Ray 지표 파일 마운트
    ports:
      - "9090:9090"
    restart: unless-stopped

  grafana:
    image: grafana/grafana:10.0.3
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin # 초기 비번
    restart: unless-stopped
EOF
```

Ray가 생성하는 JSON 파일을 공유하기 위해 경로를 맞춘다. prometheus.yml 에서 Ray 자동 감지 설정을 한다. 
```
cat <<EOF > prometheus.yml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: 'ray-metrics'
    file_sd_configs:
      - files:
          - '/tmp/ray/prom_metrics_service_discovery.json'
EOF
```
도커를 실행한다.
```
sudo curl -SL "https://github.com/docker/compose/releases/latest/download/docker-compose-linux-$(uname -m)" -o ~/docker-compose
sudo chmod +x ~/docker-compose

./docker-compose up -d
```

### 대시보드 연결 ###
    * Grafana 접속: http://<HEAD_IP>:3000 (ID/PW: admin/admin)
    * 데이터 소스 추가: Add Data Source -> Prometheus 선택 -> URL에 http://prometheus:9090 입력 후 Save & Test.
    * 대시보드 임포트: Ray 공식 Grafana 대시보드(ID: 16098)를 임포트하면 즉시 화려한 차트가 나옵니다.

### 주의사항 ###
* 보안 그룹: AWS Console에서 헤드 노드의 보안 그룹 규칙에 9090과 3000 포트를 열어주어야 외부 브라우저에서 보입니다.
* 권한: /tmp/ray 폴더의 권한 때문에 Prometheus가 파일을 못 읽는다면 chmod -R 755 /tmp/ray를 실행하세요.

## 레퍼런스 ##
* https://leehosu.github.io/kube-prometheus-stack

