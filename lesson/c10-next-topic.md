워크숍 완성도를 높일 3가지 제안
* C6 - Object Spilling & OOM 방지:
텍스트 데이터는 개별 크기는 작지만 수백만 개가 쌓이면 메모리 압박이 심합니다.
RAY_object_spilling_config를 통해 S3를 스필링 위치로 설정하는 트릭을 넣으시면, 로컬 디스크가 부족한 환경에서도 테라바이트급 데이터를 안전하게 처리하는 실습이 가능해집니다. Ray Object Spilling Guide를 참고해 보세요.

* C8 - 텍스트 전처리 (Docling 연동):
단순 정규식 전처리 외에, 앞서 논의한 Docling 같은 라이브러리를 Ray의 map_batches에 얹어서 "PDF to Clean Markdown" 파이프라인을 실습 예제로 넣으시면 수강생들의 만족도가 매우 높을 것 같습니다.

* C5 - 스팟 인스턴스 전략:
스팟 인스턴스 중단(Preemption) 시 Ray의 Fault Tolerance가 어떻게 작동하는지(상태 복구)를 시각적으로 모니터링(C7)과 연결해 보여주면 "천재들의 설계"를 가장 확실히 체험하는 포인트가 될 것입니다.
