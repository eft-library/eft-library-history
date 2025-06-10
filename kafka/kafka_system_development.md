# 📂 목록

- 🛠️ [Kafka 환경 구축](./kafka_system_development.md)
- 🚀 [ClickHouse와 PostgreSQL 비교 및 테이블 설계](./clickhouse_postgresql.md)
- 📊 [대시보드 성능 테스트](./dashboard.md)

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
# java 21 설치
apt install openjdk-21-jre-headless

# kafka 4.0.0 다운로드 및 압축해제
wget https://downloads.apache.org/kafka/4.0.0/kafka_2.13-4.0.0.tgz
tar -xzf kafka_2.13-4.0.0.tgz

# 외부에서 접근할 수 있게 방화벽 해제
ufw allow 8123
ufw allow 9232 # 기본 port는 9000임, 현재 환경에서는 겹치기에 9232로 변경해서 사용

# 로그 디렉토리 초기화 (메타 데이터 설정)
bin/kafka-storage.sh format --config config/server.properties --cluster-id $(bin/kafka-storage.sh random-uuid) --standalone

# Kafka Stand-alone 구동 테스트
# 토픽 생성
/home/kafka/kafka_2.13-4.0.0/bin/kafka-topics.sh --create --topic web-logs-topic --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1

# 토픽 리스트 확인
/home/kafka/kafka_2.13-4.0.0/bin/kafka-topics.sh --list --bootstrap-server localhost:9092

# 메시지 Producer
/home/kafka/kafka_2.13-4.0.0/bin/kafka-console-producer.sh --topic web-logs-topic --bootstrap-server localhost:9092

# 메시지 Consume
/home/kafka/kafka_2.13-4.0.0/bin/kafka-console-consumer.sh --topic web-logs-topic --from-beginning --bootstrap-server localhost:9092

# 이전에 작성한 FastAPI를 사용해서 Middleware Producer 개발

# 이전에 작성한 PostgreSQL 테이블 생성

# ClickHouse Stand-alone
# Ubuntu 커널 매개변수 활성화 - 재부팅해도 적용되게 설정
echo 1 | sudo tee /proc/sys/kernel/task_delayacct
sudo sh -c 'echo "kernel.task_delayacct=1" >> /etc/sysctl.conf'
sudo sysctl -p

# ClickHouse Package 설정 및 설치 - 설치 할 때 default 계정 비밀번호 입력하라고 나오면 입력
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg

curl -fsSL 'https://packages.clickhouse.com/rpm/lts/repodata/repomd.xml.key' | sudo gpg --dearmor -o /usr/share/keyrings/clickhouse-keyring.gpg

ARCH=$(dpkg --print-architecture)

echo "deb [signed-by=/usr/share/keyrings/clickhouse-keyring.gpg arch=${ARCH}] https://packages.clickhouse.com/deb stable main" | sudo tee /etc/apt/sources.list.d/clickhouse.list

sudo apt install clickhouse-server clickhouse-client -y

# ClickHouse 설정 수정
sudo vi /etc/clickhouse-server/config.xml

# 이미 사용중인 port가 있어서 안겹치게 수정
tcp port 9000 => 9232

# 외부에서도 접속할 수 있게 0.0.0.0으로 변경
<listen_host>0.0.0.0</listen_host>

# 저장
:w!

# ClickHouse 실행 및 접속 - 설치할 때 입력한 default 계정의 비밀번호 입력 필요
sudo clickhouse start
clickhouse-client --port 9232 # port가 달라서 부여한 값

# ClickHouse 사용자, DB 생성 및 권한 부여
create database prd;
create user test IDENTIFIED With plaintext_password by 'password';
GRANT SELECT, INSERT, CREATE, UPDATE, DELETE, TRUNCATE, DROP ON prd.* TO test;

# 이전에 작성한 ClickHouse 테이블 생성
```

## Kafka 작성

초창기에 작성하여 적용한 버전입니다.

<details>
<summary>🔍 <strong>🔍 <strong>Kafka Code</strong></summary>

**config.py**

```python
import os
from dotenv import load_dotenv

load_dotenv()

LOG_DIR = os.getenv("LOG_DIR", "./logs")
LOG_FILE = os.path.join(LOG_DIR, "consumer.log")

# PostgreSQL
PG_CONFIG = {
    "host": os.getenv("DB_HOST"),
    "port": os.getenv("DB_PORT"),
    "dbname": os.getenv("DB_NAME"),
    "user": os.getenv("DB_USER"),
    "password": os.getenv("DB_PASSWORD"),
}

# ClickHouse
CH_CONFIG = {
    "host": os.getenv("CH_HOST", "localhost"),
    "port": int(os.getenv("CH_PORT", "8123")),
    "username": os.getenv("CH_USER", "default"),
    "password": os.getenv("CH_PASSWORD", ""),
}

# Kafka
KAFKA_CONFIG = {
    "bootstrap.servers": os.getenv("BOOTSTRAP_SERVER"),
    "group.id": os.getenv("GROUP_ID"),
    "auto.offset.reset": os.getenv("OFFSET_RESET_CONFIG"),
    "topic": os.getenv("TOPIC"),
}
```

**ch_client.py**

```python
import clickhouse_connect
from consumer.config import CH_CONFIG
from consumer.logger import logger
from datetime import datetime, timezone, timedelta
import re


def parse_timestamptz(dt_str):
    # ISO 8601 포맷 맞춰서 초 단위 이하 자릿수 정리 (필요시)
    # '2025-05-19T13:45:30.123+09:00' 같은 문자열을 datetime 객체로 변환

    # 타임존 오프셋 포함된 부분 분리
    match = re.match(r"(.*)([+-]\d{2}):(\d{2})$", dt_str)
    if match:
        dt_without_tz = match.group(1)
        tz_hours = int(match.group(2))
        tz_minutes = int(match.group(3))
        tz_delta = timezone(timedelta(hours=tz_hours, minutes=tz_minutes))
        dt = datetime.strptime(dt_without_tz, "%Y-%m-%dT%H:%M:%S.%f")
        dt = dt.replace(tzinfo=tz_delta)
    else:
        # 타임존이 Z(UTC)일 경우 처리
        dt = datetime.strptime(dt_str, "%Y-%m-%dT%H:%M:%S.%fZ")
        dt = dt.replace(tzinfo=timezone.utc)

    # UTC로 변환 후 tzinfo 제거
    dt_utc = dt.astimezone(timezone.utc).replace(tzinfo=None)
    return dt_utc


def get_clickhouse_client():
    return clickhouse_connect.get_client(**CH_CONFIG)


def save_to_clickhouse(client, data):
    query = """
    INSERT INTO prd.user_footprint (link, request, footprint_time)
    VALUES (%(link)s, %(request)s, %(footprint_time)s)
    """

    params = {
        "link": data["link"],
        "request": data["method"],
        "footprint_time": parse_timestamptz(data["footprint_time"]),
    }

    client.command(query, params)
    logger.info("데이터 ClickHouse 저장 완료")
```

**pg_client.py**

```python
import psycopg2
from consumer.config import PG_CONFIG
from consumer.logger import logger


def get_pg_connection():
    return psycopg2.connect(**PG_CONFIG)


def save_to_postgresql(conn, data):
    with conn.cursor() as cur:
        insert_query = """
        INSERT INTO user_footprint (request, link, footprint_time)
        VALUES (%s, %s, %s)
        """
        cur.execute(
            insert_query, (data["method"], data["link"], data["footprint_time"])
        )
    conn.commit()
    logger.info("데이터 PostgreSQL 저장 완료")

```

**main.py**

```python
import json
from confluent_kafka import Consumer, KafkaError
from consumer.logger import logger
from consumer.config import KAFKA_CONFIG
from consumer.pg_client import get_pg_connection, save_to_postgresql
from consumer.ch_client import get_clickhouse_client, save_to_clickhouse


def main():
    consumer = Consumer(
        {
            "bootstrap.servers": KAFKA_CONFIG["bootstrap.servers"],
            "group.id": KAFKA_CONFIG["group.id"],
            "auto.offset.reset": KAFKA_CONFIG["auto.offset.reset"],
        }
    )
    topic = KAFKA_CONFIG["topic"]
    consumer.subscribe([topic])
    logger.info(f"Subscribed to topic: {topic}")

    pg_conn = get_pg_connection()
    ch_client = get_clickhouse_client()
    logger.info("DB 연결 성공")

    try:
        while True:
            msg = consumer.poll(1.0)
            if msg is None:
                continue
            if msg.error():
                if msg.error().code() == KafkaError._PARTITION_EOF:
                    logger.warning(
                        f"End of partition: {msg.topic()} [{msg.partition()}]"
                    )
                else:
                    logger.error(f"Kafka error: {msg.error().str()}")
                continue

            try:
                data = json.loads(msg.value().decode("utf-8"))
                logger.info(f"Received message JSON: {data}")

                save_to_postgresql(pg_conn, data)
                save_to_clickhouse(ch_client, data)

            except json.JSONDecodeError as e:
                logger.error(f"JSON 디코딩 실패: {e}")
            except Exception as e:
                logger.error(f"DB 저장 실패: {e}")

    except KeyboardInterrupt:
        logger.info("Consumer 종료됨")

    finally:
        consumer.close()
        pg_conn.close()
        logger.info("Consumer 및 PostgreSQL 연결 종료")


if __name__ == "__main__":
    main()
```

</details>
