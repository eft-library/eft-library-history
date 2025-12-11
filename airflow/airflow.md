# 📂 목록

- 🪤 [Airflow 3.1.3 구축하기](./airflow.md)
- ⚠️ [데이터 불일치](./different_data.md)
- 🌐 [다국어 데이터 매핑 및 처리량 증가 문제](./i18n_mapping.md)
- 🔹 [다국어 원천 데이터의 신뢰도 문제](./untranslated_data.md)
- 📦 [Data Dump 자동화 설정](./data_dump.md)
- 🐹 [시스템 Health Check 구축](./health_check.md)
- 🧠 [아이템 상세 페이지 성능 튜닝 후기](./item_detail.md)

# 🪤 Airflow 3.1.3 구축하기

Airflow는 24시간 실행되어야 하므로, 클라우드를 사용할 경우 지속적인 비용이 발생합니다. 이를 고려하여 **온프레미스 환경**에서 운영하는 것이 비용 면에서 더 유리하다고 판단했습니다.

# Rocky Linux 10.1 버전에 Airflow 설치하기

공식 Docker Image를 활용해, 필요한 부분을 수정하여 설치했습니다.

## Git clone

기존에 제가 작성한 코드들을 내려 받습니다.

```shell
# Git clone
mkdir /home/airflow
git clone <airflow_project_link>

# 필요한 디렉토리 생성 및 권한, 소유자 등록
mkdir latest_data config tmp logs
chmod 777 -R /home/airflow
chwon 50000:50000 -R /home/airflow
```

> **저는 귀찮아서 777로 다 주었는데 이러시면 안됩니다...**

# Airflow 3.1.3 설치

Docker Image를 Pull 해줍니다. - 해당 버전은 Airflow 3.1.3 입니다.

그리고 공식문서에서 docker-compose.yaml을 받아, 이를 기반으로 작성합니다.     
[Docker YML 3.1.3 Link](https://airflow.apache.org/docs/apache-airflow/3.1.3/docker-compose.yaml)

```shell
# Image pull
docker pull apache/airflow:latest-python3.13
```

## docker-compose.yml 작성

저는 LocalExecutor를 사용하고, meta를 저장할 DB는 외부에 postgresql로 이미 있기에 의존성을 제외하고 만들었습니다.

> AIRFLOW__API_AUTH__JWT_SECRET: **이거 꼭 만들어줘야 합니다. 아니면 LocalExecutor에서 state mismatch 에러가 발생합니다.**

```yml
x-airflow-common:
  &airflow-common
  image: apache/airflow:latest-python3.13
  build: ./docker-airflow-fab
  environment:
    &airflow-common-env
    AIRFLOW__CORE__EXECUTOR: LocalExecutor
    AIRFLOW__CORE__AUTH_MANAGER: airflow.providers.fab.auth_manager.fab_auth_manager.FabAuthManager
    AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://tkl:TKL%402635%21@1.1.1.1:12345/airflow_meta
    AIRFLOW__CORE__FERNET_KEY: ''
    AIRFLOW__API_AUTH__JWT_SECRET: 'pTirJtP+QjqHB0YgUcNNaCPtUdEmIrOjNt7QnQXX8XE='
    AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION: 'true'
    AIRFLOW__CORE__LOAD_EXAMPLES: 'False'
    AIRFLOW__CORE__EXECUTION_API_SERVER_URL: 'http://airflow-apiserver:8080/execution/'
    AIRFLOW__SCHEDULER__ENABLE_HEALTH_CHECK: 'true'
    _PIP_ADDITIONAL_REQUIREMENTS: ${_PIP_ADDITIONAL_REQUIREMENTS:-}
  volumes:
    - /home/airflow/eft-library-airflow/dags:/opt/airflow/dags
    - /home/airflow/eft-library-airflow/plugins:/opt/airflow/plugins
    - /home/airflow/logs:/opt/airflow/logs
    - /home/airflow/tmp:/opt/airflow/tmp
    - /home/airflow/latest_data:/opt/airflow/latest_data
  user: "${AIRFLOW_UID:-50000}:0"

services:
  airflow-apiserver:
    <<: *airflow-common
    command: api-server
    ports:
      - "8080:8080"
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:8080/api/v2/version"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always

  airflow-scheduler:
    <<: *airflow-common
    command: scheduler
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:8974/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always

  airflow-dag-processor:
    <<: *airflow-common
    command: dag-processor
    healthcheck:
      test: ["CMD-SHELL", 'airflow jobs check --job-type DagProcessorJob --hostname "$${HOSTNAME}"']
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always

  airflow-triggerer:
    <<: *airflow-common
    command: triggerer
    healthcheck:
      test: ["CMD-SHELL", 'airflow jobs check --job-type TriggererJob --hostname "$${HOSTNAME}"']
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always

  airflow-init:
    <<: *airflow-common
    entrypoint: /bin/bash
    command:
      - -c
      - |
        if [[ -z "${AIRFLOW_UID}" ]]; then
          echo
          echo -e "\033[1;33mWARNING!!!: AIRFLOW_UID not set!\e[0m"
          echo "If you are on Linux, you SHOULD follow the instructions below to set "
          echo "AIRFLOW_UID environment variable, otherwise files will be owned by root."
          echo "For other operating systems you can get rid of the warning with manually created .env file:"
          echo "    See: https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html#setting-the-right-airflow-user"
          echo
          export AIRFLOW_UID=$$(id -u)
        fi
        one_meg=1048576
        mem_available=$$(($$(getconf _PHYS_PAGES) * $$(getconf PAGE_SIZE) / one_meg))
        cpus_available=$$(grep -cE 'cpu[0-9]+' /proc/stat)
        disk_available=$$(df / | tail -1 | awk '{print $$4}')
        warning_resources="false"
        if (( mem_available < 4000 )) ; then
          echo
          echo -e "\033[1;33mWARNING!!!: Not enough memory available for Docker.\e[0m"
          echo "At least 4GB of memory required. You have $$(numfmt --to iec $$((mem_available * one_meg)))"
          echo
          warning_resources="true"
        fi
        if (( cpus_available < 2 )); then
          echo
          echo -e "\033[1;33mWARNING!!!: Not enough CPUS available for Docker.\e[0m"
          echo "At least 2 CPUs recommended. You have $${cpus_available}"
          echo
          warning_resources="true"
        fi
        if (( disk_available < one_meg * 10 )); then
          echo
          echo -e "\033[1;33mWARNING!!!: Not enough Disk space available for Docker.\e[0m"
          echo "At least 10 GBs recommended. You have $$(numfmt --to iec $$((disk_available * 1024 )))"
          echo
          warning_resources="true"
        fi
        if [[ $${warning_resources} == "true" ]]; then
          echo
          echo -e "\033[1;33mWARNING!!!: You have not enough resources to run Airflow (see above)!\e[0m"
          echo "Please follow the instructions to increase amount of resources available:"
          echo "   https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html#before-you-begin"
          echo
        fi
        echo
        echo "Creating missing opt dirs if missing:"
        echo
        mkdir -v -p /opt/airflow/{logs,dags,plugins,config}
        echo
        echo "Airflow version:"
        /entrypoint airflow version
        echo
        echo "Files in shared volumes:"
        echo
        ls -la /opt/airflow/{logs,dags,plugins,config}
        echo
        echo "Running airflow config list to create default config file if missing."
        echo
        /entrypoint airflow config list >/dev/null
        echo
        echo "Files in shared volumes:"
        echo
        ls -la /opt/airflow/{logs,dags,plugins,config}
        echo
        echo "Change ownership of files in /opt/airflow to ${AIRFLOW_UID}:0"
        echo
        chown -R "${AIRFLOW_UID}:0" /opt/airflow/
        echo
        echo "Change ownership of files in shared volumes to ${AIRFLOW_UID}:0"
        echo
        chown -v -R "${AIRFLOW_UID}:0" /opt/airflow/{logs,dags,plugins,config}
        echo
        echo "Files in shared volumes:"
        echo
        ls -la /opt/airflow/{logs,dags,plugins,config}

    environment:
      <<: *airflow-common-env
      _AIRFLOW_DB_MIGRATE: 'true'
      _AIRFLOW_WWW_USER_CREATE: 'true'
      _AIRFLOW_WWW_USER_USERNAME: 'admin'
      _AIRFLOW_WWW_USER_PASSWORD: 'password'
      _PIP_ADDITIONAL_REQUIREMENTS: ''
    user: "0:0"
```

## DockerFile 생성

Airflow 3.X 버전 부터는 Fab가 완전 제거 되었습니다. 기존의 로그인 방식이었는데 보안을 우려해서인지 완전 빼버렸습니다.      
이를 대신하는 SimpleManager를 사용해서 Json으로 계정과 비밀번호를 관리하는 방법이 있는데, Fab 쓰는게 더 편해서 수정했습니다.  

> SimpleManager를 사용하는 경우에는 아이디와, 비밀번호를 직접 만들 수 없고 api-server 에서 만들어주는 것을 사용해야 합니다.

```shell
# 폴더 생성 및 이동
mkdir docker-airflow-fab
cd docker-airflow-fab

# 작성 들어가기
vi Dockerfile

# 여기서 부터는 내용 입력
# 기존 Airflow 이미지를 기반으로 사용
FROM apache/airflow:latest-python3.13

# FAB provider 관련 패키지 설치 - 없으면 에러 남
RUN pip install --no-cache-dir --user apache-airflow-providers-fab flask_appbuilder flask-session "connexion<3"

# 다시 airflow 유저로 변경 (권장)
USER ${AIRFLOW_UID:-50000}
```

## Docker Compose UP

실행하면서 에러는 없는지 확인합니다.

```shell
docker compose up —build
```

## PG Dump 설정하기 (.pgpass)

PostgreSQL 데이터를 Dump 뜨기 위해 Airflow Dag를 만들어도 권한 문제와 비밀번호를 묻는 것이 나와서 막히게 됩니다.

Airflow Dag를 실행하는 주체는 Docker의 Container이니 내부에서 .pgpass를 설정하여 비밀번호를 묻지 않게 설정해줘야 합니다.

```shell
# scheduler container 접속
docker exec -it --user airflow <scheduler container id> /bin/bash

# pgpass 작성
cd /home/airflow
echo "192.168.219.102:dbport:db:user:password" >> ~/.pgpass
chmod 600 /home/airflow/.pgpass
chown airflow:root .pgpass

# test
pg_dump -h 192.168.219.102 -p dbport -U user --inserts db > /opt/airflow/latest_data/test_backup.sql
```

# UI에서 커넥션 설정하기

DB와 Mail 커넥션을 설정했습니다.

## DB Connection

아래 사진에 정보를 입력하면 됩니다.

## Gmail Connection

저는 메일의 경우 Gmail을 사용했는데, 다음과 같이 하면 됩니다.

비밀번호는 [Gmail IMAP 생성](https://milkyspace.tistory.com/131) 여기를 보고 따라하시면 될 것 같습니다.

이렇게 하면 모든 설정이 완료 됩니다.
