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
* 사전 준비 (헤드 노드)
```
sudo dnf install -y docker
sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user
# 재로그인 또는 현재 세션 적용
newgrp docker
```

* (1) prometheus.yml (Ray 자동 감지 설정)
Ray가 생성하는 JSON 파일을 공유하기 위해 경로를 맞춘다.
```
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: 'ray-metrics'
    file_sd_configs:
      - files:
          - '/tmp/ray/prom_metrics_service_discovery.json'
```
* (2) docker-compose.yml
Ray의 지표 파일이 있는 /tmp/ray를 컨테이너 안으로 볼륨 마운트하는 것이 핵심입니다.
```
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
```

* (3) 실행 및 확인
```
docker-compose up -d
```

* (4) 대시보드 연결
    * Grafana 접속: http://<HEAD_IP>:3000 (ID/PW: admin/admin)
    * 데이터 소스 추가: Add Data Source -> Prometheus 선택 -> URL에 http://prometheus:9090 입력 후 Save & Test.
    * 대시보드 임포트: Ray 공식 Grafana 대시보드(ID: 16098)를 임포트하면 즉시 화려한 차트가 나옵니다.

### 주의사항 ###
* 보안 그룹: AWS Console에서 헤드 노드의 보안 그룹 규칙에 9090과 3000 포트를 열어주어야 외부 브라우저에서 보입니다.
* 권한: /tmp/ray 폴더의 권한 때문에 Prometheus가 파일을 못 읽는다면 chmod -R 755 /tmp/ray를 실행하세요.

## 레퍼런스 ##
* https://leehosu.github.io/kube-prometheus-stack

