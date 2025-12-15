# 📂 목록

- 🛠️ [Kafka 환경 구축](./kafka_system_development.md)
- 🚀 [ClickHouse와 PostgreSQL 비교 및 테이블 설계](./clickhouse_postgresql.md)

# 🛠️ Kafka 환경 구축

현재 사이트의 **사용자 행동 데이터는 Google Analytics에 의존**하고 있었습니다.  
하지만 도구 자체에 익숙하지 않고, 관리자 접속 정보까지 포함되는 등 **불편한 점이 있었습니다.**

또한 **더 정밀한 사용자 행동 분석을 위해 별도의 수집 시스템이 필요하다고 판단**했습니다.  
이에 따라, **Kafka를 도입하여 사용자 페이지 이동 히스토리를 직접 수집하는 방식을 채택**하였습니다.

Kafka를 선택한 주요 이유는 다음과 같습니다:

- **실시간 데이터 스트림 처리에 적합한 구조**를 가지고 있어 사용자 이벤트 수집에 매우 적합합니다.
- 추후 **커뮤니티 기능을 다시 활성화**하게 될 경우,
  - **댓글 알림**, **좋아요 알림** 등의 실시간 이벤트 처리에도 Kafka를 활용할 수 있다는 점에서 확장성 면에서도 유리하다고 판단했습니다.
- 단순한 로그 수집을 넘어 **서비스 전반에 이벤트 기반 구조(event-driven architecture)**를 설계할 수 있는 기반을 마련하고 싶었습니다.

앞으로 Kafka를 기반으로 한 사용자 행동 분석과 더불어, 이벤트 기반 알림 시스템까지 범위를 확장할 계획입니다.

> Version: 2.13-4.1.1

## 🧱 설계 및 저장소 결정

초기에는 **PostgreSQL에만 적재하는 방식으로 생각**했지만,

대시보드나 통계 처리 목적을 고려하면 **OLAP 성능이 뛰어난 ClickHouse가 더 적합할 수 있다는 판단**이 들었습니다.

처음 도입하는 만큼, 두 시스템 모두에 데이터를 저장하여 **성능 및 활용성 비교도 함께 진행**해보기로 했습니다.

## ⚙️ 구축 개요

**PostgreSQL은 기존에 설치되어 있는 상태입니다.**

**Kafka와 ClickHouse를 단일 서버(Stand-alone) 환경에서 구축합니다. (리소스가 부족해 분산 환경은 고려하지 않았습니다.)**

## 🧩 Ubuntu 커널 매개변수 설정 이유

    Kafka와 ClickHouse를 운영하기 전, 다음과 같은 커널 설정이 필요한 이유를 이해하고 적용해야 합니다
    Delay Accounting은 리눅스 커널에서 프로세스가 I/O, 스케줄링 지연, 락 대기 등으로 인해 실제로 수행되지 못한 시간을 측정하는 기능입니다.
    ClickHouse에서는 이를 기반으로 다음과 같은 내부 통계를 수집합니다
    OSIOWaitMicroseconds : ClickHouse가 I/O 대기 시간과 관련된 지연을 측정하는 데 사용하는 지표입니다.
    해당 기능이 비활성화된 경우, 정확한 성능 진단 및 지연 통계 수집이 불가능해질 수 있습니다.

**구축 과정**

```shell
# 자바 21 설치
sudo rm -f /etc/yum.repos.d/adoptium.repo

# repo 추가
sudo tee /etc/yum.repos.d/adoptium.repo <<EOF
[Adoptium]
name=Adoptium
baseurl=https://packages.adoptium.net/artifactory/rpm/rhel/9/x86_64
enabled=1
gpgcheck=1
gpgkey=https://packages.adoptium.net/artifactory/api/gpg/key/public
EOF

# 캐시 날리기 및 설치
sudo dnf clean all
sudo dnf makecache
sudo dnf install temurin-21-jdk -y

# 클러스터 id 생성 - r5Gh_pP6Tf-L-wwH_aP4Vg
/bin/kafka-storage.sh random-uuid

# server.properties 수정
controller.quorum.voters=1@localhost:9093 // 이거는 없으면 추가
listeners=PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
advertised.listeners=PLAINTEXT://localhost:9092,CONTROLLER://localhost:9093

# 메타데이터 초기화
bin/kafka-storage.sh format -t r5Gh_pP6Tf-L-wwH_aP4Vg -c config/server.properties

# log, pid파일 생성
nohup /home/kafka/kafka_2.13-4.1.1/bin/kafka-server-start.sh /home/kafka/kafka_2.13-4.1.1/config/server.properties \
    > /home/kafka/kafka_2.13-4.1.1/kafka.log 2>&1 & echo $! > /home/kafka/kafka_2.13-4.1.1/kafka.pid

# kafka 실행
/home/kafka/eft-library-kafka/kafka.sh start

# 토픽 생성
/home/kafka/kafka_2.13-4.1.1/bin/kafka-topics.sh \
  --create \
  --topic web-logs-topic \
  --bootstrap-server localhost:9092 \
  --partitions 1 \
  --replication-factor 1 \
  --config retention.ms=86400000
/home/kafka/kafka_2.13-4.1.1/bin/kafka-topics.sh \
  --create \
  --topic user-notification \
  --bootstrap-server localhost:9092 \
  --partitions 3 \
  --replication-factor 1 \
  --config retention.ms=604800000

# 토픽 리스트 확인
/home/kafka/kafka_2.13-4.1.1/bin/kafka-topics.sh --list --bootstrap-server localhost:9092
```

# 현재 사용 현황

현재는 2가지 방식으로 사용하고 있습니다.

1. web log 수집

    사용자들이 방문한 페이지들을 수집하여 사이트 방문 통계를 만들고 있습니다.

2. 사용자 알림 전송

    커뮤니티 기능이 추가되면서 댓글, 글 작성 및 좋아요 알림에 사용하고 있습니다.

추후 써드파티로 WPF를 사용하여 내 위치 찾기와 연걸하는 윈도우 앱을 만들 예정인데 그 때도 사용할 예정입니다.