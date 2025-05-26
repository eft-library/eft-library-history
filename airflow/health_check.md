# 📂 목록

- 🪤 [Airflow 구축하기 (Nas, Ubuntu)](./airflow.md)
- ⚠️ [데이터 불일치](./different_data.md)
- 🌐 [다국어 데이터 매핑 및 처리량 증가 문제](./i18n_mapping.md)
- 🔹 [다국어 원천 데이터의 신뢰도 문제](./untranslated_data.md)
- 📦 [Data Dump 자동화 설정](./data_dump.md)
- 🐹 [시스템 Health Check 구축](./health_check.md)

# 🐹 시스템 Health Check 구축

2025년 5월 22일, 외부에서 서버로의 접속이 단절되는 문제가 발생하였습니다.

조사 결과, **LG 인터넷 회선 점검 이후 외부 IP가 변경된 것이 원인**이었습니다.

- 문제 인지 시점: 접속 단절 발생 후 약 1시간 20분 경과 후
- 조치 소요 시간: 약 10분
- 총 장애 시간: 약 1시간 30분

문제 발생 당시, 서버가 원격 지역에 위치해 있었고 외부 접속이 불가능한 상황이라 즉각적인 대응이 어려웠습니다.

이로 인해 상당한 아쉬움과 무력감을 느꼈습니다.

또한, 원인을 파악하기 전까지는 SSD 고장으로 인해 데이터가 모두 손실되었을 가능성을 우려하며 큰 불안을 겪었습니다.

장애 복구 후, 이러한 상황에 빠르게 대응하기 위해서는 서버 상태를 실시간으로 모니터링할 수 있는 시스템이 필요하다는 점을 절실히 느꼈고, 이후 직접 점검 시스템을 개발하게 되었습니다.

처음 설계는 3가지 시스템을 만들기로 기획 했는데, 마지막 1개는 환경이 갖추어 지지 않아 보류하고 있습니다.

- DB Data Dump후 Email로 전송 기능 - 기존에는 dump만 진행
- 내부 시스템 Health Check - 신규 기능
- 외부 접속 상태 Health Check - 신규 기능, 여건상 보류

# ✅ DB Data Dump 후 이메일 전송 기능 개선

기존에는 DB Data Dump 후 4일이 지나면 파일을 삭제하는 방식으로만 운영되고 있었습니다.

하지만 주기적으로 로컬에 백업 파일을 다운로드하지 않으면, SSD 고장 등 비정상적인 상황 발생 시 데이터를 복구할 수 없는 문제가 있었습니다.

이에 따라, 최우선 과제로 개선 작업을 진행했습니다.

**기존**

![스크린샷 2025-05-21 오전 9 20 51](https://github.com/user-attachments/assets/6cf84d60-7e56-43da-a59a-07cee0da86f0)


**수정**

![스크린샷 2025-05-26 오후 1 51 02](https://github.com/user-attachments/assets/ab62b0c6-9980-45b0-a3bc-2a041b0105b7)


## ✉️ 이메일 설정 방법

Airflow에서 이메일 기능을 설정할 때, 많은 블로그나 문서에서 airflow.cfg를 수정하라고 안내하지만,

공식적으로는 권장되지 않는 방법이며, 설정 시 Warning 메시지도 발생합니다.

✅ 올바른 설정 방법 (Airflow 권장 방식)

    1. Admin → Connections 메뉴 진입
    2. 새 Connection 생성
    3. Conn Type을 Email로 선택
    4. SMTP 서버 정보 입력 (host, port, username, password 등)

**Conn Type Email**
![스크린샷 2025-05-26 오후 2 19 38](https://github.com/user-attachments/assets/8b4408b8-e66f-4ec9-86be-3bdda86761a0)

## 📎 데이터 첨부하여 이메일 전송

초기에는 설정이 올바른데도 불구하고 "Connection Refused" 에러가 지속적으로 발생했습니다.

처음에는 방화벽이나 SMTP 설정 문제로 생각했지만, 실제 원인은 파일 용량으로 인한 전송 지연이었습니다.

🔧 해결 방법

    Data Dump → gzip으로 압축 → 이메일 전송

    예: 원본 10MB → gzip 압축 시 2MB 미만으로 감소
    (※ 텍스트 기반 파일은 압축률이 매우 높습니다.)

**파일 용량 / 압축 용량**
![스크린샷 2025-05-26 오후 2 21 41](https://github.com/user-attachments/assets/43ffc4a8-9fdd-4305-ab55-d13a3901beb9)

## ✅ DAG 결과 확인

DAG 실행 결과, 의도한 대로 데이터가 정상적으로 Dump되고, 이메일로 전송된 후, 4일 뒤 삭제되는 전체 흐름이 정상적으로 작동함을 확인했습니다.

**이메일 결과**
![스크린샷 2025-05-26 오후 2 24 26](https://github.com/user-attachments/assets/12e21b7b-5a6a-4509-bd76-54f5b3d1b7a3)

# 🩺 내부 서비스 Health Check 구축

이번 장애를 계기로, 내부 서비스의 상태를 주기적으로 점검할 수 있는 기능의 필요성을 절감했습니다.

간단한 주기적 Health Check 로직을 작성하여 적용하였습니다.

## 주의할 점

해당 스크립트는 실행 후 문제가 있을 경우 메일로 내용을 보내야 합니다.

그러니 주기적 스케줄링 설정을 crontab이 아닌 Airflow를 사용해야 합니다.

한가지 문제는 Airflow는 Docker container 에 띄워져 있기에 접근할 수 있는 방법이 다릅니다.

그러므로 ip 관련 접속 정보를 남깁니다.

**Airflow Container 패키지 설치**

```shell
docker exec -u 0 -it 861824de8429 bash
apt-get install clickhouse-client
apt-get install -y kafkacat
apt-get install jq
```

**접속 방식**

Airflow: localhost

나머지 서비스: 192 아이피

## Health Check 목록 및 방식

점검하고자 하는 목록과 Health Check 방식입니다.

**Check List**

    - NextJS
        - Health Check API 개발 - /api/health
    - FastAPI
        - Health Check API 개발 - /api/news/health
    - Kafka
        - topic 살아 있는지 확인
    - PostgreSQL
        - 접속 후 select 1 테스트
    - ClickHouse
        - 접속 후 select 1 테스트
    - Docker
        - Airflow
            - 제공해주는 api 테스트 - /health
        - Nginx Proxy Manager
            - curl 테스트
    - Minio
        - 제공해주는 api 테스트 - /minio/health/live

## 내부 서비스 Health Check 구현 스크립트

```shell
#!/bin/bash

LOG_FILE="/opt/airflow/health_check/logs/health_check.log"
TIMESTAMP=$(date "+%Y-%m-%d %H:%M:%S")

rm -f "$LOG_FILE"

log() {
    echo "[$TIMESTAMP] $1" >> "$LOG_FILE"
}

check_http() {
    local name=$1
    local url=$2
    if curl -sf "$url" > /dev/null; then
        log "$name: OK"
    else
        log "$name: FAIL"
    fi
}

check_postgres() {
    if PGPASSWORD=pwd psql -U user -h psql_url -p 11111 -d db -c "SELECT 1;" > /dev/null 2>&1; then
        log "PostgreSQL: OK"
    else
        log "PostgreSQL: FAIL"
    fi
}

check_clickhouse() {
    if echo "SELECT 1;" | clickhouse-client --host house_url --port 1111 --user user --password pwd > /dev/null 2>&1; then
        log "ClickHouse: OK"
    else
        log "ClickHouse: FAIL"
    fi
}

check_kafka_topic() {
    local BROKER="kafka_url:port"
    local TOPIC="topic"

    if kcat -b "$BROKER" -L 2>/dev/null | grep -q "topic \"$TOPIC\""; then
        log "Kafka topic '${TOPIC}': OK"
        return 0
    else
        log "Kafka topic '${TOPIC}': FAIL"
        return 1
    fi
}

check_airflow_health() {
    local url="http://airflow_url:port/health"
    if curl -sf "$url" | jq -e '.metadatabase.status == "healthy" and .scheduler.status == "healthy"' > 2>/dev/null; then
        log "Airflow API Health: OK"
    else
        log "Airflow API Health: FAIL"
    fi
}

check_npm_health() {
    local url="http://npm_url:port"
    if curl -sf "$url" > /dev/null; then
        log "Nginx Proxy Manager Health: OK"
    else
        log "Nginx Proxy Manager Health: FAIL"
    fi
}

### 실행 부분 ###
check_http "Next.js" "http://front_url:port/api/health"
check_http "FastAPI" "http://back_url:port/api/news/health"
check_http "MinIO" "http://minio_url:port/minio/health/live"

check_postgres
check_clickhouse
check_kafka_topic
check_airflow_health
check_npm_health

```

# 🌐 외부 접속 서비스 Health Check (보류 중)

해결하려고 했던 문제 중 하나는, 외부에서 접속 가능한 서비스의 Health Check였습니다.

하지만 해당 작업은 내부 서버 환경에서는 불가능했고, 별도의 장비가 필요한 상황이었습니다.

⚠️ 현재 상황

    외부 접속 모니터링 기능은 여건상 구현 보류

    추후 여유 장비 (예: 라즈베리 파이) 확보 후 외부에서 접근 가능한 Health Check 시스템 구축 예정
