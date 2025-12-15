# 📂 목록

- 🪤 [Airflow 구축하기 (Nas, Ubuntu)](./airflow.md)
- ⚠️ [데이터 불일치](./different_data.md)
- 🌐 [다국어 데이터 매핑 및 처리량 증가 문제](./i18n_mapping.md)
- 🔹 [다국어 원천 데이터의 신뢰도 문제](./untranslated_data.md)
- 📦 [Data Dump 자동화 설정](./data_dump.md)
- 🐹 [시스템 Health Check 구축](./health_check.md)
- 🧠 [아이템 상세 페이지 성능 튜닝 후기](./item_detail.md)

# 🐹 시스템 Health Check 구축

2025년 5월 22일, 외부에서 서버로의 접속이 단절되는 문제가 발생하였습니다.

조사 결과, **LG 인터넷 회선 점검 이후 외부 IP가 변경된 것이 원인**이었습니다.

- 문제 인지 시점: 접속 단절 발생 후 약 1시간 20분 경과 후
- 조치 소요 시간: 약 10분
- 총 장애 시간: 약 1시간 30분

문제 발생 당시, 서버가 원격 지역에 위치해 있었고 외부 접속이 불가능한 상황이라 즉각적인 대응이 어려웠습니다.

이로 인해 상당한 아쉬움과 무력감을 느꼈습니다.

또한, 원인을 파악하기 전까지는 SSD 고장으로 인해 데이터가 모두 손실되었을 가능성을 우려하며 큰 불안을 겪었습니다.

이후 장애 복구 후, 서버 상태를 실시간으로 모니터링할 수 있는 시스템 필요성을 절실히 느꼈고 점검 시스템을 개발하게 되었습니다.

- DB Data Dump후 Email로 전송 기능 - 기존에는 dump만 진행
- 내부 시스템 Health Check - 신규 기능
- 외부 접속 상태 Health Check - 신규 기능

> Airflow 3.1.3 기준으로 내용 수정

# ✅ DB Data Dump 후 이메일 전송 기능 개선

기존에는 DB Data Dump 후 4일이 지나면 파일을 삭제하는 방식으로만 운영되고 있었습니다.

하지만 주기적으로 로컬에 백업 파일을 다운로드하지 않으면, SSD 고장 등 비정상적인 상황 발생 시 데이터를 복구할 수 없는 문제가 있었습니다.

이에 따라, 최우선 과제로 개선 작업을 진행했습니다.

**기존**

![스크린샷 2025-05-21 오전 9 20 51](https://github.com/user-attachments/assets/6cf84d60-7e56-43da-a59a-07cee0da86f0)

**수정**

![스크린샷 2025-05-26 오후 1 51 02](https://github.com/user-attachments/assets/ab62b0c6-9980-45b0-a3bc-2a041b0105b7)

**서비스 상태 및 응답시간 적재 Task 추가 (사이트 통계)**

![스크린샷 2025-06-15 083945](https://github.com/user-attachments/assets/2d967165-eb7a-4903-9b28-69b85fbc4e16)

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

## ⚠️ 주의할 점

해당 스크립트는 실행 후 문제가 발생할 경우 이메일로 오류 내용을 전송해야 합니다.

메일기능을 이용하기 위해서는 단순히 crontab으로 주기 실행을 설정하는 것은 적절하지 않습니다.

Airflow는 Docker Container 내부에서 실행되므로, 외부에서 접근하려면 별도의 IP 및 포트 설정 정보가 필요합니다.

제가 설정했던 IP 접속 방식을 남깁니다.

**Airflow Container 패키지 설치**

```shell
docker exec -u 0 -it 861824de8429 bash
apt-get install clickhouse-client
apt-get install -y kafkacat
apt-get install jq
```

**접속 방식**

Airflow: localhost

나머지 서비스: 192 아이피 - Ubuntu와 동일(docker를 설치한 환경)

## Health Check 목록 및 방식

점검하고자 하는 목록과 Health Check 방식입니다.

**Check List**

    - NextJS
        - Health Check API 개발 - /api/health
        - 서비스 응답 시간 Check
    - FastAPI
        - Health Check API 개발 - /api/news/health
        - 서비스 응답 시간 Check
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

현재 적용중인 스크립트 정보 입니다.

<details>
<summary>🔍 <strong>🔍 <strong>Health Check Script</strong></summary>

```shell
#!/bin/bash

# =========================
# 기본 설정
# =========================
LOG_FILE="/opt/airflow/latest_data/health_check.log"
TIMEOUT=5

# 로그 초기화
: > "$LOG_FILE"

# =========================
# 로그 함수
# =========================
log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" >> "$LOG_FILE"
}

check() {
  local name="$1"
  local cmd="$2"

  if timeout ${TIMEOUT}s bash -c "$cmd" >/dev/null 2>&1; then
    log "✅ [OK]   ${name}"
  else
    log "❌ [FAIL] ${name}"
  fi
}

log "==================== Health Check Start ===================="

# =========================
# HTTP 서비스
# =========================
check "FastAPI" \
  "curl -sf http://192.168.219.102:9022/api/news/health"

check "NextJS" \
  "curl -sf http://192.168.219.102:4002/api/health"

check "ClickHouse" \
  "curl -sf http://192.168.219.102:8123/ping"

check "MinIO" \
  "curl -sf http://192.168.219.102:9000/minio/health/live"

# =========================
# TCP 서비스
# =========================
check "Redis" \
  "nc -z 192.168.219.102 6379"

check "Kafka" \
  "nc -z 192.168.219.102 9092"

# =========================
# Database
# =========================
check "PostgreSQL" \
  "psql -U tkl -h 192.168.219.102 -p 13245 -d prd -c 'SELECT 1;'"

# =========================
# Docker 내부 서비스
# =========================
check "Nginx Proxy Manager" \
  "curl -sf http://npm:81"

check "Airflow API Server" \
  "curl -sf http://airflow-airflow-apiserver-1:8080/api/v2/monitor/health"

log "==================== Health Check End ======================"
echo "" >> "$LOG_FILE"
```

</details>

# 🌐 외부 접속 서비스 Health Check

해결하려고 했던 문제 중 하나는, 외부에서 접속 가능한 서비스의 Health Check였습니다.

하지만 해당 작업은 내부 서버 환경에서는 불가능했고, 별도의 장비가 필요한 상황이었습니다.

웹 서핑 중 우연히 [UptimeRobot](https://uptimerobot.com/) 이라는 유용한 사이트를 알게 되었습니다.

무료 플랜에서도 5분 간격으로 서버 상태를 체크할 수 있고, 장애가 발생하면 이메일로 알림을 받을 수 있습니다.

이렇게 간단한 서비스라면 조금만 더 관심을 가지고 미리 찾아봤으면 좋았겠다는 아쉬움이 남습니다.

비록 최대 5분간은 장애를 인지하지 못할 수 있지만, 기존에 1시간 20분 동안 문제를 전혀 알지 못했던 상황보다는 훨씬 개선된 환경이 될 것입니다.

**대시보드**

![스크린샷 2025-05-27 오전 8 26 15](https://github.com/user-attachments/assets/4d84e52a-a167-4238-86de-5235d7a2b71d)

**Notification Test**

![스크린샷 2025-05-27 오전 8 26 02](https://github.com/user-attachments/assets/3d86e744-d1bb-4992-9a19-e2fe9772b461)

---

## Health Check Dag 코드

현재 적용중인 Health Check 코드입니다.

<details>
<summary><strong>🔍 <strong>Dag Code</strong></summary>

```python
from airflow import DAG
from airflow.providers.standard.operators.bash import BashOperator
from airflow.providers.standard.operators.python import BranchPythonOperator
from airflow.providers.standard.operators.python import PythonOperator
from airflow.providers.postgres.hooks.postgres import PostgresHook
from contextlib import closing
from custom_module.psql_func import read_sql
from airflow.providers.smtp.operators.smtp import EmailOperator
from airflow.providers.standard.operators.empty import EmptyOperator
from datetime import datetime, timezone
import time
import os
import requests
import re

LOG_PATTERN = re.compile(
    r"""
    \[(?P<ts>[\d\-:\s]+)\]      # timestamp
    \s+
    (?:✅|❌)                   # emoji
    \s+
    \[(?P<status>OK|FAIL)\]    # status
    \s+
    (?P<service>.+)            # service name
    """,
    re.VERBOSE,
)

log_path = "/opt/airflow/latest_data/health_check.log"

default_args = {
    "owner": "airflow",
    "email_on_failure": False,
    "retries": 0,
}

with DAG(
    dag_id="dags_health_check",
    default_args=default_args,
    schedule="*/5 * * * *",
    start_date=datetime(2024, 1, 1, tzinfo=timezone.utc),
    catchup=False,
    tags=["health", "monitoring"],
) as dag:

    # 1. health_check.sh 실행
    run_health_check = BashOperator(
        task_id="run_health_check",
        bash_command="""
        bash /opt/airflow/plugins/script/health_check.sh
        """,
    )

    def measure_response_time(postgres_conn_id, **kwargs):
        services = {
            "Next.js": "https://eftlibrary.com/health",
            "FastAPI": "https://back.eftlibrary.com/health",
        }

        postgres_hook = PostgresHook(postgres_conn_id)
        sql = read_sql("insert_response_time.sql")

        with closing(postgres_hook.get_conn()) as conn:
            with closing(conn.cursor()) as cursor:
                for service_name, url in services.items():
                    start = time.time()
                    try:
                        r = requests.get(url, timeout=10)
                        elapsed = time.time() - start
                    except Exception:
                        elapsed = None  # 실패 시 NULL 처리

                    cursor.execute(sql, (service_name, elapsed, datetime.now()))
            conn.commit()


    def save_health_check(postgres_conn_id, **kwargs):
        if not os.path.exists(log_path):
            return

        postgres_hook = PostgresHook(postgres_conn_id)
        sql = read_sql("insert_health_check.sql")

        with closing(postgres_hook.get_conn()) as conn:
            with closing(conn.cursor()) as cursor:
                with open(log_path, "r") as f:
                    for line in f:
                        match = LOG_PATTERN.search(line)
                        if not match:
                            continue  # START / END 등은 무시

                        checked_at = datetime.strptime(
                            match.group("ts"), "%Y-%m-%d %H:%M:%S"
                        ).replace(tzinfo=timezone.utc)

                        service_name = match.group("service").strip()
                        status = match.group("status")

                        cursor.execute(
                            sql,
                            (service_name, status, checked_at),
                        )
            conn.commit()

    # 2. 로그 내용 검사 (FAIL 포함 여부)
    def check_fail_in_log(**kwargs):
        if os.path.exists(log_path):
            with open(log_path) as f:
                content = f.read()
                if "FAIL" in content:
                    return "prepare_email_body"
        return "success_action"

    # 3. 실패한 경우: 로그 내용을 읽어 XCom으로 전달
    def prepare_email_content(**kwargs):
        ti = kwargs["ti"]
        if os.path.exists(log_path):
            with open(log_path) as f:
                content = f.read()
                html_content = f"<pre>{content}</pre>"
                ti.xcom_push(key="email_body", value=html_content)

    check_log_result = BranchPythonOperator(
        task_id="check_log_result",
        python_callable=check_fail_in_log,
    )

    save_to_postgres = PythonOperator(
        task_id="save_to_postgres",
        python_callable=save_health_check,
        op_kwargs={"postgres_conn_id": "tkl_db"},
    )

    prepare_email_body = PythonOperator(
        task_id="prepare_email_body",
        python_callable=prepare_email_content,
    )

    measure_response_time_task = PythonOperator(
        task_id="measure_response_time",
        python_callable=measure_response_time,
        op_kwargs={"postgres_conn_id": "tkl_db"},
    )

    # 4. 이메일 전송 (FAIL이 있을 때만 실행됨)
    send_email = EmailOperator(
        task_id="send_email",
        to=["a@gmail.com"],
        cc=["b@gmail.com", "c@gmail.com"],
        subject="🚨 EFT Library 서비스에 문제가 생겼습니다.",
        html_content="{{ task_instance.xcom_pull(task_ids='prepare_email_body', key='email_body') }}",
        conn_id="smtp_gmail",
        from_email="a@gmail.com",
    )

    # 5. 성공 시 아무 것도 안 함
    success_action = EmptyOperator(task_id="success_action")

    # DAG 연결
    (
        run_health_check
        >> save_to_postgres
        >> measure_response_time_task
        >> check_log_result
    )
    check_log_result >> prepare_email_body >> send_email
    check_log_result >> success_action

```

</details>
