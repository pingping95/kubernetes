# Architecture

# ELK Stack?

- Logstash
    - Opensource
    - 다양한 소스(DB, csv 파일 등)의 로그 또는 트랜잭션 데이터를 수집, 집계, 파싱하여 Elasticsearch로 전달
- Elasticsearch
    - Logstash 로부터 받은 데이터를 검색 및 집계를 하여 필요한 관심 있는 정보를 **획득**
- Kibana
    - Elasticsearch의 빠른 검색을 통해 데이터를 시각화 및 모니터링

# Architecture

![Untitled](https://user-images.githubusercontent.com/67780144/98467353-2ded0680-2218-11eb-9c2d-4dfa45a3234b.png)

1. Log를 생성하는 Server들에 Filebeat를 설치하고, Log를 집계할 서버에 ELK를 설치한다.
2. Filebeat에서 LogStash로 Log를 전송하고 LogStash에서 한 번 필터링을 거친 로그들이 Elasticsearch에 저장된다.
3. 저장 된 Log는 Kibana를 통해 시각화하여 볼 수 있다.

# 간단한 요약, 필요한 지식...

- Daemonset과 Statefulset에 대한 개념을 파악할 수 있어야 하며 Beats, Syslogs 등에 대한 이해, Logstash에 대한 이해, 왜 RDB가 아닌 Elasticsearch를 사용하는지, 시각화를 위한 Kibana와의 연동..., Ingress를 위한 Traefik, Curator은 왜 사용하는지... Curator... Cronjob
- Filebeat or Metricbeat ⇒ Logstash ⇒ Logstash ⇒ Kibana
- Traefik
- Curator



###  Reference

[https://steemit.com/elk/@modolee/elk-stack](https://steemit.com/elk/@modolee/elk-stack)

[https://medium.com/@tharangarajapaksha/elk-stack-in-k8s-cluster-13bb509185e0](https://medium.com/@tharangarajapaksha/elk-stack-in-k8s-cluster-13bb509185e0)
